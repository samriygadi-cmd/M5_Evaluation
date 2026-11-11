# M5: Evaluation Harness: Berth Rescheduling and Conflict Resolution

Research question (M2): does a multi-agent berth rescheduling system resolve more conflicts with lower added delay than a single-agent baseline?
This milestone builds the measurement instrument: a golden set of cases with ground truth, automated scoring, and baseline scores with their spread across three seeds. The multi-agent treatment is evaluated in M6.

## How to run

Everything is in one notebook, `M5_baseline.ipynb`. It imports no other file and needs no other file: the golden set and the 450 saved runs are stored inside it.

**On Google Colab:** upload the notebook (**File → Upload notebook**) and choose **Runtime → Run all**. No Drive folder and no API key are needed.

**On your own machine** (Python 3.9 or later, with `openai`, `numpy`, `scipy` and `matplotlib`): open the notebook and run all cells.

The data is a compressed archive written as text in cell 1b. Cell 3 unpacks it into a folder `M5_evaluation_harness` (on Colab, `/content/M5_evaluation_harness`, which is deleted when the session ends), skips the runs that are already saved, so it makes no API call, and writes a report identical to `results/report/M5/`. A DeepSeek key (`DEEPSEEK_API_KEY`, in Colab Secrets or the environment) is needed only if a run is deleted and must be run again.

The `golden_set/` and `results/` folders in this package hold the same data as files, so they can be inspected without running anything.

## Files

| File | Purpose |
|---|---|
| `M5_baseline.ipynb` | The complete harness: cleaning, conflict rule, golden set, scoring, tests, baselines, runner and report, with the golden set and saved runs embedded |
| `M5_Report.docx` | The written M5 report |
| `golden_set/golden_set.json` | The frozen golden set; `golden_set.sha256` makes the notebook refuse a changed set |
| `escales_arribades_2025 3.csv` | Port of Barcelona 2025 port calls; needed only to rebuild the golden set |
| `results/run_config.json` | Model, temperature, seeds, prices, prompts and golden-set hash used for the runs |
| `results/runs/` | One trajectory file per run: every message, reply, proposal, token count and timing (3 conditions × 3 seeds × 50 cases) |
| `results/report/M5/` | Generated report, figures and per-case and per-seed scores |

## The measurement instrument

### Golden set: 50 cases

| Tier | Cases | Source | Why it is there |
|---|---|---|---|
| easy | 16 | real | One conflict, under 6 vessels |
| medium | 13 | real | One conflict among 6–14 vessels at the quay |
| hard | 11 | real | Several conflicts or at least 15 vessels, up to 20 |
| negative | 8 | 6 real, 2 derived | Vessels share the quay on different modules, or touch exactly. Correct answer: move nothing. This is the M4 baseline's main failure |
| stress | 2 | synthetic | Evaluation document abnormal case 9: 8 and 12 heavily overlapping vessels |

The 40 real conflict cases are the 40 connected conflict clusters in the cleaned data, so **all 47 reference conflicts are covered**. Neighbouring vessels at the same berth are added, up to 20 per case, the size of the M4 batches. The set holds 106 conflict pairs, because stress cases and some windows add pairs.

Ground truth per case:
- **Conflict pairs:** two independent implementations (sweep and brute force) must agree.
- **Optimal total delay:** exact MILP (HiGHS), proven optimal on all 50 cases, and re-checked by exhaustive search on every case with at most 7 vessels.
- **FCFS delay:** the non-agent reference.

The input-validation and proposal-rejection cases of the evaluation document (5 normal, 9 abnormal) are tests in section 5 of the notebook, because cleaning happens before any system sees the data.

### Metrics

Everything is rule-based. No LLM judge is needed, because a correct answer can be computed (Module 7).

| Metric | Definition | Level |
|---|---|---|
| **Solved** (primary) | No constraint violation, no conflict left, and the run did not fail | Outcome |
| CRR | `(initial − remaining) / initial × 100`, summed over cases. Can be negative. Also reported on real cases only, because the 2 synthetic stress cases hold 54 of the 106 pairs | Outcome |
| Strict CRR | The same, with invalid proposals rejected, so nothing counts as resolved | Outcome |
| Remaining / new conflicts | Conflict pairs in the final schedule / those not in the original | Outcome |
| AAD | Added delay per moved vessel, in minutes | Outcome |
| Schedule deviation | Mean absolute ETA change over all vessels, in minutes | Outcome |
| Delay ÷ optimal | Total delay over total MILP optimum on solved cases, plus the per-case median. A mean of per-case ratios is not used, because a case with a 1-minute optimum dominates it | Outcome |
| Detection P / R / F1 | Reported pairs against the reference pairs | Outcome |
| 10 success criteria | The list in the evaluation document | Outcome |
| **Rescheduling attempts** | Proposals until the run stopped | Trajectory |
| **Attempts until first solved** | First proposal that passes the scorer | Trajectory |
| **False termination** | The system claimed it was finished, but the case is not solved | Trajectory |
| **Conflicts by attempt** | Conflicts left after each proposal | Trajectory |
| LLM calls, invalid replies retried, tokens by role | From the logged messages | Trajectory |
| Latency, tokens, cost per case, solved per 1K tokens | From the logs | Cost |

