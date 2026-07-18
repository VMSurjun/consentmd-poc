# Consent MD

**A modular consent engine that assembles patient- and procedure-specific surgical consent documents from stacked, independently maintained risk layers — instead of one-size-fits-all boilerplate.**

🔗 **[Live demo](https://vmsurjun.github.io/consentmd-poc/)**

## The problem

Surgical informed consent is generic today — the same boilerplate form regardless of whether a patient is getting a routine procedure or something with materially different risk. Inadequate informed consent is a commonly cited factor in surgical malpractice claims.

Consent MD assembles consent documents on the fly from four independent, versioned risk layers — **Specialty → Subspecialty → Procedure → Patient** — so the final document reflects the actual patient and the actual procedure, not a generic template.

## What this proof of concept demonstrates

A single self-contained HTML file (no backend, no build step) that walks through the full mechanism end to end:

1. **Mock EMR import** — select from synthetic patients with a full chart: vitals, allergies, conditions, medications, dated labs, and free-text clinical notes (deliberately including things a diagnosis-code checklist would miss, like a recent fall or missed medication doses).
2. **Procedure selection** — Orthopedic Surgery → Total Joint Replacement: THA, TKA, partial knee, total shoulder, with technique modifiers (robotic-assisted, bilateral, reverse construct).
3. **Layered assembly** — visually stacks the four risk layers, each maintained as an independent, color-coded source.
4. **Generated consent document** — all four layers assembled into one readable, attributed document.
5. **Signable PDF output** — real AcroForm fields (patient name, signatures, date, acknowledgment checkbox), generated client-side and downloaded directly from the browser.
6. **Portability QR code** — encodes the assembled record's parameters into a URL that reopens the same patient/procedure/modifier combination.
7. **Agentic layer** — the core differentiator:
   - Three manual single-call buttons (draft risk language, scan the chart for overlooked risks, review document completeness).
   - One **autonomous workflow** — Claude is given real tools (`lookup_taxonomy_layer`, `flag_risk`, `finalize_document`) and decides for itself what to look up, what to flag, and when it's done, in an actual tool-use loop rather than a single prompt/response. A live activity log shows each tool call as it happens.
   - Every live feature gracefully falls back to a realistic scripted example if no API key is present (or a call fails) — the demo never breaks on stage.

All risk content and patient data is **synthetic and illustrative**, hand-written for this demo — not real PHI, not the validated production risk dataset.

## Running it

No install required — this is a single static HTML file.

- **Online:** open the [live demo](https://vmsurjun.github.io/consentmd-poc/)
- **Locally:** download `consentmd_poc.html` and open it directly in any browser

To try the live agentic features (instead of the scripted fallback), paste an Anthropic API key into the field in the "Agentic layer" panel. The key is used only for direct browser calls to `api.anthropic.com` — it's never persisted or sent anywhere else.

## Tech

Single-file HTML/CSS/vanilla JS. [pdf-lib](https://pdf-lib.js.org/) for client-side PDF generation, [qrcodejs](https://davidshimjs.github.io/qrcodejs/) for the portability QR code, direct browser calls to the Anthropic Messages API (including tool use) for the agentic layer.

## Boundaries

This is a proof of concept for the *assembly mechanism* — not a claim that EMR integration, e-signature, or clinical risk data validation are solved. See [`CONSENTMD_HANDOFF.md`](./CONSENTMD_HANDOFF.md) for the fuller architecture context, governing taxonomy principles, and roadmap.
