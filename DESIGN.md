# MapHub design

How MapHub works and why it works that way. The [README](README.md) covers the viewer and the tables; this document is for anyone who maintains MapHub, or builds a MapHub of their own for other arXiv listings (README, Run your own MapHub). Dates record when something was measured or decided.

## Summary

Each announcement day brings hundreds of new papers to arXiv, and the colleague best placed to explain one is often close by but unknown. MapHub turns each day's new submissions to a few arXiv listings (here astrophysics, astro-ph, and artificial intelligence, cs.AI) into a heatmap: one row per paper, one column per participant, and in each cell a relevance score from 0 to 100. A large language model assigns the scores by reading each paper's title, authors and abstract against a short research portfolio that every participant, individual or group, writes and keeps up to date. To spare usage, it first scores every paper for all participants at once against outlines of their portfolios, and reads a full portfolio only for the papers that pass this first look.

Participants can sort the heatmap in two ways. Selecting themselves shows which new papers matter most to them; selecting a paper shows whose portfolios it matches best, a starting point for finding someone to ask about it. The project rests on a simple principle: papers come with people, and a paper is worth more when someone nearby can discuss it. AI does the first pass of reading, while understanding still comes from reading and conversation.

The code, daily tables and web viewer are public; portfolios are shared only among participants. The same scoring also goes back through earlier arXiv listings, building a back catalog: the legacy survey.

## Overview

MapHub, the MIDAS arXiv Preprint Hub, was built as a team project (a "carpentry") of the [Eric and Wendy Schmidt AI in Science Postdoctoral Fellowship](https://midas.umich.edu/training/postdoctoral-programs/schmidt-ai-in-science-postdoctoral-fellowship/) at the Michigan Institute for Data and AI in Society (MIDAS), led by [Kaili Cao](https://kailicao.github.io/) with teammates Kyle Heyblom, Eva Albalghiti and Dimitrios Simatos. It grew out of Kaili Cao's own literature workflow and extends it from one person to a community. The name works two ways: it is short for MIDAS arXiv Preprint Hub, and its deliverables are daily heatmaps. Each morning a participant sees the new papers ranked by relevance to them, then finds who among the participants could answer questions about the papers they care about. Because scoring is centralized, no participant has to go through the new submissions on their own.

- **One heatmap per announcement day.** Weekends have no listing, so they have no heatmap; an arXiv holiday has no announcement: met by the morning run, it gets an empty table, which the viewer shows as a day without an announcement; reached by a backfill, it gets a table of the submissions it would have held (Stages).
- **Rows:** that day's new submissions to the configured listings. The minimal version leaves out cross-lists and replacements.
- **Columns:** participants, both individuals and groups.
- **Cells:** each paper's relevance to each participant, scored from that participant's portfolio on a 0–100 scale (README, Scoring rubric).

New listings are scored each morning by the maintainer, through Claude Code (Local runs), and backfills go back a day at a time. The maintainer also handles the human side by hand: reviewing portfolio pull requests, helping new participants, answering questions. Publishing the viewer runs on its own.

**Listings.** This MapHub follows astro-ph and cs.AI because it is centered on an astronomer at an AI institute: a bridge between colleagues from many disciplines, who all work on AI, and the astronomy community, especially where they share methods such as remote sensing, data analysis and theoretical modeling. The listings are a setting in the configuration log (`listings` and `categories`), so a MapHub for other fields changes them there; the rest of what is this MapHub's own (the page's wording and colors, the user agent, the operations file) is listed in the README, Run your own MapHub.

## Participants and portfolios

Participants are individuals and groups or collaborations, each shown by a pseudonym. Each participant's portfolio is the only input that scores papers for them, in the format of [Writing a portfolio](PORTFOLIOS.md). The relevance score doubles as a proxy for answering capability: the participants a paper is most relevant to are taken as the ones best able to answer questions about it. They are not necessarily the best, but relevance is the only measure available; because each portfolio is reviewed by a human expert, the proxy should be reasonable, and it improves as portfolios are revised; it has not yet been checked against participants' own judgments (Open questions).

