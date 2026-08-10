# Sideline HQ

Single-file fantasy football command center: `index.html`, no build step, no
server. Deployed by GitHub Pages **from `main`** → https://gmensteve-glitch.github.io/sideline-HQ/

**Commit to `main` and push.** Anything on a branch never reaches the site.

## Goal

Cut the time spent setting lineups and working waivers across ~22 Sleeper
leagues to as close to zero as possible. Every feature is judged on that.

## How it works

Lineups are **math, not an LLM call**. One request returns points, game dates
and bye info for every relevant player:

```
GET https://api.sleeper.app/projections/nfl/{season}/{week}
    ?season_type=regular&position[]=QB&...&position[]=DEF&order_by=pts_ppr
```

- `stats.pts_ppr|pts_half_ppr|pts_std` — pick by `league.scoring_settings.rec`
- `date` — game day; drives lock ordering (no kickoff *times* are published)
- `game_id` + `team` — the set of teams playing; a team absent from it is on bye

`optimize()` greedily fills the hardest-to-fill slots first (fewest eligible
positions), scores bye/OUT/IR players at -1 so they're used only as a last
resort, then diffs the ideal lineup against the actual one.

## Views

- **Now** — grouped by league; every league shows its moves or "Lineup good"
- **Leagues** — all leagues, worst-first
- **League page** — verdict banner → sit/start → add now / bid on waivers →
  lineup with projections → bench

## Sleeper facts worth not rediscovering

- `/v1/players/nfl` is ~14MB / 12k players → cached in IndexedDB for 24h
- Player objects have **no** `bye_week`; derive byes from projections (above)
- `practice_participation` and `injury_start_date` are **empty** (0/170 populated)
- `injury_notes` is sparse (~17%); `injury_body_part` and `news_updated` are reliable
- Drop → waivers for `settings.waiver_clear_days`, then plain free agent.
  That timestamp split is what powers "free now" vs "bid on waivers".
- CORS is `*`, so `file://` works for local testing

## AI usage — keep it small

Only two optional calls, both need a user-supplied key, neither on the critical
path: `deepDive()` (Haiku, on tap) and `fetchWaiverWhy()` (Sonnet, per sync).
Everything else is arithmetic. Don't reintroduce per-league LLM calls — v21 did
3 tabs × 22 leagues per app open and that was the thing worth deleting.

## Verifying changes

Cheap and sufficient, in this order:

1. Syntax: extract the `<script>` and `new Function(src)` it via
   `osascript -l JavaScript` (no node on this machine).
2. Logic: the optimizer is pure — run it in JavaScriptCore against crafted
   rosters (bye, OUT, empty slot, SUPER_FLEX, duplicate check).
3. Real data: mirror the pipeline in Python against the live Sleeper API and
   assert no duplicate players, no illegal slot assignments, no regressions.
4. Rendering: build a fixture of captured Sleeper responses, stub `jget`, and
   `--dump-dom` in headless Chrome. **Inspect the DOM text, not screenshots** —
   a full-page PNG costs ~100k tokens, the DOM check costs ~1k and catches the
   same problems. Screenshot only when the question is genuinely visual, and
   crop or downscale first.

Gotchas: `const S` is a lexical global and is **not** on `window`.
Headless Chrome writes the DOM then never exits — background it and kill it.

## Next up

Main waiver days. Settings are available per league (`waiver_type: 2` = FAAB,
`waiver_budget`, `waiver_day_of_week`, `daily_waivers_hour`), so run time and
budget are arithmetic. The open design question is **bid sizing** — the most
promising option is mining winning `waiver_bid` values from each league's
transaction history.
