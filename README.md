# pci-dss-verify

A Claude Code skill that audits a codebase against **PCI DSS v4.0.1** and emits a structured, machine-readable report.

It is a source-level review driven by the standard's checklists — not a scanner, not a certification tool. The scope of what it can and cannot establish is spelled out below.

---

## What it does

On `/pci-dss-verify` the skill:

1. Resolves the **entity type** (`merchant` / `provider` / `issuer`) and **level** (1–4), from arguments or by asking
2. Resolves which **areas** to audit: code, network, data, logs
3. Walks the repository against the PCI DSS v4.0.1 checklists that apply to that entity type
4. Reports each finding with file, line, requirement number, severity, the offending snippet, why it fails, and a concrete fix
5. Optionally persists the report to `.claude/pci-dss/YYYY-MM-DD.json` for follow-up work

Concretely, it reads and reasons over:

| Area | What it inspects |
|------|------------------|
| **Code** | Payment handlers, storage and encryption paths, SAD/PAN handling, masking, secrets in source, auth and access-control logic, dependency manifests and lock files |
| **Network** | `docker-compose.yml`, nginx/traefik/Caddy configs, published ports, TLS versions and cipher suites, segmentation, Terraform/Pulumi/CDK security groups |
| **Data** | Schema and ORM models, DB connection strings and roles/grants, at-rest and in-transit encryption settings, seeds and fixtures containing live PANs |
| **Logs** | What the logging calls emit near payment logic, PAN/CVV leakage, retention and rotation config, audit-event coverage under Req 10.2 |

Entity type changes the checklist set: `provider` adds the 11 SP-only requirements, `issuer` applies the Req 3.3.3 SAD exception. Level changes the evidentiary bar, not the technical requirements — at Level 1 each finding is annotated with the documented evidence a QSA will ask for.

## What it does not do

- **It is not a compliance certification.** It produces no ROC, no SAQ, no Attestation of Compliance. Those require a QSA or a formal self-assessment.
- **It does not scan running systems.** No ASV scan, no port scan, no pentest, no traffic capture. Everything is inferred from files in the repository.
- **It only sees what is in the repo.** Infrastructure configured outside version control — cloud console settings, hand-edited server configs, KMS policies, SIEM setup — is invisible to it, and their absence from the report is not evidence of compliance.
- **It cannot verify organizational requirements.** Most of Req 9 (physical access) and Req 12 (policy, training, incident response) are process controls. The skill can flag that a document is missing from the repo; it cannot confirm that a control is actually operating.
- **It cannot confirm runtime behavior.** Whether MFA is actually enforced, whether logs really reach a centralized server, whether backups are really encrypted — the skill reports what the configuration claims.
- **Findings are heuristic.** Detection leans on pattern matching plus reading the surrounding code, so both false positives (a variable named `pin` that is not a PIN) and false negatives (obfuscated or dynamically constructed paths) are possible. Treat the output as reviewed input to a human audit, not as a verdict.
- **It does not modify your code.** The report is read-only output; remediation is a separate, explicit step you drive.

---

## Installation

The skill installs by cloning this repository into a `skills` directory inside your Claude configuration.

**Global install** (available in every project):

```bash
# macOS / Linux
git clone https://github.com/zxc0zxc0zxc/claude-pci-dss-verify.git \
  ~/.claude/skills/pci-dss-verify

# Windows (PowerShell)
git clone https://github.com/zxc0zxc0zxc/claude-pci-dss-verify.git `
  "$env:USERPROFILE\.claude\skills\pci-dss-verify"
```

**Project-level install** (current repository only):

```bash
git clone https://github.com/zxc0zxc0zxc/claude-pci-dss-verify.git \
  .claude/skills/pci-dss-verify
```

Restart Claude Code after installing — the skill then responds to `/pci-dss-verify`.

**Updating:**

```bash
# macOS / Linux
cd ~/.claude/skills/pci-dss-verify && git pull

# Windows (PowerShell)
cd "$env:USERPROFILE\.claude\skills\pci-dss-verify"; git pull
```

### Claude.ai (web app)

1. Open **Settings → Claude Code → Skills**
2. Click **Add skill from Git**
3. Paste the repository URL: `https://github.com/zxc0zxc0zxc/claude-pci-dss-verify.git`
4. Click **Install**

> For a private repository you will need a Personal Access Token with `repo` scope.

---

## Usage

```
/pci-dss-verify [entity_type] [level]
```

