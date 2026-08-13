---
title: "Building a Fantasy Football Draft Engine for Seven Leagues at Once"
date: 2026-09-08
draft: false
tags: ["python", "fantasy-football", "nfl", "textual", "tui", "statistics", "sleeper", "yahoo"]
---

I play in seven fantasy football leagues. Five on Sleeper, two on Yahoo. Six snake drafts and one auction. Team counts of 10 and 12. PPR, half-PPR, and one league that pays a point for every first down. One of them scores individual defensive players. One scores kickers purely on field goal yardage with no penalty for a miss.

Every ranking site gives you one list. That list is built for a league that isn't yours. So I spent a week building a draft assistant that takes each league's actual scoring rules, rescores every player from raw stat projections, computes value over replacement against that league's roster construction, and puts the whole thing in a terminal UI I can drive during a live draft.

8,677 lines of Python across 32 files. Here's what's in it.

## The problem: the same player is not the same player

The clearest way to see why one ranking list can't work is to score the same five players under all seven of my leagues' rules. These are 2026 projections run through each league's actual scoring settings:

```text
player                Keepin' It  Culver's   ACC Shit  military  Disney   Grand Exp  FACtasy
Josh Allen                 388.5     412.5      368.8     368.8     368.8      337.5    416.6
Ja'Marr Chase              336.0     324.9      324.6     266.9     324.6      246.3    308.3
Trey McBride               243.5     245.7      245.7     194.5     245.7      181.2    223.0
Brandon Aubrey             191.1     189.8      178.2     213.4     178.2      191.1    178.2
Bijan Robinson             411.0     378.0      349.5     313.6     349.5      290.2    358.2
```

Josh Allen swings 79 points between my cheapest and most generous league. Ja'Marr Chase swings 90. Bijan Robinson swings 121 — more than a third of his own projection. Brandon Aubrey, a kicker, is worth 35 more points in the military industrial complex enthusiasts league than in ACC Shitfest, because one of those leagues pays per field-goal yard and the other doesn't.

And the rank ordering moves too. The best quarterback by VOR lands at overall #9 in Keepin' It Tight and overall #26 in Culver's Value Basket Lovers. Same player, same projection, same week — a first-round consideration in one league and a fourth-round one in the other, entirely because of roster slots and passing scoring.

That's the whole thesis. A consensus ranking is a weighted average of leagues you're not in.

## Getting the data

Two sources, merged on the player.

**Sleeper's player database** (`/v1/players/nfl`) is the spine: about 1,100 fantasy-relevant players with IDs, teams, positions, bye weeks, and ages. It's also where prior-season stats come from, which matter later for durability and kicker distance profiles. Sleeper reports listed positions rather than fantasy slots, so the defensive side arrives as nine different strings that all have to collapse into three:

```python
_POSITION_MAP: dict[str, Position] = {
    "QB": Position.QB, "RB": Position.RB, "WR": Position.WR, "TE": Position.TE,
    "K": Position.K, "DEF": Position.DST, "DST": Position.DST,
    "DL": Position.DL, "DE": Position.DL, "DT": Position.DL,
    "NT": Position.DL, "EDGE": Position.DL,
    "LB": Position.LB, "OLB": Position.LB, "ILB": Position.LB, "MLB": Position.LB,
    "DB": Position.DB, "CB": Position.DB, "S": Position.DB,
    "SS": Position.DB, "FS": Position.DB,
}
```

**FantasyPros projection exports** give the per-player stat lines — passing yards, carries, receptions, targets, field goal attempts — as CSVs per position. This is the part that matters: I need *stats*, not points. A points number is already baked to somebody else's scoring rules and can't be un-baked.

Both sources project the same season, so where they overlap I blend them on the stat line at equal weight:

```python
# How much weight Sleeper's projections carry against the FantasyPros CSVs when
# both cover a player. The two agree closely (r=0.965 on 2026), so this mostly
# damps the outliers where one source hasn't caught up with a depth-chart or
# injury change. Blending happens on the stat line, not on points, so
# everything downstream stays consistent.
_SLEEPER_PROJECTION_WEIGHT = 0.5
```

