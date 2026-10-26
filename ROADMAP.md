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

The player and the constraints in `lab/scout.py`: a floor of 45% (about 40% as Black) at depth 20 or more,
a horizon of five of my moves then three of the model's. My candidates are the policy's likeliest moves (up
to 95%) and Stockfish's best; at equal value (within a point) the more natural one, easier to remember and
the one I play if I forget (2026-10-26). How principled the repertoire is, is ζ: the opponent plays
Stockfish's best with probability ζ on each move, 0 by default (2026-10-26, [LAB.md](LAB.md), "ζ"); no
bound on what a move concedes, the floor alone ("The floor alone").

Accepted when, in three positions of the test zone, the scout given no candidate ranks first 7.N1e2 and 7.Nh3
with the Nh5 cluster (Caro-Kann 6.h4 h6), 6.Neg5 (Fort Knox 5...Bc6) and 8.Neg5 (Rubinstein 7...b6), or ranks
above them a move that holds up when I examine it (a better deviation is a find, not a failure). First run
on a spot VM, 2026-10-26: 20 candidates in the three positions at the full horizon, before ζ and the evals at
fixed depths ([LAB.md](LAB.md), "Evals at fixed depths"); then the head of each position again at fixed depths,
ζ 0 and 0.2. Then the scan: my positions along the test zone's lines, a short horizon to sort the deviations,
the full horizon for the best, ranked by their gain times my p.

## 3. Which opponent moves, and where lines end (lab)

The second question of [LAB.md](LAB.md): the opponent moves a repertoire keeps. A line ends in a calm
position: instinct 0.95 or more for both sides, a level eval. Then a whole Caro-Kann and French Rubinstein
drafted by the lab, compared with mine.

## 4. The new builder in `rep`

Classes, `Parameters`, tests. It replaces the rules that grew case by case (the habit margin, the
transposition priority, the temporary lock). It works on a whole subtree (the Caro-Kann after 1.e4 c6): at
each of my positions it decides my move, the main line's or a deviation there, with the full horizon from that
position (a receding horizon: near the horizon the scout's later moves are decided short-sighted; the searches
overlap and share their evals through the caches); at each of the opponent's, the replies kept and where the
line ends (phase 3).

Its first pass finds the backbone and its decision points exactly (2026-10-26, agreed; [LAB.md](LAB.md), "The
backbone"): my moves fixed, the opponent's club moves above a threshold in n/Nt. The deviations are searched at
the decision points only, the rest of the subtree cheaply. Fixing moves first shrinks the count, and the
threshold can then go lower. The target: the Caro-Kann after 3.Nc3 at 3e-5 (some 60 decision points) cached in
under three hours on the VM, so that the Najdorf as Black, the hardest, fits in ten. At the estimated 35,000
nodes a point and a million a night on 32 vCPUs, that is about a factor 7: a larger spot VM for the same
vCPU-hours (once the search runs in parallel, the time divides by the vCPUs, about 3.5 from 32 to 112), and
cheaper nodes, each lever measured against the scout's value as `branch` was (depth 12 past my horizon, fewer
judged moves, a short horizon to sort the candidates before the full one), and the searches of a decision point
sharing the subtrees of the next ones. The first VM run measures the nodes per candidate first.

The risk is the cost: hours per candidate move at the full horizon on the Mac with the practical-model player
(the ten minutes of the first prototype are past), Stockfish taking 94% of the CPU, Python 4.5% and Lc0 2.4%
(the scouts on the VM, 2026-10-26). The plan (2026-10-26, agreed), after the first results of the scout:

1. **The search in two passes.** The tree's shape depends only on each position's evals (the moves judged, the
   replies followed, my moves above the floor), not on the values backed up. So: an expansion level by level,
   every search of a level sent at once (the policies, then the MultiPV of the judged moves at depth 14, then
   the floors of my moves at 20: some 16 levels of 3 stages, hundreds of searches each), then the values backed
   up in Python from the caches. A position is judged at a canonical p, so the result does not depend on the
   order. Checked against the present search on 7.N1e2 at a short horizon.
2. **A local pool** of single-threaded Stockfish: four on the Mac give more than one on four threads at depth 14.
3. **A worker on a spot VM**: one ssh connection, JSON lines both ways (a position, its moves, the depth; the
   evals back), a single-threaded Stockfish on every vCPU. The latency counts once a batch, not once a search.
   The builder stays on the Mac with its caches: no merge, a preemption loses only the searches in flight; the
   Mac kept awake (`caffeinate`), the connection resumed and the requests sent again. Lc0 stays in the builder
   on the Mac (Metal): the policy comes first at each position, costs little, and its cache is there; every new
   policy from one backend. ssh gives encryption and authentication with no port opened: no HTTP service, no
   queue.
4. **Then Python is the bottleneck**: about 1.3 cores for 30 engines, in one process. SQLite writes in one
   transaction per batch, lighter handling of moves, a level's expansion over two processes if needed; Lc0 in
   true batches if it ever limits (through UCI, one position per call; its Python bindings batch, 17 times
   faster on Metal: [LAB.md](LAB.md), "Engines").

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
- No variables at module level (2026-10-26, `tools/CLAUDE.md`): done in the package and in the lab
  modules of the fit and the scout; `tests/test_layout.py` lists the lab modules still to convert.
