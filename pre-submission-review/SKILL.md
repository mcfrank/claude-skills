---
name: pre-submission-review
description: Thorough AI review of a lab paper before it is submitted, required by the lab's AI policy. Read-only on the manuscript - it produces a numbered findings file for the author to decide on, and never edits or drafts prose. Use when the user asks for a pre-submission check or final review of their own paper, or says a paper is about to be submitted. Not for manuscripts received for peer review (the venue's AI policy governs those), and not for audits of published work (see reproducibility-audit).
---

# Pre-submission review

Lab policy: no paper is submitted before it has been reviewed by the author, often
several times, and then by a thorough AI agent. Authors need not accept every suggestion,
since some will be wrong, but they must consider each one. Source:
https://babieslearninglanguage.blogspot.com/2026/10/my-labs-ai-policies-and-norms.html

This review comes after the author's own passes and does not replace them. Never say a
paper is "ready to submit"; report what you checked and what you found.

## The hard rule

**Do not edit the manuscript, and do not draft replacement text.** Lab papers contain no
AI-authored prose, so a suggested sentence is a way of getting AI text into the paper.
Describe the problem and where it is; the author writes the fix. The only file you create
is the findings file.

## Setup

- Gather the rendered manuscript, its source, the analysis code and outputs, the prereg,
  and the bibliography. List anything missing as a limit on the review.
- Note the version reviewed (commit hash, or file name and date).
- **Review from a fresh context.** If this session helped build the analysis or the paper,
  run the review in a fresh subagent or session, as in adversarial-protocol-review.
- **Participant data rules still apply.** Check numbers against aggregate outputs and code.
  Don't open raw data containing identifiers, video, audio, or free text to do it.

## What to check

1. **Numbers.** Compare every reported number with the rendered output or the code that
   produces it: statistics, sample sizes, exclusions, and numbers repeated between the
   abstract, text, tables, and figures.
2. **Citations.** Check that each cited work exists, that its bibliography entry is
   correct, and that it supports the claim attached to it. Verify against the paper itself
   or a source you actually opened, never from memory. Report anything you couldn't check
   as "unverified", not as an error.
3. **Claims against evidence.** Look for causal language from correlational designs,
   generalization beyond the sample, exploratory results presented as confirmatory, and an
   abstract or conclusion that says more than the results do.
4. **Methods against prereg and code.** Compare the sample, exclusions, measures, and
   analysis specification, and check that deviations are disclosed.
5. **Figures and tables.** Look at them. Check them against their captions and against what
   the text says they show.
6. **Validity.** For numeric claims, apply adversarial-protocol-review: what could
   manufacture this result, or hide a real one?
7. **Argument and clarity.** Look for internal contradictions, logical gaps, undefined or
   inconsistently used terms, and places a reader would get lost.
8. **Required statements.** Check for the AI disclosure statement, which every paper needs
   even if no AI was used. Also check data and code availability statements, and the venue's
   requirements if you know the venue.

## Findings file

Write `pre_submission_review_<YYYY-MM-DD>.md` next to the manuscript. Don't commit it
unless asked. Start with the version reviewed, the model used for the review, a short
summary of the most serious problems, and what you did not or could not check. Then:

```markdown
| # | Severity | Location | Finding | Evidence | Author decision and reason |
|---|----------|----------|---------|----------|----------------------------|
| 1 | Must check | Results, para 3 | Text says N = 212; output gives 207 | `outputs/sample.csv` | |
```

- **Severity:** `Must check` (could be a factual error or invalidate a claim), `Consider`
  (a real weakness that's the author's call), or `Minor`.
- **Decision column:** leave it blank for the author, since the policy requires a decision
  on each finding. Don't fill it in or mark findings as resolved.
- **Evidence:** give something the author can check. If you aren't sure, say so in the
  finding.
- **Don't pad.** Report only real problems, ranked by severity, and say so if a category
  turned up nothing.

## Afterwards

Stop and hand the file to the author. If they ask, list the models and uses you can
document, as notes for the disclosure statement they will write. After revisions, re-run
on the new version in a new findings file.
