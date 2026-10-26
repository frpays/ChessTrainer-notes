# Repertoires

How a repertoire is built from its spec, the rules the builder follows and why, how it is annotated, and the
parts of it published (its study) and trained (its training). Code: `tools/src/repertoire/build.py` (the
builder), `draft.py` (the draft), `forced.py` (line ends), `style.py` (published look), `cut.py` (study and
training), `spec.py`. The
commands are in [ARCHITECTURE.md](ARCHITECTURE.md).

## Three words

Three things were all called "study"; since 2026-10-24 each has its word, in the code and the notes:

- **Repertoire**: a spec (`<name>.spec`), what the builder drafts from it (`<name>.pgn`, every line it
  followed) and my frozen moves (`<name>.lock`). The reference a new draft starts from.
- **Study**: the part of a repertoire published on Lichess, `<name>.study.pgn` (`study_size`).
- **Training**: the part the trainer serves, `<name>.training.pgn` (`training_size`, never larger than the
  study).

A cut that keeps the whole repertoire is not written: the trainer then serves the study, or the repertoire.

## Winning chances, not centipawns

Every comparison is made in winning chances, with the Lichess curve
`win% = 50 + 50·(2 / (1 + exp(−0.00368208·cp)) − 1)`. Two evals are compared as `Δp = p2 − p1`, never as
`cp2 − cp1`; to speak in centipawns, `Δp` is converted back to the eval that gives `50 + Δp` ("neutral
cp"). Three thresholds, in points of winning chances: inaccuracy 5 (the tolerance), mistake 10, blunder 15.
Rules are general, not tuned case by case.

## The spec

`repertoire/<colour>/<name>.spec`:

- **Header**: `color`, `start` (the moves to the repertoire's first position), `title` (the Lichess study's
  name, also shown in the trainer), `max_ply` (the depth: lines end there or once the game is decided),
  `min_reach` (a reply is followed while this share of club games reaches it: 5%, 2% for the
  Scandinavian), and switches: `prefer_habits`, `punish_tricks`, `tolerance`, `floor_cp`;
  `study_size` and `training_size` (see below).
- **[policy]**: my imposed moves, as move sequences. Never overridden; the opponent moves on the way are
  always in the tree, played in club games or not.
- **[chapters]**: move prefixes and chapter names; the rest goes to "Other lines".
- **[notes]**: plan notes placed on a move; **[intro]**: the Overview chapter; **[exclude]**: lines covered
  elsewhere; **[models]**: model games for the plans.

## Opponent moves

- The club replies (Lichess explorer, 1800-2499, blitz and rapid): at least 5% of the position's games and
  `min_reach` of the chapter's, or the main reply, which is always followed.
- **Shares are estimates** (2026-10-24): a reply's club games plus 50 games split equally between the replies
  the engine finds sound (within 5 points of its best), over the position's games plus 50. Where club games
  are plentiful little moves (at 2,000 games, a share by 2.5 points at most); after a rare move of mine they
  thin out and the engine fills in, down to a position no club game reached, played on with the engine's
  reply ("club 34%, unplayed"). The comments carry these shares, so the trainer, the cuts and the builder
  count the same games.
- **No line ends because club games are few** (2026-10-24): `min_games` is gone. It stopped half the lines
  (500 games; 2,000 for the Rubinstein, by default), at move 7 or 8, and dropped policy lines below it
  (Caro-Kann 11.Qh4); my own rare move was enough to end a line. `max_ply` and a decided game end them now.
- The **best defence**, even if rare.
- **Traps** (`punish_tricks`): dubious replies that still score well for the opponent, followed down to 1%
  of games, and "dangerous" replies against which I score far below average.
- Mistakes are marked `?`/`??` with their refutation. Several mistakes refuted the same way are generic:
  one example and the safe move, never the whole list.

## My moves

At each of my positions, in order:

1. the spec's move;
2. the frozen move (`.lock`), unless a sound move transposes more directly into the repertoire;
3. otherwise a **sound** move (within the tolerance of the best, and not below `floor_cp`), preferring:
   - one that **transposes** into a position the repertoire already covers, at once or after the opponent's most
     played reply, so that no position is learned by two roads. `rep draft` builds twice so that the second
     pass knows the whole tree. No exception for a small material gain: a pawn worth 17 cp in blitz is
     initiative for the other side (2026-10-23);
   - **my habit** (`prefer_habits`, at least 3 games of mine), so that a sound move I already play stays;
     among moves that transpose as directly, it comes first too (2026-10-24: in the Najdorf, 6.f3 e5 7.Nb3
     Be6 had replaced Be7, my move in 83 games, because both transpose after 8.Be3 and the tie went to the
     practical value);
   - the most **practical** move: how it does in club games and in 2500+ blitz.

A spec move that blocks a transposition is kept; the builder says so.

In the opening, Stockfish's preference within a small margin is not trusted over the natural, common move
that keeps the system (2026-10-23: 4.d4 rather than 4.c4 in the Scandinavian, 8.Bg5 rather than 8.Neg5
after ...Bb7 in the Rubinstein). Such choices go into the spec.

## The main line

At the opponent's nodes the main line goes on with the most played reply that is not a mistake (the best
defence if it is the only one), then come the traps (`?`/`??`), then the pointers (transpositions, other
repertoires). A main line never continues with a blunder (2026-10-23, Bronstein-Larsen 12...Qc7??), and a reply
that only transposes is never the main line, even when it is the most played. An inaccuracy (5 to 10 points)
club players often choose stays the main line: it is the one met over the board (2026-10-24, kept after the
Codex review rather than reordering every repertoire).

