# Lab: toward the next builder

Started 2026-10-25, when I asked to rethink the builder from the start: the rules had grown case by case
(a habit margin of one point nobody had chosen, a transposition priority, a temporary lock), and the
builder chooses each move locally, so it cannot find what my best repertoires are built on. The lab
(`tools/lab/`, outside the `rep` package) holds instrumented sketches; nothing in it changes a repertoire.

## What a repertoire should be (2026-10-25, my words)

1. **Key choices**: a short catalog of main openings playable at a high level, mostly already in my games
   (Caro-Kann with 3.Nc3 dxe4 4.Nxe4, 1.e4 e5 2.Bc4 against the Petrov, the Open Sicilian, the
   Scandinavian with 3.Nf3; as Black the Sicilian with 2.Nf3 d6 and the Najdorf). My games are not to be
   over-read: where I play anything (as Black against the English and the London, as White against the
   Pirc, the Modern, the Alekhine), start from scratch.
2. **Main body**: a few moves of main lines, chosen to lead where a favourable deviation exists,
   preferring moves I already play and moves that transpose.
3. **Deviation**: an inferior but sound move, not lost against the best defence, outside the opponents'
   knowledge and hard to play for an unprepared opponent: in the Caro-Kann, 6.h4 h6 7.N1e2, Nf4 and Ngh5
   (7.Nh3 against 6...h5); in the French, the Neg5 lines. A generous limit in points below the best move
   (2026-10-26, at first none; see "Where to deviate"). Then
   every way of punishing the opponent's mistakes, until a calm, equal position.

The phases are not strict: the Benoni is both a key choice and a deviation. The test zone is the whole
Caro-Kann and French (3.Nc3): does the lab find the zones I built by hand?

## Engines (2026-10-25)

In `~/Workspace/Chess/engines/` (outside git and `data/`): `lc0` 0.32.1 (Homebrew, Metal) with three
networks, T1 256x10, T1 512x15 and BT4 1024x15; Maia-1 (1100 to 1900); Maia-3 5M, 23M and 79M
(rating-conditioned, 600 to 2600 Lichess blitz) in their own Python environment.

- **Lc0 without search** (`lab/policy.py`): one inference on the position gives the policy and the value;
  then every legal move's position is evaluated once, in one batch, and no deeper. In UCI: a search of
  (legal moves + 1) nodes with an optimistic first-play value (`FpuStrategy absolute`, `FpuValue 1`, at the
  root too, no smart pruning), so every unvisited move looks best and gets exactly one visit.
- lc0 writes castling king-takes-rook (`e8h8`), as the explorer does (`e1h1`); both are converted.
- **Lc0's Python bindings** (`lczero.backends`, tried 2026-10-26 in a scratch environment, not adopted): built
  from the v0.32.1 sources (`pip install` of the repository, Meson; Metal, BLAS through Accelerate and Eigen
  compiled in). On 483 positions around the three test positions, with their history, the same moves and the
  same numbers as `lc0` over UCI: the policy within 0.0006 (UCI's percentages rounded to 0.01%, then
  normalized), the value within 0.003 points, Metal and BLAS alike. The small network: 12.8 ms a position over
  UCI; with the bindings on Metal 5.4 ms alone, 1.2 ms in batches of 8, 0.74 ms from 64 (17 times faster);
  on BLAS 11 ms alone, 4.8 ms from 32. Each batch size builds its graph on first use.

## The intuition index (2026-10-25)

Independent of any level: the contrast between what looks good without calculation (Lc0 at one node) and
what the tactics say (Stockfish, depth 14). Each move's loss against the best, in points of winning
chances, becomes a Lichess move accuracy (100 for the best move, about 65 for a loss of 10 points, 50 for
15). Two indices from 0 (the natural moves lose) to 1 (the natural moves are good):

- **instinct**: the accuracy of the natural move, weighted by the policy (95% of its weight, 12 moves at
  most);
- **glance**: the accuracy of the move picked by looking at every legal move once with the value head.

First witnesses (`lab/intuition.py`): 1.e4 e5, White to move, 0.91 to 0.95; Karpov 4...Nd7 5.Ng5 Ngf6
6.Bd3 e6 7.N1f3, 0.92 to 0.95; Rubinstein 7...b6 8.Neg5, 0.53 to 0.57 (glance 0.41): the natural 8...O-O
loses 12 points and the only defence, 8...Bb4+, has under 5% of the policy.

Validation (`lab/validate_intuition.py`, `lab/validate_more.py`): the index must predict the accuracy
humans actually reach in the same positions, at three levels (club, 2500+ blitz, masters); if it predicts
all three, it measures the position. On 1,200 repertoire positions with 200 club games or more (2026-10-26),
Spearman rank correlation with the human accuracy:

| index | club | 2500+ blitz | masters |
|---|---|---|---|
| instinct, T1 256x10 (small) | +0.63 | +0.54 | +0.46 |
| instinct, T1 512x15 | +0.59 | +0.54 | +0.52 |
| instinct, BT4 1024x15 (large) | +0.54 | +0.50 | +0.54 |
| glance (small to large) | +0.15 to +0.23 | +0.16 to +0.19 | +0.28 to +0.31 |
| Stockfish only: loss of the second best move | -0.15 | -0.18 | +0.02 |
| Stockfish only: number of good moves | +0.06 | +0.08 | -0.12 |
| Maia-3 instinct at 1500 / 1900 / 2300 | +0.84 / +0.93 / +0.94 | +0.64 / +0.77 / +0.92 | +0.41 / +0.58 / +0.77 |

