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
| **Pace** | projected 162-game record at the current win rate |
| **BLL** | Blown Lead Losses (see definition below) |

Tile markup pattern:
`<div class="stat"><div class="k">LABEL</div><div class="v num">VALUE</div></div>`

**Pace** = project the current record over 162 games: `wins = round(winPct * 162)`,
`losses = 162 - wins`, formatted `W-L` (e.g. 12-0 → `162-0`, an 8-4 start → `108-54`).

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
