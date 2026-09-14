# ⚽ World Cup 2026 Analyst - Elastic Agent Builder Guide

After running the notebook to ingest the 2026 World Cup data, this guide walks you through building a conversational AI agent on top of it in **Elastic Agent Builder** - no extra code required.

You'll build three custom ES|QL tools and wire them into a custom agent with a football analyst persona. The agent uses the complete tournament results to analyse teams, compare performances, and surface the tournament's top scorers.

---

## Prerequisites

- Elastic Serverless project running
- `wc2026_matches` index populated (run the notebook first)
- Agent Builder enabled (on by default on Serverless)

---

## Navigate to Agent Builder

In your Elastic Serverless project, open **Kibana** and look for **Agents** in the main navigation.

> **Don't see it?** Agent Builder is on by default on Serverless. If it's missing, search for **Agent Builder** in Kibana's global search bar (or check **Management → Advanced Settings**) to enable it.

---

## Quick Setup via Dev Tools Console (fast path)

If you just want everything created in one go, paste the API commands below into Kibana's **Dev Tools Console** instead of clicking through the UI.

Open the **hamburger menu (top left) → Management → Dev Tools**, paste each block, and press the **play button**. Dev Tools uses your current Kibana session, so there's no endpoint to find and no auth headers to set.

Run them in order: the three tools first, then the agent. Once all four have run, open **Kibana → Agents** and the **World Cup 2026 Analyst** is ready to chat.

> Prefer to understand each tool field-by-field, or tweak things in the UI? Skip this section and follow the manual **Step 1 - Step 5** walkthrough below instead. The two paths produce the same result.

### Tool 1: `get_team_form`

```
POST kbn://api/agent_builder/tools
{
  "id": "get_team_form",
  "type": "esql",
  "description": "Returns a team's 2026 World Cup match-by-match results - wins, losses, draws, goals scored and conceded, opponent, date, and stage. Use this when the user asks about a team's results, record, or how they performed in the tournament.",
  "configuration": {
    "query": "FROM wc2026_matches | WHERE status == \"played\" AND (team1 == ?team_name OR team2 == ?team_name) | EVAL goals_scored = CASE(team1 == ?team_name, score_ft1, score_ft2), goals_conceded = CASE(team1 == ?team_name, score_ft2, score_ft1), result = CASE(winner == ?team_name, \"win\", winner == \"draw\", \"draw\", \"loss\") | KEEP date, team1, team2, score_ft1, score_ft2, group, stage, result, goals_scored, goals_conceded, winner, stadium | SORT date ASC",
    "params": {
      "team_name": {
        "type": "string",
        "description": "The team name exactly as it appears in the data, e.g. France"
      }
    }
  }
}
```

### Tool 2: `get_team_stats_2026`

```
POST kbn://api/agent_builder/tools
{
  "id": "get_team_stats_2026",
  "type": "esql",
  "description": "Returns aggregated 2026 tournament statistics for a team: total matches played, wins, draws, losses, goals scored, and goals conceded. Use this when comparing two teams statistically or assessing overall tournament performance.",
  "configuration": {
    "query": "FROM wc2026_matches | WHERE status == \"played\" AND (team1 == ?team_name OR team2 == ?team_name) | EVAL goals_scored = CASE(team1 == ?team_name, score_ft1, score_ft2), goals_conceded = CASE(team1 == ?team_name, score_ft2, score_ft1), is_win = CASE(winner == ?team_name, 1, 0), is_draw = CASE(winner == \"draw\", 1, 0), is_loss = CASE(winner != ?team_name AND winner != \"draw\", 1, 0) | STATS matches_played = COUNT(*), wins = SUM(is_win), draws = SUM(is_draw), losses = SUM(is_loss), total_scored = SUM(goals_scored), total_conceded = SUM(goals_conceded)",
    "params": {
      "team_name": {
        "type": "string",
        "description": "The team name, e.g. Germany"
      }
    }
  }
}
```

### Tool 3: `get_top_scorers`

```
POST kbn://api/agent_builder/tools
{
  "id": "get_top_scorers",
  "type": "esql",
  "description": "Returns the leading goalscorers of the 2026 World Cup, ranked by goals scored, across the whole tournament. Use this when the user asks who scored the most goals or about the Golden Boot. This is not scoped to a single team.",
  "configuration": {
    "query": "FROM wc2026_matches | WHERE status == \"played\" | MV_EXPAND goals.scorer | STATS goal_count = COUNT(*) BY scorer = goals.scorer | SORT goal_count DESC | LIMIT 20",
    "params": {}
  }
}
```

### Agent: `wc2026_analyst`