- The idea holds: the contrast between intuition (Lc0's policy) and tactics (Stockfish) predicts human
  accuracy at every level, and Stockfish alone (only moves, how many good moves) predicts nothing. Club and
  2500+ accuracy correlate +0.84 between themselves: a position hard for one is hard for the other.
- The size of the network acts like a level: the small one fits the club best, the large one the masters.
- Looking one move ahead with the value head (glance) adds little: the policy is what matters.
- Maia fits far better, but it learnt from these very games, and at one level only; the Lc0 index is what
  can be trusted on rare positions, where there are no games to check against.
- The index is also calibrated: by sixths of the positions, the small network's instinct against the
  club's accuracy is 0.80/0.80, 0.90/0.88, 0.93/0.91, 0.95/0.93, 0.96/0.95, 0.98/0.96 (means 0.922 and
  0.906); the medium and large networks match 2500+ blitz the same way (0.950 and 0.956 against 0.946);
  masters (0.966) are above every network. So the small network's policy is an opponent as accurate as the
  club, with no level given: the opponent model where there are no games. The depth of the position (its
  ply) does not explain any of this.
- Among the 12 hardest of the 1,200 for the small network, the zones I built by hand: for the opponent, the
  Fort Knox 6.Neg5 Nd7 7.Bd3 Ngf6 8.O-O (the natural ...h6 is a mistake; club 0.62); for me, the Rubinstein
  8.Neg5 Bb7 (the natural 9.Ne5 is a mistake; club 0.63) and the Karpov 5.Ng5 h6 6.Ne6 Qa5+ 7.Bd2 Qb6 8.Bd3
  fxe6 (club 0.83: the follow-up is known). Hard positions on my side are the moves to train most.

The zones (`lab/zones.py`, 2026-10-26): every opponent position of my Caro-Kann and French Rubinstein, and
of the club's main paths (`lab/crawl.py`, every move reached by 0.1% of the club games, up to ply 22: 776
and 794 positions). Share of the opponent's positions with an instinct under 0.80, by ply:

| ply | Caro-Kann, my lines | Caro-Kann, club paths | French, my lines | French, club paths |
|---|---|---|---|---|
| 10 to 13 | 11% | 3% | 19% | 2% |
| 14 to 17 | 9% | 4% | 27% | 3% |
| 18 to 21 | 20% | 0% | 24% | 0% |

My hand-built lines take the opponent into counter-intuitive positions far more often than the main
paths, at the same depth. The hardest ones are mine: the Tartakower 6.c3 Bd6 7.Bd3 O-O 8.Qc2 Re8+ 9.Ne2
lines (club 0.52, the club scores 17 points below the eval), the Karpov 5.Ng5 lines (9.Ne5: club 0.59,
-18; 7...h6 8.Nxe6 fxe6 9.Bg6+ Ke7 10.O-O: club 0.52, -20 over 6,025 games), the Nh5 cluster (7.N1e2 Nd7
8.Nf4 Bh7 9.c3 e6 10.Bc4 Be7 11.Ngh5: 0.61), 6...h5 7.Nh3 (10.O-O, hidden ...Ng4); in the French, the
Fort Knox 6.Neg5 (natural ...h6: club 0.66, -13 over 7,875 games), 8.Neg5 against 7...b6, the 4...Be7 and
4...Nf6 5.Nxf6+ Qxf6 lines with h4, and the Alekhine-Chatard 6.h4 c5 7.Nb5 (club 0.49, -16).

The depth of intuition (`lab/depth.py`): Lc0 at 1, 10, 100, 1,000, 10,000 nodes; the number of nodes it
needs to find a good move is a scale with no level (intuitive, short tactic, deep tactic). Witnesses: after
8.Neg5 against 7...b6, all three networks play the natural 8...O-O, then 8...h6 (worse) up to 1,000 nodes,
and only the medium one finds 8...Bb4+ at 10,000; in the Fort Knox position the small network needs 100
nodes to drop ...h6, the larger ones never play it. Too slow on the M2 to run widely (two minutes a
position); not yet validated.

## The level-free deviation search (2026-10-26)

`lab/deviations.py --opponent blind`: at one of my positions, each candidate move is followed k opponent
moves deep, the opponent playing the small network's policy (as accurate as the club, with no level), I
play Stockfish's best; `net` is my expected winning chances then, minus the best move's eval now. Three
of my decision points, k = 3:

- Fort Knox 5...Bc6: 6.Neg5 comes first by far (net +21.0, against +9.3 for the main 6.Bd3), though it
  costs 4 points and the club plays it 2% of the time: the method finds my deviation by itself.
- Rubinstein 7...b6: 8.Neg5 is both the best move and the best practical one (+21.6).
- Caro-Kann 6.h4 h6: 7.N1e2 and 7.Nh3 do not stand out (+1.9, against +4.7 for 7.Nf3), nor at k = 5: the
  Nh5 positions come four or five opponent moves later, and my moves are Stockfish's, not the ones that
  steer there.

