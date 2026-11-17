# FinPulse — prediction and action agents

Tells one customer what can happen if they keep their habits, what can happen if they change
them, and what to do first — measured against their own goals.

Python 3.9+, **standard library only**. No framework, no dependencies.

## The two agents

### Prediction agent — `prediction_agent.py`

| Field | What it says |
|---|---|
| `headline` / `if_nothing_changes` | the main goal and its chance with the same habits |
| `best_change` | the single change that helps the main goal most, why it fits this customer, and the first step |
| `goals` | every goal with its chance today and with its best change |
| `watch_out` | the risk event most likely to hit this customer, and the change that reduces it |
| `trade_off` | what gets worse (for example: safer, but the best futures are less high) |
| `comparison` | one sentence per scenario |
| `disclaimer` | fixed sentence |

### Action agent — `action_agent.py`

A ranked plan in three groups:

- **this week** — quick wins the customer can do alone: start a buffer, cancel unused subscriptions, set a limit for a category far above people like them, put a big purchase on hold
- **this month** — documents, such as a will
- **with your advisor** — diversification plan, investment plan, crash rules, income protection, bundled into **one** meeting rather than four

Rules decide which actions apply, their order, titles, steps, verification and effect on the
main goal (re-simulated by the engine). The AI writes only the intro, one personal "why" per
action, and the meeting sentence. **It never gives buy/sell instructions and never names a
product** — investment topics always go to the advisor, who decides after a suitability check.

## How the work is split

| Layer | Job |
|---|---|
| **Engine** (`engine.py`) | Simulates the futures and produces every number, as a display string |
| **Rules** (`derive`, `build_action_input`) | Differences to the baseline, best change per goal, main risk, trade-off flag, which actions apply. No AI |
| **AI** | Words only: explains the facts in the customer's own terms |
| **Validator** (`validate`, `guardrails.py`) | Rejects invented figures, unknown ids, wrong picks, promise or buy/sell wording, missing fields, over-long text |
| **Fallback** | Deterministic text from the same facts after one failed retry |

The product never shows unchecked AI text, and it works with no API key at all.

## The engine

`engine.py` turns a customer's intake into the agents' input: a "same habits" baseline, one
scenario per habit change not yet made, and all changes together.

Seeded Monte Carlo, **2,000 futures over 15 years**, three pots (company shares, invested
money, cash). Every scenario runs on the **same random draws**, so differences come from the
customer's choices and not from luck.

Modelled behaviour:
- **panic seller** — after a fall of 15% or more, sits in cash and misses the modelled rebound
- **a year without income** — a small chance each year; without cover the whole spending gap comes out of savings, with cover 60% of income is replaced
- **saving** — monthly amounts into cash and investments; a full buffer (3 months of spending) spills over into investments
- **diversification plan** — the company share glides down 11 points a year to 60%

> `ASSUMPTIONS` (returns, volatility, chance of income loss) are **illustrative demo values,
> not calibrated capital-market assumptions**. No tax, costs, inflation or salary growth.
> Replace them with the bank's own figures before any production use.

## Guardrails

`guardrails.py` checks every AI text field before anyone sees it:

- **Numbers** — every EUR amount (compared by value, so €1.07m = €1,070k), %, point difference, duration and plain number must appear in the facts given for that field: the customer, the baseline, and the scenario or action the field is about
- **Wording** — no promises or marketing ("guaranteed", "safe", "certain", "peace of mind", "start today"); "eliminated / removed / gone" only when the simulation really shows 0% for that risk *and* the sentence says "in the simulation"
- **Ids** — no raw internal ids such as `save_more_tomorrow` in customer-facing text
- **Instructions** — no buy/sell wording for securities

On a violation the model gets **one retry** with the list of violations. If it fails again,
only the failing fields are replaced by rule-based text (`source: ai_partial`); the record
stays in `validation`. Every event is logged to the console and to `guardrail.log` as JSON
lines, so the guardrail itself can be demonstrated.

Differences under **2 percentage points** are treated as Monte Carlo noise and reported as
"no clear change". Each scenario carries `risks_remaining` (removed / reduced / unchanged /
increased), and every life fact and impact carries its `scenario_id`.

## Writing for a person, not a spreadsheet

Both agents show what the money means in someone's life, while every figure stays verified:

- **Life facts are computed by the rules, never by the AI** — "your cash would cover 1.1 months of your bills"; "€154k more at 41, which is 4.6 years of your current spending"; "€4,800 a year, €72k set aside over 15 years"
- The prediction agent adds two **"future you" scenes** (same habits vs. with changes); the action agent ties each step to the person's stated purpose
- The prompts give the model a creative brief (profile, purpose, varied language, temperature 0.6) with hard limits: no invented figures or durations, no invented family or life events, no outcome stated as certain — "within reach", never "you own a home"
- Intake supports `profile` (heir / founder / pro), `purpose`, `horizon_years`, a `home` goal and income-scaled saving steps. A customer who spends more than they earn gets "Spend no more than you earn" first.

