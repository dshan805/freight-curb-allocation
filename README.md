# freight-curb-allocation

CET 513 group project (Autumn 2026): a network optimization model for parking
enforcement in commercial vehicle load zones (CVLZs) in Downtown Seattle. The
goal is to schedule and route enforcement officers to maximize profit (fines
from illegally parked vehicles minus officer wages), which frees curb space for
delivery vehicles.

The [project plan](CET%20513%20Project%20Plan%20Parking%20Enforcement%20in%20CVLZs.md)
records the first proposal (maximize tickets, in two stages). The
[current model](#optimization-model) replaces it with a single profit model.

## Status

- **Exploratory data analysis:** in `notebooks/01-eda.ipynb`. It cleans the
  survey data, joins curb space and weather data, adds the fine for each stay,
  looks at illegal parking and stay duration, finds the most vehicles parked at
  once on each segment, and maps the surveyed loading zones.
- **Segment-to-segment distances:** computed in the same notebook for the 28
  surveyed street segments, as straight-line distance and as walking distance
  on the OpenStreetMap street network. The notebook compares the two and maps
  the walking route between two chosen segments.
- **Optimization model:** written out [below](#optimization-model), not yet
  coded or solved.

## Optimization model

The model is solved once for each survey date. It chooses how many officers to
use, when each one starts, where they walk and which vehicles they ticket.

It uses the observed arrival and departure times, so its result is the best plan
in hindsight, not a forecast.

### Assumptions

| Item | Value |
|---|---|
| Day modeled | 7:00 to 18:00, in 1-minute steps (660 minutes) |
| Shifts | 8 hours, starting at 7:00, 8:00, 9:00 or 10:00, at most one per officer per day |
| Officer cost | $278.84 per shift ($34.855 per hour, the average of $32.53 and $37.18) |
| Fine | $78 for `CVLZ`, `CVLZ/PS`, `PS-CVLZ`; $63 for `L/UL`, `LOAD`, `PS-VEN` |
| Ticketing time | 2 minutes per vehicle, one vehicle at a time per officer |
| Walking speed | 84 m per minute (about 1.4 m/s) |
| Officers per segment | No limit |

### Notation

| Symbol | Meaning |
|---|---|
| $I$ | Set of all parking stays (vehicles) on the date |
| $i \in I$ | One parking stay (vehicle) from $I$ |
| $j, l$ | Street segments (`ELMNTKEY`) |
| $k$ | Possible officer (how many to allow is not yet set) |
| $t, t'$ | Minutes, 1 to 660 |
| $p \in P$ | Shift start minutes: 1, 61, 121, 181 |
| $x_i$ | 1 if vehicle $i$ is illegally parked |
| $F_i$ | Fine for vehicle $i$ |
| $C$ | Officer cost per shift |
| $a_i, e_i$ | First and last full minute vehicle $i$ is parked |
| $s_i$ | Segment where vehicle $i$ is parked |
| $\tau_{jl}$ | Walking minutes from $j$ to $l$: distance / 84, rounded up |

Decision variables, all binary:

| Variable | Equals 1 if |
|---|---|
| $y_i$ | vehicle $i$ is ticketed |
| $z_{ikt}$ | officer $k$ starts ticketing vehicle $i$ at minute $t$ |
| $u_{jkt}$ | officer $k$ is at segment $j$ at minute $t$ |
| $w_k$ | officer $k$ is used |
| $v_{kp}$ | officer $k$ takes the shift starting at $p$ |

$z_{ikt}$ is 0 for any $t$ outside $a_i$ to $e_i - 1$.

### Objective

```math
\max \sum_{i} F_i x_i y_i - \sum_{k} C w_k
```

### Constraints

1. Only illegally parked vehicles are ticketed:

```math
y_i \le x_i \qquad \forall i
```

2. The officer is at the vehicle's segment for both minutes of the ticket, for
   all $i$, $k$ and $t \in \{a_i, \dots, e_i - 1\}$:

```math
z_{ikt} \le u_{s_i kt}, \qquad z_{ikt} \le u_{s_i k,t+1}
```

3. Each officer used has exactly one shift:

```math
\sum_{p \in P} v_{kp} = w_k \qquad \forall k
```

4. An officer writes one ticket at a time:

```math
\sum_{i} \left( z_{ikt} + z_{ik,t-1} \right) \le 1 \qquad \forall k, t
```

5. A vehicle counts as ticketed only if a ticket starts while it is parked:

```math
y_i = \sum_{k} \sum_{t = a_i}^{e_i - 1} z_{ikt} \qquad \forall i
```

6. Officers work only during their shift:

```math
u_{jkt} \le \sum_{p \in P \,:\, p \le t \le p + 479} v_{kp} \qquad \forall j, k, t
```

7. Walking time, which also keeps an officer to one segment at a time, for all
   $k$, $j \ne l$ and
   $t' - t \in \{0, \dots, \tau_{jl}\}$:

```math
u_{jkt} + u_{lkt'} \le 1
```

8. All decision variables are binary:

```math
y_i, z_{ikt}, u_{jkt}, w_k, v_{kp} \in \{0, 1\}
```

Written this way, constraint 7 produces millions of constraints per officer if
all 28 segments and the full day are included, so start testing on a small case.

## Data

| Data | Source | How the notebook gets it |
|---|---|---|
| Curbside parking observations (IDAX rounds 1 to 3) | Seattle SMART project, Urban Freight Lab | Local CSV files in `data/` |
| Curb space inventory | SDOT ArcGIS API (`Curb_Space_Categories`) | Downloaded when the notebook runs |
| Hourly temperature and precipitation | Open-Meteo archive API | Downloaded when the notebook runs |
| Street network | OpenStreetMap, through OSMnx | Downloaded once and saved to `data/street_graph.graphml`, then loaded from that file |

The notebook expects the survey files here:

```
data/SMART Project Data/dot_87983_DS1/
├── IDAX Round 1 Data Collection.csv
├── IDAX Round 2 Data Collection.csv
└── IDAX Round 3 Data Collection.csv
```

Running the notebook needs an internet connection for the downloaded data and
the map backgrounds.

If you have `data/street_graph.graphml`, skip the notebook cell that calls
`ox.graph_from_polygon` and run the cell that loads the file:

```python
street_graph = ox.load_graphml('../data/street_graph.graphml')
```

The download goes to a public map server, which has refused requests
(`JSONDecodeError`) and timed out (`ConnectTimeout`). Run the download cell only
if you don't have the file.

Things to know before using the results:

- The analysis uses round 1 only, which covers 28 street segment keys in
  Belltown and Denny Triangle, on weekdays from April 30 to June 13, 2024.
- "Illegally parked" means `activity_type == 'Private'`. This is a proxy chosen
  by the group, not a confirmed violation.
- Parking events can't be tied to a specific curb space, so each `ELMNTKEY`
  (one side of one block) is treated as one location, represented by the
  central point of its curb space.
- The survey covers 28 of the 255 segments with at least one loading zone in the
  two neighborhoods (11%).
- Walking distances move each central point to its nearest street node, so short
  distances are approximate. In 16 of 378 segment pairs the walking distance is
  shorter than the straight line, and some neighboring segments show 0 m.

[docs/eda-variables.md](docs/eda-variables.md) describes the data pipeline and
the known data problems.

## Folder structure

```
freight-curb-allocation/
├── data/               # raw data (not on Git, see .gitignore)
├── docs/               # data pipeline notes
├── lit_review/         # papers for the literature review
├── notebooks/          # exploration and analysis (01-eda.ipynb)
├── CET 513 Project Plan Parking Enforcement in CVLZs.md
├── CET513-Project.pdf
└── requirements.txt
```

## Setup

1. Clone the repo.
2. Download the data from the shared Google Drive folder into `data/`, matching
   the paths under [Data](#data).
3. Create and activate a virtual environment (PowerShell, from the project folder):
   ```powershell
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   ```
   If activation is blocked by the execution policy, run this once:
   ```powershell
   Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
   ```
4. Install dependencies:
   ```powershell
   pip install -r requirements.txt
   ```
5. To use the notebooks in VS Code, open a notebook, click **Select Kernel**,
   and choose the `.venv` environment.

Activation only lasts for the current terminal session, so repeat step 3's
activation command in each new terminal (VS Code may do this automatically
once `.venv` is selected as the Python interpreter).

The terminal and the notebook pick their Python separately. If a notebook cell
raises `ModuleNotFoundError` after installing, run this in a cell and check that
the path points into `.venv`:

```python
import sys
print(sys.executable)
```
