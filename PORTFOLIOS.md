# Writing a portfolio

A portfolio is the only thing MapHub knows about a participant: every score comes from it. This page gives the format and the portfolio-drafting prompt, which writes a first portfolio from a publication record. It works for any subject: a researcher, a group or a collaboration, in any field.

## Format

The method is general, for any field. The examples come from astronomy, where papers are also identified by NASA ADS bibcodes; other fields use their own identifiers, such as INSPIRE-HEP records, or DOIs.

The portfolio is built from papers, never from stated interests:

- **Bottom-up from papers.** Topics come from the papers the subject has, starting with their own. No topic is decided in advance.
- **Interests enter only through reading.** A topic the subject hasn't published on appears only once their reading on it builds up. Interest without reading doesn't count.
- **The subject's own work, AI-drafted and corrected by the subject.**
- **Names and instructions.** A portfolio may name people the subject knows and add scoring instructions, within the length; both affect only that participant's column.
- **One paper can sit under several topics.** Those overlaps are often the most thought-provoking.
- **Versions live in git.** There's no changelog in the file, and each scoring run can be traced to a commit of the portfolios repository.

```markdown
# Research portfolio: <subject>

## Overview
<about 100–150 words: the subject's main research thread and current focus>

## <Topic name>
Basis: <own papers | reading | both>

Summary: <about 2–4 sentences: what the subject has done or learned here, and their position>

Expertise: <methods, concepts and data sets the subject commands>

Papers: <arXiv ID, bibcode or DOI, role (first author / coauthor)>, …

## <Next topic>
…
```

The lengths are guides: an Overview or Summary may run longer, with short bullet lists, when the extra detail comes from the papers, as in a group's portfolio that follows a series of papers. A group's portfolio may end with a line defining the marks in its Papers lists (the portfolio-drafting prompt). The whole portfolio runs to about 1,500 words for a person and about 3,000 for a group or collaboration.

The portfolio must stand on its own: it contains no scores, and everything in it must make sense to a reader with no other context, such as MapHub's scorer.

## The portfolio-drafting prompt (Deep Research)

Run this once, in a chatbot's Deep Research mode. Attach or paste at least one of the subject's profile page, publication list or CV. The result is a preliminary portfolio; save it as `portfolio.md`, keeping its Markdown headings (copying from a chat window can drop the `#` marks). The prompt names NASA ADS, INSPIRE-HEP and arXiv, and asks for whatever databases suit the subject's field.

```text
Build a preliminary research portfolio for the researcher, group or collaboration that the attached documents describe (the subject). I have attached at least one of: a profile page, a publication list, a CV. Work out who the subject is from them. Treat them as the starting point, and use NASA ADS, INSPIRE-HEP, arXiv or other databases suited to the field to find, check and complete the subject's publication record.

Research steps:
1. Compile the subject's publications: title, arXiv ID (or ADS bibcode or DOI if not on arXiv), year, and role (first author, or coauthor with position in the author list). For a group or collaboration, see the note after these steps. Flag anything you are unsure belongs to the subject.
2. Read each paper's abstract, and for the most central papers (first-author papers, for a person) also the introduction and conclusions. arXiv's HTML full texts (arxiv.org/html/<ID>) are usually the most reliable way to read a paper; arXiv abstract pages and ADS pages often fail, and repeated requests get rate-limited. Keep track of papers you could read only at title level.
3. Group the papers into topics from the bottom up: let the topics emerge from what the papers actually do, not from a predefined list. A paper may belong to several topics. Do not add any topic that none of the papers support, even if a profile mentions it as an interest. A topic with only one or two papers may be merged into a close neighbour; ask me when unsure.

For a group or collaboration, decide case by case which papers count as the group's, from the documents and the papers themselves (author lists, acknowledgments, the group's stated aims), and ask me about every borderline case. Papers that may belong but are not confirmed can be included with less weight: mark them in the Papers lists, define the marks in one line at the end of the portfolio, and end with a line saying which expertise rests on confirmed papers and which on marked ones.

Write the portfolio in exactly this format:

# Research portfolio: <subject>

## Overview
<100–150 words: the subject's main research thread and current focus>

## <Topic name>
Basis: own papers

Summary: <2–4 sentences: what the subject has done here and their position>

Expertise: <methods, concepts and data sets the papers show the subject commands; weigh central papers more than papers where the subject's role was small>

Papers: <arXiv ID, bibcode or DOI, role>, …

Rules: base every statement on the papers or the documents provided, not on guesses; write in plain language that a reader with no other context can follow; make this first draft comprehensive, including every topic the papers support, but write it with an eventual length of about 1,500 words (about 3,000 for a group or collaboration) in mind. The Overview and Summary lengths are guides: go past them, with short bullet lists if that helps, when the detail comes from the papers. Include the limits and future work the papers state. If the draft falls short of the target length, expand from the main papers' introductions and conclusions; never pad. Where the documents and the papers disagree, follow the papers.

Output only the portfolio, not your working publication list, and then list the papers you could read only at title level.

Then talk the draft through with me. Ask, one question at a time, about (a) papers you were unsure belong to the subject, (b) topic boundaries you were unsure about, and (c) anything in the documents that contradicts the publication record. Then help me condense the portfolio to about 1,500 words (3,000 for a group or collaboration) by proposing what to merge, shorten or cut. Revise until I say it is final.
```
