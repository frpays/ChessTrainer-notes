# Roadmap

The order of the work, agreed 2026-10-26, when I decided to redo every repertoire with a new builder (see
[LAB.md](LAB.md)). A phase starts when the one before is done. I do not train during the redesign: I work on
it (2026-10-26). The current repertoires stay in the trainer until their replacements are validated.

## 0. Before redoing anything (first, 2026-10-26)

- **Back up the Mac's `data/`** to the GCS bucket: the caches (hours of Stockfish and explorer queries), the
  games and the lab's results. Done 2026-10-26, `tools/data-sync push` of each folder but `data/trainer`:
  never the Mac's, since robin's progress is the real one, already pushed every 30 minutes
  ([ARCHITECTURE.md](ARCHITECTURE.md)).
- **`rep push` keeps the state of `studies.json` when a push fails halfway**, since every study will be
  pushed again. Done 2026-10-26 (`StudyPush`, [REPERTOIRES.md](REPERTOIRES.md)).
- **The club band**: the explorer's `1800,2000,2200` covers 1800 to 2499, each Lichess bucket running to the
  next one. Kept (2026-10-26); the notes and the code that said 1800-2200 are corrected.

## 1. The practical-model player (lab)

The fit of [LAB.md](LAB.md), "The practical-model player", in `lab/player.py`: p', the arithmetic or the
geometric form, K1, K2, the temperature and N, by maximum likelihood on the caches. One exception: the
explorer was asked for the 1,940 positions the cache lacked on the way, where the spec set my move
(2026-10-26, "The gaps" in LAB.md).

Fit done 2026-10-26 ([LAB.md](LAB.md), "The fit"), on 16,085 positions once the policy and Stockfish were
computed for the 13,866 that lacked them (Stockfish on a spot VM): the arithmetic form, a constant α of
0.26 (p changes nothing), N = 6.2 games.

## 2. Where to deviate (lab)

At the decision points of the backbone, by the club's own scores (2026-10-27, agreed; [LAB.md](LAB.md),
"The backbone and its decision points", "The club's verdict first"):

- **The backbone**, from a start: the opponent's every club move whose position keeps n/Nt at or above a
  threshold (3e-5 by default), one move of mine at each of my positions (given in the spec, else the masters',
  2500+ blitz's, or the club's most played). Moves given in advance shrink the decision points (2026-10-26).
- **At each decision point**, my candidate moves by their prepared value (2026-10-27, [LAB.md](LAB.md), "The
  prepared value"): my moves Stockfish's best, the opponent's as the club plays them (the model where it has
  few games), five moves of mine, Stockfish at the leaves; Stockfish's floor kept (45% as White, 40% as Black,
  at depth 20). The club's raw score is the club against the club and underrates the sharp lines a prepared
  player wants (6.e5 in the Pin Variation): it finds the decision points and the opponent's moves, it does not
  judge mine.
- **Early deviations first** (2026-10-27, my decision): each further move of the main line gives the opponent
  the chance to deviate first, and a deviation's club score already counts every reply after it. Valuing the
  main move by the gains found below it ("nested points") is kept in mind, not adopted.

`lab/screen.py` finds the decision points (the Caro-Kann after 3.Nc3, 54 in 10 minutes; the Sicilian with
2...e6, 81 in 27) and ranks my moves by the club's raw score: it found the three deviations of the test zone
(Caro-Kann 6.h4 h6 7.Nh3 and 7.N1e2, Fort Knox 5...Bc6 6.Neg5, Rubinstein 7...b6 8.Neg5). Next, the prepared
value in the lab (a prototype gives 6.e5 twelve points over 6.Bd3 in the Pin Variation, two to three minutes a
candidate), its breadth measured, the candidates sorted first; the acceptance criterion again; the reach along
the tree instead of n over the start's games (the explorer counts the games of every move order).

The scout of `lab/scout.py` (the expectimax against the practical-model player, three moves of mine tried at
each of my positions) did not finish a candidate in 12 hours on a spot VM (2026-10-26); the prepared value is
its principle at one move of mine, the club as the opponent.

## 3. Which opponent moves, and where lines end (lab)

