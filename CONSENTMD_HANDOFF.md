# Consent MD — Hackathon Build Handoff

**Purpose of this file:** drop this into your Claude Code working directory (alongside `consentmd_poc.html`) at the start of your session. It gives Claude Code — and whoever you're pairing with — full context on what exists, why it's built the way it is, and what's worth building next, without you having to re-explain everything verbally.

---

## 1. The problem, in one paragraph

Surgical informed consent is generic today — the same boilerplate form regardless of whether a patient is getting a routine procedure or something with materially different risk. Inadequate informed consent is a commonly cited factor in surgical malpractice claims. Consent MD is a modular consent engine that assembles patient- and procedure-specific consent documents from stacked, independently maintained risk layers, instead of static one-size-fits-all forms.

## 2. What exists right now: `consentmd_poc.html`

A single self-contained HTML file (no backend, no build step — open it in any browser). It demonstrates the full mechanism end to end:

1. **Mock EMR import** — select from 5 synthetic patients, each with a fuller chart: vitals, allergies, active conditions, medications, prior surgeries, dated labs, social history, and free-text clinical notes (deliberately including things a diagnosis-code checklist would miss — e.g. a recent fall, missed medication doses, chronic immunosuppression).
2. **Procedure selection** — Orthopedic Surgery → Total Joint Replacement family: THA, TKA, partial knee, total shoulder arthroplasty, with technique modifiers (robotic-assisted, bilateral, reverse construct).
3. **Layered assembly** — visually stacks four independent risk layers: Specialty (universal surgical risks) → Subspecialty (TJR-specific risks) → Procedure (procedure-specific risks) → Patient (EMR-derived adjustments). Each layer is a separate array in the JS, mirroring the real system's versioned, independently-maintained layer architecture.
4. **Generated consent document** — assembles all four layers into a single readable document, color-coded by source layer.
5. **Signable PDF output** — client-side PDF generation (pdf-lib) with real AcroForm fields: patient name, patient signature, date, surgeon signature, plus an acknowledgment checkbox. Downloads directly from the browser.
6. **Portability QR code** — encodes the assembled record's parameters into a URL; scanning it (once hosted) reopens the same patient/procedure/modifier combination.
7. **Agentic layer** — this is the most important piece to understand:
   - Three **manual** single-call buttons (draft risk language, scan chart for overlooked risks, review document completeness) — each is one Claude API call with a scripted fallback if no API key is present.
   - One **autonomous workflow** button — this gives Claude actual **tools** (`lookup_taxonomy_layer`, `flag_risk`, `finalize_document`) and lets it decide for itself what to look up, what to flag, and when it's done, executing a real tool-use loop rather than a single prompt/response. A live activity log shows each tool call as it happens. This is the "wow" moment — Claude is deciding steps, not just autocompleting text.
   - All live calls require the user to paste their own Anthropic API key into a password field (client-side only, never persisted, never sent anywhere but api.anthropic.com). Every button gracefully falls back to a realistic scripted example if no key is present or the call fails — so the demo never breaks on stage.

## 3. Governing principles from the underlying taxonomy (Phase 1, closed)

These are binding architectural decisions from the broader Consent MD taxonomy project. The POC's data model is a deliberately simplified slice of this — worth knowing so you don't accidentally violate the architecture while extending it:

- **P1** — Procedures are classified primarily by the specialty of the surgeon performing them.
- **P2** — Technique modifiers (laparoscopic, robotic, bilateral, etc.) are attributes/dropdowns on a procedure, not separate procedures.
- **P3** — Adjunctive procedures performed as part of a primary operation are not standalone procedures.
- **P4** — Procedures are assigned to the specialty that most commonly performs them in standard practice, not by anatomy/organ system.
- **P5** — A procedure performed by multiple specialties may appear under each, with shared ownership noted.
- **P6** — Procedure extent (partial/total, unilateral/bilateral) is a modifier, not a separate classification.
- **P7** — The taxonomy is version-controlled: each specialty has a versioned static export and a changelog.
- **P8** — Each specialty/subspecialty includes an "Other" category for unlisted procedures.
- **P9** — Decisions made in a session are binding for the remainder of the work unless explicitly revisited.
- **P10** — Independent ranked datasets (volume, settlement, claim frequency) must be sourced and ranked independently — never derived from each other.
- **P11** — **Consent is assembled from independent, stackable risk layers**: Specialty → Subspecialty → Procedure, each independently maintained and version-controlled. (The POC adds a fourth layer, Patient, on top of this — an extension beyond what Phase 1 locked, worth flagging as a deliberate addition rather than an inherited decision.)

