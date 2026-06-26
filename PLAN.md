# Allard Lab Manual — Improvement Plan

This plan has two independent parts:

- **Part 1 — Tech stack & infrastructure** (build, repo hygiene, CI/deploy).
- **Part 2 — Scientific & teamwork content** (writing, tone, risks in `source/*.rst`).

The two parts share no dependencies and can be tackled in either order.

---

# Part 1 — Tech stack & infrastructure

## Tech stack

- **Sphinx 8.2.3** (installed locally; builds clean) generating HTML from reStructuredText.
- **Theme:** `alabaster` (Sphinx default), no extensions enabled.
- **Source:** `source/*.rst` (9 content pages + `index.rst` + one blog post), images as
  paired `.pdf`/`.png`, binary assets (`.key`, `.pdf`) in `source/Content/`.
- **Build/deploy:** `make html` → output in `build/`, then manually copied into `docs/`,
  which GitHub Pages serves from the `docs/` folder on `main`.
  Site: https://allardlab.com/AllardLabManual/
- **Helpers:** `Makefile` (+ a custom `github` target), `make.bat`,
  `scripts/makeFigs.sh`, `drafts/` (working SVGs).

## Build health

The build **succeeds** with exactly **one warning**:

```
blog250131_project_org2025.rst: WARNING: document isn't included in any toctree
```

That's a content/structure issue (orphaned page) — flagged for Step 2.

## Problems worth fixing (infrastructure)

**1. Build artifacts are committed to git — the big one.**
`build/` has **241 tracked files** (HTML, `.doctrees` pickles, LaTeX intermediates,
duplicate `build/html/` *and* `build/` HTML trees). These are 100% regenerable and
shouldn't be in version control. They also bloat the repo with large binaries (PDFs,
`.key` files duplicated across `build/`, `build/html/`, and `build/_downloads/`).

**2. The committed build is stale / inconsistent** — evidence the manual copy step drifts:
- `build/.doctrees/03DevOps.doctree` exists, but source is now `03CodersAtWork.rst`
  (renamed page, old artifact lingering).
- `docs/blog_project_org2025.html` is orphaned (renamed to `blog250131_...`), still served.
- Two different rename conventions between README (`cp -r build/html docs` → makes a
  `docs/html/` subdir) and `Makefile` `github` target (`cp -a build/html/. docs` →
  flattens). They disagree.

**3. `.gitignore` is one line (`.DS_Store`).** It doesn't ignore `build/`, so artifacts
keep getting committed. (`.DS_Store` itself is correctly not tracked.)

**4. Dead config in `conf.py`:** `html_output_dir = 'docs'` is **not a real Sphinx
setting** — it's silently ignored. The actual output location comes from the `cp` step,
not config. Misleading.

