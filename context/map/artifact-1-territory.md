# Artifact 1 — Territory Map (FastAPI)

**Repo:** FastAPI  **Window:** last 12 months (2025-07-16 → 2026-07-16)
**Method:** `git log --name-only`, default branch, ranked by *number of commits touching a path* (not lines changed).
**Noise filter:** of **1,624** commits, **263** survive after removing release bumps, `📝 release-notes`/`🌐 translation` auto-commits, pure docs/prose (`docs/**`, `README.md`), `.github/**`, lockfiles, configs, snapshots. `docs_src/*` (example code) and `tests/test_tutorial/*` (its mirror tests) are kept as real code.

---

## TL;DR

- **Gravitational center of the code:** `fastapi/dependencies/utils.py` (46 changes) + `fastapi/routing.py` (36) — the DI resolver and the request-handling spine. Everything else orbits these.
- **The year's shape:** a big **Pydantic-v2 / param-annotation wave crested in winter (Q4'25 → Q1'26)** — ~1,600 file-touches/quarter, sweeping framework internals *and* every tutorial example at once — then **collapsed ~10× in 2026**. Effort has since pivoted to tooling, docs infra, and a brand-new `fastapi/.agents` skills folder.
- **Fragile heart (bug hotspots):** 🔴 `dependencies/utils.py` (41% of its commits are fixes) and `_compat/v2.py` (48%) are both high-churn *and* high-bug-density. Quiet-but-bug-dense: 🟠 `security/oauth2.py` (50%), `dependencies/models.py` (44%). By contrast `routing.py`/`openapi/utils.py` churn from *features*, not bugs (~22–25%).
- **Load-bearing but stable (safe anchors):** `security/base.py` (fan-in 5, untouched since 2018), `types.py` (fan-in 8, 4 changes), `logger.py` (frozen since 2019), `exceptions.py` (most-imported file, fan-in 11).
- **The architectural seam** where change is hard to isolate: a 4-file knot — **`_compat/v2.py` + `dependencies/utils.py` + `openapi/utils.py` + `routing.py`**. `_compat/v2.py` is a coupling hub touching all six field/validation/serialization files (the "Pydantic-v2 blast radius").
- **Cross-area "common denominator":** exists only in docs — `docs/en/mkdocs.yml` (structural glue) and the bot-driven `release-notes.md` (pure noise). **No code file** plays this role; inside `fastapi/` the closest connector is `routing.py`, and only functionally.
- **Data-integrity caveat:** every file the conclusions rest on still exists. But the old `fastapi/_compat.py` monolith and 5 transient v1-mixing files were created-then-deleted this year — **anchor future work on `_compat/v2.py`, not `_compat.py`.**
- **By functional domain:** activity splits ~data/validation 37% · runtime 20% (but the *most commits*, 103) · auth 6% · build 5% · public-API 2%. Auth is low-volume but persistent; runtime is the steady workhorse; data/validation is spiky (winter sweeps).
- **Consistency ≠ volume:** `scripts/` is active *every* month (13/13); `dependencies/utils.py` & `routing.py` are the steadiest code (8/13 months). `_compat/v2.py` is high-volume but spiky (6 months, one Q4 sweep).
- **Bus factor:** the project is BDFL-concentrated — Sebastián Ramírez authors 48–83% of every fragile file (`routing.py` 83%, `params.py` 73%). `dependencies/utils.py` (18 authors) and `security/oauth2.py` are the most distributed.