An `r` of 0.965 means the blend does almost nothing most of the time. It earns its keep on the handful of players where one source is stale on a depth chart change — the average pulls those halfway back rather than letting a single source's mistake propagate through the whole draft.

Name matching between the two is `rapidfuzz` with a threshold of 75, plus a tiebreak: when two players share a name key, the active one on a real NFL roster wins over the practice-squad namesake.

**Durability** comes from prior-season games played, and applies as a straight discount to projected points:

```python
"""Players who appeared in fewer than 14 games get a discount:
``durability = min(1.0, gp / 16)``.
"""
```

It's blunt. A player who managed 9 games last year carries a 0.56 multiplier into this year's projection. My own draft history says this is my single worst habit — Christian McCaffrey in round 1 for 4 games, Anthony Richardson in round 13 for 2 — so I'd rather over-correct than trust myself in the moment.

## Scoring from stat lines, not position averages

The first version of the scoring engine did what most of the DIY tools do: take a projection in standard scoring, then add a position-level delta for the league's rules. QBs get +X, RBs get +Y. It's fast and it's wrong in a specific way — every running back gets the same PPR adjustment even though one of them catches 80 balls and another catches 12.

The rewrite scores every player from their own stat line:

```python
def score_from_stats(stats: dict[str, float], scoring: ScoringSettings) -> float:
    pts = 0.0
    pts += stats.get("pass_yds", 0.0) * _pts_per_yard(scoring.passing.yards_per_point)
    pts += stats.get("pass_tds", 0.0) * scoring.passing.td
    pts += stats.get("pass_int", 0.0) * scoring.passing.interception
    pts += stats.get("pass_cmp", 0.0) * scoring.passing.completions
    # Projections don't break out sacks taken or pick-sixes, so estimate both.
    pts += stats.get("pass_att", 0.0) * _SACK_RATE * scoring.passing.sack
    pts += stats.get("pass_int", 0.0) * _PICK_SIX_RATE * scoring.pick_six
    ...
```

The position-average path still exists, but only as a fallback for players with no stat line at all. Anyone carrying real stats — including an all-zero line, which is genuinely worth about zero rather than a position average — gets scored directly.

The estimated rates are measured, not guessed:

```python
# Share of carries / catches that pick up a first down. Measured over the 2025
# season across every QB/RB/WR/TE with the split populated: 3140/14283 carries
# and 5037/11033 catches.
_RUSH_FIRST_DOWN_RATE = 0.22
_REC_FIRST_DOWN_RATE = 0.46

# Likewise for the QB rates, over every 2025 passer with 100+ attempts:
# 1187 sacks on 16331 attempts, and 24 pick-sixes on 345 interceptions.
_SACK_RATE = 0.073
_PICK_SIX_RATE = 0.07
```

### Milestone bonuses are a distribution problem

FACtasy Football pays +5 for a 300-yard passing game, another +5 at 350, another +5 at 400. Season projections don't tell you how many of those a quarterback gets — they tell you he'll throw for 4,200 yards. You have to model the week-to-week distribution.

The naive move is a normal approximation around the per-game average. That badly undercounts, because weekly yardage is right-skewed: a receiver averaging 45 yards a game still breaks 100 a few times a year, and a symmetric distribution around 45 says he basically never does.

So it's lognormal, with a coefficient of variation that falls as volume rises:

```python
def _milestone_points_per_game(milestones, avg_per_game, cv_model=_REC_YARD_CV) -> float:
    a, b = cv_model
    cv = min(a * (avg_per_game ** -b), _MAX_MILESTONE_CV)
    sigma_sq = math.log(1.0 + cv * cv)
    sigma = math.sqrt(sigma_sq)
    mu = math.log(avg_per_game) - sigma_sq / 2.0
    denom = sigma * math.sqrt(2.0)

    total = 0.0
    for m in milestones:
        prob = 0.5 * (1.0 - math.erf((math.log(m.threshold) - mu) / denom))
        total += prob * m.points
    return total
```

