# RH Furniture: Route Optimization + BI Pipeline

A Python solver for the **Vehicle Routing Problem (VRP)**, which works out which truck visits which stops and in what order. It's built on Google OR-Tools and feeds a live **Power BI** dashboard that a furniture retailer can use to plan each day's deliveries.

Built with Jainam Gogree and Regina Garfias for **DAT-8564 Business Intelligence** (MS Business Analytics, Hult International Business School, Spring 2026).

---

## The problem

RH Unlimited Furniture runs deliveries every day from a single warehouse to customers across a metro area. Every dispatcher faces the same questions:

1. **What's the cheapest set of routes** that delivers all of today's active orders?
2. **How does each route compare to a naive baseline** (one truck per customer, or dispatching in no particular order)?
3. **What's the CO₂ impact** of the optimized plan vs. the naive one?
4. **What if an order gets canceled midday?** Replan in seconds without leaving Power BI.

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

The whole thing runs off the template. There are no hardcoded constants and no hardcoded list of furniture items. Everything is read from the Input Template's four sheets:

| Sheet | What it holds |
|---|---|
| `Model Config` | Solver parameters, vehicle capacities, depot location |
| `Furniture Reference` | Product specs (dimensions, weight, fragility) |
| `Delivery Orders` | Active orders for the day (filtered by `Status == Active`) |
| `Customer Directory` | Customer addresses |

## What the optimizer does

### Vehicle Routing Problem (VRP) with capacity limits

We solve it with the Constraint Programming solver in Google's **OR-Tools**. The rules every plan has to follow:

- Each delivery node (one stop on a route) is visited exactly once
- Each truck stays within its volume and weight capacity
- Each route starts and ends at the depot (the warehouse)
- Each delivery has to fit inside the customer's accepted delivery window AND the driver's shift

### Bin packing scheduler

Once the routes are set, a separate bin packing step (fitting loads into the fewest trucks and time slots) assigns deliveries to specific trucks and time slots. It also splits **oversized orders**, such as several sofas going to one address, across more than one truck or day when no single truck can carry them.

### Cost & CO₂

Each route is scored on:
- **Total distance** (miles, from a Google Maps API distance matrix)
- **Total duration** (drive time plus service time at each stop)
- **Cost** (crew labor rounded to the quarter hour, truck cost per mile, and a holding cost for orders that wait extra days)
- **CO₂ emissions** (rate per mile × distance, plus idle time at stops)

The notebook also computes a **naive baseline** (each day's customers visited in a fixed order, with no route optimization) and reports the optimized plan as the difference from it.

## The "live disruption" workflow

We designed this so the Operations team can replan in real time without writing any code:

1. **Edit the `Status` column** in the Input Template on SharePoint. Mark a canceled order as `Inactive`, or mark an urgent new order as `Active`.
2. **Click Refresh** in Power BI.
3. The pipeline rereads only the Active orders, **reruns the OR-Tools solver from start to finish**, and the dashboard updates with the new routes, costs and CO₂.

The notebook also has a **Disruption Scenarios** section with ready made test cases (customer not available, partial order cancellation, vehicle breakdown). Uncomment a block, hit Refresh, and see the impact.

## Results

- **Pricing:** the real cost averaged **$761.97 per delivery** against RH's flat **$299** White Glove fee, so we recommended raising the fee to **$350 to $400**, with a surcharge for volume, weight or time.
- **Routing:** in this notebook, optimized routing saved **32 miles** and 3.4% of CO₂ against the naive baseline.
- **Fewer truck runs:** another version of our routing model cut truck runs from **20 to as few as 12** (40% fewer), and its cost optimized plan cut cost by **12.65%** and CO₂ by **31.33%** against its baseline.

## Deliverables in this repo

- [`route_optimization.ipynb`](route_optimization.ipynb): the full Python pipeline (14 markdown sections, 30 cells total).
- [`rh_furniture_dashboard.pbix`](rh_furniture_dashboard.pbix): the Power BI dashboard.
- [`input_template.xlsx`](input_template.xlsx): the four sheet input template (Model Config, Furniture Reference, Delivery Orders, Customer Directory).

## Stack

- **Python**: `pandas`, `numpy`, **`ortools`** (Google's OR-Tools VRP solver)
- **Google Maps API** for the distance/duration matrix (cached locally so results can be reproduced)
- **Power BI** (Power Query, DAX, and Refresh triggering a rerun of the Python script behind it)
- **SharePoint Excel** as the one editable source of truth

## Course context

DAT-8564 Business Intelligence · Hult International Business School · Spring 2026.
