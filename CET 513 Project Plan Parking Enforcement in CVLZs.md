# CET 513 Project Plan: Parking Enforcement in CVLZs

Nov 8, 2026 · @David

Objective: maximize the number of parking tickets given to vehicles illegally parked in commercial vehicle load zones (CVLZs) in Downtown Seattle.

## Problem and assumptions

SDOT found that over 80% of vehicles using CVLZs were unauthorized (no valid Commercial Load Permit). Unauthorized vehicles take space delivery trucks need, which pushes trucks into double-parking. Better-targeted enforcement frees that space.

Assumptions:

- Illegal parking in CVLZs can be predicted or forecast from the data we have
- More tickets will decrease illegal parking in CVLZs, freeing space for commercial vehicles
- Officers patrol on foot along the street network

## Proposed model (for team discussion)

The problem is TSP-like: officers move through a network of CVLZs. Because not every CVLZ must be visited and the goal is maximizing tickets, it is closest to the Travelling Officer Problem (an orienteering problem). Index j, k = nodes (CVLZs or groups of them); t = time slot.

### Objective

```latex
\max Z = \sum_{j} \sum_{t} w_{jt}
```

Maximize total tickets w issued across all nodes and time slots.

### Decision variables

- Number of officers in each area per time slot (y\_jt)
- Officers moving between areas from one slot to the next (z\_jkt)
- Tickets issued in each area per slot (w\_jt)
- Hours working: which slots officers are on duty

### Parameters

| Parameter | Meaning | Source |
| --- | --- | --- |
| d\_jk | Walking distance between loading zones | Dijkstra on the street network |
| V\_jt | Illegally parked vehicles at node j, slot t | Data (historical average or forecast) |
| r | Tickets one officer can issue per slot | Data or assumption; Seattle averages about 11 per officer per day |
| N | Total available officers | Assumption or SDOT staffing |
| s | Walking speed | Assumption, about 80 m/min |
| L | Slot length | To decide after data check |
| Ticket cost | Fine per ticket | Seattle fine schedule; only needed if the objective becomes revenue |

### Constraints

- Number of total available officers: sum of officers deployed in a slot ≤ N
- Time of day officers are working: officers deployed only in on-duty slots, within shift length
- Size of jurisdiction: each officer covers a limited set of blocks
- Tickets one officer can issue per unit time: w ≤ r × officers present, which covers time to park, patrol and ticket
- Can't ticket more cars than are illegally parked: w\_jt ≤ V\_jt
- Officer capacity with walking loss: walking time between nodes reduces tickets that slot
- Can't move further than humanly possible: no move longer than one slot of walking (d\_jk / s ≤ L)
- Flow conservation: officers at a node next slot = officers who stayed or walked there

## Proposed approach (for team discussion)

Two stages: stage 1 decides which CVLZs each officer is responsible for, stage 2 decides the order each officer walks through them. Dijkstra feeds distances to both.

1. **Distances (Dijkstra).** Build the downtown street network, snap each CVLZ to its nearest intersection, and compute shortest walking distance d\_jk for every pair. This runs once, before any optimization.
2. **Violations V\_jt.** Start with historical averages by node, time slot, and weekday vs. weekend. Deep learning forecasting of V\_jt is an upgrade, worth it only if the data has enough history and day-specific prediction matters.
3. **Stage 1: assignment (which CVLZs each officer covers).** Splits the CVLZs among the available officers, e.g. Officer 1 → {1, 2, 4}, Officer 2 → {3, 6, 9}, Officer 3 → {5, 7, 8}. Each CVLZ goes to at most one officer; low-value CVLZs can be left unassigned. Each officer's set must be workable in their shift: time spent ticketing plus estimated walking (from Dijkstra distances) ≤ shift length. Objective: maximize total tickets from the assigned CVLZs. Stage 1 does not decide the walking order.
4. **Stage 2: routing (TSP per officer).** Takes each officer's set from stage 1 and finds the shortest order to walk through it, using Dijkstra distances, e.g. Officer 1: 2 → 1 → 4. Then checks the real route fits the shift; if it doesn't, adjust that officer's set in stage 1 and re-run.

Idea to test once we see the data: a single CVLZ may have only 1 to 3 violations, which leaves an officer idle for much of a slot. Grouping nearby CVLZs into beats (one beat per officer per slot) keeps stage 1 simple and puts the TSP inside each beat.

## Next steps

Look at the data first; most model choices depend on it.

- [ ] Location: is each record tied to a CVLZ, blockface, address, or GPS point?
- [ ] Time: are there timestamps, and how fine?
- [ ] Volume: roughly how many violations per CVLZ per hour?
- [ ] Coverage: how many CVLZs, and which part of downtown?

Then decide:

- [ ] Nodes: single CVLZs or beats
- [ ] Time slot length (15, 30 or 60 min)
- [ ] Study area
- [ ] Parameter values: r, N, walking speed
- [ ] Historical averages or DL forecast for V\_jt

## Readings

Parking enforcement models:

- [Improving Parking Enforcement Scheduling using Integer Linear Programming (U of Arkansas capstone, 2025)](https://engineering.uark.edu/_resources/images/industrial/ineg-capstone-poster-uofa-parking-transit-2025.pdf): closest to our stage 1
- [Planning of parking enforcement patrol (Lei, Zhang & Ouyang, TR Part B 2017)](https://par.nsf.gov/servlets/purl/10079021)
- [Arc Routing for Parking Enforcement Officers (Ferreira et al., EJOR 2022)](https://econpapers.repec.org/RePEc:eee:ejores:v:299:y:2022:i:1:p:283-301)
- [Spatial-Aware Deep RL for the Traveling Officer Problem (Strauß & Schubert, 2024)](https://www.arxiv.org/pdf/2401.05969): future work

TSP and orienteering:

- [Cornell LP course, Lecture 23: TSP (video)](https://people.orie.cornell.edu/huseyin/lp_course/lp_course.html)
- [MIT 15.053: shortest path, network flows, TSP (lecture notes)](https://ocw.mit.edu/courses/15-053-optimization-methods-in-management-science-spring-2013/resources/mit15_053s13_lec17)
- [Gurobi TSP notebook](https://www.gurobi.com/jupyter_models/traveling-salesman/)

Seattle context:

- [SDOT V2C pilot: over 80% of CVLZ users unauthorized](https://www.itskrs.its.dot.gov/2026-b02039)
- [UFL: Tracking Curb Use in Seattle (2019)](https://urbanfreightlab.com/?p=3516)
- [Seattle 2024 citations and officer count](https://www.spokesman.com/stories/2025/jan/01/seattle-parking-tickets-will-cost-more-in-2025)
