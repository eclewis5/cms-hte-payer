# CMS Health Tech Ecosystem Payer Participation Specification

**Version 1 · June 2026**

**General Availability Date: July 4, 2026**

**Editor:** Liz Lewis (b.well Connected Health)  
**Feedback:** via CMS Health Technology Ecosystem working groups

---

## 1.0 Overview

This specification defines the technical and operational requirements for Payers participating in the CMS Health Tech Ecosystem (HTE) Interoperability Framework. It covers four use cases spanning both patient-facing (B2C) and system-to-system (B2B) data exchange.

**Payer Objective:** Join or create a CMS Aligned Network and provide claims and clinical data to CMS Aligned Networks when requested by patients, providers, and, where appropriate, other payers.

Payers must:

- Make claims data accessible to CMS Aligned Networks, in alignment with Patient and Provider Access API standards.
- Respond to patient, provider, and where appropriate, payer requests.
- Implement CMS Interoperability Framework criteria, including clinical data as appropriate.

Payers must meet the **full list of GA Requirements** below to be considered in the Payer category of the CMS HTE.

---

## 2.0 GA Requirements for Payers

The following eight requirements apply to all Payers for General Availability on July 4, 2026.

### 2.1 Claims Data Sharing

Respond to patient, provider, and payer queries for claims and benefit information.

Payers must respond to claims and coverage queries via a CMS-aligned Network from providers in their network, other payers, and patients (via their chosen app, with zero portal interaction required). Payers must also respond to queries from any application listed in the Medicare App Library.

Payers must return claims data for providers in their network through CMS-Aligned Networks they participate in without requiring portal accounts. Proactive CDA pushes do not satisfy this requirement; a query-based response is required.

### 2.2 Clinical Data Query

Payers can query providers for clinical data supporting claims submitted in the prior 60 days.

Payers may request relevant clinical data for claim adjudication or quality purposes. Responses should include structured FHIR data or PDFs.

**OPTIONAL** for the July 4, 2026 GA milestone, though encouraged.

### 2.3 Clinical Quality Query

Payers, including CMS, and other value-based care organizations may query for measure-specific clinical data under the healthcare quality improvement (`HQUALIMP`) purpose of use.

Queries may be used for quality measurement, care gap identification, quality reporting, and value-based care programs. Providers should respond with the agreed-upon data elements necessary for the applicable measure, consistent with the HIPAA minimum necessary standard.

**Status:** Evaluate Feasibility — spec from community pending finalization.

### 2.4 National Provider Directory Integration

Support creation, maintenance, and submission of provider directory entries in the National Provider Directory (NPD).

Payers must register with the NPD to publish provider organization, location, endpoint, and verification metadata, and provide the locations of required Payer FHIR endpoints to the NPD. Payers publish directly because they typically maintain their own endpoint and identity infrastructure.

If a Payer participates in a CMS Aligned Network, their network can handle NPD updates on their behalf.

### 2.5 Patient Matching

Incorporate CMS-approved patient matching logic.

Patient matching is the process of correctly identifying a person across different health systems, networks, and data sources. Every CMS-aligned network must reliably confirm that the person requesting or associated with the data is the correct individual.

If a query arrives and a match is made using any of the 27 approved combinations, the Payer must respond, subject to applicable law.

The spec defines required normalization rules for names, dates, identifiers, and addresses. Failure to return records when a listed combination uniquely matches is non-compliant. Implementation of patient matching logic does not relieve participants of independent HIPAA obligations.

**Reference:** CMS Patient Matching Proposal v3.2.2 — pinned in #identity-verification-and-authentication (CMS HTE Slack).

A CMS-aligned network may accept the IAL2 token, validate it, and handle patient matching on behalf of the payer — this is acceptable.

### 2.6 Identity & Trust

Participate in the National Provider Directory (NPD) for endpoint and identity verification.

Payers must ensure their query endpoints, organizational identity, and contact metadata appear in the NPD to support secure, trusted exchange across networks.