The `cv = a * mean ** -b` shape is fitted per stat family against 2021–2025 weekly game logs — 2,409 player-seasons. Low-volume players carry far more relative variance (cv around 1.9 at 5 yards per game versus 0.5 at 70), which a fixed spread simply can't express.

```python
_PASS_YARD_CV = (4.35, 0.50)
_RUSH_YARD_CV = (3.25, 0.42)
_REC_YARD_CV  = (1.20, 0.18)
```

I also tried the obvious refinement — use each player's *own* prior-season volatility instead of a volume-fitted curve. It's worse out of sample, and not marginally:

| Stat family | SSE, own volatility | SSE, fitted curve |
|-------------|--------------------:|------------------:|
| Receiving   | 474 | 389 |
| Rushing     | 270 | 231 |
| Passing     | 248 | 191 |

Best blend weight came out between 0 and 10%. Week-to-week boom-bust just doesn't persist year over year. It feels like a player trait and it isn't one.

Calibration check: the fitted model reproduces the 2025 season's actual count of 100-yard rushing games to within 4%, and 300-yard passing games to within 7%.

### Defenses have their own distribution problem

Points-allowed scoring is a ladder — 10 points for a shutout, 7 for holding an offense to 1–6, down to −5 for giving up 35+. Projections give you a season total of points allowed. The ladder pays per game.

