# FinPulse — backend (Java 21 / Spring Boot)

Turns a customer's financial data and personal context into an assessment, recommendations
(linked to bank products and learning modules), a prediction, and one `TreeState` JSON that
the frontend renders as a living tree.

No auth, no database. Four demo personas live in memory: **Lena**, **Alex** and **Maya** from
the challenge brief, plus **Max** from the team demo.

Connected setup and the UI contract: [`docs/CONNECTED_API.md`](docs/CONNECTED_API.md) — the
single server-side AI key and the project-planning endpoint.

## Pipeline

```
Data inputs ─────────────┐                        ┌─> Recommendations ─> Actions (bank products)
 savings pots            │                        │                   └> Knowledge trainings
 user data / income      ├─> Assessment ──────────┤
 portfolio (holdings)    │   (scores 0-100,       └─> Prediction (simulate N months)
 transactions            │    habits, findings)
Non-deterministic inputs ┘            │
 goals, own situation, age, risk      └──────────────> TreeState (roots, trunk, leaves, branches, soil, weather)
Projects (house, travel...) ──────────────────────────> branches
```

The core (`model` → `tree`) has **no Spring dependency**, so it is easy to test and to explain.

## Run

Requires **Java 21** and Maven.

```bash
mvn spring-boot:run          # http://localhost:8080
mvn test                     # core pipeline tests
```

For the full demo — backend, Python agents and dashboard together — use the runner from the
repository root:

```bash
python3 backend/tools/run_demo.py --components .     # API :8080 · bridge :8091 · dashboard :5500
```

Or start the pieces yourself:

```bash
# 1 · the Python agents (3.9+); the key stays in backend/.env
python3 backend/tools/kairos_server.py \
        --agent-dir prediction-agent --projects-dir projects-section     # :8090

# 2 · this backend
cd backend && mvn spring-boot:run                                        # :8080

# 3 · the dashboard: any static server in projects-section/, then open
#     index.html?customer=http://localhost:8080/api/users/maya/kairos
```

Without an API key both agents return their rule-based text, so everything still works offline.

## API

Ids: `lena`, `alex`, `maya`, `max`. Real responses for every persona are in
[`samples/`](samples/) (`*-tree`, `*-prediction`, `*-kairos`, `*-intake`, …), so the frontend
can be built before the backend runs.

| Method | Path | What |
|---|---|---|
| GET | `/api/users` | persona picker, with headline and health |
| GET | `/api/users/{id}` | raw inputs |
| GET | `/api/users/{id}/assessment` | scores, habits, findings, interpreted context |
| GET | `/api/users/{id}/tree` | **the main endpoint**: the `TreeState` to render |
| GET | `/api/users/{id}/recommendations` | recommendations with action, product, training, predicted gain |
| GET | `/api/users/{id}/scanner` · `/scanner/answers` | wellbeing scanner output · the questionnaire answers |
| POST | `/api/users/{id}/scanner/what-if` | body `{"SAVINGSRANGES": 5}` → the same output, re-scored |
| GET | `/api/users/{id}/intake` | exactly what is sent to the Python engine |
| GET | `/api/users/{id}/prediction` | engine + chart + prediction agent + action agent (`RUNNING` → poll) |
| GET | `/api/users/{id}/kairos?wait=60` | the projects-section format; waits for the result |
| GET | `/api/users/{id}/forecast/chart` | fan chart (Python engine; Java fallback when the bridge is down) |
| POST | `/api/users/{id}/projects/plan` | AI project planner: body `{"type","name","description"}` |
| POST | `/api/users/{id}/projects` | add or update a project |
| POST | `/api/users/{id}/simulate` | prediction. Body `{"actionIds":["act-buffer"],"months":12}`. Omit `actionIds` = apply all; `[]` = do nothing |
| POST | `/api/users/{id}/storm` | market-crash test result for the animation |
| POST | `/api/users/{id}/actions/{actionId}/commit` · `/drop` | commit to a recommended action · drop it |
| POST | `/api/users/{id}/advance-month?months=1` | demo time machine: events + the new tree |
| GET | `/api/users/{id}/history` · `/progress` | all snapshots for a timeline · trend and latest events |
| PUT | `/api/users/{id}/context` | update age, `riskTolerance` (CAUTIOUS/BALANCED/GROWTH), `situation`, `goals` |
| POST | `/api/demo/reset` | back to the original demo data |

```bash
curl localhost:8080/api/users/maya/tree
curl -X POST localhost:8080/api/users/maya/simulate \
     -H 'Content-Type: application/json' -d '{"months":12}'
```

## The tree

