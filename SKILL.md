---
name: pci-dss-verify
description: Audits a codebase for PCI DSS v4.0.1 compliance. Supports entity types (merchant/service_provider/issuer) and levels (1–4). Checks backend code, network configuration, data storage and logging. Triggers on /pci-dss-verify, /pci-dss-verify provider 2, /pci-dss-verify issuer 1, or when the user asks for a PCI DSS / card data compliance audit.
---

# PCI DSS Verify

This skill audits a project against PCI DSS v4.0.1 requirements, taking the entity type and compliance level into account.

## When to use

- The user invoked `/pci-dss-verify` (with or without arguments)
- The user asks for a PCI DSS compliance audit
- The user asks whether their application handles payment card data safely

## Argument syntax

```
/pci-dss-verify [entity_type] [level]
```

- `entity_type`: `merchant` | `provider` | `issuer`  
  (aliases: `service_provider` = `provider`, `mp` = `merchant`, `sp` = `provider`)
- `level`: `1` | `2` | `3` | `4` (for merchant), `1` | `2` (for provider/issuer)

Examples:
- `/pci-dss-verify` — no arguments, ask the questions
- `/pci-dss-verify provider 1` — service provider, level 1
- `/pci-dss-verify merchant 2` — merchant, level 2 (default level)
- `/pci-dss-verify issuer` — issuer, ask for the level

## Flow

### Step 0. Determine entity type and level

If arguments were passed — use them. Otherwise ask via `AskUserQuestion`.

**Question 1 — entity type** (if not specified):

```
What is your organization from a PCI DSS perspective?

• Merchant — you accept card payments for your own goods/services
• Service Provider — you process/store CHD on behalf of others (gateway, processing, SaaS)
• Issuer — a bank or organization that issues payment cards
```

**Question 2 — level** (if not specified; ask only after the type):

For **merchant**:
```
Which PCI DSS compliance level?

• Level 1 — >6M transactions/year (ROC required, QSA needed)
• Level 2 — 1–6M transactions/year (SAQ + pentest) [recommended default]
• Level 3 — 20,000–1M e-commerce transactions
• Level 4 — <20,000 e-commerce, or any merchant under 1M total
```

For **provider**:
```
Which PCI DSS level for the Service Provider?

• Level 1 — >300,000 transactions/year (ROC required)
• Level 2 — ≤300,000 transactions/year (SAQ) [recommended default]
```

For **issuer**:
```
Issuer levels are not standardized by the payment systems the way merchant levels are.
The audit will run against the full set of requirements (equivalent to Level 1).
```

> **Default:** if the user makes no choice — `merchant`, level `2`.

Once the answers are in, print a short summary:
```
Audit: [entity_type] · Level [N]
Validation method: [ROC/SAQ-D/etc.]
SP-only requirements: [applicable / not applicable]
Issuer exceptions: [applicable / not applicable]
```

More detail — `references/levels.md`.

### Step 1. Determine the audit scope

Ask the user via `AskUserQuestion` which areas to check. Possible areas:

1. **Code (backend)** — middleware, payment data handling, storage, encryption, secrets, input validation
2. **Network** — docker networks, open/closed ports, nginx/traefik/reverse-proxy configs, TLS, segmentation
3. **Data** — database type, how the backend connects to it, isolation from external networks, storage of cards/CVV/PAN, users and privileges
4. **Logs** — what is logged, where it is stored, whether PAN/CVV/track data leaks into logs, rotation, access to logs

Let the user pick one area, several, or all (`multiSelect: true`). Recommend "All" by default.

### Step 2. Run the audit

For each selected area, walk the codebase in a targeted way. Use parallel `Grep`/`Glob`/`Read` where possible. For broad searches, delegate to `Agent` (subagent_type=Explore).

**What to look for in each area** — see the "Checklists" section below.

Apply the checklists according to the entity type:
- All entities → run the **general checklists**
- `provider` → additionally the **SP-only checklist**
- `issuer` → apply the **issuer modifications** (SAD may be stored under 3.3.3)
- Level 1 (any type) → require `documented evidence` for every finding; add the note `⚠ Level 1: documented evidence required for the ROC`

If the project has a `references/` directory next to SKILL.md, it holds the current standard requirements and the level descriptions (`levels.md`) — read them before drawing conclusions.

### Step 3. Produce the report

For each problem found, print a block in this format:

````
### [Short problem title]

**File:** `path/to/file.ext:NN`
**PCI DSS requirement:** [requirement number, e.g. 3.4 / 4.1 / 8.2]
**Severity:** critical | high | medium | low

```language
[code or config snippet, verbatim]
```

---

**Why it is non-compliant:** [explanation of which clause of the standard is violated and why it is a risk]

