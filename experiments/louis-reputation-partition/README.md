# Reputation Under a Split Reporting Network

## Choice and pre-run hypothesis

I chose the built-in `reputation` scenario because its central observer resembles a
surveillance system: individual traders produce observations, the observer aggregates
them, and warnings are sent back to the population.

I changed exactly one supported setting by adding
`failures.network_partition.groups`. The 20 traders are split into two mixed groups of
eight honest and two malicious agents. `observer-0` is in Group A only. The seed, agent
roles, layers, task configuration, duration, metrics, and `message_drop: 0.0` are
unchanged.

Before running the partitioned scenario, I expected some traders to stall when they
selected a peer across the partition because the reference agents have no timeout or
retry. I also expected the observer to receive reports only from Group A, and any
warnings to reach only Group A. I expected reduced activity, but thought the main effect
would be selective observability rather than complete system-wide paralysis.

## Before and after

Both runs used seed 42. The baseline is the unmodified built-in scenario.

| Trace evidence | Baseline | Partitioned |
| --- | ---: | ---: |
| Total events | 665 | 148 |
| Message events (`send` + `receive`) | 620 | 81 |
| Sends | 280 | 53 |
| Receives | 340 | 28 |
| Partition drops | 0 | 25 |
| Trade attempts | 100 | 32 |
| Completed trade rounds | 100 | 12 |
| Traders completing all 5 rounds | 20/20 | 0/20 |
| Cheat messages | 13 | 2 |
| Reports sent toward the observer | 80 | 9 |
| Reports received by the observer | 80 | 4 |
| Bad reports received by the observer | 10 | 0 |
| Warning broadcasts | 3 | 0 |

All 20 traders eventually sent one `trade:` message across the partition. Those 20
messages were dropped, and every initiator then stopped advancing its own rounds. The
other five drops were reports from Group B to `observer-0`.

The registered validators reported:

```text
baseline:
FAIL reputation_scoring - cheaters not reported: {'malicious-3'}
PASS reputation_warnings - 3 warnings issued, 3 needed

partitioned:
FAIL reputation_scoring - cheaters not reported: {'malicious-1'}
PASS reputation_warnings - 0 warnings issued, 0 needed
```

The built-in `delivery_rate` is `receive` events divided by `send` events. It changes
from 1.2143 to 0.5283, but the baseline can exceed 1.0 because one `broadcast` event
creates 20 `receive` events while the metric does not count the broadcast as a send. I
therefore used the event counts above rather than interpreting it as an end-to-end
success rate.

## What surprised me

I expected partial degradation, not all 20 traders to stall. Each group still contained
ten traders, but agents continued choosing peers from the full 20-trader roster. At this
fixed seed, every agent eventually selected an unreachable peer before completing five
rounds. A partition between communities therefore became a liveness failure for every
individual workflow, not just a reporting gap.

The second surprise was the validator evidence. `malicious-2` cheated an honest trader,
which sent `report:1:malicious-2:bad`; the report was dropped before reaching the
observer. Nevertheless, `reputation_scoring` treated that cheater as reported because it
counts `send` events. The observer actually received zero bad reports. `malicious-1`
cheated another malicious trader, which never produces reports, so it was the only agent
the validator called unreported.

## Investigation

I matched every dropped `trade:` event to its sender and found 20 unique initiators. In
[`reputation.py`](../../packages/nest-core/nest_core/scenarios_builtin/reputation.py), an
agent starts its next round only after receiving `deliver:` or `cheat:`. There is no
timeout or retry path, so a dropped request permanently halts that agent's initiated
sequence.

I then matched report correlation IDs. Nine reports were sent, four reached
`observer-0`, and five were dropped. The observer updates scores only in
`on_message`, so it had no evidence of the one bad report. In
[`validators.py`](../../packages/nest-core/nest_core/validators.py), however, both
reputation validators inspect sender-side `send` events. That explains why the scoring
validator counted the dropped bad report and why the warning validator could pass with
`0 warnings issued, 0 needed` even though the runtime observer received no bad reports.

## What I learned

A reporting system needs to distinguish an attempted report from evidence actually
received by the decision-maker. It also needs bounded waiting or retry behavior. Here,
the network partition affected three layers at once: activity stopped, observations were
lost, and warnings disappeared. A green warning validator did not mean the monitoring
system remained healthy; it meant the validator's sender-side reconstruction found no
score at the warning threshold.

The next service I would build is an evidence-delivery monitor that reconciles `send`,
`dropped`, and `receive` events by correlation ID. It would report observation coverage
per community and fail closed when the observer cannot receive enough evidence to make a
reputation decision.

## Reproduce

From the repository root:

```bash
uv sync

# Baseline
uv run nest run reputation -o /tmp/reputation-baseline.jsonl

# Single-setting experiment
uv run nest run \
  experiments/louis-reputation-partition/reputation_partition.yaml \
  -o /tmp/reputation-partitioned.jsonl

# Inspect both traces
uv run nest inspect /tmp/reputation-baseline.jsonl
uv run nest inspect /tmp/reputation-partitioned.jsonl

# Run registered validators
uv run python - <<'PY'
from pathlib import Path
from nest_core.validators import validate_trace

for label in ("baseline", "partitioned"):
    path = Path(f"/tmp/reputation-{label}.jsonl")
    print(f"[{label}]")
    for result in validate_trace(path, "reputation"):
        print("PASS" if result.passed else "FAIL", result.name, "-", result.detail)
PY
```

Re-running the partitioned scenario at seed 42 produces a byte-identical trace.

## Tools and help used

I used OpenAI Codex to read the official quickstart and scenario guide, compare prior
public submissions, inspect the relevant NANDA Town source, execute the local runs and
tests, derive counts from the JSONL traces, and edit this write-up. The experiment choice
and pre-run hypothesis were approved before the partitioned scenario was executed. AI
use is allowed by the assessment; all claims here are tied to reproducible commands and
trace evidence.