| Tree part | Comes from | Key fields |
|---|---|---|
| **Roots** (one per savings category: Safety, Goal, Deep/retirement, Anchor/protection) | savings pots, project savings | `depth`, `thickness`, `health`, `status` |
| **Trunk** | portfolio | `thickness`, `height`, `rings` (years of consistency), `cracks` (risks), `stability` |
| **Leaves** (one cluster per spending category) | transactions | `size` (scales with spending), `leafCount`, `health` HEALTHY/WILTING/ROTTEN, `rottenRatio` |
| **Branches** | projects | `length` (= progress), `angleDeg`, `fruit` (funded), `blossom` (≥50%) |
| **Soil** | knowledge check + completed trainings | `richness`, `layers` |
| **Weather** | situation (e.g. variable income) | `CALM / BREEZY / WINDY / STORM` |
| **Stage** | total wealth | `SAPLING / YOUNG / MATURE` → which sprite set to use |

All visual values are normalised **0..1**, so the frontend only maps them to sprites, sizes
and animations. Every part also carries a `detail` sentence for the tap-to-inspect panel.

## Financial health scanner (CFPB wellbeing model)

`scanner/FinancialHealthScanner` is a Java port of the Python reference implementation. It
reads the same `src/main/resources/scanner/model_params.json` and passes the original test
cases (`ScannerParityTest`). The model, its items and its caveats are documented in
`src/main/resources/scanner/SCANNER_README.md`.

| Scanner output | Where it shows in the tree |
|---|---|
| `reserves` | health of the safety roots |
| `planning_protection` | health of the deep (retirement) and anchor (protection) roots |
| `debt` | `trunk.barkHealth`; below 50 adds a crack |
| `confidence_skill` | soil richness |
| `predicted_score_percentile` | half of `overallHealth` (the other half is the bank-data score) |
| everything | `tree.wellbeing` (predicted CFPB score, percentiles, habits, warnings) |

- Questionnaire answers are hardcoded per customer in `scanner/ScannerAnswers` (CFPB PUF codes).
- Each recommendation gets a `wellbeingGain`: the scanner's what-if for the answers that action would change (mapping and assumptions in `ScannerAnswers.changeFor`). It is `null` when the questionnaire cannot measure it.
- The prediction view ("with the plan") re-scores the questionnaire with all completed actions.
- **Caveat:** the model is fitted on US data from 2016 with dollar bands; the demo treats € like $.

## How the pieces connect

```
                   ┌──────────────── Java backend (this project, :8080) ────────────────┐
 bank data,        │ tree · scanner (CFPB) · benchmarks · recommendations · progress    │
 Destatis,  ─────► │                                                                     │
 questionnaire     │ kairos/IntakeBuilder: live data ─> customer intake ───────┐         │
                   └───────────────────────────────────────────────────────────┼─────────┘
                                                                                │ POST /assessment
                   ┌──────── prediction-agent (Python) via tools/kairos_server.py ────────┐
                   │ engine.py (Monte Carlo) ─► prediction_agent + action_agent (parallel)│
                   └───────────────────────────────────────────────────────────────────────┘

 Frontend: the tree app reads /api/users/{id}/tree, /prediction, /history …
           the projects section reads /api/users/{id}/kairos
```

- **One customer, one source of truth.** `IntakeBuilder` turns each customer's live data into the Python intake: wealth, cash, company share, spending, saving habit (plus committed standing orders), crash reflex, cover, buffer and diversification plans (from verified actions), goals, wants vs **Destatis peers** (sent as `peer_median`), subscriptions. Extras the bank data cannot know — profile, purpose, crash reflex, will, when a subscription was last used — live in `kairos/KairosPersonas`.
- **Max matches the Python `EXAMPLE_CUSTOMER` exactly** (€490k, €3,100 cash, 82% company, 40% goal chance), which keeps both sides verifiably in sync.
- **Predictions follow the tree.** The cache key is the intake: when a month passes or an action is verified, the pipeline runs again — so Maya's buffer and cover scenarios disappear once they are done.
- **The AI only ever sees "Customer".** The real first name is added back afterwards, for the projects section.
- The Java Monte Carlo port (`forecast/`) is an **offline fallback** (`/forecast/agent-input`, and `/forecast/chart` when the bridge is down). The Python `engine.py` is the source of truth.

## Memory: the tree grows and shrinks over time

`progress/ProgressService` remembers every user and saves to `data/wealth-tree-state.ser`
(it survives restarts — delete it or call `POST /api/demo/reset` to start over).

The rule: **only inputs change; every score is recalculated.** Nothing is updated by hand.
Each month:

1. **Spending** continues the customer's own pattern (half of the recent trend), so bad habits keep growing unless a limit is committed.
2. **Committed actions run**, and are **verified only when the data shows them**: a standing order arrives, an investment step executes, a category stays under its limit, the advisor confirms cover. On verification the questionnaire answers update too.
3. Money left over lands on the current account; a shortfall comes out of the safety pot.
4. **Markets move** (seeded), so investments can fall and the trunk can shrink.
5. **Facts are re-derived** from the data (savings band, products held). Self-assessed answers change only through actions or a new check-up.
6. A full year with ≥10 invested months adds a **ring**.
7. The tree is recalculated and compared with last month. A **snapshot** stores the vital signs and what changed, in plain words ("Eating out leaves recovered: rotten → wilting"). **Milestones** become permanent blossoms.

Tree health = 50% CFPB peer percentile (what the customer tells us) + 50% bank-data score
(what the accounts show). Over 12 months, Maya with a plan goes 49 → 68; doing nothing,
49 → 44.

## Benchmarks: when is a behaviour good or bad?

| Behaviour | Benchmark | Source |
|---|---|---|
| **Spending per category (leaves)** | the average of households in the same net-income class | **Destatis, Laufende Wirtschaftsrechnungen 2024** (`assessment/SpendingBenchmarks`) |
| Leaf health | healthy up to 1.25× the peer average; wilting above that or when growing ≥25%; rotten above 1.75×, or above 1.25× and growing ≥40%. Rent and groceries can only wilt. | our thresholds |
| Scanner dimensions and overall health | percentile among peers of the same age band | CFPB National Financial Well-Being Survey 2016 |
| Emergency buffer | 3–6 months of spending | common financial-planning guideline |
| Savings rate / wants | 20% saved, wants ≤ 30% of income | 50/30/20 rule |
| Concentration | one position ≤ 30% (a family company glides down to 60% step by step) | diversification rule of thumb |

Destatis table (EUR per household and month, 2024), used as-is:

| Category (Destatis group) | <€1,300 | €1,300–2,600 | €2,600–3,600 | €3,600–5,000 | €5,000+ |
|---|---|---|---|---|---|
| Housing (Wohnen, Energie) | 605 | 816 | 1,064 | 1,271 | 1,666 |
| Groceries (Nahrungsmittel, Getränke) | 238 | 306 | 403 | 502 | 652 |
| Transport (Verkehr) | 111 | 204 | 354 | 468 | 787 |
| Eating out (Gastronomie und Beherbergung) | 62 | 102 | 160 | 226 | 378 |
| Shopping (Bekleidung + Innenausstattung) | 86 | 135 | 215 | 272 | 437 |
| Travel & leisure (Freizeit, Sport und Kultur) | 76 | 147 | 246 | 309 | 545 |
| Subscriptions (Information und Kommunikation) | 72 | 104 | 126 | 148 | 192 |
| Other (Andere Waren und Dienstleistungen) | 41 | 63 | 93 | 108 | 169 |

[Source table](https://www.destatis.de/DE/Themen/Gesellschaft-Umwelt/Einkommen-Konsum-Lebensbedingungen/Konsumausgaben-Lebenshaltungskosten/Tabellen/liste-monatlichen-haushalts-nettoeinkommen.html)

**Caveats:** the classes are *household* income and richer households are on average larger,
so for single people the benchmark is generous. The category mapping is approximate — eating
out includes hotels, travel includes all leisure.

Other methodology:

- Bank-data score = 30% savings + 30% investing + 25% spending + 15% knowledge. With the scanner, tree health is the CFPB peer percentile.
- Investing: invested share, diversification (1 − HHI; broad funds count as diversified), consistency (months invested of the last 12).
- The projection replays recent spending, applies committed actions, and grows investments at an illustrative return per risk profile.

## Code map

```
model/           inputs: profile, pots, holdings, transactions, projects
assessment/      Assessor (deterministic scores) + SituationInterpreter (non-deterministic inputs)
recommendation/  RecommendationEngine, Catalog (products + trainings)
projection/      ProjectionService (prediction / "with your plan")
tree/            TreeBuilder -> TreeState (+ storm test)
forecast/        MonteCarloEngine, ForecastProfiles, AgentInputBuilder (offline fallback)
prediction/      PredictionService + HttpKairosBridge (calls the Python pipeline)
kairos/          IntakeBuilder (live data -> intake), KairosPersonas (profile extras)
scanner/         FinancialHealthScanner, ScannerAnswers, WellBeing
progress/        ProgressService (months, verification, snapshots, milestones), FileStateStore
data/            in-memory repository + demo personas
app/, api/       Spring Boot wiring and REST controllers
tools/           kairos_server.py (bridge to the Python agents), run_demo.py (one-command demo)
```

## Next steps

- `SituationInterpreter`: replace the keyword version with an LLM call returning the same `SituationContext`. Keep the numbers in code; let the model only interpret text and phrase explanations.
- `Catalog`: replace the placeholder product names with real ones.
- Swap `InMemoryUserRepository` for real data (open banking) when needed.