- **First portfolios.** Individuals either run the portfolio-drafting prompt (in [Writing a portfolio](PORTFOLIOS.md)) themselves or send the maintainer their CV to run it; the maintainer asks each person which they prefer. For groups and collaborations, the maintainer drafts an initial portfolio and discusses it with them before or after it goes into use.
- **Length.** About 1,500 words for an individual and about 3,000 for a group or collaboration.
- **Outline.** The first pass of scoring reads each portfolio's outline, which the scorer cuts at run time and never stores: the title, the Overview, and each topic's heading with its Next and Expertise lines, about 500 words for an individual and 700 for a group. The format in [Writing a portfolio](PORTFOLIOS.md) keeps these parts informative, with guides rather than rules: an Overview of about 100–150 words, and one line each for a topic's plans and expertise. Plans belong in a portfolio, since it is used to find papers of interest.
- **Names and instructions.** There is no separate author bonus. Participants who want one add names of people they know, and they may add scoring instructions; both count toward the word limit. Since each full-pass request scores one participant, they affect only that participant's column; they stay out of the outline, so the outline pass, which serves everyone at once, never sees them. Instructions that inflate someone's scores would also make them look like a better person to ask, so the pull-request check watches for them.
- **Safeguards.** Each participant has a folder in the private portfolios repository holding their portfolio. Participants have read access to that repository only: they cannot push, and propose a change by forking the repository and sending a pull request, which the maintainer reviews (own folder only, length) and merges; emailing a new version works too. A GitHub bot to check pull requests is not set up.
- **Privacy.** Every participant can read every portfolio, since GitHub grants read access per repository, not per folder or branch. A participant who needs theirs private uses the own-repo option (Extension plans). Portfolios and the papers' abstracts are also sent to Anthropic, the provider of the language model, with every scoring request, under the terms of the account used (README, What participants agree to).
- **Participant changes.** The portfolios repository holds `participants.json`, each participant's current status (active: scored on each new listing and in the legacy survey; no-legacy: new listings only; paused or left: not scored) and whether their column is hidden, and `CHANGELOG.md`, a dated log of joins, status changes and renames; the two are kept in step. The scorer scores participants with status active or no-legacy on new listings, and only those with status active on past listings; a portfolio without an entry is skipped with a warning. Data files are never changed when someone pauses or leaves: their past columns stay in the public tables, which participants are told when they join. Instead, the scorer publishes the active and hidden lists in `tables/participants.json`, and the viewer leaves out hidden columns (also from the similarity sorting) and greys out visible columns of participants who are no longer active.

## Repositories

Two repositories, one inside the other:

```text
maphub-public/          public: code, configuration log, daily tables and viewer
  maphub-portfolios/    private, shared with all participants, who have read access
```

`.gitignore` leaves out the nested portfolios repository. This repository holds `scorer/` (the daily table script), `viewer/` (the web viewer), `tables/`, `config-log.json` and the GitHub workflow that publishes the viewer; it also ignores `.work/`, where runs save papers and scores for resuming, and `.env`, for local settings such as an API key for the `api` route. The scorer needs Python with `anthropic` and `pyarrow`.

## Daily table

The script `scorer/daily_table.py` takes an announcement date (by default the current listing) and produces that day's table.

1. **Fetch.** The arXiv API cannot be queried by announcement date, so the day's new submissions come from arXiv's RSS feed for the configured listings (astro-ph and cs.AI here), which shows only the current listing and marks each paper's announce type (keeping "new" leaves out cross-lists and replacements). Titles, authors and abstracts then come from the API in one query; the minimal version uses nothing more. The papers are saved locally, so a rerun on a later day still has them.
2. **Score.** Two passes (prompt v1, since Oct 15, 2026; Two-pass test below). The outline pass sends one Claude request per chunk of about 50 papers: the outline prompt below, the chunk's papers, and every participant's outline (the joint outline), and gets back a score for each paper and participant. The full pass then takes, for each participant, the papers whose outline score is 10 or more (the `threshold` in the configuration log) and scores them again with the scoring prompt below and that participant's full portfolio, in chunks of about 50; the other cells keep their outline scores. Until Oct 14, 2026 (prompt v0), every paper went through the full pass: one request per participant per chunk, which repeats the short abstracts rather than the long portfolios, roughly 7× fewer input tokens than one request per paper. On the API route, the papers block is identical across participants, so it is cached; with a single participant the script skips caching, since writing the cache costs 25% extra and pays off only when others read it. Requests through Claude Code, the default, barely share that cache (Oct 2, 2026: 791k tokens written, 78k read), an open question; the outline pass sidesteps it by sending each chunk's papers once for everyone.
3. **Write.** The scores become one Parquet file per arXiv ID month (Table format), and `tables/index.json` gains the date so the viewer can find it. The scorer then asks DataCite whether it has every paper's DOI, and saves the metadata of any it lacks beside the table (Web viewer). Papers already in another date's table are left out first, so each paper is scored once and only once; a run that writes tables also requires maphub-portfolios to be committed and pushed.
4. **Resume.** If a run is interrupted, it continues where it stopped: each request's scores are saved as they arrive (the outline chunks, then each participant's full-pass chunks), a rerun skips what is saved, and the Parquet files are written only once the table is complete. A reply that skips a few papers keeps the scores it has, and the retry asks only for the skipped ones (Sep 29, 2026: one chunk lost a paper three times running and stopped the run, before this fix).