| Argument | Accepted values | Default |
|----------|-----------------|---------|
| `entity_type` | `merchant`, `provider`, `issuer` (aliases: `mp`, `sp`, `service_provider`) | `merchant` |
| `level` | `1`, `2`, `3`, `4` — merchant; `1`, `2` — provider/issuer | `2` |

Anything omitted is asked for interactively.

```
# Merchant, level 2 (the most common case)
/pci-dss-verify merchant 2

# Payment gateway, level 1 (QSA required)
/pci-dss-verify provider 1

# Issuer (bank)
/pci-dss-verify issuer 1

# Aliases work too
/pci-dss-verify sp 2
/pci-dss-verify mp 1
```

After the parameters are settled, the skill offers the audit areas:

```
Which areas should be audited?
  ☑ Code (backend)  — storage, encryption, middleware, secrets
  ☑ Network         — docker networks, ports, TLS, segmentation
  ☑ Data            — database, access, isolation, backups
  ☑ Logs            — what is logged, retention, log protection
```

All areas are selected by default.

### Finding format

Each finding is emitted as:

````
### CVV is persisted to the database after authorization

**File:** `src/payments/store.ts:42`
**PCI DSS requirement:** 3.3.1.2
**Severity:** critical

```typescript
await db.cards.insert({ pan, cvv, expiry });
```

---

**Why it is non-compliant:** PCI DSS 3.3.1.2 absolutely prohibits storing
CVV/CVC/CAV2 after authorization — even encrypted.

**How to fix:** Drop the cvv field from the INSERT. CVV is used only within
the request to the payment gateway and is never persisted.
````

Severity maps to the standard's own weighting: `critical` for absolute prohibitions (SAD storage, plaintext PAN), `high` for exposed CDE components and privilege violations, `medium`/`low` for hardening and retention gaps.

---

## Working with reports

### Persisting

At the end of the audit the skill asks whether to save. On yes, it writes:

```
.claude/pci-dss/2026-05-23.json
```

Report structure (full JSON Schema in `templates/report-schema.json`):

```json
{
  "date": "2026-05-23",
  "entity_type": "merchant",
  "level": 2,
  "validation_method": "SAQ-D + quarterly ASV scan",
  "scope": ["code", "network", "data", "logs"],
  "summary": { "critical": 1, "high": 2, "medium": 1, "low": 0 },
  "findings": [
    {
      "id": "PCI-001",
      "area": "code",
      "requirement": "3.3.1.2",
      "severity": "critical",
      "title": "CVV is persisted to the database",
      "file": "src/payments/store.ts",
      "line": 42,
      "snippet": "...",
      "why": "...",
      "fix": "..."
    }
  ]
}
```

The report is stable JSON, so it can be diffed between runs, fed back into a later session, or consumed by other tooling.

### Remediation workflow

**1. Load the report into a later session:**

```
Here is the PCI DSS audit report: .claude/pci-dss/2026-05-23.json
Help me fix finding PCI-001
```

Claude reads the report, opens the referenced file, and proposes a concrete fix.

**2. Track progress:**

```
Show how many findings from .claude/pci-dss/2026-05-23.json are still open
```

**3. Re-audit after fixes:**

```
/pci-dss-verify merchant 2
```

A new JSON is written under today's date. Compare the two:

```
Diff .claude/pci-dss/2026-05-20.json against .claude/pci-dss/2026-05-23.json
Show which findings are closed and what remains
```

**4. Preparing for a QSA audit (Level 1):**

```
Using .claude/pci-dss/2026-05-23.json, build the list of documents
to prepare for the QSA for each finding
```

The `level1_note` field on each finding states the evidence type required for the ROC.

---

## Repository layout

```
.
├── SKILL.md                        # The skill itself: triggers, flow, checklists
├── README.md                       # This file
├── LICENSE                         # MIT
├── references/
│   ├── README.md                   # Navigating the standard (PDF page map)
│   └── levels.md                   # Entity types and levels: differences and validation
└── templates/
    ├── report-schema.json          # Report JSON Schema (for validation)
    └── report-example.json         # A filled-in example report
```

### Supplying the standard text

The skill reads PDF/MD documents from `references/` before drawing conclusions. To have it reason directly against the text of the standard:

```bash
# Copy the PDF into the references directory
cp ~/Downloads/PCI-DSS-v4_0_1.pdf .claude/skills/pci-dss-verify/references/
```

Without it, the skill relies on the checklists embedded in `SKILL.md`.

---

## Requirements

- **Claude Code** — a recent version with skills support
- **Git** for installing and updating
- Access to the project repository (the skill reads files from the current working directory)

No additional dependencies, package managers, or build step.

---

## License

MIT — see [LICENSE](LICENSE).
