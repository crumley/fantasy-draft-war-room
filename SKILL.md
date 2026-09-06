---
name: fantasy-draft-war-room
description: Build a complete fantasy-football draft operation for any league on any host — a value board scored under the league's own rules from free data, a draft simulator calibrated on the league's own history, and live draft-day machinery (event log, pick forecasts, dashboard, agent session protocol). Use when someone wants to prep for a fantasy draft, build draft tooling for their league, or run an agent-assisted live draft.
---

# Fantasy draft war room

A checklist for building the whole operation, from empty repo to a live draft with an agent
in the loop. It is host-agnostic (Yahoo, Sleeper, ESPN, NFL.com, anything) and
league-agnostic: every step names the **what and why**, proven in a real league; **you fill
in the how** for the league in front of you. Free data sources and manual collection are the
backbone — a host API, if you have one, is a bonus path layered on top, never the
foundation.

Work the phases in order; each builds on the last. Every phase ends with **Blanks to
fill** — the questions this skill cannot answer because they belong to your league. Answer
them by asking the user or reading the league's settings pages, and record the answers in
the repo, not the conversation.

## Phase 0 — Structure the repo around intent

Before any data or code, set up the repository so that the durable understanding of the
league governs the tooling, not the other way around. If the `intent-architecture` skill is
available, use it to scaffold the four legs (`intent/`, `design/`, `src/`, `test/`);
otherwise apply the same discipline with whatever structure fits: **the what/why lives apart
from the how, every idea has one home, every choice carries its reason.**

- [ ] `intent/` (or equivalent) holds what stays true if you rewrote every tool:
  - **Vision**: beat *this* league, under *its* rules, using its own history — not
    consensus rankings, which are priced for a league you are not in.
  - **Principles** worth writing down on day one, each with its why:
    1. *Score everything under the league's own rules.* Consume stat-level projections and
       apply the league's scoring vector yourself; a publisher's pre-scored points embed a
       scoring system that is not yours.
    2. *One number to rule the board*: value over replacement, where replacement falls out
       of the league's roster shape. Everything else (keeper surplus, survival odds,
       strategy grids) derives from it.
    3. *Calibrate on the league's own history, not on theory.* Your room is jumpier and
       more opinionated than any public model assumes.
    4. *Deterministic tools remember; conversations are disposable.* Anything an agent or
       human needs mid-draft must be recomputable from files on disk by one command.
    5. *Predictions are recorded as issued and graded honestly.* No hindsight grading.
    6. *Refuse, never guess.* When live input is ambiguous, report it unresolved with a
       one-command fix, rather than silently picking a candidate.
    7. *Draft day needs no network* (beyond the draft room itself). Cache everything.
    8. *Secrets never enter the repo.* Credentials live in the user's home config,
       untracked, entered by the user themselves.
  - **Concepts**: the scoring vector, replacement level and VOR, ADP vs *your room's*
    prices, projection disagreement as signal, the availability prior, keeper surplus, the
    draft event log, the forecast ledger.
  - **Subsystems as swappable seams** — this is the load-bearing move: *projection
    sources*, *ADP sources*, *the league host*, and *live pick ingest* are each a contract
    with multiple interchangeable implementations behind it. The host being a seam is what
    makes this skill portable and what saves your draft when the host's API lets you down.
- [ ] `design/` records the choices as you make them (language, storage, which sources, why
  a player is faded) — appended, never overwritten, so next season you can read why.
- [ ] Two kinds of documents, kept distinct throughout: **generated reports** (rebuilt by
  one command from current data, never hand-edited) and **hand-maintained pages** (the
  draft-day plan, the ops runbook). Mark which is which; the failure mode is hand-editing a
  file a regeneration will silently clobber.

**Blanks to fill:** language and stack (record as a decision, not a debate — anything with
good CSV/JSON handling and a testing culture works); repo name; what "winning" means in
this league (championship? regular season points? beating one specific rival?).