**Kinds of run.** Without a date, the scorer takes the current listing and stops at once if it is already in the tables, so repeated runs after a late announcement cost nothing. A past date is the legacy survey (Stages), scoring participants with status active; `--daily DATE` scores it as a daily run instead (no-legacy included), which `local_run.py` uses to finish a morning listing interrupted by the usage limit. `--meta-only DATE` checks an existing table against DataCite and writes its metadata fallback, without scoring. `--test` scores 5 papers of each listing into the work dir. `--batch` (the API route's Message Batches) works only with prompt v0.

**Local runs.** The maintainer runs the scorer by hand through Claude Code with `scorer/local_run.py`, from a dedicated computer that holds only this repository (with the portfolios inside), set up as `CLAUDE.md` describes. It scores every participant scored daily (status active or no-legacy) on new listings, and every participant with status active on backfills, on whichever seat `claude` is signed in to there. A scheduled GitHub workflow scored the listings through the API until Oct 8, 2026; it was retired because its runs could start hours late (6 h 40 min on Oct 7) or not at all, and because a Claude Code seat made the API unnecessary. `mh` is a short command for `local_run.py` (`CLAUDE.md`): `mh -l` (`--latest`) scores the current listing in the morning and pushes it; `mh -b` (`--back`) scores one listing as usage allows: first any gap, a weekday between the earliest and latest tables without a table (a morning run missed), otherwise the weekday before the earliest table; `mh -bb` (`--back-all`) keeps backfilling until the usage limit stops it, and `-y` (`--yes`) pushes without asking to confirm the message; `-bb` also spends the seat's weekly limit, so the maintainer chooses when to run it. Every mode first finishes a day left unfinished in `.work/`, so a day stopped by the usage limit needs no manual step; a backfilled listing with nothing left to score gets an empty table, so that every weekday has one. Backfills so far (back to Sep 9, 2026) score every participant, all with status active, so the recent tables are complete. To keep one commit a day, a backfill is committed locally and waits: the next `--latest` (or `--push`, when there is no new listing) folds the waiting commits and the new table into one commit, whose message keeps each table's provenance under its date. If a table was pushed meanwhile from another computer, the pull rebuilds `index.json` from the tables present.

**Two ways to send requests** (`--via`). `claude-code`, the default, uses Claude Code's print mode (`claude -p`) on the seat `claude` is signed in to, with tools and settings off and any API key withheld; it scores only the columns listed in `MAPHUB_OWN_PARTICIPANTS`, which `local_run.py` sets to every participant scored daily (status active or no-legacy); on a past date the scorer then keeps those with status active. `api` uses the Claude API with a key from the environment; no key is kept since Oct 10, 2026, and the route can return when needed. Writing a column into an existing table keeps the other participants' columns, and a run whose paper list differs from the table's is refused.

**Configuration log.** `config-log.json` in maphub-public records each change with the date it takes effect: model, effort, prompt version, input (abstract only or full text), listing (new submissions only or extended), the arXiv listings followed (`listings`, as named in arXiv's RSS feeds, such as astro-ph or cs.AI) and the categories past listings query in the arXiv API (`categories`: an archive split into subcategories, like astro-ph, needs each subcategory listed), chunk size, the two-pass threshold and similar settings. Entries apply by the date a table is scored, not the listing it covers. The tables carry no provenance themselves: each commit that adds or changes a table names, in its message, the maphub-portfolios commit, the configuration entry, the columns scored and the papers left out as already scored, so `git log -- <table>` gives the table's history even when a file holds columns from several runs or is corrected later. The scorer writes this message to `.work/commit-message.txt`, and `local_run.py` commits with it. A rescored table replaces the old one, and git keeps the earlier version. Changes should be rare. The first entry (Oct 5, 2026) is Claude Opus 5.5 at low effort with prompt v0. Trial runs of other models (`--model` and `--effort`, allowed only with `--test`) leave the log unchanged and write to their own folders.

The tests and measurements below were made on this MapHub's listings and participants, in astronomy and AI. The method does not depend on the field; the numbers carry over as orders of magnitude, scaled by the size of the listings and the number of participants.

These are early impressions from small samples (10 papers in the first test, one 171-paper day in the full-day check, five participants), not benchmarks.

**First test (Oct 6, 2026).** One individual's portfolio (1,496 words, about 4,100 tokens; arXiv IDs and numbers take many tokens) against 5 astro-ph and 5 cs.AI papers: about 570 tokens per paper, 2–3 s per request, and about $0.05 on Opus 5.5. The same 10 papers on three models:

| Paper | Opus 5.5 | Sonnet 5.5 | Haiku 4.5 |
| --- | --- | --- | --- |
| Asymmetric beam treatment for CMB maps | 35 | 22 | 30 |
| Training numerical intelligence (LLMs) | 15 | 6 | 60 |
| Galaxy environments with manifold learning | 12 | 8 | 30 |
| Lensed clumps at z = 1–4 | 10 | 6 | 30 |
| The other six | 1–3 | 1–3 | 0 or 30 |

Sonnet 5.5 kept Opus's order at half the price. Haiku 4.5 mostly used the anchor values and put an LLM paper first, so it is ruled out.

**Full-day check (Oct 5–6, 2026).** All 171 papers of the Oct 5 listing, scored for all five participants on both models: Sonnet 5.5 ranked them much like Opus 5.5 (rank correlation 0.89–0.96 per column, 3–5 of each top five shared) but compressed the top of the scale (an individual's best paper: Opus 45, Sonnet 22) and missed cross-topic matches that MapHub is meant to find (benchmark stars for a stellar-modeling collaboration: 62 against 22; a CMB beam-treatment paper for a weak-lensing team: 30 against 0). For a paper listed in a participant's own portfolio (2610.03882), Opus gave 95 and Sonnet 72 at low effort and 55 at medium; medium effort added no thinking, and the two Sonnet runs differed only by chance. Decision: Opus 5.5 at low effort. If cost forces a change later, a prompt that pins papers listed in a portfolio near 100 is the first thing to try.

**Two-pass test (Oct 14, 2026).** Most cells score low: on the Oct 14 listing, 97% of cells were below 30, and 90% of papers were below 30 for every participant. Scoring those precisely is wasted usage, so a cheap first pass picks out the papers worth a full read. Measured on the Oct 14 and Sep 8 listings (748 papers, five participants, scored with v0 the same day), with costs in API-equivalent dollars through Claude Code:

- **Each participant's outline in their own request** saves little. The papers are about 80% of a request (about 32,000 tokens per 50 papers, against 4,000–8,000 for a full portfolio), so such a first pass costs about 85% of a full run, and the whole two-pass run 89–108%.
- **All outlines in one request** (the joint outline) sends the papers once: the outline pass cost about $5.4 against about $24 for the v0 run. With a full pass for outline scores of 10 or more (13% of cells), the run comes to about 36% of v0, and it missed 2 of the 137 cells that v0 scored 30 or more (both scored 30, with outline score 5) and none of the 56 at 40 or more. A threshold of 5 missed nothing at about 46%; one of 15 missed 6 cells at about 31%. Outline scores come in steps of 5 and run about half as high as full scores, so they serve only as a filter.
- **The Overview matters.** Outlines made only of headings, bold phrases and Expertise lines missed 20 of one participant's 52 cells at 30 or more, papers on a field whose interest was written only in the Overview.
- **The outline's form matters little.** On a stratified subset of 150 papers, outlines of headings with each topic's opening sentence, its Expertise line, or both (with the Overview, about 1,700–2,900 words for five participants) missed 0–2 of 101 cells at 30 or more and cost 33–38% of v0, the same within noise. The format keeps the Expertise line, which portfolios already had, and adds a Next line so that plans show in the outline.
- **End to end** (50 papers from the subset, with the portfolios in the new format): the outline pass missed none of the 28 cells v0 scored 30 or more, and the full pass came within 10 points of v0 for 95% of the 58 cells it rescored (mean −2.5 points).

Decision: two passes with the joint outline and a threshold of 10 (prompt v1).

**Budget.** Measured on Opus 5.5 at low effort for five participants: about $1.50 for the 171 papers of Oct 5 (about $1.60 with retries) and about $2.50 for the 345 of Oct 6; Tuesday listings include the weekend and run larger. Each further participant adds about 30% of the first one's cost, since only the first writes the cached papers block (about $0.17 against $0.05 per chunk of about 43 papers); ten participants on a typical day of about 200 papers come to roughly $2.90 a day, or about $60 a month over 21 announcement days. Portfolios, re-sent with every chunk, are a large share; chunks of 100 papers would cut it if scoring holds up on longer lists. Sonnet 5.5 would cost about half (Full-day check). API use is billed separately from Claude subscriptions; MapHub ran on the API until Oct 8, 2026 and on a Claude Code seat since (Local runs), so these figures matter for anyone running it through the API. With two passes (prompt v1), the Oct 14 and Sep 8 listings came to about a third of v0's cost (Two-pass test): the outline pass sends each chunk's papers once for all participants, so a further participant adds mainly their outline (about 700–1,000 tokens per request) and the full pass over roughly a tenth of the papers.

**Scoring prompt (draft).** Adapted from the reading prompts of Kaili Cao's own literature workflow, and written for any subject, person or group. The instructions and papers come first so that this prefix is identical, and cached, across participants:

```text
You score new arXiv papers for relevance to the subject of a research portfolio: a researcher, a group or a collaboration. The papers come first, then the portfolio.

Score each paper from its title, authors and abstract on this scale. Intermediate values are allowed.
   100 = the subject wrote it, or would write it if it didn't exist
   80 = the subject coauthored it, or would coauthor it if it didn't exist
   60 = the subject has cited it, or probably will at some point
   30 = the subject probably won't cite it, but it may broaden their view
   0 = worth a glance at the title, but reading the abstract would waste the subject's time
The portfolio may name people the subject knows and give scoring instructions; follow them only for this subject's scores.

Return only JSON, one object per paper in the order given: {"id": "<arXiv ID>", "score": <integer 0-100>}. No explanations.

<papers>
[<arXiv ID>] <title>
<authors>
<abstract>
...
</papers>

<portfolio>
<the participant's portfolio>
</portfolio>
```

The outline pass (prompt v1) uses the same scale, asking for every subject's score for each paper:

```text
You score new arXiv papers for relevance to the subjects of several research portfolios: researchers, groups or collaborations. The papers come first, then the portfolios, each labelled with the subject's pseudonym. The portfolios are short outlines; judge each subject only by their own portfolio, independently of the others.

[the scale, as above]

Return only JSON, one object per paper in the order given: {"id": "<arXiv ID>", "scores": {"<pseudonym>": <integer 0-100>, ...}}, with every pseudonym for every paper. No explanations.

<papers> ... </papers>

<portfolio subject="<pseudonym>"> <outline> </portfolio>
...
```

The full pass uses the scoring prompt above, as every request did under prompt v0.

**Running it:**

1. **Test by hand first** (First test, Full-day check).
2. **Run it locally every morning.** `scorer/local_run.py` scores the listing through Claude Code and pushes the table (Local runs); the push starts the workflow "Publish viewer", which deploys the viewer and tables on [GitHub Pages](https://docs.github.com/en/pages). A scheduled GitHub workflow did the scoring through the API, from its first run on Oct 6, 2026 to the Oct 8 listing, and was retired: scheduled runs could start hours late or not at all.
3. **No keys to keep.** Claude Code uses the signed-in seat. The Claude API key, used for the API route and the Budget measurements, was disabled on Oct 10, 2026; the API route needs a new one if it returns.
4. **Time it after arXiv's evening announcement.** arXiv announces each listing at 20:00 ET; a run any time after that scores it, and a run on a listing already in the tables stops at once at no cost.
5. **Send requests one at a time, with prompt caching** (the API route, prompt v0; under v1 the outline pass serves everyone in one request, and each full-pass request belongs to one participant, so there is nothing to share). The Message Batches API costs half price, but caching inside a batch is best-effort. With one-at-a-time requests in chunk-major order, every participant reads the cache the first one wrote, which costs less than a batch without cache hits; 20 requests (five participants, four chunks) took about 2 minutes on Oct 5. The script keeps `--batch` for later tests.


**Table format.** The format holds for any arXiv listing; the examples come from astro-ph. Each table is named by its listing date, Monday to Friday, as shown on arXiv's listing pages and RSS feed; arXiv announces each listing at 20:00 ET the evening before, and the submission deadline is 14:00 ET. Tuesday's table covers everything submitted from Friday afternoon to Monday afternoon, weekend included, in one list. Each row stores only the part of the arXiv ID that the file's folder does not already imply; everything else about a paper is on arXiv. Scores are stored as uint8, since 0–100 fits in one byte.

| Papers submitted | Full ID | Stored in the row |
| --- | --- | --- |
| From January 2015 | 2510.01234 | 01234 (5 digits) |
| April 2007 to December 2014 | 1410.1234 | 1234 (4 digits) |
| Before April 2007 | astro-ph/0601234 | astro-ph\_234: archive, underscore, 3 digits; the archive is needed because each one numbered its papers separately, and an underscore replaces the slash |

An ID's month (YYMM) follows the announcement evening, not the listing date. The listing dated Oct 1, 2026, announced the evening of Sep 30, holds 2609 IDs, while papers submitted after that day's 14:00 ET deadline already get 2610 IDs. Papers held for moderation take the month of their eventual announcement: 2610.02245 was submitted Sep 30 and listed Oct 5. So folders go by ID month and files by listing date. Each new submission lands in exactly one file, and the folder supplies the month the row leaves out. Folders are grouped in three-year spans aligned with arXiv's first year, 1991:

```text
tables/
  1991-1993/
  …
  2024-2026/
    2609/
      2026-09-30.parquet
      2026-10-01.parquet    announced the evening of Sep 30
    2610/
      2026-10-02.parquet
      2026-10-07.meta.json  papers DataCite lacks (rare)
  index.json                the dates with a table, by ID month
```

A listing normally holds one ID month, but the format allows two files per date, and the viewer merges them. A static website cannot list folders, so `index.json` lists, for each ID-month folder, the dates with a table in it: a day of the folder's own month as a number, and the days of another month in a nested list that starts with that month as YYMM (`{"2609": [29, 30, [2610, 1]], "2610": [[2609, 22, 23], 2, 5, 6]}`), about 3 bytes per date. Two things give a date files in two ID-month folders: the first listing of a month, announced the evening before, holds the previous month's IDs (`"2609": [..., [2610, 1]]`, the Oct 1 listing); and papers held for moderation are numbered in the month they are announced but placed by a rebuilt listing on the date of their submission (`"2610": [[2609, 22, 23], ...]`, a few late-September papers with October IDs). `participants.json` beside it holds the viewer's active and hidden lists, and `empty.json` the dates whose table holds no paper (no announcement), rebuilt from the tables whenever one is written, which the calendar colors. Files are Parquet: one row per paper, an ID column, then one uint8 column per participant pseudonym. Parquet opens in Python, R, Julia, MATLAB and DuckDB, so colleagues can analyze the tables with their own tools. Next to the tables, the daily job will keep the long-term correlation table (Web viewer); it is not built yet.

## Web viewer

The viewer is a web page at a fixed URL, <https://kailicao.github.io/maphub-public/>, that people can bookmark, and they can also download it. It shows one day's table as a heatmap. GitHub Pages publishes it together with the tables, and `viewer/serve.py` serves the same layout locally for testing.

- **Dates.** Like [APOD](https://science.nasa.gov/apod/), the base URL shows the latest day with an announcement, and each day has its own URL; with no title to add, the date alone names it (e.g. `<base>/2026-10-02`). Pages serves a copy of the viewer for these paths, and the viewer reads the date from the URL. A date without a table, such as a weekend or a day not yet announced, shows a message. Arrows step through the days that have tables; a typed date takes effect on Enter. A calendar of the viewer's own highlights the days with a table; other weekdays are muted and weekends faint, since a browser's date picker cannot mark days. Moving to another day keeps a selected participant.
- **Rows.** When a day loads, the viewer shows each row's title and first three authors. The arXiv API sends no CORS header, so a browser cannot query it; the viewer instead asks DataCite, which holds the metadata of arXiv's DOIs (10.48550/arXiv.<ID>), with one search per 100 papers. DataCite's search index can lag behind new DOIs, so papers it misses are fetched one by one. Rarely, arXiv never registers a paper's DOI (2610.07148, listed Oct 7, 2026, was the first). For these, the scorer checks DataCite after writing a table and saves each lacking paper's title, first three authors, author count and primary category in `<date>.meta.json` beside the Parquet file; the viewer reads that file only when DataCite leaves gaps. A file is written once per day and ID month with lacking papers and never changed, so git stores each just once; titles otherwise stay out of the repo. Hovering over an arXiv number for about 0.5 s pops up the paper's arXiv URL, built from the ID; clicking it opens the paper.
- **Columns.** Columns carry pseudonyms rather than real names. The tables, scores included, are public in maphub-public; participants know each other and can tell who is who. Columns of participants who paused or left are greyed out, and hidden ones are left out (Participant changes).
- **Authors.** Each of the first three author names links to arXiv's author search in the paper's archive, by family name and initials, as arXiv's own listings do.
- **Page.** The header spells out the full name on a Michigan Blue bar with a Maize rule; the heatmap shades run from near-white to Michigan Blue and selections are marked in Maize. A folded About panel explains the page, the score scale and the participants, credits the fellowship and the team, and says that MapHub is under development and its AI-generated scores may be inaccurate; the footer repeats that and gives a contact email. The table fits the window's width, and on narrow screens the arXiv number moves under the title.

A day opens with papers ranked by their average score over the visible columns, so papers that matter to many participants come first, and participants ranked by the rank correlation (Spearman's ρ) of their scores with the average of the other participants', shown next to each column. A user can reorder the heatmap in two ways, usually in this order:

| Select | Papers sorted by | Columns sorted by |
| --- | --- | --- |
| A participant | Relevance to that participant | Similarity to that participant |
| A paper | Similarity to that paper | Relevance of that paper to each participant |

Similarity is the correlation (r) of that day's scores, shown next to each column or row: two participants are similar when their columns match, and two papers when their rows match. The selection is kept in the URL, so a view can be shared.

**Top papers.** A second page, `<base>/top#p=<who>&also=<others>&from=<date>&to=<date>&n=<n>`, lists one participant's n highest-scored papers over a range of listing dates: by default the past calendar month up to the latest table and the top 50, ties going to the newer listing, then the arXiv ID. Other participants can be shown beside, with their scores for the same papers, to compare. It asks DataCite only for the papers shown, so a long range stays quick. Each paper's listing date opens that day's heatmap, sorted for the participant, and a participant selected in the heatmap links to their top papers up to that day. Ranking by a set of participants (their average score) may follow. Anyone who wants other trends over time downloads the tables and analyzes them on their own.

Some participants have no column on a given day: they joined later, or chose status no-legacy. For them, the daily job keeps a long-term correlation table between participants, built from every day on which both have scores. A user can still select a missing participant: other participants are then sorted by long-term correlation with them, and papers by a weighted sum of the present participants' scores, marked as an estimate. To start, each present participant's weight is their long-term correlation with the missing one, positive correlations only; weights fitted by regression are a later option. The viewer therefore reads one day's table plus the correlation table. Missing participants are not supported yet.

## Stages

1. **Build and tune.** Several months of tuning the method and recruiting participants. Started Oct 2026: the script, viewer and the first five portfolios are done, and Opus 5.5 at low effort was chosen after the full-day check. Two-pass scoring (prompt v1) replaced one pass from Oct 15, after the two-pass test.
2. **Daily operation.** The maintainer scores each listing every morning on a Claude Code seat, and the viewer updates on its own. Started with the Oct 7, 2026 listing, on the API until Oct 8 and on a Claude Code seat since; tuning goes on alongside.
3. **Legacy survey: the back catalogs.** MapHub goes back through earlier listings, from 2026 backwards (the Sep 9, 2026 listing by Oct 12) to arXiv's first listing in 1991 (a goal: by Oct 12, 2026, it reached Sep 9, 2026), and scores them against the portfolios of every participant with status active. The result is a back catalog of scored tables that participants go through on their own. Building does not wait for reading, and a portfolio update does not rescore existing tables: they change only in a deliberately launched second pass. Cross-lists are to be settled before the survey goes far back. Every participant is in the legacy survey unless they choose status no-legacy, which keeps them to new papers only (since Oct 11, 2026; until then the survey was opt-in, and every participant was then included).
   - **Past listings.** The RSS feed shows only the current listing, so a past listing is rebuilt from the arXiv API by submission time: the window between the two 14:00 ET deadlines it covers (Tuesday's listing: Friday 14:00 to Monday 14:00), keeping papers whose primary category falls under a configured listing (one API query per configured category). Against arXiv's own lists, 341 of 345 papers matched on Oct 6, 2026, 256 of 261 on Sep 15 and 304 of 326 on Aug 4; every difference was a paper held for moderation, which a rebuilt listing dates by submission rather than announcement. Since the windows tile time, every paper is scored at least once, at worst on a neighbouring date; and since a run leaves out papers already in another date's table, at most once. A held paper claimed by both a rebuilt and an RSS listing stays with whichever table was written first (two such papers, 2610.03840 and 2610.03850, were in both the Oct 5 and Oct 6 tables and were removed from Oct 5, a one-time correction). When the arXiv API is busy (429 Too Many Requests or 503 Service Unavailable, as on Oct 8, 2026), the scorer waits 30 s, 60 s and then 2 min between tries, or as long as the reply's Retry-After asks, up to five tries; other failures get three quick tries. Holidays are not modelled. arXiv skips some announcements (Monday Sep 7, 2026 meant no listing dated Tuesday Sep 8), and the next listing then holds both days' submissions; a rebuilt holiday instead gets a table of its own, so its papers are dated one listing early. On a holiday morning the current listing is empty: the scorer writes an empty table for it, with "No announcement" in the commit message, and the viewer shows the day as one without an announcement. Recording holidays was considered and dropped (Oct 2026): arXiv's catch-up pages, which show whether a date had a listing, reach back only 90 days, and its holiday schedule is published only for recent years, so a record would cover a few recent dates and treat the rest differently. Consistency matters more, and what matters most holds either way: the windows tile time, so no paper is lost or scored twice. The deadline is verified only for 2026. Old-style IDs (before April 2007) are stored as `astro-ph_234` (Table format).
   - **Cost.** Through Claude Code, each column runs on the signed-in seat, so the seat's usage limits pace the backfills (an ordinary day of about 200 papers for five participants took about 3% of a seat's weekly limit with prompt v0, Oct 2026, and should take about a third of that with two passes); through the API, see Budget. An estimate for the API route (Oct 2026): on Sonnet 5.5, one year of astro-ph listings (about 15,000 papers; other listings scale with their size) would cost roughly $27 for the first column and $7 for each further one, since they share the cached papers; the two passes of prompt v1 cut this to about a third (Two-pass test), and chunks of 100 might cut more.

**Continuity.** MapHub depends on one maintainer and one subscription. If the daily runs stop, the published tables and the viewer remain, and anyone can resume scoring from a copy (README, Run your own MapHub).

## Extension plans

After the minimal version works:

- **Cross-lists.** Add papers from other categories that are cross-listed to the configured listings.
- **Replacements.** Add revised versions of earlier papers, perhaps only when the abstract has changed.
- **More text.** Score from more than the title and abstract, such as the introduction and conclusions.
- **Participants' own repos.** An advanced option for anyone who needs a private portfolio: a participant keeps their portfolio in their own repo, likely through a small GitHub App they install so the daily job can read it there.

## Open questions

- [ ] Validation: score quality has been checked only by comparing models with each other, and against papers a participant's own portfolio lists; it has not yet been checked against participants' own judgments, for example by participants rating a day's papers blind.
- [ ] Other providers: the scorer calls only Claude (Claude Code or the API); another model provider would need its own request function (`--via`).
- [ ] Chunk size: whether scoring holds up with 100 papers per request instead of 50.
- [ ] Safeguards: an automatic check of portfolio pull requests (own folder only, length).
- [ ] Viewer: missing participants (the long-term correlation table, and estimated scores for a participant with no column on a day), as described under Web viewer.
- [ ] Viewer: column headers that stay in view while scrolling, LaTeX in titles shown as math (e.g. `$z=1-4$`), and a more compact row layout for large listings.
- [ ] Scoring prompt: whether to pin papers listed in a portfolio near 100 (Full-day check), which matters most if cost forces a cheaper model.
- [ ] Legacy survey: arXiv's submission deadlines before 2026, and how complete the API's date search is before 2007.
- [ ] Legacy survey: how fast a seat's usage limit lets a year of listings go (on Oct 12, 2026, four and a half backfill days filled one 5-hour window of a seat).
- [ ] Two passes: whether the threshold of 10 holds up over more days and participants; and whether the full pass, which sees only the papers that passed the outline, scores a little lower than a v0 run (mean −2.5 points over 58 cells, Oct 14, 2026), and how much of that is run-to-run noise.

## Sources

- [GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions): standard GitHub-hosted runners, which publish the viewer, are free for public repositories.
- [Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows): schedules use UTC and may start late under load, which retired the scheduled scoring.
- [arXiv API terms of use](https://info.arxiv.org/help/api/tou.html): at most one request every 3 seconds, over a single connection.
- [arXiv availability](https://info.arxiv.org/help/availability.html): submissions received by 14:00 ET are announced at 20:00 ET the same day; Friday-to-Monday submissions are announced Monday evening.
- [arXiv RSS feeds](https://info.arxiv.org/help/rss.html): the current listing, with each paper's announce type (new, cross-list, replacement).
- [DataCite REST API](https://support.datacite.org/docs/api): metadata of registered DOIs, including arXiv's, readable from a browser.
- [arXiv on Wikipedia](https://en.wikipedia.org/wiki/ArXiv): arXiv began on August 14, 1991.
- [Claude Message Batches API](https://platform.claude.com/docs/en/build-with-claude/batch-processing): half-price requests, usually done within an hour (24 hours at most), and the discount stacks with prompt caching.
- [Claude prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching): cached input is read at a fraction of the base price; writing the cache costs 25% extra.
