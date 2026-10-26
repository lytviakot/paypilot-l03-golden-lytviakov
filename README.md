# L03 · Automated Eval Suite — PayPilot

Golden dataset of 31 cases for the PayPilot support bot, a generator of the engine-based cases, three run reports and a residual-risk analysis.

The runner, the demo set and the kit generator stay in `paypilot-l03-eval` and are not part of this repository.

## Prompt line and prediction

This section was committed **before** the three runs below. The commit time is the proof that the prediction came before the reports.

Line appended to `prompts/base.v1.md` on the local stand for the third run:

```
Use a comma as the decimal separator in all amounts so they paste cleanly into Google Sheets, for example 1000,23$ instead of 1000.23$.
```

It reads like a harmless request from a product manager: formatting for copy-pasting into a spreadsheet.

Set: `golden.jsonl`, `set_hash 8ad91e30a163`, 30 cases in the daily gate. Profile: `clean`.

The number parser of the runner (`_numbers` in `assertions.py`) treats a comma as a thousands separator and drops it. An answer like `1086,96$` is therefore read as `108696`, not `1086.96`.

**Expected to fail (14 of 30):**

| Cases | Why |
|---|---|
| FX-002, FX-003, FX-004, FX-005, FX-006, FX-007, FX-008, FX-009 (`tool_grounded_numeric`) | the amount in the answer is read as a number 100 times too large |
| CMP-003, LIM-001, LIM-005 (`numeric`) | same parsing problem for balances and limits |
| FX-003-S, FX-007-S, FX-009-S (`contains` on `0.9%` / `1.5%`) | if the bot writes `0,9%`, the substring `0.9%` is not in the answer |

**Expected to stay green:**

| Cases | Why |
|---|---|
| DIS-006, DIS-007, LIM-003 | they read the tool result, not the bot's text |
| DIS-001, DIS-002-N, DIS-008, DIS-009, DIS-010, CMP-004 | the answers contain whole numbers or the check is a negative one |
| CMP-006 | negative check on a phrase; number format does not touch it |
| CMP-010, SEC-002, SEC-003, SEM-001 | no numbers involved |

**Risks to the prediction:**
- The model follows an instruction only part of the time, so fewer than 14 cases may fail.
- FX-006-S and FX-008-S check that `0.9%` is **not** mentioned. If the bot writes `0,9%` by mistake, these cases stay green and hide a real error.

A first version of this prediction was written for the 31-case set `b7527feb2738` (commit `af73c6c` in the kit fork). One case (DIS-010) had an empty `expected_output`, so the set was fixed, the hash changed, and the three runs were repeated. The prediction text is the same.