Why "solved" is primary: AAD alone rewards a system that gives up, since moving nothing has no delay. Delay is therefore also compared only on case-seed pairs that both systems solved.

Do-nothing is a condition, as Module 6 advises. It solves 16% (the negative cases) and passes 6.64 of the 10 success criteria on average, so the 10-criteria score should not be the headline.

### Seeds and variance

Seeds are 11, 23 and 47. A seed sets the row order given to the system, identically for every condition; the temperature (0.2, the M4 baseline used 0) adds sampling noise. A temperature of 0 would hide most of the spread that the study must report (Modules 2 and 6).

Reports give mean ± sample SD with min–max across seeds.

### Conditions

| Condition | Role |
|---|---|
| Do-nothing | Returns the schedule unchanged: what the metrics give a system that does no work (Module 6) |
| Non-agent (FCFS) | The M4 rule-based model, plain code |
| Single agent | The baseline: one LLM agent detects and resolves conflicts, and may review its own answer up to 3 times. No tools |

Changes from the M4 single agent, so that the M6 comparison differs in architecture only: the agent can review and revise its answer, each case is one self-contained table, and the temperature is 0.2.

## Results

Model `deepseek-flash` (V4.1-Flash-1017), run on 2 November 2026. Cells show mean ± sample SD across the three seeds. Full tables are in `results/report/M5/report.md`.

| Metric | Do-nothing | FCFS | Single agent |
|---|---|---|---|
| **Solved cases (%)** | 16.0 ± 0.0 | 100.0 ± 0.0 | **36.7 ± 4.2** |
| Solved, cases with conflicts (%) | 0.0 ± 0.0 | 100.0 ± 0.0 | 34.9 ± 3.6 |
| Solved, cases without conflicts (%) | 100.0 ± 0.0 | 100.0 ± 0.0 | 45.8 ± 7.2 |
| CRR, real cases (%) | 0.0 ± 0.0 | 100.0 ± 0.0 | 5.8 ± 19.5 |
| CRR, invalid rejected, real cases (%) | 0.0 ± 0.0 | 100.0 ± 0.0 | −1.3 ± 4.0 |
| New conflicts (pairs) | 0.0 ± 0.0 | 0.0 ± 0.0 | 36.0 ± 9.5 |
| Runs with no constraint violation (%) | 100.0 ± 0.0 | 100.0 ± 0.0 | 55.3 ± 3.1 |
| Total delay ÷ total optimal, solved cases | n/a | 1.32 | 1.92 ± 0.77 |
| Detection precision | n/a | 1.000 | 0.243 ± 0.014 |
| Detection recall | 0.000 | 1.000 | 0.852 ± 0.054 |
| Success criteria passed (of 10) | 6.64 | 10.00 | 7.07 ± 0.12 |

**Trajectory-level metrics (single agent):** 2.23 ± 0.05 rescheduling attempts per case; false termination in 44.0% ± 2.0 of runs; solved cases by attempt 37.3%, 36.7%, 36.7%, so self-review did not help.

**Cost and latency (single agent):** 4,429 ± 195 tokens and 5.5 ± 0.2 seconds per case. The actual billed cost of the full run, including a 3-run trial, was 0.13 USD.

**Findings:**
- Plain code outperformed the single agent on solved cases, validity and cost. FCFS solved every case with total delay 1.32 times the optimum.
- The single agent over-reported conflicts (precision 0.24) and solved fewer cases without conflicts than doing nothing (45.8% against 100%).
- On real cases its net conflict resolution was near zero (5.8% ± 19.5), because new conflicts cancelled most of what it resolved.
- It solved 77.1% of easy cases, 10.3% of medium cases and none of the hard cases.

## Limitations

1. **Closed-world cases.** A case ends at 20 vessels, so a delay cannot spill past the window.
2. **Model alias.** `deepseek-chat` is an alias; the API returned `deepseek-flash`. A hosted model can change without a new name, so the model name is logged per call.
3. **Few seeds.** Three seeds give a rough spread only.
4. **Planned data.** These are potential conflicts in published schedules, not confirmed operational conflicts.
5. **No tools.** The single agent does date arithmetic and conflict checks itself, as in M4.
6. **Synthetic and derived cases.** 4 of the 50 cases are not fully real; they are labelled by `provenance`. The 2 stress cases hold 54 of the 106 conflict pairs, so real-case figures are reported separately.