Same treatment, normal approximation this time (points allowed isn't nearly as skewed as receiving yards), with a spread of 45% of the mean:

```python
def _points_allowed_points(season_total: float, dst: DSTScoring) -> float:
    mean = season_total / GAMES_PER_SEASON
    sigma = mean * _PTS_ALLOWED_STDEV_FRACTION

    per_game = 0.0
    for low, high, field in _PTS_ALLOWED_TIERS:
        award = getattr(dst, field, 0.0)
        if not award:
            continue
        lower = _normal_cdf(low - 0.5, mean, sigma) if low else 0.0
        upper = 1.0 if high is None else _normal_cdf(high + 0.5, mean, sigma)
        per_game += (upper - lower) * award

    return per_game * GAMES_PER_SEASON
```

### Kickers need stats nobody projects

The Grand Experiment scores kickers on field goal yardage only — 10 yards per point, no flat award per make, no penalty for a miss. The FantasyPros CSV gives you `fg_made`, `fg_att`, and `xp_made`. It does not give you field goal *yards*.

So the player database derives them from prior-season history: projected makes × the kicker's own average FG distance, with a league-wide fallback for anyone under 5 prior makes. Extra point misses come from his prior miss rate.

The check that it works is a single line in the league config:

```python
# Yahoo scores kickers purely on FG yardage + extra points here --
# no flat per-FG award and no penalty for misses. Verified against
# Brandon Aubrey's 2025 line: 1474 FG yds / 10 + 47 XP = 194.4 vs
# the 195.00 on the league page.
```

194.4 against the league page's 195.00. That's the kind of validation you want — take last season's real stats, run them through your scoring model, and compare against the number the platform itself computed.

### Reading scoring from the platform

Hand-maintained scoring configs drift silently. Somebody changes a setting in the offseason, you don't notice, and your entire draft board is subtly wrong. So the Sleeper leagues pull their settings from the API and translate them, with the hardcoded blocks demoted to fallbacks.

Sleeper reports scoring as a flat dict of about 67 keys, most of them zero in any given league. The translator maps the ones the engine understands — and, importantly, complains about the ones it doesn't:

```text
Keepin' It Tight: 6 Sleeper scoring rule(s) not modelled, points will be understated:
  def_2pt=4, def_st_ff=1, def_st_fum_rec=1, def_st_td=4, st_ff=1, st_fum_rec=1
ACC Shitfest: 6 Sleeper scoring rule(s) not modelled, points will be understated:
  def_st_ff=1, def_st_fum_rec=1, def_st_td=6, idp_qb_hit=1, st_ff=1, st_fum_rec=1
```

Nothing projects special-teams forced fumbles, so there's no honest way to model them. But an unmodelled rule that shows up in the log is a known limitation; an unmodelled rule that silently returns zero is a bug you find in week 6.

## VOR: what a player is worth *here*

Points alone don't rank players — the top quarterback outscores the top running back most years and is still not the first pick. What matters is the margin over the player you'd otherwise be starting at that position, which depends entirely on how many of them get started in your league.

Replacement level is the first player who *isn't* expected to start, one pick past the last starter across all teams:

```python
thresholds = {
    Position.QB: teams * roster.qb + sflex_qb + 1,
    Position.RB: teams * roster.rb + flex_rb + 1,
    Position.WR: teams * roster.wr + flex_wr + 1,
    Position.TE: teams * roster.te + flex_te + 1,
    Position.K:  teams * roster.k + 1,
    Position.DST: teams * roster.dst + 1,
}
```

FLEX slots get split 45/45/10 across RB/WR/TE. SUPERFLEX slots go onto the QB pool. IDP_FLEX gets distributed across DL/LB/DB in proportion to the dedicated slots each already has — without that, those positions get a replacement level of zero and the engine values every starting linebacker at his full projection.

In a 12-team league with one flex, replacement lands at QB13, RB25, WR25, TE13.

On top of raw VOR sit three multipliers.

**Predictability.** Kicker and defense scoring is close to random week to week, so their VOR edge doesn't survive contact with the season:

```python
_PREDICTABILITY = {
    Position.QB: 1.0, Position.RB: 1.0, Position.WR: 1.0, Position.TE: 1.0,
    Position.K: 0.15,    # kicker VOR is ~85% noise
    Position.DST: 0.20,  # DST is ~80% noise
}
```

This matters more than it looks. In ACC Shitfest, the top kicker's raw VOR of 33.3 would make him the 35th-best player on the board — a mid-round pick. Deflated, he's worth 5.0 and lands at 64th, which is where a kicker belongs. Same story for the top defense: 36.0 and 34th becomes 7.2 and 55th.

**Replaceability.** How easy it is to find replacement production on the waiver wire:

```python
_REPLACEABILITY = {
    Position.QB: 0.90,   # easy to stream on waivers
    Position.RB: 1.0,    # hardest to replace
    Position.WR: 0.95,   # moderate waiver availability
    Position.TE: 1.10,   # very hard to replace — premium TEs are scarce
}
```

**Age curves**, applied after VOR is computed, most severe match first:

```python
_AGE_CURVES = {
    Position.RB: [(30, 0.70), (29, 0.80), (28, 0.90)],
    Position.WR: [(33, 0.85), (31, 0.95)],
    Position.QB: [(38, 0.85), (35, 0.95)],
    Position.TE: [(31, 0.90)],
}
```

Here's the top of the ACC Shitfest board, 12-team PPR, out of 1,083 players:

```text
   name                     pos     proj     vor tier    adp
 1 Jahmyr Gibbs             RB     352.9   178.2    1    2.3
 2 Bijan Robinson           RB     349.5   174.9    1    1.9
 3 Puka Nacua               WR     339.6   137.3    1    4.8
 4 Ja'Marr Chase            WR     324.6   123.0    2    3.7
 5 Jonathan Taylor          RB     293.6   118.9    2    6.4
 6 Jaxon Smith-Njigba       WR     305.8   105.2    3    7.3
 7 Amon-Ra St. Brown        WR     301.0   100.6    4    8.2
 8 Trey McBride             TE     245.7    96.9    1   17.1
 9 Christian McCaffrey      RB     312.9    96.8    3    5.7
10 De'Von Achane            RB     268.8    94.1    3   13.6
```

McBride at #8 with an ADP of 17.1 is the model disagreeing with the market. He's projected for 245.7 points — 67 fewer than McCaffrey — and worth marginally more, because the tight end you'd otherwise start is so much worse than the running back you'd otherwise start.

### Tiers from natural breaks

Tiers aren't fixed-size buckets. A new tier starts wherever the VOR gap between consecutive players exceeds the mean gap by more than one standard deviation:

```python
gaps = [(group[i].vor or 0.0) - (group[i + 1].vor or 0.0) for i in range(len(group) - 1)]
threshold = statistics.mean(gaps) + std_factor * statistics.stdev(gaps)

tier = 1
group[0].tier = tier
for i in range(1, len(group)):
    if gaps[i - 1] > threshold:
        tier += 1
    group[i].tier = tier
```

Which produces this at tight end in ACC Shitfest:

```text
  T1 Trey McBride           vor   96.9
  T2 Colston Loveland       vor   60.3
  T3 Tyler Warren           vor   49.0
  T4 Brock Bowers           vor   33.3
  T4 Harold Fannin          vor   30.1
  T4 Kyle Pitts             vor   28.5
  T5 Travis Kelce           vor   18.4
  T6 Jake Ferguson          vor    6.6
  T6 Mark Andrews           vor    6.3
  T6 Isaiah Likely          vor    2.8
```

That's the useful shape: three players in tiers of one, then a cluster of three you can treat as interchangeable, then a long flat tail. If you miss McBride, the decision isn't "take Loveland instead" — it's "wait, because Bowers/Fannin/Pitts are the same pick."

## Auction pricing is a different problem

The Grand Experiment is a $200 auction. My first attempt reused `player.vor` and converted it to dollars proportionally. It priced the best player in the league at $118 — 59% of a roster's entire budget on one man, with $82 left for fourteen more.

The bug is conceptual. VOR measures against the last *starter*, which is right for ranking a snake draft, where you're asking "who do I start." An auction fills every bench spot on every team, so the marginal player is far deeper and the value curve is much flatter.

So auctions get their own replacement depths, with bench spots split across the skill positions in proportion to starters:

```python
depths = {'QB': 21, 'RB': 41, 'WR': 62, 'TE': 21, 'K': 12, 'DST': 12}
```

WR62 rather than WR25. That alone flattens the curve enormously.

The second correction is reliability. A projected edge only pays out to the extent the rankings hold up, which I measured from the archived backtest data as the Spearman correlation between preseason rank and actual finish, 2022–2025:

| Position | Reliability |
|----------|------------:|
| QB | 0.77 |
| RB | 0.77 |
| WR | 0.79 |
| TE | 0.79 |
| K  | 0.39 |
| DST | 0.38 |

Skill positions cluster near 0.78. Kickers and defenses are barely half that. A ranking that correlates 0.39 with the outcome delivers roughly 0.39 of the gap it predicts — ordinary regression to the mean — so surplus gets shrunk by these factors before it's priced.

The sanity check on the size of that correction is what those tiers actually returned. Mean points scored by the preseason-drafted tier over the waiver tier below it: RB +79.7, WR +66.0, TE +36.7, K +18.1, DST +20.9. Drafting a kicker well is worth about 18 points across a season — roughly a quarter of what it's worth at running back, which is the same ratio the reliability numbers imply.

Then the arithmetic: reserve $1 per roster spot, distribute the rest by share of shrunk surplus.

```python
return {
    pid: max(1, round(1 + value / total_surplus * distributable))
    for pid, value in surplus.items()
}
```

Which gives, for a 12-team $200 league:

```text
  $ 52 Bijan Robinson         RB
  $ 51 Jahmyr Gibbs           RB
  $ 48 Puka Nacua             WR
  $ 44 Ja'Marr Chase          WR
  $ 41 Christian McCaffrey    RB
  $ 41 Jaxon Smith-Njigba     WR
  $ 40 Jonathan Taylor        RB
```

$52 for the top player rather than $118. That's a number you can actually bid.

## The recommender: cost of waiting, not shape of the pool

Ranking is the easy part. The hard part happens at the table: it's pick 19 of a snake draft, you're up again at 30, and you have to decide between the best player available and the position you'll never fill if you wait.

The original scarcity metric measured VOR dropoff across a fixed window of the top 12 remaining players at each position. It's a bad metric, and it fails in a specific and infuriating way: it reads a position as "flat" *precisely because* its best players just went off the board. The cliff already happened; the metric sees smooth ground.

The replacement measures the actual quantity you care about — what passing on this player costs you, given who's likely to still be there at your next pick:

```python
def _wait_cost(engine, player, next_pick, runs) -> tuple[float, Player | None]:
    """Value given up by passing on *player* and addressing the spot later.

    His value minus the best player at the same position likely to still be
    there when you're next up. That difference is the real cost of waiting --
    a tight end 36 points clear of his own fallback is a more urgent pick
    than a receiver 8 points clear of his, whatever their raw values say.
    """
    mine = player.vor or 0.0
    for other in engine.get_available(player.position.value):
        if other.id == player.id or other.vor is None:
            continue
        survival = _survival_probability(other.adp, next_pick)
        if survival is None:
            continue
        if in_run:
            survival *= _RUN_SURVIVAL_PENALTY
        if survival >= 0.5:
            return max(0.0, mine - other.vor), other
    # Nobody else at this position is expected to last: the whole value walks.
    return mine, None
```

Survival probability comes from ADP treated as a distribution rather than a promise:

```python
def _survival_probability(adp, next_pick):
    sigma = max(_ADP_SIGMA_FLOOR, adp * _ADP_SIGMA_FRACTION)
    gone = 0.5 * (1.0 + math.erf((next_pick - adp) / (sigma * math.sqrt(2.0))))
    return max(0.0, min(1.0, 1.0 - gone))
```

Sigma is a quarter of ADP with a four-pick floor. The spread widens later in the draft where opinions diverge, which the proportional term captures.

### Run detection, and how it embarrassed me

If four running backs go in six picks, ADP is lying to you — the pool is draining faster than the averages assume, and the player you were counting on falling won't. So a detected run applies a 0.75 penalty to every survival probability at that position.

The first version fired constantly in round 1. Three receivers in twelve picks is 25%, which is roughly the ordinary early-round share of receivers — not a run. Worse, with a 12-pick lookback window, the opening picks of a draft only fill part of it, so an unguarded count read "3 of 5" as though it were "3 of 12."

```python
# At pick 6 that fired a spurious URGENT run and outranked a player with 18 more VOR.
_RUN_MIN_PICKS = 8
_RUN_MIN_COUNT = 3
_RUN_MIN_SHARE = 0.40
```

Three guards: a minimum sample, a minimum count, and a minimum share of the window.

### What it looks like at the table

Simulating an ACC Shitfest draft where everyone else picks at ADP and I'm at slot 6:

```text
--- USER PICK 6 (round 1), next pick at 19 ---
  100% Jonathan Taylor      RB  vor  118.9 | Gone by your next pick: ADP 6, you're up
       again at 19; BPA: highest VOR available; Waiting costs 46: next RB at 19 is
       Kenneth Walker
   98% Jaxon Smith-Njigba   WR  vor  105.2 | Gone by your next pick: ADP 7 ...
       Waiting costs 54: next WR at 19 is George Pickens
   95% Amon-Ra St. Brown    WR  vor  100.6 | ... Waiting costs 49: next WR at 19 is
       George Pickens
   86% Ashton Jeanty        RB  vor   93.3 | ... Waiting costs 21: next RB at 19 is
       Kenneth Walker
```

Taylor ranks first despite Smith-Njigba having a *higher* wait cost, because raw VOR still carries the largest single weight. But look at Jeanty and Achane: nearly the same VOR as Smith-Njigba, and a wait cost of 21 instead of 54, because the running back you'd settle for at pick 19 is much closer to them than the receiver you'd settle for is to Smith-Njigba. That's the signal I actually want at the table, and it's the one a static ranking can't give you.

By round 2 the run detector has something to say:

```text
--- USER PICK 19 (round 2), next pick at 30 ---
  100% Kenneth Walker       RB  vor   72.8 | RUN! 6 of the last 12 picks were RBs —
       you have 1/2 ... Waiting costs 27: next RB at 30 is Travis Etienne
```

The composite score behind the ordering:

| Component | Weight |
|-----------|-------:|
| VOR (normalized to best available) | 35% |
| Starter need (2+ empty starter slots) | up to 55% |
| Cost of waiting | 15% |
| Position run urgency | up to 15% |
| Survival to next pick | 10% |
| QB/pass-catcher stack | +5% |
| Bye week conflict | −5% |

Need dominating VOR is deliberate. A roster with two empty starting receiver slots in round 9 has a problem that no amount of best-player-available solves.

## Backtesting: does any of this hold up?

Five seasons, 2021–2025, seven leagues. Preseason FantasyPros consensus rankings against actual end-of-season results, rescored under each league's rules, with players matched across sources by fuzzy name.

```text
                       Multi-Year Consistency (2021-2025)
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━┓
┃ League                      ┃ Avg Corr ┃ Avg MAE ┃ Avg VOR Edge ┃   Win Rate ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━┩
│ Keepin' It Tight            │    0.768 │   107.2 │       +405.3 │  80% (4/5) │
│ Culver's Value Basket       │    0.764 │   107.8 │       +588.9 │ 100% (5/5) │
│ ACC Shitfest                │    0.766 │   107.6 │       +547.0 │ 100% (5/5) │
│ military industrial complex │    0.760 │   108.6 │       +501.7 │ 100% (5/5) │
│ Disney Dreamland            │    0.766 │   107.6 │       +434.2 │  80% (4/5) │
│ The Grand Experiment        │    0.763 │   108.0 │       +452.6 │ 100% (5/5) │
│ FACtasy Football            │    0.763 │   108.0 │       +573.0 │ 100% (5/5) │
└─────────────────────────────┴──────────┴─────────┴──────────────┴────────────┘
```

**The correlation number is the honest headline.** Spearman correlation between preseason consensus rank and actual finish sits at 0.76, remarkably stable across leagues and years — 0.74 in 2021 up to 0.79 in 2024. Mean absolute rank error is about 107 places, on a pool of roughly 500 players.

Read that again: the consensus is *directionally right and individually terrible*. It gets the broad shape of the season correct while being off by a hundred ranking places on the average player. Every hour you spend agonizing over pick 4 versus pick 7 is being spent on a distinction the data cannot support.

**The "VOR Edge" column needs a caveat, and it's a big one.** That column compares a draft run in perfect end-of-season order (under the league's own scoring) against a draft run in preseason consensus order. It is a *ceiling* — the most a perfectly scoring-aware ranking could have captured — not a measurement of what my engine would have done live, because it uses actual results rather than projections. Roughly 500 points a season is the size of the prize available to anyone who ranks in their league's scoring rather than someone else's. It is not 500 points I'm claiming to bank.

