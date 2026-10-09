# SystemOne in SeerScore: workflow review

A documentation-only review of where a typed, local SystemOne decision component could help the SeerScore prediction pipeline. It contains independent perspectives from Hermes Agent and Cursor Agent (Claude Opus 5.5, Medium), based on repository inspection; it does not contain integration code.

- [Hermes Agent's perspective](HERMES-REVIEW.md)
- [Cursor Agent / Opus review](OPUS-REVIEW.md)

## Bottom line

Use SystemOne for bounded classification and routing decisions—not for deciding whether a real-world prediction came true. Start in shadow mode; keep ambiguous and failed classifications in review rather than deleting claims. Validate schemas, thresholds, and model behavior on labeled examples before allowing any automated destructive action.

## Scope

The reviews refer to paths within the SeerScore repository (`veracity2/`), which is not included here. File paths and line numbers are pointers into the inspected checkout and may drift as it changes. Latency estimates are explicitly labeled as unmeasured hypotheses. No private endpoint, credential, or model-host address is included.
