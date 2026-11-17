# FinPulse — Financial Health Check-Up

**FinPulse turns a customer's bank data into a living picture of their financial wellbeing, simulates where their habits lead, and shows the value of every decision before they make it.**

Built for the Deutsche Bank *Next Gen* challenge at Future Build Days.

> **Prototype notice:** all customers are synthetic demo personas. Simulations are illustrative and do not constitute financial advice.

---

## Highlights

| Feature | What the customer sees |
|---|---|
| **My wellbeing** | An interactive 3D *wealth tree*: roots = savings, trunk = investments, leaves = spending categories, soil = financial knowledge. A health score from 0 to 100 and peer comparisons for every spending category. |
| **My next steps** | A ranked, personalised action plan (this week / this month / with your advisor). Committed actions are tracked and verified against later account data. |
| **My prediction** | A Monte Carlo simulation of 2,000 possible futures over 15 years. An interactive chart compares *same habits* with each change, showing the probability of reaching the customer's goal. |
| **Life projects** | Describe a plan in plain words (car, home, trip, business…). The AI estimates the costs; deterministic bank rules enforce affordability and loan limits. |
| **Learn & grow** | A short learning path that feeds back into the knowledge score. |
| **Book a call** | One-click advisor booking through an embedded Calendly scheduler. |

---

## How the score works

The tree health score (0–100) combines **how customers feel** with **what their accounts show**:

```
Tree health = 50% questionnaire percentile + 50% bank-data score
```

- **Questionnaire percentile:** a well-being regression model built on the CFPB *National Financial Well-Being Survey* (2016; R² = 0.57), ranked against people of the same age. Four dimensions: reserves, confidence & skill, debt, planning & protection.
- **Bank-data score:** saving 30% · investing 30% · spending 25% · knowledge 15%.
- **Spending benchmarks:** each category is compared with German households in the same income class (Destatis, *Laufende Wirtschaftsrechnungen* 2024). A category is *healthy* up to 25% above peers, *wilting* above that, and *rotten* above 75% (discretionary categories only).

All scores come from fixed, documented formulas. The language model only writes explanations, and a guardrail layer verifies every number, percentage and claim it produces against the calculated facts. Text that fails validation is replaced by rule-based wording.

---

## Architecture

```
┌──────────────────────┐      ┌──────────────────────────┐      ┌────────────────────────────┐
│  Dashboard (static)  │ ───▶ │  Backend API             │ ───▶ │  AI bridge (Python)        │
│  HTML · CSS · JS     │      │  Java 21 · Spring Boot   │      │  Monte Carlo engine        │
│  :5500               │      │  :8080                   │      │  Prediction & action agents│
└──────────────────────┘      │  tree · scanner · scores │      │  Guardrails · OpenAI       │
                              │  progress · learning     │      │  :8091                     │
                              └──────────────────────────┘      └────────────────────────────┘
```

| Folder | Contents |
|---|---|
| [`backend/`](backend/) | Spring Boot API: assessment, wealth tree, CFPB scanner, progress tracking, learning, and the Python bridge (`backend/tools`). |
| [`prediction-agent/`](prediction-agent/) | Simulation engine, prediction agent, action agent and guardrails (Python standard library only). |
| [`projects-section/`](projects-section/) | The customer dashboard and life-projects planner (static site, no build step). |
| [`docs/`](docs/) | Setup guide and action-mapping reference. |
| [`tools/`](tools/) | End-to-end integration and demo checks. |

---

## Getting started

### Prerequisites

- Java 21 or newer, and Maven 3.9+
- Python 3.10+
- Node.js 18+ (used by the project planner)

### 1. Configure the API key (optional)

```bash
cp backend/.env.example backend/.env
# edit backend/.env and set OPENAI_API_KEY=...
```

Without a key, FinPulse still runs end to end: all numbers come from the simulation, and explanations use rule-based wording.

### 2. Build the backend

```bash
cd backend
mvn package -DskipTests
cd ..
```

### 3. Start all services

```bash
python backend/tools/run_demo.py --components .
```

This starts the API (`:8080`), the AI bridge (`:8091`) and the dashboard (`:5500`). Logs are written to `backend/data/run/`. Stop with `Ctrl+C`.

### 4. Open the dashboard

**http://localhost:5500/index.html?user=max**

Available demo customers: `max`, `maya`, `lena`, `alex` (you can also switch in the header).

---

## Configuration

| File | Purpose | Required |
|---|---|---|
| `backend/.env` | `OPENAI_API_KEY`, optional `OPENAI_MODEL` and `OPENAI_BASE_URL` | For AI-written explanations |
| `prediction-agent/.env` | Gateway key when running the agents standalone | No |
| `projects-section/js/secrets.js` | Browser key for the standalone planner (copy `secrets.example.js`) | No |
| `projects-section/js/studio.js` | `BOOKING_URL`: Calendly scheduling link for *Book a call* | No |

All key files are listed in `.gitignore`. Never commit real credentials.

---

## Testing

```bash
# Java unit tests
cd backend && mvn test

# Python engine and agent tests (offline)
cd prediction-agent && SKIP_AI=1 python -m pytest -q
```

---

## Data sources and limitations

- **CFPB National Financial Well-Being Survey (2016):** US data; euro amounts are mapped to dollar bands.
- **Destatis LWR 2024:** household income classes, not exact individual peers; for single-person households the benchmark is generous.
- **Simulation:** seeded Monte Carlo with illustrative market assumptions; results are ranges, not forecasts or promises.
- **Persistence:** demo customers live in memory and progress is stored in a local file; there is no authentication. Production use would require a database, authentication and regulatory review.

---

## Further documentation

- [docs/DEMO_SETUP.md](docs/DEMO_SETUP.md): detailed setup and rehearsal guide
- [docs/ACTION_MAPPING.md](docs/ACTION_MAPPING.md): how AI actions map to trackable commitments
- [backend/README.md](backend/README.md): API endpoints and the wealth-tree data model
- [prediction-agent/README.md](prediction-agent/README.md): simulation engine, agents and guardrails
- [projects-section/README.md](projects-section/README.md): dashboard and life-projects planner
