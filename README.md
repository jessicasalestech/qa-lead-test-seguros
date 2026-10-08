<!-- language selector -->
[🇧🇷 Português](README-pt-BR.md)

# QA Lead Senior | Insurance — Technical Test

**Decision document** for the *Nova Jornada de Sinistros* (New Claims Journey) program at **NFoque**.

A senior QA analysis of how to build a Quality Assurance front for a critical, large-scale
program under schedule pressure — investigation, risk, strategy, governance, indicators,
automation and leadership communication. No code required; this is a decision document.

## Deliverables

| File | Format |
|---|---|
| [`Jessica_Sales_testetecnico.md`](Jessica_Sales_testetecnico.md) | Markdown |
| [`Jessica_Sales_testetecnico.docx`](Jessica_Sales_testetecnico.docx) | Word |

The document reproduces the client's original statement plus the full answers, in Portuguese.

## What's inside

1. **Investigation before answering the sponsor** — hypotheses for the four simultaneous
   symptoms (240 stuck claims, 62 contradictory messages, tripled antifraud volume, duplicate
   payment), why a 100%-green homologation can coexist with production failures, an
   evidence-based investigation plan, how to size the real impact, and the objective conditions
   behind a gated Wave 3 decision.
2. **Quality strategy and test plan** — test levels (including the missing cross-system
   journey tests and contract tests), objective entry/exit criteria, the two integrations
   without a homologation environment, and a risk-based coverage criterion.
3. **Governance** — defect workflow, severity/priority and response agreement, definition of
   ready/done replacing *"it worked in the demo"*, and six quality indicators (each tied to a
   concrete decision), plus the indicators deliberately not adopted.
4. **Automation nobody trusts** — triage of 220 flaky E2E scenarios, a new automation
   strategy, and which results block a release versus merely inform.
5. **Team and beyond-scope proactivity** — restructuring 3 QA analysts across 4 squads, the
   risk-based case for more QA capacity, and three self-initiated improvements (rollback
   plan, requirement-to-defect traceability, test data and LGPD).
6. **AI in QA** — where to rely on AI, where to validate, where it creates false coverage,
   and the LGPD/security risks.

## About

**Jessica Sales Melo** — QA Engineer Senior (7+ yrs) in banking, financial services,
e-commerce and insurance; ISTQB CTFL; automation across Playwright, Cypress, Selenium,
Appium, Robot Framework, Rest Assured; event-driven AWS architectures, CI/CD and BDD.

*This is an interview technical test submission — a decision document, not application code.*