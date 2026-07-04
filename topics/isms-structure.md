---
title: "A Practical ISMS Documentation Structure for ISO/IEC 27001:2022"
description: A reference folder structure for an ISO/IEC 27001:2022 Information Security Management System, organised around the four Annex A control themes and the management-system clauses (4-10), with notes on placement and mandatory documented information.
last_verified: 2026-07-03
---

# A Practical ISMS Documentation Structure for ISO/IEC 27001:2022

When you build an Information Security Management System (ISMS), one of the first
practical questions is *"where does every document go?"* A clear, consistent file
structure makes the ISMS easier to operate day to day and far easier to evidence
during a certification or surveillance audit.

This page proposes a reference structure with two distinct halves, which reflects
how ISO/IEC 27001:2022 is actually organised:

1. **Annex A controls**: grouped into the **four control themes** introduced in
   the 2022 revision. Annex A lists **93 controls** across:
   - **A.5 Organizational** (37 controls)
   - **A.6 People** (8 controls)
   - **A.7 Physical** (14 controls)
   - **A.8 Technological** (34 controls)

   This replaced the 2013 edition's 14 domains (A.5-A.18, 114 controls).

2. **Management-system clauses**: clauses **4 to 10** of the main standard
   (Context, Leadership, Planning, Support, Operation, Performance Evaluation,
   Improvement). These hold the **mandatory documented information** that a
   certification auditor will expect to see, independent of which Annex A
   controls you apply.

> **How to read the tree.** The numbered policy folders and the theme grouping
> are *this organisation's own taxonomy*, not a literal copy of Annex A control
> numbers. A policy often supports several controls, and a few controls (noted
> below) sit in a different theme than where the policy is filed for operational
> convenience. Empty `SOPs/` and `Templates & Logs/` folders are intentional
> placeholders, to be populated with evidence and records as the ISMS matures.

```
ISMS Documentation/
├── A.5 Organizational Controls/
│   ├── Evidence/
│   ├── Policies/
│   │   ├── 01-Backup and Restoration Policy/
│   │   ├── 02-Business Continuity and Disaster Recovery Policy/
│   │   ├── 03-Corporate Ethics Policy/
│   │   ├── 04-Incident Management Policy/
│   │   ├── 05-Information Security Policy/
│   │   ├── 06-Internal Audits Policy/
│   │   ├── 07-Risk Assessment Policy/
│   │   ├── 08-Vendor Management Policy/
│   │   ├── 09-Acceptable Use Policy/
│   │   └── 10-Social Media Policy/
│   ├── SOPs/
│   │   ├── User Provisioning (Onboarding)/
│   │   └── User Provisioning (Offboarding)/
│   └── Templates & Logs/
│       ├── Asset_Inventory.xlsx
│       ├── New Hire IT Form/
│       └── Vendor Security Assessment (VSA) Questionnaire/
├── A.6 People Controls/
│   ├── Evidence/
│   ├── Policies/
│   │   ├── 11-Disciplinary Policy/
│   │   ├── 12-Personnel Security Policy/
│   │   └── 13-Security Awareness and Training Policy/
│   ├── SOPs/
│   └── Templates & Logs/
├── A.7 Physical Controls/
│   ├── Evidence/
│   ├── Policies/
│   │   ├── 14-Clean Desk and Clear Screen Policy/
│   │   ├── 15-Equipment Handling and Disposal Policy/
│   │   └── 16-Physical and Environmental Security Policy/
│   ├── SOPs/
│   └── Templates & Logs/
├── A.8 Technological Controls/
│   ├── Evidence/
│   ├── Policies/
│   │   ├── 17-Access Control Policy/
│   │   ├── 18-Data Integrity Policy/
│   │   ├── 19-Data Retention and Disposal Policy/
│   │   ├── 20-Key Management and Cryptography Policy/
│   │   ├── 21-Network Security Policy/
│   │   ├── 22-Server Security Policy/
│   │   ├── 23-Vulnerability and Penetration Testing Policy/
│   │   ├── 24-Workstation and Mobile Device Policy/
│   │   ├── 25-Change Management Policy/
│   │   ├── 26-Secure Development Policy/
│   │   ├── 27-Cloud Security Policy/          # supports A.5.23 (an Organizational-theme control)
│   │   └── 28-Bring Your Own Device (BYOD) Policy/
│   ├── SOPs/
│   │   └── Change Management Process/
│   ├── Standards/
│   │   └── Data Governance Standard/
│   └── Templates & Logs/
├── Clause-Based Documentation/
│   ├── 4. Context of the Organization/
│   │   ├── Context_Analysis_4.1_and_4.2.docx
│   │   ├── Interested Parties & Requirements (4.2)/
│   │   └── ISMS Scope (4.3)/                  # mandatory
│   ├── 5. Leadership/
│   │   ├── Information Security Policy (5.2)/  # the top-level policy; see also A.5 / 05
│   │   └── Roles, Responsibilities & Authorities (5.3)/
│   ├── 6. Planning/
│   │   ├── Risk Assessment Methodology (6.1.2)/
│   │   ├── Risk Treatment Plan (6.1.3 e)/     # mandatory
│   │   ├── Statement of Applicability (SOA).xlsx   # mandatory (6.1.3 d)
│   │   ├── Information Security Objectives (6.2)/   # mandatory
│   │   └── Planning of Changes (6.3)/              # new in 2022; no mandatory record
│   ├── 7. Support/
│   │   ├── Competence & Training Records (7.2)/    # mandatory
│   │   ├── Communication Plan (7.4)/
│   │   └── Documented Information Control (7.5)/
│   ├── 8. Operation/
│   │   ├── Operational Planning & Control (8.1)/
│   │   ├── Risk_Assessment_Tracking.xlsx      # risk assessment results (8.2)
│   │   └── Risk Treatment Results (8.3)/
│   ├── 9. Performance Evaluation/
│   │   ├── Monitoring & Measurement Results (9.1)/
│   │   ├── Internal Audit Programme & Reports (9.2)/
│   │   └── Management Review Minutes (9.3)/
│   └── 10. Improvement/
│       ├── Continual Improvement Log (10.1)/
│       └── Nonconformities & Corrective Actions (10.2)/
├── Annex_Controls_Aligned_With_SOA.xlsx
├── Security Stack/
└── Meeting Minutes/
```