`lab/scout.py` chooses my later moves too: among my sound moves (within 3 points of Stockfish's best:
its three best and the policy's two most natural), the one with the best practical value, an expectimax
against the same blind opponent. Caro-Kann 6.h4 h6, k = 5 (about 10 minutes a move): 7.Nh3 +4.8, 7.N1e2
+4.3, 7.Nf3 +2.9, 7.h5 +2.2; the likeliest line after both knight moves is 7...Nf6 8.Nf4 Bh7 9.Bc4 e6
10.O-O Bd6 11.Ngh5 O-O. My hand-built deviation and its Nh5 cluster come out by themselves once the
search steers and looks five opponent moves ahead (at k = 3, all four are within 1.6 points).

With `--club`, the opponent plays the practical game where the club has played (club games plus 50 games'
worth of the policy) and by intuition only off the beaten track: 7.Nf3 +8.9 (then 7...Nd7 8.Bf4, a
sideline of its own), 7.Nh3 +6.1, 7.N1e2 +5.8, 7.h5 +0.9; after the knight moves, 7...e6 8.Nf4 Bh7 9.c3
Nf6 10.Nfh5 Nbd7 11.Bc4. The two models agree on the Nh5 cluster and on 7.h5 being the least practical;
they disagree on 7.Nf3, whose value comes from where the club actually goes wrong.

Five more decision points against the club opponent, k = 5, by value (my winning chances; within a
position compare values, since `net` subtracts a depth-14 eval of the root that can be off by several
points, as 4.Bg5's price of -4.7 shows); 3 to 30 minutes a move:

- Fort Knox 5...Bc6: 6.Neg5 79.0 (net +24.0), 6.Bd3 64.3, 6.Ng3 59.7. My move, by more than against the
  blind opponent; likeliest 6...Nd7? 7.Nxf7 Kxf7 8.Ng5+.
- Caro-Kann 4...Nd7: 5.Bd3 64.4, 5.Bc4 63.0, 5.Nf3 62.7, 5.Ng5 62.6 (my move). All within two points;
  5.Ng5 and 5.Bd3 reach the same main line (...Ngf6, Bd3 or Ng5, ...e6 Qe2 Bd6 N1f3 h6 Ne4).
- Caro-Kann 4...Nf6: 5.Qd3 58.6, 5.Ng3 56.4, 5.Nxf6+ 54.7 (my move). The Tartakower positions where the
  club goes wrong (8.Qc2 Re8+ 9.Ne2, club accuracy 0.52, gap -17) come at the last opponent move of the
  horizon, where only the immediate loss counts: the search may undervalue them.
- Winawer 3...Bb4 (not in my repertoire): 4.Ne2 62.8, 4.e5 61.0, 4.Qd3 58.9, 4.exd5 58.3, 4.a3 56.5. The
  reconnaissance's 4.Qd3 does not stand out; 4.Ne2 (price 0.5) does, to be checked.
- Classical 3...Nf6 (not in my repertoire): 4.Bg5 60.6, 4.exd5 58.0, 4.e5 56.9; after 4.Bg5 Be7 5.e5 Nfd7
  the search plays 6.h4, the reconnaissance's Alekhine-Chatard idea.

## Three players (2026-10-26, my framing)

Each is a move distribution, kept in the caches:

1. Perfect: Stockfish. Its evals become a distribution through a softmax of the losses, in points of
   winning chances: p(m) ∝ exp(-loss(m) / T), with T = 2 points, about the uncertainty of a depth-14 eval
   (`lab/perfect.py`). `evals.sqlite` holds one eval per position: a move's is that of the position after.
2. Practical: what the club plays (Lichess, blitz and rapid, 1800 to 2499). Not theory but a mixture:
   players who analysed the position, who remember a move or more, who play by intuition at various levels
   and thinking times. `explorer.sqlite` holds games, results and average rating for the 15 most played.
3. Intuitive: the small Lc0 network's policy, with no level. It alone says whether a position is hard to
   play, and whether a move is hard to remember. `policy.sqlite` holds the whole policy.

Rubinstein 7...b6 8.Neg5, Black to move: perfect 8...Bb4+ 81%; practical 8...Bb7 50%, O-O 26%, h6 18%;
intuitive O-O 46%, Bb7 26%, h6 18%. In a quiet position (Caro-Kann 6.h4 h6) the perfect player spreads
over a dozen moves within 2 points: depth 14 cannot tell them apart.

## Toward the builder: a proposal (2026-10-26, not decided)

- The opponent (superseded the same day by the practical-model player, below): the club's moves where it
  has played, the small Lc0 network's policy elsewhere (in place of the equal split between sound moves),
  blended as the builder's prior already does. The same model
  serves both choices: my moves (the expectimax) and the opponent moves the repertoire covers. A small
  bias: where both exist, the club is a little less accurate than the policy (0.906 against 0.922), so
  the blend slightly favors popular lines over rare deviations; a policy temperature can correct it.
- My moves: the expectimax of `lab/scout.py` (five opponent moves ahead, my later moves chosen among sound
  ones) instead of one move at a time; my habits and transpositions as tie-breaks.
- Where lines end: a calm position is one where the natural moves are good for both sides (instinct 0.95
  or more) and the eval is level; the index gives that criterion with no level.
- The trainer: my own positions with a low instinct are the moves that intuition will not find; they are
  the ones to train first.
- The cost: about ten minutes per candidate move at k = 5 on the M2, shared between moves through the
  caches; a whole repertoire needs the search to share its subtrees, not one search per decision.

## Where to deviate (2026-10-26, agreed)

Two questions remain: where to leave the main lines (playing them to the end of theory is not in my
interest), and which opponent moves the repertoire keeps. This section answers the first. The Caro-Kann
shows what a good deviation is: 7.N1e2 and 7.Nh3 score better than elsewhere, and they are not bad lines,
only slightly inferior and playable.

- **Constraints**, on every one of my repertoire moves, not only the deviation:
  - relative: at most δ points below the best move, with a generous δ (8 proposed; 6.Neg5 costs 4.4).
    Replaced the same day by ζ (below, "ζ"), then removed (below, "The floor alone");
  - absolute: never lost against the best player. Stockfish's eval stays above a floor (45% proposed,
    about -0.5 pawn; nearer 40% as Black), at depth 20 or more since it is a safety constraint. The floor
    bounds the sum of small concessions.
- **Objective**: the expected result against the practical-model opponent (below), as in `lab/scout.py`,
  times my own reach p (the objective probability below, from the opponent's branching): a deviation
  early in a frequent line pays in more games.
- **My own intuition**: out of my repertoire I am modelled by the same practical-model player (my p is
  low there, so I play close to intuition): a line whose natural moves are good for me scores higher with
  no extra weight. Cheaper fallback: my instinct index at the end of the line.
- **Horizon**, until the second question is answered: five of my repertoire moves, then three moves each
  played by the model, then Stockfish.

## The practical-model player (2026-10-26, agreed)

The club's shares are the natural opponent, but they get noisy as games become few. My proposal, with the
adjustments agreed the same day:

- **Objective probability p of a node**, for the side to move: the probability that a player with a fixed
  repertoire reaches it, from the other side's branching probabilities only (his own moves count as 1).
  Summed over the move orders, not one path: some positions with 200 club games or more have a path
  product of 1e-9, reached by another order. Computed by propagation from the root, like the builder's
  reach. On the 764 validation positions whose ancestors are all cached, log10 p spans -9 to 0, median
  -3.2.
- **The practical-model player**, set by p; two forms to compare:
  - arithmetic (my proposal): α·perfect + (1-α)·intuitive;
  - geometric: p(m) ∝ intuitive(m)·exp(-β·loss(m)), intuition filtered by knowledge or calculation. It
    never gives mass to a move nobody thinks of (after Caro-Kann 6.h4 h6 the perfect player gives a4 4%
    and Rh3 3%), and it needs Stockfish on the natural moves only.

  In both, α or β = σ(K1 + K2·log p): in log p, since p spans orders of magnitude. Plus a temperature on
  the intuitive part: the club is a little less accurate than the policy (0.906 against 0.922).
- **Blend with the observed player**: α'·practical + (1-α')·practical-model, with α' = n/(N+n) and n the
  club games. It is a smooth transition from the well-observed practical player to the model, which comes
  close to the intuitive player as p falls.
- **p'**: p computed with this blended player at every node (each branching probability smoothed, then
  multiplied). It equals p where games are plentiful and is never zero deep down. Chosen over
  p'' = α'p + (1-α')p', which mixes whole products with the last node's α'. One pass from the root: each
  node depends on its ancestors only.
