# Frameworks and standards

The guides and topics in this repository are written against published security
frameworks and standards. The source documents themselves are **not committed
here**. Several carry licenses that restrict redistribution, so each is kept as a
local reference only and should be downloaded from its official source below.

The subfolders (`AWS/`, `CIS/`, `ISO/`, `NIST/`, `OWASP/`) are where local copies
live on a working checkout; the PDFs are excluded by `.gitignore`.

## Sources

- **[NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)** -
  Security and Privacy Controls for Information Systems and Organizations. Public
  domain (US Government work); free to download.
- **[OWASP Application Security Verification Standard 5.0.0](https://owasp.org/www-project-application-security-verification-standard/)** -
  Application security requirements and verification criteria. Licensed CC BY-SA;
  free to download.
- **[CIS Ubuntu Linux 24.04 LTS Benchmark](https://www.cisecurity.org/benchmark/ubuntu_linux)** -
  Consensus hardening baseline (current published version v2.0.0). Free download
  after registering an email with CIS; redistribution is restricted by the CIS
  license.
- **[AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)** -
  Cloud architecture pillars and best practices. Copyright Amazon Web Services;
  read online, not licensed for redistribution.
- **[ISO/IEC 27001:2022](https://www.iso.org/standard/27001)** -
  Information security management systems, Requirements (Edition 3). Copyrighted
  standard, purchase from ISO.

## Which guides use them

- **CIS** backs the [CIS with OpenSCAP](../guides/cis-hardening-with-openscap.md),
  [host firewall](../guides/host-firewall-hardening.md), and
  [kernel sysctl](../guides/kernel-network-hardening-sysctl.md) guides.
- **NIST SP 800-53** informs the host hardening and remote access guides.
- **Mozilla TLS** guidance and the **AWS Well-Architected** security pillar back the
  [nginx reverse proxy on AWS](../guides/nginx-reverse-proxy-letsencrypt-aws.md)
  guide.
- **ISO/IEC 27001** underpins the governance topics
  ([Shadow IT](../topics/shadow-it.md),
  [ISMS documentation structure](../topics/isms-structure.md)).
