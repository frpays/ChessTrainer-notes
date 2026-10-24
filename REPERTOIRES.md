# Repertoires

How a repertoire is built from its spec, the rules the builder follows and why, how it is annotated, and the
parts of it published (its study) and trained (its training). Code: `tools/src/repertoire/build.py` (the
builder), `forced.py` (line ends), `style.py` (published look), `cut.py` (study and training), `spec.py`. The
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
  name, also shown in the trainer), `max_ply`,
  `min_reach` (a line is followed while this share of club games reaches it: 5%, 2% for the
  Scandinavian), `min_games`, and switches: `prefer_habits`, `punish_tricks`, `tolerance`, `floor_cp`;
  `study_size` and `training_size` (see below).
- **[policy]**: my imposed moves, as move sequences. Never overridden.
- **[chapters]**: move prefixes and chapter names; the rest goes to "Other lines".
- **[notes]**: plan notes placed on a move; **[intro]**: the Overview chapter; **[exclude]**: lines covered
  elsewhere; **[models]**: model games for the plans.

## Opponent moves

- The club replies (Lichess explorer, 1800-2200, blitz and rapid) above the reach and game thresholds.
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
   - the most **practical** move: how it does in club games and in 2500+ blitz.

A spec move that blocks a transposition is kept; the builder says so.

In the opening, Stockfish's preference within a small margin is not trusted over the natural, common move
that keeps the system (2026-10-23: 4.d4 rather than 4.c4 in the Scandinavian, 8.Bg5 rather than 8.Neg5
after ...Bb7 in the Rubinstein). Such choices go into the spec.

## The main line

At the opponent's nodes the main line goes on with the most played **sound** reply (the best defence if it
is the only one), then come the traps (`?`/`??`), then the pointers (transpositions, other repertoires). A main
line never continues with a blunder (2026-10-23, Bronstein-Larsen 12...Qc7??), and a reply that only
transposes is never the main line, even when it is the most played.

## Where a line ends

- **Always on my move**, the one to memorise; never on the opponent's move with "Next: ..." in a comment.
  Only lines covered in another repertoire may end on the opponent's move.
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
  keep everything).

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
  line. The trainer says "to move N" only for a goal below the training's size.
- **The end of a training line says how the study goes on**, in its last comment, where the study goes
  further: "In the study: 7.Nf3 (7.Nb3 Be7 =) (7.Nf5 d5 =) 7...Be7 =", the study's lines in full up to
  three variations, else its main line and the first move of the others. Only the trainer shows it.
- **Stable**: a cut and the trainer compare the same rounded numbers (`reach.DIGITS`), so a position at the
  limit is in both or in neither, and `rep cut` run twice writes the same files (2026-10-24: a rounding
  mismatch had cut 8...Nc6 in the Rubinstein while counting it as known).
- **Growing keeps my choices**: raising a size brings lines of the repertoire back as they are, without a
  draft. A deeper draft of the repertoire makes a cut of the same size a little larger: it is a share of a
  larger tree.
- Najdorf (2026-10-24): at 80%, 50 of 177 lines and 120 of my 468 moves, the positions reached by 0.46% of
  club games or more; at 90%, 69 lines and 184 moves.

## Publishing and freezing

- A push publishes the study, the repertoire cut to `study_size`; a chapter the cut leaves out stays online
  until a push with `--prune`.

- `rep edits` before every push: it lists what was changed online since the last push. A push overwrites
  online changes (a backup goes to `data/backups/`): explorations on the Lichess board are throwaway. To add
  a line, play it on Lichess, say so, and it goes into the spec.
- **Freeze** (`rep freeze`) once a study is validated: its moves as published go to the lock and later
  drafts keep them; the moves a cut leaves out stay free, I have not reviewed them. Repertoires under review
  stay unfrozen.
- Regenerating an unfrozen, published repertoire moves many of my moves (the caches hold deeper evals by
  then): write a temporary lock from the published version, draft, then delete the lock.