- **The fit**: maximum likelihood of the club's moves (cross-entropy), checked on held-out positions with
  many games (the test zone and the validation set). It gives K1, K2, the temperature, and which form
  wins. N the same way: subsample the games of rich positions and find the N that best recovers the full
  shares. Each position counts once, the score weighted by games given beside it: weighted by games, the
  first moves alone would decide (1.5 billion games at the root, 200 deep down) (2026-10-26, agreed).
  Stockfish only for missing natural moves.
- **The gaps** (2026-10-26, agreed): the explorer cache lacks the positions where the spec set my move
  (1.e4 c6 is missing, 1.e4 c6 2.d4 is there), and p needs the club's shares there: only 13,920 of the
  27,594 cached club positions were reached, none of the test zone. Two estimates from the cache alone
  failed. The shares from the next positions' games came out too high (median ×1.3, one move in ten ×4.5),
  transpositions counting in. p from the position's own frequency, divided by my own shares, was off by a
  factor of 10 in median beyond ply 12, the main move order not being the path's. So the explorer was
  asked for the 1,940 missing positions on the paths to a position the fit can use (`lab.player --fill`,
  cached for good): the one exception to "from the caches only".

### The fit (2026-10-26, `lab/player.py`, `data/lab/player-fit.json`)

16,085 positions with 200 club games or more, Lc0's policy and Stockfish on 95% of it: 11,941 to fit,
4,144 held out in the test zone, 241 of them below p = 1e-4. A first fit on the 1,828 positions that had
both could not measure p (36 test positions below 1e-4), so the policy was computed for the 12,914 positions
without one (3 minutes) and Stockfish for 13,866 (16 minutes on a 32-vCPU spot VM, 32 single-threaded
workers). Mean cross-entropy on the test zone, in bits per move (the club's own entropy, the floor, is 1.67):

| Player | Test | Fitted |
|---|---|---|
| Intuitive (the policy) | 2.521 | |
| Intuitive, tempered | 2.506 | τ = 1.12 |
| Perfect (T = 2) | 3.464 | |
| Arithmetic, constant α | **2.448** | α = 0.26, T = 9.1 points, τ = 0.94 |
| Arithmetic, α set by p | 2.448 | α 0.22 at p = 1 to 0.28 at 1e-4 |
| Geometric, constant β | 2.479 | β = 0.041 per point, τ = 1.28 |
| Geometric, β set by p | 2.476 | β 0.063 at p = 1 to 0.033 at 1e-4 |

- **The arithmetic form wins**, by 0.03 bits, as in the first fit. It takes 0.07 bits off the policy
  alone, a tenth of the way to the floor; weighted by games, the same order (2.188 against 2.191).
- **p changes nothing**: set by p, α or β gains 0.003 bits at most, with p now spanning 1e-6 to 1. The
  club does not play more by intuition in rarer positions, not among positions of 200 games or more: the
  model player takes a constant α. The club's own entropy rises as p falls (1.35 bits above 0.1, 1.77
  below 1e-4), and the model's gap to it a little (0.76 to 0.84 bits).
- **N = 6.2 games**: past about ten games, the observed shares beat the model. The blend beats both the
  shares alone and the model alone at every sample size, from 5 games to 200.

Next: the player and the constraints in `lab/scout.py` (rewritten 2026-10-26, the level-free version in
the history). After that, the second question: which opponent moves the repertoire keeps.

### The scout's breadth (2026-10-26)

Past my horizon, a move of either side less likely than `model_branch` counts at its eval. Its value,
measured on 7.N1e2 after Caro-Kann 6.h4 h6, one move of mine then three each by the model, settles
around 0.07, the default since:

| model_branch | 0.3 | 0.2 | 0.15 | 0.1 | 0.07 | 0.05 | 0.03 | 0.02 |
|---|---|---|---|---|---|---|---|---|
| Value (%) | 52.4 | 52.9 | 53.3 | 54.0 | 54.1 | 54.3 | 54.2 | 54.3 |
| Nodes | 19 | 27 | 36 | 50 | 67 | 108 | 247 | 393 |

