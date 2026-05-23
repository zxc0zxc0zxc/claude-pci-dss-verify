# PCI DSS: entity types and compliance levels

> Source: payment system requirements (Visa, Mastercard) and PCI DSS v4.0.1.  
> Levels determine the **validation method**, not the technical requirements — the technical controls (Req 1–12) are the same for everyone.

---

## Entity types

### Merchant
Accepts card payments for its own goods or services. The primary subject of PCI DSS.  
**Key difference from an SP:** not required to meet the SP-only requirements.

### Service Provider
Stores, processes, or transmits CHD on behalf of other organizations.  
Examples: payment gateways, processing centers, hosting providers, MSPs, SaaS platforms with payment logic.  
**Key difference:** must meet 11 additional requirements (SP-only) and is held to a stricter validation regime.

### Issuer
A bank or financial institution that issues cards.  
**Key difference:** permitted to store SAD (Req 3.3.3) provided it is strongly encrypted — an exception available only to issuers.

---

## Merchant levels

| Level | Annual transaction volume | Validation method |
|-------|---------------------------|-------------------|
| **1** | >6M transactions on any single brand | Annual ROC (Report on Compliance) performed by a QSA + quarterly ASV scan + annual pentest |
| **2** | 1–6M transactions | Annual SAQ (Self-Assessment Questionnaire, type D for merchants) + quarterly ASV scan + annual pentest |
| **3** | 20,000–1M e-commerce transactions | Annual SAQ + quarterly ASV scan |
| **4** | <20,000 e-commerce, or any merchant under 1M total | Annual SAQ (recommended) + quarterly ASV scan |

**ASV** — Approved Scanning Vendor, external perimeter scanning.  
**QSA** — Qualified Security Assessor, an accredited auditor.  
**SAQ** — self-assessment; the type depends on how payments are accepted (A, B, C, D and others).

## Service Provider levels

| Level | Annual transaction volume | Validation method |
|-------|---------------------------|-------------------|
| **1** | >300,000 transactions | Annual ROC (QSA) + quarterly ASV scan + annual pentest |
| **2** | ≤300,000 transactions | Annual SAQ (type D for SPs) + quarterly ASV scan |

---

## What changes with the level

The technical requirements (Req 1–12) **do not change**. The level affects:

| What | Level 1 | Levels 2–4 |
|------|---------|------------|
| Assessment | An on-site ROC performed by a QSA | Self-assessment (SAQ) |
| External scanning | Quarterly (ASV) | Quarterly (ASV) |
| Pentest | Annually + after significant changes | Annually (recommended at Level 4) |
| Audit rigor | Maximum — documented evidence for every item | Proportional to risk |
| SP quarterly reviews | Req 12.4.2 — mandatory | Not applicable to merchants |

---

## SP-only requirements (v4.0.1)

On top of the standard Req 1–12, service providers must also meet:

| Req | Substance |
|-----|-----------|
| **8.3.10** | Customer/user passwords must be rotated at least every 90 days, or dynamic account activity analysis must be applied |
| **10.7.1** | Failures of critical security systems must be detected and reported in real time (a best practice until 2025-03-31; mandatory afterwards via 10.7.2) |
| **11.4.6** | Penetration testing must cover all components (internal and external) and be performed by a qualified specialist |
| **11.4.7** | Multi-tenant SPs must support external penetration testing of the infrastructure by their customers (on request) |
| **11.5.1.1** | IDS/IPS must detect and prevent covert data-exfiltration channels |
| **12.4.1** | Executive management formally assigns responsibility for PCI DSS |
| **12.4.2** | Quarterly reviews confirming that requirements are being performed (personnel task review) |
| **12.4.2.1** | Quarterly review results documented and signed off by management |
| **12.5.2.1** | Scope confirmation performed at least every 6 months (annually for merchants) |
| **12.5.3** | Significant organizational changes → mandatory analysis of the impact on PCI DSS scope |

---

## Issuer-only exceptions (v4.0.1)

| Req | Substance |
|-----|-----------|
| **3.3.3** | Issuers **may** store SAD (track data, CVV, PIN) provided that: (1) the business need is documented, (2) the data is encrypted with strong cryptography, (3) it is retained only for the minimum necessary time |

Every other requirement applies to issuers exactly as it does to merchants.

---

## Using this in an audit

1. **Merchant** → run the standard checklists (code, network, data, logs). At Level 1, flag every finding with the documented evidence required.
2. **Service Provider** → standard checklists plus the SP-only block. At Level 1, additionally check the quarterly review process (12.4.2) and scope confirmation (12.5.2.1).
3. **Issuer** → standard checklists, plus SAD may be stored under 3.3.3 — but verify the encryption and the documented business justification.
