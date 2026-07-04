# How to Run nginx as a TLS Reverse Proxy in Front of a Private Backend on AWS (Let's Encrypt, Ubuntu 24.04 and Amazon Linux 2023)

> **Published:** 2026-07-04. **Last verified:** 2026-07-04 against the nginx
> reference documentation (`ngx_http_proxy_module`, `ngx_http_v2_module`,
> `ngx_http_core_module`), the [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
> (guidelines v6.0, intermediate profile), the Certbot user guide and
> [`certbot-dns-route53`](https://certbot-dns-route53.readthedocs.io/en/stable/)
> docs, Let's Encrypt's own posts on certificate profiles, OCSP end-of-life, and
> expiration emails, and AWS documentation for VPC, security groups, NAT gateways,
> Session Manager, and IMDSv2. Package versions checked: Ubuntu Server 24.04 LTS
> (`nginx` 1.24.0, `certbot` from the EFF snap; the `apt` `certbot` is 2.9.0),
> Amazon Linux 2023 (`nginx` 1.28.0, `certbot` from a pip virtualenv). Covers
> Ubuntu Server 24.04 LTS and Amazon Linux 2023.

> A reverse proxy is a server that accepts requests from the internet and forwards
> them to an application that never faces the internet directly. This guide puts
> nginx in a public subnet with a Let's Encrypt certificate, terminates TLS there,
> and forwards to a backend in a private subnet that has no public IP address. The
> result is one hardened, patchable front door and an application you can move,
> replace, or scale behind it without changing what the public sees.

* * *

## What you are building

```
                Internet
                   │  443 (TLS), 80 (redirect + ACME)
                   ▼
        ┌─────────────────────┐   public subnet (route to internet gateway)
        │   nginx reverse      │   Elastic IP, DNS A record points here
        │   proxy  (EC2)       │   terminates TLS, adds security headers
        └─────────┬───────────┘
                  │  HTTP (or re-encrypted TLS) over the private network
                  │  allowed only from the proxy's security group
                  ▼
        ┌─────────────────────┐   private subnet (no route to internet gateway)
        │   backend app (EC2)  │   no public IP, reachable only from the proxy
        │   listens on :8080   │   outbound updates via NAT gateway
        └─────────────────────┘
```

Three ideas shape the design:

- **The subnet type is defined by routing.** A public subnet is one whose route
  table has a route to an internet gateway. A private subnet has no such route,
  so nothing in it is reachable from the internet, and it reaches out only through
  a NAT gateway.
- **The security group is the real firewall on AWS.** Security groups are
  stateful and attach to instances. The backend accepts the application port only
  from the proxy's security group, referenced by security-group ID rather than by
  IP range, so the rule keeps working when addresses change.
- **nginx is the single audited entry point.** It is the only host with a public
  address, the only place TLS is terminated, and the only place you manage
  certificates and security headers.

What this does not do: it is not a web application firewall, it does not patch the
backend for you, and it does not authenticate users. It reduces the internet-facing
surface to one server you control and harden. Application-layer attacks that ride
valid HTTPS requests still reach the backend, so keep the app itself maintained.

* * *

## Prerequisites

- An AWS account, and a registered domain whose DNS you can edit. Route 53 is
  assumed for the DNS-01 path in Step 6, but any DNS provider works for the
  HTTP-01 path.
- A VPC you can add subnets and route tables to. The default VPC works for a first
  run, but the steps below assume you create the pieces yourself so the routing is
  explicit.
- Familiarity with launching EC2 instances and attaching an IAM instance role.
- A backend application that listens on a known port (this guide uses `8080` on
  `10.0.20.10`, the backend's private IP). Anything that speaks HTTP works: a
  container, a Node or Python service, another web server.
- Two warnings before you start. TLS configuration and certificate issuance are
  reversible, but the AWS charges are not free: a public IPv4 address (including an
  in-use Elastic IP) is billed hourly since 1 February 2024, and a NAT gateway
  bills per hour and per gigabyte. Tear the lab down when you are done.

* * *

## Step 1: Build the network

You need one VPC, two subnets, an internet gateway, a NAT gateway, and two route
tables. The console's "VPC and more" wizard creates all of this in one pass; the
pieces and the routing that make a subnet public or private are what matter.

- **Internet gateway (IGW):** attach one to the VPC. There is one IGW per VPC.
- **Public subnet:** its route table sends `0.0.0.0/0` to the IGW. Put the nginx
  proxy and the NAT gateway here.
- **Private subnet:** its route table sends `0.0.0.0/0` to the NAT gateway, not to
  the IGW. Put the backend here.
- **NAT gateway:** create it in the public subnet and give it an Elastic IP. It
  lets the private backend reach the internet outbound (package updates, and the
  Route 53 API if you run DNS-01 from the backend) while staying unreachable from
  it. A NAT gateway is billed per hour and per gigabyte processed. For IPv6-only
  outbound, an egress-only internet gateway does the same job without the NAT
  charge; a self-managed NAT instance is the older, cheaper, higher-maintenance
  alternative.

Give the proxy a stable public address and point DNS at it:

- Allocate an **Elastic IP** and associate it with the proxy instance (Step 3).
- Create a DNS **A record** for `example.com` (and `www` if you use it) pointing to
  that Elastic IP.

Since 1 February 2024, every public IPv4 address is billed at about `$0.005` per
hour whether or not it is attached, so the proxy's Elastic IP and the NAT
gateway's Elastic IP both cost roughly `$3.65` a month each. Budget for them; they
are not the old "free while in use" Elastic IPs.

* * *

## Step 2: Write the security groups

Two security groups enforce the segmentation. Security groups are stateful, so you
only write the inbound rules; return traffic is allowed automatically.

**Proxy security group** (attached to the nginx instance):

| Direction | Port | Source | Why |
| --- | --- | --- | --- |
| Inbound | 443 | `0.0.0.0/0`, `::/0` | Public HTTPS |
| Inbound | 80 | `0.0.0.0/0`, `::/0` | HTTP→HTTPS redirect and Let's Encrypt HTTP-01 |
| Inbound | 22 | *(omit; use Session Manager)* | Avoid a public SSH port |
| Outbound | all | default allow | Reach the backend, Let's Encrypt, updates |

**Backend security group** (attached to the app instance):

| Direction | Port | Source | Why |
| --- | --- | --- | --- |
| Inbound | 8080 | **the proxy security group ID** (`sg-…`) | Only the proxy may reach the app |
| Inbound | 22 | *(omit; use Session Manager)* | No management port exposed |
| Outbound | all | default allow | Updates via the NAT gateway |

The load-bearing detail is the backend's inbound rule: set the **source to the
proxy's security-group ID**, not a CIDR. AWS supports referencing one security
group from another (within the same VPC, or across peering or a transit gateway),
and it is the pattern AWS documents for a web tier in front of an app tier. Nothing
outside the proxy group can open the application port, and the rule survives
instance replacement and IP changes.

Do not open port 22 to the internet on either host. Step 3 uses AWS Systems
Manager Session Manager for shell access, which needs no inbound port at all.

* * *

## Step 3: Launch the instances

Launch two EC2 instances, the proxy into the public subnet (with the Elastic IP
from Step 1) and the backend into the private subnet (no public IP). Two settings
matter for security on both.

**Require IMDSv2.** Set instance metadata to token-required (`HttpTokens=required`).
A reverse proxy is a classic server-side request forgery (SSRF) target, and IMDSv2
blocks the standard path where a tricked proxy is used to read instance
credentials from `169.254.169.254`: the metadata service requires a PUT to mint a
token before any read, and the token response carries an IP hop limit of 1 by
default, so it cannot be relayed off the instance. Set this at launch.

**Attach an IAM instance role for Session Manager.** Give both instances an
instance profile carrying the AWS managed policy `AmazonSSMManagedInstanceCore`.
The SSM Agent is preinstalled on the standard Ubuntu 24.04 LTS and Amazon Linux
2023 AMIs. With the role attached and a network path to the SSM endpoints (via the
NAT gateway, or via `ssm` and `ssmmessages` VPC interface endpoints plus an S3
gateway endpoint if you want the backend to need no NAT at all), you get a shell
on the private backend with:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

No bastion, no SSH key, no inbound port. That command runs from your workstation
and needs the AWS CLI Session Manager plugin installed there, which is a separate
download from the AWS CLI itself; without it, `start-session` fails with a
missing-plugin error. Confirm the agent is running on either host with
`systemctl status amazon-ssm-agent` (Ubuntu: `snap.amazon-ssm-agent…`).

* * *

## Step 4: Install nginx on the proxy

The package and paths differ by distribution.

**Ubuntu 24.04:**

```bash
sudo apt update && sudo apt install nginx
sudo systemctl enable --now nginx
```

nginx runs as `www-data`. Server blocks live in `/etc/nginx/sites-available/` and
are enabled by symlink into `/etc/nginx/sites-enabled/`; the default web root is
`/var/www/html`.

**Amazon Linux 2023:**

```bash
sudo dnf install nginx
sudo systemctl enable --now nginx
```

nginx is in the default AL2023 repositories, so there is no `amazon-linux-extras`
step (that mechanism belonged to Amazon Linux 2). nginx runs as `nginx`. Server
blocks live in `/etc/nginx/conf.d/*.conf`; the default web root is
`/usr/share/nginx/html`.

On both, validate any config change with `sudo nginx -t` and apply it with
`sudo systemctl reload nginx`.

Confirm you can reach the proxy over plain HTTP on its public address before going
further. If AL2023 is switched to SELinux enforcing mode, note the reverse-proxy
gotcha in Step 5.

* * *

## Step 5: Configure the reverse proxy (HTTP first)

Get proxying working over plain HTTP, then add TLS in Step 6. Testing in two
stages keeps a TLS problem from hiding a proxy problem.

Create a server block. On Ubuntu put it in
`/etc/nginx/sites-available/example.com` and symlink it into `sites-enabled/`; on
AL2023 put it in `/etc/nginx/conf.d/example.com.conf`.

```nginx
upstream backend {
    server 10.0.20.10:8080;   # the backend's private IP and port
    keepalive 16;
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
    }
}
```

What each proxy header does, and why the defaults are wrong for this job:

- `Host $host` overrides nginx's default of sending the upstream's own name, so the
  backend sees the hostname the client actually requested.
- `X-Forwarded-For $proxy_add_x_forwarded_for` appends the client IP to any
  existing chain, so the backend can log the real client rather than the proxy.
- `X-Forwarded-Proto $scheme` tells the backend the original request was HTTPS once
  TLS is in place, which is what lets the app build correct `https://` URLs and set
  `Secure` cookies.
- `proxy_http_version 1.1` with `keepalive` reuses upstream connections. Set it
  explicitly for portability across nginx versions.

For a WebSocket backend, add the connection-upgrade headers on that location:

```nginx
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection "upgrade";
```

Reload, then confirm a request to the proxy reaches the backend:

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I http://example.com/
```

**AL2023 with SELinux enforcing:** AL2023 ships SELinux in permissive mode by
default, so a fresh box proxies fine. The moment you harden it to enforcing, nginx
is blocked from opening the outbound connection to the backend and every request
returns 502 with a permission-denied entry in the error log. Allow it with the
boolean that fits your backend port:

```bash
# backend on a non-HTTP port such as 8080:
sudo setsebool -P httpd_can_network_connect 1
# or, to allow relaying only to standard HTTP ports (80, 443, 8008, 8443, ...):
sudo setsebool -P httpd_can_network_relay 1
```

Ubuntu does not confine nginx with AppArmor by default (the nginx package ships no
profile), so there is no equivalent gate there.

* * *

## Step 6: Get a Let's Encrypt certificate

Two things to know about Let's Encrypt before you issue anything in 2026, because
both changed recently:

- **You self-monitor renewal now.** Let's Encrypt stopped sending expiration
  reminder emails on 4 June 2025. Rely on the automated renewal from Step 8 and add
  your own certificate monitoring; do not count on a warning from the CA.
- **Certificate lifetimes are shrinking.** The default `classic` profile is still
  90 days. The `tlsserver` profile moved to 45 days on 13 May 2026, and a 6-day
  `shortlived` profile is generally available. Certbot selects a profile with
  `--preferred-profile <name>`; leaving it unset gives you the 90-day default,
  which is right for a first deployment. Whatever the lifetime, automated renewal
  is what keeps the site up.

Rate limits worth knowing while testing: 50 certificates per registered domain per
week, 5 duplicate certificates (identical name set) per week, and 5 failed
validations per hostname per hour. Use the `--staging` environment while you are
working out the flow so a mistake does not burn a limit.

### Install Certbot

**Ubuntu 24.04** (EFF recommends the snap, which stays current and installs its own
renewal timer; the `apt` package is 2.9.0 and lags):

```bash
sudo apt remove certbot 2>/dev/null    # drop any apt version first
sudo snap install core && sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

**Amazon Linux 2023** has no EPEL support, no `dnf` Certbot package, and no snapd,
so install Certbot into a Python virtualenv. AL2023's system Python is 3.9, and
current Certbot needs Python 3.10 or newer, so build the venv with a newer
interpreter; `python3.11` is packaged for AL2023:

```bash
sudo dnf install -y python3.11 python3.11-pip augeas-libs
sudo python3.11 -m venv /opt/certbot/
sudo /opt/certbot/bin/pip install --upgrade pip
sudo /opt/certbot/bin/pip install certbot certbot-nginx
sudo ln -s /opt/certbot/bin/certbot /usr/bin/certbot
```

The pip install does not create a renewal timer. Step 8 adds one by hand for this
path.

### Path A: HTTP-01 (simplest, needs port 80)

The proxy already answers on port 80 and its security group allows it, so the
nginx plugin can prove control of the name and install the certificate:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot obtains the certificate, edits the server block to add the `listen 443 ssl`
configuration and the certificate paths, and offers to add the HTTP→HTTPS
redirect. HTTP-01 cannot issue wildcards. If you would rather Certbot obtain the
certificate without making permanent edits to your nginx config, use
`sudo certbot certonly --nginx -d example.com` (the nginx plugin authenticates
only, reverting its temporary changes) and wire up the TLS server block yourself
from Step 7.

### Path B: DNS-01 via Route 53 (no port 80 needed, supports wildcards)

DNS-01 proves control by writing a TXT record instead of answering on port 80, so
it works when port 80 is closed and it is the only way to get a wildcard from
Let's Encrypt. Install the Route 53 plugin (`sudo /opt/certbot/bin/pip install
certbot-dns-route53` on AL2023; on the Ubuntu snap, `sudo snap set certbot
trust-plugin-with-root=ok && sudo snap install certbot-dns-route53`), then attach
an IAM instance role to the host running Certbot with this least-privilege policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["route53:ListHostedZones", "route53:GetChange"],
      "Resource": ["*"] },
    { "Effect": "Allow",
      "Action": ["route53:ChangeResourceRecordSets"],
      "Resource": ["arn:aws:route53:::hostedzone/YOURHOSTEDZONEID"] }
  ]
}
```

Route 53 does not support resource scoping on `ListHostedZones` or `GetChange`, so
those stay on `*`; the write action is scoped to your one hosted zone. Then issue,
quoting the wildcard so the shell does not expand it:

```bash
sudo certbot certonly --dns-route53 -d example.com -d '*.example.com'
```

Prefer the instance role over static access keys. Credentials come from the
standard AWS SDK chain, so no keys need to sit on disk.

* * *

## Step 7: Harden the TLS configuration

If you used `certbot --nginx`, it wrote a working but generic TLS block. Replace
the TLS and header settings with the current Mozilla "intermediate" profile plus a
small set of response headers. This block assumes Certbot placed the certificate at
the standard path.

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;                       # see the version note below
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Mozilla intermediate (guidelines v6.0), ECDHE-only, no custom dhparam needed
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:MozSSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # Response headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Content-Security-Policy "frame-ancestors 'self';" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    server_tokens off;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

The choices, and the ones people get wrong in 2026:

- **No OCSP stapling.** Let's Encrypt stopped putting an OCSP URL in its
  certificates in May 2025 and shut its OCSP responders off in August 2025.
  `ssl_stapling` has nothing to fetch for a Let's Encrypt certificate and does
  nothing, so it is left out rather than switched on. Revocation now rides CRLs.
- **No custom `ssl_dhparam`.** The current intermediate cipher list is ECDHE-only;
  Mozilla dropped the DHE ciphers that needed a Diffie-Hellman parameter file, so
  there is none to generate.
- **`http2 on;` needs nginx 1.25.1 or newer, which not every package has.** Amazon
  Linux 2023 ships nginx 1.28.0, so `http2 on;` works there as written. Ubuntu
  24.04 LTS is held at nginx 1.24.0, which predates the directive; on stock Ubuntu,
  use the older form `listen 443 ssl http2;` instead of a separate `http2 on;`
  line, or add the official [nginx.org repository](https://nginx.org/en/linux_packages.html)
  (stable branch well past 1.25.1) to get the modern directive and a newer nginx.
- **HSTS with care.** `max-age=63072000; includeSubDomains` tells browsers to use
  HTTPS only for two years. Add `; preload` and submit to hstspreload.org only once
  every subdomain serves HTTPS, because preload is hard to undo. The header is
  honored only over HTTPS.
- **`X-XSS-Protection` is deliberately absent.** It is deprecated and can introduce
  vulnerabilities; a Content-Security-Policy is the replacement.
- **`add_header` inheritance gotcha:** an `add_header` inside a `location` block
  replaces, not extends, the headers set in the parent `server` block. If you add
  headers in a location, repeat the ones you still want.

Reload and test:

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -sI https://example.com/ | grep -i -E 'strict-transport|x-frame|x-content|server'
```

For an external grade, run the domain through the Qualys SSL Labs test or
`testssl.sh`. A clean intermediate profile scores an A.

* * *

## Step 8: Automate renewal

**Ubuntu (snap or apt):** a systemd timer that runs `certbot renew` twice a day is
installed for you. Certbot only renews a certificate once it is inside the renewal
window (less than a third of its lifetime remaining, so about 30 days out on a
90-day certificate), and doing nothing otherwise. Attach a reload hook so nginx
picks up the new certificate, and dry-run against staging:

```bash
sudo certbot renew --deploy-hook "systemctl reload nginx" --dry-run
```

The hook you pass at issuance or on a real renewal is saved into the certificate's
renewal config and reused automatically, so you do not need to set it every time.

**Amazon Linux 2023 (pip venv):** the pip install created no timer, so add one.
The simplest reliable option is a cron entry:

```bash
echo '0 0,12 * * * root /opt/certbot/bin/certbot renew --deploy-hook "systemctl reload nginx" -q' \
  | sudo tee /etc/cron.d/certbot-renew
```

Confirm the renewal path works before you rely on it:

```bash
sudo certbot renew --dry-run
```

Because Let's Encrypt no longer emails you before expiry, add external monitoring
that alerts on approaching expiry independent of the box doing the renewing. A
failed timer on a machine you are not watching is how sites go dark.

* * *

## Optional: re-encrypt to the backend

Terminating TLS at the proxy and forwarding plain HTTP over the private subnet is
common and is protected here by the backend security group. Where a policy
requires encryption in transit all the way to the application (regulated data, a
zero-trust posture, or a shared network you do not fully trust), have nginx open a
TLS connection to the backend instead. nginx does not verify the backend
certificate by default, so turn verification on explicitly:

```nginx
location / {
    proxy_pass https://backend;            # backend listens on TLS, e.g. :8443
    proxy_ssl_verify              on;      # default is off; you must enable it
    proxy_ssl_trusted_certificate /etc/nginx/backend-ca.pem;
    proxy_ssl_name                backend.internal;
    proxy_ssl_server_name         on;      # send SNI; default is off
    proxy_ssl_protocols           TLSv1.2 TLSv1.3;
    # ... the same proxy_set_header lines as before ...
}
```

This adds certificate management on the backend (its own certificate and a CA the
proxy trusts). An internal CA or AWS Private CA issues those; public Let's Encrypt
certificates are for the public name on the proxy, not for private backends.

* * *

## Troubleshooting

- **502 Bad Gateway.** nginx cannot reach the backend. Check the backend security
  group allows the app port from the proxy's security-group ID, that the app is
  listening on the private IP (not only `127.0.0.1`), and, on AL2023 in enforcing
  mode, that `httpd_can_network_connect` is set (Step 5).
- **Certbot HTTP-01 fails to validate.** Port 80 must be open to the internet on
  the proxy security group and reach nginx. Confirm the DNS A record points to the
  proxy's Elastic IP and has propagated.
- **Certbot DNS-01 fails with an access error.** The IAM role is missing or
  unscoped. Recheck the three Route 53 actions and that the write action names your
  hosted-zone ID.
- **Certbot on AL2023 refuses to install or runs an old version.** You built the
  venv with system Python 3.9. Rebuild it with `python3.11` (Step 6).
- **The certificate is not renewing.** On AL2023, confirm the cron entry exists;
  on Ubuntu, check `systemctl list-timers | grep certbot`. Run
  `sudo certbot renew --dry-run` to see the actual failure.
- **Cannot reach the backend to administer it.** It has no public IP by design. Use
  `aws ssm start-session --target <instance-id>`; if that fails, the instance is
  missing the `AmazonSSMManagedInstanceCore` role or has no network path to the SSM
  endpoints (NAT gateway or VPC endpoints).
- **`http2 on;` is rejected by nginx -t.** Your nginx is older than 1.25.1 (stock
  Ubuntu 24.04 is 1.24.0). Use `listen 443 ssl http2;`, or install nginx from
  nginx.org. Amazon Linux 2023 (nginx 1.28.0) accepts `http2 on;` as written.

* * *

## What you achieved

- A single hardened, internet-facing nginx proxy terminates TLS with a current
  Mozilla intermediate profile and a Let's Encrypt certificate that renews itself.
- The application runs in a private subnet with no public IP, reachable only from
  the proxy's security group, and administered without a bastion or an open SSH
  port through Session Manager.
- The design is repeatable: the certificate renews on a timer, the backend can be
  replaced behind the proxy without touching the public entry point, and you can
  re-encrypt to the backend when policy calls for it.

* * *

## Framework alignment

Two published frameworks are anchored here, each checked against its current text.

**AWS Well-Architected Framework, Security pillar:**

- **Least-privilege access (SEC03-BP02).** The backend security group admits only
  the proxy's security group on the application port, and the DNS-01 Route 53 role
  is scoped to a single hosted zone.
- **Apply security at all layers.** Public exposure is one hardened proxy, the
  backend has no public IP, and administration runs through Session Manager rather
  than an open SSH port.
- **Reduce the attack surface on compute.** IMDSv2 is required, which closes the
  SSRF path to instance credentials.
- **Encryption in transit (SEC09-BP02).** TLS terminates at the proxy with the
  Mozilla intermediate profile, with the option to re-encrypt to the backend.

**OWASP Application Security Verification Standard 5.0.0.** The TLS and
response-header settings in Step 7 implement these requirements:

| This guide sets | ASVS 5.0.0 |
| --- | --- |
| HSTS, `max-age` two years with `includeSubDomains` | 3.4.1 |
| `Content-Security-Policy` with `frame-ancestors 'self'` | 3.4.3, 3.4.6 |
| `X-Content-Type-Options: nosniff` | 3.4.4 |
| `Referrer-Policy` | 3.4.5 |
| TLS 1.2 and 1.3 with recommended cipher suites | 12.1.1, 12.1.2 |
| Publicly trusted TLS to external clients (Let's Encrypt) | 12.2.1, 12.2.2 |

ASVS 3.4.6 treats `X-Frame-Options` as obsolete and relies on the CSP
`frame-ancestors` directive, which this guide sets; the `X-Frame-Options` line is
kept only for older browsers.

* * *

## References

- [nginx: `ngx_http_proxy_module`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [nginx: `ngx_http_v2_module` (the `http2` directive)](https://nginx.org/en/docs/http/ngx_http_v2_module.html)
- [nginx official Linux packages](https://nginx.org/en/linux_packages.html)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [`certbot-dns-route53` plugin](https://certbot-dns-route53.readthedocs.io/en/stable/)
- [Let's Encrypt: certificate profiles](https://letsencrypt.org/docs/profiles/)
- [Let's Encrypt: OCSP service end of life](https://letsencrypt.org/2025/08/06/ocsp-service-has-reached-end-of-life/)
- [Let's Encrypt: expiration notification service has ended](https://letsencrypt.org/2025/06/26/expiration-notification-service-has-ended/)
- [Let's Encrypt: rate limits](https://letsencrypt.org/docs/rate-limits/)
- [AWS: subnets and routing (public vs private)](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [AWS: NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [AWS: security group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [AWS: Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [AWS: use IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)
- [AWS: public IPv4 address charge](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)
- [MDN: HTTP security headers (HSTS, X-Frame-Options, Referrer-Policy)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Strict-Transport-Security)
- [AWS Well-Architected Framework: Security pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- [OWASP Application Security Verification Standard (ASVS)](https://owasp.org/www-project-application-security-verification-standard/)
