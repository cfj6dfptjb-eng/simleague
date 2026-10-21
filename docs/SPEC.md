# NHL Hockey Simulation — Design Spec

Goal: simulate NHL hockey from a single game up to a full multi-season league, event by event. It covers:
- game engine, play-by-play and statistics
- player, staff and team ratings with human factors
- 84-game seasons and playoffs
- front office: contracts, trades, draft, waivers, minor leagues
- a human owner mode, plus team and super admin consoles

Single games should feel realistic, and aggregates over games and seasons should match real league distributions.

**Ground rules for implementation:**
- All numbers are calibration targets or defaults, not hard-coded values. Verify them against recent NHL data; open items are listed in **section 21**.
- **Every setting needs a default value and an allowed range** (min/max) in config. Where this spec gives no default, the implementer proposes one and records it in the config, with a comment and the realism test that validates it.
- **Build in phases** (section 22). Each phase must pass its realism tests (sections 10, 9.6 and 20) before the next starts.
- **Single source of truth:** each rule is defined in exactly one section. Other sections refer to it rather than restating it.

---

## 1. Approach

Use an **event-driven simulation**:
- The clock runs forward. At each moment, the current game state determines the rate of each possible event.
- Time to the next event is drawn from an exponential distribution with rate = sum of all event rates (or use small fixed ticks of 1–5 s as a simpler alternative).
- Pick which event happens with probability proportional to its rate, resolve it, and update the state.

## 2. Game structure

- 3 × 20-minute periods, with 18-minute intermissions. During intermissions the ice is resurfaced and players partly recover.
- If tied in the regular season: 5-minute **3-on-3 overtime**, sudden death.
- If still tied: **shootout**, 3 rounds, then sudden-death rounds.
- Playoff overtime, points and standings: defined once, in section 2a.
- Detailed rule edge cases (penalties in OT, stacked penalties, buzzer goals and so on): section 9.7.

## 2a. League structure (mirrors the current NHL, 2026–27)

### Teams, conferences and divisions
32 teams, 2 conferences, 4 divisions of 8 teams each.

| Conference | Division | Teams |
|---|---|---|
| Eastern | Atlantic | Boston, Buffalo, Detroit, Florida, Montreal, Ottawa, Tampa Bay, Toronto |
| Eastern | Metropolitan | Carolina, Columbus, New Jersey, NY Islanders, NY Rangers, Philadelphia, Pittsburgh, Washington |
| Western | Central | Chicago, Colorado, Dallas, Minnesota, Nashville, St. Louis, Utah (Mammoth), Winnipeg |
| Western | Pacific | Anaheim, Calgary, Edmonton, Los Angeles, San Jose, Seattle, Vancouver, Vegas |

Each NHL team has its own minor-league (AHL) affiliate (section 17.5).

### Regular season
The 2026–27 season expanded from 82 to **84 games per team**, 1,344 games in total, adding two divisional games for each team.

**Scheduling matrix (per team):**

| Opponents | Games | Detail |
|---|---|---|
| Own division (7 teams) | 28 | 4 each: 2 home, 2 away |
| Other division, same conference (8 teams) | 24 | 3 each. Against 4 of them: 2 home, 1 away; against the other 4: 1 home, 2 away. Rotates yearly. |
| Other conference (16 teams) | 32 | 2 each: 1 home, 1 away |
| **Total** | **84** | 42 home / 42 away |

**Calendar (2026–27):**
- Season runs Sep 29 – Apr 10.
- Holiday break Dec 23–25. All-Star break Feb 4–7.
- Realistic density: about 13 back-to-backs per team, with road trips and homestands typically up to 4–5 games.
- Special events: outdoor games, international Global Series games at neutral or overseas sites.

**Full league year (the season-mode calendar):**

| Phase | What happens |
|---|---|
| Training camp (September) | Camp battles; cuts to the 23-man roster. Prospects are returned to junior, college or Europe, or sent to the minors. |
| Preseason | **4 games per team** (new agreement). Veterans with 100+ NHL games play at most 2. Results don't count; the games affect roster decisions, conditioning and prospect evaluation. |
| Regular season | Sep 29 – Apr 10 (2026–27) |
| Holiday roster freeze | Around Dec 19–27: no trades or waivers. Dates are config. |
| Trade deadline | Early March, date is config. After it, trades are frozen until the end of the playoffs. **Playoff eligibility and the playoff salary cap** are judged as of the deadline. |
| International breaks | Olympics or other best-on-best events in seasons that have them (config). They add injury risk for participants and rest for everyone else. |
| Playoffs | Mid-April to June |
| Draft lottery / draft | Early May / late June (17.11) |
| Buyout window, qualifying offer deadline, interview window | Late June (17.9) |
| Free agency | Opens July 1 |
| Arbitration | Late July – August |

**Schedule generator requirements:**
- Generate a full schedule that satisfies the matrix.
- Spread games realistically across the calendar.
- Cluster road trips geographically.
- Feed travel distance and time zones into the fatigue model.

**Points:**
- Win = 2 points (regulation, OT or shootout).
- OT or shootout loss = 1 point.
- Regulation loss = 0 points.

**Standings tiebreakers (in order):**
1. Points %, or fewer games played when comparing points
2. Regulation wins
3. Regulation + OT wins (excluding shootout wins)
4. Total wins
5. Points in head-to-head games
6. Goal differential
7. Goals for

The same tiebreakers decide playoff seeding and home ice. The draft lottery and draft order use the inverse of these tiebreakers. Verify the exact order against the current NHL rulebook (section 21).

### Playoffs (Stanley Cup Playoffs)
- **16 teams qualify:** the top 3 in each division (12 teams) plus 2 wild cards per conference. Wild cards are the next 2 best teams in each conference, regardless of division.
- **First round, per conference:**
  - The division winner with more points plays the lower wild card (WC2).
  - The other division winner plays WC1.
  - Second place plays third place in each division.
- **Bracket is fixed. There is no reseeding.**
  - Round 2: the winners of each division's two first-round series play each other.
  - Round 3: Conference Final.
  - Round 4: Stanley Cup Final, East champion vs West champion.
- **Every round is best-of-7.** The team with more regular-season points has home ice, with a 2-2-1-1-1 game pattern.
- **Playoff overtime:** full 20-minute 5v5 sudden-death periods, repeated until a goal. No 3v3 and no shootout.
- Rest days between games and between series depend on the calendar. They feed into fatigue recovery and rust (13.12, 16.3).

### Configurability
The whole structure lives in config and is editable in the admin console:
- teams and their names
- conference and division assignments
- the schedule matrix and number of games
- calendar dates and breaks
- points system and tiebreaker order
- playoff qualification and bracket rules

Ship presets:
- "NHL 2026–27 (84 games)", the default
- "NHL 2021–26 (82 games)"
- custom leagues

## 3. Game state to track

- Score, period, time remaining
- Strength state: 5v5, 5v4, 5v3, 4v4, 3v3, 6v5 (empty net), etc.
- Active penalties with expiry times
- Players on ice (lines and defense pairs), shift time elapsed
- Puck zone (optional: offensive, neutral or defensive zone for each team)
- Goalie in net or pulled

## 4. Event types and base rates (per team, per game, roughly)

