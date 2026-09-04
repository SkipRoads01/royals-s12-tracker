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
Numbers` and `Game Log`, subtitle `current form`. It covers **player-level
recent form** — active hitting streaks, streaks just snapped, short hot/cold
runs, recent baserunning. Keep it distinct from `Inside the Numbers`, which
stays **team-level season context** (records, paces, rate stats). Do not repeat
a fact in both; in particular, hitting-streak lines belong here, not there.

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
- **Use the sport's own idiom** where it is the plainer wording &mdash; went
  down on strikes, the other way, singled home, first one to get away, on the
  year. Avoid inventing color that the recaps do not support; section 8.5 still
  governs the facts.

### 8.8 When there is little to say, say little

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

### 10.1 Every pitcher on the staff gets a card

Both panels list the whole staff, not only the arms that have appeared. A
pitcher who has not yet thrown carries his meta line and a single `Has not
pitched` row where his game log would go; his season stats are `0` for the
counting columns and `&mdash;` for ERA, WHIP, K/9 and BAA, which have no value
at 0 IP. The panel subtitle counts both: `2 of 5 used`, `6 of 8 used`.

Cards for arms that have pitched come first, in order of first appearance;
those that have not follow, in roster order.

### 10.2 Decision chips

`.dec` chips on a game line come in `w`, `l`, `nd`, `sv` and `hld`. A starter
always carries one &mdash; `ND` included, since a start with no decision is
still a fact about the start. A reliever carries one only when he earned it
(`W`, `L`, `SV`, `HLD`); an ordinary middle-innings outing gets no chip.

## 11. Roster page player cards

On the `Roster` page, every player whose `League` is `MLB` has a clickable name
that opens a season card. Minor leaguers stay as plain text &mdash; the card is
about how a season is going, and theirs is not being tracked here.

Each card carries three things:

1. **A photo**, in a 74px square slot. 22 of the 26 carry one, embedded as a
   base64 JPEG (148px square, quality 80, ~6KB each) and cropped to fill via
   `object-fit: cover`. Images are always embedded, never hot-linked. A player
   with no free photo keeps the KC mark at half opacity rather than a broken
   image or a blank box.

   Photos come from Wikimedia Commons via the Wikipedia REST summary API, and
   the footer credits them. Two cautions learned the hard way: **disambiguate
   the title** &mdash; a plain `Carlos Santana` lookup returns the guitarist, so
   query `Carlos Santana (baseball)` &mdash; and **look at what came back**
   before publishing it. Crop square from the top of a portrait (the head sits
   in the upper third), centered on a landscape.
2. **A season stat line** &mdash; nine tiles for hitters
   (`G AB R H HR RBI SB AVG OPS`), ten for pitchers
   (`G W-L IP H R ER BB K ERA WHIP`). A player who has not appeared shows `0`
   for counting stats and `&mdash;` for the rates, exactly as the pitcher cards
   in section 10.1 do.
3. **One or two sentences on how the season is going**, good or bad.

### 11.1 The note is prose, so section 8 governs it

The stat tiles may show `AVG` and `OPS` at any sample, because the Team Batting
table already does. **The note may not.** It is prose and every rule in section
8 applies: no rate stat below the section 8.3 minimums, no extrapolation before
game 25, no walk-total observations, nothing that is not derivable from the
recaps. Through two games that means "four hits in eight at-bats", never
"hitting .500".

The note should say what a beat writer would say about that player right now
&mdash; the shape of his season, not a restatement of the tiles above it. Name
the game and the moment where one exists ("the leadoff double he gave up in the
7th came around to score"), and be as willing to write a bad one as a good one.
A player who has not appeared gets `Has not played.` or `Has not pitched.` and
nothing more.

Markup: a hidden `<div class="pm-body" id="pm-{slug}">` per player, cloned into
`#pmodal` on click; `{slug}` is the full name lowercased with non-alphanumerics
stripped. Closing works on the backdrop, the X, and Escape.
