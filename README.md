# Ajith Crypto — 332 Parts Segment Bots

A futures-first crypto trading system built on one architectural rule: every
feature is a "part" — the same template, its own process, an explicit on/off
switch, and contracts computed from what it consumes and produces rather than
wired by hand. 332 parts across 27 blocks, running spot/futures/options
segment bots (futures built first and fully; spot and options are skeleton
by design) on top of a shared runtime substrate.

This repository is a full mirror of the working project (`ajit-segment-bots`),
history included — see **Project design** below for where the real rulebook
lives. This file only covers getting it running.

## What's actually in here

- `parts/` — the 332 parts, one file per part, grouped into blocks (`ai_brain`,
  `bull_bot`, `bear_bot`, `paper_live_trading`, `portfolio_state`,
  `risk_capital_allocation`, `market_data_feed`, …).
- `runtime/` — the substrate every part is built on: the message bus, input
  assembly (`Batch`/`LatestByKey`/`LatestValue`), level publishing, settings
  loading, the resource governor.
- `dashboard/` — board generators (`build_status_board.py`,
  `build_part_monitor.py`, `build_trade_board.py`, the live part board) and the
  contract/wiring checkers.
- `settings/*.example.toml` — the settings schema and commented defaults. Real
  values live outside the repo (see below).
- `operate/` — the systemd units and scripts that run the live spine and the
  tape capture, until the resource governor and stream-budget-planner exist as
  parts in their own right.
- `docs/` — the design corpus: `docs/features.json` (the 332-part blueprint),
  `docs/contracts.md` (R-01, how a wire's edges are computed), `docs/proposals/`,
  and `docs/superpowers/specs/2026-08-20-part-runtime-design.md` (the
  implementation spec every part is built against).
- `tests/` — one test file per block, plus `tests/integration/` for
  multi-part, real-process tests.
- `measurements/` — dated, reproducible measurements behind specific decisions
  in `CLAUDE.md` (what a part costs, what fits on a given box).

## Prerequisites

- Linux, standard **CPython 3.14.x** (not the free-threaded build — several
  pinned dependencies, including `ta-lib`-adjacent numeric packages, ship no
  `cp314t` wheel). `pyproject.toml` pins `requires-python = "==3.14.*"`.
- No C compiler is assumed or required: every pinned dependency resolves to a
  binary wheel for this interpreter.
- `git`, and (only if you want the live board / systemd-managed spine)
  `systemd --user` support.

## Setup

```bash
git clone https://github.com/ajith4134/ajith-crypto-332-parts-segment-bots.git
cd ajith-crypto-332-parts-segment-bots

python3.14 -m venv .venv
.venv/bin/pip install -e ".[test,docs]"
```

`pyproject.toml` pins every dependency exactly (`numpy`, `threadpoolctl`,
`watchdog`, `ccxt`, `websockets`, plus `pytest` and `markdown-it-py` as
optional extras) — no version resolution surprises.

### Settings

Real settings live **outside the repository**, at
`~/.config/ajit-segment-bots/settings/` (or under `$XDG_CONFIG_HOME` if set),
one TOML file per scope. The repo only ships the schema and commented
defaults as `settings/*.example.toml`. To get a working local config:

```bash
mkdir -p ~/.config/ajit-segment-bots/settings
cp settings/runtime.example.toml            ~/.config/ajit-segment-bots/settings/runtime.toml
cp settings/main-account.example.toml       ~/.config/ajit-segment-bots/settings/main-account.toml
cp settings/part-priority.example.toml      ~/.config/ajit-segment-bots/settings/part-priority.toml
cp settings/segments-futures.example.toml   ~/.config/ajit-segment-bots/settings/segments-futures.toml
```

Every setting in `runtime.example.toml` carries a `value`, a `unit`, and a
`note` explaining who set it and why — read the note before changing the
value, several exist because of a specific measured incident.

## Run the tests

```bash
.venv/bin/python3 -m pytest tests/ -q
```

Two markers matter: `slow` (forks/kills/waits on writeback — real integration
tests, not mocks) and `cgroup` (places a process under a real systemd
transient scope). Run everything except the slow ones for a fast loop:

```bash
.venv/bin/python3 -m pytest tests/ -q -m "not slow"
```

## Verify the design is intact

Before trusting any change, run the checkers the pre-commit hook already
enforces:

```bash
.venv/bin/python3 dashboard/check_contracts.py       # every part's consumes/produces matches docs/features.json
.venv/bin/python3 dashboard/check_payload_reads.py    # every field a part reads, some producer actually carries
.venv/bin/python3 dashboard/check_part_calls.py       # every method a part calls on its own object, it actually has
```

## Run it

**Paper trading, one-off, in the foreground** — the fastest way to see it
work without systemd:

```bash
.venv/bin/python3 operate/run_live_spine.py
```

This starts the market-data feed, the bull bot, the trading half, and the
governor (observing, not switching parts) as real processes, wired the way
`docs/features.json` declares. Stop it with `Ctrl-C` / `SIGTERM` — that flushes
the tape and journals cleanly; don't `kill -9` it.

**As a supervised, reboot-safe service** (what the original box actually
runs):

```bash
mkdir -p ~/.config/systemd/user
cp operate/ajit-spine.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now ajit-spine
systemctl --user status ajit-spine
loginctl enable-linger "$USER"   # so it survives you logging out
```

`operate/README.md` has the full detail on the capture-only fallback
(`ajit-capture@.service`), why the spine and the capture script refuse to run
together, and how to read what's actually been captured.

### Look at what it's doing

```bash
.venv/bin/python3 dashboard/build_status_board.py       # dashboard/status-board.html
.venv/bin/python3 dashboard/build_part_monitor.py        # dashboard/part-monitor.html
.venv/bin/python3 dashboard/build_trade_board.py         # dashboard/trade-board.html
.venv/bin/python3 dashboard/build_wiring_explorer.py      # dashboard/wiring-explorer.html
```

Every board is generated from probes that actually ran — a part with no
evidence renders as `NOT BUILT` / `NOT MEASURED` / `NOT RUNNING`, never as
healthy by default. Open the resulting `.html` file directly in a browser;
nothing here needs a server for the static boards. The live part board
(`dashboard/part_health_api.py` + `dashboard/web/`) does need one — see
`dashboard/web/` for its own build steps if you want the live-polling version
instead of a static snapshot.

## Project design — read before changing anything

This project is built to one non-negotiable rule set, `docs/transistor-rule.md`:
every feature is the same shape (a part), control is separate from data,
"off" releases real resources, a part never imports another part's module,
states are explicit, and the system grows by adding parts, not by making one
cleverer. **`CLAUDE.md` at the repository root is the authoritative, continuously
updated account of every design decision, incident, and measurement behind
this codebase** — read it before proposing or building anything. It is long
because the project's own standard is "no shortcuts, no placeholders, no
hardcoded values, real learning in the code" (RL-058), and every rule in it
exists because of something that actually happened and was measured, not
because it sounded prudent.

## What this repository is, precisely

This is a **mirror** of `ajith4134/ajit-segment-bots`, pushed with matching
git history (same commit hashes on `main` and `phase-1-market-data-feed`) so
it is an exact copy of that project as of the mirror date, not a re-export or
a snapshot with history flattened. Local, untracked, or gitignored material —
build output, rendered doc previews, session state, machine-specific
measurement samples, and anything under `.claude/` or `.state-backup*/` — is
intentionally not part of either repository; none of it is source, and some
of it (session transcripts, live account state) should never leave the
machine that produced it.
