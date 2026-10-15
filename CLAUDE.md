# maphub-public: operations notes for an AI assistant

This file tells an AI coding assistant how this MapHub is set up and run day to day; people can read it as a runbook too. Claude Code loads it automatically, since a file named `CLAUDE.md` is its convention. With another assistant (one that reads `AGENTS.md`, say), point it to this file. The steps name Claude Code, which KC uses, and another assistant follows them the same way; only the notes on Claude Code's `!` prompt are specific to it. The scoring itself is separate from the assistant: by default it calls Claude Code (`claude -p`), and with `--via api` the Claude API.

MapHub, the MIDAS arXiv Preprint Hub: the scorer, the daily tables and the web viewer (see `README.md`; the design is in `DESIGN.md`). Its maintainer, Kaili Cao (KC), runs it every day from a dedicated computer that holds only this repo, with the private portfolios repo `maphub-portfolios/` inside it; code and documents are developed on KC's main computer. "The seat" below is the Claude subscription account that Claude Code is signed in to on that computer, which pays for the scoring.

This repo is public, and so is this file: nothing private goes here. It describes Kaili Cao's MapHub; a copy run by someone else replaces it with its own (README, Run your own MapHub).

## Setting up a MapHub computer

KC clones this repo to `~/maphub-public` through VS Code (where he is signed in to GitHub), starts Claude Code in it from VS Code's terminal, and asks Claude to set up the computer. Claude Code and Anaconda are already installed. Claude then does the following, asking KC before installing anything:

1. **GitHub access.** VS Code passes KC's GitHub sign-in to git only in its own terminal. If a clone asks for credentials, stop and ask KC to start Claude Code from VS Code's terminal (or to sign in with `! gh auth login`).
2. **The portfolios.** Clone the private portfolios repo inside this one (`.gitignore` leaves it out):
   ```bash
   git clone -q https://github.com/kailicao-lsap/maphub-portfolios.git ~/maphub-public/maphub-portfolios
   ```
3. **Git identity.** If `git config --global user.name` or `user.email` is empty, ask KC for the values; never guess them.
4. **Python.** Install the scorer's packages (`anthropic`, `pyarrow`) for the Python that `python` runs, which should be Anaconda's: `python -m pip install -r ~/maphub-public/scorer/requirements.txt`.
5. **Claude Code.** Confirm with KC that the signed-in account is the seat meant for MapHub: the scorer runs `claude -p` on whichever seat is signed in.
6. **The `mh` command.** Create an executable `~/.local/bin/mh`, and check that `~/.local/bin` is on the search path both in a login shell and in Claude Code's `!` prompt:
   ```bash
   #!/usr/bin/env bash
   # MapHub local runs: mh -l (latest), mh -b (back one day), mh -bb (back until the usage limit), mh -p (push)
   exec python "$HOME/maphub-public/scorer/local_run.py" "$@"
   ```
7. **Report** what was done and what KC still has to do. No test runs are needed.

Only one computer runs MapHub: once this one does, `mh` is removed from any other.

## Running MapHub

- **Commands.** `mh -l` (`--latest`) scores the current listing and pushes it with any waiting backfills; `mh -b` (`--back`) scores one listing and commits locally: first any weekday the morning runs missed, otherwise the weekday before the earliest table; `mh -bb` (`--back-all`) keeps backfilling until the usage limit stops it; `mh -p` (`--push`) pushes waiting backfills; `-y` (`--yes`) pushes without asking. Every mode first finishes a day left unfinished in `.work/`: a backfill day is committed locally, the latest listing committed and pushed. New listings get a column for every participant scored daily (status active or no-legacy), backfills for every participant with status active, all through Claude Code (`--via api` would use the Claude API instead). KC runs `mh -l` each morning (any time after arXiv's 20:00 ET announcement) and `mh -b` or `mh -bb` as his seat's usage allows, from VS Code's terminal, where his GitHub sign-in reaches git; `-bb` also spends the seat's weekly limit, so he chooses when to run it.
- **From Claude Code's `!` prompt.** `~/.bashrc` is not loaded there, which is why `mh` is a script and not a shell function. The confirmation question cannot be answered there: use `mh -p -y`, or run `mh -p` in a terminal. Commands longer than 2 minutes move to the background.
- **Commits.** Table commits come only from `local_run.py`: one commit a day, folding waiting backfills with the latest listing, with the scorer's provenance message under each date (newest first) and no seat or usage details; the subject names the dates in short, newest first, with each run of consecutive weekdays as a range and the year once (`Tables for Oct 15, Sep 1–7, 2026 (via claude-code)`), and a fold made by hand follows the same form; the script shows the message and asks before pushing. Code and documentation are changed on KC's main computer, and any commit there is drafted and shown to KC first.
- **New participants.** When a folder in `maphub-portfolios/` has a `portfolio.md` but no entry in `participants.json` (the scorer then warns and leaves it out), ask KC whether the participant joins the legacy survey of past listings (status `active`) or only new listings (status `no-legacy`); never assume. Then add the entry to maphub-portfolios' `participants.json` with a line in its `CHANGELOG.md`, and keep its README's "Who is who" table in step.
- **Two computers, one repo.** The dedicated computer pushes tables; the main computer pushes code. `local_run.py` pulls before each run (and merges `tables/index.json` if needed); on the main computer, pull before editing (KC's sessions there start by pulling every repo).
- **Resuming.** A run that stops (usage limit, a failed request) keeps every saved request in `.work/<date>/`; the next run of any mode finishes that day first, sending only the rest. Delete `.work/` only once its tables are pushed.
- **Papers held for moderation.** A rebuilt (past) listing places them by submission time, so a backfilled day can have a second file in the next ID month (`2610/2026-09-28.parquet`); a later RSS listing that announces them skips them as already scored.
- **Known arXiv and DataCite quirks.**
  - The arXiv API's index can lag behind the listing (2610.09954 on Oct 8, 2026); the scorer takes such papers from the RSS feed.
  - The arXiv API rate-limits busy clients (429, 503). The scorer waits up to about 5½ minutes; a limit on the computer's address can last an hour or more (Oct 8–9, 2026), so wait before rerunning.
  - DataCite registers each listing's DOIs some time after the 20:00 ET announcement (about 30 minutes for Oct 7, 2026), and occasionally never (2610.07148). The scorer then writes `<date>.meta.json` beside the table; the viewer reads it only when DataCite has no record.
  - Claude occasionally skips a paper in a reply; the scorer keeps the scores it got and asks again for the rest.
  - arXiv skips announcements on its holidays, and the next listing holds both days' submissions. On such a morning `mh -l` finds the current listing empty, writes an empty table for it (the viewer shows it as a day without an announcement), and pushes it as usual; a holiday the morning run missed is a gap that `mh -b` fills first, also with an empty table. Older holidays, reached by backfills, get a table of the submissions they would have held, so a few papers are dated one listing early, but none is lost or scored twice. A backfilled listing with nothing left to score gets an empty table, so every weekday has one.
- **Viewer deploys.** Each push that changes `viewer/`, `tables/` or `config-log.json` deploys through "Publish viewer". Script and stylesheet links carry the commit (`?v=…`), so a reload shows a new viewer at once.