## Phase 1 — League facts as configuration (the constitution)

Capture the league's rules as data before touching projections. Everything downstream reads
this; nothing downstream should hard-code a rule.

- [ ] The **scoring vector**, stat by stat, including every quirk: points per passing TD,
  turnovers, yardage bonuses, kicker distance buckets, team-defense tiers. The quirks are
  the whole point — e.g. a league with 6-point passing TDs values QBs far above consensus,
  and no public ranking will tell you that. Read it off the league's settings page by hand;
  it takes ten minutes and is the highest-leverage ten minutes of the project.
- [ ] The **roster shape**: starting slots (including flex definitions), bench, IR. This is
  where replacement level comes from.
- [ ] The **draft**: type (snake/auction/linear), date, team count, pick order, your slot.
  From slot and team count derive your exact pick numbers now — they drive strategy,
  rehearsal, and the live-day session protocol.
- [ ] **Keeper/dynasty rules** if any: how many, what each costs, restrictions. Also a
  place to record *other* teams' announced keepers as news lands — they leave the pool
  before pick 1 and change every simulation.
- [ ] A **manager map**: team name → human, across seasons (team names churn; managers
  persist). Needed for everything in Phase 5.

**Blanks to fill:** every field above, from the league's settings page. If the host has an
API, later verify this config against the API's settings payload — hand-entry then API
cross-check catches both kinds of error.

## Phase 2 — Collect free data

All of this is available for nothing. Build a fetch/import habit, not a one-off download:
each source lands as a dated file, superseded files are archived (not deleted), and one
command refreshes everything.

- [ ] **Stat-level projections, two or more independent sources.** The rule: prefer
  sources that publish *stat lines* (yards, TDs, receptions…) over sources that publish
  only their own fantasy points, because only stat lines can be scored under your rules.
  Known-good free options:
  - *FantasyPros* — free account, then hand-download each position's draft-projection
    export (CSV) plus an ADP export. Manual collection is fine — it is minutes of work,
    and a consensus of ~8 publishers is excellent signal. Build an importer that reads
    the raw export shape verbatim, so a refresh is download-and-rerun.
  - *Sleeper's public API* — free, no auth, regardless of where your league lives:
    season-long stat-level projections, ADP, the full NFL schedule (derive byes as the
    week each team has no game), and multi-season per-player stats (fuel for the
    availability prior in Phase 3).
  - A source that only offers points for some position (kickers everywhere, defenses
    often) gets a *fallback-points* column used only while no source can fill the stat
    line — never averaged against a real stat-scored value.
- [ ] **ADP from the population that will actually be in your room** — the host's own ADP
  if you can see it (that is the price *your* opponents see on their screens), else
  FantasyFootballCalculator (free API, format-specific) or Sleeper. Record which you chose
  and why.
- [ ] **A player-identity crosswalk** — the unglamorous keystone. Every source keys players
  its own way; you need one canonical id with per-source id columns. The
  DynastyProcess/nflverse player-ids table is free and covers the major hosts' ids. Names
  are the *fallback*, never the key: a string-only match should warn loudly, and team
  defenses deserve their own scheme (key by nickname). Budget real time here; every
  downstream join stands on it.
- [ ] **Your league's history** — the highest-value dataset you will touch, and the one
  nobody else has. Every past draft (pick, player, team, keeper flag) and every final
  standing, as far back as the league goes. Collect it however the host allows: API if
  available, otherwise logged-in page scrapes or even hand transcription — a 16-year league
  is ~2,500 picks and worth an evening. Keep the raw captures beside the normalized files.
- [ ] **The NFL schedule** — byes and the fantasy-playoff-week opponents — distilled into a
  small checked-in file so draft day reproduces offline.

**Blanks to fill:** which sources your ecosystem offers this season (they shift; verify the
export URLs and API endpoints still work); how far back the league's history is
recoverable; whether your host exposes its ADP.

