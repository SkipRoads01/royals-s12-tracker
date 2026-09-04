# Royals S12 Tracker — site build spec

`index.html` is regenerated and published from game notes (this happens via
Claude Code on the Mac, which pushes "Publish site update" commits). This file
records customizations that MUST be preserved on every regeneration, so they
don't get dropped when the site is rebuilt. Treat everything below as a
requirement of the generated `index.html`.

## 1. Overview stat strip tiles

The overview strip (`.strip`) must include these tiles, in this order. The last
three are custom additions beyond the auto-generated stats:

| Label | Meaning |
|---|---|
| Runs For | total runs scored |
| Runs Ag. | total runs allowed |
| Run Diff | run differential (`+N`, green when positive) |
| 1-Run | record in one-run games |
| Sweeps | series sweeps |
| Shutouts | shutouts thrown |
| Comebacks | come-from-behind wins, plus their share of total wins (see below) |
| Errors | total errors |
| Most Runs | most runs scored in a single game |
| **Most Runs Allowed** | most runs allowed in a single game (max opponent R over all games) |
| **Pace** | projected 162-game record at the current win rate (**only from game 10 on** — see below) |
| **BLL** | Blown Lead Losses (see definition below) |

Tile markup pattern:
`<div class="stat"><div class="k">LABEL</div><div class="v num">VALUE</div></div>`

**Pace** = project the current record over 162 games: `wins = round(winPct * 162)`,
`losses = 162 - wins`, formatted `W-L` (e.g. 12-0 → `162-0`, an 8-4 start → `108-54`).

**Pace is omitted entirely until 10 games have been played.** A projection off
two or three games is noise, not a record. Before game 10, do not emit the tile
at all — no dash, no placeholder — so the strip carries 11 tiles early and 12
from game 10 on. This is the only conditional tile; every other tile above is
always present, including `BLL` at 0.

**Comebacks** carries a second value: comeback wins as a percentage of TOTAL
WINS, rendered with the tile's built-in `<small>` slot:
`<div class="v num">5 <small>38.4%</small></div>`. One decimal place,
truncated rather than rounded (5 of 13 wins = 38.4615... -> `38.4%`).

The `<small>` must sit flush against the tile's right edge, on the same
baseline as the big number. That comes from `.stat .v`, which must stay a
baseline-aligned flex row:
`display: flex; align-items: baseline; justify-content: space-between; gap: 6px;`
Tiles with a single value are unaffected (one flex item sits at flex-start).

**BLL (Blown Lead Losses)** = number of games where KC held a lead at any point
and went on to LOSE that game. A true blown-lead loss. Determine "held a lead"
from the inning-by-inning recap (box scores only show finals). While the team is
undefeated this is `0`. Keep the tile even when it is 0.

## 2. Batting "FP Hits" tile — show share of total hits

The batting-splits "FP Hits" tile must include an `st-sub` line expressing
first-pitch hits as a percentage of the team's TOTAL hits, matching the format
of the strikeout tiles:

`<div class="st-sub">of {totalTeamHits} H &middot; {pct}%</div>`

where `pct = round(fpHits / totalTeamHits * 100)`. (This applies to the batting
FP Hits tile only, not the pitching "FP Hits Allowed" tile.)

## 3. No instruction text

Do NOT add instructional helper text anywhere on the site — e.g. "tap a tile for
the rundown", "tap a card for games", or similar. Section subtitles should carry
data only (e.g. "KC batters", "KC staff"), never usage hints.

## 4. Mobile-safe text (no raw non-ASCII typography)

Some mobile browsers/webviews decode the page as Latin-1 and turn literal UTF-8
punctuation into mojibake (e.g. `·` → `Â·`, `–` → `â€"`). Emit all such
characters as charset-independent ASCII so the page renders identically
everywhere. The output `index.html` should contain NO bytes > 0x7F outside of
embedded base64 image data.

Use, by context:

