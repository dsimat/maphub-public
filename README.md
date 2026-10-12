# MapHub: MIDAS arXiv Preprint Hub

**Viewer: <https://kailicao.github.io/maphub-public/>**

Each arXiv announcement day, MapHub scores the new submissions to astrophysics (astro-ph) and artificial intelligence (cs.AI) for relevance to each participant, and shows the result as a heatmap: one row per paper, one column per participant, and in each cell a score from 0 to 100. A large language model assigns the scores by reading each paper's title, authors and abstract against a short research portfolio that every participant, individual or group, writes and keeps up to date. Selecting yourself shows which new papers matter most to you; selecting a paper shows who is most likely to answer questions about it.

MapHub is a carpentry of the [Eric and Wendy Schmidt AI in Science Postdoctoral Fellowship](https://midas.umich.edu/training/postdoctoral-programs/schmidt-ai-in-science-postdoctoral-fellowship/) at the Michigan Institute for Data and AI in Society (MIDAS), led by Kaili Cao with teammates Kyle Heyblom, Eva Albalghiti and Dimitrios Simatos. Participants are researchers and research groups, shown by pseudonym; to join, email kailicao@umich.edu. Scores are public and stay in the tables after a participant pauses or leaves; the viewer can hide their column. This repository holds its code, configuration log, daily tables and web viewer, all public; portfolios are shared only among participants.

## Viewer

- The base URL shows the latest day; each day also has its own URL, such as `https://kailicao.github.io/maphub-public/2026-10-06`. Weekends have no arXiv listing and no table. arXiv holidays have no listing either, but a backfilled holiday gets a table of the submissions it would have held. The arrows, the date box and the calendar, where days with a table are highlighted, move between days and keep a selected participant.
- By default, papers are ranked by their average score over the participants shown, and participants by the rank correlation (ρ) of their scores with the average of the others'. Select a participant (column name) to sort papers by relevance to them and participants by similarity to them. Select a paper (title) to sort papers by similarity to it and participants by its relevance to them. Similarity is the correlation (r) of that day's scores.
- **Top papers** (`<base>/top`) lists one participant's highest-scored papers over a range of listing dates (by default the last 30 listings, top 20), ties going to the newer listing; each listing date opens that day's heatmap. Its URL keeps the choice, as in `<base>/top#p=52454&from=2026-09-09&to=2026-10-12&n=20`. Selecting a participant in the heatmap offers a link to their top papers up to that day.
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
```

Each file has one row per paper: an `id` column holding the part of the arXiv ID the folder does not supply (`02245` for 2610.02245; `astro-ph_234` for the old-style astro-ph/0601234), then one `uint8` column of scores per participant. Each paper is scored once and appears in exactly one file. A date can have two files when its papers carry two ID months; `index.json` then lists the date under the other month too, in a nested list that starts with the date's own month as YYMM. This happens on the first listing of a month, announced the evening before and so holding the previous month's IDs (`"2609": [..., [2610, 1]]` for Oct 1), and when papers held for moderation are numbered in the month they are announced but belong by submission to an earlier listing (`"2610": [[2609, 22, 23], ...]`). A past listing with no paper left to score gets an empty table. On the rare day when DataCite has no record of a paper's DOI, a `<date>.meta.json` beside the Parquet file gives that paper's title, first three authors, author count and primary category, keyed like the `id` column; it is written once and never changed. Parquet opens in Python, R, Julia, MATLAB and DuckDB, for example:

```python
import pandas as pd
df = pd.read_parquet("tables/2024-2026/2610/2026-10-06.parquet")
```

## Configuration log

[`config-log.json`](config-log.json) records each change to the scoring setup with the date it takes effect: model, effort, prompt version, input (abstract only or more), listing (new submissions only or extended) and chunk size. Entries apply by the date a table is scored, not the listing it covers. Each commit that adds or changes a table names, in its message, the portfolios commit, the configuration entry and the columns behind it, so `git log -- <table>` gives the table's history. A rescored table replaces the old one, and git keeps the earlier version.

## Repository layout

| Path | Contents |
| --- | --- |
| `scorer/` | `daily_table.py`, which fetches a day's papers, scores them for each participant with Claude (through Claude Code or the API) and writes the tables; `local_run.py`, the daily routine around it |
| `viewer/` | the web viewer (`index.html`, `viewer.js`, `viewer.css`, and its icon: `favicon.svg` with PNG versions) and `serve.py`, a local server for testing |
| `tables/` | the daily tables (written by the scorer) |
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

The scorer can also be run directly, for example for a test; through Claude Code it scores the columns named in `MAPHUB_OWN_PARTICIPANTS`:

```bash
pip install -r scorer/requirements.txt
MAPHUB_OWN_PARTICIPANTS=<pseudonym> python scorer/daily_table.py --test   # 5 astro-ph and 5 cs.AI papers
python viewer/serve.py                   # viewer at http://localhost:8000/
```
