# System 1 — Offline Execution Note

The `policy-extractor pipeline` CLI expects an active `ANTHROPIC_API_KEY` to connect to the live API endpoints. In accordance with the course capstone instructions ("No API access? Capture `.venv/bin/pytest tests/test_us04_routing.py -v` instead of the live run"), `pipeline-run-OFFLINE-FALLBACK.txt` serves as the official offline replacement for `pipeline-run.txt` and `routing_decisions.json`.

This test suite executes the complete deterministic routing workflow (`route_extraction`, confidence evaluation, independent reviewer reconciliation, and stratified sampling) using offline recorded fixtures. Consequently, the high-level routing summary counts in the reflection brief are recorded as N/A, and questions 1a and 1b are grounded in the specific deterministic test records (`test_ac_01_04_missing_source_halts_immediately` and `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review`).
