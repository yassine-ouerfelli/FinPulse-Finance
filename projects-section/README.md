# FinPulse — Projects section

The "life projects" part of FinPulse. A client describes a plan in their own words — a car,
a home, a trip, a business, a family, studies, a sabbatical — and the AI planner estimates
what it costs once, what it costs every month, and whether a bank loan fits.

The client's real income, spending and savings decide the answer: the app computes hard
affordability limits itself and **enforces them on whatever the model returns**.

## Run it

From the repository root, with `OPENAI_API_KEY` set in `backend/.env`:

```bash
python3 backend/tools/run_demo.py --components .
```

Then open:

```
http://localhost:5500/index.html?customer=http://localhost:8080/api/users/maya/kairos
```

Also available: `lena`, `alex`, `max`.

AI requests go to `POST /api/users/{id}/projects/plan`. The backend supplies the customer's
facts, owns the API key and enforces affordability — the browser never calls an AI provider
directly and does not read `js/secrets.js` in this mode. The contract is in
[`backend/docs/CONNECTED_API.md`](../backend/docs/CONNECTED_API.md); the full walkthrough is
in [`docs/DEMO_SETUP.md`](../docs/DEMO_SETUP.md).

### Standalone (no backend)

The section also runs on its own against a local customer file — useful for working on the UI:

- `index.html?customer=test-worst-case.json` — a minimal profile (income, spending, savings,
  inheritance received or only expected)
- `index.html?customer=input.txt` — a full assessment as produced by the prediction and
  action agents

In this mode the browser needs its own key: copy `js/secrets.example.js` to `js/secrets.js`
and fill it in. `js/secrets.js` is git-ignored — never commit a real key.

## Files

| File | What it does |
|---|---|
| `js/config.js` | rules, rates, market data, AI providers and models |
| `js/customer.js` | loads the customer file and derives the five vitals |
| `js/engine.js` | `affordability()` — the bank's own limits computed from the real numbers |
| `js/ai.js` | planner prompt, provider calls with fallback, and `fixPlan()`, which re-computes and enforces every limit on the model's answer |
| `js/app.js` | project cards, portfolio view, advisor view (donut + mandate moments) |
| `js/dashboard.js`, `js/charts.js`, `js/tree3d.js`, `js/learn.js` | the surrounding dashboard: charts, the wealth tree and the learning modules |

## The limits it enforces

Computed in `engine.js`, applied to the AI's plan in `fixPlan()`:

- **Car** — loan payment plus running costs ≤ 15% of income; loan at most 80% of the price
- **Home** — mortgage payment ≤ 35% of income; equity must cover the 12% purchase costs plus at least 10% of the price
- **Business** — six months of personal living costs kept private, own contribution ≥ 15% of start-up costs, loan covered by profit with a 30% margin
- **Trips, sabbaticals, family** — savings goals only; never financed
- When nothing fits today, the planner reports the year it would fit rather than refusing outright

If the model returns a plan that breaks any of these, the numbers are recomputed and the
verdict is corrected before the client sees it.
