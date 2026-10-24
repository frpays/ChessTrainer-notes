# Architecture

How the project is laid out, what each part does and how data flows between them. For the trainer's
memory model, see [MEMORIZATION.md](MEMORIZATION.md).

## Layout

```
~/Workspace/Chess/
  tools/        the code: Python package `repertoire`, CLI `rep`       github.com/frpays/ChessTrainer
  repertoire/   the repertoires: spec, PGN, lock, study and training   github.com/frpays/ChessTrainer-repertoire
  data/         not in git: my games, caches, trainer progress           backed up to gs://chess-training-aid1ie
  notes/        this documentation
```

The PGN files of `repertoire/` are the source of truth. A repertoire (`<name>.pgn`, drafted from
`<name>.spec`) is cut to the sizes I chose: its study (`<name>.study.pgn`) is published on Lichess, its
training (`<name>.training.pgn`) is served by the trainer; see [REPERTOIRES.md](REPERTOIRES.md).

## The tools package

`tools/src/repertoire/`, run with `uv run rep <command>` on the Mac (`rep --help` lists the commands).

| Area | Modules | Role |
|---|---|---|
| Lichess and engine | `lichess.py`, `games.py`, `studies.py`, `explorer.py`, `engine.py` | API client; my games (`data/games/*.ndjson`); study exports; opening explorer (club and masters, cached forever in `data/cache/explorer.sqlite`); Stockfish evaluations (`data/cache/evals.sqlite`) |
| Analysis of my games | `tree.py`, `report.py`, `mistakes.py`, `practical.py`, `ideas.py`, `models.py` | Move trees and statistics of my games; recurring losing moves; practical value of candidate moves; short ideas instead of long variations; model games for plans |
| Building repertoires | `spec.py`, `build.py`, `forced.py`, `style.py`, `starts.py`, `reach.py`, `cut.py` | Spec files (my choices, chapters, notes) and locks; the builder; forced sequences at line ends; how comments look once published; how often club games reach each repertoire, and each move of a repertoire (the coverage of a set of moves); the study and the training |
| Publishing | `sync.py`, `export.py` | Push to Lichess studies, detect online edits; one-game-per-line export |
| Trainer | `trainer.py`, `trainer/` | HTTP server; browser app (`app.js`, with chessground and chess.js vendored) |
| Entry point | `cli.py`, `config.py` | `rep` commands; paths and tokens |

## From a spec to a study and a training

1. **Spec** (`repertoire/white/<name>.spec`): start position, thresholds, my imposed moves (`[policy]`),
   chapters, plan notes, intro.
2. **Draft** (`rep draft <spec>`): the builder expands the tree from the start position. Opponent moves
   come from the club explorer (above a reach threshold, the engine filling in where club games are few,
   plus the best defence and the traps that work in club games); my moves are, in order, the spec's, the
   frozen ones (`.lock`, unless `--no-lock`), else a sound move (within
   5 points of winning chances of the best) that transposes, my habit (`prefer_habits`), or the most
   practical one. Forced sequences are played on at line ends. Output: `<name>.pgn`, and its entry in
   `starts.json`.
3. **Cut** (`rep cut <spec>`, also run by draft and refresh): below a size of 100, writes the study
   (`<name>.study.pgn`) and the training (`<name>.training.pgn`).
4. **Publish** (`rep push <pgn>`): updates the Lichess study (ids in `studies.json`) with the study; `rep edits` first shows what was changed online since the last push.
5. **Freeze** (`rep freeze <spec>`): once I have validated a study, all my moves as published go to
   `<name>.lock`, so that a later draft keeps them.
6. **Refresh** (`rep refresh <pgn>`): applies newer rules to a repertoire without a new draft.

Drafts take minutes to hours: every new position costs an explorer request and a Stockfish evaluation;
both are cached, so a second draft is fast.

## The trainer

`rep serve` builds every line of every training from the PGN files and serves a
single-page app.

| Endpoint | |
|---|---|
| `GET /lines.json` | repertoires (their training), chapters, lines and moves, with club probabilities; rebuilt when a PGN or `starts.json` changes, with an ETag |
| `GET, POST /progress` | `data/trainer/progress.json`; a POST merges the moves just trained |
| `GET, POST /settings` | `data/trainer/settings.yaml`, validated, with the defaults |
| `POST /reviews` | appends answers to `data/trainer/reviews.csv` |

The browser keeps unsaved progress and answers in local storage and sends them in batches, so a lost
connection costs nothing. Static files are served with `no-cache`: a phone always gets the current page.

## Infrastructure

- **robin**, a Raspberry Pi 4 on the home network (`ssh francois@robin.local`), runs the trainer for the
  iPhone at http://robin.local:8000. Code and repertoires are git clones in `~/projects/chess/`, with a pip
  venv in `tools/.venv`. Two systemd user units: `chess-trainer.service` (`rep serve --port 8000`) and
  `chess-trainer-backup.timer`, which pushes `data/trainer` to the GCS bucket every 30 minutes
  (`tools/data-sync push data/trainer`). robin is the only server whose progress is real.
- **The MacBook Air** drafts repertoires (Stockfish via Homebrew, the explorer) and validates every trainer
  change before it reaches robin: back up robin, `tools/data-sync pull --delete data/trainer`,
  `uv run rep serve --port 8000`, test at http://localhost:8000, then pull again afterwards: progress made
  on the Mac is thrown away.
- **Deploying**: push both repos, then on robin `git pull` in `tools` and `repertoire`. PGN and static
  files are picked up at once; a change to the Python code, or a new repertoire, needs
  `systemctl --user restart chess-trainer`.
- **Lichess**: tokens in `tools/.env`, never in git. Never two processes querying Lichess at once (drafts,
  pushes, explorer scripts): too many requests got the network blocked by Cloudflare once (2026-10-21).
- **Remote access**: the server has neither authentication nor HTTPS; from outside the home network, a VPN
  (the Freebox's WireGuard or Tailscale), never an open port.

## Dropped

- **Chessable** (2026-10-22): `rep export` writes one PGN game per line, and the import into a private
  Chessable course works, but the iOS app cannot pick a chapter to train and Chessable blocks scripts
  (Cloudflare). Replaced by the trainer.

## Later

- **Lc0 on a GPU spot VM** (2026-10-23, deferred): Lc0 on a cloud GPU, driven over SSH as a UCI engine,
  with its own cache. First use: an audit of my moves in every repertoire; Lc0 for positional choices,
  Stockfish for tactics.
