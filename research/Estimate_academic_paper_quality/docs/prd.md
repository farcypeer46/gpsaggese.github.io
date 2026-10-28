# Estimate Paper Quality Product Requirements

## Goal

- Build a pipeline that turns a loose list of papers into a ranked list of candidate
  trading strategies, with a score per paper and per criterion
- Users: quant researchers, who use the ranking to pick the next papers to replicate
- Background:
  - Idea: `research/Estimate_paper_quality/idea.md`
  - Reference format: `.claude/skills/references.rules.md`
  - Existing paper rating: `.claude/skills/paper.rules.md`, `# Paper Rating`
  - Download and conversion tool:
    `helpers_root/dev_scripts_helpers/download/download_academic_paper_to_md.py`

## Users and Use Cases

- **Quant researcher, first run**:
  - Pastes a list of papers collected from many sources (partial titles, URLs, DOIs)
    and gets one clean list of references
  - Gets the full text of each paper as markdown, to read it and to feed it to other
    tools
  - Ranks the papers by how promising they are and by how easy they are to replicate
    with Atlas data
  - Reads why a paper got its score, to accept or override the score
- **Quant researcher, rerun**:
  - Adds new papers to a list that was already scored
  - Gets an updated ranking without paying again for the papers already scored

## Requirements

### Input

- `R1`: The pipeline accepts a text file with one paper per line, where a line is any
  of:
  - A partial or full title
  - An arXiv URL or arXiv ID
  - A DOI or DOI URL
  - An SSRN URL
  - A publisher URL or a direct PDF URL

### Normalize the List

- `R2`: For each resolved line, the pipeline outputs one reference in the format of
  `.claude/skills/references.rules.md`, `# Format for General Text`
  - E.g., `2009, Pearl, "Causality: Models, Reasoning, and Inference" (<Link>)`
- `R3`: The year, the first author, and the title of each reference match a
  bibliographic source (arXiv, Crossref, or Semantic Scholar), not the memory of an
  LLM
- `R4`: Each reference has a download link to the full text, or a mark that no
  download link was found
- `R5`: When a line matches no paper or more than one paper, the pipeline:
  - Marks the line as unresolved
  - Lists the candidate papers, if any
  - Goes on with the other lines
- `R6`: Two input lines that resolve to the same paper produce one reference
- `R7`: The normalized list is in reverse chronological order, per
  `.claude/skills/references.rules.md`

### Download and Convert

- `R8`: For each resolved paper with a download link, the pipeline saves the PDF and
  its markdown conversion
- `R9`: Both files use the base name of `.claude/skills/references.rules.md`, `#
  Format to Use in File Names`
  - E.g., `2016.Ribeiro_et_al.Why_Should_I_Trust_You.pdf` and `.md`
- `R10`: When the download or the conversion of a paper fails, the pipeline records
  the reason for that paper and goes on with the other papers
- `R11`: A rerun does not download or convert again a paper whose PDF and markdown
  already exist

### Score

- `R12`: The pipeline scores each converted paper on each of these criteria:

  | Criterion          | What it measures                                           |
  | :----------------- | :--------------------------------------------------------- |
  | Paper quality      | Authority of the authors, the institution, and the venue   |
  | Performance        | Reported Sharpe ratio and other performance metrics        |
  | Recency            | Publication year                                           |
  | Asset class        | Target asset class, e.g., US equities                      |
  | Theoretical domain | Domain of the theory, e.g., micro vs. macro                |
  | Data availability  | Availability in Atlas of the data the paper needs          |
  | Mechanism          | Nature of the reported mechanism: causal vs. associational |

- `R13`: Each criterion score is an integer from 1 (low) to 5 (high), or `unknown`
  when the paper and the lookup sources do not give the information
- `R14`: Each criterion score comes with its evidence:
  - The value it is based on, e.g., `Sharpe 1.4, out of sample, 2005 to 2020`
  - Where the value comes from: section, table, or page of the paper, or the name of
    the external source