```
POST kbn://api/agent_builder/agents
{
  "id": "wc2026_analyst",
  "name": "World Cup 2026 Analyst",
  "description": "Ask me anything about the 2026 World Cup - results, form, stats, and standout performances from the finished tournament.",
  "labels": ["worldcup", "hacknight"],
  "avatar_color": "#16C47F",
  "avatar_symbol": "⚽",
  "configuration": {
    "instructions": "You are a football analyst specialising in the 2026 FIFA World Cup, held in the USA, Mexico, and Canada.\n\nYou have access to complete tournament data: every match result, team form, and goal-by-goal scoring records.\n\nWhen a user asks about a team's tournament:\n1. Call get_team_stats_2026 for their aggregate numbers\n2. Call get_team_form for their match-by-match results\n3. Produce an analysis structured as:\n   - Tournament summary: how far they went and their final record\n   - Attack and defence: goals scored and conceded, their biggest win\n   - Standout matches: their most emphatic result and their worst defeat\n   - Verdict: one line on whether they over- or under-performed\n\nWhen comparing two teams, run the same tools for both and contrast them directly.\n\nWhen a user asks who scored the most goals, about the Golden Boot, or about the tournament's leading scorers, call get_top_scorers. This returns a tournament-wide leaderboard and is not scoped to a single team - do not use it to claim who scored for a particular team.\n\nAlways ground your analysis in the 2026 data. Never invent a result, scoreline, or scorer you did not retrieve from a tool.\n\nKeep your tone punchy and engaging. You are talking to developers at a hackathon, not writing a press release.\n\nOnly answer questions about the 2026 World Cup. Politely decline anything off-topic.",
    "tools": [
      {
        "tool_ids": [
          "get_team_form",
          "get_team_stats_2026",
          "get_top_scorers"
        ]
      }
    ]
  }
}
```

---

## Manual Walkthrough (UI, step by step)

The remaining steps build the exact same three tools and agent through the Agent Builder UI - useful if you want to understand each field or tweak things. If you already ran the Dev Tools commands above, you can skip straight to **Step 5 - Chat With Your Agent**.

---

## Step 1 - Create Tool: `get_team_form`

This is the core tool - it returns a team's 2026 match-by-match results: wins, goals scored, goals conceded, stage reached, and each opponent.

Go to **Manage components** (bottom left sidebar) → **Tools** → **New tool**.

| Field | Value |
|---|---|
| **Tool ID** | `get_team_form` |
| **Type** | ES\|QL |
| **Description** | Returns a team's 2026 World Cup match-by-match results - wins, losses, draws, goals scored and conceded, and each match result. Use this whenever the user asks about a team's results, record, or how they performed in the tournament. |

**Query:**

```esql
FROM wc2026_matches
| WHERE status == "played"
  AND (team1 == ?team_name OR team2 == ?team_name)
| EVAL
    goals_scored   = CASE(team1 == ?team_name, score_ft1, score_ft2),
    goals_conceded = CASE(team1 == ?team_name, score_ft2, score_ft1),
    result         = CASE(
        winner == ?team_name, "win",
        winner == "draw",    "draw",
                              "loss"
    )
| KEEP date, team1, team2, score_ft1, score_ft2, group, stage, result, goals_scored, goals_conceded, winner, stadium
| SORT date ASC
```

**Add one parameter:**

| Name | Type | Description |
|---|---|---|
| `team_name` | keyword | The team name exactly as it appears in the data, e.g. France |

Save the tool.

---

## Step 2 - Create Tool: `get_team_stats_2026`

Aggregates a team's 2026 performance into headline numbers for easy comparison.

| Field | Value |
|---|---|
| **Tool ID** | `get_team_stats_2026` |
| **Type** | ES\|QL |
| **Description** | Returns aggregated 2026 tournament statistics for a team: total matches played, wins, draws, losses, goals scored, goals conceded, and goal difference. Use this when comparing two teams statistically or assessing their overall tournament performance. |

**Query:**

```esql
FROM wc2026_matches
| WHERE status == "played"
  AND (team1 == ?team_name OR team2 == ?team_name)
| EVAL
    goals_scored   = CASE(team1 == ?team_name, score_ft1, score_ft2),
    goals_conceded = CASE(team1 == ?team_name, score_ft2, score_ft1),
    is_win  = CASE(winner == ?team_name, 1, 0),
    is_draw = CASE(winner == "draw", 1, 0),
    is_loss = CASE(winner != ?team_name AND winner != "draw", 1, 0)
| STATS
    matches_played  = COUNT(*),
    wins            = SUM(is_win),
    draws           = SUM(is_draw),
    losses          = SUM(is_loss),
    total_scored    = SUM(goals_scored),
    total_conceded  = SUM(goals_conceded)
```

**Add one parameter:**

| Name | Type | Description |
|---|---|---|
| `team_name` | keyword | The team name, e.g. Germany |

Save the tool.

---

## Step 3 - Create Tool: `get_top_scorers`

Returns the tournament's leading goalscorers - lets the agent answer Golden Boot questions and surface individual standout performances.

