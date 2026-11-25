# Perturbation Log

**Author:** VEERA SATYAVANI RAJANA  
**Date:** November 25, 2026  

This log documents controlled failure experiments across each of the three systems. For each system, a targeted perturbation was introduced to verify that the corresponding safety mechanism and fallback logic fire as designed.

---

### System 1 — Validated, Routed Insurance Policy Extraction Pipeline

- **Change I made (file + what I changed):**
  Due to running offline without an active `ANTHROPIC_API_KEY`, I executed the targeted unit test designed specifically for unrecoverable omission failures: `tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately`. This test evaluates the retry engine by feeding an extraction result where `"endorsements": None` (simulating a policy document lacking Schedule A endorsements) into `extract_with_retry`.
- **Command I ran:**
  `.venv\Scripts\pytest.exe tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately -v`
- **What I predicted:**
  When a required field is genuinely missing from source text, retrying the model is futile. I predicted that `extract_with_retry` would recognize this as an unrecoverable absence and immediately abort to a `RetryFutileEscalation` after a single call, preventing pointless token spend and hallucinated guesses.
- **What actually happened (paste the key output line):**
  `PASSED [100%]`
  The function halted on the initial attempt and returned an instance of `RetryFutileEscalation` with attributes:
  `result.field == "endorsements"` and `result.detected_pattern == "endorsements_absent"`.
  Because the underlying `RecordedClient` had only one mock response queued, the test passing confirms that the system did not initiate any subsequent retry attempts.
- **How this differs from the unperturbed run:**
  In contrast, recoverable syntax or formatting anomalies (such as those tested in `test_ac_01_03_format_failure_retries_with_error_appended`) trigger the standard retry loop where validation feedback is attached to the conversation history to guide self-correction. Incurable data omissions bypass retries entirely, avoiding hallucinations when the source is silent.

---

### System 2 — Schema-Enforced Two-Pass Mortgage Document Extraction System

- **Change I made (file + what I changed):**
  I evaluated the pipeline against the perturbed earnings statement in `fixtures/documents/income_sum_mismatch.txt`, where the itemized earnings lines ($5,416.67 base + $2,140.00 commission + $1,250.00 bonus + $385.50 overtime + $450.00 other = $9,642.17) mathematically conflict with the printed `TOTAL MONTHLY EARNINGS 10,892.17`. I contrasted this directly against the clean appraisal document `fixtures/documents/appraisal_informal_sqft.txt`.
- **Command I ran:**
  `.venv\Scripts\python.exe -m mortgage_extractor fixtures/documents/income_sum_mismatch.txt --mode replay`
  versus
  `.venv\Scripts\python.exe -m mortgage_extractor fixtures/documents/appraisal_informal_sqft.txt --mode replay`
- **What I predicted:**
  The LLM extraction layer would accurately parse the stated text values without modifying them. Subsequently, the independent mathematical validation layer would compute the true total from individual items, detect the $1,250 arithmetic discrepancy, and flag the record as inconsistent.
- **What actually happened (paste the key output line):**
  `"consistent": false`
  Discrepancy record:
  `{"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0}`
  (Full output captured in `02-mortgage-extraction/discrepancy-run.txt`). Conversely, the clean appraisal record produced `"consistent": true, "discrepancies": []`.
- **How this differs from the unperturbed run:**
  Both documents adhere to the strict JSON Schema enforced by tool choice. However, the perturbed document exposes the necessity of secondary arithmetic validation: syntactically valid JSON can still harbor severe internal numerical contradictions that only dedicated logic checks can catch.

---

### System 3 — Supply Chain Risk Multi-Source Synthesis

- **Change I made (file + what I changed):**
  I introduced a simulated network failure using the coordinator's `--simulate-timeout` parameter. This triggers a timeout on the `logistics` data reader while keeping the remaining three readers (`supplier_audit`, `internal_quality`, `industry_news`) operating normally.
- **Command I ran:**
  `.venv\Scripts\python.exe -m supply_chain_risk meridian --offline --simulate-timeout`
- **What I predicted:**
  Rather than unhandled exception propagation aborting the script, the resilient coordinator would catch the timeout, annotate the logistics source as unavailable, categorize its missing metrics under `## Incomplete`, and successfully render the remaining findings.
- **What actually happened (paste the key output line):**
  The rendered briefing headers indicate:
  `> Sources unavailable: logistics unavailable (timeout)`
  And under the `## Incomplete` section:
  `### late_shipment_count  [missing source: timeout reading logistics]`
  `- missing source: timeout reading logistics`
  The command exited with code 0 and generated the executive report (`03-supply-chain/timeout-run.txt`).
- **How this differs from the unperturbed run:**
  In the healthy baseline run (`03-supply-chain/investigation-run.txt`), `late_shipment_count` appears under `## Well-Established` with a confirmed value of `11.0 shipments — logistics`, and `on_time_delivery_rate` is classified under `## Contested`. When timed out, the missing feed moves strictly into `## Incomplete`, ensuring stakeholders see partial evidence without mistaking a communication blackout for a clean audit.