**How to fix:** [a concrete remediation — no filler, with a code/config example where relevant]
````

Close the report with a summary: how many problems by severity, which areas were covered, which were skipped.

### Step 4. Ask about saving

Via `AskUserQuestion`, ask: "Save the audit for later analysis and remediation?" with Yes/No options.

### Step 5. Save (if Yes)

Save the report to `.claude/pci-dss/{YYYY-MM-DD}.json` following the schema (see `templates/report-schema.json`).  
Include the `entity_type` and `level` fields at the root of the object:

```json
{
  "date": "2026-05-23",
  "entity_type": "merchant",
  "level": 2,
  "validation_method": "SAQ-D + quarterly ASV scan",
  "scope": ["code", "network", "data", "logs"],
  "summary": {
    "critical": 0,
    "high": 0,
    "medium": 0,
    "low": 0
  },
  "findings": [
    {
      "id": "PCI-001",
      "area": "code",
      "requirement": "3.5.1",
      "severity": "critical",
      "title": "...",
      "file": "src/payments/store.ts",
      "line": 42,
      "snippet": "...",
      "why": "...",
      "fix": "...",
      "level1_note": null
    }
  ]
}
```

The `level1_note` field is filled in only for Level 1 — a string such as `"Documented evidence required for the QSA ROC"`, or `null`.

If the `.claude/pci-dss/` directory does not exist — create it. If a file for today already exists — append a `-N` suffix to the name.

---

## Checklists (PCI DSS v4.0.1)

> Source: PCI DSS Requirements and Testing Procedures, v4.0.1, June 2024.  
> Cite the specific requirement number for every finding.

### Code (backend)

**SAD storage (Req 3.3)**
- [ ] **3.3.1.1** — full track data is not stored anywhere after authorization. Grep: `track`, `track1`, `track2`, `magnetic`
- [ ] **3.3.1.2** — CVV/CVC/CAV2/CID is not stored anywhere after authorization. Grep: `cvv`, `cvc`, `cav`, `card_verif`
- [ ] **3.3.1.3** — PIN and PIN block are not stored. Grep: `pin`, `pin_block`

**Protecting stored PAN (Req 3.4 / 3.5)**
- [ ] **3.4.1** — when PAN is displayed on screen, written to logs, or returned by an API, it is masked to `BIN******LAST4`. Grep: output of `pan`, `card_number`, `cardNumber` without masking
- [ ] **3.5.1** — PAN is stored unreadable in the database: strong-crypto encryption, truncation, or a keyed hash (HMAC-SHA256+). Plaintext storage is not acceptable
- [ ] **3.5.1.1** — if a PAN hash is used, it must be a keyed cryptographic hash (HMAC), not a plain SHA/MD5
- [ ] **3.6 / 3.7** — encryption keys are not hardcoded, are not stored alongside the data, and are managed through a KMS/HSM or equivalent

**Encryption in transit (Req 4.2)**
- [ ] **4.2.1** — all PAN transmissions use strong cryptography protocols only. SSL, TLS 1.0, TLS 1.1 are prohibited. TLS 1.2 minimum, TLS 1.3 recommended. Grep nginx/traefik/app configs for `ssl_protocols`, `TLSv1.0`, `TLSv1.1`
- [ ] **4.2.1** — there is no fallback to insecure versions (no-fallback). Grep: `SSLv2`, `SSLv3`, `TLSv1\b`, `TLSv1.1`
- [ ] **4.2.2** — PAN is not transmitted over unprotected channels (email, messaging, HTTP). Grep: card data sent over HTTP/without TLS

**Secure development (Req 6.2 / 6.3 / 6.4)**
- [ ] **6.2.4** — protection against the OWASP Top 10: injection, XSS, broken auth, insecure deserialization — look inside the payment controllers
- [ ] **6.3.3** — dependencies are protected against known vulnerabilities. Check lock files, the date of the last update, and whether `npm audit` / `pip-audit` / Dependabot is in place
- [ ] **6.4.2** — public-facing web applications with payment forms must have a WAF (application firewall) or equivalent
- [ ] **6.5.5** — pre-production environments (dev, staging) must not contain live PANs. Grep test fixtures and seed files

**Authentication and access control (Req 8.2 / 8.3 / 8.4)**
- [ ] **8.2.1** — every user has a unique ID; there are no shared/generic accounts such as `admin`, `app`, `service`
- [ ] **8.2.2** — group/shared accounts are prohibited for CDE access
- [ ] **8.3.1** — all users authenticate with at least one of: password, token, certificate
- [ ] **8.3.6** — passwords are ≥12 characters (or ≥8 if the system cannot do more), containing both numbers and letters
- [ ] **8.4.1** — MFA for all non-console administrative access to the CDE (SSH, admin panel, admin API)
- [ ] **8.4.3** — MFA for all remote access into the CDE from outside the perimeter