A cut too coarse undervalues the line by two points: the replies it drops are the opponent's mistakes.
My moves to the horizon cost the most (13, 61, 201 and 636 nodes for two to five of them, the model
phase cut), about 22 nodes per position the model phase starts from: some 11,000 nodes, 3.5 to 7 hours,
for one candidate at the full horizon on the Mac (more: below, "The club's verdict first").

`branch`, the same cut within my horizon, measured with three moves of mine and no model phase, at fixed
depths (below, "Evals at fixed depths"; the first values, with the cache's mixed depths, were about 1.3 points
higher on 7.N1e2):

| branch | 0.3 | 0.15 | 0.07 | 0.05 | 0.03 |
|---|---|---|---|---|---|
| 7.N1e2, value (%) | 53.5 | 53.6 | 53.6 | | 54.0 |
| nodes | 32 | 59 | 81 | | 166 |
| Fort Knox 6.Neg5, value (%) | 61.3 | 73.9 | 75.4 | 76.4 | 76.7 |
| nodes | 1 | 64 | 139 | 172 | 243 |

On 7.N1e2 it moves by half a point at most; not on 6.Neg5, where each rarer reply followed is one more
mistake: 2.8 points from 0.15 to 0.03. 0.15 stays for now; where the traps are, it undervalues them a little.
`my_choices`, the same way: 7.N1e2 53.6, 53.6 and 54.2 for 3, 4 and 5 (59, 87 and 114 nodes), 6.Neg5 73.9 for
all three (76, 90 and 104 nodes): 3 stays.

The club's own scores (the explorer cache, 1800-2499, blitz and rapid, the mover's score) agree with my moves:
6.Neg5 63.6% (7,692 games) against 6.Bd3 50.7%; 7.Nh3 58.3% and 7.N1e2 58.2% against 7.Nf3 54.5% and 7.h5
51.6%; 8.Neg5 72.3% (764 games) against 8.Ne5 64.5%. The scout overrates the size: 74 to 77% for 6.Neg5
against 64% (the Lichess curve is not the club's, 6.Bd3's eval of 55% scores 50.7%; the short horizon counts
8.Nxf7 Kxf7 at its eval). The players of a rare move often know it, which flatters its score (as I will
know mine).

### The floor alone (2026-10-26, my decision)

δ is gone, even as a bound of the search: what matters is that the line is not lost. That Stockfish finds it
less clear once out of complications it knows how to escape matters little if my opponent has not escaped the
ones I set; whether the opponent escapes is ζ's question, not δ's. The floor already bounds what a move
concedes, by the eval now minus the floor: about 8.5 points after Caro-Kann 6.h4 h6 (53.5%), 10 after Fort Knox
5...Bc6 (55%), 21 after Rubinstein 7...b6 (66%, where Black has erred). So a bound on the concession would only
bind where I am already better, punishing, which is where giving back some of the edge to keep the opponent in
the complications can pay. My candidates are every judged move that keeps the floor; one below it is not
searched (its floor is one search at depth 20). Within the search, my later moves are the `my_choices` (3)
conceding least that keep the floor: there Stockfish's view still filters, a cost bound to be measured.

### Evals at fixed depths (2026-10-26)

The package's cache keeps one eval per position, the deepest of any search. The scout reads its prices at
depth 14 and its floors at 20: a floor's search overwrote the depth-14 eval of the same position, and the
value of a line depended on what was searched before, by about a point (7.N1e2 at `branch` 0.3: 53.8 on a
first pass, 54.8 on the second). So the scout has its own cache, `evals-by-depth.sqlite`, by position and
depth (`DepthEvaluator` in `lab/scout.py`): a move is priced at exactly the depth of its MultiPV search (the
position after it at 13), a floor at exactly 20, and the first eval at a depth stays. Each time it opens, it
takes in the package's evals at their own depths (305,000 rows in 0.6 s); a position whose shallow eval was
overwritten is searched again. The floors go to the package's cache too, never replacing a deeper eval: `rep`
searches at 20 as well. Scouts run on a cloud VM, each with its own copy of the caches (one SQLite file shared
by 20 processes would wait on its locks), are merged back on the Mac: `lab.scout --merge` for the caches by
depth, `lab.player --merge` and `--merge-policies` for the others. The first VM run (2026-10-26) predates it:
its values carry this noise, and a gap under a point between two candidates decides nothing.

### ζ: how principled the repertoire is (2026-10-26, my proposal, agreed)

δ had no theoretical ground, and 8 points is not generous. In its place, the opponent of the search is
J_ζ = ζ·perfect + (1-ζ)·J, J the practical-model player blended with the club's shares: on each of its
moves it plays Stockfish's best with probability ζ, 0 by default (`ScoutParameters.zeta`).

- **The ground**: Huber's contamination model, the classic of robust decisions. I trust the model at 1 - ζ;
  otherwise the opponent is the worst case for me, which in a zero-sum game is the perfect player. ζ is my
  doubt about the model: a stronger opponent than the band, a prepared one, a model that overrates mistakes.
- **The soft form of δ**: each point my move concedes against best play costs about ζ points of value. δ
  forbade beyond a bound, ζ prices every point (the multiplier of the constraint). At ζ = 0.2, conceding 10
  points pays only if the practical gain is above 2.
- **Unlike α**: α only counts where the club has few games (its shares weigh 89% from 50 games); ζ applies
  after the blend, so in the first moves too. ζ = 0 is the practical repertoire, ζ = 1 the engine's
  (against best defence, to the horizon).
- **In the search**: on the opponent's moves, in both phases, each move weighs (1-ζ)·q, Stockfish's best
  ζ more, followed when its weight reaches `branch`; p still follows the practical player (how often games
  go there). No engine work more: the best move is among the judged ones. The floor stays, an absolute
  guarantee that also covers the shallow depth-14 evals.

## The backbone and its decision points (2026-10-26, agreed)

A deviation lies on a main axis of the opening, and early: 7.N1e2 comes at move 7. So a builder's first pass
finds the backbone, and searches for deviations only at its decision points; the rest of the tree is cheap.

