# FIFA World Cup Standings ETL Pipeline

A Python ETL pipeline I built to pull FIFA World Cup group-stage standings from a
live football API, transform the data with pandas, and load it into a local MySQL
database — following an ETL crash-course tutorial and adapting it along the way.

## What it does

1. **Extract** — sends a `GET` request to API-Football's `/standings` endpoint for a
   given league and season.
2. **Transform** — flattens the nested group-stage response (8 groups × 4 teams) into
   a single pandas DataFrame, one row per team.
3. **Load** — upserts the DataFrame into a MySQL table, so re-running the pipeline
   updates existing rows instead of creating duplicates.

## Tech stack

- Python 3.14
- `requests` — API calls
- `pandas` — data transformation
- `mysql-connector-python` — MySQL connection
- `python-dotenv` — environment variable management
- MySQL 8.0 (local instance)
- `uv` — virtual environment & package management

## The build

The original tutorial used API-Football via RapidAPI and pulled Premier League
standings. I adapted both of those along the way:

- **RapidAPI → direct API-Sports connection.** The RapidAPI listing I needed wasn't
  available, so I signed up directly at api-sports.io instead. That simplified the
  request headers down to a single `x-apisports-key`, rather than the pair of
  `x-rapidapi-key` / `x-rapidapi-host` headers RapidAPI requires.
- **Premier League → FIFA World Cup.** I queried the `/leagues` endpoint to find the
  right league ID (`1`) for the senior men's World Cup, rather than hardcoding
  whatever the tutorial used.
- **Flat table → group stage.** A domestic league standings response is one flat list
  of teams. The World Cup is played in 8 groups of 4, so the API returns a *list of
  lists* — I had to add a second loop (`for group in standings_list: for team in
  group:`) to flatten it properly before building the DataFrame.
- **Season limitation.** The free API-Football tier only covers completed seasons
  (2022–2024), not the live, in-progress 2026 tournament — so the pipeline currently
  pulls 2022 (Qatar) data. Swapping `SEASON = 2022` for `SEASON = 2026` is the only
  change needed once that season becomes accessible (on a paid plan, or once the
  tournament concludes and rolls into the supported range).

## Bugs squashed along the way

A few of the more interesting ones, for posterity:

- **A silent unique key collision.** My table had a `UNIQUE KEY (season, position)`
  constraint — fine for a domestic league where "position" is unique across the whole
  table, but wrong for World Cup groups, where 8 different teams all hold "position
  1" (one per group). Every upsert was silently overwriting the previous group's #1
  team instead of inserting a new row, so I ended up with only 4 rows (one group's
  worth) instead of 32. Dropped the constraint and kept `(season, team_id)` as the
  real primary key.
- **A misplaced git repo.** `git init` had been run at the root of my Windows user
  folder at some point, meaning `git status` was tracking my entire home directory —
  `.ssh`, `Desktop`, browser data, the works. Removed it and reinitialized properly
  scoped to the actual project folder.
- **Jupyter's variable caching.** `.env` changes and `load_dotenv()` don't
  automatically propagate to variables already assigned earlier in a session — cost
  me a few rounds of "the file is fine, the terminal proves it, why is Python still
  using the old value" before realizing I needed `override=True` and to re-run the
  assignment cells.

## Project structure

```
FIFA-Standings-2026/
├── .env                     # API + MySQL credentials (not committed)
├── .gitignore
├── main.py
├── world_cup_notebook.ipynb # main build notebook
└── assets/
```

## Database schema

```sql
CREATE TABLE standings (
    season          INT NOT NULL,
    position        INT NOT NULL,
    team_id         INT NOT NULL,
    team            VARCHAR(100) NOT NULL,
    played          INT NOT NULL,
    won             INT NOT NULL,
    draw            INT NOT NULL,
    lost            INT NOT NULL,
    goals_for       INT NOT NULL,
    goals_against   INT NOT NULL,
    goals_diff      INT NOT NULL,
    points          INT NOT NULL,
    form            VARCHAR(5) NOT NULL,
    PRIMARY KEY (season, team_id)
);
```

`(season, team_id)` is the primary key, so re-running the pipeline for the same
season updates existing teams' stats rather than duplicating rows.

## Following the current tournament

To pull a different season once it's available on my plan:

```python
LEAGUE_ID = 1     # FIFA World Cup (senior men's) — stays the same
SEASON = 2026      # swap in the season I want
```

No other code changes needed — the extraction/transform/load logic holds regardless
of season. One thing to watch for: 2026 is the first 48-team World Cup (12 groups of
4 instead of 8), so expect more groups/teams than 2022 once that season is queryable.
