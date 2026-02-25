## Key Takeaways

Essential aspects of SMART-on-FHIR—what to internalize so you can build and troubleshoot real integrations.

### Core Concepts

1. **SMART-on-FHIR = OAuth + context, on top of FHIR**
   - **FHIR** answers: *what data exists and how do I query it?*
   - **SMART** answers: *who is the user, what can they access, and what patient/encounter is “in context”?*

2. **Everything interesting in FHIR is a graph**
   - You rarely want a single resource in isolation.
   - You will constantly traverse references: `Observation.subject → Patient`, `DiagnosticReport.result → Observation`, `Encounter.participant → PractitionerRole`, etc.
   - Your app architecture should assume *graph navigation* and *partial/missing edges*.

3. **Bundles are not “just arrays”**
   - Search results, includes, paging links, transactions, and errors all show up in bundle-shaped responses.
   - Production apps must follow `Bundle.link[relation="next"]` and handle “include” entries distinctly from “match” entries.

4. **Scopes are a product requirement, not an implementation detail**
   - Scopes drive user trust, security review outcomes, and whether an EHR will even approve the app.
   - Start with least privilege; expand only when you can explain the user value.
   - Expect variation between servers (supported scopes and policy decisions).

5. **Terminology is the difference between “works” and “clinically correct”**
   - If you query by strings (“glucose”), you’ll ship a fragile app.
   - Query by codes (LOINC/SNOMED/RxNorm) and be explicit about systems.

### Practical Implementation

The interactive React application demonstrates:

* Connecting to the SMART Health IT sandbox
* Retrieving various resource types
* Displaying clinical data in user-friendly format
* Handling errors gracefully

### Best Practices

* Always validate resources before submission (base spec **and** required profiles)
* Implement structured error handling:
  - Parse and display `OperationOutcome.issue[]`
  - Treat 401/403 distinctly (auth vs authorization/policy)
  - Build “retry vs fail” rules around 429/5xx
* Treat discovery as mandatory:
  - Cache `/.well-known/smart-configuration` (with sensible TTL)
  - Don’t hardcode endpoints, PKCE support, or scope availability
* Handle token lifecycle intentionally:
  - Expiration, refresh (where allowed), revocation/log-out, and safe storage
* Test against more than one dataset:
  - Multiple patients, edge cases (no vitals, missing references, sparse coding)
* Prefer an implementation guide (IG) mindset:
  - Base FHIR is not enough; most real deployments expect an IG like US Core

### Resources for Further Learning

* SMART App Launch Framework (Official HL7 specification): <br>[http://hl7.org/fhir/smart-app-launch/](http://hl7.org/fhir/smart-app-launch/)
* FHIR R4 Specification: <br>[http://hl7.org/fhir/R4/](http://hl7.org/fhir/R4/)
* US Core Implementation Guide: <br>[http://hl7.org/fhir/us/core/](http://hl7.org/fhir/us/core/)
* Mobile app security guidance (OAuth 2.0 for Native Apps (RFC 8252): <br>[https://tools.ietf.org/html/rfc8252](https://tools.ietf.org/html/rfc8252)
* SMART Health IT Sandbox: <br>[https://launch.smarthealthit.org](https://launch.smarthealthit.org)
* Threat models and mitigations (FHIR Security Documentation): <br>[https://www.hl7.org/fhir/security.html](https://www.hl7.org/fhir/security.html)