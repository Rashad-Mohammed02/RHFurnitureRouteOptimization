# RH Furniture: Route Optimization + BI Pipeline

A **Vehicle Routing Problem (VRP)** solver in Python (Google OR-Tools) feeding a live **Power BI** dashboard for daily delivery planning at a furniture retailer.

Built for **DAT-8564 Business Intelligence** (MS Business Analytics, Hult International Business School, Spring 2026).

---

## The problem

RH Unlimited Furniture runs daily deliveries from a single warehouse to customers across a metro area. The questions on every dispatcher's desk:

1. **What is the minimum-cost set of routes** that delivers all active orders today?
2. **How does each route compare to a naive baseline** (one-truck-per-customer or unsorted dispatch)?
3. **What's the CO₂ impact** of the optimized vs. naive plan?
4. **What if an order gets cancelled mid-day?** Re-plan in seconds without leaving Power BI.

## Architecture

```
┌─────────────────────┐
│  SharePoint Excel   │  ← Input Template (orders, fleet, addresses, config)
│  RH_Input_Template  │
└──────────┬──────────┘
           │
           ▼
┌────────────────────────────────────────────────┐
│  Python pipeline (route_optimization.ipynb)    │
│  ────────────────────────────────────────────  │
│  1. Read all params from Input Template        │
│  2. Apply furniture specs (volume, weight)     │
│  3. Split oversized orders into delivery nodes │
│  4. Build distance/duration matrix             │
│  5. Cost & CO₂ functions                       │
│  6. OR-Tools VRP solver                        │
│  7. Bin-packing scheduler (time-window check)  │
│  8. Per-day detail + loading manifest          │
│  9. CO₂ comparison vs naive baseline           │
│ 10. Export 4 CSVs                              │
└──────────┬─────────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│   Power BI (.pbix)  │  ← Single-Refresh dashboard
│ Optimized routes    │
│ Cost & CO₂ deltas   │
│ Loading manifests   │
└─────────────────────┘
```

The whole thing is **template-driven** — no hardcoded constants, no hardcoded item dictionary. Everything reads from the Input Template's three sheets:

| Sheet | What it holds |
|---|---|
| `Model Config` | Solver parameters, vehicle capacities, depot location |
| `Furniture Reference` | Product specs (dimensions, weight, fragility) |
| `Delivery Orders` | Active orders for the day (filtered by `Status == Active`) |
| `Customer Directory` | Customer addresses |

## What the optimizer does

### Capacity-constrained Vehicle Routing Problem (VRP)

Solved via Google's **OR-Tools** Constraint Programming solver. Constraints:

- Each delivery node visited exactly once
- Each truck respects volume + weight capacity
- Each route starts and ends at the depot
- Time-window feasibility: a delivery must fit within the customer's accepted delivery window AND the driver's shift

### Bin-packing scheduler

After the route-set is determined, a separate bin-packing step assigns deliveries to specific trucks and time slots, splitting **oversized orders** (e.g. multiple sofas to one address) across multiple trucks/days when no single truck can carry them.

### Cost & CO₂

Each route is scored on:
- **Total distance** (km, from a Google Maps API distance matrix)
- **Total duration** (drive time + service time per stop)
- **Fuel cost** (per-km rate × distance × vehicle fuel-economy)
- **CO₂ emissions** (per-km rate × distance × vehicle emission profile)

The notebook always computes a **naive baseline** (e.g. one truck per customer, no consolidation) and reports the optimized plan as a delta against it.

## The "live disruption" workflow

Designed so the Operations team can re-plan in real time without writing any code:

1. **Edit `Status` column** in the Input Template on SharePoint → mark a cancelled order as `Inactive`, mark an urgent new order as `Active`.
2. **Click Refresh** in Power BI.
3. The pipeline re-reads only the Active orders, **re-runs the OR-Tools solver end-to-end**, and the dashboard updates with new routes, new costs, new CO₂.

The notebook also includes a **Disruption Scenarios** section with pre-built test cases (truck breakdown, road closure, surge demand) — uncomment a block, hit Refresh, see the impact.

## Deliverables in this repo

- [`route_optimization.ipynb`](route_optimization.ipynb) — the full Python pipeline (14 markdown sections, 30 cells total).
- [`rh_furniture_dashboard.pbix`](rh_furniture_dashboard.pbix) — the Power BI dashboard.
- [`input_template.xlsx`](input_template.xlsx) — the four-sheet input template (Model Config, Furniture Reference, Delivery Orders, Customer Directory).

## Stack

- **Python**: `pandas`, `numpy`, **`ortools`** (Google's OR-Tools VRP solver)
- **Google Maps API** for the distance/duration matrix (cached locally for reproducibility)
- **Power BI** (Power Query, DAX, Refresh-driven re-runs of the upstream Python script)
- **SharePoint Excel** as the editable single source of truth

## Course context

DAT-8564 Business Intelligence · Hult International Business School · Spring 2026.
