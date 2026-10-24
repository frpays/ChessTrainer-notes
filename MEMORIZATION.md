# Memorization

How the trainer (`rep serve`, see [ARCHITECTURE.md](ARCHITECTURE.md)) decides what to show me, what it
records, and how it estimates what I still remember. Code: `tools/src/repertoire/trainer/app.js` (grading,
sessions, Lines page) and `tools/src/repertoire/trainer.py` (files, settings).

## The unit: one move of mine in one position

A move is identified by the position where I play it and the move itself:

```
rnbqkbnr/pp1ppppp/2p5/8/4P3/8/PPPP1PPP/RNBQKBNR w KQkq - d2d4      (2.d4 in the Caro-Kann)
```

The position is a FEN without move counters (pieces, side to move, castling, en passant), the move is in
UCI. The key does not depend on the line, the chapter or the study: positions reached by several move
orders (transpositions) and moves shared by several studies (1.e4) are one item with one history. Lines
are only the way moves are presented.

## State of a move: Leitner boxes

`data/trainer/progress.json` keeps, for every move I have learned:

| Field | Meaning |
|---|---|
| `box` | 1 to 7, how consolidated the move is |
| `due` | when it comes back for review (ms since 1970, UTC) |
| `lapses` | how many times I missed it |
| `last` | when I last played it correctly or missed it (since 2026-10-23) |

Each box has an interval, the `intervals` setting: 1, 3, 7, 14, 30, 60, 120 days.

- **Found when due**: one box up, due again after the interval of the new box.
- **Missed** (wrong move, Hint, or Analyse while the move is expected): back to box 1, due at once,
  one more lapse.
- **Found before it is due** (a known move replayed for context, or in a test): the box does not change;
  only `last` is updated. A miss still counts.
- **A new move missed in the quiz** stays new.

Before `last` existed, the last review is derived as `due − interval(box)`.

The Leitner system is a heuristic: the intervals are a reasonable but arbitrary choice, and the box is a
coarse summary of a move's history. It decides *when* to review; it does not claim to measure memory.

## Sessions

- **Learn**: new lines, `new_lines` per session (5). Each line is first shown, new moves with arrows, then
  played from memory ("Play from memory"). Lines come skeleton first: in each chapter the main line, then
  the lines that leave it earliest. The learning goal (a slider on the home page) can restrict them to the
  positions club games reach most often.
- **Review**: the lines owning a due move, the most overdue first, `review_lines` per session (20), played
  in tree order so that going from one line to the next is a short way back up the tree. A move shared by
  several lines (1.e4) is reviewed in one of them only, its owner: the first in tree order.
- **Test**: positions drawn at random among the learned moves, set up at once, weighted by how often club
  games reach them, never on the trunk shared by every line of the opening; I play on for `test_moves` of
  my moves (5). A success leaves the schedule as it is; a miss sends the move back to box 1.
- **Error review**: in every mode, a line with a miss comes back once at the end of the session.

A line does not start from move 1: it starts `context` of my moves (2) before the first move to train, or
where it leaves the previous line.

Why (2026-10-22): drilling every line from move 1 (as in Chessable) was tedious, but a pure tree drill
loses the sense of where the line sits; starting a little up the tree re-locates it. Once a line starts I
play all my moves to its end, the known ones included (auto-playing them was disturbing); only the
opponent's moves are played for me. Playing from memory, nothing gives an answer away: the opponent's
`?`/`?!` appear once my answer is played, comments and the line's title only at its end.

## What I still remember: the Lines page

The Lines page estimates, for every study, chapter and line, how likely a game falls into a line I no
longer know, and sorts by it ("Priority first").

**Reach** comes from club games (Lichess explorer, 1800-2200, blitz and rapid): a study's is how often club
games of its colour enter it (`repertoire/starts.json`); a chapter's, its share of the study; a line's,
its share of the chapter.

**Memorized** is the chance that I still know a line, that of its weakest move. A move last played `t` ago
is remembered with probability `exp(−t/S)`; a line not learned yet is at 0. Two models of `S` are kept
side by side, as two sort orders, since they only rank and display (2026-10-24):

| Order | Stability `S` | Setting |
|---|---|---|
| Priority first (τ) | the same for every move | `forgetting_days`, 30 |
| Priority first (boxes) | `interval(box) / −ln R`: the chance to remember a move on its due date is `R` | `due_retention`, 90% |

With boxes, a move just learned (box 1, `S` ≈ 9.5 days) falls to about 50% in a week; a move in box 5
(`S` ≈ 285 days) stays above 90% for a month. With a single τ, a line seen once two hours ago counts as
well learned as one found six times in a row, which is why the box model was added.

**Priority** is reach times non-retention (1 − memorized). A chapter's non-retention is the sum of its
lines' priorities; a study's, the sum of its chapters' priorities. The memorized figure of a study is
divided by the share of its games its chapters keep, so that a study never learned shows 0%.

## Limits

- The exponential is an approximation; on real data, forgetting often fits a power law better.
- `R` = 90% is a target, not a measurement of my memory.
- Club probabilities are not my opponents' choices, though my opponents play at club level.

## The review log, to measure

Since 2026-10-23, `data/trainer/reviews.csv` records every move I am asked for:

```
time,position,move,result,mode,box,line,study
2026-10-23T20:58:06Z,rnbqkbnr/ppp1pppp/8/3p4/4P3/8/PPPP1PPP/RNBQKBNR w KQkq -,e4d5,missed,quiz,,bb803bd8fa40,white/scandinavian
```

`result` is `found`, `missed` or `hint`; `mode` is `context`, `learn`, `quiz`, `review` or `test`; `box`
is the box before the answer (empty for a new move). The progress keeps the state; the log keeps the
history. Appended only; backed up with the progress.

## Next

- After a few weeks of log: success rate by box and by time since the last answer. Check which model fits
  (single τ, boxes, power law), and set `R`, or `S` per box, from the data.
- Then, if worth it, a model of the FSRS kind: a stability per move that grows with each successful
  review, fitted on my history.

## Dropped

- **Priority by frequency drives the sessions** (2026-10-23): Learn ordered across studies by how often
  club games reach a move, reviews the most frequent first, a global learning goal. Rejected: the sessions
  stay as they were; priority only ranks and displays on the Lines page.
- **Weighing studies by my own games** (2026-10-23): rejected; my games built the repertoire, they do not
  weigh it. All probabilities come from club games.
