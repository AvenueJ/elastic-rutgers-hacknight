# Hack Night Starter Project - World Cup Analyst Agent

Analyse the completed 2026 World Cup using real tournament data from Elasticsearch. You'll build a **World Cup Analyst agent** that you can ask things like:

> *"Who won the Golden Boot, and how many goals did they score?"*
> *"Compare Brazil and Morocco based on their 2026 tournament results"*
> *"How did Germany's tournament go? Did they over- or under-perform?"*

The agent pulls stats from Elasticsearch, reasons over them, and gives you a structured analysis: tournament summary, attack and defence numbers, standout matches, and a verdict. The project has two parts that build on each other:

- **Part 1 - Notebook:** fetch the complete 2026 match data from a public API and ingest it into Elasticsearch
- **Part 2 - Agent Builder:** build a conversational AI agent on top of that data inside Kibana — no extra code, LLM included

Run the notebook first to populate the index, then move to Agent Builder.

---

### Teams in the Dataset

All 48 qualified nations from the 2026 World Cup - including `Brazil`, `Germany`, `France`, `Argentina`, `Spain`, `England`, `USA`, `Mexico`, `Canada`, `Morocco`, `Japan`, `Portugal`, and more.

The index contains every match result from the completed tournament, so the agent can answer questions about form, results, and individual performances.

---

### Prerequisites

- **Elastic Serverless** project (sign up at [Elastic Cloud Serverless free-trial](https://www.elastic.co/cloud/cloud-trial-overview))

---

### Part 1 - Notebook

[world_cup_analyst.ipynb](world_cup_analyst.ipynb)

Run this first - it fetches the complete 2026 tournament data and ingests it. All queries and analysis happen in Agent Builder (Part 2). The notebook's only job is to populate the index.

#### Option A - Google Colab (recommended, no install needed)

The fastest way to get started. Just a browser - no Python install required.

1. Download the notebook
2. Go to [colab.research.google.com](https://colab.research.google.com)
3. Click **File → Upload notebook** and upload `world_cup_analyst.ipynb`
4. Fill in your credentials in the Section 1 cell (see below)
5. Click **Runtime → Run all** to run every cell top to bottom

That's it. Colab handles all the dependencies automatically when the first cell runs `pip install`.

---

#### Option B - Run locally

If you'd prefer to run on your own machine:

**Requirements:** Python 3.9+ with pip installed. Check with `python --version` in a terminal.

```bash
# Install Jupyter and dependencies
pip install jupyter elasticsearch requests

# Launch Jupyter and open the notebook
jupyter notebook world_cup_analyst.ipynb
```

This opens a browser tab. Run cells one at a time with **Shift + Enter**, or run all at once via **Cell → Run All**.

---

#### Filling in your credentials

Whichever option you use, fill in these two values in the Section 1 cell before running:

```python
ELASTIC_ENDPOINT = "https://your-project.es.region.aws.elastic.cloud"
ELASTIC_API_KEY  = "your-elastic-api-key"
```

> **Finding your credentials:** Elastic Cloud Console → Your Project → Connection Details

#### What the notebook does

**Section 1 - Connect** to Elastic Serverless.

**Section 2 - Fetch data** from [openfootball/worldcup.json](https://github.com/openfootball/worldcup.json) - a public domain GitHub repo with the complete 2026 tournament results. No API key required. Results are fetched and printed so you can see what's in the data.

**Section 3 - Create index** `wc2026_matches` with explicit mappings, including a nested `goals` field capturing scorer name, minute, team, and goal type.

**Section 4 - Enrich and ingest** all matches. Computed fields added at ingest: `status` (`played` for all completed matches), `winner`, `total_goals`, `team1_win`, `team2_win`, `stage`.

**Section 5 - Verify** with quick sense-check queries confirming the data landed correctly. This is the last step in the notebook - building the agent happens in Part 2.

**To refresh with latest results:** re-run cells 2-4. The index is dropped and recreated each time so there are no duplicates. Takes about 30 seconds.

#### Index schema - `wc2026_matches`

| Field | Type | Notes |
|---|---|---|
| `date` | date | Match date |
| `round` | keyword | e.g. `Matchday 1`, `Quarter-finals` |
| `group` | keyword | e.g. `Group A`, `Knockout` |
| `stage` | keyword | `group`, `round_of_32`, `round_of_16`, `quarter`, `semi`, `third_place`, `final` |
| `status` | keyword | `played` for all completed matches |
| `team1` | keyword | |
| `team2` | keyword | |
| `score_ft1` | integer | Team 1 full-time goals |
| `score_ft2` | integer | Team 2 full-time goals |
| `score_ht1` | integer | Team 1 half-time goals |
| `score_ht2` | integer | Team 2 half-time goals |
| `total_goals` | integer | Computed at ingest |
| `winner` | keyword | Team name or `draw` |
| `team1_win` | boolean | |
| `team2_win` | boolean | |
| `goals` | object | Multi-valued: `goals.scorer`, `goals.minute`, `goals.team`, `goals.type` (`goal`/`penalty`/`own_goal`) |
| `stadium` | keyword | Host city/venue |

---

### Part 2 - Agent Builder

Everything for Part 2 lives in [agent_builder_guide.md](agent_builder_guide.md). It gives you two paths to the same result:

- **Fast path - Dev Tools Console:** the guide's *Quick Setup* section has the four API commands (three tools + one agent) ready to paste into Kibana's **Dev Tools Console** (hamburger menu → Management → Dev Tools). Paste each block, press play, and you're done - no Kibana endpoint to find, no auth headers, Dev Tools uses your current session.
- **Manual walkthrough:** the rest of the guide builds the same tools and agent field-by-field through the Agent Builder UI, if you want to understand each piece or tweak it.

Either way, once the three tools and the agent exist, open **Kibana → Agents** and the **World Cup 2026 Analyst** is ready to chat.

#### Navigate to Agent Builder

In your Elastic Serverless project, look for **Agents** in the main navigation. 

---

#### The Three Tools

| Tool ID | What it does |
|---|---|
| `get_team_form` | Full match-by-match results for a team - scores, opponents, stage, outcome |
| `get_team_stats_2026` | Aggregated stats: wins, draws, losses, goals scored/conceded |
| `get_top_scorers` | Tournament-wide goalscorer leaderboard - ranked by goals scored |

See `agent_builder_guide.md` for the exact ES|QL query and parameter config for each tool.

---

#### The Custom Agent

| Field | Value |
|---|---|
| **Agent ID** | `wc2026_analyst` |
| **Display name** | World Cup 2026 Analyst |
| **Tools** | All three above |

**Custom instructions summary:** The agent pulls team stats and match-by-match results to produce a structured analysis: tournament summary, attack and defence numbers, standout matches, and a verdict on whether the team over- or under-performed. For Golden Boot questions it calls `get_top_scorers` instead. Full instructions in `agent_builder_guide.md`.

---

#### Step 5 - Chat With Your Agent

Try these - all grounded in real 2026 tournament data:

```
Who won the Golden Boot, and how many goals did they score?
```
```
Compare Brazil and Morocco based on their 2026 tournament results
```
```
How did Germany's tournament go? Did they over- or under-perform?
```
```
How did England perform in the group stage?
```

Watch the **thinking trace** - you'll see the agent calling tools in sequence before forming its answer.

---