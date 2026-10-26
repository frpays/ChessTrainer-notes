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

The player and the constraints in `lab/scout.py`: at most δ = 8 points below the best move, a floor of 45%
(about 40% as Black) at depth 20 or more, a horizon of five of my moves then three of the model's. My
candidates are the policy's likeliest moves (up to 95%) and Stockfish's best; at equal value (within a
point) the more natural one, easier to remember and the one I play if I forget (2026-10-26).

Accepted when, in three positions of the test zone, the scout given no candidate ranks first 7.N1e2 and 7.Nh3
with the Nh5 cluster (Caro-Kann 6.h4 h6), 6.Neg5 (Fort Knox 5...Bc6) and 8.Neg5 (Rubinstein 7...b6), or ranks
above them a move that holds up when I examine it (a better deviation is a find, not a failure). Then the
scan: my positions along the test zone's lines, the deviations ranked by their gain times my p.

## 3. Which opponent moves, and where lines end (lab)

The second question of [LAB.md](LAB.md): the opponent moves a repertoire keeps. A line ends in a calm
position: instinct 0.95 or more for both sides, a level eval. Then a whole Caro-Kann and French Rubinstein
drafted by the lab, compared with mine.

## 4. The new builder in `rep`

Classes, `Parameters`, tests. It replaces the rules that grew case by case (the habit margin, the
transposition priority, the temporary lock). The risk is the cost, about ten minutes per candidate move at
k = 5 on the M2: the search must share its subtrees; if that is not enough, Lc0 on a GPU spot VM
([ARCHITECTURE.md](ARCHITECTURE.md), "Later").

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

- My own positions with a low instinct first, the moves intuition will not find: a priority shown on the
  Lines page, not imposed.
- From mid-November, the memory models fitted on the review log ([MEMORIZATION.md](MEMORIZATION.md), "Next").
- The Lines page counts games twice at a transposition (display only).

## Along the way

- The golden comparison moves into `tools/`: every `rep` command run on copies of the repertoire
  repository, each run from its own copy of the caches. Two runs from different cache states differ
  (Stockfish on 6 threads, a deeper eval replacing one a run already used); from the same copy, the
  restructured code matched the original on every output (2026-10-26).
- A lab module moves to classes when it is touched.
