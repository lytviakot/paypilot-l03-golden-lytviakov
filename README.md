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

## Run

The runner, the stand and the kit live in `paypilot-l03-eval`. Put `golden.jsonl` into `sets/` of the kit as `golden.jsonl`, `generate_golden.py` into the kit root, and run:

    docker compose run --rm eval --set golden --profile clean

`set_hash`: `8ad91e30a163`

Agent model: `claude-haiku-5-5` (stand default, provider `anthropic`). The judge is not used in these runs.

Regenerate the generated cases (the output is selected by hand, see "Generator"):

    docker compose run --rm eval python generate_golden.py > sets/_golden_generated.jsonl

## Complaints

| Complaint | Case | Customer / operation | Oracle |
| --- | --- | --- | --- |
| C-11 | DIS-008 | CUS-0009, TX-0902, `fraud_card_not_present`, window 120 days | engine (`policy.DISPUTE_WINDOWS_DAYS`) |
| C-12 | DIS-009 | CUS-0002, TX-0201, `duplicate_charge`, window 60 days | engine (`policy.DISPUTE_WINDOWS_DAYS`) |
| C-03 | DIS-010 | CUS-0004, TX-0402 (TechMart, 63 days old, window 60): the bot must not offer a dispute | engine (`disputes.check`) |
| C-09 | LIM-005 | CUS-0010, monthly remaining limit | engine (`limits.status`) |
| C-10 | CMP-010 | CUS-0006, TX-0601: the refusal must not mention the compliance review | human |
| C-17 | TON-001 | angry customer charged twice: acknowledge, no "as per our terms", next step | human |

C-11 and C-12 look contradictory (120 vs 60 days) but are both right: the dispute engine gives 120 days for `fraud_card_not_present` and 60 for `duplicate_charge`.

C-03 and C-09 had no customer id or operation in the complaint. The context came from the seed: "the TechMart guy" is CUS-0004 with TX-0401 / TX-0402, and "Jonas from the GBP account" is CUS-0010.

Not taken into the set: C-02, C-15, C-16, C-20 (not reproducible or not about the agent, see `residual-risk.md`).

## Generator

`generate_golden.py` is a copy of the kit generator `generate_from_engines.py`. Added:
- four edges for the free FX allowance, which is all-or-nothing: FX-006 (CUS-0007, 1000 EUR, exactly on the limit), FX-007 (1001 EUR, one euro over), FX-008 (CUS-0001, tier 1, 380 EUR: 120 used + 380 = 500, the allowance is used up to the cent), FX-009 (381 EUR, one euro over)
- a `-S` twin for each edge: which spread applies
- four complaint cases (DIS-008, DIS-009, DIS-010, LIM-005)
- `context.clock` on every case and `added_in: hw2` on the new ones

Edges added: the allowance rule changes behaviour exactly at its limit, so each pair sits on both sides of it. A bot that charges the spread only on the part above the limit (D21) is wrong only on the "over" side.

Selection criterion: the demo set (14 cases) already covers the base of FX, limits and disputes. From the generator I took only what the demo set does not have: the edges and the cases that come from complaints. The rest of the kit generator output (about 20 cases) was left out, because it repeats what the demo set checks and adds cost without adding sensitivity.

Findings:
- Tier 1 has a sharper cliff than tier 2. At 380 EUR the customer receives 413.04 USD, at 381 EUR only 407.92 USD: one euro more costs 5.12 USD, because the 1.5% spread lands on the whole amount. For tier 2 the same step is 1086.96 to 1078.25 USD (8.71 USD at 0.9%, on a bigger base).
- The daily limit remainder of CUS-0010 is 95,086 EUR, which is the "about 95k" in complaint C-09. The monthly remainder is 964,666 EUR. So the complaint is the bot giving the daily figure as the monthly one.

## Runs

| Run | Profile | Result | Report |
| --- | --- | --- | --- |
| Baseline | `clean` | 30/30 | `reports/1-baseline-clean.json` |
| Defective system | `lesson-03` | 18/30 | `reports/2-defects-lesson-03.json` |
| Prompt change | `clean` + line | 16/30 | `reports/3-prompt-comma.json` |

The case with `gate: release` (TON-001) is not in the runs, so M (30) is smaller than the set (31).

All three reports have `set_hash 8ad91e30a163`. The files are copies of the raw reports of the runner.

Line in the prompt: `Use a comma as the decimal separator in all amounts so they paste cleanly into Google Sheets, for example 1000,23$ instead of 1000.23$.`

Prediction before the run: see "Prompt line and prediction" above.

Commit with the prediction: `65334dc`

What failed (14 cases): FX-002, FX-003, FX-004, FX-005, FX-006, FX-007, FX-008, FX-009, CMP-003, LIM-001, LIM-005, FX-003-S, FX-007-S, FX-009-S. Deviation from the prediction: none, the 14 predicted cases are exactly the 14 that failed. The reports show the mechanism: in CMP-003 the answer `5200,75` is read as `520075.0`, in LIM-001 `964666,00` is read as `96466600.0`.

Control run after the prompt was restored (`--only FX-004`): PASS.

Defects of the `lesson-03` profile. The profile switches all five on at once, so the attribution below comes from the numbers in the answers, not from isolated runs:
- **D19** (wrong window for a reason code): caught by DIS-006 (the tool says `eligible = True` for the 63-day-old TX-0402, expected False). The text cases with the window length (DIS-001, DIS-008, DIS-009) stay green, because they check what the bot says about the window, not the tool verdict.
- **D20** (spread of the neighbouring tier): caught by FX-003 (expected 6463.04, got 6423.91, which is a 1.5% spread instead of 0.9%), FX-003-S and FX-005.
- **D21** (spread only on the part above the allowance): caught by FX-007 (expected 1078.25, got 1088.03, almost no spread) and FX-009 (expected 407.92, got 414.12). FX-002 and FX-004 fail because of D20 and D21 together.
- **D22** (daily remainder shown as monthly): caught by LIM-003 only, which reads the tool payload (95,086 instead of 964,666). LIM-001 and LIM-005 (the bot's text) stay green, because the bot recomputes the remainder from the other fields.
- **D26** (compliance hold not seen by the bot): caught by DIS-007 (tool verdict `eligible = True`). CMP-010 (the refusal must not name the compliance review) failed on the same profile in an earlier run of the same set, but passed in this one, so it is not a reliable catch.
- Missed: none at the set level. D19, D22 and D26 each depend on one case.