**Secrets in code**
- [ ] No hardcoded secrets: `Grep` for `password\s*=\s*['"]`, `secret\s*=\s*['"]`, `api[_-]?key\s*=\s*['"]`, `private[_-]?key`
- [ ] `.env` files are not committed to git. Check `.gitignore`

**Configurations (Req 2.2)**
- [ ] **2.2.2** — all vendor default passwords have been changed (postgres/postgres, admin/admin, root/root and the like)
- [ ] **2.2.4** — only necessary services and ports are enabled; the rest are disabled

### Network

**Restricting access to the CDE (Req 1.3 / 1.4)**
- [ ] **1.3.1** — inbound traffic into the CDE is restricted to what is genuinely necessary. Grep `docker-compose.yml` and `nginx.conf` for open ports
- [ ] **1.3.2** — outbound traffic from the CDE is restricted to authorized destinations
- [ ] **1.4.4** — components that store CHD (databases, caches holding CHD) **must not** be directly reachable from public networks — no direct `ports:` mapping to the host
- [ ] **1.4.5** — internal IPs and architecture are not disclosed externally (no debug headers, no verbose error pages)

**Segmentation**
- [ ] Is there a dedicated docker network (or VLAN) for the payment perimeter, separate from other services
- [ ] Database containers do not publish ports externally (`ports:` on a db service is a critical finding). Only `expose:` inside the network
- [ ] Exposed admin panels (phpmyadmin, adminer, redis-commander, mongo-express) are absent from the production config or are closed off from the internet

**TLS and reverse proxy (Req 4.2.1)**
- [ ] Nginx/Traefik/Caddy config: `ssl_protocols TLSv1.2 TLSv1.3` — no TLSv1.0/1.1/SSLv
- [ ] Cipher suites — no weak ones (RC4, DES, 3DES, EXPORT, NULL)
- [ ] HSTS is enabled (`Strict-Transport-Security`)

**IaC (if present)**
- [ ] Terraform/Pulumi/CDK — security groups do not open 0.0.0.0/0 on database ports (5432, 3306, 27017, 6379)
- [ ] **1.5.1** — devices that connect both to the internet and to the CDE have security controls (VPN, endpoint protection)

### Data

**What is stored and how (Req 3.3 / 3.5)**
- [ ] **3.3.1.1–3.3.1.3** — SAD (track data, CVV, PIN) is not stored anywhere — not in the database, not in logs, not in caches
- [ ] **3.5.1** — PAN in the database is encrypted or truncated; check the table schema and the ORM models
- [ ] Database type and connection method: read `docker-compose.yml`, `.env.example`, and ORM configs (Prisma, SQLAlchemy, Sequelize, Hibernate)

**Network isolation of the database (Req 1.4.4)**
- [ ] The database listens only on the internal docker network, not on `0.0.0.0`
- [ ] No `ports:` on the db service — only `expose:`
- [ ] The connection string does not contain `localhost` in the production config (a sign of a monolith without segmentation)

**Database access (Req 7.2 / 8.2)**
- [ ] **7.2.2** — the application connects to the database under a dedicated least-privilege role (only SELECT/INSERT/UPDATE on the tables it needs), not as `root`/`postgres`/`admin`
- [ ] **7.3.3** — default-deny: no wildcard grants such as `GRANT ALL ON *.*`
- [ ] Separate roles for app, migrations, and read-only (reporting) — the least privilege principle

**Data encryption (Req 3.5 / 3.6)**
- [ ] At-rest encryption is enabled at the database level (TDE, pgcrypto, encrypted volumes), or the data is encrypted at the application level
- [ ] In-transit encryption: TLS between the backend and the database (`sslmode=require` for Postgres, `ssl: true` for MySQL/Mongo)
- [ ] **3.6.1.2** — encryption keys are not stored alongside the encrypted data; a KMS or a separate secret manager is used

**Backups**
- [ ] Backups are stored separately and encrypted
- [ ] Access to backups is restricted; there is no public, unencrypted S3 bucket

**Test data (Req 6.5.5)**
- [ ] No real PANs in seeds/fixtures/tests (grep `\b4[0-9]{12}(?:[0-9]{3})?\b` — Visa format)

### Logs

**Which events must be logged (Req 10.2.1)**
- [ ] **10.2.1.1** — every user access to cardholder data (CHD) is logged
- [ ] **10.2.1.2** — all actions taken with administrative access (including interactive use of app/system accounts)
- [ ] **10.2.1.3** — all access to the audit logs themselves
- [ ] **10.2.1.4** — all failed login attempts
- [ ] **10.2.1.5** — all credential changes (passwords, keys, tokens)
- [ ] **10.2.1.6** — initialization, stopping, and clearing of audit logs
- [ ] **10.2.1.7** — creation and deletion of system-level objects

