# CET 513 Project Plan: Commercial Double-Parking in Downtown Seattle

Nov 8, 2026 · @David

## Project overview

Goal: develop a network optimization model to minimize the total time commercial delivery vehicles spend double-parking in Downtown Seattle by improving the allocation and utilization of existing loading zones.

Planned approach: forecast loading zone occupancy or available capacity with deep learning, then feed it into a linear programming model that assigns deliveries to loading zones. The objective minimizes cruising time plus double-parking time.

Phase 1 is a gate. Everything in Phases 2 to 4 depends on what the data check finds, and the problem statement gets narrowed once data availability is known.

## Phase 1: Literature review, data check, scoping (do first)

These three steps run in parallel and must finish before Phase 2 starts.

### 1.1 Literature review

- [ ] Curb and loading zone allocation models, truck parking assignment (objective and constraints)
- [ ] Deep learning for parking or curb occupancy forecasting
- [ ] Predict-then-optimize frameworks (DL forecast feeding an LP)
- [ ] Urban Freight Lab Seattle studies for parameters: dwell times, commercial vehicle parking behavior

### 1.2 Data check

| Source | What to check | Status |
| --- | --- | --- |
| [Seattle SMART Grant dataset (ROSA P)](https://rosap.ntl.bts.gov/view/dot/87983) | Download the 94.5 MB ZIP; look for timestamped CVLZ occupancy, dwell events, permit transactions. Open access, CC BY 4.0 | To do |
| SDOT open data | CVLZ locations and space counts; paid parking occupancy (does it cover load zones?) | To do |
| [Tracking Curb Use in Seattle (UFL, 2019)](https://depts.washington.edu/toolkit/sites/default/files/toolkit_downloads/Final-50-Feet-Tracking-Curb-Use-in-Seattle_0.pdf) | Dwell time and occupancy parameters for five downtown areas | To do |
| [Battle for the Curb (UFL, TRR 2019)](https://urbanfreightlab.com/?p=2408) | Vehicle types, dwell time distributions, curb use choice | To do |
| [UFL Belltown sensor data](https://urbanfreightlab.com/research_keywords/curb-usage) | Not found publicly; email UFL to request access | To do |
| Traffic flow / sensor data | Whether usable for cruising time | To do |

Outcome: decide what the DL model can realistically predict, most likely CVLZ occupancy or available capacity by zone and hour.

### 1.3 Scoping

Refine the problem statement once the data check is done.

- [ ] Study area boundary (keep it small; match it to where the data is)
- [ ] Time period modeled (e.g. weekday peak delivery hours)
- [ ] Operational definition of double-parking time
- [ ] Final research question, agreed by the group

## Phase 2: GIS and supporting data

Build the spatial base and clean data for the models. Budget extra time: joining sources by location is usually the slowest step.

- [ ] Loading zone layer clipped to the study area
- [ ] Street network from OpenStreetMap (OSMnx) for cruising distances between zones
- [ ] Business and land use data for demand estimation (King County parcels, business licenses, or OSM points of interest)
- [ ] Delivery rates by business type, from the literature
- [ ] Clean and join all sources by location and time

## Phase 3: Models

### 3.1 Objective model

- [ ] Objective: minimize cruising time plus double-parking time
- [ ] Assignment of deliveries to loading zones under capacity constraints, peak period first
- [ ] Baseline for comparison: current layout or a nearest-zone rule
- [ ] Solver choice (Gurobi academic license, or PuLP / Pyomo)

#### Decide: time slot length

The slot length for t is a modeling choice, not fixed at one hour. Set it close to the typical dwell time; 30 minutes is a reasonable starting point.

- Finer slots (15 min) catch short peaks, but a delivery can span several slots, which needs extra constraints carrying occupancy across slots
- Finer slots also mean fewer observations per slot (noisier averages) and more variables
- Slots cannot be finer than the data's recording interval
- Keep slot length as a code parameter and test 15, 30 and 60 minutes as a sensitivity check

### 3.2 Deep learning forecasting

- [ ] Predict occupancy or available capacity by zone and hour
- [ ] Evaluate with forecast error (e.g. MAE) against a simple baseline such as historical average, to show DL adds value
- [ ] Output feeds the LP as its capacity input

### 3.3 LP optimization layer

- [ ] Solve with forecast capacity plus estimated demand
- [ ] Measure reduction in double-parking and cruising time vs. baseline

### 3.4 Sensitivity and scenarios

- [ ] Run low, medium and high demand scenarios to show results hold despite estimated demand

## Phase 4: Results and deliverables

- [ ] Maps of zone utilization, before vs. after optimization
- [ ] Final report, including stated assumptions (estimated demand, peak-period snapshot)
- [ ] Final presentation

## Risks and fallback

The main risk is that no usable time-series occupancy data exists for load zones. In that case, drop the DL layer and run the LP with estimated capacity and demand across scenarios. The group should agree on this fallback upfront.

## Open items

- Course deadlines (proposal, check-ins, final report, presentation)
- Group members and role split (lit review, data, modeling, writing)

## Plan A: should we use deep learning?

Plan A should vary by hour and by weekday vs. weekend either way. Deep learning is only needed if we predict a specific day rather than a typical day.

### Formulation

Decision variable: x\_ijt = deliveries from block i in hour t sent to loading zone j. Demand D\_it is fixed; d\_it (double-parked deliveries) is the leftover outcome.

```latex
\min \sum_{i,j,t} c_{ij} x_{ijt} + p \sum_{i,t} d_{it}
\quad \text{s.t.} \quad \sum_j x_{ijt} + d_{it} = D_{it}, \quad \sum_i x_{ijt} \le C_{jt}, \quad x, d \ge 0
```

c\_ij = cruise and walk time from block i to zone j; p = time penalty per double-parked delivery; C\_jt = zone capacity in hour t.

### Time and day type

Capacity and demand differ by hour and between weekdays and weekends. C\_jt and D\_it are set per hour and day type, and the LP runs for each day type. This holds with or without DL.

### Two ways to set capacity C\_jt

|  | Historical average | Deep learning forecast |
| --- | --- | --- |
| Question answered | What does a typical weekday 11am look like? | What will this Tuesday at 11am look like, given conditions now? |
| Input | Average free spaces per zone, by hour and day type | Recent occupancy, time, day type, possibly weather and events |
| Use case | Planning study | Real-time truck guidance |
| Data needed | Moderate | Large occupancy time series per zone |
| Effort | Low | High |

### Why use deep learning

- Reacts to today's conditions: given what is occupied now, predicts the next 30 to 60 minutes
- Captures day-specific deviations (events, holidays, unusually busy mornings) that averages smooth out
- Can learn patterns shared across nearby zones

### Why not

- Averages by hour and day type already capture the typical pattern, including passenger cars occupying load zones
- Needs a large occupancy time series; the SMART pilot may be too small
- Only adds value if Plan A is framed as real-time guidance, not typical-day planning
- The LP treats the forecast as truth, so forecast error flows straight into the assignment

### Recommendation

Build the LP with historical averages by hour and day type first. Add DL only if the data check finds enough occupancy history and the group frames Plan A as real-time guidance. Then compare DL against the averages; either result is a finding.

## Sources

- [Seattle SMART Grant Digital Commercial Vehicle Permit Project dataset (ROSA P)](https://rosap.ntl.bts.gov/view/dot/87983)
- [The Final 50 Feet: Tracking Curb Use in Seattle (UFL, 2019)](https://depts.washington.edu/toolkit/sites/default/files/toolkit_downloads/Final-50-Feet-Tracking-Curb-Use-in-Seattle_0.pdf)
- [Commercial Vehicle Parking in Downtown Seattle: Insights on the Battle for the Curb (UFL summary)](https://urbanfreightlab.com/?p=2408)
- [UFL curb usage research, incl. Belltown sensor project](https://urbanfreightlab.com/research_keywords/curb-usage)