| Field | Value |
|---|---|
| **Tool ID** | `get_top_scorers` |
| **Type** | ES\|QL |
| **Description** | Returns the leading goalscorers of the 2026 World Cup, ranked by goals scored, across the whole tournament. Use this when the user asks who scored the most goals or about the Golden Boot. This is not scoped to a single team. |

**Query:**

```esql
FROM wc2026_matches
| WHERE status == "played"
| MV_EXPAND goals.scorer
| STATS goal_count = COUNT(*) BY scorer = goals.scorer
| SORT goal_count DESC
| LIMIT 20
```

No parameters — this tool returns a tournament-wide leaderboard.

> **Note:** `MV_EXPAND` works here because `goals` is mapped as a plain `object` (not `nested`), so `goals.scorer` is a multi-valued keyword field ES|QL can expand. This is set up correctly by the notebook.

Save the tool.

---

## Step 4 - Create the Custom Agent

Go to **Manage components → Agents → New agent**.

### Settings tab

| Field | Value |
|---|---|
| **Agent ID** | `wc2026_analyst` |
| **Display name** | World Cup 2026 Analyst |
| **Display description** | Ask me anything about the 2026 World Cup - results, form, stats, and standout performances from the finished tournament. |
| **Avatar** | Green or football-themed |

**Custom instructions** - paste this exactly:

```
You are a football analyst specialising in the 2026 FIFA World Cup, held in the USA, Mexico, and Canada.

You have access to complete tournament data: every match result, team form, and goal-by-goal scoring records.

When a user asks about a team's tournament:
1. Call get_team_stats_2026 for their aggregate numbers
2. Call get_team_form for their match-by-match results
3. Produce an analysis structured as:
   - Tournament summary: how far they went and their final record
   - Attack and defence: goals scored and conceded, their biggest win
   - Standout matches: their most emphatic result and their worst defeat
   - Verdict: one line on whether they over- or under-performed

When comparing two teams, run the same tools for both and contrast them directly.

When a user asks who scored the most goals, about the Golden Boot, or about the tournament's leading scorers, call get_top_scorers. This returns a tournament-wide leaderboard and is not scoped to a single team - do not use it to claim who scored for a particular team.

Always ground your analysis in the 2026 data. Never invent a result, scoreline, or scorer you did not retrieve from a tool.

Keep your tone punchy and engaging. You are talking to developers at a hackathon, not writing a press release.

Only answer questions about the 2026 World Cup. Politely decline anything off-topic.
```

### Tools tab

Assign all three tools:
- `get_team_form`
- `get_team_stats_2026`
- `get_top_scorers`

> **Tip:** Leave built-in Elastic tools unassigned to keep the agent focused.

### Save

Click **Save and chat**.

---

## Step 5 - Chat With Your Agent

Try these prompts - they're all grounded in real 2026 tournament data:

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

Watch the **thinking trace** as the agent calls tools in sequence before responding - that's the context engineering loop in action.

---

## Hack Extensions

### 🟢 Beginner
- **Change the persona** - make the agent sound like a specific pundit, add drama, or give it strong opinions about the host nations
- **Add a `get_group_standings` tool** - query by group to surface final group tables: who finished top, who went through as runner-up, who was eliminated
- **Add a `get_stage_results` tool** - query by stage (`quarter`, `semi`, `final`) to let the agent reconstruct the knockout bracket match by match

### 🟡 Intermediate
- **Per-team and per-type scorers** - the starter `get_top_scorers` tool is tournament-wide because flattening `goals` loses scorer↔team correlation. Build `team1_scorers` / `team2_scorers` / `penalty_scorers` as flat arrays at ingest (edit `enrich()` in the notebook), then add a tool that answers *"Who scored for Brazil?"*
- **Add a squad tool** - ingest `worldcup.squads.json` (same GitHub URL base) as a second index; add a tool that lets the agent discuss each team's key players
- **Log analysis summaries** - create a `wc2026_takes` index and a write tool; instruct the agent to save a one-paragraph verdict after each team analysis, then query which teams drew the most interest

### 🔴 Advanced
- **Historical head-to-head** - ingest a historical World Cup dataset as a second index; add a tool so the agent can contrast a team's 2026 campaign against their all-time tournament record
- **Expose via MCP** - use Agent Builder's built-in MCP server to connect the analyst to Claude Desktop or a custom app
- **Vector search on match narratives** - generate text descriptions of each match ("Brazil drew 1-1 with Morocco in a tight Group C encounter") and store embeddings; add a semantic similarity tool so the agent can find comparable historical matches

---

## Reference

- [Agent Builder docs](https://www.elastic.co/docs/explore-analyze/ai-features/elastic-agent-builder)
- [Custom tools](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/tools/custom-tools)
- [Custom agents](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/custom-agents)
- [ES|QL reference](https://www.elastic.co/docs/explore-analyze/query-filter/languages/esql-kibana)