## Placement notes and rationale

A few placement decisions are worth calling out, because they are common points
of confusion:

- **Social Media Policy lives under A.5 Organizational, not A.6 People.**
  Governing acceptable use of communications and information maps to the
  Organizational theme (A.5.10 *Acceptable use of information and other
  associated assets*, with A.5.14 *Information transfer* for what may be posted),
  so it sits alongside the **Acceptable Use Policy**. A.6 People is reserved for
  the human-resource lifecycle (screening, terms, awareness, disciplinary,
  remote working, event reporting).
- **Asset_Inventory.xlsx is filed under A.5 Organizational.** The inventory of
  information and other associated assets is control **A.5.9** (Organizational)
  and is ISMS-wide, so it does not belong under A.8 Technological.
- **Risk assessment records live under the clause-based area, not Annex A.** The
  risk assessment *process* is a clause 6.1.2 / 8.2 activity and the risk
  *treatment* is 6.1.3 / 8.3. These are management-system records, so
  `Risk_Assessment_Tracking.xlsx` sits under **8. Operation**.
- **Cloud Security Policy is filed under A.8 for operational convenience**, but
  the underlying control **A.5.23** (*Information security for use of cloud
  services*) is formally in the **Organizational** theme. Either placement is
  defensible as long as the mapping is documented (e.g. in the SoA).
- **Workstation/Mobile (24) and BYOD (28)** overlap (both relate to A.8.1 *User
  endpoint devices*); keep them separate only if your BYOD rules differ
  materially from corporate-device rules, otherwise consider consolidating.
- **Clause 6.3 *Planning of changes* is new in the 2022 revision.** It requires
  changes to the ISMS to be made in a planned way but mandates no documented
  information of its own. The `Planning of Changes (6.3)` folder above is a
  convenience placeholder; the planning is usually evidenced through management
  review (9.3) and the change management process, so it carries no mandatory record.
- **Clause 4 context now includes climate change.** Amendment 1:2024 to ISO/IEC
  27001:2022 added a requirement to clause 4.1 (determine whether climate change
  is a relevant issue) and a note to 4.2 (interested parties can have
  climate-related requirements). Record that determination in the
  `Context_Analysis_4.1_and_4.2` document; the amendment adds no controls and
  changes no numbering.

## Mandatory documented information (ISO/IEC 27001:2022)

The clause-based half exists so the documents an auditor *must* see are easy to
locate. At minimum, ISO/IEC 27001:2022 requires documented information for:

| Clause | Documented information |
| --- | --- |
| 4.3 | ISMS scope |
| 5.2 | Information security policy |
| 6.1.2 / 8.2 | Risk assessment process **and** results |
| 6.1.3 / 8.3 | Risk treatment process, Risk Treatment Plan, **and** results |
| 6.1.3 d | Statement of Applicability (SoA) |
| 6.2 | Information security objectives |
| 7.2 | Evidence of competence |
| 8.1 | Evidence that operational processes were carried out as planned |
| 9.1 | Monitoring and measurement results |
| 9.2 | Internal audit programme **and** results |
| 9.3 | Management review results |
| 10.2 | Nonconformities and corrective actions |

> This list is the practical baseline most implementers work from; confirm the
> exact requirements against your copy of the published standard, since the
> wording is what an auditor assesses against.
