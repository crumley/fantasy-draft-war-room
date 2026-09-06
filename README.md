# fantasy-draft-war-room

An agent skill that builds a complete fantasy-football draft operation for **any league on
any host**: a value board scored under the league's own rules from free data, a draft
simulator calibrated on the league's own history, and live draft-day machinery — an
append-only event log, pick forecasts graded honestly, an auto-refreshing dashboard, and a
session protocol for an agent that rides shotgun through the whole draft.

```
draft room (host's UI in a browser)
      │  screen-read the results pane  (or poll the host API, if proven)
      ▼
pick ingest ───────────────► append-only event log  (THE state)
                                   │            ▲
              replayed by every consumer        │  manual pick/undo (fallback writer)
      ┌────────────────────────────┼────────────┘
      ▼                            ▼
live dashboard               the "brief" ───► forecast ledger (predictions as issued)
(second terminal,            (compact deterministic
 read-only, flair)            state block for the agent)
```

The skill is a phased checklist ([`SKILL.md`](SKILL.md)). Every step states the **what and
why**, proven in a real league; the agent following it fills in the **how** for the league
in front of it — each phase ends with the "Blanks to fill" that belong to your league, not
the recipe.

## What's inside

| Phase | What it builds |
|---|---|
| 0 | The repo, structured so intent governs the tooling — the league host is a *swappable seam* |
| 1 | League facts as configuration: the scoring vector, roster shape, draft, keepers |
| 2 | Free data: stat-level projections (FantasyPros exports, Sleeper API), ADP, an id crosswalk, your league's own history |
| 3 | The board: blend → league-scored VOR → ceiling → availability prior → news overrides |
| 4 | Keeper evaluation (subsets, surplus, survival odds) |
| 5 | Room calibration: a noise model and per-manager profiles fit on *your* league's picks |
| 6 | Strategy search: positional round-windows priced by simulation from your slot |
| 7 | The hand-maintained one-page draft-day plan |
| 8 | Live machinery: event log, failover ingest, forecast engine, dashboard, agent protocol, rehearsal |
| 9 | If your host has an API: what's worth pulling, and the auth traps to expect |
| 10 | The retro, and feeding the draft back into next season's calibration |

Free data sources and manual collection are the backbone throughout; a host API is a bonus
path to prove, never the plan.

## Install

For [Claude Code](https://claude.com/claude-code), clone into a skills directory —
personal (all projects):

```sh
git clone https://github.com/crumley/fantasy-draft-war-room ~/.claude/skills/fantasy-draft-war-room
```

or per-project:

```sh
git clone https://github.com/crumley/fantasy-draft-war-room .claude/skills/fantasy-draft-war-room
```

Then ask your agent to prep your draft — the skill triggers on fantasy draft prep, draft
tooling, and agent-assisted live drafts. Any other harness that reads `SKILL.md`-format
skills works the same way; the file is plain Markdown either way, and humans can just
follow the checklist.

## Provenance

Distilled from a working toolchain built for one 12-team league with sixteen seasons of
history — the league-specific parts (its scoring quirks, its managers, its host) removed,
the hard-won rules kept: score everything under the league's own rules, calibrate on the
league's own history, one persistent noise world per simulated draft, predictions recorded
as issued and graded honestly, refuse-never-guess ingest, the event log as the bus.
