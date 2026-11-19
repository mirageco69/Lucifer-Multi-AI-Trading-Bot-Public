# Change notice, 19 November 2026: two AI input fixes made without the Solera owner's sign-off

This note is public so that the Solera owner, and anyone else following the project, can read what was changed, why, and how to undo it. It is deliberately limited to the two changes below. It contains no account details, results, or security findings.

## What changed

Both changes are switched by one setting each, read on every debate, so each can be undone in seconds without a restart.

| # | Change | Setting | Files |
|---|---|---|---|
| 23 | The **adversarial** agent runs on the system prompt it was trained with, instead of the longer prompt in the editable prompts file | `debate.use_training_prompts: ["adversarial"]` | `services/lucifer_ai/lucifer_inference.py`, `services/adversarial_shadow.py` |
| 24 | The **chart summary** shown to the agents is built from true hourly candles, with the key readings first | `debate.chart_summary_v2: true` | `services/ai_context_builder.py` |

Nothing else changed: no retraining, no change to the dataset, the checkpoint, the encoder, the field set or the schema.

## Why

**Entry 23.** The live adversarial check was not running: in the last 24 hours before the change, 318 of 351 crypto debates (91%) carried the fallback verdict, and a fallback reads downstream as "no objection". The live prompt and the training prompt share 10 of 50 system tokens. On a replay with the same checkpoint the training prompt produced 6 of 6 valid verdicts and the live prompt 0 of 6. The cause is that the live path never fills the pre-encoded prompt cache that the encoder is built to use, so it falls back to the prompts file. All 15 agents have a prompt mismatch of this kind; only the adversarial agent has replay proof, so only it was switched.

**Entry 24.** The chart agent was being given 42 four-hour candles labelled as hourly (the candle fetcher had no exchange plugin to use, so it fell back to a free data source that returns four-hour bars for a seven-day request). As a result the moving-average stack was "unknown" in about three debates in four, the volume-weighted price was always missing, and the encoder's 120-character cut hid the SuperTrend and Bollinger lines every time. Measured over 4,621 crypto debates, the chart verdict carried no information about the next 24 hours. The new summary reads the newest 100 hourly bars the chart collector already stores, falls back to the old path if they are missing, stale or not hourly, and puts one short line of the deciding readings at the very start.

## First result (the first three debates after the change)

All three returned a real adversarial verdict (severity MEDIUM, HIGH and MEDIUM, confidence adjustments of -1, -2 and -1), where 91% of debates had carried the fallback before. The one crypto debate in that sample shows the new chart summary: the first line now reads like `EMA mixed | ST DOWN | %B 0.874 sq n | vsEMA21 +0.9% | mom5 +2.1%`, followed by the full text built from 100 true hourly candles. Three debates is only a first sign, not evidence; the fallback share, schema validity and the chart verdict against the price that follows will be read over several days.

## Why we went ahead without waiting

The shared-set rule (Non-Negotiable 10 in the project file, agreed by both owners on 2 November 2026) says changes to what a live debate sends need both owners' written agreement first. The LUCIFER owner decided on 19 November 2026 not to wait, for three reasons: the system has been underperforming for weeks, an adversarial check that does not run is part of that, and there had been no reply from the Solera owner to the open entries since 18 November. This is recorded in the change log as a **departure from the rule, not as an agreement**. The Solera line in the log is left empty, and the automatic shared-surface check still reports the one changed file (`lucifer_inference.py`) on purpose, so the departure cannot be missed.

We expect the fixes to help, but we are not claiming they will fix the underperformance on their own. Separate measurements point mainly at the quality of entries, and these two changes only correct two inputs.

## If you disagree

Say so, in any channel. The LUCIFER owner will review it together with you and change or revert either entry, including by simply switching the setting off. The log entries (`docs/SHARED_SET_CHANGE_LOG.md`, entries 23 and 24) name the evidence to watch and the condition for reverting: the adversarial fallback share should fall, no agent's valid-schema share should fall, and the chart verdict should be compared with the price move that follows.

## What we still need from the Solera side

1. Whether your production path already runs the training prompts (your encoder fills the prompt cache at start-up, so it appears to).
2. Which exit-planner training text your checkpoint was trained with (the two repositories differ).
3. Whether your chart summary shows the same four-hour candle problem (your candle fetcher is identical, which suggests it does).