### 2.7 Patient Access (IAL2)

Support patient access to claims and clinical data using CMS-approved IAL2 credentials.

Payers must accept IAL2 identity verification for patient queries without requiring portal login, consistent with the Interoperability Framework.

A CMS-aligned network may accept the IAL2 token and validate it on behalf of the payer.

### 2.8 Auditability

Provide data holder–level, network-level audit logs for payer access.

Payers must log data requests and responses (who, when, purpose of use) and provide these network-level audit logs to networks for patient-facing transparency. Logs must be retained for at least seven (7) years, or longer if required by applicable law (45 CFR 164.530(j)).

This addresses network-level data holder–level logging only; payers' independent obligation to log workforce access within their own systems under 45 CFR 164.312(b) is not displaced.

---

## 3.0 Use Case - Patient Access (B2C)

### 3.1 Overview

Under the CMS Health Tech Ecosystem Interoperability Framework, Payers are required to honor patient-authorized data requests from third-party Patient Apps — without requiring patients to log into a payer portal. Each request arrives via a CMS-aligned Network that has already validated the requesting app's standing, attestations, and IAL2 chain of trust.

**Standards basis:**

- FHIR R4 data model
- NIST 800-63 IAL2 identity verification (via approved Credential Service Providers)
- SMART App Launch v2 — Client Confidential Asymmetric profile
- OAuth 2.0 `client_credentials` grant with signed JWT `client_assertion`

There are two authorization paths Payers will see, depending on how the CMS-aligned Network and the Payer's authorization server handle consent. Both are fully compliant; choice depends on the Payer's governance posture.

### 3.2 Path 1 (Ideal): Network-Aligned Client Credentials with IAL2

The Patient App captures patient consent and verifies patient identity to IAL2 with an approved Credential Service Provider (CSP). The Network presents the Payer's `/token` endpoint with a `client_credentials` request whose signed `client_assertion` JWT carries the IAL2 `id_token` in a structured `cms_smart` extension. No redirect to the Payer's domain. No consent UI hosted by the Payer.

**What arrives at the Payer's `/token` endpoint**

The inbound `client_assertion` JWT carries a structured `cms_smart` extension with:

- `version` — fixed value `"1"`
- `purpose_of_use` — fixed value `"PATRQT"` (Patient Request)
- `id_token` — CSP-issued IAL2 `id_token` with the patient's identity assertions (name, DOB, address, last 4 of SSN, etc., per the CMS-approved patient matching spec)

The outer JWT is signed by the Network (RS384 or ES384, 5-minute lifetime) and addressed to the Payer's `/token` endpoint.

**Payer validation steps**

1. Validate the outer JWT signature against the Network's registered JWKS
2. Validate the nested IAL2 `id_token` against the CSP's JWKS, including `identity_assurance_level = 2` and recency claims
3. Perform patient matching against the member/patient roster per the CMS-approved patient matching specification
4. Issue an access token scoped to the requested SMART v2 scopes

### 3.3 Path 2 (Alternate): Partner-Hosted Consent with Attestation

Same `client_credentials` shape as Path 1, with one addition: the Payer hosts a consent policy at a stable, publicly available URL. The Patient App renders the policy to the patient in-app and captures acceptance before any request is made to the Payer. The Network includes a `consent_attestation` in the inbound JWT.

**What the Payer publishes**

A consent policy at a stable URL. This URL should always point to the current version of the policy — do not change the URL when the policy changes.

**What additionally arrives in the JWT `cms_smart` extension**

- `consent_policy` — array containing the URL of the policy the patient reviewed and accepted
- `consent_attestation` — `{ user_action: "approved", user_action_timestamp: <ISO-8601> }`

**Additional Payer validation steps**

Validate `consent_attestation` freshness against the Payer's defined window; reject as `consent_expired` if the `user_action_timestamp` is stale.

A third fallback using the `authorization_code` grant is available for organizations not yet ready for either path above. Contact your CMS-aligned Network for details.

