# MapHub: MIDAS arXiv Preprint Hub

**Viewer: <https://kailicao.github.io/maphub-public/>**

Each arXiv announcement day, MapHub scores the new submissions to a few arXiv listings for relevance to each participant, and shows the result as a heatmap: one row per paper, one column per participant, and in each cell a score from 0 to 100. A large language model assigns the scores by reading each paper's title, authors and abstract against a short research portfolio that every participant, individual or group, writes and keeps up to date. Selecting yourself shows which new papers matter most to you; selecting a paper shows whose portfolios it matches best: a starting point for finding someone to ask about it. The method works for any listings, and anyone can run a MapHub of their own (Run your own MapHub, below); this one, centered on astronomy and AI, follows astrophysics (astro-ph) and artificial intelligence (cs.AI).

MapHub is a team project (a "carpentry") of the [Eric and Wendy Schmidt AI in Science Postdoctoral Fellowship](https://midas.umich.edu/training/postdoctoral-programs/schmidt-ai-in-science-postdoctoral-fellowship/) at the Michigan Institute for Data and AI in Society (MIDAS), led by [Kaili Cao](https://kailicao.github.io/) with teammates Kyle Heyblom, Eva Albalghiti and Dimitrios Simatos. As of October 2026, MapHub has five participants and has scored every listing since Oct 7, 2026 (and earlier ones back to Sep 9); it is under development. Participants are researchers and research groups, shown by pseudonym; to join, read What participants agree to (below) and email kailicao@umich.edu. This repository holds its code, configuration log, daily tables and web viewer, all public; portfolios are shared only among participants. [DESIGN.md](DESIGN.md) explains how it works and why.

## Viewer

- The base URL shows the latest day with an announcement; each day also has its own URL, such as `https://kailicao.github.io/maphub-public/2026-10-06`. Weekends have no arXiv listing and no table. arXiv makes no announcement on its holidays: a holiday met by the morning run gets an empty table, which the viewer shows as a day without an announcement (sand-colored in the calendar), while a holiday reached by a backfill gets a table of the submissions it would have held. The arrows, the date box and the calendar, where days with a table are highlighted, move between days and keep a selected participant.
- By default, papers are ranked by their average score over the participants shown, and participants by the rank correlation (ρ) of their scores with the average of the others'. Select a participant (column name) to sort papers by relevance to them and participants by similarity to them. Select a paper (title) to sort papers by similarity to it and participants by its relevance to them. Similarity is the correlation (r) of that day's scores.
- **Top papers** (`<base>/top`) lists one participant's highest-scored papers over a range of listing dates (by default the past calendar month up to the latest table, top 50), ties going to the newer listing; other participants can be shown beside, with their scores for the same papers; each listing date opens that day's heatmap. Its URL keeps the choice, as in `<base>/top#p=52454&also=4547,25222&from=2026-09-12&to=2026-10-12&n=50`. Selecting a participant in the heatmap offers a link to their top papers up to that day.
- Hover over an arXiv ID for its URL, or over a cell for its score. The selection is kept in the URL, so a view can be shared.
- Columns carry pseudonyms rather than names.
- Titles and authors come from [DataCite](https://datacite.org/), which holds the metadata of arXiv's DOIs; the arXiv API cannot be queried from a browser. For the rare paper whose DOI DataCite lacks, the scorer saves them in a small file beside the day's table (Tables).

## Tables

One Parquet file per announcement date and arXiv ID month:

```text
tables/
  2024-2026/                 three-year spans aligned with 1991
    2610/                    arXiv ID month (YYMM)
      2026-10-06.parquet     papers with 2610 IDs announced on Oct 6, 2026
      2026-10-07.parquet
      2026-10-07.meta.json   metadata of papers DataCite lacks, only on days with any
  index.json                 the dates with a table, by ID month: {"2610": [[2609, 30], 1, 2]}
  participants.json          who is active, and whose columns the viewer hides
  empty.json                 the dates whose table is empty: no announcement (a holiday)
```

Each file has one row per paper: an `id` column holding the part of the arXiv ID the folder does not supply (`02245` for 2610.02245; `astro-ph_234` for the old-style astro-ph/0601234, and likewise `hep-ph_234` for hep-ph/9901234 in any archive), then one `uint8` column of scores per participant. Each paper is scored once and appears in exactly one file. A date can have two files when its papers carry two ID months; `index.json` then lists the date under the other month too, in a nested list that starts with the date's own month as YYMM. This happens on the first listing of a month, announced the evening before and so holding the previous month's IDs (`"2609": [..., [2610, 1]]` for Oct 1), and when papers held for moderation are numbered in the month they are announced but belong by submission to an earlier listing (`"2610": [[2609, 22, 23], ...]`). A past listing with no paper left to score gets an empty table. On the rare day when DataCite has no record of a paper's DOI, a `<date>.meta.json` beside the Parquet file gives that paper's title, first three authors, author count and primary category, keyed like the `id` column; it is written once and never changed. Parquet opens in Python, R, Julia, MATLAB and DuckDB, for example:

```python
import pandas as pd
df = pd.read_parquet("tables/2024-2026/2610/2026-10-06.parquet")
```

## Scoring rubric

Each paper is scored from its title, authors and abstract against a participant's portfolio, by a language model (Claude), on this scale (prompt version v0; the exact prompt is `PROMPTS` in `scorer/daily_table.py`, and `config-log.json` names the version in use). Intermediate values are allowed.

| Score | Meaning |
| --- | --- |
| 100 | The subject wrote it, or would write it if it didn't exist |
| 80 | The subject coauthored it, or would coauthor it if it didn't exist |
| 60 | The subject has cited it, or probably will at some point |
| 30 | The subject probably won't cite it, but it may broaden their view |
| 0 | Worth a glance at the title, but reading the abstract would waste the subject's time |

The anchors guide the model; they are not calibrated probabilities. Most papers score low for most participants: in the tables from Sep 9 to Oct 12, 2026, 97% of the scores are below 30, and 0.3% are 60 or above. The portfolio may name people the subject knows and give scoring instructions, followed only for that subject's scores. Scores are AI-generated and can be wrong: they are pointers to papers and people, not judgments of anyone's work.

## Terms

- **Listing:** the new submissions arXiv announces for a date, Monday to Friday, at 20:00 ET the evening before; a table is named by its listing date.
- **ID month:** the year and month in an arXiv ID (2610 in 2610.02245), which names the table folders.
- **Backfill:** scoring a past listing, rebuilt from the arXiv API; the back catalog of past listings is the legacy survey.
- **Status:** active (scored on new listings and backfills), no-legacy (new listings only), paused or left (not scored).
- **Seat:** a Claude subscription account that Claude Code is signed in to, whose usage limits pace the runs.

## What participants agree to

- **Public scores.** Every score is public, in the tables and the viewer, and stays public: after a participant pauses or leaves, their column can be greyed out or hidden in the viewer, but the tables, and their git history, keep it.
- **Pseudonyms are not anonymity.** Pseudonyms keep names off the page, but a participant's own papers score near 100, so a column can be linked to the person or group behind it. Join only if you are comfortable with your relevance scores being public.
- **Portfolios.** A portfolio is visible to every participant, and is sent with every scoring request to Anthropic, the provider of the language model (Claude), under the terms of the account used. Put nothing in it that you would not share that way.

## Configuration log

[`config-log.json`](config-log.json) records each change to the scoring setup with the date it takes effect: model, effort, prompt version, input (abstract only or more), listing (new submissions only or extended), the arXiv listings followed (`listings`: astro-ph and cs.AI here) with the categories queried for past listings (`categories`), and chunk size. Entries apply by the date a table is scored, not the listing it covers. Each commit that adds or changes a table names, in its message, the portfolios commit, the configuration entry and the columns behind it, so `git log -- <table>` gives the table's history. A rescored table replaces the old one, and git keeps the earlier version.

## Repository layout

| Path | Contents |
| --- | --- |
| `DESIGN.md` | how MapHub works and why: the daily table, the viewer, the legacy survey, measurements and open questions |
| `PORTFOLIOS.md` | the portfolio format, and the portfolio-drafting prompt, which writes a first portfolio from a publication record |
| `scorer/` | `daily_table.py`, which fetches a day's papers, scores them for each participant with Claude (through Claude Code or the API) and writes the tables; `local_run.py`, the daily routine around it |
| `viewer/` | the web viewer (`index.html`, `viewer.js`, `viewer.css`, and its icon: `favicon.svg` with PNG versions) and `serve.py`, a local server for testing |
| `tables/` | the daily tables (written by the scorer; CC BY 4.0) |
| `.github/workflows/` | publishing the viewer and tables on GitHub Pages |

## Running locally

Scoring needs the private portfolios repository checked out as `maphub-portfolios/` (one folder per pseudonym, each holding `portfolio.md`), kept out of git. Kaili Cao runs MapHub from a dedicated computer holding only this repo and the portfolios; `CLAUDE.md` describes setting one up and running it. The daily routine runs through Claude Code on whichever seat `claude` is signed in to (it must be installed and signed in), with the column of every participant scored daily (status active or no-legacy; backfills: active):

```bash
python scorer/local_run.py -l    # --latest, in the morning: score the current listing, push it with any waiting backfills
python scorer/local_run.py -b    # --back: fill a missed day, else one listing further back; commit locally
python scorer/local_run.py -bb   # --back-all: keep backfilling until the usage limit stops it
python scorer/local_run.py -p    # --push: push waiting backfills when there is no new listing
# add -y (--yes) to push without confirming the commit message
```

Every mode first finishes a day left unfinished (say by the usage limit). Each backfilled day is committed locally and waits; the next `--latest` or `--push` folds the waiting commits and the new table into one commit, keeping each table's provenance. Pushing a table starts "Publish viewer", which deploys the viewer and tables on GitHub Pages.

The scorer can also be run directly, for example for a test; through Claude Code it scores only the columns named in `MAPHUB_OWN_PARTICIPANTS` (comma-separated pseudonyms; `local_run.py` sets it to every participant scored):

```bash
pip install -r scorer/requirements.txt
MAPHUB_OWN_PARTICIPANTS=<pseudonym> python scorer/daily_table.py --test   # 5 papers of each listing
python viewer/serve.py                   # viewer at http://localhost:8000/
```

## Run your own MapHub

MapHub is meant to be copied: a high-energy physics department, say, can run one for hep-ph and hep-th without joining this one. The scoring code is general; the settings, the page's wording and colors, the user agent and `CLAUDE.md` are this MapHub's own.

1. **Make your copy.** To start with a clean history, create a new repository and copy these files into it; a fork carries this MapHub's tables and history, and GitHub leaves a fork's workflows off until you enable them on its Actions tab. Delete `tables/`, which the first run recreates.
2. **Choose the listings** in `config-log.json`. Keep one entry and edit it:
   - `listings`: the listings as arXiv's RSS feeds name them, such as `["hep-ph", "hep-th"]`.
   - `categories`: the arXiv API categories that past listings are rebuilt from, one query each. Leave it out when every listing is a single category; an archive split into subcategories, like astro-ph or cond-mat, needs each one listed (see this repository's entry).
   - `effective`: on or before the day of your first run, even a test; entries take effect by the run date.
   - `model`, `effort` and `prompt`: the Claude model and effort level, and the prompt version (`v0`, in `scorer/daily_table.py`). `fallbacks` applies only to the API route. Replace `note`.
3. **Create a private portfolios repository** and clone it inside your copy as `maphub-portfolios/` (`.gitignore` leaves it out). It holds one folder per participant, named by a pseudonym and holding `portfolio.md` ([PORTFOLIOS.md](PORTFOLIOS.md): the format, and the portfolio-drafting prompt), and `participants.json`:
   ```json
   {"participants": {
     "1234": {"kind": "individual", "status": "active", "hidden": false, "since": "2027-01-04"},
     "5678": {"kind": "group", "status": "no-legacy", "hidden": false, "since": "2027-01-04"}}}
   ```
   - `status`: `active` is scored on each new listing and on backfills of past listings; `no-legacy`, new listings only; `paused` or `left`, not scored. `hidden: true` hides the column in the viewer without stopping its scoring. `kind` and `since` are for your records.
   - Pseudonyms are letters, digits, `_` or `-`, up to 40 characters; this MapHub uses random numbers. Keep who stands behind each one in the portfolios repository, for participants only.
   - Commit and push every portfolio change before a run: the scorer refuses portfolios with uncommitted or unpushed changes, and the daily routine pulls the portfolios first.
4. **Make the page yours:**
   - `viewer/index.html`: the title and description (lines 6–7), the full name in the header, the About panel (credits, contact, the GitHub link) and the footer's contact link;
   - `viewer/viewer.css`: the colors at the top (here Michigan Blue and Maize); `viewer/viewer.js`: the heatmap's color ramps, near the top;
   - the icons `viewer/favicon.svg`, `favicon-32.png` and `apple-touch-icon.png`;
   - this README, and `CLAUDE.md`, which is this MapHub's operations file (its maintainer, paths and portfolios URL); rewrite it for yours, or delete it;
   - `MAPHUB_USER_AGENT` in `.env`: how your MapHub names itself to arXiv and DataCite, with a contact, such as `MapHub/0.1 (+https://github.com/<you>/<repo>; you@example.edu)`.
5. **Publish:** in your repository's Settings > Pages, set the source to GitHub Actions; each push of tables then deploys the viewer at your Pages address.
6. **Run it** each morning after arXiv's 20:00 ET announcement, with `python scorer/local_run.py -l` (Running locally). It needs either Claude Code (`claude` on the search path, recent enough for `--json-schema`), signed in to a Claude subscription whose terms allow this use (check them), whose usage limits then pace the runs: for scale, an ordinary day of about 200 papers scored for five participants took about 3% of one subscription's weekly limit (Oct 2026); or the Claude API, with `ANTHROPIC_API_KEY=...` in `.env` and `--via api`. `CLAUDE.md` shows a short `mh` command for these runs; its path assumes the clone is at `~/maphub-public`.
7. **Optionally, go back in time:** `local_run.py -b` scores one earlier listing for participants with status `active`, and `-bb` keeps going until the usage limit stops it, back as far as you choose (arXiv began in 1991).

Questions are welcome: kailicao@umich.edu.

## License

The code and documents are under the MIT License ([LICENSE](LICENSE)); the tables (`tables/`) are under [Creative Commons Attribution 4.0](https://creativecommons.org/licenses/by/4.0/) (CC BY 4.0).
