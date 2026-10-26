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
  - relative: at most δ points below the best move, with a generous δ (8 proposed; 6.Neg5 costs 4.4);
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

### First fit (2026-10-26, `lab/player.py`, `data/lab/player-fit.json`)

1,828 positions with 200 club games or more, Lc0's policy and Stockfish on 95% of it: 785 to fit, 1,043
held out in the test zone. log10 p runs from -6.3 to -0.5, median -2.5; 80% lie between -3.5 and -1.5.
Mean cross-entropy on the test zone, in bits per move (the club's own entropy, the floor, is 1.45):

| Player | Test | Fitted |
|---|---|---|
| Intuitive (the policy) | 2.217 | |
| Intuitive, tempered | 2.200 | τ = 1.08 |
| Perfect (T = 2) | 2.972 | |
| Arithmetic, constant α | **2.138** | α = 0.25, T = 6.2 points, τ = 0.92 |
| Arithmetic, α set by p | 2.138 | α 0.16 at p = 1 to 0.31 at 1e-4 |
| Geometric, constant β | 2.166 | β = 0.052 per point, τ = 1.31 |
| Geometric, β set by p | 2.166 | β 0.086 at p = 1 to 0.040 at 1e-4 |

- **The arithmetic form wins**, by 0.03 bits. It takes 0.08 bits off the policy alone, a tenth of the way
  to the floor.
- **p is not detected**: K2 gains under 0.001 bits, in opposite directions in the two forms. The range is
  too narrow: 36 test positions below 1e-4, where the model matters.
- **N = 6.5 games**: past about ten games, the observed shares beat the model. The blend beats both
  the shares alone and the model alone at every sample size, from 5 games to 200.
- **What limits the data is the policy**: 12,914 positions reached have none. Taking positions of 30
  games or more instead of 200 adds only 79.

Next: the policy and Stockfish on the positions reached (phase 1, an engine job), to widen p downward and
measure K2; then the player and the constraints in `lab/scout.py`. After that, the second question: which
opponent moves the repertoire keeps.

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
