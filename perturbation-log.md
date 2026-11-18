# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** POL-2025-001.txt: blanked the "Policy Type: " field (deleted value, kept label)
- **Command I ran:** `policy-extractor pipeline /tmp/policies_perturbation/ --routing-out /tmp/routing_perturbation.json`
- **What I predicted:** The extraction would fail or generate null for policy_type, causing escalation since policy_type is required.
- **What actually happened (paste the key output line):** Pipeline output shows "escalations": 1 in the final summary (from routing_perturbation.json). Despite the blank field, the model inferred "Personal Automobile" with 100% confidence and routed to human_review with reviewer_disagreement instead of escalation. The escalation came from a different policy (validation_failed in logs indicates POL-2025-010 had the actual escalation).
- **How this differs from the unperturbed run:** The unperturbed run produced routing decisions with 1 auto_approve, 7 human_review, and 1 spot_check, while the perturbed run produced 0 auto_approve, 8 human_review, and 1 spot_check. Policy POL-2025-001 routed to human_review in both runs, so the lost auto_approve represents live-model variance rather than the effect of the perturbation. The blanked field did not cause escalation because the model inferred "Personal Automobile" for policy_type at 100% confidence, demonstrating that blanking a required field does not guarantee extraction failure when the LLM confidently fills the gap from context.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** income_sum_mismatch.txt copied to /tmp: changed "TOTAL MONTHLY EARNINGS" from 10,892.17 to 9,642.17 (correcting the stated total to match calculated sum)
- **Command I ran:** `mortgage-extract /tmp/income_sum_mismatch.txt --mode record`
- **What I predicted:** Extraction would succeed and the validator would now show consistent=true with no discrepancies, since the stated total matches the calculated total.
- **What actually happened (paste the key output line):** From perturbation-run.txt line 24-26: "validation": {"consistent": true, "discrepancies": []}. The stated_monthly_total (line 21) shows 9642.17, matching the sum of individual income components. The discrepancy vanished.
- **How this differs from the unperturbed run:** Unperturbed (discrepancy-run.txt): extraction succeeded with consistent=false showing calculated 9642.17 vs stated 10892.17 (delta -1250.0). Perturbed: consistent=true with no discrepancies. This demonstrates the validator catches mathematical inconsistency when present and passes when the stated total is correct.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** No code change; used command-line flag `--simulate-timeout` to inject a failure on the logistics source
- **Command I ran:** `supply-chain-investigate meridian --simulate-timeout --offline`
- **What I predicted:** The logistics source would fail to load, causing fields that depend on logistics (late_shipment_count) to be missing or moved to incomplete section. The system would continue and produce a partial briefing rather than abort.
- **What actually happened (paste the key output line):** From timeout-run.txt line 2: "Sources unavailable: logistics unavailable (timeout)". The late_shipment_count moved to Incomplete section with "missing source: timeout reading logistics". The on_time_delivery_rate became single-source (only supplier_audit). Contested section is now empty (because the conflict it had was with logistics data).
- **How this differs from the unperturbed run:** Unperturbed (investigation-run.txt): late_shipment_count is in Well-Established (single source: logistics). on_time_delivery_rate is in Contested (conflict between supplier_audit 95% and logistics 78%). Perturbed: late_shipment_count missing entirely, on_time_delivery_rate becomes single-source uncontested. The system gracefully degraded: it marked missing data rather than fabricating it, and the briefing is accurate but incomplete.
