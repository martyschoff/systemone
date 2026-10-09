# Cursor Agent review — Claude Opus 5.5 Medium

**Scope:** Read-only review of the SeerScore (`veracity2`) repository for the highest-value places to use a typed local decision component such as Ollama SystemOne. No repository files were changed by the reviewer. Paths and line numbers refer to the inspected checkout and may drift.

## Executive conclusion

The largest benefit is a **shared classifier for claim ownership/source attribution**, replacing several overlapping prompt-and-parse implementations. The second is a **single claim-type gate** with safe abstention behavior. SystemOne should not decide final true/false verdicts; use it for bounded classification and routing, leaving evidence-backed resolution to the existing process.

## Ranked opportunities

### 1. Shared ownership/source classifier — biggest system-wide benefit

Related logic appears in `src/extractor.py` (43–48), `src/pipeline.py` (66–75, 233–253), `scripts/origin_resweep.py` (35–41, 134–143), `scripts/attribution_resweep.py` (10–18, 80–81), `scripts/deepqa_judge.py` (27–47, 141–168), and `src/speaker_extract.py` (246–249), with differing labels and failure behavior.

Suggested schema: `origin` (`own`, `reported`, `quoted`, `assent_only`, `not_found`, `insufficient_context`), `third_party`, a verbatim `evidence_span`, and confidence. Inputs should include claim, transcript excerpt, speaker, date/channel/title, and optionally a wider source-context window. Put it before the existing deep QA path and preserve secondary-judge escalation for removals. Map labels to actions in deterministic code. Insufficient context should trigger source-grounded QA, not deletion.

### 2. Unified claim-type gate with safe abstention

`scripts/claim_gate.py` and `scripts/gate_sweep_pool.py` classify with different models but do not compare outputs. `scripts/classify_worker.py` uses overlapping labels. In `classify_apply.py` (40–56), a failed/missing response can become `UNCLEAR` and then remove a row to the guest pool; the reviewer recommends changing this to preserve the claim and queue it for review. `claim_gate.py`'s docstring describes extraction use, but `src/pipeline.py` does not call it.

Suggested schema: `claim_type` (`prediction`, `conditional`, `prescription`, `fact`, `self_report`, `position`, `personal`, `promo`, `unclear`), `implicit_forecast`, `world_outcome_part`, and confidence. Candidate integration points include after `pipeline.py` line 253, `gate_sweep.py` 37–52, and `classify_apply.py` 40. Abstentions should route to review, not removal.

### 3. Pre-sort the human review queue

The review found 307 entries in `data/implicit_queue.json` marked pending, resolved manually by `_resolve_implicit` in `src/app.py` (289–315). SystemOne could pre-fill a suggested category/action and sort by confidence, with a human retaining the final decision. Human choices can provide labels for later evaluation.

### 4. Due-date and judgment routing

The review found `monte_carlo.py` (110–112) uses the first year in a claim as a due date; `test_eligible_at` is read but not written; and `is_eligible` in `delphicursor_queue.py` (71–98) does not check whether a claim is due. `VERDICT_SWEEP_REVIEW.md` P0.1 also documents future-date bias. A typed date/horizon classifier could populate `test_eligible_at` and help route work, but date interpretation needs evaluation against labeled examples.

Possible schema: resolution (`due`, `not_due`, `open_ended`, `unresolvable`), target date, horizon kind, and whether post-cutoff facts are required.

### 5. Measurement type: safe initial shadow-mode candidate

`src/pipeline.py` (262–267) marks any claim containing a four-digit number—including a year—as quantitative; this affects the site's default objective-only filter (`README.md` 79). A bounded measurability classifier (`quantitative`, `directional`, `vague`) with optional metric/threshold fields is a relatively safe shadow-mode test because the finding need not delete a claim.

### 6. Borderline duplicate checks

`scripts/llm_dedupe.py` (34–53) groups 55 claims in one call and merges them without confidence. For borderline pairs, a typed `same_claim` / `number_drift` / `different` decision could gate merges; only high-precision cases should merge automatically.

## Keep out of SystemOne's scope

- Final correct/wrong verdicts and evidence lookup (`pipeline.py` `llm_assess`, `fact_check_lookup`, `panel_adjudicate.py`): these depend on up-to-date external facts. The review notes `OPERATIONS.md` 74–75 warns about hallucinated panel facts, and `panel_adjudicate.py` line 21 says the 8B pool votes unclear on everything.
- The Monte Carlo persona swarm, whose value is a spread of opinions.
- `marty_verdict`, which is final by rule.
- Open-ended extraction and speaker identification; use it only afterward for closed-set checks.
- Replacing the 120B `qa_approve` judge. The reviewer found the parser defaults `approve` to true when the key is missing (line 456) and approves when the QA call fails (452–459); fix this fail-open behavior in deterministic code.

## Evaluation and operating safeguards

- Do not trust raw self-reported confidence thresholds. Compare against gold labels and calibrate.
- For routing/tagging, require agreement or calibrated thresholds. For destructive actions (removal, guest-pool transfer, merge), require very high measured precision (the review suggests at least 0.95 on gold labels), independent agreement, and a review path for all other cases.
- Begin in shadow mode. Sidecar JSONL should include claim ID, schema version, model, prompt hash, output, and timestamp; compare with existing gate/classification/QA results.
- The review observed only 7 `marty_verdict`s and no resolved queue items; it recommends hand-labeling a stratified set of about 200 origin/type examples before calibration.
- Add endpoint health checks and per-instance concurrency caps. On service/model failure, preserve the claim and route it to review; never delete it.
- Use sidecar files and the repository's `locked_data()` discipline. The reviewer flagged unlocked writes in `gate_sweep.py` (55/60) and `origin_resweep.py` (145–152) as a race risk.
- Version the schema because the current code has multiple overlapping vocabularies.

## Confirmed findings vs hypotheses

The duplicated classifiers, failure/deletion paths, approval-on-failure behavior, regex-based due dates and measurement type, pending queue count, verdict count, and unlocked writes were confirmed in code/data by the reviewer. Accuracy gains, latency, the recency-factor behavior, and whether a small model is accurate enough for ownership remain hypotheses. The latency estimate (1–3 seconds per item) was not measured and must not be treated as a benchmark.

## Additional code issues spotted

The reviewer reports `scripts/panel_adjudicate.py` references `DATA`, `NON_VOTERS`, and `due` (lines 63, 79, 82) without defining them, likely causing `NameError`. It also reports no code writes `test_eligible_at`. These are review findings, not fixes made in this repository.