The right way to close that gap is to run the engine's *projections* through the same simulation for each historical year, which needs archived preseason projections rather than archived preseason rankings. That's the next thing I'm building.

Year by year for one league, where the caveat is visible:

```text
                  Keepin' It Tight — Year-by-Year
┏━━━━━━┳━━━━━━━━━┳━━━━━━━┳━━━━━━━┳━━━━━━━━━━━┳━━━━━━━━━━━┳━━━━━━━━┓
┃ Year ┃ Players ┃  Corr ┃   MAE ┃ VOR Draft ┃ ECR Draft ┃   Edge ┃
┡━━━━━━╇━━━━━━━━━╇━━━━━━━╇━━━━━━━╇━━━━━━━━━━━╇━━━━━━━━━━━╇━━━━━━━━┩
│ 2021 │     520 │ 0.745 │ 111.4 │    2895.5 │    3032.3 │ -136.8 │
│ 2022 │     534 │ 0.758 │ 109.9 │    2851.3 │    2301.6 │ +549.6 │
│ 2023 │     498 │ 0.749 │ 104.5 │    2897.9 │    2260.6 │ +637.3 │
│ 2024 │     545 │ 0.796 │ 110.3 │    2961.4 │    2361.6 │ +599.8 │
│ 2025 │     516 │ 0.791 │  99.7 │    2856.3 │    2479.7 │ +376.5 │
└──────┴─────────┴───────┴───────┴───────────┴───────────┴────────┘
```