Full specialty list (13 parents, 7 children) and the ~430-procedure registry exist in the Phase 1 taxonomy export — not included in this file, but relevant if you want to expand beyond the TJR slice used in the demo.

## 4. Data model reference (as implemented in the POC)

```
SPECIALTY_RISKS            // flat array, universal risks — Orthopedic Surgery
SUBSPECIALTY_RISKS.tjr     // array — Total Joint Replacement domain risks
PROCEDURES = {
  tha: { label, subspecialty, risks: [...] },
  tka: { ... },
  pka: { ... },
  rsa: { ... }
}
PATIENTS = {
  p1: {
    name, demo, mrn, procedureFit,
    vitals: { bp, bmi, smoking },
    allergies: [...], conditions: [...], meds: [...],
    priorSurgeries: [...],
    labs: [{ name, value, date, flag, note }],
    social: "...",
    notes: "...",           // free text — the anomaly-scan target
    riskFlags: [{ trigger, text }]
  },
  ...
}
```

Synthetic data only — written by hand, not real de-identified patient records. Deliberately not using real PHI, even anonymized, in something taken to a public event.

## 5. What's realistic to build next (in rough priority order)

1. **Move the API key server-side.** Right now the key lives in a browser input field — fine for a controlled demo, not fine for anything real. A minimal backend (even a single serverless function) that proxies the Claude API call would remove this exposure. This is probably the single highest-value thing to build with an engineer today.
2. **Swap mock patients for Synthea-generated data.** [Synthea](https://github.com/synthetichealth/synthea) is an open-source synthetic patient generator that outputs realistic FHIR bundles — more statistically believable than hand-written examples, still synthetic, still safe to demo publicly.
3. **Real FHIR sandbox connection.** Epic on FHIR or Cerner Code Console both offer free developer sandboxes with synthetic patient data. This is the legitimate "next sprint" step toward real EMR integration — not production access (that requires a hospital's IT/compliance sign-off, a business process not an engineering one), but a real SMART-on-FHIR OAuth2 flow against sandbox data.
4. **Expand the taxonomy slice.** The POC only covers Ortho TJR (4 procedures). The full Phase 1 taxonomy has ~430 procedures across 13 specialties — extending the demo to a second specialty would show the architecture generalizes.
5. **Harden the agentic tool-use loop.** Right now it runs a fixed 6-turn budget with three tools. Worth exploring: letting the agent request a human-in-the-loop review before finalizing, or adding a tool that lets it search the taxonomy for the correct subspecialty/procedure address rather than assuming it (this is the actual "routing" job the real taxonomy does).
6. **Real e-signature.** The current PDF's "signature" fields are plain fillable text fields — not legally binding e-signature. DocuSign or Adobe Sign integration would be the real answer; out of scope for a hackathon but worth naming explicitly if asked.

## 6. Boundaries worth stating explicitly to teammates/judges

- This is a proof of concept for the *assembly mechanism*, not a claim that EMR integration, e-signature, or clinical risk data validation are solved.
- All risk content is illustrative/demo data, clearly labeled as such in the UI and PDF output — not the validated, sourced risk dataset from the underlying taxonomy project.
- The buyer/business model (malpractice insurance carriers as primary customer) is intentionally not part of the public pitch — keep that in your back pocket, not on stage.

## 7. Files in this handoff

- `consentmd_poc.html` — the working demo, open directly in a browser
- `CONSENTMD_HANDOFF.md` — this file

---

*Prepared for hackathon build session, July 18, 2026.*