## Phase 3 — The board (blend, VOR, ceiling, availability)

One pipeline from raw sources to a single ranked board, rebuilt by one command.

- [ ] **Score** every source's stat lines under the league's scoring vector.
- [ ] **Blend** sources into a consensus, and keep the *disagreement* per player — it is
  signal, not noise. Only comparable lines count toward disagreement: a source that
  couldn't be scored under your rules (it published only its own points) is excluded from
  the spread, not averaged in.
- [ ] **Replacement level** per position from the roster math (starters × teams, flex
  apportioned), then **VOR = points − replacement**. This one number ranks the board.
- [ ] **Ceiling**: consensus plus some share of the disagreement — late-round picks are
  lottery tickets, so from the middle rounds on, rank by optimistic value rather than the
  mean.
- [ ] **Availability prior**: expected games missed from each player's own multi-season
  games-played record, shrunk toward the positional base rate; scale the
  above-replacement part of the ceiling by it. Keep it a tiebreaker that never tilts one
  position against another. Know its blind spot: games *not played* conflates benchings
  with injuries.
- [ ] **News overrides beat stale projections.** A projection line that predates a
  suspension, a holdout, or a season-ending injury is wrong, and blends move slowly.
  Record each fade or boost as a dated decision with its why — and look for the
  second-order trade the market is slow to make (the faded starter's handcuff is now
  underpriced).
- [ ] Sanity-check the output against consensus rankings and *understand every large
  divergence* — each one is either your edge (a scoring quirk doing its job) or a bug.
  Both are worth finding now.

**Blanks to fill:** replacement-level N per position for your roster shape; which
positions in your ecosystem need fallback points; whether any position's projections need
regression toward the mean before VOR (spread-heavy positions like team defense often do).

## Phase 4 — Keepers (skip if your league has none)

- [ ] Surplus per keeper = their VOR minus the expected best-available VOR at the pick
  their cost consumes. Evaluate *subsets* (all combinations up to the max), not players in
  isolation — two keepers interact through the picks they occupy.
- [ ] The "best available at pick N" term needs the simulator (Phase 5) — a deterministic
  ADP cut is an acceptable v1.
- [ ] Compute each candidate's odds of *surviving* to the pick they'd cost — "could I just
  draft them there?" is the question that kills most keeper cases.
- [ ] Model opponents' keepers the same way, and note that a manager's past keeper
  behavior predicts their announcements — worth a look at their history before assuming
  they keep their best player.

**Blanks to fill:** the league's cost formula and limits; where announced keepers are
published and when.

## Phase 5 — Know your room (calibration and profiles)

This is where the league's history pays off. Public draft models assume a public room;
yours is not one.

- [ ] **Fit a noise model** from historical picks vs contemporaneous ADP: how far off
  consensus does this room actually draft? Expect the spread to *grow with ADP* and to be
  substantially larger than a naive guess. Expect positional biases (leagues commonly take
  some position a round early, systematically).
- [ ] **Per-seat profiles**: each manager's first-QB/TE/K/DST round, early-round position
  mix, reach vs consensus — shrunk toward the league prior by sample size, with
  history-less managers at league average. Two outputs: parameters the simulator reads,
  and a human-readable scouting report on the room.
- [ ] **The calming study**: draft slot vs final result over the league's history. It is
  almost certainly noise; proving it to the user is worth the hour.

**Blanks to fill:** with no league history (a new league), calibrate on public mock-draft
data and run every seat at league average — and say so in the report, so nobody mistakes
the defaults for knowledge.

## Phase 6 — Strategy search

- [ ] Simulate the draft from your slot many times per candidate strategy: opponents pick
  via the calibrated noise + their seat profiles; you follow the strategy under test;
  score each resulting roster as its best starting lineup.