2021 goes negative — even perfect hindsight ordering lost to consensus order in that league that year, because roster-limit interactions in the snake simulation can hand the consensus draft a better positional mix. That single negative row is the most useful thing in the table. A backtest that never loses isn't measuring anything.

## The interface

All of this runs in a [Textual](https://textual.textualize.io/) TUI, because a draft is a keyboard-driven, 90-second-per-pick environment and a terminal is genuinely the right tool for it.

```text
Ctrl+Z  Undo        Ctrl+S  Save         Ctrl+L  Sync
Ctrl+D  Draft log   Ctrl+B  Draft board  /       Search
F1 All  F2 QB  F3 RB  F4 WR  F5 TE  F6 K  F7 DST
```

Picks are entered by fuzzy name match, so "jeff" finds Justin Jefferson. When several matches score within 5 points of each other, the higher-projected player wins the tie — which is what you meant when you typed three letters with the clock running.

Sleeper leagues sync live. A background poller watches the draft API and records picks as other managers make them, by player ID rather than name, so the board stays current without me typing anyone else's picks.

And because I don't always want to draft from the machine the code lives on, it serves over HTTP via `textual-serve`:

```bash
uv run draft serve --host 0.0.0.0 --public-url https://draft.example.com
```

Two things I learned building that. First, the served page builds its stylesheet, script, and websocket URLs from the address the server was given — so binding `0.0.0.0` without `--public-url` emits `http://0.0.0.0:8000/static/...`, and the browser loads the intro dialog and then nothing. There's now a warning for exactly that mistake. Second, it has no authentication of its own, so it prints a loud warning when bound to anything but localhost:

```text
Binding to 0.0.0.0: anyone who can reach this port can make picks.
No password is required.
```

Reach it over Tailscale or an authenticated tunnel. Do not open a port to the internet that lets strangers draft your team.

## Caveats

- **Projections are the ceiling.** Everything downstream — VOR, tiers, auction values, wait cost — inherits whatever error is in the FantasyPros and Sleeper projections. A better scoring model on bad projections is still bad. The 0.76 correlation is roughly what preseason consensus is worth, and no amount of arithmetic on top of it changes that.
- **The FLEX split is a guess.** 45/45/10 across RB/WR/TE is a reasonable approximation of how flex slots get used, not a measurement. It moves replacement level, which moves everything.
- **Some scoring rules aren't modelled.** Special-teams forced fumbles, defensive two-point returns, and QB hits show up in the warning log because nothing projects them. Those leagues' point totals are slightly understated across the board — which mostly cancels out in a ranking, but not entirely.
- **Age curves and replaceability multipliers are judgment, not regression.** The predictability deflators for K and DST come from measured correlations. The RB age cliff at 28/29/30 and the 1.10 tight end premium come from received fantasy wisdom that I happen to believe. They should be fitted.
- **The backtest measures a ceiling, not the engine.** Covered above, but worth repeating, because a table full of green numbers is exactly the kind of thing that talks you out of reading the code that produced it.

## Technical notes

Python 3.13+, `uv` for dependency management, `pydantic` for the config and state models, `httpx` for every API call, `rapidfuzz` for name matching, `click` for the CLI, `textual` for the UI. No pandas, no numpy — the whole engine is standard-library math over lists of pydantic models. Rescoring all 1,083 players to a league's rules and computing VOR and tiers takes about 12 ms.

Draft and auction state serialize to JSON and reload, so a crashed terminal costs nothing.

Historical API responses (ECR, actual stats, the Sleeper player map) are cached gzipped under `data/backtest/`, which is why a five-season, seven-league backtest runs offline in a couple of seconds instead of hammering two APIs for several minutes.

The whole thing is containerized and built in CI, mostly so that on draft night the only thing standing between me and a working board is `docker run`.