**5. No pinned dependencies / no CI.** No `requirements.txt`, no `.github/workflows/`.
Builds depend on whatever Sphinx is on the author's machine; deploy is a manual sequence
(`make html` → `rm -rf docs` → `cp` → `touch docs/.nojekyll`) that's easy to get wrong
(and clearly has — see #2).

## Best-practice recommendations

### `.gitignore` — stop tracking build output

```
.DS_Store
/build/
__pycache__/
*.pyc
```

Then `git rm -r --cached build/` to untrack the 241 files.

### Pin the toolchain

Add `requirements.txt` (or `docs/requirements.txt`):

```
sphinx==8.2.3
```

### GitHub Actions for build + deploy

The recommended modern approach is GitHub's native Pages deployment (Actions artifact),
which **eliminates the committed `docs/` folder entirely**. Switch Pages source from
"Deploy from branch → /docs" to "GitHub Actions," then:

```yaml
# .github/workflows/deploy.yml
name: Deploy Sphinx site
on:
  push: { branches: [main] }
  workflow_dispatch:
permissions: { contents: read, pages: write, id-token: write }
concurrency: { group: pages, cancel-in-progress: true }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.x' }
      - run: pip install -r requirements.txt
      - run: sphinx-build -b html -W --keep-going source _site
      - uses: actions/upload-pages-artifact@v3
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: { name: github-pages, url: '${{ steps.deploy.outputs.page_url }}' }
    steps:
      - id: deploy
        uses: actions/deploy-pages@v4
```

Benefits: reproducible builds, no stale artifacts, no manual `cp` dance, `-W` catches the
toctree warning in CI. The artifact route auto-handles `.nojekyll`, so the `touch` step can
be dropped too. If keeping the `docs/`-folder method instead, at minimum a CI build-check
(`sphinx-build -W`) on PRs is worth adding.

### Trade-off to decide

Moving to Actions deployment means deleting committed `build/` **and** `docs/`. That's the
cleaner long-term setup but changes the publish mechanism (must flip the Pages setting in
repo settings — requires Jun, not automatable here). The lower-risk incremental step is
just #1–#4 (gitignore `build/`, fix `conf.py`, reconcile the copy step) while keeping the
`docs/`-folder deploy.

## Open decision

Pick the deploy path before Step 2:
- **(A) Full modern setup:** GitHub Actions deploy, delete `build/` + `docs/` from repo,
  flip Pages setting to "GitHub Actions."
- **(B) Incremental cleanup:** keep `docs/`-folder deploy; do gitignore `build/`, fix
  `conf.py`, reconcile README/Makefile copy step, add a CI build-check.

---

# Part 2 — Scientific & teamwork content

Review of `source/*.rst` against the target voice: **humble, lightly fun (not jokey or
colloquial), minimalist — always prefer to say less.** Vibe:
BCG/McKinsey/EMBO/Nature/Cell, blue-collar curiosity-driven. Ordered by priority. None of
this is a judgment on the *ideas*, which are strong — these are surface, risk, and register
fixes.

## A. Higher-stakes content risks (address first)

**1. Live Google Drive links leak into a public site.** The manual is published at
allardlab.com and embeds direct Drive URLs to working docs:
- `05Group.rst:10,20` — the "Machines" doc. Prose says this info is "*not* in a sharable
  resource… for best-practices reasons," then pastes the full Google Doc URL **twice**. If
  that doc is "anyone with link," the URL *is* the access. Self-contradictory exposure.
- `PaperWritingTips.rst:39–45` — links to a specific past revision's Big Notes / Dashboard /
  response-letter docs (may contain unpublished manuscript text and reviewer content).
- `05Group.rst:22` HPC budget sheet; `05Group.rst:40` Illustrator doc.

  *Action:* audit each Drive link's sharing setting; for anything sensitive, replace the
  public URL with "ask Jun for access."

**2. The PhD-completion standard is stated as fact.** `04WhatIsAPhD.rst:20`: "Three
published or publishable papers, at least two of which are first-author, makes a PhD." In a
quasi-contractual lab manual, an absolute degree requirement set by the PI can create
grievance risk and may conflict with the program/committee's actual authority. Softened at
L24–25, but the lead sentence reads as a hard gate. Reframe as "my expectation/guideline,
subordinate to your program's official requirements."

**3. The remote-work passage reads as a verdict against WFH.** `05Group.rst:42`: "the
advantages of asynchronous Work From Home are *multiplicative*, while… in-person work are
*exponential*… at first it seems like WFH is better, but you pay the price later." Working
arrangements intersect with caregiving/disability accommodations; a manual appearing to
disfavor WFH carries risk. Reframe as a question about preserving collaboration value, not a
conclusion about remote work.

**4. "I invoke my prerogative to be more controlling."** `02Elements.rst:200`. Honest, but
"controlling" is loaded in an advisor–trainee power asymmetry. "I stay more hands-on at
first" carries the same meaning without the connotation. Same for "I get over my trust
issues" (`02Elements.rst:216`).

## B. Typos, grammar, broken markup

| Location | Issue |
|---|---|
| `01OurMission.rst:25` | "addresses a gaps" → "a gap" |
| `01OurMission.rst:72` | "We aspire for other people want to work" → "…people to want to work" |
| `01OurMission.rst:75` | "BlueSky LinkedIn, other?." — double punctuation; missing comma |
| `02Elements.rst:13` | "presentations, poster, paper figures" → "posters" |
| `02Elements.rst:153` | stray `"""`; "Tyler **Cowan**" → "Cowen" |
| `02Elements.rst:171` | link URL is identical to L169 — wrong link for "code review anxiety workbook" |
| `02Elements.rst:173` | "intruiging" → "intriguing" |
| `02Elements.rst:216` | "as you become more experience" → "experienced" |
| `03CodersAtWork.rst:59` | "Here are some rough guideline" → "guidelines" |
| `03CodersAtWork.rst:66,71,73` | "within a scripts" → "within a script"; "Design a single source of truth" bullet duplicated |
| `03CodersAtWork.rst:81` | malformed link — stray `"` before URL and trailing `)>`_` |
| `04WhatIsAPhD.rst:8` | Milojevic link broken: title text is in the URL slot, no trailing `_` → renders as plain text |
| `05Group.rst:20` | facility name missing; "facility, `…`., under" — stray `.,` |
| `05Group.rst:22` | unmatched `[` in "make a [budget" |
| `06UserGuideToJun.rst:23,25` | two list items both numbered "6" |
| `06UserGuideToJun.rst:29` | "forward momention" → "momentum"; "subconsiously" → "subconsciously" |
| `PaperWritingTips.rst:4` | "tips and trick" → "tricks" |
| `PaperWritingTips.rst:59,61,65` | links missing trailing `_` → render as plain text, not hyperlinks |
| `PaperWritingTips.rst:71` | `:download:` pointed at a Google Drive **URL** — `:download:` is for local files; won't work |

Plus: mixed straight/curly quotes throughout, and US/UK spelling mixed ("defence" /
"memorialize", "tonne"). Pick one of each for consistency.

## C. Tone / minimalism ("always prefer to say less")

Two sections clash with the terse, plain voice of the rest of the manual and read as
machine-drafted — heavy em-dashes, curly quotes, flourishes:

- **`02Elements.rst:79–102`** ("Jun's definition of baseline scientific rigor"). Phrases like
  "This interplay creates a healthy tension," "the art—and the joy—of scientific work," and a
  near-verbatim repeat of "The *science* does not need to be complete, but the *report* needs
  to be complete" (also at L115). Cuttable by ~half with no loss.
- **`06UserGuideToJun.rst:29`** ("the fog of unfinished ideas," "every ounce of scientific
  wonder"). Same treatment.

Smaller redundancy: `06UserGuideToJun.rst:21` says "I take this seriously" essentially **four
times** in one paragraph; "pressure is on" is more colloquial than the target register.

## D. Unfinished-in-public notes

Visible to readers; reads as a draft rather than a manual:
- Open questions to the team rendered live: "Should we make a group Discord?" (`05:30`),
  "What shared resources should we develop?" (`05:36`), "weekly Starbucks/Peet's coffee
  break?" (`05:55`), "[Jun Question for Team]" on social media (`01:75`).
- `"damned poster/talk"` (`02:213`) — mild, but the only profanity; trim for a clean
  BCG/EMBO register.
- Several `.. DRAFT / END DRAFT` blocks and `TODO`s; the blog post
  (`blog250131_project_org2025.rst`) isn't linked from any toctree (the Part 1 build
  warning). Pages `OldEmails`, `PaperWritingTips`, `PeerReviewing` are reached only via
  parent-page toctrees, not `index.rst` — intentional, but worth confirming.

## Suggested execution order for Part 2

1. Section A risks (audit Drive links; reframe PhD standard, WFH, "controlling").
2. Section B unambiguous typo/markup fixes (mechanical, low-risk).
3. Section C verbose-section rewrites (need Jun's approval on voice).
4. Section D cleanup (resolve or hide open questions and DRAFT blocks).