After each move chosen, the lines (2026-10-27, my proposals; [LAB.md](LAB.md), "Lc0 where the club runs out"):

- **The opponent's replies kept**: those the club plays enough, and the mistakes Lc0 foresees: a reply natural
  for its policy that Stockfish finds losing (10 points or more, Lichess's mistake; the trap of the intuition
  index), weighed by its probability times what punishing it gains. The club measures the mistakes that are
  played, Lc0 foresees the others, deep in the lines where the games run out.
- **My moves there**: the club's best where it has measured them, else Stockfish's best above the floor; at
  equal value the more natural (the policy), easier to remember.
- **Where a line ends**: it goes on while keeping the advantage needs a move of mine Lc0 finds unnatural
  (Stockfish's best low in my policy, my natural moves losing much of the advantage), and ends when my natural
  moves keep it. The depth follows the difficulty, and these hidden moves are the ones to learn. How much loss
  is worth one more move: its reach times the loss avoided, against the cost of a move to learn (not measured
  yet).

Tried first on the Pin Variation of the Sicilian with 2...e6 (5...Bb4 6.e5), forcing and counter-intuitive
on both sides (2026-10-27, my choice). Then a whole Caro-Kann and French Rubinstein drafted by the lab,
compared with mine.

## 4. The new builder in `rep`

Classes, `Parameters`, tests. It replaces the rules that grew case by case (the habit margin, the
transposition priority, the temporary lock). It works on a whole subtree from a start, with the moves the spec
gives: a first pass for the backbone and my move at each decision point (phase 2), a second for the lines after
each (phase 3), the model (Lc0's policy, Stockfish) where the club lacks games.

The target: under 10 hours of caching per repertoire, the Caro-Kann after 3.Nc3 at 3e-5 in under three, so
that the Najdorf as Black, the hardest, fits. The prepared value at every candidate would take some 25 hours
on the Mac: the candidates sorted first, a few per point, and the parallel search planned on 2026-10-26
([LAB.md](LAB.md), "The parallel search"): a pool of single-threaded Stockfish, a worker on a spot VM over
ssh, the builder and Lc0 on the Mac.

The moves are marked as good too (2026-10-27, my decision; [LAB.md](LAB.md), "Good moves"): □ an only move,
! a good move intuition does not find, !! a hidden one, !? my deviations; the mistakes as before. Few marks,
on one score: what intuition loses at the position, its levels from its distribution over a repertoire.

## 5. Redoing the repertoires, one at a time

The test zone first (Caro-Kann, French 3.Nc3), then the key choices of [LAB.md](LAB.md) (2.Bc4 against the
Petrov, the Open Sicilian, the Scandinavian with 3.Nf3; as Black 2.Nf3 d6 and the Najdorf), then where I
play anything (as Black against the English and the London; as White against the Pirc, the Modern, the
Alekhine). Each one: draft, my review, freeze, push, trainer.

The repertoires and studies made so far, the drafts never committed with them, are backed up and then
deleted as the new ones replace them (2026-10-26). The new ones need not resemble them: the Caro-Kann and the
French Rubinstein will probably come out close, and if not, so be it; I spent little time on the Sicilian
with 2...e6, and the rest may change freely.

## 6. The trainer

- My own positions with a low instinct first, the moves intuition will not find (the hidden moves of phase 3):
  a priority shown on the Lines page, not imposed.
- From mid-November, the memory models fitted on the review log ([MEMORIZATION.md](MEMORIZATION.md), "Next").
- The Lines page counts games twice at a transposition (display only).

## Along the way

- The golden comparison moves into `tools/`: every `rep` command run on copies of the repertoire
  repository, each run from its own copy of the caches. Two runs from different cache states differ
  (Stockfish on 6 threads, a deeper eval replacing one a run already used); from the same copy, the
  restructured code matched the original on every output (2026-10-26).
- A lab module moves to classes when it is touched.
- No variables at module level (2026-10-26, `tools/CLAUDE.md`): done in the package and in the lab
  modules of the fit and the scout; `tests/test_layout.py` lists the lab modules still to convert.