### Contents
1. [Most frequently modified — folders & files (repo-wide)](#1-most-frequently-modified-repo-wide)
2. [Quarterly workload](#2-quarterly-workload)
3. [`fastapi/` internals deep dive](#3-fastapi-internals-deep-dive)
4. [Bug hotspots](#4-bug-hotspots)
5. [Stable-but-central files](#5-stable-but-central-files)
6. [Directory & file couplings](#6-directory--file-couplings)
7. [Cross-area "common denominator" files](#7-cross-area-common-denominator-files)
8. [Existence check & data caveats](#8-existence-check--data-caveats)
9. [Functional-domain intersection](#9-functional-domain-intersection)
10. [Consistency ranking (active months)](#10-consistency-ranking-active-months)
11. [Ownership / bus factor](#11-ownership--bus-factor)

---

## 1. Most frequently modified (repo-wide)

### a) Folders / modules
Top-level (`tests`, `fastapi`, `docs_src`) was too coarse, so drilled to real areas of activity.

| # | Folder / module | Commits | What it is |
|---|-----------------|--------:|------------|
| 1 | `tests/test_tutorial/*` | 1005 | Tests mirroring every doc example |
| 2 | `tests/` (root-level) | 511 | Core framework tests |
| 3 | `tests/test_request_params` | 167 | New query/param test suites |
| 4 | `fastapi/` (core top level) | 137 | The framework itself |
| 5 | `scripts/` | 100 | Docs/translation/CI tooling |
| 6 | `docs_src/dependencies` | 84 | DI examples |
| 7 | `docs_src/security` | 69 | Security/auth examples |
| 8 | `docs_src/settings` | 69 | Settings/config examples |
| 9 | `fastapi/_compat` | 61 | Pydantic v1/v2 compat layer |
| 10 | `fastapi/dependencies` | 55 | DI resolution engine |

### b) Files (code only; `README.md` excluded as prose)

| # | File | Changes | Area |
|---|------|--------:|------|
| 1 | `fastapi/dependencies/utils.py` | 46 | DI resolution core |
| 2 | `fastapi/routing.py` | 36 | Router / request handling |
| 3 | `fastapi/_compat/v2.py` | 25 | Pydantic v2 compat |
| 4 | `scripts/docs.py` | 23 | Docs build tooling |
| 5 | `scripts/translate.py` | 21 | Translation automation |
| 6 | `fastapi/openapi/utils.py` | 20 | OpenAPI schema generation |
| 7 | `fastapi/applications.py` | 15 | App / `FastAPI` class |
| 8 | `fastapi/encoders.py` | 13 | `jsonable_encoder` |
| 9 | `fastapi/utils.py` | 12 | Shared helpers |
| 10 | `fastapi/params.py` | 11 | `Query`/`Path`/`Body` defs |

**Takeaways:** the DI + request-lifecycle code is the center; Pydantic-v2 (`_compat`) is still a live cost; two `scripts/` tooling files crack the top 5 (docs/i18n infra); `docs_src/*` edits show as coupled churn with `tests/test_tutorial` because examples are tested.

---

## 2. Quarterly workload

Calendar quarters; **2025-Q3 and 2026-Q3 are partial** (window runs Jul 16 → Jul 16).

| Quarter | Span | Raw commits | Kept code commits | File-touch events |
|---------|------|------------:|------------------:|------------------:|
| 2025-Q3 | Jul 16–Sep *(partial)* | 206 | 38 | 87 |
| 2025-Q4 | Oct–Dec | 403 | 82 | **1,561** |
| 2026-Q1 | Jan–Mar | 496 | 88 | **1,656** |
| 2026-Q2 | Apr–Jun | 399 | 50 | 152 |
| 2026-Q3 | Jul 1–16 *(partial)* | 120 | 5 | 7 |

**Headline:** file-touches explode in Q4'25–Q1'26 (~1,600 each) then collapse ~10× — moderate commit counts with huge files-per-commit = **massive multi-file sweeps**, not many small changes.

- **2025-Q3** *(partial ramp)* — quiet; security examples, `translate.py`, README upkeep.
- **2025-Q4** — *peak framework work.* `fastapi/` internals hottest (67); `_compat/v2.py` (15), `dependencies/utils.py` (26); new `test_request_params` (138). Pydantic-v2 + param push.
- **2026-Q1** — *peak examples/docs churn.* `tests/test_tutorial` 574; `scripts/` 90. Mass tutorial rewrite; framework code cools.
- **2026-Q2** — *sharp slowdown & pivot.* Code churn drops ~3×; effort moves to `scripts/`, README, and a **new `fastapi/.agents` skills folder**.
- **2026-Q3** *(2 weeks)* — minimal; `routing.py`, `test_frontend.py`.

**Shape of the year:** a winter wave (coordinated Pydantic-v2 / param overhaul touching internals + every tutorial) → a pronounced 2026 calm, effort shifting to tooling and agent-skills scaffolding.

---

## 3. `fastapi/` internals deep dive

`fastapi/` is a fairly *flat* package — only 6 subfolders, most logic in top-level single-file modules. The folder list is short by design; the file list carries the detail.

### a) Top folders / modules (logical view — each top-level `.py` = its own module)

| # | Logical module | Touches | Kind |
|---|----------------|--------:|------|
| 1 | `fastapi/_compat` | 61 | folder (v1/v2 shim) |
| 2 | `fastapi/dependencies` | 55 | folder (DI engine) |
| 3 | `fastapi/routing.py` | 36 | top-level module |
| 4 | `fastapi/openapi` | 33 | folder (schema gen) |
| 5 | `fastapi/security` | 28 | folder (auth schemes) |
| 6 | `fastapi/applications.py` | 15 | top-level module |
| 7 | `fastapi/.agents` | 15 | folder (**new** — agent skills) |
| 8 | `fastapi/encoders.py` | 13 | top-level module |
| 9 | `fastapi/utils.py` | 12 | top-level module |
| 10 | `fastapi/params.py` | 11 | top-level module |

Pure-folder ranking (all 6 that exist): `_compat` 61 › `dependencies` 55 › `openapi` 33 › `security` 28 › `.agents` 15 › `middleware` 2.

### b) Top 10 files

| # | File | Touches | | # | File | Touches |
|---|------|--------:|---|---|------|--------:|
| 1 | `dependencies/utils.py` | 46 | | 6 | `encoders.py` | 13 |
| 2 | `routing.py` | 36 | | 7 | `utils.py` | 12 |
| 3 | `_compat/v2.py` | 25 | | 8 | `params.py` | 11 |
| 4 | `openapi/utils.py` | 20 | | 9 | `_compat/__init__.py` | 10 |
| 5 | `applications.py` | 15 | | 10 | `dependencies/models.py` | 9 |

### Quarterly (`fastapi/` only)

| Quarter | Commits | File-touches | Dominant area |
|---------|--------:|-------------:|---------------|
| 2025-Q3 *(partial)* | 15 | 21 | DI + top-level |
| 2025-Q4 | **51** | **167** | `_compat` (41) + `dependencies` (34) |
| 2026-Q1 | 41 | 113 | top-level + `openapi` (16) |
| 2026-Q2 | 16 | 28 | top-level + **`.agents` (8)** |
| 2026-Q3 *(partial)* | 2 | 2 | `routing.py` |

**What the internals reveal (hidden in the repo-wide view):**
- **Pydantic-v2 compat work was sharply time-boxed to Q4'25** — `_compat` went 41 → 18 → 2 → 0; by Q2'26 essentially silent (v2 stabilization done).
- **`dependencies/utils.py`** is the perennial hotspot but front-loaded (26 of 46 touches in Q4'25).
- **`routing.py`** is the steadiest file — active in *every* quarter.
- **`fastapi/.agents`** is brand new (Q2'26), already 7th-ranked — a pivot toward agent/AI tooling.

---

## 4. Bug hotspots

A commit is a **bugfix** if its subject has `🐛` or matches `\bfix` (excludes "prefix"/"suffix"; includes "fixes"/"fixed"). **73 of 263 kept commits (28%) are bugfixes.** `fix%` = bugfix share of a path's commits.

### Root-level — files & modules

| File | Fixes | Total | fix% | | Module | Fixes | Total | fix% |
|------|------:|------:|-----:|---|--------|------:|------:|-----:|
| `dependencies/utils.py` | 19 | 46 | 41% | | `tests/test_tutorial` | 156 | 1005 | 16% |
| `_compat/v2.py` | 12 | 25 | **48%** | | `tests/` | 118 | 511 | 23% |
| `routing.py` | 8 | 36 | 22% | | `scripts/` | 51 | 158 | 32% |
| `scripts/translate.py` | 5 | 21 | 24% | | `tests/test_request_params` | 28 | 167 | 17% |
| `openapi/utils.py` | 5 | 20 | 25% | | `fastapi/dependencies` | 23 | 55 | **42%** |
| `security/oauth2.py` | 4 | 8 | **50%** | | `fastapi/` (top-level) | 22 | 137 | 16% |
| `dependencies/models.py` | 4 | 9 | 44% | | `fastapi/_compat` | 17 | 61 | 28% |
| `tests/test_compat.py` | 4 | 10 | 40% | | `fastapi/security` | 10 | 28 | 36% |

> Tests top the raw counts only because a fix ships its regression test alongside the code — the shadow of the code fixes, not independent hotspots. Rank real signal by `fix%`.

### Churn vs. bug-density (the key framing)

| | High bug-density (≥40%) | Low bug-density (<25%) |
|---|---|---|
| **High churn** | 🔴 `dependencies/utils.py` (46, 41%) · `_compat/v2.py` (25, 48%) | 🟡 `routing.py` (36, 22%) · `openapi/utils.py` (20, 25%) |
| **Low churn** | 🟠 `security/oauth2.py` (8, 50%) · `dependencies/models.py` (9, 44%) | (unremarkable) |

- **🔴 True bug hotspots:** `dependencies/utils.py` + `_compat/v2.py` — constantly changed *and* ~half the changes are firefighting. The fragile heart.
- **🟡 High churn, low bug rate:** `routing.py` / `openapi/utils.py` — feature work, lower risk.
- **🟠 Low churn, high bug density:** `security/oauth2.py`, `dependencies/models.py` — rarely touched, almost only to fix defects. Handle with care.
- **Module level:** `fastapi/dependencies` (42%) and `fastapi/security` (36%) most defect-prone; `fastapi/openapi` churns but is stable (15%).

---

## 5. Stable-but-central files

Load-bearing, quiet foundations of `fastapi/`. **Fan-in** = other `fastapi/` modules importing it (intra-package, AST parse); **churn** = 12-month commits; **age** = first→last commit.

| File | Fan-in | Churn | Last | First | Read |
|------|-------:|------:|:----:|:-----:|------|
| `exceptions.py` | **11** | 9 | 2026-02 | 2019-02 | Most depended-on file; changes rarely relative to reach |
| `types.py` | 8 | **4** | 2026-02 | 2020-12 | 🟢 Textbook: type aliases, 7 lifetime commits, 8 importers |
| `openapi/models.py` | 8 | 8 | 2026-05 | 2018-12 | Schema models — broad fan-in, moderate change |
| `datastructures.py` | 6 | 8 | 2026-03 | 2019-03 | `UploadFile`, `Default` — old, steady |
| `security/base.py` | 5 | **0** | **2018-12** | 2018-12 | 🟢 Purest case — `SecurityBase`, untouched ~7.6 yrs |
| `utils.py` | 5 | 12 | 2026-02 | 2018-12 | Central helpers, but churns |
| `logger.py` | 3 | **0** | **2019-12** | 2019-12 | 🟢 Frozen — one commit ever (2019), 3 importers |

**Clearest answers:** `security/base.py` (archetype — 0 commits/yr, base class since 2018), `types.py` (high fan-in, lowest churn), `logger.py` (frozen), `exceptions.py` (most central; stabler than its importance predicts).

**Central but NOT stable (don't mistake):** `__init__.py` (fan-in 7 but churn 76 = `__version__` bumps); `_compat/__init__.py` (fan-in 10 but born Oct 2025); `routing.py`/`params.py`/`encoders.py` (high fan-in, actively worked).

**Caveats:** fan-in is intra-package only — publicly-central files (`responses.py`, `testclient.py`, `requests.py`) score low internally yet are widely used externally; re-exports via `from fastapi import X` are attributed to `__init__.py`.

---

## 6. Directory & file couplings

How often distinct paths appear in the *same* commit (263 commits). Directional confidence = P(B touched | A touched).

### Directory-level pairs

| Rank | Pair | Co-commits | Jaccard | Note |
|------|------|-----------:|:-------:|------|
| 1 | `fastapi/` + `tests/` | 41 | 0.34 | 63% of core commits also touch tests |
| 2 | `fastapi/dependencies` + `tests/` | 39 | 0.36 | **78%** of DI commits touch tests |
| 3 | `tests/` + `tests/test_tutorial` | 23 | 0.20 | 52% |
| 4 | `fastapi/` + `fastapi/openapi` | 19 | 0.26 | code↔code |
| 5 | `fastapi/` + `fastapi/dependencies` | 19 | 0.20 | code↔code |

**Top three read:** (1) `fastapi/`↔`tests/` = healthy change-with-its-test discipline; (2) `dependencies`↔`tests/` = tightest bond, driven by *risk* (busiest+buggiest file); (3) `tests/`↔`test_tutorial` = the wide-sweep ripple behind the winter spike.
**Most informative structural coupling:** the triplet `fastapi/` + `fastapi/_compat` + `fastapi/dependencies` (12) — the real seam.

### File-level pairs inside `fastapi/`

| Rank | Pair | Co-commits | Jaccard |
|------|------|-----------:|:-------:|
| 1 | `openapi/utils.py` + `routing.py` | 12 | 0.27 |
| 1 | `_compat/v2.py` + `dependencies/utils.py` | 12 | 0.20 |
| 1 | `dependencies/utils.py` + `routing.py` | 12 | 0.17 |
| 4 | `dependencies/utils.py` + `openapi/utils.py` | 11 | 0.20 |
| 5 | `_compat/__init__.py` + `_compat/v2.py` | 10 | **0.40** |
| 5 | `_compat/v2.py` + `routing.py` | 10 | 0.20 |

**Top three read:** (1) `openapi/utils.py`↔`routing.py` — schema gen welded to routing ("declare endpoint → describe it"); (2) `_compat/v2.py`↔`dependencies/utils.py` — coupling of the two buggiest files; (3) `dependencies/utils.py`↔`routing.py` — the request-lifecycle spine.

**The hub:** `_compat/v2.py` pairs strongly with **six** core files (`dependencies/utils.py`, `routing.py`, `openapi/utils.py`, `utils.py`, `encoders.py`, `datastructures.py`) — the **Pydantic-v2 blast radius**. The dense knot: a **4-file cluster** `_compat/v2.py` + `dependencies/utils.py` + `openapi/utils.py` + `routing.py` accounts for nearly every top pair/triplet — where change is hardest to isolate.

---

## 7. Cross-area "common denominator" files

Per-file breadth = number of *distinct areas* it co-changes with.

### Repo-wide (incl. noise files)

| File | Areas | Commits | Nature |
|------|------:|--------:|--------|
| `docs/en/mkdocs.yml` | **31** | 29 | 🟢 Nav manifest — genuine structural glue |
| `docs/en/docs/release-notes.md` | 27 | **870** | 🔴 Bot-appended per PR — process artifact, no signal |
| `docs/*/docs/index.md` (all langs) | 27 | 9–18 | Landing pages (mirror README) |
| `README.md` | 26 | 35 | Root readme |

- **`mkdocs.yml`** is the real denominator — edited whenever any doc page anywhere moves.
- **`release-notes.md`** is the noise file — bot-driven, ~3× any other file, zero design meaning.
- **No *code* file** is a cross-area denominator; the repo's glue lives in the docs toolchain.

### Inside `fastapi/` — No.

Breadth over 22 internal areas (subpackages + each top-level module):

| File | Breadth | Commits | Note |
|------|--------:|--------:|------|
| `routing.py` | 20 | 36 | steady across many commits |
| `openapi/utils.py` | 20 | 20 | inflated by sweeps |
| `dependencies/utils.py` | 17 | 46 | steady |
| `_compat/v2.py` | 17 | 25 | mixed |
| `applications.py`, `encoders.py`, `params.py`, `datastructures.py`, `utils.py`, … | 16–17 | 8–15 | inflated |
| *(dead)* `temp_pydantic_v1_params.py`, `_compat/main.py`, `v1.py`, `model_field.py` | 16 | **3–6** | 🚩 sweep artifact |

- **Distribution is flat, not spiky** — top ~12 files cluster at 16–20 of 22; no outlier hub (unlike docs' `mkdocs.yml`=31).
- **Breadth is inflated by giant Q4'25 v2-refactor sweeps** — proof: dead files hit breadth 16 in only 3–6 commits.
- **Closest internal connector = `routing.py`** (20/22 across 36 *separate* commits; the only file reaching `middleware`, `responses.py`, `sse.py`) — a *functional* hub (request spine), not mechanical glue.
- **Key distinction:** `mkdocs.yml` glues by *mechanism*; inside `fastapi/` nothing does — it's a tightly-interconnected core amplified by sweep refactors, centered on `routing.py`.

---

## 8. Existence check & data caveats

**Every file the conclusions rest on is still tracked** — all top coupling-pair files, stable-central files, and bug hotspots (`dependencies/utils.py`, `routing.py`, `openapi/utils.py`, `_compat/v2.py`, `security/base.py`, `types.py`, `exceptions.py`, …). ✅

**Six files from the raw history are gone (none reached any top ranking):**

| Dead file | What happened |
|-----------|---------------|
| `fastapi/_compat.py` (monolith) | Split into the `_compat/` package on 2025-10-11 (#14168); concern lives on in `_compat/v2.py`. |
| `fastapi/temp_pydantic_v1_params.py` | Transient scaffolding — added 2025-10-11, deleted 2025-12-27 (#14609 "Drop support for `pydantic.v1`"). |
| `_compat/v1.py`, `_compat/may_v1.py`, `_compat/main.py`, `_compat/model_field.py` | Same short-lived Pydantic-v1-mixing experiment; added Oct, removed Dec 2025. |

**Caveat for downstream use:** `_compat` was a moving target — monolith → package (Oct), a v1/v2-mixing feature added then rolled back (Dec). Part of the heavy Q4'25 `_compat` churn is create/refactor/delete motion, not steady editing. The current `_compat/` package is just `__init__.py`, `shared.py`, `v2.py`. **Anchor future work on `_compat/v2.py`, not the historical `_compat.py`.**

**General method notes:** ranks by *commits touching a path*, not lines changed; default branch only; partial first/last quarters. Possible follow-up: re-run weighted by lines added/deleted.

### Noise categories — verified

Beyond the standard filter (release bumps, translations, docs/prose, lockfiles, configs, snapshots), a scan of subject lines confirmed:

- **161 `⬆` dependency-bump commits** (mostly `ruff`/dependabot) in the window — already excluded via the *file* filter (they touch lockfiles/configs), but worth naming as a category.
- **CI/tooling commits** (`👷`/`🔧`/`🔨`, incl. a one-off `pre-commit autofix` sweep #14585) leak into `scripts/` churn, which is deliberately counted as *real activity* (docs/i18n infra). Judgment call, flagged here.
- **No big-bang "mass formatting" commit exists** — FastAPI runs `ruff` continuously via pre-commit, so there is no reformat sweep to filter. The absence is itself a finding: line-weighted churn won't be distorted by formatting noise.

---

## 9. Functional-domain intersection

The 263 kept commits projected onto capability domains (files mapped by path/topic; `dependencies/*` → runtime, `_compat`/`encoders`/`params`/`types` → data/validation, `security/*` → auth, `openapi`/`__init__` → public-API, `scripts`/`.agents` → build).

| Domain | File-touches | Share | Distinct commits |
|--------|-------------:|------:|-----------------:|
| data/validation | 1278 | 36.9% | 76 |
| examples/tests *(unclassified)* | 952 | 27.5% | 73 |
| runtime | 710 | 20.5% | **103** |
| auth | 216 | 6.2% | 30 |
| build/tooling | 173 | 5.0% | 74 |
| public-api/schema | 76 | 2.2% | 35 |
| integration/misc | 14 | 0.4% | 14 |

### Per-quarter (touch events)

| Domain | Q3'25 | Q4'25 | Q1'26 | Q2'26 | Q3'26 |
|--------|------:|------:|------:|------:|------:|
| data/validation | 15 | **682** | 558 | 23 | 0 |
| runtime | 20 | 307 | **352** | 29 | 2 |
| auth | 19 | 80 | **114** | 3 | 0 |
| build/tooling | 9 | 25 | **97** | 41 | 1 |
| public-api/schema | 3 | 30 | 40 | 3 | 0 |

**Reads:**
- **Two lenses disagree, usefully.** By *touches*, **data/validation dominates (37%)** — but that's inflated by winter sweeps. By *distinct commits*, **runtime leads (103)** — it's touched by the most separate pieces of work, the steady backbone.
- **Auth is low-volume but concentrated in Q1'26** (114 touches) then dies — a discrete security-hardening burst, not steady work. Matches `security/` being bug-dense (§4).
- **Build/tooling is the only domain still rising into 2026** (Q1 97, Q2 41) — the pivot to `scripts/` + `.agents`, while every code domain collapses.
- **Public-API/schema is tiny (2%)** — the public surface is stable; most work is *internal* plumbing.

**Caveat:** 27.5% of touches fell in `examples/tests` that keyword-matching couldn't confidently assign to a domain — treat the domain shares as ±5pp, directional not exact.

---

## 10. Consistency ranking (active months)

"Consistently active" = present in many *distinct months* (of 13 in the window), separating steady workhorses from one-off refactor spikes. Contrast with raw volume (§1).

### `fastapi/` files by active months

| File | Active months | Commits | Pattern |
|------|:-------------:|--------:|---------|
| `dependencies/utils.py` | **8 / 13** | 46 | steady *and* high-volume — the true backbone |
| `routing.py` | **8 / 13** | 36 | steady *and* high-volume |
| `applications.py` | 7 | 15 | steady, moderate volume |
| `_compat/v2.py` | 6 | 25 | 🚩 high volume but **spiky** (Q4 sweep) |
| `openapi/utils.py` | 6 | 20 | steady-ish |
| `encoders.py` | 6 | 13 | steady trickle |
| `security/oauth2.py` | 6 | 8 | 🔎 **low volume, persistent** — steady auth fixes |
| `security/api_key.py` | 6 | 6 | 🔎 low volume, persistent |
| `params.py` | 5 | 11 | — |

### Directories by active months

| Dir | Active months | Note |
|-----|:-------------:|------|
| `scripts/` | **13 / 13** | active *every* month — continuous docs/i18n/CI tooling |
| `README.md` | 11 | badge/sponsor upkeep |
| `tests/test_tutorial` | 11 | continuous example testing |
| `fastapi/(top)` | 10 | core always in motion |
| `tests/` | 10 | — |
| `fastapi/dependencies` | 8 | — |

**Reads:** volume and consistency diverge sharply. `_compat/v2.py` ranks #3 by volume but is *spiky* (6 months, driven by the one Q4 refactor). The **auth files (`oauth2.py`, `api_key.py`) are the opposite** — low total commits but present in 6 different months: a persistent trickle of small fixes, not a project. And **`scripts/` never goes quiet** (13/13) — the most *consistently* active area in the whole repo, though it never tops the volume charts.

---

## 11. Ownership / bus factor

Author concentration on the fragile + central files (§4, §6). Names de-mojibaked.

| File | Authors | Top author's share | Primary author |
|------|--------:|:------------------:|----------------|
| `routing.py` | 7 | **83%** | Sebastián Ramírez |
| `params.py` | 4 | 73% | Sebastián Ramírez |
| `openapi/utils.py` | 7 | 70% | Sebastián Ramírez |
| `dependencies/models.py` | 4 | 67% | Sebastián Ramírez |
| `exceptions.py` | 4 | 67% | Sebastián Ramírez |
| `_compat/v2.py` | 5 | 60% | Sebastián Ramírez |
| `encoders.py` | 5 | 54% | Sebastián Ramírez |
| `dependencies/utils.py` | 18 | 48% | Sebastián Ramírez |
| `security/oauth2.py` | 5 | 38% | Sebastián Ramírez |

**Reads:**
- **The whole project is BDFL-concentrated** — Sebastián Ramírez (tiangolo) is the top author of *every* fragile file, 38–83%. This is expected for FastAPI but means bus-factor risk is systemic, not file-specific.
- **`routing.py` is the sharpest single-owner risk** — 83% one author on the request-handling spine. If any file needs a documented second owner, it's this one.
- **`dependencies/utils.py` is the healthiest** despite being the #1 hotspot — 18 contributors, top share only 48%. The busiest/buggiest file also has the broadest shared understanding, which softens its risk.
- **`security/oauth2.py`** has the most *distributed* ownership (38%) — good, given it's bug-dense (§4); no single point of failure on auth.

**Caveat:** author counts are over the 12-month window on the current file paths; pre-window history and the `_compat.py`→`_compat/` rename are not followed.