### 3.4 Token Request Structure

```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&scope=patient/Patient.rs patient/Coverage.rs patient/ExplanationOfBenefit.rs
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion={SIGNED_JWT}
```

Client assertion header:

```json
{
  "alg": "RS384",
  "kid": "key-id",
  "typ": "JWT"
}
```

Client assertion payload:

```json
{
  "iss": "client-id",
  "sub": "client-id",
  "aud": "https://payer-token-endpoint.com/token",
  "jti": "unique-token-id",
  "exp": 1774651583,
  "extensions": {
    "cms_smart": {
      "version": "1",
      "purpose_of_use": "PATRQT",
      "id_token": "<IAL2 Token>",
      "consent_policy": ["https://payer.com/consent-policy"],
      "consent_attestation": {
        "user_action": "approved",
        "user_action_timestamp": "<ISO-8601>"
      }
    }
  }
}
```

### 3.5 What Payers Must Return

In response to patient-authorized queries, Payers must return claims, coverage, and clinical data scoped to the requesting patient. At minimum: USCDI FHIR R4 resources covering the dataset relevant to the Payer's role — `Patient`, `Coverage`, `ExplanationOfBenefit`, `Condition`, `Observation`, `MedicationRequest`, etc.

All data that is online and available to patients must be provided, whether the patient is a current member or not.

SMART v2 scope enforcement: Issue tokens limited to the scopes the app requested and enforce scope-level access on every FHIR call.

### 3.6 Technical Requirements

| Capability | Requirement | Path |
|---|---|---|
| USCDI FHIR API (read) | Patient-facing FHIR R4 endpoints covering USCDI dataset | Baseline |
| `client_credentials` + `client_assertion` JWT | Auth server accepting SMART Client Confidential Asymmetric profile with structured extension parsing | Baseline |
| JWKS endpoint | Publicly-resolvable URL hosting auth server's JWK Set, with key rotation | Baseline |
| IAL2 `id_token` validation | Validate nested CSP-issued `id_token` against CSP's JWKS | Baseline |
| Patient matching | Resolve demographics in `id_token` to a single patient per CMS-approved spec | Baseline |
| SMART v2 scope enforcement | Issue and enforce scoped tokens on every FHIR call | Baseline |
| Hosted consent policy URL | Stable, versioned consent policy URL; validate attestation freshness | Path 2 only |

### 3.7 Registration

**Current state (manual):** Provide a publicly-resolvable JWK Set URL, `/token` endpoint, FHIR base URL, and (if on Path 2) hosted consent policy URL to the CMS-aligned Network. The Network registers the Payer and surfaces it to participating Patient Apps.

**Future state:** Auto-registration aligned with UDAP / dynamic registration patterns is forthcoming. Build against the manual flow now.

---

## 4.0 Use Case - Claim-Triggered Query (B2B)

### 4.1 Overview

When a claim is received, a Payer may query the submitting provider's systems for supporting clinical data. This enables utilization management, claims adjudication, and quality program needs without manual chart pull requests. The scope is limited to clinical data relevant to claims submitted in the prior 60 days.

**GA Requirement:** Payers can query providers for clinical data supporting claims submitted in the prior 60 days. Responses should include structured data or PDFs.

### 4.2 Workflow

1. **Claim received** — Payer receives a claim from a provider via standard claims channel.
2. **Query initiation** — Within 60 days of claim receipt, the Payer initiates a targeted clinical data query to the submitting provider's FHIR endpoint.
3. **Provider response** — Provider returns relevant clinical data as structured FHIR resources or PDF attachments.
4. **Payer ingestion** — Payer ingests the returned data for claim adjudication, utilization management, or quality purposes.

### 4.3 Scope and Parameters