**What every log record must contain (Req 10.2.2)**
- [ ] User identification (who)
- [ ] Event type (what)
- [ ] Date and time (when)
- [ ] Success or failure
- [ ] Origin of the event (where from — IP, device ID)
- [ ] The component/resource that was accessed (what was touched)

**What must NOT reach the logs (Req 3.3 / 3.4.1)**
- [ ] **3.4.1** — PAN is not logged in the clear; if it appears, it must be masked to `BIN******LAST4`
- [ ] **3.3.1.2** — CVV/CVC/track data does not reach the logs. Grep: `console.log`, `logger.*`, `print(` near payment logic

**Log storage and protection (Req 10.3 / 10.5)**
- [ ] **10.3.1** — read access to audit logs is limited to authorized individuals
- [ ] **10.3.2** — logs are protected from modification (append-only, write-once, centralized log server)
- [ ] **10.3.3** — logs from external-facing services (reverse proxy, WAF) are backed up to a secure centralized server
- [ ] **10.5.1** — logs are retained for ≥12 months, of which ≥3 months are "online" (immediately available for analysis)

**Time synchronization (Req 10.6)**
- [ ] **10.6.1** — all systems are synchronized via NTP against an authorized source
- [ ] **10.6.2** — NTP configs are correct and protected from modification

---

### SP-only checklist (entity_type = provider only)

> Apply **in addition** to the general checklists. Every item is a PCI DSS v4.0.1 requirement that applies exclusively to service providers.

**Authentication (Req 8.3.10)**
- [ ] **8.3.10** — user passwords are changed at least every 90 days, or dynamic account behavior analysis (risk-based authentication) is in place. Look at auth service configs and password policies
- [ ] **8.3.10.1** — if a password is the only factor for customer access, a 90-day rotation is mandatory

**Failure monitoring (Req 10.7.1)**
- [ ] **10.7.1** — alerts are configured for failures of critical security systems (IDS, FIM, auth service, log pipeline) in real time. Check the monitoring config (alertmanager, PagerDuty, etc.)

**Penetration testing (Req 11.4.6)**
- [ ] **11.4.6** — the pentest covers all network layers and components (not just the perimeter). Is there a pentest scope document covering internal + external?
- [ ] **11.4.7** (for multi-tenant SPs) — customers can request an external pentest of their portion of the infrastructure. Is there a procedure/policy for this?

**IDS/IPS (Req 11.5.1.1)**
- [ ] **11.5.1.1** — IDS/IPS is configured to detect covert data-exfiltration channels (covert channels, DNS tunnelling, steganography in HTTP). Check the Snort/Suricata/Zeek/cloud WAF config

**Managing the PCI DSS program (Req 12.4)**
- [ ] **12.4.1** — executive management is formally assigned responsibility for PCI DSS. Is there a documented assignment?
- [ ] **12.4.2** — quarterly reviews confirming that personnel are performing their security tasks (task compliance reviews)
- [ ] **12.4.2.1** — quarterly review results are documented and signed off by management

**Scope management (Req 12.5.2.1 / 12.5.3)**
- [ ] **12.5.2.1** — the PCI DSS scope is confirmed at least every 6 months (annually for merchants). Is there a recent scope document (less than 6 months old)?
- [ ] **12.5.3** — any significant organizational/architectural change → a mandatory impact assessment on the PCI DSS scope. Is there a procedure for this?

---

### Issuer modifications (entity_type = issuer only)

> Issuers have **one exception** to the general rules. Everything else applies in full.

**SAD storage (Req 3.3.3) — an exception for issuers only**
- [ ] **3.3.3** — SAD (track data, CVV, PIN) **may be stored** only if:
  1. There is a documented business justification
  2. The SAD is encrypted with strong cryptography (AES-256 or equivalent)
  3. The data is retained only for the minimum necessary time

  If SAD is stored without all three conditions met, it is a violation even for an issuer.

> For every other entity type, storing SAD after authorization is **absolutely prohibited** (Req 3.3.1.1–3.3.1.3).

---

## Behavior

- If some areas do not exist in the project (e.g. no docker) — skip the network section and mark it "not applicable"
- If nothing critical was found — say so; do not invent problems
- Cite the file and line for every problem — the user must be able to click through to the exact spot
- When saving JSON, do not duplicate full snippets in `summary`; keep them only in `findings[].snippet`
- **Level 1:** add a `level1_note` to every finding — what specifically needs to be documented for the QSA (evidence type: policy doc, screenshot, log sample, etc.)
- Read `references/levels.md` when you need to clarify the difference between entity types or levels
