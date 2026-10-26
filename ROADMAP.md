# Roadmap

The order of the work, agreed 2026-10-26, when I decided to redo every repertoire with a new builder (see
[LAB.md](LAB.md)). A phase starts when the one before is done. The current repertoires stay in the trainer
until their replacements are validated.

## 0. Before redoing anything (first, 2026-10-26)

- **Back up the Mac's `data/`** to the GCS bucket: the caches (hours of Stockfish and explorer queries), the
  games and the lab's results, not backed up since 2026-10-22. Never the Mac's `data/trainer`: robin's
  progress is the real one, already pushed every 30 minutes ([ARCHITECTURE.md](ARCHITECTURE.md)).
- **`rep push` keeps the state of `studies.json` when a push fails halfway**: every study will be pushed again.
- **The club band**: the explorer's `1800,2000,2200` covers 1800 to 2499, each Lichess bucket running to the
  next one. Kept (2026-10-26); the notes and the code that say 1800-2200 are corrected.

## 1. The practical-model player (lab)

The fit of [LAB.md](LAB.md), "The practical-model player", in `lab/practical.py`: p', the arithmetic or the
geometric form, K1, K2, the temperature and N, by maximum likelihood on the caches (no Lichess query).

## 2. Where to deviate (lab)

The player and the constraints in `lab/scout.py`: at most δ = 8 points below the best move, a floor of 45%
(about 40% as Black) at depth 20 or more, a horizon of five of my moves then three of the model's. Accepted
when, on the test zone, it finds by itself 7.N1e2 and 7.Nh3 with the Nh5 cluster (Caro-Kann 6.h4 h6), 6.Neg5
(Fort Knox) and 8.Neg5 (Rubinstein 7...b6).

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
Alekhine). Each one: draft, my review, freeze, push, trainer, with the list of the learned moves that change.
An unchanged move keeps its progress, which is kept by position and move ([MEMORIZATION.md](MEMORIZATION.md)).

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

## Open

- During the redesign, keep learning new lines of the current repertoires, or only review them? A move
  learned now may change.
- The drafts never committed (Italian, Benoni, White Najdorf, the Scandinavian's changes): their specs as a
  starting point and their trees dropped?