- [ ] Express strategies as **positional round-windows** ("QB in rounds 3–4", "TE early
  vs mid vs late"), and search over a grid of the two or three timings your scoring makes
  interesting. The output is a grid of expected points with a best cell and — just as
  useful — the *cost of the worst cell*, which tells you how much discipline is worth.
- [ ] Treat the result as **windows, not a script**. The draft plan (Phase 7) turns the
  grid into decisions; the grid itself just prices the timings.

**Blanks to fill:** which timings matter in your league (a scoring quirk usually nominates
one position); whether openers (RB-RB vs WR-WR) or double-QB deserve cells of their own.

## Phase 7 — The draft-day plan (hand-maintained, one page)

The single page open on draft day. Hand-maintained on purpose — it is judgment, and no
regeneration should touch it.

- [ ] Your exact pick numbers, paired at the snake turns, each with its window from the
  strategy grid and the names likely on the board there.
- [ ] A **decision stack** per pick: what to check, in order, ending in a default.
- [ ] **Tiebreakers, ordered** (e.g. survival odds → ceiling in late rounds → keeper
  economics → roster fit → byes dead last), so a coin-flip never burns clock.
- [ ] Live alerts: the runs that would change your plan, the faded players you refuse at
  any price, the targets worth a reach.
- [ ] The room notes from Phase 5, distilled to a paragraph.
- [ ] A **morning-of checklist**: late news → config, one full data refresh, regenerate
  reports, start the draft-day machinery.

## Phase 8 — Live draft machinery

The architecture that makes draft day calm. One picture:

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

- [ ] **The append-only event log is the bus.** Every pick is an event; every consumer
  replays the log from the top; any writer appends through the same idempotent path. A
  crashed terminal, dashboard, or agent session loses nothing — the log is the state.
  This single decision buys crash-safety, replayability, and multi-process draft day for
  free.
- [ ] **Ingest paths, built in failover order:**
  1. *Screen-read the host's draft-results pane* (primary). A browser-automation agent
     reads the page text; a parser turns it into numbered pick events. Expect
     abbreviated names — resolve them through the pane's own position/team/initial
     columns, applied oldest-first so each resolution shrinks the pool; a
     position mismatch is refused, never guessed; a true tie is reported unmatched with
     the one-command fix. Re-ingest the *whole* pane every time (idempotent), never
     diffs.
  2. *Host API poll* (bonus — see Phase 9). Writes to the same log through the same
     append path: a peer writer, not a second pipeline.
  3. *Typed by hand* (last resort, and the fix-up tool for unmatched picks).
- [ ] **Single-writer discipline** during the real draft: one process owns ingest;
  the dashboard is read-only; nothing else holds board state in memory.
- [ ] **The forecast engine.** After each of your picks, Monte-Carlo every pick until
  your next turn using the calibrated noise and seat profiles. Two hard-won rules:
  - Each simulated world draws **one persistent noise value per player**, not fresh noise
    per pick — fresh-per-pick noise makes simulated opponents far jumpier than the
    calibration says, and your stated probabilities stop meaning anything. Verify the
    fix: top-3 hit rate should roughly match the probabilities you print.
  - Record every forecast **as issued** in a ledger keyed by board state, and grade
    later against what happened — both the freshest forecast per pick and the
    from-your-turn horizon. Honest grading is what makes "how did my predictions hold
    up" a real answer.
- [ ] **The brief**: one command that emits the compact, deterministic block an agent (or
  human) reads before your pick — board state, top candidates with survival odds, the
  forecast to your next turn, tier alerts, prediction accuracy so far. The consumer
  paste-summarizes; it never re-derives.
- [ ] **The dashboard**: a second terminal, auto-refreshing off the log — recent picks
  with prediction verdicts (hit / near / miss), the forecast, your board with survival,
  per-team value totals against an ADP baseline (who is drafting well). This is the
  spectator surface; put some flair into it.
- [ ] **Agent session protocol**, if an agent session runs the draft with you:
  - One session for the whole draft; the deterministic tools carry all memory, so the
    session's context is disposable by design.
  - **Compact right after your pick at each snake turn away** — that is when the most
    picks pass before you matter again, so compaction gets a quiet stretch. Compute the
    exact pick numbers from Phase 1's snake math and write them in the runbook.
  - After any compaction: one status command + one brief rebuilds the entire picture.
  - Keep replies to a few lines; never paste raw page text or full briefs into the
    conversation.
- [ ] **Safety rails**: the rehearsal auto-drafter *refuses* the real log's name without
  an explicit force flag; ingest never overwrites an existing pick (conflicts are
  reported, not applied); credentials stay outside the repo.
- [ ] **Rehearse, twice.**
  1. Offline: auto-draft the opponents against a practice log, make your picks, watch
     the dashboard, read the briefs — the full loop with no room.
  2. Live: most hosts offer free **mock drafts** — join one in the browser and dry-run
     the real ingest loop. The exact shape of the results pane is the one thing you
     cannot test offline, and calibrating the parser against a real room is what makes
     pick 1 of the real draft boring. Expect a handful of unmatched picks the first
     time; every one should come back with a reason and a fix command.
- [ ] Write the **ops runbook** (hand-maintained): the architecture picture, the ingest
  failover order with current status of each path, the session protocol with your
  compaction points, the rehearsal recipe, the safety rails.

**Blanks to fill:** your host's results-pane shape (find the densest "all picks so far"
view and keep it active); your compaction pick numbers; which ingest path is primary on
the day (whatever the mock rehearsal proved).

