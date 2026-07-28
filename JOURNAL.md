## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/147 

**Issue title:** Resume section detection fails on text with leading whitespace #147

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The `_detect_sections()` method in `resume_parser.py` uses regex patterns that
anchor section headers (like "Education" or "Skills") strictly to the start of
a line using `^` and `\n`. PDF-extracted text frequently preserves leading
indentation/whitespace before section headers, so these anchors never match
and `detected_sections` returns an empty list even when sections clearly
exist. This breaks downstream logic in the ingestion pipeline that depends on
knowing which resume sections are present. A successful fix will make the
regex patterns tolerant of leading whitespace so indented section headers are
correctly detected.

**Branch name:** fix/147-resume-section-detection-leading-whitespace

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

### Part 1 — Understanding the Issue

**Can I explain what this issue is asking for in my own words?**

Paraphrase the issue without looking at it. If you can't, you don't understand it well enough yet. Read the full issue body, look at any linked PRs or comments, and try again.

[X] I can explain the problem and the expected behavior in 2–3 sentences without reading the issue.

**Do I understand which part of the app is affected?**

Check the labels on the issue — they often indicate the area (api, rag, ingestion, frontend, etc.). Look at the referenced files if any are mentioned. Find those files in the repo.

[X] I've located the relevant files and confirmed they exist in the codebase.

**Do I understand what "done" looks like?**

Can you describe what the app should do (or not do) once the issue is fixed? If the issue has acceptance criteria, read them carefully. If it doesn't, try writing your own — that forces you to understand the scope.

[X] I can describe a concrete before-and-after: what the user sees before the fix and what they see after.

---

### Part 2 — Tier Fit

**Is the tier a realistic match for where I am right now?**

[X] If this is my first open source contribution: I'm choosing Tier 1.

[ ] If I've contributed to large codebases before: Tier 2 or 3 is fair game.

[ ] I'm not choosing a Tier 3 issue to "challenge myself" if I haven't completed a Tier 1 or 2 first — scope surprises in Week 9 don't have a safety net.

---

### Part 3 — Codebase Readiness

**Can I find the relevant code?**

Before claiming the issue, locate the specific function, route, or module it describes. Don't rely on grep alone — open the file, read the surrounding context, and confirm you're in the right place.

[X] I've found and read the specific code the issue references (not just the file — the function or section).

**Do I understand the surrounding code well enough to change it safely?**

You don't need to understand the whole codebase. But you need to understand the file you're about to edit well enough to predict what a change will break. Read the function signatures, docstrings, and any callers.

[X] I've read enough surrounding context that I can write a rough plan for the fix without looking anything up.

**Have I read the relevant test file?**

Find the test file for the module your issue touches (tests/unit/ is the right place to start). Look at how existing tests are structured — fixtures, assertions, mock patterns. You'll need to write at least one new test.

[X] I've found the test file for my module and read at least one test end-to-end.

---

### Part 4 — Scope and Time

**How many others are already working on this issue?**

Claims are non-exclusive — more than one student may work on the same issue, and your grade comes from your own artifacts, never from being first. Still, check the issue comments and the Claims column in the Issue Catalog tab of the cohort ledger: a less-crowded issue of the same tier can mean smoother coaching and peer review.

[X] I've checked the issue comments and the ledger's Claims count, and I'm fine with how many others are on this issue.

**Is the scope realistic for Weeks 8–9?**

You have roughly two weeks to implement, test, and submit a PR. Tier 1 issues should take 3–6 hours of focused work. Tier 2 issues may take 8–12 hours. Tier 3 issues can take significantly longer.

Think about your week — other classes, work, other commitments. Is this achievable?

[X] I've estimated the time this will take and I'm confident I can complete it before the Week 9 deadline.

**Are there any blockers or dependencies?**

Some issues say "blocked by #X" or reference another issue that needs to be resolved first. Check the issue for any such dependencies.

[X] This issue has no open blockers or dependencies on other unresolved issues.

---

## Week 8 — Reproduction & solution planning

### Reproducing the Bug

**Command used:**
```
cd pathreview
python -m pytest tests/unit/test_resume_parser.py -k "no_work_experience or detect_sections" -v
```

**Result:** 2 failed, 8 deselected

```
FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_parse_resume_no_work_experience
  assert False
   +  where False = any(<generator ...>)
  # detected_lower has no "education" entry

FAILED tests/unit/test_resume_parser.py::TestResumeParser::test_detect_sections
  assert 0 > 0
   +  where 0 = len([])
  # _detect_sections() returned [] entirely
```

**Why this reproduces it without writing new test code:**
Both existing tests build their sample resume text as indented Python
triple-quoted strings (e.g. `resume_no_work = """\n        Education:\n ..."""`),
which is a natural way to write multi-line text inline in a test function.
Because the string is indented to match the surrounding code, every line
carries leading whitespace before words like `Education:` and `Skills:`.

`_detect_sections()`'s regex patterns anchor with bare `^`/`\n` immediately
followed by the section word (e.g. `^education\s*[:|-]`), with no `\s*`
allowance *before* the word. Leading indentation breaks that anchor, so the
section is never matched — `detected_sections` comes back empty even though
"Education" and "Skills" are clearly present in the text.

**Minimal interactive repro:**
```python
from ingestion.parsers.resume_parser import ResumeParser

parser = ResumeParser()
text = "        Education:\n        BS Computer Science"
parser._detect_sections(text)
# Actual:   []
# Expected: ["Education"]
```

**Root cause location:** `_detect_sections()` patterns at
[resume_parser.py:134-139](pathreview/ingestion/parsers/resume_parser.py#L134-L139) —
`\s*` appears only after the section word (trailing whitespace / before the
`:`/`|`/`-` separator), never before it, so indented headers are silently
missed.

**Reproduction commit link:** https://github.com/EmiEscu/pathreview/commit/588c30e

**PLAN.md link:** https://github.com/EmiEscu/pathreview/blob/fix/147-resume-section-detection-leading-whitespace/PLAN.md

**Walkthrough video (recommended):** 

**Blockers or open questions:**
Still deciding between `\s*` and `[ \t]*` for the whitespace allowance — need to confirm which avoids false-matching across newlines.