- **HTML text**: entities — `&middot;` (·), `&ndash;` (–), `&mdash;` (—),
  `&rsaquo;` (›), `&lsaquo;` (‹). Records/scores read fine with `&ndash;` or a
  plain ASCII hyphen.
- **CSS `content:`**: unicode escapes — `\25B2` (▲), `\25BC` (▼).
- **JavaScript**: `\u` escapes in regex/strings — `–` (–), `−` (−);
  use ASCII (`->`, `--`) in comments. Note the sort-key regex must still match
  the en-dash that entities decode to in the DOM, so write it with the escape:
  `/^(\d+)[\u2013-](\d+)$/`.

## 5. "Recent Headlines" section (Overview)

The Overview panel must carry a `Recent Headlines` section between `Inside the
Numbers` and `Game Log`. It takes **no subtitle** — the heading stands on its
own. It covers **player-level** notes: active hitting streaks, streaks just
snapped, short hot/cold runs, recent baserunning, a start or a relief outing.
Keep it distinct from `Inside the Numbers`, which stays **team-level season
context**. Do not repeat a fact in both; hitting-streak lines belong here.

**Every row names the game or series it comes from** — `in G3`, `in the 8-2
win at ATL`, `in all 3 games at ATL`. Never a bare "in the finale".

**A row need not be about the last game.** The upcoming series is fair game
— a homestand starting, a first look at an opponent — as long as it is
drawn from the schedule already on the page rather than invented.

Markup: a `.hl-card` wrapper holding one `.hl` row per headline:

```html
<div class="hl"><div class="hl-fig num">14</div><div>
  <div class="hl-k">Hit streak</div>
  <div class="hl-txt"><b>Garcia</b> has hit safely in every game this season.</div>
</div></div>
```

- `.hl-fig` is a fixed-width, right-aligned column so 1- and 2-digit figures
  share a right edge and every headline's text starts at the same x.
- Add `past` to the figure (`class="hl-fig num past"`) for anything in the past
  tense — a snapped streak, a cold stretch. It renders muted instead of blue so
  those items recede.