- `R15`: The pipeline extracts these facts from each paper and reports them as
  values:
  - Reported performance metrics: Sharpe ratio, annual return, volatility, max
    drawdown
  - Sample period, and in sample vs. out of sample
  - Target asset class and universe
  - Data the paper needs: assets, frequency, fields
- `R16`: For each dataset the paper needs, the pipeline reports one status from the
  Atlas coverage list:
  - Available in prod
  - Available in eval only
  - Not available
- `R17`: The pipeline computes a total score per paper as a weighted mean of the
  known criterion scores
  - The user can change the weights without changing code
  - The report states which criteria were `unknown` and left out of the mean
- `R18`: A rerun does not score again a paper that already has scores, unless the
  user asks for it

### Output

- `R19`: The pipeline writes one report that ranks the scored papers by total score,
  from high to low, with one row per paper:
  - The reference
  - The score of each criterion
  - The total score
- `R20`: The report lists the unresolved papers and the failed papers, with the
  reason, apart from the ranked papers
- `R21`: For each scored paper, the pipeline writes a score file with the criterion
  scores, the evidence, and the extracted facts
  - The score file uses the same base name as the PDF of the paper (see `R9`)

## Non-Goals

- Does not search for papers: it works only on the list from the user
- Does not replicate or backtest the strategy of a paper
- Does not check that the reported results are true: it reports what the paper claims
- Does not add missing data to Atlas
- Does not bypass paywalls or access controls: a paper without a public PDF is
  reported as failed
- Does not replace the general paper rating of `.claude/skills/paper.rate/SKILL.md`

## Success Metrics

- **Resolution accuracy**: at least 95% of the lines of a hand-labeled input list
  resolve to the right paper (first author, year, and title match)
- **Download coverage**: at least 80% of the resolved papers that have a public PDF
  yield both a PDF and a markdown file
- **Extraction accuracy**: at least 90% of the extracted Sharpe ratios match the
  value that a researcher reads in the paper, on a labeled set of 20 papers
- **Ranking agreement**: Spearman correlation of at least 0.6 between the total score
  and the ranking of a researcher, on a labeled set of 20 papers

## Constraints and Assumptions

- Download and conversion reuse
  `helpers_root/dev_scripts_helpers/download/download_academic_paper_to_md.py`, as
  the idea states
- Atlas is a data platform
  - It has a coverage list (assets, frequencies, eval vs. prod) that the pipeline can
    read
- Bibliographic facts and citation counts come from a lookup source, never from the
  memory of an LLM, per `.claude/skills/paper.rules.md`
- The papers describe trading strategies or return predictors
  - The criteria Performance, Asset class, and Data availability assume this
- Some sources block scripted downloads (e.g., SSRN)
- The user runs the pipeline with network access and with access to Atlas

## Open Questions

1. What maps each criterion to the 1 to 5 scale?
   - E.g., which Sharpe ratio scores 5
   - E.g., which asset classes and which theoretical domains prefers
   - E.g., how to tell a causal mechanism from an associational one
2. Which Sharpe ratio counts for Performance: gross or net of costs, in sample or out
   of sample?
3. Does Paper quality reuse the rating of `.claude/skills/paper.rules.md` (page
   length, affiliation, citations), or does it use the authors, the institution, and
   the venue, as the idea states?
4. What are the default weights of the criteria in the total score?
5. Are some criteria filters instead of scores?
   - E.g., drop a paper whose data Atlas does not have, instead of ranking it low
6. How does the pipeline read the Atlas coverage list, and who owns it?
7. Is data available only in eval worth less than data available in prod, and by how
much?
8. What does the pipeline do with a paper that does not describe a trading strategy
(e.g., a survey)?
9. Can the user give a local PDF for a paper that cannot be downloaded (e.g.,
paywall, SSRN block)?
10. Where are the outputs stored, and are the PDFs committed to the repo?
11. What is the expected size of an input list (10, 100, or 1000 papers)?
    - The answer sets the limits on cost and run time
12. Are the targets in "Success Metrics" right, and who labels the test sets?
13. How do the top-ranked papers become replication tasks (e.g., one GitHub issue per
    paper)? -> Project
