# Reflection Brief — Evaluation and Observability Capstone

**Name:**
**Date:** 2026-11-18

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux gvisor 4.19.0-gvisor x86_64 |
| Python version | 3.11.18 (via uv, seed venvs) |
| Date run | 2026-11-18 |
| Ran any system live? (which) | Yes: pipeline-run, routing_perturbation, perturbation-run (record) and 3 live tests; rest replay/offline |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 48 passed (01-policy-pipeline/tests.txt line 71) |
| Routing output file | routing_decisions.json (9 decisions written) |
| auto_approve / human_review / spot_check counts | 1 / 7 / 1 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> My perturbation did not escalate. Policy POL-2025-009 escalated with reason "endorsements_absent" (missing_source category) shown in pipeline-run.txt lines 42-43. The pipeline output shows "validation_failed" (line 26) indicating extraction completely failed on the missing endorsements data. The system made single API calls per policy (HTTP "200 OK" markers in logs show 1 call per policy ID). The test test_ac_01_04_missing_source_halts_immediately (tests.txt line 25) verifies this behavior. Retrying is futile because the source data is genuinely absent, not a transient extraction error. Escalating immediately saves computational cost versus retry attempts that will fail identically.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> Policy POL-2025-001 in routing_decisions.json shows "decision": "human_review" with "reviewer_disagreements": ["coverage_limit", "deductible"]. Despite confidence scores of 1.0 for coverage_limit and 1.0 for deductible, the independent reviewer disagreed on both fields. If confidence alone had been trusted, this would have auto-approved, missing the actual reviewer objection.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> From calibration-report.txt line 4: "umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865" — confidence 93% but accuracy 0%. Overall brier=0.291 (line 5). By policy_type×field slicing, we catch that umbrella/exclusions is systematically wrong (0% accuracy) despite high confidence, while auto/premium_amount is perfect (conf=0.95 acc=1.00). An aggregate metric would average these and hide the catastrophic cell.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed (02-mortgage-extraction/tests.txt) |
| Document run | extract-run.txt (appraisal document) |
| Classified type | appraisal |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From discrepancy-run.txt (income_sum_mismatch.txt): "consistent": false, discrepancies show "calculated": 9642.17 vs "stated": 10892.17 (delta -1250.0). Tool use enforces valid JSON structure but doesn't validate semantic correctness of values. The sum validator catches mathematical inconsistency that JSON schema cannot. Tool use cannot catch: a valid JSON number for the wrong semantic meaning. Validator cannot catch: a wrong value that adds up.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From missing-field-run.txt line 16-20: "bonus_monthly": null, "bonus_ytd": null, "commission_monthly": null. The null values appear for fields missing from the income statement. The schema design uses nullable union from schema.py, permitting null outputs. This is better than invented values because it distinguishes "field existed but value is zero" from "field not present in source data".

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> From extract-run.txt: "gross_living_area_sqft": 2400 (line 19). Source appraisal document states "about 2,400 sq ft" but extraction produces integer 2400. Normalizing at extraction time enables immediate schema validation, allows HITL validators to work with consistent types, and makes the validator catch range/consistency errors early rather than downstream code discovering format mismatches.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed (03-supply-chain/tests.txt line 45) |
| Briefing file | investigation-run.txt |
| Section the conflict landed in | Contested |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> From investigation-run.txt (Contested section): on_time_delivery_rate: supplier_audit reports 95.0% (as of 2026-04-10) vs logistics reports 78.0% (as of 2026-04-05). Preserving both values lets readers see the 17% gap suggests either a recent improvement, measurement methodology difference, or data quality issue. A single reconciled number would hide the significant disagreement and uncertainty.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From timeout-run.txt (line 4): "Sources unavailable: logistics unavailable (timeout)". The late_shipment_count field moves to Incomplete section with "missing source: timeout reading logistics". Timeout is treated as failure (source unreachable) distinct from "empty result" (source reported no data). The run finishes because the system annotates missing data rather than aborting — it returns a partial briefing with marked gaps, letting downstream work with incomplete but accurate information.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> From investigation-run.txt (Well-Established section): defect_rate_ppm reports 180.0 ppm from supplier_audit (2026-04-10) and 190.0 ppm from internal_quality (2026-04-08). These appear near each other with dates differ by 2 days. The 10 ppm difference could reflect natural variation or operational change between measurement dates. Without date stamps, these would appear contradictory. Dates let the system distinguish "measured differently at different times" from "true contradiction in simultaneously observed data".

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> System 1 (routing_decisions.json): POL-2025-001 shows confidence 1.0 for coverage_limit and deductible, yet the independent reviewer disagreed on both fields. Trust alone would auto-approve. The reviewer caught something confidence predicted was perfect. This one decision demonstrates the whole discipline: validation means external verification, not confidence scores.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> System 1 mattered most. The calibration-report.txt line 4 shows umbrella/exclusions at 93% confidence but 0% accuracy — a catastrophic gap. In insurance, high-confidence extraction of exclusions that's actually wrong could cause claim denials or compliance failures. This gap would only appear through external validation (calibration against ground truth); confidence score alone suggests correctness.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> Real workflow: extracting project metadata from unstructured proposal documents to populate a contract database. I'd reach for validated retry with escalation first: parse structured data (scope, budget, timeline), validate completeness, retry on missing required fields (project fails gracefully with clear errors), then escalate genuinely ambiguous proposals to humans. Instrument: (1) track escalation rate (should be <5% of normal run), (2) measure validation failure reasons to detect document format changes, (3) sample escalated cases to monitor false negatives in the validator.