## Use it

```python
from engine import build_agent_input, EXAMPLE_CUSTOMER
from prediction_agent import predict

res = predict(build_agent_input(EXAMPLE_CUSTOMER))   # intake -> engine (~1 s) -> agent (~10-25 s)
res["source"]     # "ai" | "ai_partial" | "fallback"
res["output"]     # the fields above
res["derived"]    # rule-based facts for charts and buttons
res["problems"]   # validator messages when an answer was rejected
```

```python
from engine import EXAMPLE_CUSTOMER
from action_agent import build_action_input, recommend, EXAMPLE_SPENDING

plan = recommend(build_action_input(dict(EXAMPLE_CUSTOMER, spending=EXAMPLE_SPENDING)))
```

Quick try: `python3 prediction_agent.py example_input.json`

A fresh call takes roughly **10 to 25 seconds** depending on the model — call it in the
background. The intake format is `EXAMPLE_CUSTOMER` at the top of `engine.py`; the spending
intake (subscriptions with last use, categories with a peer median, planned purchases) is
documented at the top of `action_agent.py`.

## Setup

```bash
cp .env.example .env     # then put the gateway key after HIP_API_KEY=
```

Never commit `.env`, and never put a real key in `.env.example`.

Running through the FinPulse backend instead? Then the key lives in `backend/.env` and this
step is not needed — see the [root README](../README.md).

## Files

| File | What it is |
|---|---|
| `engine.py` | Simulation engine: intake → agent input (seeded Monte Carlo, 2,000 futures, 15 years) |
| `prediction_agent.py` | Prediction agent: rules, AI wording, validator, fallback |
| `action_agent.py` | Action agent: ranked plan with engine-computed impact |
| `guardrails.py` | Shared output checks for both agents |
| `eval_agent.py`, `eval_cases.json` | Live model evaluation on 8 synthetic customers |
| `playground.py` | Try the agents interactively |
| `example_input.json` | Example agent input produced by the engine |
| `test_engine.py` | Engine tests, including exact parity with the JavaScript engine in the browser |
| `test_action_agent.py` | Triggers, grouping, advisor routing, validator |
| `test_all_customers.py` | Every sample customer, rules-only and live AI: no raw ids, no forbidden words, traceable numbers |

```bash
SKIP_AI=1 python3 -m pytest -q      # offline, no network
```

## Evaluation

`eval_cases.json` holds 8 synthetic customers (two goals, a safety-first goal, an unrealistic
goal, one change left, nothing left to change).

```bash
python3 eval_agent.py                                  # current model
python3 eval_agent.py --runs 3 --models "model-a,model-b"
python3 eval_agent.py --show c07                       # print the answers for one case
```

Results, 11 November 2026 (8 cases × 3 runs, 6 parallel requests):

| Model | Accepted first try | After retry | Fallback | Avg seconds |
|---|---|---|---|---|
| Mistral-Medium-3.5-128B | 24/24 | 0 | 0 | 20.5 |
| Qwen3.8-27B | 23/24 | 1 | 0 | 10.9 |
| gemma-4-31B-it (1 run) | 8/8 | 0 | 0 | 20.4 |
| Kolibri-1 (1 run) | 2/8 | 6 | 0 | 7.2 |
| Ministral-3-14B (1 run) | 0/8 | 6 | 2 | 22.2 |
| GLM-5.3-Flash (1 run, earlier prompt) | 0/8 | 0 | 8 | 49.6 |

Each run uses a fresh customer name so the gateway cannot answer from cache.
Action agent, same day, 3 customers × 3 runs: Mistral Medium 9/9 first try (11 s),
Qwen 27B 9/9 (6 s).

## Input contract

Whatever produces the scenarios must supply: one `baseline`, one `change` scenario per habit
change the customer has not made yet (with its `change` text and ideally a `first_step`), an
optional `combo`, and for each of them `range`, `middle`, `risk_below_today`, `goal_success`
per goal, plus optional `events` with a `chance`. `example_input.json` is a complete example.

## Known limits

- The validator checks figures, ids, picks and wording. It does **not** check that a sentence states the right direction ("rises" vs "falls") — the evaluation answers were read by hand.
- The engine's assumptions are not calibrated.
- Use pseudonymous customer data with the gateway: the AI only ever sees "Customer", and the real first name is added back afterwards.