## Where a line ends

- **Always on my move**, the one to memorise; never on the opponent's move with "Next: ..." in a comment.
  Only lines that go on elsewhere may end on the opponent's move: in another repertoire, or in another
  chapter ("has its own chapter").
- **Never in the middle of an exchange**: at the end of a line, the opponent's best reply is played on, with
  my answer, while it is a capture or a check that threatens something. A quiet threat counts when passing
  would cost a mistake's worth; a check, when my second best answer would (2026-10-23: spite checks such as
  8...Bb4+ 9.c3 are no longer shown).
- **Stop once the result is clear**: no moves after a won position just to show it.

Refutations show the engine's line until a material gain is visible (at most six plies), ending on my
move. Known weakness (2026-10-23): the opponent's moves there follow Stockfish's principal variation, which
can be unnatural (7...Bb4+ 8.c3 Nxd4); choosing instead the most played reply among those within the
tolerance was proposed and set aside.

## Annotations

- Ideas and plans in words, a few moves on the board; no long variations in comments.
- Speak like a player: "wins a piece", not "wins a bishop" after exchanges; claim only gains visible in the
  moves shown. Only mistakes are marked; no "!" on an obvious recapture.
- Published studies carry no statistics and no centipawns: the standard symbols (=, ⩲, ±, +−) on the last
  move of a line, and words such as "Mating attack" (`repertoire/style.yaml`, applied at push; the PGN files
  keep everything). Notes written in the spec say it in words too: "a common accident in club games", not
  "White scores 76%" (2026-10-24: five notes of the Najdorf and the other second moves still had figures).

## Sizes: the study and the training

The repertoire (`<name>.pgn`) is everything drafted, about the size of an advanced club player's
repertoire: too much to learn at once (2026-10-24). Two sizes cut it, set per repertoire in its spec, with
defaults in `repertoire/style.yaml` (90 and 80): `study_size`, its study, published on Lichess
(`<name>.study.pgn`), and `training_size`, its training, served by the trainer (`<name>.training.pgn`),
never larger. A size is the share of the repertoire's club games in which I know all my moves, the scale of
the trainer's learning goal. `rep cut` writes the cuts; `rep draft` and `rep refresh` run it after them.

- **What a cut keeps**: the positions club games reach at least L times and the moves on the way to them,
  with the highest L that brings the repertoire to its size. A position's reach counts every move order: the
  lines that transpose into it bring their games, so a frequent position stays even when the line that
  develops it is rare (25 of the 102 transpositions had a pointing line more played than the developed one).
  What a node needs (the highest reach below it) never grows along a line, so a cut is a clean subtree.