- **The backbone** (my proposal): my moves fixed, the opponent's every club move with n/Nt at or above a
  threshold, n the games of the position in the club band (1800-2499, blitz and rapid), Nt all the band's
  games (1,510,337,517). The threshold bounds the nodes, not the candidates: a candidate deviation may never
  have been played. My fixed moves come from the masters where they have enough games, then 2500+ blitz,
  then my habit or Stockfish; whether the masters add anything is open. Stockfish could draw the backbone
  instead.
- **A decision point** is one of my positions on the backbone. Its threshold has a meaning: a node holding a
  share s of the opening's games adds at most s times the deviation's gain to the repertoire's score. After
  3.Nc3 in the Caro-Kann (7.5e-3 of Nt), a node at 1e-4 holds 1.3% of the games: a deviation gaining 10
  points there adds 0.13 points.
- **How many**, my positions counted over the explorer's cache. "Every" follows every club move of mine;
  "one" follows mine (here the club's most played, not yet my own moves). In brackets: positions above the
  threshold the cache lacks, their subtrees uncounted.

| Start | Games | 1e-4 every / one | 3e-5 every / one | 1e-5 every / one |
|---|---|---|---|---|
| Caro-Kann (White), 3.Nc3 | 11.4M | 52 (6) / 29 | 159 (34) / 61 (4) | 379 (185) / 94 (30) |
| French Rubinstein (White), 4.Nxe4 | 4.6M | 18 / 14 | 53 / 31 (2) | 94 (76) / 47 (24) |
| Sicilian 2...e6 (White), 4.Nxd4 | 13.6M | 43 (19) / 27 (6) | 107 (120) / 44 (32) | 181 (333) / 54 (64) |
| Najdorf (White), 5...a6 | 13.1M | 40 (3) / 9 | 108 (45) / 20 (7) | 219 (191) / 30 (22) |
| Najdorf (Black), 5...a6 | 13.1M | 37 (3) / 31 (1) | 80 (45) / 60 (14) | 161 (191) / 93 (55) |

- **At 1e-4 they look alike**: 9 to 31 points, down to moves 8 to 12. Fixing my moves shrinks the count as
  White, 1.3 to 4.4 times at 1e-4 and more below. As Black it barely shrinks (37 to 31): the opponent's many
  sixth moves are the branching, each with many games.
- **Below, the Sicilians diverge**: the cache was filled for the test zone (the Caro-Kann and the French) and
  holds only what the drafts asked of the Sicilians. At 1e-5 the Sicilian 2...e6 lacks more positions than it
  has; the Najdorf as Black is probably above 150. Filling the gaps costs a few hundred to a few thousand
  requests, under an hour.
- **The cost**: under 10 hours of caching per repertoire, with room in the Caro-Kann and the Rubinstein,
  since the Sicilians are the real test (the target in [ROADMAP.md](ROADMAP.md), phase 4). The scout at every
  candidate is far from it: 200,000 nodes or more a decision point (below, "The club's verdict first").

## The club's verdict first (2026-10-27, agreed)

The first VM run (2026-10-26, 12 hours on a spot c2d-highcpu-32, one candidate a vCPU, one Stockfish thread
each, the code before ζ): no scout finished. Each had searched some 8,000 to 17,000 nodes (60,000 to 136,000
new evals, about 430 a minute), past the 11,000 estimated for a candidate. A decision point and its dozen
candidates cost 200,000 nodes or more at the full horizon; my later moves multiply the tree by about 3 each
(`my_choices` 3). The run's caches are merged into the Mac's: 4.9 million evals, 510,000 policies (Lc0 on
BLAS).

Yet at a decision point the club has played what the scout predicts. At 3e-5 a decision point has 45,000
games or more: each of my moves played 600 times has its score within ±2 points. And the club alone ranks
first the deviations of the three test positions, the scout's acceptance criterion (ROADMAP phase 2):

| Position | Move | Games | Club score | Stockfish | Mover |
|---|---|---|---|---|---|
| Caro-Kann 6.h4 h6 | 7.Nf3 (main) | 534,243 | 54.5 | 52.1 | 2092 |
| | 7.h5 | 436,102 | 51.6 | 52.5 | 2041 |
| | **7.Nh3** | 74,339 | **58.3** | 52.3 | 2144 |
| | **7.N1e2** | 36,661 | **58.2** | 51.6 | 2124 |
| | 7.f4 | 12,190 | 57.2 | 50.8 | 2157 |
| Fort Knox 5...Bc6 | 6.Bd3 (main) | 321,079 | 50.7 | 54.7 | 2115 |
| | **6.Neg5** | 7,692 | **63.6** | 50.6 | 2169 |
| | 6.Ned2 | 2,879 | 56.7 | 53.7 | 2180 |
| Rubinstein 7...b6 | 8.O-O (main) | 2,827 | 55.5 | 51.9 | 2154 |
| | 8.Ne5 | 1,457 | 64.5 | 58.8 | 2204 |
| | **8.Neg5** | 764 | **72.3** | 67.0 | 2214 |

The club's score is the mover's, Stockfish's the eval after the move (the package cache, the Lichess curve);
two moves the scout never judged come out: 7.f4 and 6.Ned2. "Mover" is the explorer's `averageRating`, the
rating of the player who made the move (`LichessEntry::new_single` in lila-openingexplorer; a game's band is
the midpoint of both ratings). The players of a deviation are some 50 points stronger, enough to explain its
edge if their opponents were not. Within each band the gap nearly vanishes, and the edge stays (the explorer by
band, 2026-10-27; the mover's rating against the main move's in brackets):

| Edge over the main move, points | 1800-1999 | 2000-2199 | 2200-2499 |
|---|---|---|---|
| Caro-Kann 7.Nh3 | +4.6 (+23) | +3.9 (+12) | +3.5 (-1) |
| Caro-Kann 7.N1e2 | +2.9 (+7) | +4.4 (+6) | +3.7 (+5) |
| Caro-Kann 7.f4 | +3.2 (+17) | +3.3 (+9) | +2.3 (+6) |
| Fort Knox 6.Neg5 | +11.7 (-1) | +14.3 (+20) | +12.5 (+19) |
| Fort Knox 6.Ned2 | +3.3 (-9) | +4.7 (+20) | +8.2 (+46) |
| Rubinstein 8.Neg5 | +20.0 (+10) | +19.1 (+30) | +12.5 (+7) |
| Rubinstein 8.Ne5 | +5.3 (+28) | +11.6 (+20) | +6.0 (+12) |

The overall gap came from the mix of bands: the deviations are played more in the upper ones. Twenty points of
rating are worth 3 points of score at most, and less since Lichess pairs close ratings. The edges shrink a
little in the top band (8.Neg5 from 20 to 12.5): stronger opponents escape more often.

- **Measure where the club has played, model where it has not.** At each decision point, my moves the club
  has played enough, by their score against the main move's (the eval's difference aside, once the club's
  curve is fitted), Stockfish's floor at depth 20 kept. One explorer request (the backbone's) and one MultiPV
  search a point: about an hour of the Mac a repertoire.