| Parameter | Value |
|---|---|
| Time window | Query must be issued within 60 days of claim receipt |
| Query scope | Targeted to the specific claim encounter — not bulk population queries |
| Purpose of Use | `CLMATTCH` (Claim Attachment) — [HL7 v3 Purpose of Use code set](https://terminology.hl7.org/en/ValueSet-v3-PurposeOfUse.html) |
| Data format | Structured FHIR R4 resources preferred; PDF attachments accepted |

### 4.4 Known Issues

- Some providers do not sign clinical notes, so data will not be released via API call. A mechanism for providers to indicate "pending/partial response" is under discussion in the workgroup.
- The 60-day window is recognized as tight. The workgroup may revisit this constraint in future releases.
- Providers who are ready to respond to payer queries are being identified in the Provider Workgroup — coordinate with your CMS-aligned Network to identify testing partners.

### 4.5 Standards

- Da Vinci CDex (Clinical Data Exchange) IG — for structured attachment and documentation request workflows
- SMART Backend Services — for system-to-system authorization
- CMS-approved patient matching spec

---

## 5.0 Use Case 3: Payers Initiating Queries for Quality Measure Reporting (B2B)

### 5.1 Overview

Payers, including CMS, and other value-based care organizations may query for measure-specific clinical data from providers under the healthcare quality improvement (`HQUALIMP`) purpose of use. Queries support quality measurement, care gap identification, quality reporting, and value-based care programs.

**GA Requirement Status:** Evaluate Feasibility — content hold pending community spec finalization.

This use case documents the current working model developed by the Payer/Provider Small Group and is intended to guide early adopters and testing partners.

### 5.2 Workgroup Agreements

- The minimum FHIR resources for a quality measure are defined by the measure authority (NCQA or CMS). US Quality Core is the long-term target but not viable today.
- The DEQM Data Exchange MeasureReport wrapper is optional when a provider sends data to a payer. The clinical payload is what matters.
- A Payer will not run live queries on a Provider's system for quality purposes — the Provider packages and delivers the data.

### 5.3 End-to-End Workflow

The workflow involves four participants: Measure Authority, Payer, Provider, and Measure Calculation Engine.

| Step | Actor | Action |
|---|---|---|
| 1 | Measure Authority | Publishes Measure Resource and Library Resource, including `dataRequirements` and CQL/measure logic. Both Payer and Provider confirm access before testing begins. |
| 2 | Payer | Shares attributed member roster with the Provider as a Da Vinci ATR `Group` resource. Minimum refresh cadence: monthly. Payer may push or Provider may pull via Provider Access API. |
| 3 | Provider | Consumes ATR `Group`, reconciles against MPI, and reports unmatched members back to the Payer. |
| 4 | Provider | Determines required data by consulting `dataRequirements` in the Library Resource (`$data-requirements`) and collects data for the attributed population. |
| 5 | Provider | Assembles a Bulk FHIR file (NDJSON of bundles organized by Patient). DEQM Data Exchange MeasureReport metadata is optional — the clinical resource bundle is the required payload. |
| 6 | Payer | Performs patient match if needed. If Provider stored and returned Payer identifier, this step may not be required. |
| 7 | Payer | Combines EHR data with claims data and submits to a Measure Calculation Engine (CQL or equivalent). May use current systems if CQL is not yet available. |
| 8 | Payer | Shares patient-level results → Provider (DEQM Individual MeasureReport) [secondary priority] and population-level summary → Measure Authority (DEQM Summary MeasureReport) [optional]. |
| 9 | Provider | (Optional) Based on Gaps in Care Report, repeats data-sharing process focusing on patients with open gaps. |

### 5.4 Measure Selection

Each Payer/Provider pair selects at minimum one (1) measure for testing. Any eCQM or HEDIS measure is eligible.

- **eCQMs:** eCQI Resource Center — https://ecqi.healthit.gov/ep-ec/ecqms
- **HEDIS measures:** NCQA — https://www.ncqa.org/hedis/

CMS eCQM data requirements are available via open source (eCQI Resource Center). NCQA HEDIS content requires licensure — confirm access arrangements before testing begins.

### 5.5 FHIR Terminology Mapping

| Concept | FHIR Representation |
|---|---|
| Population definition / roster | Da Vinci ATR `Group` resource |
| Measure logic and data requirements | Library Resource (`$data-requirements`) |
| Clinical data payload | FHIR Bundle (NDJSON, organized by Patient) |
| Measure calculation trigger | CQL / Measure Engine running `$evaluate` |
| Patient-level results | DEQM Individual MeasureReport |
| Population-level results | DEQM Summary MeasureReport |
| Care gap feedback to provider | DEQM Gaps in Care Report |

### 5.6 Known Gaps

**eCQM gap (USCDI v3 + 10 additional elements)**

USCDI v3 alone does not carry enough for most eCQMs. Ten additional data elements are needed: Follow-Up Plan, Discharge Summary Sent, Assessment Tool Score, Section GG Functional Status, PHQ-9, Blood Culture Collected, Medication Reconciliation List, Medication Administration, Specialist Report Received, Device Days. `MedicationStatement` and `DeviceUseStatement` are not profiled in US Core. `Specimen` and `MedicationDispense` exist only in STU6+.

**HEDIS gap (HEDIS IG vs. US Core)**

HEDIS IG builds on US Core but adds payer-side resources with no USCDI equivalent (`Claim`, `EOB`, `ClaimResponse`, `Coverage` extensions, `RelatedPerson`) and stricter clinical requirements: `MedicationDispense.daysSupply` and `whenHandedOver` required for PDC; `Condition.onset` required; `Observation.status` restricted to `final/amended/corrected`; `Encounter.period` required.

**Infrastructure**

Bulk FHIR has shown reliability issues in real-world testing. Patient-by-patient queries work but are slow. Build patient-by-patient as the baseline; optimize with Bulk FHIR later.

**Provenance**

HEDIS metatag (`meta.tag.hedisDataSource`) is being retired. Use standard FHIR `Provenance` until the workgroup converges on a replacement. NCQA data quality validation (to replace PSV) is targeted for 2027.

### 5.7 Open Issues

- Push vs. pull is not resolved. Infrastructure to date is query/pull. HEDIS push has been proposed but not agreed.
- CQL Engine certification: DQIC is vetting engines for consistency. Confirm whether the selected Measure Authority requires a certified engine before production deployment.
- Standardizing patient-level calculated measure communication (payer → provider): Excel spreadsheets are not viable for production. Pending workgroup agenda item.
- Submit written comments to ONC on USCDI+Quality v1 covering the +10 eCQM gap and HEDIS-specific requirements.

### 5.8 References

- [HL7 Da Vinci ATR v2.1.0](https://hl7.org/fhir/us/davinci-atr/)
- [HL7 Da Vinci DEQM v5.0.0](https://hl7.org/fhir/us/davinci-deqm/)
- [HL7 FHIR CQM Capability Statements](https://build.fhir.org/ig/HL7/fhir-cqm/en/capabilities.html)
- [eCQI Resource Center](https://ecqi.healthit.gov/ep-ec/ecqms)
- [NCQA HEDIS Measures](https://www.ncqa.org/hedis/)

---

## 6.0 Use Case 4: Payers Responding to Queries from Providers and Other Payers (B2B)

### 6.1 Overview

In addition to patient-initiated queries, Payers must respond to queries from Providers in their network and, where appropriate, from other Payers. These are system-to-system (B2B) exchanges operating under HIPAA TPO purposes of use and applicable network agreements.

**GA Requirement:** Respond to patient, provider, and payer queries for claims and benefit information via a CMS-aligned Network.

### 6.2 Provider Queries to Payers

Providers may query Payers for claims and clinical data about shared patients to support care coordination, utilization management, and treatment. Payers must:

- Return claims data for providers in their network through CMS-Aligned Networks without requiring portal accounts.
- Respond with USCDI v3 or greater (required minimum: claims data; USCDI v3 clinical data is best practice).
- Support provider queries without requiring IAL2 — the CMS-aligned Network uses the NPD to validate provider identity.
- A proactive CDA push does not satisfy this requirement; a query-based response is required.

**Purpose of Use:** HIPAA Treatment (`TREAT`) or Healthcare Operations (`HOPERAT`), as applicable.

### 6.3 Payer-to-Payer Queries

When a patient changes health plans, the new Payer may query the prior Payer for claims history and clinical data to support care continuity.

- **Patient consent:** Member consent and selection of prior plan is required, typically captured at enrollment.
- **Data scope:** Claims history, coverage information, and clinical data per PDex IG scope.
- **Network routing:** Payer must be registered in the NPD and reachable via a CMS-aligned Network.

### 6.4 Minimum NPD Directory Fields

| Field | Notes |
|---|---|
| Organization Name | Parent/brand name — what the patient would recognize (e.g., "Anthem") |
| Plan Name | Specific plan name (e.g., "Anthem Virginia") |
| Endpoint | Can be shared across plans; same or different endpoint per plan is acceptable |
| Payer ID + Status | |
| Tax ID | |
| NPI | |
| NAIC | |
| HIOS ID | |

Minimum data for member matching/routing: Member ID + Full Name + DOB enables routing to the correct entity.

**Standards basis:**
- HL7 Da Vinci PDex (Payer Data Exchange) IG v2.x
- PDex v2.2 introduces NDJSON response format — confirm version support with your network partner

### 6.5 Technical Requirements

| Requirement | Description |
|---|---|
| CMS-Aligned Network participation | Payer must be connected to at least one CMS-aligned Network capable of routing B2B queries |
| NPD registration | Endpoint and identity metadata published and kept current in the NPD |
| FHIR R4 APIs | FHIR R4 endpoints for claims, coverage, and clinical data |
| SMART Backend Services | System-to-system OAuth 2.0 for B2B authorization |
| Patient matching | Apply CMS-approved patient matching spec to incoming queries |
| Scope enforcement | Enforce purpose-of-use and data scope per query context |
| Audit logging | Log all inbound queries and responses per the Auditability GA requirement |

### 6.6 Payvider Considerations

Payviders with full authority to adjudicate claims and act as a Payer are subject to all Payer requirements. Each Payer is responsible for informing their Payviders of NPD update obligations. The Payvider is the responsible party for maintaining their own NPD entries.

### 6.7 Secondary Use Restrictions

The following uses of data queried through CMS-aligned Networks are not permitted:

- Pricing or contract negotiation
- Denying coverage to members in future enrollment periods

The workgroup is developing a formal list of permitted and non-permitted secondary uses. These restrictions should be reflected in network participation agreements.

---

## 7.0 Appendix: Key References

| Resource | URL |
|---|---|
| CMS HTE Interoperability Framework | https://www.cms.gov/health-technology-ecosystem/interoperability-framework |
| CMS HTE Categories | https://www.cms.gov/health-technology-ecosystem/categories |
| CMS Patient Matching Proposal v3.2.2 | Pinned in #identity-verification-and-authentication (CMS HTE Slack) |
| SMART App Launch v2 | https://hl7.org/fhir/smart-app-launch/ |
| SMART Backend Services | https://build.fhir.org/ig/HL7/smart-app-launch/backend-services.html |
| Da Vinci CDex | https://hl7.org/fhir/us/davinci-cdex/ |
| Da Vinci ATR v2.1.0 | https://hl7.org/fhir/us/davinci-atr/ |
| Da Vinci DEQM v5.0.0 | https://hl7.org/fhir/us/davinci-deqm/ |
| Da Vinci PDex | https://hl7.org/fhir/us/davinci-pdex/ |
| CDex Signatures | https://build.fhir.org/ig/HL7/davinci-ecdx/signatures.html |
| eCQI Resource Center | https://ecqi.healthit.gov/ep-ec/ecqms |
| NCQA HEDIS | https://www.ncqa.org/hedis/ |
| CARIN Code of Conduct | https://www.carinalliance.com/code-of-conduct |