- The figure carries the number; the sentence must add NEW information rather
  than restating it (write "extended it with a 3-for-4 night", not "riding a
  12-game streak" next to a `12`).
- Aim for ~6 rows, active/positive items first, past-tense items last.

**Derive streaks from the recaps, never by assumption.** A streak spans games
the player actually appeared in — a game missed does not break it (Perez's
12-game streak skips G10, when he did not play). Verify any derivation against
known values before publishing.

## 6. Game-card chips

Game-log cards carry optional tag chips in `<span class="tags">`. When a game
earns no chip, omit the `tags` span entirely rather than emitting an empty one.
The vocabulary so far:

| Chip | Class | Meaning |
|---|---|---|
| Sweep | `chip gold` | series sweep completed |
| Shutout | `chip gold` | KC threw a shutout |
| Walk-off | `chip gold` | KC won it in its last at-bat |
| Comeback | `chip blue` | KC trailed at some point and won |

Gold marks a rare team accomplishment, blue a game-shape note. A game can carry
more than one (G15 is both Comeback and Walk-off).

## 7. Pitching Leaders — "3-Pitch K Looking" tile

The Pitching Leaders grid must carry a `3-Pitch K Looking` tile, placed last,
after `K Looking`. The three K tiles therefore run in order: `3-Pitch K`,
`K Looking`, `3-Pitch K Looking`.

**3-Pitch K Looking** = the batter struck out on three pitches AND took the
third one for a called strike. It is its own stat, not a synonym for either
neighbour:

| Tile | Counts |
|---|---|
| `3-Pitch K` | any strikeout on three pitches, swinging or looking |
| `K Looking` | any strikeout ending on a called third strike, at any count |
| `3-Pitch K Looking` | both at once — three pitches, third one called |

It is a subset of each of the other two, so a pitcher's `3-Pitch K Looking`
total can never exceed either. In the game notes it appears verbatim as
"3-pitch K looking" (e.g. G16 TOP 9, `Hays 3-pitch K looking`), and that
phrasing is the only thing that counts — a bare "0-2 K" does not, even though
it also took three pitches. The same event increments all three tiles.

The season totals already live in the Pitching Splits tile of the same name;
the leaders tile shows the top of that rundown.

**A category nobody has recorded has no leader.** When the top mark in a
leaders category is `0` &mdash; no saves yet, no walks drawn yet &mdash; do not
crown the first name off the roster. Emit a single row reading `None yet` with
the `0`, the same wording the splits rundowns use. Naming a starter as the
saves leader because everyone is tied at zero is exactly the kind of line
section 8 exists to keep off the page. This applies to both leaders grids.

**Nobody leads a category without appearing.** A leaders row requires the
player to have actually played: a hitter needs a plate appearance, a pitcher an
appearance on the mound. A pitcher at 0 IP has no ERA, no WHIP and no
strikeouts &mdash; his card reads `N/A` per section 10.3, and `N/A` is never a
value that can top a list. The `min` in the grid's subtitle is the floor, not
the only test.

**Ties show every leader.** `.lead` stacks one `.row` per name, and the grid
row absorbs the extra height, so emit a row for each pitcher on the top mark
rather than picking one (Cameron and Leahy both sit on 3 through G16).

## 8. Sample-size discipline — write like a baseball writer

`Inside the Numbers` and `Recent Headlines` are prose. They must read the way a
beat writer would actually write, not the way a spreadsheet would. A stat that
is technically correct but that nobody covering the sport would ever say does
not go on the page.

### 8.1 No 162-game extrapolations before 25 games

Do NOT project any counting stat over a full season until **25 games** have been
played. No "162-homer pace" off an opening-day home run, no "972-run pace" off a
six-run game, and the same for RBI, stolen bases, doubles, strikeouts, saves, or
anything else. Before game 25 the honest framing is the raw total and the game
it came in.

This covers prose only. The `Pace` tile has its own, earlier threshold (game 10,
see section 1) because a projected W-L is a standings convention rather than a
claim about a player.

| Games played | Allowed |
|---|---|
| 1&ndash;9 | raw totals, game shape, what actually happened |
| 10&ndash;24 | the above, plus the `Pace` tile |
| 25+ | the above, plus season-long extrapolations in prose |

### 8.2 No streak or record framing until the streak is one

A streak needs to exist before it is a story:

- Do not name a win or loss streak in prose until it reaches **4 games**. One
  win is a win. (The header `streak-pill` — `W1`, `L2` — is exempt: it is the
  standings-style current-streak indicator every scoreboard carries, not a
  note.)
- Do not invoke a franchise or MLB record as a comparison unless the team or
  player is realistically in range of it: within **3** of the mark, and never
  before game 25 for a season-long record. "12 more wins would tie the record
  for a season-opening win streak" after one win is exactly the line to cut.

### 8.3 Rate stats need a sample

Do not present a rate stat whose whole content is one game restated. After one
start, "a 2.00 team ERA" and "14.0 K/9" are just "two earned runs" and "14
strikeouts" wearing a disguise; write the plain version. Minimums before a rate
stat may be quoted in prose:

| Stat | Minimum |
|---|---|
| Team ERA, opponent AVG, K/9, team AVG/OPS | 10 team games |
| Individual AVG / OBP / SLG / OPS | 25 plate appearances |
| Individual ERA / WHIP | 15 innings pitched |

Below the minimum, use the counting stat and the context ("two hits in the
opener", not "hitting .500").

### 8.4 Walks are scarce in this game &mdash; a low total is not a story

Walks come far less often in this franchise than in real baseball. A small walk
count is the normal state of things here, not a finding, so it does not earn a
line. Do not write "21 hits and not one walk," "the offense has drawn three
walks all year," or any comparison of the team's walk rate to a real-life one.
The same goes for the staff's walk total.

A walk that *did* something is fair game, because the story is the inning
rather than the count: one that loaded the bases, forced in a run, or set up
the go-ahead hit. Atlanta's 7th in G2 &mdash; three doubles and a walk, and the
walk scored &mdash; is the shape that earns a mention.

### 8.5 Everything must trace to the recaps

Every figure and every claim in these two sections must be derivable from the
game notes on the page — the box scores and the inning-by-inning recaps. Do not
assume, round up, or carry a fact forward from an earlier build without
re-deriving it. Do not invent record-book context; if a record is cited, it must
be one that is actually verifiable, and section 8.2 still gates whether it
belongs at all.

### 8.6 Grammar

Agree number with the value: "through 1 game", not "through 1 games"; "1 home
run", not "1 home runs". This bites most often on opening day, when nearly every
count is 1.

### 8.7 Voice: short, active, and in the vernacular

Write these two sections the way a broadcast or beat writer talks, not the way
a recap generator writes.

- **Name the club `KC` or `the Royals`.** `Kansas City` is fine once for
  variety but reads formal on a page that is entirely about them.
- **Do not lean on `it`.** A clause like "the 4&ndash;2 edge it carried into the
  seventh is the first lead it has given away" is two pronouns chasing one
  noun. Recast so the subject is concrete, or drop the pronoun outright: "the
  4&ndash;2 lead in the seventh was the first one to get away."
- **One idea to a sentence.** Prefer a period to a semicolon; prefer an active
  verb to a noun phrase ("Mears faced three men and retired none", not "Mears
  recorded no outs against three batters faced").
- **Innings are numerals with an ordinal suffix** &mdash; `the 7th`, `the 2nd`,
  `the 9th`, never `the seventh`. This is how a box score and a broadcast both
  write them.
- **Avoid `that` as a pointer back to an earlier bullet** &mdash; `that lead`,
  `that seventh`, `that inning`. Each bullet has to stand on its own, so name
  the thing or restate the number: "to go from two runs down to a 5&ndash;4
  lead", not "turned that lead into a deficit".
- **Reach for the plain verb first.** `homered`, not "went deep"; `4 of the 13
  were home runs`, not "went over the wall"; `struck out 11`, not "went down on
  strikes". The sport's idiom is welcome only where it is genuinely the plainest
  wording &mdash; the other way, singled home, on the year. Invented color the
  recaps do not support is worse than a flat sentence, and section 8.5 still
  governs the facts.

### 8.8 Numerals for stats, and locate every stat in a game or series

- **Write a stat value as a numeral**, however small: `4 errors`, `9 starters`,
  `struck out 11`, `7 home runs, 6 doubles and 1 triple`. Spelling one out is
  right only where the number is editorial rather than a stat &mdash; "through
  three games", "all 3 games at Atlanta" (3 is the stat, three is the frame).
- **Give the percentage with any N-of-M stat.** `14 of KC's 34 hits have gone
  for extra bases (41%)`, `4 of the 13 (31%) went over the wall`. Round to a
  whole number.
- **Name the game *and* the opponent: `in G3 vs. ATL`.** A bare `G3` says when
  but not who, and a reader six weeks later has no idea. Use `vs. {OPP}` for
  either venue &mdash; it reads as "against", not as a home/away marker, which
  the schedule already carries. Once a line has named the opponent, later game
  numbers in the same line stay bare: "led off G1 and G2 vs. ATL, then singled
  twice in G3", not the opponent three times in one sentence. **"through three games" is too
  vague** for a season total &mdash; use `vs. ATL` when the whole total came in
  one series, or `last 3 games` / `last 5 games` when it did not. (The
  `story-lead` line that states the record, "2-1 through 3 games", is the
  exception: it is the standings frame, not a stat locator.)
- **Call them `KC pitchers`, not "the staff".** `Staff` stays as the totals-row
  label in `Opp Pitching` and in the `Staff rundown` hints, where it is a table
  heading rather than prose.
- **Quantify a vague plural.** `Lowe made 2 errors`, never "made both errors"
  &mdash; the reader should not have to look up how many both is.
- **Bold the whole stat expression, never a fragment of it.** `<b>4 of the 13
  (31%)</b>` and `<b>14 of KC's 34 hits (41%)</b>` &mdash; not `<b>4</b> of the
  13 (31%)`. The count, what it is out of, and the percentage are one figure and
  are emphasized as one.

Section 8.3 still governs *which* stats may appear at all; this section governs
how the ones that qualify are written.

### 8.9 When there is little to say, say little

The sections are sized to the season, not to a template. `Inside the Numbers`
aims for ~6 bullets by midseason but should carry only 3&ndash;4 after a game or
two; padding it is what produces the lines this section exists to forbid.
`Recent Headlines` is the opposite case — single-game detail is exactly its
subject matter, so it can run its usual ~6 rows from game 1.

## 9. Always publish

Finish the job. When a change to this site is done, put it on `main` and push
it, so the live GitHub Pages build has it. Do not park work on a side branch and
ask whether to publish, do not open a pull request, and do not ask a second time
for something already agreed. Publishing is the default and needs no approval.

If a change is genuinely risky or ambiguous, ask the one question before doing
the work, then publish the result.

## 10. Pitcher card meta lines

Every card in `Starters` and `Relievers` carries a `starter-meta` line under
the name. It holds roster data, not game data, and the format differs by kind:

| Kind | Format | Example |
|---|---|---|
| Starter | `HAND &middot; AGE/GRADE/RATING (SALARY/YEARS)` | `R &middot; 28/B/83 (4.4/2)` |
| Reliever | `ROLE &middot; HAND &middot; AGE/GRADE/RATING (SALARY/YEARS)` | `SU &middot; R &middot; 29/C/70 (1.9/1)` |

Rules:

- **Role** is the roster role (`CP`, `SU`, `MRP`, `LRP`), not the role the arm
  happened to fill in a game. A reliever who worked the ninth once is still
  `MRP` if that is his listed role. When a pitcher carries an alternate
  position, join it with a slash: `LRP/SP`.
- **Hand** is `L` or `R`, and every pitcher carries one. The notes and the
  roster mark only lefties, as `(L)` after the name (`Hentges (L) pitching`).
  **An unmarked pitcher is right-handed** &mdash; that is the convention, not an
  absence of data, so never leave the hand off because no `(L)` appeared, and
  never carry real-world handedness over a roster that says otherwise. The same
  rule reads the opponents in `Opp Pitching`, whose subtitle states it.
- **Salary** keeps the roster's own units &mdash; `820k` stays `820k`, millions
  are bare numbers (`1.9`, `11.1`). **Years** is the contract length.
- Starters omit only the role; the panel they sit in already says starter.

If a value is genuinely unknown, leave the whole meta line off rather than
invent or half-fill it. A card with no meta renders fine.

Cards in `Starters` and `Relievers` use the pitcher's **full name** on both
faces, not the surname the recaps and tables use. `POG` cards and the split
tiles share the `.starter-name` class and are not affected.

### 10.1 Every pitcher on the staff gets a card

Both panels list the whole staff, not only the arms that have appeared. A
pitcher who has not yet thrown carries his meta line and a single `Has not
pitched` row where his game log would go; his season stats are `0` for the
counting columns and `N/A` for ERA, WHIP, K/9 and BAA, which have no value at
0 IP. The panel subtitle counts both: `2 of 5 used`, `6 of 8 used`.

Cards for arms that have pitched come first, in order of first appearance;
those that have not follow, in roster order.

### 10.2 Decision chips

`.dec` chips on a game line come in `w`, `l`, `nd`, `sv` and `hld`. A starter
always carries one &mdash; `ND` included, since a start with no decision is
still a fact about the start. A reliever carries one only when he earned it
(`W`, `L`, `SV`, `HLD`); an ordinary middle-innings outing gets no chip.


### 10.3 Empty stat values

A rate stat with no value yet &mdash; ERA, WHIP, K/9 or BAA at 0 IP, AVG or OPS
at 0 AB &mdash; reads `N/A`, never a dash. Counting stats in the same row still
read `0`, because zero is the true count. `INF` is not the same thing and stays
as it is: a pitcher charged with earned runs without recording an out has a
genuinely infinite ERA, which is a result rather than a blank.

## 11. Roster page player cards

On the `Roster` page every player's name is clickable and opens a card. What
the card holds depends on the level.

**Big leaguers** get a headshot, a season stat line and a note on how the season
is going. **Minor leaguers** get their affiliate's logo, their roster numbers,
and a line on their 40-man and service status &mdash; no season stats and no
prose about form, because none is tracked below the majors. Never invent one.

The three affiliate logos are Royals-specific and come from
`www.mlbstatic.com/team-logos/{teamId}.svg`:

| League | Affiliate | Team id |
|---|---|---|
| `AAA` | Omaha Storm Chasers | 541 |
| `AA` | Northwest Arkansas Naturals | 1350 |
| `A` | Quad Cities River Bandits (High-A) | 565 |

The game carries one A level; it is mapped to the High-A club. Columbia
Fireflies (3705) is the Low-A affiliate if that ever turns out to be the better
match.

Each logo is defined **once**, as a base64 SVG background on a `.mlogo.lv-*`
class, and every card references the class. Never inline the image per player
&mdash; 64 copies of a 10KB logo would add most of a megabyte for nothing.

Each card carries three things:

1. **A photo**, in a 74px square slot. All 26 carry one, embedded as a base64
   JPEG. Images are always embedded, never hot-linked.

   Source them from MLB's own headshots, not from Wikipedia. Look the player up
   by name at `statsapi.mlb.com/api/v1/people/search?names=...` to get his
   MLBAM id, then fetch
   `img.mlbstatic.com/mlb-photos/image/upload/d_people:generic:headshot:67:current.png,w_426,q_auto:best/v1/people/{id}/headshot/67/current`.
   These are uniform: same framing, same flat grey ground, face centered. To fit
   the square slot, trim the bottom 8% (jersey), scale to fit 148px, and pad to
   148 square with the background colour sampled from a corner pixel &mdash; the
   pad is invisible and nothing gets cropped off the head. Quality 82 lands each
   file near 4KB.

   Wikipedia photos were tried first and are worse in every way: action shots at
   mismatched crops and angles, no coverage for younger players, and a plain
   `Carlos Santana` lookup returns the guitarist. **Look at what you fetched
   before publishing it** &mdash; that is what caught the guitarist.

   A player may appear in another club's cap. That is correct, not a bug: the
   headshot is his current one, and this roster is full of players acquired from
   elsewhere.

2. **A season stat line** &mdash; nine tiles for hitters
   (`G AB R H HR RBI SB AVG OPS`), ten for pitchers
   (`G W-L IP H R ER BB K ERA WHIP`). A player who has not appeared shows `0`
   for counting stats and `N/A` for the rates, exactly as the pitcher cards in
   section 10.1 do.
3. **One or two sentences on how the season is going**, good or bad.

### 11.1 The note is prose, so section 8 governs it

The stat tiles may show `AVG` and `OPS` at any sample, because the Team Batting
table already does. **The note may not.** It is prose and every rule in section
8 applies: no rate stat below the section 8.3 minimums, no extrapolation before
game 25, no walk-total observations, nothing that is not derivable from the
recaps. Through two games that means "four hits in eight at-bats", never
"hitting .500".

Section 8.8 binds these notes too: numerals for stat values, and **every note
names the game and the opponent** &mdash; `in the 9th of G3 vs. ATL`, not "in
the finale" and not a bare `G3`.

The note should say what a beat writer would say about that player right now
&mdash; the shape of his season, not a restatement of the tiles above it. Name
the game and the moment where one exists ("the leadoff double he gave up in the
7th came around to score"), and be as willing to write a bad one as a good one.
A player who has not appeared gets `Has not played.` or `Has not pitched.` and
nothing more.

### 11.2 Player names are clickable wherever they appear

The card opens from the `Roster` table and from every player name on the
`Overview` panel &mdash; both leaders grids and the Team Batting and Team
Pitching tables. Overview uses surnames, so each button still carries the full
`data-p` slug: `<button type="button" class="pname" data-p="kyleisbel">Isbel</button>`.
A `None yet` row is never a button.

In a leaders tile the name keeps the normal ink colour (`.lead .pname { color:
inherit }`) and only turns blue on hover, because the value beside it is already
blue and two blues in one row read as noise. In the tables the name is blue, as
on the Roster page.

**`#pmodal` must sit outside every `.panel`.** Panels are `display: none` when
inactive, so a modal parked inside one opens invisibly from anywhere else. It
belongs between the last panel and the footer.

Markup: a hidden `<div class="pm-body" id="pm-{slug}">` per player, cloned into
`#pmodal` on click; `{slug}` is the full name lowercased with non-alphanumerics
stripped. Closing works on the backdrop, the X, and Escape.

## 12. Schedule roster notes

A game in the `Schedule` panel may carry **pre-game roster notes** &mdash;
lineup and position changes made before first pitch. They are not part of every
game's notes, so most rows never have them.

A game **with** notes becomes a dropdown; a game without one stays a plain
`div.sgame` row. Never emit an empty dropdown.

```html
<details class="sgamed next"><summary class="sgame"><span class="g">G3</span>
  <span class="mu"><span class="loc">@</span> <span class="opp">ATL</span></span>
  <div class="out"><span class="nextchip">Next</span></div>
  <span class="chev">&rsaquo;</span></summary>
  <div class="rnotes"><div class="rn-k">Roster Notes</div>
    <p>Garcia has been moved to CF, with Pasquantino taking 3B and Isbel sitting.</p>
  </div></details>
```

- The `next` and `upcoming` modifiers move from the row to the `details`, which
  now carries the border, background and gold ring. The summary keeps the same
  grid with a fourth column for the chevron, so a row with notes and a row
  without line up exactly.
- **Check the notes against the box score before attaching them to a played
  game.** Notes are written before a game and the lineup can change; if they
  name a player sitting who then took four plate appearances, they belong to the
  next game, not that one. This is how the G2/G3 notes were sorted out.

## 13. Clickable strip tiles

`Sweeps`, `Shutouts`, `Comebacks`, `Most Runs`, `Most Runs Allowed` and `BLL`
each refer to specific games, so each opens the game or games behind it in the
same `#pmodal` used by player cards.

- A tile with **at least one game behind it** is a `<button class="stat"
  data-p="stat-{slug}">` and carries a `&rsaquo;` on its label line. A tile
  sitting at **0** stays a plain `div.stat` &mdash; there is nothing to open, so
  it gets no click affordance.
- The modal body is `<div class="pm-body" id="pm-stat-{slug}">`: a
  `.pm-sthead` with the tile's name and value, one `.gref` row per game, and a
  `.gref-note` saying what earned it ("Atlanta scored 3 in the 7th to turn a
  4-2 deficit into a 5-4 win").
- `.gref` is the game-log row packed left &mdash; `grid-template-columns: auto
  auto auto auto; justify-content: start` &mdash; so the box score sits beside
  the matchup instead of across a gap, and there is no chevron.

The click handler matches `[data-p]`, not `.pname`, so player names and stat
tiles share one path.

## 14. Opponent logo in game rows

Game-log rows carry the opponent's mark **to the right of the abbreviation**:
`<span class="loc">@</span> <span class="opp">ATL</span> <span class="tlogo
lg-ATL"></span>`. The `.lg-*` classes already exist for all 30 clubs, so this
costs nothing.

It is a **51px** mark in the game log (`.matchup .mteam .tlogo`) &mdash; large
enough to read as the club's identity rather than a bullet point. The compact
`.gref` rows inside a stat tile's modal keep a 17px version, since those rows
are meant to be scannable at a glance.

## 15. Next Game tile

The hero carries a `Next` tile under the record, reading left to right: a gold
`Next` chip, the game number, `vs`/`@` plus the abbreviation, the opponent's
54px `.tlogo` **after** the abbreviation, and &mdash; pushed to the right edge
&mdash; the season series record against that club with a `Season series` label
beneath it. As in the game log (section 14), the mark follows the letters.

```html
<div class="nextgame"><span class="ng-k">Next</span><span class="ng-g">G4</span>
  <span class="ng-opp"><span class="loc">vs</span> MIN</span>
  <span class="tlogo lg-MIN" aria-hidden="true"></span>
  <span class="ng-series"><b>0&ndash;0</b><small>Season series</small></span></div>
```

The series record is KC's mark against that opponent this season and must match
the club's row in `vs. Divisions`; before the clubs have met it is `0-0`. The
tile always describes the **next unplayed game**, so it moves on after every
publish and disappears once game 162 is in the book.


## 16. Handedness splits and the challenge log

`Batting Splits` carries **AVG vs RHP** and **AVG vs LHP** tiles. Split KC's
at-bats by the handedness of the pitcher on the mound, which the `Opp Pitching`
table already records &mdash; lefties are marked `(L)`, everyone else is a
righty (section 10). The tile's `st-sub` reads `N for M` and the back lists each
arm faced with KC's line against him, hits first.

The `Challenges` table names **who challenged on the Royals side**, as a `.sub`
line under the record: `Perez 4`, `Santana 2, Lowe 1`. Opponents stay anonymous
by design.

A batter challenge is attributed to the batter named in the recap. A catcher
challenge belongs to whoever was catching, and **Perez is the catcher unless the
pre-game roster notes say otherwise** &mdash; Jensen is the backup and a start
behind the plate for him will be called out in those notes (section 12). Two
catchers appearing in the same lineup does not mean the backup caught; the
other is at DH.


## 17. Masthead

The crest and title are one `<button class="mast-home">` that returns to the
`Overview` panel from anywhere on the site. It carries no visible chrome; the
title turns blue on hover so the target is discoverable.

The stamp on the right is **`Through G{n}` over the publish date**, and both
move with every publish. `Through G1` sat there for three games because it was
never wired to anything &mdash; treat it as part of the regeneration, not
decoration.

## 18. Schedule rows: wordmarks, and home vs away

Each schedule row carries the opponent's **wordmark** behind it and a left
accent bar saying whether the game is home or away.

**Wordmark.** `<span class="wm wm-ATL" aria-hidden="true">` is the first child
of the row, absolutely positioned right, at `opacity: .13`. It is **not an
image** &mdash; it is an alpha mask coloured by CSS (`background-color:
var(--ink)`, `mask-image: var(--wm)`), so it is greyscale by construction and
follows the theme into dark mode without a second asset.

Source them from MLB, the same place as the headshots and cap logos:
`www.mlbstatic.com/team-logos/team-wordmark-on-light/{teamId}.svg`. Render each
to a **72px-tall PNG whose own alpha channel carries the shape** &mdash;
`Image.merge("LA", (white, alpha))`. A CSS mask reads the image's **alpha**, so
writing the shape into a greyscale channel and leaving the image opaque makes
every mask resolve to "show everything" and each row paints a solid grey block.
That is what it looks like when this is wrong.

Two things matter for size:

- The raw SVGs are 377KB and a greyscale raster is little better, because the
  anti-aliased edges do not compress. The alpha mask is ~197KB for all 30.
- Define the data URI **once**, as `--wm` on the `.wm-{ABBR}` class, and read it
  from `.wm` with `mask-image: var(--wm)`. Writing it into both the prefixed and
  unprefixed properties stores every mask twice and costs another 157KB.

KC's abbreviations match MLB's except **`ARI`**, which MLB serves as `AZ`.

**Home and away.** `vs` gives the row `home`, `@` gives it `away`. Home takes a
royal left bar and a faint royal tint; away takes a muted grey bar and no tint.
Rows are `position: relative; overflow: hidden` and every child except `.wm`
gets `z-index: 1` so the wordmark stays behind the text. Row padding is 13px so
the mark has room to read.
