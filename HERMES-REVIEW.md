# Hermes Agent perspective

## Main recommendation

The strongest practical fit is a typed classification/routing layer around decisions the pipeline already makes with generative prompts and hand-written parsers. Highest-value candidates:

1. **Normalize claim ownership and claim type** with one versioned schema instead of maintaining overlapping prompts, labels, and parse behavior.
2. **Pre-screen ambiguous claims and implicit forecasts** before expensive deep QA; abstentions go to a human/review queue.
3. **Triage the pending human review queue** by suggesting a category/action and confidence while keeping the person as decision-maker.

I originally highlighted `scripts/claim_gate.py` as the clearest low-friction entry point because it already classifies prediction, prescription, fact, and conditional claims. After the deeper repository review, I agree ownership/source attribution is likely the larger system-wide benefit: the same decision appears in multiple stages, with inconsistent labels and consequences.

## Suggested first implementation sequence

- Run SystemOne in **shadow mode** against current claim-gate and ownership decisions; log outputs in sidecar records keyed by claim/prediction ID and schema version.
- Prefer a schema that supports explicit uncertainty (`insufficient_context` / `unclear`) and source evidence spans. Keep the existing source-grounded QA path for uncertainty.
- Measure agreement and precision against hand-labeled examples. Only then route high-confidence cases automatically; do not let model self-reported confidence alone authorize removal or merging.
- Keep application code responsible for actions and fallback behavior. On timeout, malformed response, or service outage, preserve the claim and send it to review; never fail closed by deleting it.

## Do not delegate to SystemOne

Do not use it to determine truth/resolution of geopolitical, financial, or other time-sensitive predictions. Those decisions need current evidence and the existing evidence/reasoning workflow. Keep deterministic policy—not the model—in charge of publish holds, removals, and final verdicts.

## Why this is promising

The component's structured output and probabilities fit the pipeline's repeated bounded classification tasks better than asking a general model to emit free-form labels. Benefits could include consistent labels, cheaper first-pass triage, and fewer malformed-output edge cases. These are hypotheses to validate, not established accuracy improvements.