- **The scout for what the club has not played**: a move Lc0 or Stockfish likes with few games. Before it, its
  cost (`my_choices` 1 and 2, untried) and its calibration against the club where the club has played (the
  run's candidates, on the Mac at a short horizon).
- **After the deviation**, punishing until calm: the club's replies are data too while their games last, my
  moves Stockfish's best above the floor, the model deeper, where the games run out.
- **The caveats**: the players of a rare move often know it, as I will; blitz and rapid are mixed.

### The screen on the Caro-Kann (2026-10-27, `lab/screen.py`)

From 3.Nc3, 4.Nxe4, 5.Ng3 and 6.h4 given, the threshold 3e-5 (45,310 games), moves measured from 625 games:
54 decision points, in 10 minutes on the Mac (the explorer's missing positions, Stockfish at depth 20 on the
measured moves); `data/lab/screen/2026-10-27T1848.json`.

- **It finds the test zone's deviations**: after 6.h4 h6, 7.Nh3 +3.7, 7.N1e2 +3.6 and 7.f4 +2.6 over the
  masters' 7.Nf3; after 6.h4 h5, 7.Nh3 +5.1 and 7.N1e2 +5.0.
- **The largest gain is elsewhere**: after 4...Nf6, 5.Ng5 scores 53.7% over 460,916 games against 50.9% for the
  masters' 5.Nxf6+, +2.9 on 43% of the start's games (a gain of 1.2 points), its eval 47.2% (sound by two
  points). After 4...Nd7, the masters' 5.Ng5 is the club's best too (58.6%); after 3...e6, 4.Nf3.
- **Two things for the builder.** The decision points are nested: a deviation drops the points below it, and the
  gains found after 7.Nf3 add to 7.Nf3's value (some 0.4 points from the start, about 4 points at 6...h6,
  as much as 7.Nh3's edge). So a backup along the backbone: at each point my best move, the main move worth its
  score plus the gains below it, each scaled by its share of the point. And the winner's curse: the best of
  fifteen moves near 625 games is some 3.5 points up by noise alone (12.Kf1, +7.5 over 663 games). Each edge
  shrunk by its error (empirical Bayes, the spread of true edges estimated from the data) rather than a bare
  threshold. The nested points are kept in mind, not adopted (below, "Early deviations").
- **The Sicilian with 2...e6** (from 4.Nxd4, no move of mine given): 81 decision points in 27 minutes, two of
  them waiting out the explorer's rate limit; `data/lab/screen/2026-10-27T1941.json`. Four systems: the
  Taimanov (25 points), 4...Nf6 (24, 14 of them in the Sveshnikov reached by 5.Nc3 Nc6 6.Ndb5 d6 7.Bf4 e5),
  the Scheveningen (17), the Kan (8). The largest gains: after 4...Nc6, 5.Bf4 56.3% over 13,683 games and 5.Nb5
  54.7% over 352,149 against 50.8% for the masters' 5.Nc3; after 9...b5 in that Sveshnikov, 10.Bxf6 58.0%
  against 50.8% for 10.Nd5; after 5.Nc3 Qc7, 6.Qd3 57.4% over 6,465 against 47.3% for 6.Be3. Against 4...Nf6
  every move of mine scores under 50%; in the Kan, the club's 5.Nc3 (46.0%) trails the masters' 5.Bd3 (49.9%).
  A third thing for the builder: the explorer counts every game through a position, whatever the move order,
  so a point other openings reach is overweighted (after 4...d6 5.Nc3 a6 6.Be3 Nf6, 838,826 games against
  493,067 at the position before: the Najdorf's 6.Be3 e6 arrives there). The reach along the tree (p, as
  `lab/player.py`'s `Reach`) instead of n over the start's games.

### Early deviations (2026-10-27, my decision)

Deviating early pays: each further move of the main line gives the opponent the chance to deviate first. And a
deviation's club score already counts every reply after it, where the gains found below the main move each
need their own preparation, at points the opponent may never let me reach. So the earliest sound deviation
that scores is preferred; the backup along the backbone stays an idea to keep in mind.

### The parallel search (2026-10-26, agreed; deferred 2026-10-27)

Planned for the scout at every candidate, before the club's verdict made it unneeded at the decision points;
kept for the model's share if it turns out large (ROADMAP phase 4):

1. **The search in two passes.** The tree's shape depends only on each position's evals (the moves judged, the
   replies followed, my moves above the floor), not on the values backed up. So: an expansion level by level,
   every search of a level sent at once (the policies, then the MultiPV of the judged moves at depth 14, then
   the floors of my moves at 20: some 16 levels of 3 stages, hundreds of searches each), then the values backed
   up in Python from the caches. A position is judged at a canonical p, so the result does not depend on the
   order. To be checked against the present search on 7.N1e2 at a short horizon.
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
   true batches through its Python bindings (17 times faster on Metal, above, "Engines").

## Lc0 where the club runs out (2026-10-27, my proposals)

At the decision points the club's scores replace the practical-model player, and Lc0 with it. It keeps what
the club cannot measure:

- **The opponent's mistakes to punish**: a reply natural for the policy that Stockfish finds losing (the trap
  of the intuition index above), kept in the repertoire with the move that punishes it, weighed by its
  probability times what the punishment gains. The club measures the mistakes that are played; Lc0 foresees
  the others, in the lines and depths where the games run out.
- **The lines to push**: after a mistake I have the advantage, but keeping it may need moves my intuition
  would not find (Stockfish's best low in my policy, my natural moves losing much of the advantage). A line
  goes on while it needs them and ends when my natural moves keep the advantage: the depth follows the
  difficulty, and these hidden moves are the ones to learn, first in the trainer. It sharpens the calm position
  above (instinct 0.95 or more for both sides, a level eval): on my side the question is whether my natural
  moves keep the advantage.
- **The moves the club has not played**, candidate deviations included, judged by the practical-model player;
  and the natural move among equals, easier to remember.

How much loss is worth one more move of a line is a weight, not a threshold: the reach times the loss avoided,
against the cost of one more move to learn, still to be measured.

## Theory reconnaissance (2026-10-25, web sources, not yet checked by engine)

Rare but sound White tries the lab should find, or explain why not:

- Caro-Kann 4...Bf5: 6.h4 h6 7.Nh3 or 7.N1e2, Nf4, then Nh5 or Bc4 with Nxe6 ideas: equal, rare among
  masters (7.Nf3 almost always). The punishable natural move is ...Bd6 (Tal against Botvinnik, 1960; Keene
  avoided it against Tal, 1964). Black's antidote to know is the ...e5 break. 6...h5 7.Nh3: Nf4xg6, the g5
  hole.
- Caro-Kann 4...Nd7: 5.Ng5 (7...h6? 8.Nxe6, Deep Blue against Kasparov, 1997); 5.Bc4 Ngf6 6.Ng5 e6 7.Qe2,
  forcing the misplaced ...Nb6; 5.Qe2 (5...Ngf6?? 6.Nd6 mate), a pure trap. 4...Nf6: 5.Qd3 (5...Bf5??).
- French Winawer: 4.Qd3, "hard to fully equalize" (Miedema); 4.a3 Bxc3+ 5.bxc3 dxe4 6.Qg4 with Ne2.
  Avoid 4.Qg4 and 4.Bd2.
- French Classical: 4.Bg5 Be7 5.e5 Nfd7 6.h4 (Alekhine-Chatard), Black often slightly worse; MacCutcheon
  4...Bb4 5.e5 h6 6.Bc1, rare and dangerous (6...Ne4 7.Qg4 O-O?? 8.Bxh6).
- French Rubinstein, Fort Knox: 6.Neg5 (6...h6? is the error, 6...Bd6 holds; double-edged) or 8.Ng3 to
  dodge the exchanges.

## The opponent model (2026-10-25)

Kept for later, when the builder estimates how often club opponents go wrong where club games are few.
On the 3,778 repertoire positions with 200 club games or more (`lab/calibrate.py`), Maia-3 plays the
club's most played move 82 to 86% of the time (23M and 79M, at 2000), Maia-1 1900 65%.

## What the literature says (2026-10-25, research brief)

- Engine-only complexity (Guid and Bratko 2006: the best move changing with depth) is symmetric and
  tracks the engine's difficulty, not a human's; their search-tree features (only-moves, the share of
  good moves, seemingly good tries) did classify puzzle difficulty well (Stoiljkovikj, Bratko, Guid 2015).
- Blunder potential (the share of legal moves that blunder) predicted human errors in endgames better than
  rating or clock time (Anderson, Kleinberg, Mullainathan 2016); Regan and Haworth model a fallible player
  from engine output alone.
- A strong Lc0 policy is not fully blind: at one node it solves most of the Lichess puzzle set (88.6% in
  one study) and shows learned look-ahead; small networks and Maia are blinder. Puzzle-difficulty
  competitions (FedCSIS 2025) used exactly our ingredients: the no-search probability of the solution
  under Maia and Lc0 nets of several strengths, and the search effort Stockfish needs.
- Regan and Biswas, "depth of satisficing": a trap looks best at low depth and swings down deeper; the
  depth at which a mistake is exposed characterises it (the idea behind the depth of intuition).
- Lc0's WDL "sharpness" is a style tag (double-edged or drawish), with no evidence against human errors.
- Opening traps: davidADSP's trap scorer (the probability of reaching the trap from explorer frequencies,
  times the win rate after the punishment) and thomasahle's expectimax over explorer distributions (warned
  as possibly unsound); nobody publishes how often the best reply is found at a rating.

Sources: Stoiljkovikj et al. 2015 (cogsys.org/proceedings/2015/paper-2015-7.pdf), Anderson et al. 2016
(arXiv 1606.04956), Regan and Haworth 2011 (cse.buffalo.edu/~regan/papers/pdf/ReHa11c.pdf), Maia
(arXiv 2006.01855), Maia-2 (arXiv 2409.20553), Maia-3 / Chessformer (arXiv 2605.19091), FedCSIS 2025
(annals-csis.org/Volume_43/drp/pdf/6497.pdf), depth of satisficing (rjlipton.com/2015/10/06), Lc0 WDL
sharpness (lichess.org/@/jk_182 blog), github.com/davidADSP/chess-trap-scorer,
github.com/thomasahle/chess-openings-expectimax.