- **It does not show**: a cut line ends on my move, plays on through a forced sequence like any line end
  (with the repertoire's moves) and gets its eval, so on Lichess and in the trainer it looks like a whole
  line. The trainer says "to move N" only for a goal below the training's size. The forced reply counts as
  played (its club share stays out), so that the move after it is learned with the line and a goal never
  stops in the middle of an exchange; the cost, a coverage a little too low while that move is unknown, is
  accepted (2026-10-24, Codex review).
- **The end of a training line says how the study goes on**, in its last comment, where the study goes
  further: "7.Nf3 (7.Nb3 Be7 =) (7.Nf5 d5 =) 7...Be7 =", the study's lines in full up to three
  variations, else its main line and the first move of the others. Only the trainer shows it. The label
  "In the study:" went (2026-10-24, my choice): the moves alone.
- **A move that sets traps names them** (2026-10-24, 8.Neg5 in the Rubinstein): where the training leaves
  out opponent mistakes (?, ??) the study refutes, my move before them gets them in its comment, each with
  the study's main line: "(8...h6? 9.Nxe6 fxe6 10.Bg6+ Kf8 11.Qxe6 +−)". They are seen, not learned; at
  the first count, 47 of my moves over all the trainings.
- **Stable**: a cut and the trainer compare the same rounded numbers (`reach.DIGITS`), so a position at the
  limit is in both or in neither, and `rep cut` run twice writes the same files (2026-10-24: a rounding
  mismatch had cut 8...Nc6 in the Rubinstein while counting it as known).
- **Growing keeps my choices**: raising a size brings lines of the repertoire back as they are, without a
  draft. A deeper draft of the repertoire makes a cut of the same size a little larger: it is a share of a
  larger tree.
- Najdorf (2026-10-24, redrafted free with the depth rule): at 80%, 50 of 219 lines and 124 of my 703
  moves, the positions reached by 0.43% of club games or more; at 90%, 75 lines and 206 moves.
- Caro-Kann (2026-10-24, my choice): study 100 (the whole repertoire, 171 lines and 440 moves of mine, on
  Lichess) and training 90 (71 lines, 191 moves, positions reached by 0.21% of club games or more). Going
  from 80 to 90 kept every move of the training and added 71.
- Rubinstein (2026-10-24, my choice): study 100 (89 lines, 207 moves of mine) and training 90 (36 lines, 102
  moves), so that the three replies to 7...b6 8.Neg5 that lose (8...Bb7?? 9.Nxf7, 8...O-O? 9.Nxh7, 8...h6??
  9.Nxe6), now in the policy, are trained: they are the reason for 8.Neg5.
- Kan-Taimanov (2026-10-24, my choice): study 100 (160 lines, 376 moves of mine), training 80 (44 lines,
  111 moves).

## Publishing and freezing

- A push publishes the study, the repertoire cut to `study_size`; a chapter the cut leaves out stays online
  until a push with `--prune`.

- `rep edits` before every push: it lists what was changed online since the last push. A push overwrites
  online changes (a backup goes to `data/backups/`): explorations on the Lichess board are throwaway. To add
  a line, play it on Lichess, say so, and it goes into the spec.
- **Freeze** (`rep freeze`) once a study is validated: its moves as published go to the lock and later
  drafts keep them; the moves a cut leaves out stay free, I have not reviewed them. Repertoires under review
  stay unfrozen.
- An unfrozen repertoire is redrafted free (2026-10-24, my choice): no temporary lock from the published
  version, only the spec and my own games (`prefer_habits`). Many of my moves can change (the caches hold
  deeper evals by then); the moves that changed are listed, with those I learned in the trainer, and I
  validate them before anything is published.
- `rep draft --no-lock` leaves the lock aside: every move not in the spec is chosen by the rules, which
  checks them on a frozen repertoire (2026-10-24, the Caro-Kann and the Rubinstein). The lock stays as it was
  until the next freeze.
