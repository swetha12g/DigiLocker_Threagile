# DigiLocker Threat Model Report

**Prepared by:** Gunda Swetha, Sravani Avadurthi, Nikhil Kumar, Amit Kumar Saini, Sudheer Kumar Thammana
**Date:** 01 Aug 2026
**Assignment:** Assignment 6 · Main Course · IIT Roorkee
**Engine:** [Threagile](https://threagile.io) v1.0.0 (Docker) · Model: [`digilocker.yaml`](digilocker.yaml)

---

## 1. Overview

DigiLocker is a Government of India digital document wallet that enables citizens to securely store, access, and share official digital documents. This report is an architecture risk assessment (threat model) of DigiLocker, built two ways:

1. **Structural / STRIDE analysis** with [Threagile](https://threagile.io), an open-source, YAML-driven agile threat-modeling toolkit — covers every technical asset and data flow in the architecture.
2. **Formal cryptographic-protocol verification** with the [Tamarin Prover](https://tamarin-prover.github.io/) — covers the message-logic correctness of the citizen authentication flow specifically, which Threagile can only check structurally.

## 2. How This Model Was Built

| Step | Description |
|---|---|
| 1. Define architecture model | [`digilocker.yaml`](digilocker.yaml) — 8 data assets (CIA-rated), 13 technical assets, 15 communication links, 3 trust boundaries. |
| 2. Containerized execution | `docker run -v <project>:/app/work threagile/threagile -model digilocker.yaml -output output` |
| 3. Automated risk engine | Threagile's rule engine evaluates every asset & data flow against ~40 STRIDE-based risk rules. |
| 4. Generated artifacts | `report.pdf`, `data-flow-diagram.png`, `data-asset-diagram.png`, `risks.json` / `risks.xlsx`, `stats.json`. |

## 3. Architecture & Trust Boundaries

![DigiLocker Architecture](digilocker-architecture-enhanced.svg)

*Full-size vector diagram: [`digilocker-architecture-enhanced.svg`](digilocker-architecture-enhanced.svg) · print version: [`digilocker-architecture-enhanced.pdf`](digilocker-architecture-enhanced.pdf)*

Every component is tagged with a DFD element type — **AC** = actor-driven client process, **P** = process, **DS** = data store, **EE** = external entity (3rd-party, out of trust boundary) — and every flow is numbered (1–15), matching the [Data-Flow Register](#5-data-flow-register) below.

**Trust boundaries** (from `digilocker.yaml`):

| Trust Boundary | Type | Technical Assets Inside |
|---|---|---|
| DMZ | network-cloud-security-group | API Gateway, Notification Service |
| Application Zone | network-virtual-lan | Authentication Service, DigiLocker Backend, Consent Manager, Monitoring Service |
| Data Zone | network-virtual-lan | Document Repository, Relational Database |
| *(out of trust boundary)* | external systems | Aadhaar Identity Provider (UIDAI), Issuer Service (Government), Requester Service (3rd-party) |

## 4. Component Register

Role, responsibilities, and privileges for every technical asset. (Standalone print sheet: [`digilocker-component-register.svg`](digilocker-component-register.svg) / [`.pdf`](digilocker-component-register.pdf))

| ID | Component (technology) | Type | Role | Responsibilities | Privileges |
|---|---|---|---|---|---|
| C-01 | Citizen Web Portal (`web-application`) | AC | Actor-driven client / Citizen-User interface | Renders the citizen-facing web UI; initiates login and consent/document requests; holds the citizen's browser session. | Can submit authentication requests and act on behalf of the logged-in citizen (end-user identity propagation) via the API Gateway. No direct access to any datastore or backend service. No administrative privileges. |
| C-02 | Mobile App (`mobile-app`) | AC | Actor-driven client / Citizen-User interface | Native mobile UI for citizens; initiates token-based login; requests citizen profile and documents. | Same end-user identity-propagation privileges as the Web Portal via the API Gateway. No direct backend or datastore access. |
| C-03 | API Gateway (`gateway`) | P | API Gateway / Edge router (DMZ) | Single internet-facing entry point; routes citizen and mobile requests to the Authentication Service and DigiLocker Backend; terminates the public HTTPS connection. | Forwards requests carrying the end-user's propagated identity; invokes the technical-user identity for token validation. No direct datastore access. |
| C-04 | Authentication Service (`identity-provider`) | P | SSO / Identity Provider | Authenticates citizens (Aadhaar OTP-based login — see [`digilocker-auth.spthy`](digilocker-auth.spthy) formal model); issues and validates authentication/session tokens; integrates with UIDAI for eKYC/OTP verification. | Generates authentication-tokens; reads/writes authentication-tokens in the Relational Database using technical-user credentials; calls the external Aadhaar Identity Provider using service credentials. |
| C-05 | Aadhaar Identity Provider (`identity-provider`, external) | EE | External Identity Provider (UIDAI / Government, out of DigiLocker trust boundary) | Performs Aadhaar eKYC / OTP verification for citizen identity on request from the Authentication Service. | Provides identity-verification results only. Owned and operated by UIDAI, not DigiLocker; treated as an untrusted external boundary requiring credentialed technical-user authentication. |
| C-06 | DigiLocker Backend (`application-server`) | P | DigiLocker Platform / core application server | Core business logic for citizen profile, issued/uploaded documents and consent workflows; orchestrates the Document Repository, Relational Database, Consent Manager, Issuer Service, Requester Service, Notification Service and Monitoring Service. | Full read/write access to citizen-profile, issued-documents, uploaded-documents and consent-records via the Document Repository and Database (technical-user credentials). Read-only access to the Issuer Service. Can push verified documents to the Requester Service and emit audit-logs to the Monitoring Service. |
| C-07 | Document Repository (`database`) | DS | Repository / Data store | Stores and serves issued and uploaded documents on behalf of the Backend. | Storage only — no outbound calls. Data encrypted at rest (symmetric shared key, per `digilocker.yaml`). Accessible only to the DigiLocker Backend, over encrypted JDBC. |
| C-08 | Consent Manager (`application-server`) | P | Consent Manager | Manages and validates citizen consent for sharing documents with requesting parties; persists consent-records on behalf of the Backend. | Read/write consent-records in the Relational Database using technical-user credentials. No citizen-facing exposure and no access to document content. |
| C-09 | Issuer Service (`web-service-rest`, external) | EE | Issuer (Government, out of DigiLocker trust boundary) | Issues government certificates/documents and their digital signatures, retrieved by the Backend. | Supplies issued-documents and digital-signatures to the Backend (read-only from DigiLocker's perspective). External system owned by the Government issuer. |
| C-10 | Requester Service (`web-service-rest`, external) | EE | Requester (3rd-party, out of DigiLocker trust boundary) | Requests / consumes citizen documents that DigiLocker shares after consent validation. | Receives issued-documents pushed by the Backend post-consent. Cannot pull data directly or query DigiLocker datastores; external, untrusted boundary. |
| C-11 | Notification Service (`message-queue`) | P | Notification Service (DMZ) | Sends email/SMS notifications to citizens (e.g. document-shared alerts) on behalf of the Backend. | Receives citizen-profile contact data from the Backend. Outbound-only to citizen communication channels; no datastore access modeled. |
| C-12 | Monitoring Service (`monitoring`) | P | Monitoring / Audit Service | Collects audit and security events emitted by the Backend; persists audit-logs for security investigation. | Write access to audit-logs in the Relational Database using technical-user credentials. No access to citizen profile, document or consent data. |
| C-13 | Relational Database (`database`) | DS | Repository / Data store | Persists citizen-profile, consent-records, authentication-tokens and audit-logs for the platform. | Storage only. Data encrypted at rest (symmetric shared key). Accessed only by the Authentication Service, Backend, Consent Manager and Monitoring Service, each via encrypted JDBC with technical-user credentials. |

## 5. Data-Flow Register

Protocol, authentication, encryption, data exchanged, and security controls for every communication link. (Standalone print sheet: [`digilocker-dataflow-register.svg`](digilocker-dataflow-register.svg) / [`.pdf`](digilocker-dataflow-register.pdf))

`⚠ TBV` = scheme not specified in the Threagile model — **To Be Verified** against the live implementation.

| # | Source → Destination | Protocol | Auth. | Encryption | Direction | Data Exchanged | Security Controls |
|---|---|---|---|---|---|---|---|
| 1 | C-01 Citizen Web Portal → C-03 API Gateway | HTTPS | Session ID (cookie) ⚠ TBV | TLS in transit | Bidirectional | Sent: authentication-tokens. Recv: citizen-profile, authentication-tokens. | Encryption in transit (TLS). Authorization: end-user identity propagation. |
| 2 | C-02 Mobile App → C-03 API Gateway | HTTPS | Bearer token ⚠ TBV | TLS in transit | Bidirectional | Sent: authentication-tokens. Recv: citizen-profile, authentication-tokens. | Encryption in transit (TLS). Authorization: end-user identity propagation. Token scheme (e.g. JWT/OAuth2) not confirmed by model. |
| 3 | C-03 API Gateway → C-04 Authentication Service | HTTPS | Bearer token ⚠ TBV | TLS in transit | Bidirectional | Sent: authentication-tokens. Recv: authentication-tokens. | Encryption in transit (TLS). Authorization: technical-user (service account). Used for token validation on every gateway request. |
| 4 | C-03 API Gateway → C-06 DigiLocker Backend | HTTPS | Bearer token ⚠ TBV | TLS in transit | Bidirectional | Sent: citizen-profile, authentication-tokens. Recv: citizen-profile, issued-documents, uploaded-documents, consent-records. | Encryption in transit (TLS). Authorization: end-user identity propagation — Backend acts on behalf of the authenticated citizen. |
| 5 | C-04 Authentication Service → C-05 Aadhaar Identity Provider (UIDAI) | HTTPS | Credentials (technical) | TLS in transit | Bidirectional | Sent: aadhaar-data. Recv: aadhaar-data. | Encryption in transit (TLS). Authorization: technical-user, pre-registered government API integration. **Crosses out of the DigiLocker trust boundary.** OTP protocol logic formally modelled in [`digilocker-auth.spthy`](digilocker-auth.spthy). |
| 6 | C-04 Authentication Service → C-13 Relational Database | JDBC (encrypted) | Credentials (technical) | Encrypted JDBC connection | Bidirectional | Sent: authentication-tokens. Recv: authentication-tokens. | Encrypted JDBC transport. Authorization: technical-user. Destination encrypted at rest (symmetric shared key). |
| 7 | C-06 DigiLocker Backend → C-07 Document Repository | JDBC (encrypted) | Credentials (technical) | Encrypted JDBC connection | Bidirectional | Sent: issued-documents, uploaded-documents. Recv: issued-documents, uploaded-documents. | Encrypted JDBC transport. Authorization: technical-user. Destination encrypted at rest. |
| 8 | C-06 DigiLocker Backend → C-13 Relational Database | JDBC (encrypted) | Credentials (technical) | Encrypted JDBC connection | Bidirectional | Sent: citizen-profile, consent-records. Recv: citizen-profile, consent-records. | Encrypted JDBC transport. Authorization: technical-user. Destination encrypted at rest. |
| 9 | C-06 DigiLocker Backend → C-08 Consent Manager | HTTPS | Bearer token ⚠ TBV | TLS in transit | Bidirectional | Sent: consent-records. Recv: consent-records. | Encryption in transit (TLS). Authorization: technical-user. Used to check/update citizen consent. |
| 10 | C-06 DigiLocker Backend → C-09 Issuer Service | HTTPS | Bearer token ⚠ TBV | TLS in transit | One-way (data flows Issuer → Backend on response) | Sent: none (read-only fetch). Recv: issued-documents, digital-signatures. | Encryption in transit (TLS). Authorization: technical-user. Marked read-only in the model. **Crosses out of the DigiLocker trust boundary.** |
| 11 | C-06 DigiLocker Backend → C-10 Requester Service | HTTPS | Bearer token ⚠ TBV | TLS in transit | One-way (Backend → Requester push) | Sent: issued-documents. Recv: none logged. | Encryption in transit (TLS). Authorization: technical-user. Shared only after consent validation. **Crosses out of the DigiLocker trust boundary.** |
| 12 | C-06 DigiLocker Backend → C-11 Notification Service | HTTPS | Bearer token ⚠ TBV | TLS in transit | One-way (Backend → Notification trigger) | Sent: citizen-profile (contact data). Recv: none logged. | Encryption in transit (TLS). Authorization: technical-user. Triggers citizen email/SMS alerts. |
| 13 | C-06 DigiLocker Backend → C-12 Monitoring Service | HTTPS | Bearer token ⚠ TBV | TLS in transit | One-way (Backend → Monitoring event) | Sent: audit-logs. Recv: none logged. | Encryption in transit (TLS). Authorization: technical-user. Emits audit/security events for investigation. |
| 14 | C-08 Consent Manager → C-13 Relational Database | JDBC (encrypted) | Credentials (technical) | Encrypted JDBC connection | Bidirectional | Sent: consent-records. Recv: consent-records. | Encrypted JDBC transport. Authorization: technical-user. Destination encrypted at rest. |
| 15 | C-12 Monitoring Service → C-13 Relational Database | JDBC (encrypted) | Credentials (technical) | Encrypted JDBC connection | Bidirectional | Sent: audit-logs. Recv: audit-logs. | Encrypted JDBC transport. Authorization: technical-user. Destination encrypted at rest. |

## 6. Data Assets — CIA Classification

| Data Asset | Confidentiality | Integrity | Availability | Justification |
|---|---|---|---|---|
| Aadhaar Data | Strictly confidential | Mission-critical | Important | Sensitive citizen identity information. |
| Citizen Profile | Confidential | Critical | Important | Contains citizen personal information. |
| Issued Documents | Confidential | Mission-critical | Mission-critical | Official certificates and records. |
| Uploaded Documents | Confidential | Important | Important | Personal uploaded files. |
| Consent Records | Confidential | Mission-critical | Important | Legal consent information. |
| Authentication Tokens | Strictly confidential | Mission-critical | Important | Used for authentication. |
| Audit Logs | Internal | Mission-critical | Important | Security investigation records. |
| Digital Signatures | Confidential | Mission-critical | Important | Digital signatures for document verification. |

## 7. Risk Analysis Results

**74 risks** identified across 15 data flows ([`output/risks.json`](output/risks.json) / [`output/risks.xlsx`](output/risks.xlsx)):

| Severity | Count |
|---|---|
| Critical | 0 |
| High | 5 |
| Elevated | 33 |
| Medium | 25 |
| Low | 11 |

**Top risk categories:**

| Category | Count |
|---|---|
| DoS — risky access across trust boundary | 10 |
| Server-Side Request Forgery (SSRF) | 9 |
| Container base-image backdooring | 6 |
| Cross-Site Scripting (XSS) | 5 |
| SQL/NoSQL Injection | 5 |
| Unguarded direct datastore access | 5 |
| Missing WAF | 5 |
| Cross-Site Request Forgery (CSRF) | 4 |
| Unguarded access from internet | 4 |
| Unnecessary data transfer | 4 |

**All 5 High-severity findings** are SQL/NoSQL-Injection risks reachable over the JDBC-encrypted links into the two datastores — remediation is parameterized queries / prepared statements at each of these call sites:

- Authentication Service → Database (flow 6)
- Consent Manager → Database (flow 14)
- DigiLocker Backend → Database (flow 8)
- DigiLocker Backend → Document Repository (flow 7)
- Monitoring Service → Database (flow 15)

Full per-risk detail (all 74, with rule rationale) is in the Threagile-generated report — see [`output/report.pdf`](output/report.pdf), pages 2–144.

## 8. Formal Verification — Tamarin Prover

Threagile's STRIDE rules can only check the **structure** of the citizen-login flow (protocol: `https`, authentication: `token`/`credentials`). They cannot prove anything about the actual **message logic** of that flow. [`digilocker-auth.spthy`](digilocker-auth.spthy) closes that gap with a symbolic (Dolev-Yao) model of the Aadhaar OTP-based citizen authentication path:

```
citizen-web-portal / mobile-app → api-gateway → authentication-service
authentication-service → identity-provider (UIDAI, Aadhaar)
```

Four properties are modelled as lemmas:

| Lemma | Property |
|---|---|
| `executable` | Sanity check — the protocol run is reachable at all. |
| `token_secrecy` | The issued session token stays secret from the network attacker. |
| `seed_secrecy` | The citizen's enrolled Aadhaar OTP seed stays secret. |
| `auth_injective_agreement` | Auth Service and citizen agree injectively on the session (no replay / no confusion between sessions). |

Documented simplifications (see the model's header comments): the Aadhaar SMS OTP channel and the Auth-Service↔UIDAI leg are both modelled as private (non-network) channels, reflecting that the OTP seed never touches the public network and that the UIDAI integration is a pre-registered, credentialed government API — not the public internet-facing surface. No long-term key-compromise / forward-secrecy is modelled (documented as future work).

A concrete-values companion demo of the same 8-step message sequence (real RSA keys, one interactive run) is provided in [`digilocker_auth_demo.py`](digilocker_auth_demo.py) (`digilocker_auth_dry_run.sh` / `.bat` for a non-interactive dry run).

> **Note:** the captured [`tamarin_full_proof.log`](tamarin_full_proof.log) shows the theory loading and derivation/saturation starting, but the run was interrupted before the final per-lemma verified/falsified summary was captured. Re-run `tamarin-prover --prove digilocker-auth.spthy` for the authoritative verdict before citing these lemmas as proven.

## 9. Model Validation — Fixes Applied During Build

1. Docker volume mounted the wrong host folder — repointed to the project directory.
2. Broken YAML indentation had collapsed `technical_assets` nesting.
3. `data_assets_processed` referenced map keys instead of the required `id` field.
4. Threagile IDs allow only `[a-z0-9-]` — snake_case renamed to kebab-case throughout.
5. Invalid enum values (`machine` / `technology` / `size`) corrected against `threagile -list-types`.
6. Added 15 `communication_links` + 3 `trust_boundaries` — risk count rose from 40 to 74 findings.

## 10. Toolchain

- Docker 29.6.2
- `threagile/threagile:latest` (v1.0.0) — architecture risk engine
- `lmandrelli/tamarin-prover:latest` — formal protocol verification
- Model: `digilocker.yaml` · Output: `report.pdf`, `risks.json`/`.xlsx`, `stats.json`, diagrams

## 11. Repository Contents

| File | Description |
|---|---|
| `digilocker.yaml` | Threagile architecture model — source of truth for this report |
| `digilocker-auth.spthy` | Tamarin formal model of the Aadhaar OTP citizen-login flow |
| `digilocker_auth_demo.py`, `.bat`, `.sh` | Concrete-values companion demo / dry-run scripts for the auth protocol |
| `tamarin_full_proof.log` | Captured Tamarin run log (see [§8](#8-formal-verification--tamarin-prover) for caveats) |
| `digilocker-architecture-enhanced.svg` / `.pdf` | Enhanced, annotated architecture & data-flow diagram |
| `digilocker-component-register.svg` / `.pdf` | Standalone Component Register sheet |
| `digilocker-dataflow-register.svg` / `.pdf` | Standalone Data-Flow Register sheet |
| `digilocker-threat-model-poster.svg` / `.pdf` | One-page assignment summary poster |
| `output/report.pdf` | Full report: 144-page Threagile risk analysis + Architecture diagram + Component Register + Data-Flow Register |
| `output/report.original-threagile.pdf` | Pristine, unmodified Threagile engine output (backup) |
| `output/risks.json` / `.xlsx` | Machine-readable / spreadsheet risk findings (74 risks) |
| `output/stats.json` | Risk counts by severity |
| `output/technical-assets.json`, `tags.xlsx` | Supporting Threagile outputs |
| `output/data-flow-diagram.png`, `data-asset-diagram.png` | Threagile's own auto-generated diagrams |
| `README.md` | This report |