| Event | Target |
|---|---|
| Shots on goal | ~28–31 |
| Total shot attempts (incl. missed and blocked) | ~2× shots on goal |
| Goals | ~3 (emerges from shots × shooting %, don't set directly) |
| Shooting % / save % | ~9–10% / ~.900–.905 |
| Faceoffs (game total) | ~55–60, after every stoppage |
| Power-play opportunities | ~3 |
| Power-play conversion | ~20% |
| Hits, blocks, giveaways, takeaways | optional flavor events |

**Shot quality:** give each shot an expected-goal (xG) value based on distance, angle, rebound, rush or one-timer. Conversion probability = f(xG, shooter skill, goalie skill).

## 5. State modifiers

- **Power play (5v4):** the power-play team's shot rate is ~2× and its shot quality is higher. A minor penalty ends early if the power play scores. Penalty mix is mostly 2-minute minors, with occasional double minors (4 min) and majors (5 min, which do not end early).
- **Score effects:** the trailing team generates more shot volume, especially in the third period. The leading team sits back, giving up volume but limiting quality.
- **Goalie pull:** a team down 1 or 2 pulls its goalie with ~1–3 minutes left. The 6v5 offense rate jumps, and empty-net goals against become likely. Teams down 1 tie it about 15–20% of the time.
- **Home ice:** target ~53–55% home win rate. **This is a total budget, not a separate multiplier.** It is split among last change, the faceoff stick-down rule, crowd energy, referee home bias and travel fatigue. See the effect budgets in 20.2.
- **Line changes and fatigue:** shifts last ~40–50 s. Rotate 4 forward lines and 3 defense pairs. Top lines get more ice time. Use dedicated power-play and penalty-kill units in those situations. All fatigue mechanics are defined once, in 13.1.

## 6. Overtime and shootout

- **3v3 OT:** much higher scoring rates than 5v5. Roughly a quarter of games reach OT, and a majority of OT games end before a shootout.
- **Shootout:** each attempt scores ~30–35% of the time, adjusted by shooter and goalie ratings.

## 7. Team and player strength

- **Phase 1 bootstrap only** (section 22): team offense, team defense and goalie save ratings that scale shot rate and conversion. This simple model lets the engine be calibrated before the player model exists.
- **Target model:** individual player ratings (section 12), changed by human factors (section 13). Who is on the ice changes rates moment to moment. Goals are credited to a shooter, and assists to 0–2 teammates.
- The simple team ratings are then **derived** from the player model, for display and AI decisions only. They are no longer an input to the engine.

## 8. Core loop (pseudocode)

```python
while not game_over:
    rates = compute_rates(state)        # shots, penalties, stoppages, per team
    dt = exponential(sum(rates))        # time until next event
    if state.time_left <= dt:
        end_period(state); continue
    state.advance(dt)                   # also expires penalties, updates shift time
    event = weighted_choice(rates)
    resolve(event, state)               # shot -> goal?, penalty -> PP, stoppage -> faceoff
    maybe_line_change(state)
    maybe_pull_goalie(state)
```

## 9. Game engine: how a game runs, play-by-play and statistics

The core loop in section 8 is the skeleton. This section defines how a game actually flows and what it records. The goal: a simulated game's play-by-play and box score should be indistinguishable in shape from a real NHL game's.

### 9.1 Pre-game
1. **Load game context:**
   - teams, venue, date, schedule fatigue
   - officials crew (13.9)
   - attendance and pre-game crowd energy (13.10)
   - occasion modifiers (13.5)
2. **Lineups:** each team's coach AI (or owner, 18.2) sets the 20-man lineup, lines, defense pairs, special-teams units and starting goalie. Scratches and injuries are recorded.
3. **Pre-game player modifiers** (13): form, confidence, fatigue, streaks. Stored per player for the game log.
4. **Random seed** stored with the game, so any game can be **replayed exactly**.

### 9.2 In-game flow: a possession and zone state machine
Instead of only firing events at random, the engine tracks **who has the puck and where**. Events then come out of that flow naturally.

**States:**
- Faceoff
- Puck possession: team, zone (offensive, neutral, defensive) and type (controlled, loose puck, dump-in, rush, cycle)
- Stoppage

**Typical sequence:**
1. **Faceoff:** the centers' faceoff ratings and the linesman decide the winner. The rule that the visiting center puts his stick down first applies.
2. **Possession phases:**
   - defensive-zone breakout: success, or a turnover
   - neutral-zone transition
   - **zone entry:** carry-in, dump-in or pass-in, which can fail into an offside or a turnover
   - offensive-zone play: cycle, point shot, net-front battle
3. **Shot attempt:**
   - The shot gets a location (x/y coordinates), a shot type, rush/rebound/one-timer flags, and an xG value.
   - **Outcome:** blocked, missed (wide, over, post or crossbar), or on goal. A shot on goal is either a save or a goal.
4. **After a save:** freeze (stoppage), rebound (a second chance within ~1–3 s), deflection to the corner, or the defense clears it.
5. **Stoppages**, each with a reason drawn from realistic frequencies:
   - goalie freeze
   - icing (the icing team can't change lines, and the faceoff moves to its zone)
   - offside
   - puck out of play / over the glass (delay-of-game penalty if shot over the glass from the defensive zone)
   - hand pass, high stick
   - net off its moorings
   - penalty, injury
   - **TV timeouts:** at the first stoppage after set game-clock marks, 3 per period in a standard NHL broadcast. Configurable.
6. **Faceoff location** follows the stoppage reason: the zone of the icing team, the defensive zone after a goalie freeze, the neutral zone after an offside, center ice after a goal.

**Line changes:**
- On the fly during play, triggered by shift length, fatigue and possession. Changing while the puck is in your own zone is risky.
- At stoppages.
- Bad changes can cause a too-many-men penalty, or an odd-man rush against.

**Penalties:**
- **Delayed penalty:** play continues until the penalized team touches the puck. Meanwhile the other team pulls its goalie for an extra attacker.
- **Coincidental penalties** lead to 4v4. Overlapping penalties lead to 5v3.
- Penalties expire on time or after a power-play goal (minors only; majors serve their full time).
- Penalty shots for fouls on breakaways.

**In-game coach and owner actions:** line juggling, timeouts, goalie pulls, coach's challenges and matchup choices. Made by the coach AI, or live by a human owner if he is watching.

### 9.3 Play-by-play event log (every event)
Modeled on the NHL's public play-by-play data.

**Event types:**
- Period start / period end, game end
- Faceoff
- Shot on goal, missed shot, blocked shot
- Goal
- Hit, giveaway, takeaway
- Penalty (including delayed), penalty shot, penalty expiry
- Stoppage, with its reason
- Line change, including shift start/end for every player
- Goalie pulled / goalie returned
- Challenge or review, with its outcome
- Injury (player leaves, returns, or is out for the game)
- Fight, scrum, ejection
- Shootout attempt
- Engine-only flavor events: momentum swings, surprise-factor sparks (13.8), crowd-energy changes

**Fields recorded for each event:**
- Event id, period, period clock (time remaining and elapsed), overall game time
- Event type and reason; for shots, the shot type: wrist, snap, slap, backhand, tip-in, deflection, wrap-around
- Team, and the primary/secondary/tertiary players involved: e.g. shooter, assist 1, assist 2, goalie, blocker, hitter and the player hit, faceoff winner and loser, penalized player and the player who drew the penalty
- **Coordinates (x, y) and zone**
- Shot details: distance, angle, xG, rebound/rush/one-timer flags, missed-shot reason
- Strength state (5v5, 5v4, 4v4, 6v5, ...) and a situation code
- Players on ice for both teams; empty-net flag
- Score after the event
- Penalty details: infraction, severity, minutes, and the true-event vs called info (13.11)
- Modifier snapshot: key active modifiers such as fatigue, momentum and sparks, stored for debugging and the "why" explanations

**Readable play-by-play text:**
- Each event gets one natural-language line with player names and varied phrasing, so repetitive situations don't read the same every time.
- Example: "12:41 2nd — Larsen wins the draw back to Kovač; point shot blocked by Marsh."
- Big moments get a richer description: goals, fights, highlight-reel saves, controversial calls.
- **Commentary levels:** compact log, standard, and a "broadcast" style with more color.

### 9.4 Statistics (computed from the event log)
**All statistics are derived from the event log**, never tracked separately, so they always match.

**Per skater:**
- Goals, assists (primary and secondary), points, +/-
- PIM and penalties drawn
- Shots on goal, missed shots, shots blocked (by opponents), total shot attempts
- Blocks, hits, hits taken
- Giveaways and takeaways
- Faceoffs: won, lost, win %, and by zone
- **Ice time:** total, even strength, power play and shorthanded; number of shifts and average shift length
- Individual xG; on-ice shot attempts for/against (Corsi), unblocked attempts for/against (Fenwick), xG for/against, and high-danger chances for/against
- Game-winning goal, empty-net goals, power-play and shorthanded points

**Per goalie:**
- Shots against, saves, goals against, save %
- Ice time and decision (W/L/OTL)
- Splits by strength state and by danger level
- Expected goals against and goals saved above expected (GSAx)
- Rebounds allowed, shutout flag, whether he was pulled
- Shootout record

**Per team:**
- Goals and shots by period
- Power plays: opportunities, goals, PP%; and penalty-kill %
- PIM, faceoff %, hits, blocks, giveaways and takeaways
- Shot attempts for/against (all situations and 5v5), xG, high-danger chances
- Time in the offensive zone (optional), time on attack by strength state
- Empty-net time; challenges (used, won)

**Game summary:**
- Final score with the result type: regulation, OT or shootout
- **Scoring summary** with goal times, scorers, assists and strength
- **Penalty summary**
- **Three stars**, picked by a game-score formula
- Attendance, officials, venue
- Game duration on the clock, plus an estimated real-world duration (which a long night with many stoppages, reviews and fights lengthens)
- Shot charts and heat maps from the coordinates
- A "story of the game" paragraph built from key events and modifier explanations

**Season aggregation:**
- Game stats roll up into season, career, playoff and minor-league stats.
- These feed standings, award races, ratings drift (16.2), contract valuation (17.8) and scouting.

### 9.5 Output formats and viewing
- **Structured log:** JSON per game, in an NHL-play-by-play-like schema. CSV export for events, shifts and box score.
- **Live viewing:** the event stream drives a game viewer with a rink map, score bug, clock and running commentary. Speed control: real-time, 2×, 10×, instant, or jump to the next key event.
- **Pause points:** the owner can pause, adjust lines or tactics, call a timeout or pull the goalie during a live game (18.2).
- **Results-only mode:** for mass simulation (other games in the league, minor leagues, validation batches). Optionally skips text generation and stores only the event log and box score, or just the box score.
- **Replay:** the stored seed plus settings reproduce any game exactly.

### 9.6 Realism checks for game output
Added to the calibration in section 10. Compare per-game distributions against real NHL play-by-play data:
- Faceoffs, hits, blocks, giveaways and takeaways per game
- **Stoppage count and the mix of stoppage reasons**
- Shift lengths and ice-time distributions per lineup slot
- Shot type and shot location distributions
- Rebound and rush shot share
- Delayed-penalty and empty-net situations, 5v3 frequency
- Fights per game
- Game duration

### 9.7 Rule edge cases (must be handled)
Each case below needs a unit test. Verify details against the current NHL rulebook.

**Penalties and strength:**
- **No team ever has fewer than 3 skaters.** Extra penalties are "stacked": the next penalty starts only when an earlier one expires.
- Coincidental minors don't change strength unless the rulebook says so (e.g. 4v4 for coincidental minors when both teams are full strength). Coincidental majors don't change strength.
- **Penalties in 3v3 OT:** the non-penalized team **adds a skater** (4v3, 5v3) instead of the penalized team dropping to 2. When the penalty expires, play returns to the matching 3v3 / 4v4 / 5v5 strength.
- A penalty is carried over between periods and from the 3rd period into OT.
- **Goalie penalties** are served by a skater who was on the ice. Goalie fights and leaving the crease are handled separately.
- **Delayed penalty:**
  - If the penalized team never touches the puck before a goal, the goal counts and the minor is cancelled. Majors still apply.
  - **Own goal into the empty net** during a delayed penalty: rare but real. It counts for the opponent, credited to the last opponent who touched the puck.

**Goals and reviews:**
- **Goals at the buzzer:** the clock is precise to tenths of a second, and the video review decides.
- **Goals that hit an official,** high-stick goals and kicked-in goals: disallowed, after review where applicable.

**Shootout:**
- Each team's shooters are chosen in order.
- **No skater shoots twice until every eligible skater on the team has shot.**
- Goalies can be changed between rounds.

**Playoff overtime:**
- 20-minute periods with full intermissions and ice resurfacing.
- Fatigue carries over through every overtime.
- Ice quality decays faster in long games.
- Multi-overtime games add extra wear for the rest of the series.

**Injuries and short benches:**
- **Emergency backup goalie (EBUG):** follow the new agreement's procedure. A registered EBUG is named before each game. If both goalies are injured, the EBUG plays with very low ratings.
- **Short bench:** if injuries or ejections drop a team below the minimums, it plays short. Forwards can play defense with an off-position penalty. The next game gets an emergency call-up.
- **Not enough healthy players before a game:**
  - emergency recall from the minors
  - goalie emergency recall
  - if neither is possible, the team plays with fewer players. Never forfeit.

**Rules that change by game state:**
- Icing (the icing team can't change lines)
- Hybrid icing
- Offside (including the delayed-offside tag-up)
- Too many men on the ice
- Hand pass
- High stick
- Playing the puck with a high stick before a goal

## 10. Calibration and validation

- Run 10,000+ simulated games between average teams.
- Compare against real-season distributions:
  - goals per team per game and the total goals distribution
  - shots per team per game
  - share of one-goal games
  - OT frequency and shootout frequency
  - power-play % and penalty-kill %
  - empty-net goals per game
  - home win %
- Tune base rates until the aggregates match. Individual games should still show natural variance, such as lopsided shot counts with close scores.
- **Calibrate outcomes, not causes.** Where several mechanisms produce the same outcome, tune them together until the final number matches. Examples:
  - **Penalties:** true infractions, phantom calls, retaliation and evening-up together must still give ~3 power plays per team per game.
  - **Fights:** fight rate per game should match the current NHL, which is low and declining.
- Season- and multi-season-level calibration (standings spread, streaks, upset rates, rating drift): section 20.

## 11. Suggested project layout

```
nhl_sim/
  config.py        # all base rates and modifiers in one place
  state.py         # GameState dataclass
  rates.py         # compute_rates(state)
  events.py        # resolve shot / penalty / faceoff / stoppage
  lines.py         # line rotation, PP/PK units, goalie pull logic
  overtime.py      # 3v3 OT + shootout
  game.py          # main loop: possession/zone state machine, stoppages, line changes
  pbp.py           # event log schema, coordinates, readable play-by-play text
  stats.py         # box score, player/goalie/team stats derived from the event log
  viewer/          # live game viewer (rink map, score bug, commentary, speed control)
  teams.py         # team/player ratings (phase 1: simple team ratings; later derived from players)
  players.py       # player model, profile, roster legality checks
  modifiers.py     # human factors, effect budgets, team-level cap, decay/ceilings
  morale.py        # morale, team culture
  medical.py       # injuries, illness, recovery, re-injury
  staff.py         # staff ratings, contracts, staff market, owner patience, firings
  validate.py      # run N games, print aggregate stats vs targets
  officials.py     # referee/linesman model, calls, reviews
  crowd.py         # attendance, crowd energy/mood, venue
  discipline.py    # retaliation, fights, ejections, suspensions
  frontoffice.py   # GM AI, trades, waivers
  contracts.py     # contract model, cap rules, validation of structure rules
  valuation.py     # fair-value / salary recommendation model
  freeagency.py    # RFA/UFA, qualifying offers, offer sheets, arbitration
  negotiation.py   # owner-mode negotiation engine, agents, calendar windows
  draft.py         # draft class generation, scouting, lottery, draft order, draft day
  waivers.py       # waiver wire, exemptions, claim priority, AI claims
  minors.py        # AHL affiliates, call-ups/send-downs
  season.py        # schedule, standings, development/aging, playoffs
  admin/           # super admin console: league/game-wide config, presets, locks, overrides
  team_console/    # per-team owner console: lineup, tactics, staff, front office, delegation
  auth.py          # roles and permissions (super admin, owner, delegates), audit log
  presets/         # saved config presets (JSON/YAML), incl. fictional-league preset
  integrity.py     # anti-exploit checks, hidden league seed
realism/           # realism report generation and stored reports per version
tests/
  rules/           # unit tests for every league/contract/game rule
  properties/      # invariants (skater minimums, stats reconcile, cap legality)
  exploits/        # scripted "bad owner" strategies that must fail
```

## 12. Rosters and player ratings

### Roster shape
- 23-player active roster. 20 dress for each game: 12 forwards (4 lines), 6 defensemen (3 pairs) and 2 goalies. The remaining 3 are healthy scratches who rotate in for injuries or slumps.

**Roster legality, checked after every transaction and before every game:**
- Active roster ≤ 23
- Exactly 2 goalies dressed (EBUG rules in 9.7)
- 50-contract limit
- Cap compliance **every day** (17.7), including the playoff cap
- Minor-league veteran rule for the affiliate's lineups
- Injured reserve / long-term injured reserve placements valid
- Waiver status respected for every move

An illegal state blocks the action (for the owner, with a clear explanation) or makes the AI fix it automatically.

### Player profile (non-rated attributes)
- Position(s): primary, plus secondary positions with an off-position penalty (e.g. a center on the wing: small; a forward on defense: large).
- **Handedness** (shoots left/right). Effects:
  - Playing on the off-wing changes shooting angles: helps one-timers, hurts carrying the puck on the forehand.
  - **Defense pairs** do better with one left- and one right-shot defenseman.
  - **Power-play units** want one-timer options from both sides.
- Height and weight: feed physicality, net-front battles, hit outcomes and injury risk; small effect.
- Birthdate and age
- Nationality: hometown and Global Series games, international breaks
- Draft history
- Injury history

### Lineup composition effects
- **Balance penalties:**
  - too many centers or no natural centers
  - defense pairs with no right-shot defenseman
  - no penalty-kill specialists on the roster
- **Line chemistry** (16.2), and how well the coach's **system fits** the roster (17.14).

### Rating scale
- Each attribute is rated 1–99, with 50 = a league-average regular.
- Each attribute converts to a multiplier on a rate or probability. For example, every 10 points above 50 adds 4–6% to the relevant rate. These multipliers are tunable.
- Talent tapers down the lineup:

| Slot | Typical overall |
|---|---|
| 1st-line F / top-pair D | 75–90 (stars 90+) |
| 2nd line / 2nd pair | 65–75 |
| 3rd line / 3rd pair | 55–65 |
| 4th line / depth | 45–55 |
| Starting goalie | 70–85 |
| Backup goalie | 55–65 |

### Forward attributes
| Attribute | Drives |
|---|---|
| Shooting | Conversion beyond xG |
| Shot volume | How often this player is the shooter |
| Playmaking | Linemates' shot quality, assist credit |
| Offensive sense | High-danger chance rate |
| Skating | Rush chances, zone entries |
| Puck handling | Fewer giveaways |
| Defensive play | Opponent shot suppression while on ice |
| Faceoffs (centers) | Faceoff win % |
| Physicality | Hits, puck battles |
| Discipline | Penalty rate (lower is better) |
| Stamina | How quickly performance drops over a shift or game |

Role tags (sniper, playmaker, two-way, grinder, PP specialist, PK specialist) guide the coach AI's deployment.

### Defenseman attributes
- Shared with forwards: skating, puck handling, physicality, discipline, stamina.
- Defense-specific:
  - Gap control / positioning: opponent shot rate and quality
  - Shot blocking: share of opponent attempts blocked
  - Breakout passing: faster zone exits
  - Point shot: blue-line shot volume, mostly low xG
  - Offensive activation: how often they join the rush, which adds offense but also counterattack risk
- Ice time: top pair ~22–25 min, third pair ~14–16 min.

### Goalie attributes
- Overall save ability: baseline save % adjustment
- Separate high-, medium- and low-danger save ratings
- Rebound control: fewer rebound chances against
- Positioning / lateral movement: performance against cross-ice passes and one-timers
- Puck handling: small effect on breakouts
- Consistency: game-to-game variance
- Back-to-back fatigue: drives the backup starting ~25–35% of games
- Shootout ability: used only in shootouts

### Hidden player attributes
- Consistency: game-to-game variance in normal games
- Injury proneness
- Streakiness
- Pressure handling is **not** a separate rating here. Clutch, big-game, mental toughness and experience all share one pressure budget (20.2), so they can't stack. Clutch stays tiny.
- Age curve (season mode): forwards peak ~24–28, defensemen slightly later, goalies ~26–31

### Coaching staff
Coaches change decisions and tendencies, not raw skill.
- **Head coach**
  - System style (offensive vs defensive): shifts the team's shot-for and shot-against rates together
  - Line deployment: shorten the bench vs roll four lines
  - Goalie-pull aggressiveness
  - Timeout usage
  - Discipline culture: team-wide penalty-rate modifier
  - Motivator rating: see section 13
- **Assistant coaches:** power-play coach (power-play quality and conversion), penalty-kill coach (shorthanded suppression), defense coach (small defensive boost)
- **Goalie coach:** goalie consistency and development
- **Video coach:** success rate on coach's challenges

### Support staff
Mostly relevant in season mode.
- Athletic trainers / medical staff: injury rate, recovery time, diagnosis accuracy (13.15)
- Strength and conditioning: team stamina, reducing late-game and back-to-back fatigue penalties
- Analytics staff: better line matchups for the coach AI, better trade and contract valuations
- Scouts: draft and trade scouting accuracy (17.11)
- Equipment staff: flavor only

Staff contracts, hiring market, relationships and team culture: section 17.14.

### Flow into the engine
1. Before the game, the coach AI sets the lineup and the power-play and penalty-kill units.
2. The **effective ratings** (section 13) of the 10 skaters and 2 goalies on the ice combine into on-ice offense and defense values, which scale the base event rates.
3. Coach modifiers apply on top.
4. Staff effects that work between games (injury recovery, conditioning) run outside the single-game sim.
5. Calibration: a league of average rosters must still hit the section 10 targets. Adjust the multipliers, not the base rates.

## 13. Dynamic performance ("human factors")

Every player keeps a fixed **baseline** rating per attribute. Each game, and within each game, the engine computes an **effective rating**:

```
effective = baseline × (1 + clamp(sum(modifiers), -cap, +cap)) × fatigue_factor
```

- Default cap: ±12–15%.
- `fatigue_factor` is applied separately, so it can always pull performance down even when the mood modifiers are positive.
- **Team-level cap:** the average modifier across a team's dressed players is also capped, default ±6%. This prevents a whole team being "hot" through stacked team-wide effects (streaks, crowd, occasion, sparks, coaching).
- Modifiers are **grouped into effect budgets** (20.2): home advantage, pressure handling, comeback push and fatigue. Inside a budget, sources share one calibrated total instead of adding up independently.
- Every modifier that depends on recent results **decays toward zero** (20.3), so no feedback loop can run away.

Most modifiers should average **near zero** across the league, so the human touch adds variance and storylines without breaking calibration. Fatigue is the exception: it is a real net negative, already reflected in league averages.

### 13.1 Fatigue (physical)
- **In-shift:** ratings decay as a shift runs long (e.g. beyond 45 s), scaled by stamina.
- **In-game load:** cumulative ice time. Heavy-minute players fade late in the third period and in OT.
- **Schedule:** back-to-backs, 3 games in 4 nights, long road trips, time-zone travel (east–west), rest days restore. Goalies are hit hardest by back-to-backs.
- **Season load (season mode):** a slow accumulation, recovered by rest. Reduced by the conditioning staff.

### 13.2 Daily form ("feeling for the game")
- A per-player daily form value drawn as a **mean-reverting random walk**:
  ```
  form_today = ρ × form_yesterday + noise
  ```
  with ρ ≈ 0.6–0.8 so good and bad stretches persist for a few games. Noise size is scaled by the consistency attribute, so low-consistency players swing more.
- Streakiness raises ρ, making hot and cold runs last longer.

### 13.3 Confidence (personal recent performance)
- Built from the player's recent games with exponential decay. The **previous game carries the most weight**.
- Skaters: points, shots, on-ice goal differential, costly mistakes such as penalties that led to goals or turnovers that led to goals.
- Goalies: save % vs expected (goals saved above expected), and whether they were pulled.
- A **mental toughness** trait dampens the effect of bad games, so tough players bounce back faster.
- Confidence feeds mostly into finishing attributes (shooting, playmaking) and goalie save ratings.
- **Anti-loop rule:** confidence is based on performance **relative to expectation** for that player (e.g. goals vs expected goals, saves vs expected saves), not raw results. A star scoring as usual doesn't keep getting more confident. Confidence always decays toward neutral (20.3).

### 13.4 Team momentum (streaks)
- Win streaks give a small team-wide boost, with diminishing returns. A 6-game streak should not be 3× a 2-game streak.
- Losing streaks give a small negative, but after a threshold (e.g. 4+ losses) a **"desperation / reset"** effect can kick in. Examples: a players-only meeting, or the coach juggling lines, which partly offsets the slump.
- **Leadership** (captain and alternates) dampens the team's emotional swings in both directions.
- A coach's **motivator** rating also dampens slumps.
- The streak effect has a **hard ceiling**, and it **counts toward the team-level cap**. In tests, streak length distributions must match real NHL data (20.1).

### 13.5 Opponent and occasion motivation
Pre-game modifiers based on context:
- Rivalry and division games: slight boost for both teams. Also raises physicality and penalty rates.
- **Revenge game:** a player facing his former team, or a rematch after a blowout loss.
- Facing a top team: focus boost. A heavy favorite playing a weak team gets a small **"trap game"** penalty, especially on a back-to-back.
- Home crowd: part of the **home-advantage budget** (20.2), not an extra boost on top of it. Scaled by an arena "loudness" setting.
- Playoff race (late season): teams in contention get a boost. Eliminated teams get a small motivation dip, partly offset by young players auditioning for roles.
- Milestones, contract year, return from injury (with rust initially): small player-level modifiers.

### 13.6 In-game momentum
- Short-lived swings that decay over ~2–5 minutes:
  - after scoring, or after a big save in a scoring chance
  - after a fight or a big hit
  - after killing off a penalty
- Being blown out can trigger "giving up" (lower effort) in the trailing team, or a push. Make this configurable.
- Comeback-related effects (score effects in section 5, in-game momentum, desperation, sparks) share the **comeback budget** (20.2). Comeback rates, e.g. how often a team down 2 goals in the 3rd period still wins, must match NHL data.
- Keep these effects small. Real-world evidence for in-game momentum is weak, so it should mostly add flavor.

### 13.7 Personality traits (hidden, per player)
These control how strongly each player reacts to the factors above:
- Mental toughness: sensitivity to confidence drops
- Streakiness: form persistence
- Big-game player: response to rivalry and high-stakes games
- Leadership: dampens the whole team's swings
- Temperament: penalty and fight tendency under emotion

### 13.8 Surprise factor (coach- and staff-driven sparks)
Short, unexpected performance boosts during a game. They represent a well-timed tactical adjustment, a fiery intermission speech, a timeout that resets the bench, or a video-room insight.

**Driver ratings:**
- Each coach and key staff member gets an **experience** rating (years and games coached, playoff runs) and an **innovation** rating (tactical creativity).
- Combined team "spark potential" = a weighted blend of:
  - head coach experience and innovation, weighted highest
  - assistant coaches
  - video coach and analytics staff
  - the head coach's motivator rating

**Trigger moments.** Each moment has a probability scaled by spark potential:
- **Intermission adjustment:** checked after the 1st and 2nd periods. More likely if the team is trailing or has been outshot.
- **Timeout:** when the coach calls a timeout, for example after conceding goals quickly or before a late power play or a 6v5.
- **Line juggling or a tactical switch:** when the coach changes lines or system mid-game, e.g. a new forecheck or a different matchup.
- **Video insight:** the staff spots an opponent tendency. This boosts a specific attribute, e.g. power-play shot quality or breakout success, rather than everything.
- **Rare random spark:** an unprompted moment such as a veteran's bench talk. Very low probability, and tied to team leadership.

**Effect:**
- Temporary boost to one player, one line, a special-teams unit, or the whole team. Default +5–10% on targeted attributes.
- Decays over ~3–8 game minutes, or lasts until the next intermission.
- Stacks with other modifiers but stays inside the overall modifier cap. The cap can be raised separately for surprise effects.
- **Opponent adaptation:** the opposing coach's experience reduces the boost's size and duration, since experienced coaches counter faster.
- **Diminishing returns:** each additional spark in the same game is weaker. A cooldown prevents back-to-back triggers.
- **Optional negative version (off by default):** a tactical change that backfires. Its probability is higher for low-experience staff.

**Season and playoff links:**
- Staff experience grows over seasons, as player experience does.
- In the playoffs, sparks are slightly more likely but opponent adaptation is stronger, since teams know each other well during a series.

**Logging:** each spark is written to the play-by-play, e.g. "2nd intermission: coach switches to a 1-3-1 forecheck; line 2 boosted for ~6 minutes", so its impact is visible.

### 13.9 Officials (referees and linesmen)
Each game is assigned a crew of 2 referees and 2 linesmen. Each official has ratings.

**Referee ratings:**
- **Experience:** NHL games worked, playoff games worked.
- **Strictness:** baseline call rate. Strict crews call more penalties, which means more power plays.
- **Consistency / accuracy:** chance of a missed call (a real infraction not called) and of a phantom call (a penalty called with no real infraction).
- **"Let them play" tendency:** fewer calls late in close games and in OT ("swallowing the whistle"), and stronger in the playoffs.
- **Game management:** ability to keep a chippy game under control. Inexperienced crews can lose control, which escalates physicality, retaliation and fights.
- **Evening-up tendency:** after a run of calls against one team, the next call is more likely to go against the other team. This is a real, observable pattern in penalty counts.
- **Home bias:** very small, scaled by crowd energy (13.10).

**Linesman ratings:**
- Offside and icing accuracy
- Faceoff management: tossing centers out of the circle, which slightly affects faceoff win %
- Fight breakup speed

**Referee "game control" factor:** the officials' counterpart to the surprise factor. When a game turns chippy (many hits, scrums, retaliation), the crew's experience and game-management ratings decide whether they calm it down or let it escalate.
- Calming it down: warnings, coincidental minors.
- Letting it escalate: more fights, more misconducts, more injuries.
- Low-experience crews also have a small random chance of a **controversial call** at a key moment, which feeds into 13.11.

**Video review and challenges:**
- Situation-room reviews: goals that were kicked in, high-sticked or that crossed the line.
- Coach's challenges: offside and goalie interference. Success depends on the video coach rating (13.8) and the true outcome.
- A **failed challenge** results in a delay-of-game minor.
- An **overturned goal** causes a sizeable momentum swing for both teams and the crowd.

### 13.10 Crowd, attendance and venue

**Attendance (pre-game):** fill % of arena capacity, from:
- home team success: win %, current streak, playoff race position
- opponent draw: rivalry, star players, Original Six opponents
- day of week (weekends higher) and time of season
- market size, plus a team "fan loyalty" rating
- playoffs: near sellout by default

Traveling fans: a share of away supporters, higher for nearby rivals. They reduce the home crowd's effect.

**Crowd energy (live, 0–100):**
- **Pre-game baseline** from attendance, team streak, rivalry and the occasion (home opener, tribute or jersey-retirement night, playoff game).
- **During the game:**
  - Energy rises: home goals, big clean hits, fights, spectacular saves, killing off a penalty, a late push or 6v5.
  - Energy falls: trailing big, long stretches without shots, late in a blowout (fans leave early, and energy can't recover fully).
- **Mood (separate from energy):**
  - positive (excited)
  - anxious (close game, late)
  - angry (a bad call against the home team, an uncalled hit on a home player): energy stays high, but it's directed at the referees and the opponent
  - disgruntled (booing the home team during losing streaks)

**Crowd effects:**
- The home boost scales with energy and mood. Positive energy lifts the home team, and anxious crowds slightly raise the home team's nerves.
- Visiting team composure: low-experience and low-mental-toughness visitors are affected more.
- Referee home bias scales with energy and angry mood. Keep this very small.
- Angry crowds after a bad call increase the home team's physicality and retaliation chance.
- A disgruntled home crowd can lower home players' confidence.

**Venue factors:**
- **Last change:** the home team gets the last line change, so it can pick favorable matchups. This is an important, real home advantage.
- **Faceoff rule:** on faceoffs, the visiting center must put his stick down first, giving a small home faceoff edge.
- **Ice quality:**
  - What lowers it: older arenas, warm climates, late season and playoffs, events held in the arena the day before.
  - What it causes: bouncing pucks, lower passing and shooting accuracy, more randomness.
  - Ice also degrades during each period and is refreshed at each intermission.
- **Board and glass liveliness:** affects rebounds off the end boards.
- **Altitude** (e.g. Denver): extra visiting-team fatigue.
- **Outdoor games:** wind, cold, sun glare, poor ice. Randomness and fatigue both go up.

### 13.11 Calls and their consequences
Every infraction is generated first as a **true event** (what actually happened). The officials model then decides whether it is called. Each combination of true event and call has its own consequences.

**Big hits:**

| What happened | Called? | Consequences |
|---|---|---|
| Clean big hit | No call (correct) | Hitting team and crowd momentum up; injury chance for the hit player |
| Clean big hit | Penalty (phantom) | Hitting team frustrated (discipline down); opponent power play; home crowd angry if the home team was penalized |
| Illegal hit (head contact, boarding, charging) | Minor / major / match penalty | Power play (5 minutes for a major, not ended by goals); possible game misconduct; injury to the hit player; victim's team rallies or loses composure (depends on leadership) |
| Illegal hit | **Not called** | Victim's team frustrated: discipline drops and retaliation chance rises sharply; crowd angry (if home victim); evening-up pressure on refs; flagged for post-game review (suspension check) |

**Other foul play** (slashing, cross-checking, spearing, butt-ending, slew-footing, kneeing, elbowing, embellishment and diving):
- Same model: true event, then call or no call, then consequences.
- Dangerous fouls carry injury risk and a supplemental discipline check.

**Retaliation and escalation:**
- Retaliation chance depends on:
  - the severity of the original act
  - whether it was called
  - the temperament of the players involved
  - team leadership (dampens it)
  - crowd mood
  - the crew's game management
- Possible outcomes: retaliation minor, scrum (coincidental minors), fight, or a dirty retaliatory hit.
- Retaliation is often the penalty that gets called, so the original offender's team can benefit from it.
- **Fights:**
  - 5-minute majors for both players.
  - A momentum swing to the team whose player wins the fight; crowd energy up.
  - Possible instigator penalty plus a misconduct.
  - Small injury risk.
  - Teams with an "enforcer"-type player draw fewer liberties against their stars.

**Game misconduct / match penalty:**
- The player is ejected.
- His team plays short: 11 forwards or 5 defensemen. Remaining players get more ice time, so fatigue rises.
- The coach AI reshuffles lines.

**Supplemental discipline (between games):**
- A "player safety" review runs after games for flagged incidents, including uncalled ones.
- Outcome: no action, fine, or suspension of N games.
- Suspension length depends on:
  - severity
  - injury caused to the victim
  - repeat-offender history (tracked per player)
  - intent
- Suspended players miss future games, and the team calls up a replacement.

**Bad blood:** serious incidents carry into the next meeting between the two teams. Physicality, retaliation and crowd intensity are all raised in the rematch.

**Other referee-call events:**
- Penalty shots for a foul on a breakaway.
- Delay of game for shooting the puck over the glass.
- Too many men on the ice.
- Unsportsmanlike conduct and bench minors (e.g. a coach arguing calls).
- Goalie interference rulings.

### 13.12 Other factors (additional, evaluated)
Each factor below is a toggleable modifier with its own weight. Factors marked *(flavor)* should stay small.

**Line matchups:** the home team's last change and the coach AI's matchup strategy (top line vs the opponent's top line, or a shutdown line vs the opponent's top line).

**Goalie decisions:**
- Starter vs backup choice: rest, back-to-backs, hot hand.
- Pulling a goalie after a bad stretch, which can also spark the team (13.8).

**Equipment and small random events:**
- Broken stick: the player is ineffective until he gets a new stick.
- Lost helmet or skate issues *(flavor)*.
- Puck deflections off skates or bodies, and goals off the end boards: pure randomness.

**Illness and injuries:** defined in 13.15.

**Rest vs rust:** long breaks such as the all-star break, holidays or a long wait between playoff rounds give some teams **rust** (sharpness down), even as fatigue recovers.

**Emotional and off-ice events:**
- Playing in his hometown or in front of family *(flavor)*.
- Tribute nights, milestone games, a first game back after a trade.
- **Trade deadline distraction** for players in trade rumors.
- Contract-year motivation.
- Media pressure in large hockey markets: amplifies confidence swings for the home team.

**Standings incentives:**
- Teams in the playoff race get a motivation boost late in the season.
- Eliminated teams give young players more minutes.
- Already-clinched teams rest stars.

**Scorekeeper bias** *(stats only)*: home-arena scorers record hits, giveaways and takeaways at different rates. This affects only the recorded stats, not the game itself, but makes generated box scores more realistic.

**Rule variants:** hybrid icing, offside rules, the playoff penalty standard and the outdoor-game ruleset all live in config.

### 13.13 Transparency
- Each game logs every player's modifier breakdown (fatigue, form, confidence, momentum, occasion), so the box score can explain why a player was hot or cold, e.g. "playing on a 5-game point streak, rivalry game."
- A master **"human factors intensity"** slider (0 = pure baseline, 1 = default, >1 = dramatic), plus per-factor on/off toggles.
- Validation runs should report results both with and without human factors, to confirm the aggregates stay on target.

### 13.14 Morale (separate from confidence)
**Confidence** is about recent on-ice performance. **Morale** is about how happy a player is with his situation. Morale changes slowly and affects effort, consistency and contract talks.

**Inputs:**
- role and ice time vs what he expects or was promised
- team success
- contract satisfaction: underpaid, or in a stalled negotiation
- relationship with the coach (17.14)
- trade rumors
- being shuttled between leagues
- locker-room culture
- personal factors: hometown, family, being far from home

**Effects:**
- A small effect on effort-related attributes and on consistency.
- Bigger effects off the ice: willingness to re-sign, trade requests, hometown discounts, free-agent destination choice.
- Very low morale can lead to a **trade request**. Visible to the owner. Leaks to the media in big markets.

Morale has its own decay and ceiling (20.3), and its in-game effect counts toward the team-level cap.

### 13.15 Injuries, illness and medical
**Injury types:**
- upper body, lower body, concussion, illness, and general wear
- realistic **recovery-time distributions** per type: day-to-day, 1–2 weeks, 4–6 weeks, season-ending

**Injury rate depends on:**
- physicality and big hits (13.11)
- fatigue and season load
- ice quality
- injury proneness
- age
- medical and conditioning staff ratings

**Re-injury risk:**
- higher shortly after returning
- higher if the player returns before full recovery: the owner or coach can choose to rush him back

**Concussions:**
- Repeat concussions mean longer recovery and higher future risk.
- A protocol keeps the player out for a minimum time.

**Playing hurt:**
- Day-to-day injuries let a player play at reduced ratings.
- In the playoffs, players play through injuries more often (16.3).

**Medical staff:**
- Their **diagnosis accuracy** determines how reliable the estimated return date is.
- Weak staff can misjudge recoveries, causing surprise setbacks.

**Illness:** a flu can spread through the locker room, with a chance of several players being out or weakened at the same time.

**Calibration:** total man-games lost per team per season, and the injury-type mix, should match NHL averages (20.1).

## 14. Experience and leadership ratings

### Experience (visible, mostly auto-calculated)
- Two components:
  - **Regular-season experience**, from NHL games played
  - **Playoff experience**, from playoff games played, with much higher weight per game
- Grows on a log curve, so the first 200 games matter far more than games 800–1000. Suggested anchors:
  - rookie ≈ 25–35
  - ~300 GP ≈ 60
  - 800+ GP veteran ≈ 85+
- Optional bonus for international or championship experience (Stanley Cup, Olympics, World Championships).
- Effects:
  - **Lower variance:** narrows daily-form noise, so experienced players have fewer disaster nights.
  - **Pressure dampening:** reduces the negative side of occasion modifiers in high-stakes moments (late close games, OT, playoffs, elimination games).
  - **Fewer high-pressure mistakes:** small reduction in penalties and turnovers when the game is close late.
  - **Rookie nerves:** low-experience players get a small negative modifier in their first games and in their first playoff games. It fades as experience grows.
- Effect size grows with stakes: small in a regular-season game in November, larger in OT and in playoff elimination games.
- **No double counting:** experience, mental toughness, the big-game trait and clutch together form the **pressure budget** (20.2). Their combined effect in any situation can't exceed that budget's ceiling. Experience reduces form noise **instead of**, not in addition to, the consistency attribute's maximum effect: the player gets the larger of the two.

### Leadership (visible, slow-changing)
- Partly innate (personality), partly earned. Grows slowly with experience and age, typically peaking in the late 20s to early 30s.
- Captain ("C") and alternate captains ("A") are chosen by the coach AI or set manually. Being named captain does not raise the rating by itself.
- **Team leadership score** = a weighted blend of the top 3–5 leaders on the roster, with diminishing returns.
- Effects:
  - Dampens team-wide emotional swings (losing streaks, blowouts, in-game momentum against).
  - Faster confidence recovery for teammates after bad games.
  - **Mentoring:** young players on a team with strong veteran leadership develop slightly faster (season mode).
  - Locker-room stability: smaller performance hit after trades, coaching changes or long losing streaks.
- Absence matters: an injured captain gives a smaller leadership effect until he returns.

## 15. Determining baseline ratings

Three supported sources. Each player record stores which one was used.

1. **From real stats (default for real players):**
   - Convert per-60-minute metrics to league percentiles, then map percentiles onto the 1–99 scale, with the 50th percentile at 50.
   - Example mappings:
     - Shooting: goals above expected per shot
     - Offensive sense: individual xG/60
     - Playmaking: primary assists/60
     - Defense: on-ice xG against/60, relative to teammates
     - Faceoffs: faceoff %
     - Discipline: penalties taken/60
     - Goalies: goals saved above expected, split by shot danger level
   - Blend multiple seasons with recent seasons weighted more (e.g. 50/30/20 across the last 3 seasons).
   - **Regress small samples toward the mean.** A player with 15 games of data should sit much closer to 50 than his raw numbers suggest.
   - Adjust for context: usage (offensive vs defensive zone starts), quality of teammates and quality of competition, where data allows.
   - Ratings with no good stat (stamina, leadership, personality traits) come from age and role defaults, or are set manually.
2. **Generated (fictional leagues):**
   - Draw ratings from distributions by roster slot (section 12 table), with position-appropriate attribute profiles.
   - Assign a random archetype (sniper, two-way, etc.) that shapes the attribute profile.
3. **Manual:** set or override any rating in the admin console.

**Overall rating:** a position-specific weighted average of the attributes. It is used for display and for the coach AI's lineup decisions. The engine always uses the individual attributes.

## 16. Rating lifecycle: in-game, season, playoffs

Ratings live in three layers that change at different speeds:

| Layer | Changes | Examples |
|---|---|---|
| **Baseline** | Slowly (weekly or monthly) | Development, aging, long-term injury effects |
| **Game-to-game** | Before each game | Form, confidence, streaks, occasion, schedule fatigue |
| **In-game** | During the game | Shift fatigue, ice-time load, in-game momentum, in-game injuries |

### 16.1 During a game
- Baseline and pre-game modifiers are fixed at puck drop.
- Effective ratings change continuously during the game:
  - shift and cumulative fatigue
  - in-game momentum swings
  - injuries mid-game (the player leaves, or plays on at reduced ratings)
- The coach AI reacts by shortening the bench, juggling lines after a bad period, or pulling a struggling goalie (goals against vs expected).
- After the final buzzer, update each player's confidence, experience (games played) and season fatigue load.

### 16.2 Across the regular season
- **Baseline recalculation:** on a schedule, e.g. every 10 games or monthly.
- **Development:** each player has a hidden potential rating and a growth rate. Young players (≈18–24) move toward their potential over the season, faster with:
  - ice time
  - good coaching
  - mentoring (team leadership)
  
  **Anti-loop rule:** each development boost has diminishing returns, and the player's potential is a hard ceiling. Real variance is kept: some players outgrow their projected potential (late bloomers), and some stall.
- **Aging:** veterans past their peak age decline gradually. Skating declines first, hockey sense and faceoffs last. Goalies decline later.
- **Small performance drift:** a sustained breakout or slump over 20+ games can move the baseline slightly, capped per recalculation. Most short streaks stay in the form layer and never touch the baseline.
- **Injuries:** temporary reductions while out, a short "rust" period on return, and occasionally a lingering small baseline drop for serious injuries.
- **Season fatigue:** accumulates with games and travel, recovers with rest days, including the all-star and holiday breaks. Reduced by strength and conditioning staff.
- **Line chemistry:** linemates who play together build a chemistry bonus over games, small and capped. It resets partly when lines are changed or players are traded.
- **Experience** grows after every game played.
- **Roster events:** trades, call-ups and waivers move a player's ratings with him. He takes a short adjustment period on the new team, and chemistry resets.

### 16.3 Playoffs
**Format:** defined in 2a (qualification, bracket, best-of-7, home ice, 5v5 overtime). Overtime edge cases: 9.7. This section covers only how playoff games *play* differently.

**Playoff environment modifiers:**
- Referees call fewer penalties, so power-play opportunities drop by a configurable amount.
- Physicality and hit rates increase.
- Tighter checking: slightly lower shot quality.

**Usage:**
- Coaches shorten the bench: stars and the top pair play more.
- The starting goalie plays nearly every game unless he is struggling.

**Experience and big-game weighting:**
- Playoff experience, mental toughness and the big-game trait carry noticeably more weight than in the regular season.
- Rookie nerves apply in a player's first playoff games.

**Series dynamics:**
- Series state modifiers:
  - leading or trailing in the series
  - elimination games (desperation for the team facing elimination, pressure on the team closing out)
  - Game 7 at home
- Momentum carries between games in a series, as a decaying modifier.
- Opponent familiarity: rivalry intensity builds over a long series.

**Accumulating wear:**
- Fatigue and injuries carry across games and rounds.
- Players can play through injuries at reduced ratings instead of sitting out.
- Rest days between series restore partially.

**Baselines** are frozen during the playoffs. Experience grows with each playoff game, and that playoff experience carries into future seasons.

### 16.4 Offseason (season mode)
- Apply annual development and aging, using a larger step than the in-season updates.
- Recompute stat-based baselines with the new season's data, if using real stats.
- Reset season fatigue. Roll experience forward.
- Re-evaluate leadership and captaincy.
- **League-wide rating normalization:** after development and aging, re-center each attribute so the league-wide distribution (mean and spread for NHL regulars) matches the definition "50 = league-average regular". This keeps ratings from drifting up or down over many seasons. Relative differences between players are preserved.
- **Retirements:** decided by age, declining ratings, injuries, contract offers, and personality (some players chase a Cup, some leave for Europe).
  - Retired players can become coaches, scouts or staff (17.14).
  - Players who aren't signed by NHL teams can sign in Europe or the minors, and occasionally return.
- **Player pool balance:** the number of new pro players entering each year should roughly match the number leaving. Sources of new players:
  - draft class signings
  - European and college free agents
  - minor-league call-ups
  
  Track the totals each season (20.1).
- **Staff offseason:** contract expiries, hiring and firing, and promotions (17.14).

## 17. Front office, trades and minor league

### 17.1 General manager (team staff)
Each team has a GM, with ratings on the same 1–99 scale.

**GM ratings:**
- **Experience:** years as GM, trades made, playoff and championship history.
- **Talent evaluation:** how accurately the GM judges player value. Low ratings mean noisy judgment: overpaying, or selling players too cheaply.
- **Negotiation:** how good a deal the GM gets in trades and contract talks.
- **Cap management:** avoids bad long contracts and keeps cap flexibility.
- **Strategy and patience:** contend vs rebuild mindset; how long he sticks with a plan.
- **Risk tolerance:** willingness to make blockbuster trades or bet on prospects.
- **Relationships:** standing with other GMs and with agents, which affects how often trades and signings happen.
- **Development focus:** how much he prioritizes the prospect pipeline and minor-league investment.

**GM surprise factor:** the front-office version of 13.8. GMs with high experience or high evaluation ratings occasionally find a "steal": a trade, waiver claim or signing worth more than it should be. Low ratings risk a "panic move", such as overpaying at the deadline during a slump.

**Team effects of GM moves:**
- A "vote of confidence" boost when a contending team buys at the deadline.
- A morale dip when the team sells while still in the race.
- Coach job security tied to results. A **coaching change** gives a short "new coach bounce" spike, then it settles. Firing logic, owner patience and the staff market: 17.14.
- GM moves are limited by the same market sanity rules as everyone else (17.13). A panic move is a bad decision, not an impossible one.

### 17.2 Contracts and salary cap
Needed for realistic trade logic.

All contract and cap rules are defined once, in **17.7**. Roster limits: section 12 (roster legality). Trade deadline: 2a calendar.

What the trade logic uses from them:
- cap hit and years remaining
- trade protection
- free-agency status
- waiver exemption
- retained-salary limits (max 50%; at most 3 retained contracts per team; the same contract can be retained again only after 75 regular-season days)

### 17.3 Trade logic (AI GMs)

**Valuing a player.** The AI combines:
- current ratings
- age curve and potential
- contract value relative to cap hit
- position scarcity
- the team's needs

The GM's evaluation noise is applied on top of this value.

**Team mode:**
- Contender: buys veterans and rentals (players in the last year of their contract).
- Bubble team: makes small moves.
- Rebuilder: sells veterans for prospects and draft picks.

The mode updates from the standings and the GM's patience rating.

**Trade timing:**
- A trade search runs periodically through the season.
- It peaks before the trade deadline.
- There is a quieter offseason window.

**Acceptance rules:**
- Both sides must see the trade as acceptable for them.
- It must be cap-legal and roster-legal.
- Trade-protection clauses must be respected; the player can waive them with a probability.
- Relationship ratings affect how often trades happen.
- Anti-exploit rules (consolidation penalty, roster-spot value, AI margin vs humans, cooldowns): **17.13**.

**Draft picks and prospects** are tradeable assets, valued by the expected value of the pick slot (17.11). The projected slot is based on the current standings, with uncertainty.

**Realistic volume:** the number of trades per season, the deadline spike and the mix of trade types should match recent NHL seasons (20.1).

### 17.4 Effects of trades on players
- **Adjustment period:** a few games of a small rating penalty on the new team, while the player learns the new system and teammates. Shorter for experienced players.
- **Chemistry reset:** linemate chemistry starts from zero with the new team.
- **Emotional response:** depends on personality.
  - Mental toughness: low means unsettled (form and confidence dip); high means motivated.
  - Being traded from a contender to a rebuilding team: motivation dip.
  - Being traded to a contender: motivation boost.
- **Revenge game:** the first game against his former team triggers 13.5.
- **Locker-room impact on the team that traded him:**
  - Losing a leader or a popular veteran lowers team leadership and gives a short morale dip.
  - Adding a respected veteran raises leadership.
- **Trade-rumor distraction** before the deadline (13.12), for players the AI marks as likely to be traded.

### 17.5 Minor league (AHL affiliate)

**Structure:**
- Each NHL team has an affiliate with its own roster of ~25–30 players.
- The affiliate has its own coach, staff and arena.
- An optional third tier (an ECHL-level league) can be added.

**Simulation fidelity:** configurable.
- Full: the same engine as the NHL, with lower ratings.
- Light: a fast results-only sim with stat lines, to save time.

**Player development:**
- Young players develop faster with **real ice time in the minors** than sitting as healthy scratches in the NHL.
- Development speed is scaled by the affiliate coach's ratings and the GM's development focus.

**Waivers:**
- Players who are no longer waiver-exempt (based on age and pro games played) must clear waivers to be sent down.
- Other teams' GMs may claim them, based on talent evaluation and team needs. The claiming team gets the player.

**Roster moves:**
- Call-ups for injuries, suspensions, slumps or rewarding a hot prospect.
- Emergency recalls, e.g. a goalie when the backup gets hurt.
- Injured players can play a short **conditioning stint** in the minors before returning.
- Healthy veterans can be sent down to clear cap space or a roster spot.

### 17.6 Effects of call-ups and send-downs

**Called-up player:**
- A short **excitement boost** to effort and energy, combined with rookie nerves (based on experience).
- An adjustment period to NHL pace.
- Earning a spot raises confidence; being sent back down quickly lowers it.

**Team receiving a call-up:**
- A small "fresh legs" energy bump if called up during a heavy schedule.
- A spark if the call-up scores early.

**Sent-down player:**
- A confidence hit.
- Then his personality decides the reaction:
  - high mental toughness: he responds with improved effort and a faster return to the NHL
  - low mental toughness: he sulks, and form dips in the minors

**Veteran sent down or waived:**
- Leadership loss for the NHL team.
- A leadership gain for the affiliate, which helps prospect development.

**Repeated shuttling** between the leagues: a small cumulative confidence and morale penalty.

### 17.7 Contract structure (NHL-style)
Default values follow the NHL rules in force from Sep 16, 2026 (the new collective bargaining agreement). Every number is a config value: verify it before release and update it each season.

**League financials (2026–27):**
- Salary cap ceiling $104M, floor $76.9M.
- Maximum player cap hit $20.8M (20% of the cap).
- Projected ceiling 2027–28: $113.5M.
- Minimum NHL salary: $850k in 2026–27, rising to $900k, then $950k, then $1M by 2029–30.
- Playoff salary cap applies from 2026–27.

**Contract term limits:**
- Max 7 years when re-signing with the current team. Applies only if the player was on that team at the last trade deadline.
- Max 6 years when signing with a new team.

**Salary structure rules:**
- **Year-to-year variance:** the change between two consecutive years can't exceed 20% of Year 1's salary.
- **Lowest year:** no year can pay less than 71% of the highest year.
- **Signing bonuses:** total signing bonuses can't exceed 60% of the contract's total value.
- **No deferred salary.**

**Contract types:**
- **One-way:** pays the NHL salary whether the player is in the NHL or the minors.
- **Two-way:** separate NHL salary and minor-league salary. Optionally, a guaranteed minimum regardless of league.
- **Entry-level contract (ELC):** for drafted or first-time players.
  - Length by signing age: roughly 3 years at 18–21, 2 years at 22–23, 1 year at 24. Configurable.
  - 2026 ELC maximums: NHL salary $1,025,000; minor-league salary $87,500.
  - Performance bonuses allowed.
  - **ELC slide:** players who sign at 18–19 and play fewer than 10 NHL games in a season don't use up a contract year.
- **35+ contracts:** special cap treatment. Configurable.
- **Minor-league-only contracts:** AHL-only and ECHL-only deals. The player isn't under NHL team control, so he can be signed to an NHL contract.
  - AHL minimum salary: approx. $62,500 in 2026–27. Verify.
  - Lower tiers have weekly salary caps.
  - AHL veteran rule: at most 6 veterans in a game-day lineup. A veteran is a skater aged 25+ with 260+ pro games.

**Contract components:**
- Base salary per year
- Signing bonus per year
- Performance bonuses (ELC and 35+ only)
- Trade protection: no-trade clause (NTC), modified NTC with a list of teams, or no-movement clause (NMC)

**Cap mechanics:**
- **Cap hit** = average annual value (AAV) over the contract.
- **Burying a contract in the minors:** a player sent down removes cap hit only up to the league minimum + $375k ($1,225,000 in 2026–27).
- **Salary retention in trades:** up to 50%; the same contract can only be retained once every 75 regular-season days.
- **Buyouts:** 2/3 of the remaining salary (1/3 for players under 26), spread over twice the remaining term. Leaves "dead cap" on the books.
- **Long-term injured reserve (LTIR):** relief beyond the cap during the regular season, limited by the playoff cap.

### 17.8 Salary valuation and recommendation
Each player gets a **fair-value estimate** that labels his current contract as **overpaid / fairly paid / underpaid**.

**Fair-value model:**
1. **Projected contribution:** a wins-above-replacement-style value from the player's ratings, role and projected ice time. Projected over the contract term with the age curve (section 16.2), so long deals for 31-year-olds lose value in later years.
2. **Market price per unit of contribution:** derived from league-wide contracts, expressed as **% of the salary cap**, so values scale as the cap grows.
3. **Status adjustments:**
   - RFAs and ELC players are discounted, because the team controls their rights.
   - UFAs are priced at full market value.
   - Arbitration-eligible players fall in between.
4. **Market adjustments:**
   - Position scarcity (e.g. top-pair right-shot defensemen)
   - Supply in the current free-agent class
   - Cap space available around the league
5. **Output:**
   - Fair AAV range for the current contract year, and for a hypothetical new contract at 1–7 years of term
   - Surplus value (fair value minus cap hit), per year and over the full contract
   - Label: within ±15% of fair = fairly paid; above = overpaid; below = underpaid. Thresholds are configurable.

**Who sees what:**
- The recommendation the owner sees comes from **his own GM and analytics staff**. Its accuracy depends on their talent-evaluation and analytics ratings: better staff give better advice, weaker staff give noisier estimates.
- An "omniscient" true value is available in the admin console for debugging.

**Effects on players:**
- Clearly underpaid veterans can get unhappy over time: a small morale dip, possibly a holdout, or a request for an extension or trade.
- Overpaid players get media and crowd pressure in big markets when they underperform (13.12).
- A slight "after the payday" complacency effect for some personalities. Small and toggleable.

### 17.9 Free agency and player rights (NHL and minors)
Exact thresholds are config values. Verify them against the current agreement.

- **Restricted free agent (RFA):** a player whose contract expires before he qualifies as a UFA.
  - The team keeps his rights by making a **qualifying offer** before the deadline. Without it, he becomes a UFA.
  - **Offer sheets:** another team can offer him a contract. His current team can match it, or let him go and receive draft-pick compensation, scaled by the AAV.
  - **Salary arbitration:** for eligible RFAs, based on years of pro experience. An arbitrator awards a salary based on comparable players.
  - **Holdout:** an unsigned RFA can miss training camp and games. Holdout players get rust and a morale cost when they return.
- **Unrestricted free agent (UFA):** at age 27 or after 7 accrued seasons, or under the "Group VI" rule (older players with few NHL games). Free to sign anywhere.
- **Free agency calendar:**
  - Extensions allowed from July 1 of the final contract year.
  - Interview window shortly before free agency.
  - **Free agency opens July 1.**
  - Arbitration hearings in late July / August.
  - Training camp in September.
- **Waiver exemption:** depends on signing age and pro games played (17.5).
- **Minor leagues:**
  - Minor-league-only contracts expire into minor-league free agency.
  - Players on minor-league-only contracts can be signed to NHL contracts.
  - **Professional tryouts (PTO):** short trial contracts.
  - Released NHL players and unsigned draftees enter this pool.
- **Draft rights:** expire after a set number of years if the team doesn't sign the player. Under the new agreement, NHL teams hold the rights to junior-league (CHL) draftees for 4 years if drafted at 18 and 3 years if drafted at 19.

### 17.10 Owner mode: the human-managed team
One or more teams can be **controlled by a human** (the owner). Every other team is run by the AI GM.

**Owner role:**
- Sets the internal budget, which can be below the cap.
- Makes or approves all contract decisions, trades and roster moves.
- Can delegate any area to his AI GM: day-to-day roster moves, minor-league contracts, or everything.
- Delegated decisions are logged and can be reviewed.

**Negotiation runs on a realistic in-game calendar, not instantly:**
- Each offer and counteroffer takes **simulated days**, with the agent responding after a delay.
- **Timing windows:**
  - **Early and mid-season:** players are mostly reluctant to negotiate. Many say they want to "focus on hockey." Talks are possible but slow, and stars often decline until later.
  - **Trade deadline and late season:** activity rises. Expiring players decide whether to extend or test the market. A failed negotiation is a trade trigger.
  - **End of season and playoffs:** the main window. Most extensions happen here. Players still in the playoffs negotiate after elimination.
  - **Offseason:** qualifying-offer deadline, the interview window, July 1 free agency, arbitration, offer sheets. Daily time steps.
- Deadlines create pressure. Unsigned UFAs go to market on July 1; RFAs without a qualifying offer become UFAs.

**Negotiation mechanics:**
- The owner proposes a full contract:
  - term
  - AAV and year-by-year salary, which must pass the 17.7 structure rules
  - signing bonus split
  - one-way or two-way
  - trade protection (NTC / NMC)
- **Each player has an agent** with ratings: toughness, patience, and leverage awareness (whether the agent knows about rival offers and cap space).
- **Player priorities**, weighted per personality:
  - money
  - term and security
  - role and ice time
  - team competitiveness ("wants to win")
  - location, family and hometown
  - loyalty to the current team
  - trade protection
- Response options:
  - accept
  - counteroffer
  - pause talks for N days
  - "test the market" (UFA)
  - demand a trade
  - hold out (RFA)
- **Relationship meter** between player and team:
  - Lowered by lowball offers, long stalls and broken promises, e.g. promising a top-6 role and not delivering.
  - Raised by fair offers, playing time and team success.
  - Affects how willing he is to give a hometown discount.
- **Contract talks affect performance:**
  - Drawn-out or hostile talks cause a distraction and lower confidence (13.3).
  - A signed extension gives a short relief boost.
  - Contract-year motivation still applies (13.12).
- **Competition:**
  - In free agency, AI teams make competing offers.
  - The player decides by comparing all offers, weighted by his priorities.
  - The owner sees reported interest from other teams, with accuracy depending on GM relationships.
- **The salary recommendation (17.8)** shows during negotiations: fair range, surplus value of the proposed deal, cap impact over the coming years, and whether each year satisfies the structure rules.

**Other owner tasks:**
- Negotiating with minor-league players
- Signing draft picks to ELCs before their rights expire
- Professional tryouts
- Buyouts during the buyout window
- Placing players on waivers

### 17.11 Entry draft
Mirrors the NHL. Every number is a config value, to be verified each season.

**Format:**
- 7 rounds, one pick per team per round: 224 picks with 32 teams.
- Held in late June after the Stanley Cup Final (2026: June 26–27).
- **Draft picks are tradeable assets** for up to ~3 future drafts, including conditional picks (e.g. "top-10 protected" or "becomes a 1st if the team reaches the conference final").
- Forfeited picks for rule violations: admin option.

**Eligibility:**
- Players who are 18 by Sep 15 of the draft year, up to an age limit. For example, the 2026 draft covered births from Jan 1, 2006 to Sep 15, 2008.
- Previously drafted but unsigned players can re-enter under the age rules.

**Draft lottery (non-playoff teams only):**
- All 16 teams that miss the playoffs take part, or the teams that own their first-round picks.
- **Two drawings:** the first decides the #1 pick, the second decides #2.
- A team can move up **at most 10 spots** by winning a draw, so only the 11 worst teams can win #1.
- A team can win the lottery at most **twice in 5 years**.
- Odds are weighted by finish; the last-place team has the best odds (18.5% in 2026).
- Picks 3–16 follow inverse standings among the remaining non-playoff teams.
- Lottery date: early May, after the regular season.

**Order for picks 17–32 (playoff teams):**
- 17–28: teams eliminated in rounds 1–2.
  - First, the non-division winners, in inverse order of regular-season points.
  - Then the division winners that lost, also in inverse order of points.
- 29–30: conference final losers, in inverse order of points.
- 31: Stanley Cup Final loser.
- 32: Stanley Cup champion.

**Rounds 2–7:** inverse standings, with no lottery. Non-playoff teams first, then playoff teams ordered by the same playoff-elimination logic.

**Draft class generation:**
- Each season generate ~250–350 eligible prospects from:
  - junior leagues (e.g. CHL)
  - US college and junior (NCAA, USHL)
  - European pro and junior leagues
- Each prospect has:
  - **true current ratings**
  - **true potential**
  - a development speed
  - personality traits
  - an age and position
- Distribution: a few franchise-level talents in some years, deep or weak classes in others, and many low-potential late-round picks. Class strength varies year to year.
- Calibration: on average, roughly half of first-round picks should become NHL regulars, and very few picks after round 3.

**Scouting and visibility:**
- Teams never see true ratings. They see **scouting reports**: estimated ratings and potential with uncertainty ranges.
- Report accuracy depends on scouting staff ratings and how much the team scouts each league.
- A league-wide **Central Scouting-style ranking**, published and updated through the season, plus a combine before the draft.
- Prospects' stats in their current leagues are simulated in light mode, so reports change during the season.

**Draft day:**
- AI GMs pick by best-available vs team needs, weighted by the GM's strategy.
- Their board uses their own (noisy) scouting, so steals and busts happen naturally.
- Picks can be traded on draft day.
- The **owner** drafts in real time, with a pick timer, a draft board, scouting reports and trade-up/trade-down offers. He can delegate the draft to his AI GM.

**After the draft:**
- **Signing rights:** the team holds a drafted player's rights for a limited time, then loses them and the player re-enters the draft or becomes a free agent.
  - Junior-league (CHL) players: 4 years if drafted at 18, 3 years if drafted at 19 (new agreement).
  - Other leagues: own configurable rule.
- **Development path per prospect:** back to junior, college or Europe, or sign an ELC and go to the AHL or NHL.
  - Junior-league players under 20 generally can't play in the AHL. The new agreement adds an AHL option for one 19-year-old CHL player per team.
- Signed prospects go on the contract list. ELC slide rules apply (17.7).

### 17.12 Waivers
Follows NHL waiver rules. Values are config, to be verified.

**When waivers are needed:** to send a waiver-eligible player to the minors, a team must place him on waivers. Waivers are also needed before a buyout or contract termination.

**Waiver period:** ~24 hours (in the NHL, noon ET to noon ET the next day). Claims are submitted during the period.

**Claim priority:** the team with the lowest points % gets priority. Early in the season (until about Nov 1, per config) the previous season's standings are used.

**After a claim:**
- The claiming team takes the player and his full contract, so it needs cap space and a roster spot.
- If nobody claims him, the player "clears" and can be sent down.
- A player who cleared stays exempt from re-waivers until he plays 10 NHL games or spends 30 cumulative days on the NHL roster.

**Waiver exemption:** depends on the player's age when he signed his first NHL contract, and ends at whichever limit comes first. Values are approximate; verify against the current agreement.

| Age at first signing | Skaters: years / NHL games | Goalies: years / NHL games |
|---|---|---|
| 18 | 5 / 160 | 6 / 80 |
| 19 | 4 / 160 | 5 / 80 |
| 20 | 3 / 160 | 4 / 80 |
| 21 | 3 / 80 | 4 / 60 |
| 22 | 2 / 80 | 3 / 60 |
| 23 | 2 / 60 | 3 / 60 |
| 24 | 2 / 60 | 2 / 60 |
| 25+ | 1 / — | 1 / — |

Adjustments:
- An 18–19-year-old skater who plays 11+ NHL games in his first season has his exemption cut to 3 seasons.
- Goalies in the same situation are cut to 4 seasons.

**AI behavior:** AI GMs evaluate every player on waivers (value vs cap hit vs needs vs roster space) and claim with a probability. Better evaluators claim the true bargains more often.

**Owner mode:**
- The owner sees a daily waiver wire with scouting estimates and can submit claims.
- Before waiving his own player, he gets a **warning with an estimated claim probability**.

**Effects on players:**
- Being waived: confidence hit.
- Being claimed: an opportunity boost (more ice time on the new team), plus the trade adjustment effects (17.4).
- Clearing and being sent down: the send-down effects (17.6).

### 17.13 Integrity and anti-exploit rules
Human owners will look for ways to exploit the AI and the rules. These safeguards keep the league realistic.

**Trades:**
- **Consolidation penalty:** two or three mid-level players aren't worth one star. Value grows faster than linearly with quality, and every incoming player costs a roster spot.
- **AI safety margin:** the AI accepts a trade with a human only if it gains value by a configurable margin, and only after checking it with its own (noisy) evaluation **plus** a sanity check against market value. No AI accepts a trade that's clearly lopsided by market value, whatever its own evaluation noise says.
- **Cooldowns and memory:**
  - repeated offers to the same team for the same player get worse responses
  - AI GMs remember past lopsided deals with an owner, which lowers their relationship rating
- **Flip restrictions:** a player can't be re-traded within a configurable number of days after signing as a free agent or being acquired, unless his agreement waives it. Follows NHL rules, where they exist.
- **Market sanity:** the same sanity check applies to AI–AI trades, so AI GMs don't fleece each other into unrealistic rosters.

**Contracts and the cap:**
- Every transaction runs a full legality check (17.7 rules and section 12 roster legality).
- Known exploits get a test: front-loading, cap circumvention through extensions, burying players in the minors, abusing long-term injured reserve, circumventing salary retention.

**Re-simulation:**
- Each game's seed is derived from a **hidden league seed**, the game id and a commitment made before the game. Re-simulating a game gives the same result.
- Owners can't reload a game.
- Only the super admin can change seeds or re-simulate, and every such action is audit-logged.

**Tanking:**
- The draft lottery rules already limit it: a team can move up at most 10 spots, and win at most twice in 5 years.
- Deliberate losing also costs:
  - attendance and fan loyalty
  - morale of veterans
  - the team's reputation with free agents
  - owner patience, for the AI GM's job security

**Sim pace in multi-owner leagues:** no owner can advance the calendar alone. The super admin sets the rules (18).

### 17.14 Staff market, relationships, team culture and finances

**Staff contracts and market:**
- Coaches, the GM, scouts, trainers and analytics staff have contracts with salary and term.
- **Staff salaries don't count against the player salary cap.** They count against the owner's operating budget.
- Fired or unsigned staff enter a **staff market** and can be hired by other teams.
- Staff can **age, improve with experience, retire**, and be promoted, e.g. a minor-league coach becoming an NHL assistant, or an assistant becoming head coach.
- Retired players can become staff, starting with lower experience.

**Owner patience and firings:**
- The owner (AI owner for AI teams) has a **patience** rating.
- Firing decisions compare results to expectations: preseason projection, spending, the team's stage of rebuild.
- Realistic firing rates: a handful of head coaches per season league-wide, GMs less often (20.1).
- Firing mid-season gives a "new coach bounce" (17.1) and a system-change adjustment period for players.

**Relationships:**
- **Coach–player relationships:** stars can clash with demanding coaches. If several key players lose faith in the coach, the coach "**loses the room**": morale drops, effort drops, and owner patience runs out faster. Personalities and the leadership core decide how likely this is.
- **GM–coach relationship:** disagreements over deployment and roster lower job security for both.

**System fit:**
- Each coach system has requirements, e.g. a heavy forecheck needs speed and physicality, and a possession system needs puck skills.
- A mismatch with the roster costs a small performance penalty until the roster or the system changes.
- The coach's adaptability rating reduces the penalty.

**Team culture and chemistry (team-wide, beyond line chemistry):**
- Built slowly by stability, winning, leadership and a good coach-team relationship.
- Hurt by many roster changes, losing, conflicts and trade requests.
- Effects: small effects on morale, consistency and resilience after losses. Counts toward the team-level cap (13).
- Changes slowly, with its own ceiling (20.3).

**Facilities and finances** (optional; the super admin can turn this off):
- **Revenue:** ticket sales (from attendance, 13.10), playoff home games, merchandise tied to stars and winning, media.
- **Costs:** player payroll, staff salaries, facilities.
- **Operating budget** limits payroll and staff spending; set by the owner (18.2), or by the AI owner for AI teams.
- **Facilities:** practice facility and training staff quality. Effects:
  - recovery and development
  - **free-agent appeal**, together with location, taxes, winning and the coach

## 18. Consoles and access levels

Settings are split across two consoles with different permissions:

- **Super admin console (18.1):** game-wide and league-wide settings. These are the rules of the simulation and the league, and they apply to every team.
- **Team console (18.2):** each owner's settings for his own team. These are the decisions a real NHL owner, GM or coach would make.

Wherever this document says "admin console", it means the **super admin console**, unless the setting is listed under the team console.

### Roles and permissions
- **Super admin:**
  - full access to every setting, every team and the "omniscient" true values
  - can lock any team-console setting league-wide (e.g. "AI teams only", "no ticket-price changes") or set its allowed range
  - can override any team's decision and take over or reassign a team
- **Team owner (human):**
  - full control of his own team console
  - no access to the super admin console
  - sees other teams only as a real owner would: public rosters, stats, standings, contracts, and his own staff's scouting estimates
- **AI-controlled teams:** run by the AI GM using the same team-console options, so AI and human teams play by identical rules.
- **Optional delegated roles per team:** e.g. an assistant GM, or read-only viewers.
- **Multi-owner leagues:** several human owners in one league, each with his own team console. The super admin controls sim pace for everyone, e.g. advancing day by day once all owners are ready, or on a timer.
- **Audit log:** every change in either console is logged with who, what and when. Super admin overrides are visible to the affected owner.

### 18.1 Super admin console (game and league-wide settings)

**Every game-wide or league-wide tunable setting in this document must be adjustable from the super admin console**, not hard-coded. That includes:

- Game structure: period length, OT length and format (3v3 on/off), shootout rounds, points system
- All event base rates (shots, attempts, penalties, faceoffs, flavor events)
- Shooting % and save % baselines, plus the xG model weights (distance, angle, rebound, rush)
- Power-play modifiers (shot-rate multiplier, quality boost) and the penalty type mix (minor, double minor, major)
- Score-effect strength and how it scales by period
- Goalie-pull logic: trigger time window, goal deficit thresholds, 6v5 rate multipliers
- Home-ice advantage multiplier
- Shift length, line ice-time distribution, fatigue penalty
- OT 3v3 scoring multiplier and shootout conversion %
- Team, goalie and player ratings: every baseline attribute, role tag and hidden trait for every player
- Coaching and staff ratings and their effect sizes
- Rating-to-multiplier conversion strength (how much 10 rating points matter)
- Human factors (section 13): **every dynamic factor has its own on/off toggle and its own weight (effect-size) slider** controlling how much it affects the simulation, e.g. 0 = no effect, 1 = default, 2 = double. This covers fatigue (shift, game, schedule, travel), daily form, confidence, team streaks, each occasion type, in-game momentum and each personality trait's influence. Also:
  - master intensity slider that scales all of them at once
  - a live preview showing a sample player's baseline vs effective rating as the sliders move
  - an impact report after batch runs showing how much each factor shifted results, e.g. win % swing and goals per game
  - modifier cap
  - form persistence (ρ) and noise
  - confidence decay rate and previous-game weight
  - streak effect sizes and desperation threshold
  - occasion modifiers (rivalry, revenge, trap game, crowd, playoff race)
  - fatigue curves (shift, game, schedule, travel)
  - in-game momentum size and decay time
- Manual overrides: set a specific player's form or confidence for the next game (e.g. "playing through injury")
- Surprise factor (section 13.8):
  - on/off toggle and overall weight slider
  - trigger probability for each trigger type
  - boost size and decay time
  - target scope (player, line, unit or team)
  - the weights of each coach and staff member in spark potential
  - opponent adaptation strength
  - cooldown and diminishing returns
  - separate cap for surprise effects
  - backfire toggle and its probability
  - playoff adjustments
  - staff experience and innovation ratings, editable per staff member
- Officials (section 13.9):
  - crew assignment (random or manual) and per-official ratings
  - call-rate baseline
  - missed-call and phantom-call rates
  - late-game and OT whistle-swallowing
  - evening-up strength and home bias size
  - game-control escalation thresholds
  - controversial-call probability
  - challenge and review rules and outcomes
- Crowd and venue (section 13.10):
  - arena capacity and fan loyalty per team
  - attendance formula weights
  - traveling-fan share
  - crowd energy gain and loss per event type
  - mood transitions
  - crowd effect on home players, visitors and referees
  - last-change matchup strength
  - faceoff stick-down edge
  - ice quality per arena and its in-period decay
  - board liveliness, altitude, outdoor-game settings
- Calls and consequences (section 13.11):
  - true-infraction rates by type
  - call probabilities for each infraction type
  - penalty severity mix
  - consequence sizes for every outcome in the call table (momentum, frustration, discipline, crowd)
  - retaliation probabilities and escalation chain
  - fight rates and momentum effects
  - instigator rules
  - game-misconduct handling
  - supplemental discipline: review triggers, suspension-length formula, repeat-offender weighting
  - bad-blood carry-over strength and duration
- Other factors (section 13.12): a toggle and weight slider for every listed factor, including scorekeeper bias (stats only)
- League structure (section 2a):
  - teams, conferences, divisions
  - games per season and the schedule matrix
  - calendar and breaks
  - points and tiebreakers
  - playoff format
  - presets: NHL 2026–27 (default), NHL 82-game, custom
- Front office (section 17):
  - GM ratings per team and the GM surprise factor (steal and panic-move probabilities)
  - coaching-change rules and new-coach bounce
  - contracts and cap settings: cap ceiling and floor, roster and contract limits, retained-salary rules, trade deadline date, trade-protection waive probability
  - trade AI: trade frequency, deadline peak, player value formula weights, evaluation noise per GM rating, team-mode thresholds, acceptance rules
  - trade effects on players: adjustment period length, emotional response sizes, locker-room impact, rumor distraction
  - minor league: on/off, number of tiers, sim fidelity (full or light), affiliate rosters and staff ratings, development multiplier for minor-league ice time, waiver rules and claim logic
  - call-up and send-down effects: excitement, nerves, confidence changes, shuttling penalty, veteran leadership transfer
  - manual trades and roster moves (force a trade, call-up or demotion)
- Contracts and finances (17.7):
  - per-season financials: cap ceiling and floor, max salary %, minimum salary schedule, cap growth projection, playoff cap
  - term limits
  - salary structure rules: variance %, lowest-year %, signing bonus %
  - ELC rules: length by age, maximum salaries, slide threshold
  - two-way and minor-league salary minimums
  - cap-burying amount, retention rules, buyout formula, LTIR
  - minor-league veteran rule
  - edit any player's contract directly
- Salary valuation (17.8):
  - contribution model weights and market rate per contribution unit
  - RFA, ELC and arbitration discounts
  - position scarcity and supply adjustments
  - overpaid/underpaid thresholds
  - staff evaluation noise
  - morale effects of being underpaid or overpaid, and payday complacency
  - toggle to show true values
- Free agency (17.9):
  - UFA/RFA eligibility thresholds
  - qualifying offer rules
  - offer sheet compensation tiers
  - arbitration eligibility and award model
  - holdout behavior
  - free agency calendar dates
  - draft-rights expiry
  - minor-league free agency and tryout rules
- Entry draft (17.11):
  - rounds, draft date, future-pick trading horizon, conditional picks
  - eligibility ages
  - lottery: participants, number of draws, odds table, max spots moved up, win limit per period
  - pick order rules for playoff teams
  - draft class: size, league mix, talent distribution, class-strength variance, bust/steal calibration
  - scouting accuracy by staff rating
  - Central Scouting-style rankings
  - AI draft strategy weights
  - pick timer
  - signing-rights durations by league
  - development path rules, including the junior-to-AHL age rule and its exception
- Waivers (17.12):
  - waiver period length
  - priority rule and switchover date
  - exemption table (skaters and goalies) and first-season adjustments
  - re-waiver exemption limits (games, days)
  - AI claim aggressiveness
  - player effects of being waived, claimed or cleared
- Owner mode (17.10):
  - which teams are human-controlled
  - delegation settings
  - negotiation speed (days per response) and the reluctance curve across the season
  - agent ratings and player priority weights
  - relationship meter effects
  - contract-talk distraction size
  - AI competing-offer aggressiveness
- Experience and leadership (section 14): experience curve anchors, playoff-experience weight, rookie-nerves size and fade rate, the stakes multiplier on experience effects, leadership growth rate, team leadership blend, mentoring boost, captain assignment (auto or manual). Each effect has its own weight slider.
- Baseline sources (section 15): choose real stats, generated or manual per player or league. Also set:
  - stat-to-rating mappings
  - season weighting (e.g. 50/30/20)
  - small-sample regression strength
  - generated-roster distributions
  - overall-rating weights per position
- Lifecycle (section 16):
  - baseline recalculation frequency
  - development and aging curves by position
  - performance-drift cap
  - injury rust and lingering effects
  - season fatigue accumulation and recovery
  - line chemistry growth, cap and reset
  - trade adjustment period
- Playoffs:
  - format (teams, series length, home-ice pattern)
  - OT rules
  - penalty-call reduction, physicality increase, checking tightness
  - bench-shortening strength
  - weight of experience and big-game traits in the playoffs
  - series momentum, elimination-game and Game 7 modifiers
  - injury carry-over and play-through-injury rules
  - rest-day recovery
- Simulation settings: random seed, number of games for validation runs, tick size or event-driven mode
- Game engine (section 9):
  - possession and zone transition probabilities (breakout, entry types, cycle)
  - stoppage reason frequencies
  - TV timeout marks
  - rebound window and probability
  - shot type and location distributions
  - line-change timing and bad-change risk
  - delayed-penalty behavior
  - three-stars formula
  - commentary level and phrasing variety
  - output detail per game type: full play-by-play for watched games, results-only for mass sims
  - export formats
  - realism-check targets (9.6)
- Realism safeguards (section 20):
  - player and team modifier caps
  - effect budgets and their split among sources
  - decay rates and ceilings for every feedback-driven modifier
  - rating normalization on/off and target distribution
  - player pool balance targets
  - anti-exploit settings: AI trade margin, cooldowns, flip restrictions, hidden league seed
  - realism report thresholds that trigger warnings
- Morale, injuries, staff and team (13.14, 13.15, 17.14):
  - morale inputs and effects
  - injury type rates and recovery distributions, re-injury and concussion rules, illness spread
  - staff market, aging and promotions
  - owner patience and firing thresholds
  - loss-of-room and system-fit effects
  - team culture growth and decay
  - finances: revenue model and budget limits
- Rule edge cases (9.7): EBUG procedure, short-bench rules, OT penalty strength rules, shootout shooter rules

Console features:
- Grouped settings sections mirroring the sections of this spec, with sensible min/max ranges and a short description for each field
- Defaults loaded from `config.py` (or a JSON/YAML file), with a "reset to defaults" option
- Save and load named presets (e.g. "2025-26 league average", "high-scoring era")
- Run a single game or a batch validation from the console, and show the resulting aggregates next to the calibration targets
- Settings persist between sessions

Implementation note: keep all parameters in one config object that the engine reads at game start, so the console only edits config and never touches engine code.

### 18.2 Team console (owner settings)

Each owner controls his own team here. Every setting works within the limits set in the super admin console, which can lock any of these or restrict their range.

**Lineup and roster:**
- Forward lines, defense pairs and the starting/backup goalie, including a goalie rotation plan
- Healthy scratches
- Power-play and penalty-kill units, plus 3v3 OT and shootout order
- Captain and alternates
- Call-ups, send-downs, conditioning stints and waiver decisions (17.5, 17.12)
- Injured reserve and LTIR placement

**Tactics and coaching instructions** (combined with the coaches' own ratings, 12 and 13.8):
- System style: offensive vs defensive; forecheck type; how often defensemen join the rush
- Physicality and discipline instructions, e.g. "play it safe" vs "finish every check"
- Line deployment: roll four lines vs shorten the bench; matchup preferences for home games with last change
- Goalie-pull rules: time and deficit thresholds
- Timeout strategy and coach's-challenge policy
- Shot selection: shoot more vs wait for higher-quality chances

**Player management:**
- Practice focus and development priorities for individual players and prospects
- Load management: rest days, maximum ice-time targets for veterans
- Role promises to players, e.g. "top-6 role", which affect the relationship meter (17.10)
- Morale overview and trade requests (13.14)
- Injury management: return dates (accuracy depends on medical staff), with the option to rush a player back at extra re-injury risk (13.15)

**Staff:**
- Hire, fire and extend the GM, coaches, minor-league coaches, scouts, trainers and analytics staff
- Staff budget and salaries
- Scouting assignments: which leagues and regions to cover

**Front office:**
- Internal payroll budget, which can be below the cap
- Contract negotiation hub with salary recommendations (17.8–17.10)
- Trade block, trade proposals, incoming offers
- Waiver wire claims
- Draft board, draft-day controls and prospect signing (17.11)
- **Cap sheet:** current and future years, with projected cap space

**AI GM delegation:** hand any area to the team's AI GM, separately for:
- lineup
- minor-league moves
- minor contracts
- waivers
- trades
- the draft
- negotiations

Each delegated area can either be approved by the owner before it happens or run fully automatically.

**Business:** optional; the super admin can turn it off.
- Ticket pricing, which affects attendance (13.10)
- Promotions and theme nights, which affect the crowd's pre-game energy
- Fan loyalty initiatives

**Notifications and sim pace (for this owner):**
- Choose which events pause the simulation or send an alert:
  - trade offers
  - injuries
  - negotiation responses and deadlines
  - waiver claims
  - milestones
  - suspensions
- Auto-sim preferences when the owner is away, e.g. "let the AI GM handle everything for a week".

**Reports the owner sees:**
- Team and player stats, the game logs with modifier breakdowns (13.13), injury reports, the cap sheet
- **Staff evaluations:** scouting, salary recommendations and trade valuations. These come from the team's own staff, so their accuracy depends on staff ratings. The owner never sees true values unless the super admin enables it.

**Not in the team console:** league rules, schedule, rating scales, human-factor weights, event rates, other teams' ratings and settings, true values. These stay in the super admin console.

## 19. Possible extensions (not in scope yet)

- Importing real team and player data from a stats source (subject to section 22.4)
- Expansion teams and an expansion draft
- Third minor-league tier (ECHL-level) in full detail
- International tournaments (Olympics, World Championships) as playable events
- Media and press: interviews, rumors and the news feed as gameplay
- Fantasy-style historical seasons (e.g. an 82-game preset from a past season)

## 20. Realism safeguards and validation

### 20.1 Season-level and multi-season realism tests
Run full seasons (and 10-season franchise runs) with all-AI teams. Compare against recent NHL seasons:

**Standings and results:**
- **Spread of team points:** best and worst team, standard deviation of points.
- **Upset rates:** how often the team with more points wins a game, and how often the lower seed wins a playoff series.
- **Streaks:** distribution of win and losing streak lengths.
- **Close games:** OT frequency, one-goal games, comebacks from 2+ goals down.
- **Home win %:** ~53–55% (20.2).
- Points leaders' totals, the shape of the goal-scoring distribution, goalie save % spread.

**Season events:**
- Injuries: man-games lost per team, injury-type mix.
- Trades per season and the deadline spike; waiver claims; coach and GM firings; contract holdouts.
- Penalties, fights and suspensions per season.

**Multi-season health:**
- **Rating drift:** league mean and spread per attribute stay stable across seasons (16.4).
- **Player pool:** roster spots filled, entries vs exits per season, age distribution of NHL players.
- **No dynasties or collapses without a cause:** flag any team whose strength changes sharply without matching roster, staff or contract events.
- **Cap health:** teams stay cap-compliant; the distribution of cap spending is realistic.

The **realism report** is generated automatically after each validation batch and after changing settings. It shows every metric next to its target and a warning when it is out of range.

### 20.2 Effect budgets (no double counting)
Effects with several sources get **one calibrated total**, split among the sources by configurable weights. Turning a source off reallocates its share instead of reducing realism.

| Budget | Calibration target | Sources sharing it |
|---|---|---|
| Home advantage | Home win % ~53–55% | Last change, faceoff stick-down rule, crowd energy, referee home bias, visitor travel fatigue, ice familiarity |
| Pressure handling | Real late-game, OT and playoff performance spread | Experience, mental toughness, big-game trait, clutch |
| Comeback push | Real comeback rates by deficit and time | Score effects, in-game momentum, desperation, coach sparks, goalie pull |
| Fatigue | Real late-period, back-to-back and late-season drops | Shift, game, schedule, travel, season load, playoff wear |
| Penalty volume | ~3 power plays per team per game | True infractions, phantom calls, evening-up, retaliation, rivalry intensity |
| Physical escalation | Real fight and major-penalty rates | Rivalry, bad blood, crowd anger, referee game control, temperament |

### 20.3 Feedback loop control
Every modifier that feeds on recent results has:
- **decay toward neutral** (a half-life in games or days)
- a **hard ceiling**
- **diminishing returns**

Inputs are measured against **expectations, not raw results**, so a good team performing normally doesn't keep gaining confidence or momentum.

Known loops to test explicitly:
- winning → confidence → winning
- home goal → crowd energy → home goal
- losing streak → GM panic move → morale → losing
- ice time → development → more ice time
- refs evening up → angry crowd → home bias
- winning → culture → winning

### 20.4 Testing strategy
- **Rule unit tests:** every rule in 2a, 9.7, 17.7, 17.9, 17.11 and 17.12, e.g. cap legality, waiver exemption, lottery limits, tiebreakers, contract structure limits, OT penalty strength, the shootout shooter rule.
- **Determinism tests:** same seed and settings → identical event log.
- **Property tests:**
  - a team never has fewer than 3 skaters
  - stats always reconcile with the event log
  - the cap is never exceeded without a legal exception
  - every player is on exactly one roster
- **Exploit tests** (17.13): scripted "bad owner" strategies, such as fleecing trades, cap tricks and tanking. They must fail or be costly.
- **Regression:** the realism report is stored per version, and a change that moves metrics out of range fails the build.

### 20.5 Performance targets
- **Quick sim:** a full season of NHL plus minor-league games with box scores in minutes, not hours, on a normal laptop. Profile the game engine first.
- **Full play-by-play** only for games that are watched or flagged. Other games store a compressed event log, or the box score only, as set in config.
- Compressed storage of event logs.
- Season and career stats are computed incrementally.

## 21. Verify list (confirm before release, update each season)
Values in this spec that come from the current NHL rules or that need confirming. Each should be checked against an official source (NHL/NHLPA agreement, NHL rulebook, NHL media releases) and stored in config with its source and date.

- League structure, calendar and dates for 2026–27: season dates, holiday freeze dates, trade deadline, playoff roster rules.
- Standings, playoff and lottery tiebreaker order.
- Salary cap figures (2026–27: $104M ceiling, $76.9M floor, $20.8M max) and the projected future cap.
- Minimum salary schedule; ELC maximums by draft year; amount that can be buried in the minors.
- Contract structure rules: term limits, variance, lowest-year %, signing bonus limit, no deferred salary.
- Playoff salary cap and long-term injured reserve rules.
- **Two-way contract minimum minor-league salary; AHL minimum salary** (current values come from weaker sources).
- **RFA/UFA eligibility thresholds, Group VI rules, qualifying offer rules** (the new agreement changed qualifying offers), arbitration eligibility, offer sheet compensation tiers.
- **Waiver exemption table**, priority switchover date, re-waiver exemption.
- Draft: eligibility ages, lottery odds table, rights durations by league, the junior-to-AHL exception.
- EBUG procedure under the new agreement; preseason limits.
- AHL veteran rule and roster limits.
- No-trade/no-movement clause eligibility; trade flip restrictions.

## 22. Build plan for Claude Code

### 22.1 Phases
Build in order. Each phase ends with its tests passing and its part of the realism report in range.

1. **Game engine core:** game state, possession and zone state machine, events, stoppages, penalties, OT and shootout, rule edge cases (9.7), event log, stats, box score. Uses simple team ratings (section 7). Calibrated to the section 10 targets.
2. **Players and ratings:** player model (12), baselines (15), lineups and line changes by players, goalie model, readable play-by-play text. Recalibrate.
3. **Human factors:** modifiers (13), experience and leadership (14), effect budgets and loop control (20.2–20.3), officials, crowd, calls and consequences, morale, injuries.
4. **Season:** league structure and schedule generator (2a), standings, playoffs, the in-season and offseason lifecycle (16), season stats, multi-season realism tests (20.1).
5. **Front office:** contracts and cap (17.7), valuation (17.8), trades with anti-exploit rules (17.3, 17.13), waivers (17.12), minor leagues (17.5–17.6), draft (17.11), free agency (17.9), staff market and culture (17.14).
6. **Owner mode and consoles:** negotiation (17.10), team console (18.2), super admin console (18.1), roles and permissions, audit log.
7. **Live viewer and polish:** rink viewer, commentary levels, exports, performance tuning.

### 22.2 Working rules for each phase
- Start each phase by listing the spec sections it covers and the defaults it will use.
- Keep all parameters in config (18.1). The engine never hard-codes tunable values.
- Write the tests (20.4) alongside the code, not after.
- Produce the realism report at the end of the phase and compare it to the targets.

### 22.3 Project layout
See section 11. Add `tests/rules/`, `tests/properties/`, `tests/exploits/` and `realism/` (report generation and stored reports).

### 22.4 Real names and licensing
- Real NHL team names, logos and player names are fine **for personal, private use**.
- A public or commercial release needs licensing from the NHL and NHLPA.
- Ship a **fictional league preset** (invented team and player names with the same structure) as the default for any public build, and keep real names in a separate, optional data pack.