## Phase 9 — If your league host has an API

Treat API access as a *bonus path to prove*, never the plan. Hosts differ wildly:

- Some are **open and free** (Sleeper: read-only, no auth — league settings, rosters,
  draft picks, transactions, all public by league id).
- Some require **OAuth with a developer app**, and increasingly an **application-review /
  approval program** on top — meaning access can be pending for days or forever, through
  no fault of your code. Two traps proven the hard way: an authorization server may
  silently reuse a *stale cached consent* (force a fresh grant explicitly — request the
  scope by name and demand the consent screen — or you get scope-less tokens that 403
  with no explanation anywhere in the flow), and a clean consented grant can *still* 403
  if the account's developer application hasn't been approved. Budget a full end-to-end
  test **days** before the draft, not hours.
- Some have **no official API**, only unofficial cookie-based access — fragile; treat as
  scraping with extra steps.

Worth pulling, when access works, in rough value order:

- [ ] **League settings** — cross-check the hand-entered scoring vector and roster from
  Phase 1. Cheap, catches real errors.
- [ ] **Draft results** — the live pick feed. Before trusting it on draft day, *verify it
  populates mid-draft* (in a mock, if possible); some hosts only publish afterwards. If
  it works, it becomes ingest path 2: same event log, same idempotent append.
- [ ] **Host player ids** — extend the crosswalk; the id table from Phase 2 likely maps
  them already. Expect team defenses to need the name-lookup fallback.
- [ ] **League history** — drafts and standings for past seasons, replacing hand scraping
  in Phase 2 if the API reaches back far enough.
- [ ] **The in-season second life**: rosters, matchups, transactions, waivers — the same
  identity + scoring + projection machinery prices free agents and start/sit decisions
  all season. Build it after the draft, on the same seams.

**Blanks to fill:** your host's auth model and approval requirements; whether draft
results populate live; how far back history reaches; rate limits.

## Phase 10 — After the draft

- [ ] Grade the forecast ledger and write the retro as a design entry: prediction
  accuracy, where the room deviated from its profiles, which fades and targets paid off.
- [ ] Feed the new draft into the history data — next season's calibration just got one
  season better.
- [ ] Update the seams that creaked: parser shapes that needed fixing, sources that went
  stale, anything the runbook said that turned out wrong. The repo is the memory; next
  August you start from here, not from zero.
