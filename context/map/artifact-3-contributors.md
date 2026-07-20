# Artifact 3 — Contributor Contact Map (FastAPI)

> **Purpose.** Synthesizes [`artifact-1-territory.md`](artifact-1-territory.md) (git-history
> hotspots, bug density, bus factor) and [`artifact-2-structure.md`](artifact-2-structure.md)
> (import graph, cycles, load-bearing invariants, testability) into the **top 5 areas where a
> change would most likely require talking to a human before or during the work** — because the
> knowledge is single-owned, undocumented, fragile, or all three.
>
> **What "requires contact" means here.** Not "who to CC on a PR," but *where tacit knowledge,
> bus-factor concentration, or hidden invariants make solo changes risky* — the places where a
> reviewer's context is a hard dependency, not a nicety.
>
> **Ownership data** = `git log --since=2023-07` author counts per file (verified against
> artifact-1 §11). Names are as they appear in git; emails omitted. "tiangolo" =
> Sebastián Ramírez (`tiangolo@gmail.com`), the BDFL.

---

## TL;DR

- **What this map is:** artifacts 1+2 (git hotspots + import structure) distilled into the **5 areas
  where a change needs a human's context before you touch it** — plus who those humans are and what
  each one knows.
- **The systemic risk is one name.** tiangolo authors 38–83% of every fragile file, **100% of the
  load-bearing import invariants**, and has **~8.5× the `fastapi/` commits** of the next human —
  bus-factor risk is structural, not per-file.
- **🔴 #1 area — Pydantic-v2 compat seam** (`_compat/__init__.py` + `_compat/v2.py`): buggiest file
  (48% fix) whose correctness rests on an **undocumented import-ordering invariant** — a
  plausible-looking "fix" can break startup.
- **🔴 #2 — `routing.py`:** 83% single-owner on a 6385-LOC, e2e-only engine — tests can't substitute
  for owner review.
- **🟠 #3 — `dependencies/utils.py`:** the #1 hotspot, but ownership is **broad-but-shallow (18
  authors)** — ask the *path* author, not one owner.
- **🟠 #4 — the hand-defused cycles:** `TYPE_CHECKING` guards & deferred imports that look like
  "cleanup bait" but are load-bearing and **100% tiangolo** — ask before you tidy.
- **🟡 #5 — auth + `openapi/models.py`:** low-churn / high-bug (50% / 44%), no clear owner, and a
  hidden schema coupling that can break **every** auth scheme at once.
- **Two de-facto second owners emerge.** **Motov Yurii** (the only non-BDFL present in *all five*
  areas — parameter parsing, auth, DI edge cases) and **Sofie Van Landeghem** (Pydantic/Python
  version-compat & typing). Looping one in is the cheapest way to cut single-owner risk.
- **The best outcome of any contact is documentation, not approval** — the invariants live only in
  git archaeology; the highest-leverage move is writing them into the tree.
- **Each area carries a "📖 Read before you change" list** — the specific PRs, commits, and edge
  cases (invariant-line history, load-bearing guards, credential-parsing regressions) worth reading
  *before* you touch it, so the context transfer starts from the tree, not a conversation.
- **Contributor data (Appendices A/B, humans only, last 12 months).** Bots and — verified — **zero**
  Claude/Codex/Copilot commits excluded. The **top 3 `fastapi/` contributors are exactly the
  fragile-core trio**; ranks 5–10 are single-commit specialists whose one change still lands on a
  hot file.

### Contents
1. [The five contact areas — at a glance](#the-five-contact-areas--at-a-glance)
2. [Area 1 — Pydantic-v2 compat seam](#1-the-pydantic-v2-compat-seam--_compat__init__py--_compatv2py)
3. [Area 2 — Request-handling spine (`routing.py`)](#2-the-request-handling-spine--routingpy)
4. [Area 3 — DI resolver (`dependencies/utils.py`)](#3-the-di-resolver--dependenciesutilspy)
5. [Area 4 — Hand-defused cycles](#4-the-hand-defused-cycles--undocumented-structural-invariants)
6. [Area 5 — Auth schemes + schema linchpin](#5-auth-schemes--their-schema-linchpin--security-and-openapimodelspy)
7. [Cross-cutting recommendations](#cross-cutting-recommendations-from-the-contact-map)
8. [Method & caveats](#method--caveats)
9. [Appendix A — Key contributors by area (12 mo)](#appendix--key-human-contributors-by-area-last-12-months)
10. [Appendix B — Top 10 `fastapi/` contributors (12 mo)](#appendix-b--top-10-human-contributors-to-fastapi-last-12-months)

---

## The five contact areas — at a glance

| # | Area | Why you must talk to someone | Primary contact | Best second opinion |
|---|------|------------------------------|-----------------|---------------------|
| 1 | **`_compat/__init__.py` + `_compat/v2.py`** — Pydantic-v2 seam | Buggiest file (48% fix) **and** its correctness rests on an *undocumented import-ordering invariant*; a moving target all year | tiangolo | Motov Yurii, Sofie Van Landeghem, Victorien |
| 2 | **`routing.py`** — request-handling spine | **83% single-owner** on a 6385-LOC engine coupled to *private* Starlette internals; effectively one person holds the model | tiangolo | Sofie Van Landeghem |
| 3 | **`dependencies/utils.py`** — DI resolver | #1 hotspot (churn 74, 41% fix); knowledge is *broad but shallow* (18 authors) — no single person owns the whole flow | tiangolo | Motov Yurii, Sofie Van Landeghem |
| 4 | **The hand-defused cycles** — `routing↔utils`, param ring, `_compat` back-edge | Correctness depends on `TYPE_CHECKING` guards & deferred imports that look like "cleanup bait"; the *reason* they exist is oral tradition | tiangolo | (whoever placed the guard — all tiangolo) |
| 5 | **`security/*` + `openapi/models.py`** — auth schemes & their schema linchpin | Low-churn / high-bug (50% / 44%), *most distributed* ownership (no clear owner), and a hidden cross-package schema contract | Motov Yurii | tiangolo; scheme-specific PR authors |

**One-line meta-finding:** four of five areas route back to **tiangolo** as the only person with
full context, and the *invariants* that make the code work (import order, type-only back-edges,
deferred imports) are **nowhere documented in the tree** — they live only in his head and in git
archaeology. The single most valuable "contact" outcome would be to get those invariants written
down. The two recurring second-owners worth engaging early are **Sofie Van Landeghem** and
**Motov Yurii**, who appear on the fragile core repeatedly.

---

## 1. The Pydantic-v2 compat seam — `_compat/__init__.py` + `_compat/v2.py`

**Contact-necessity: 🔴 highest.** This is the one place where *both* the bug data and the
structure analysis scream, and where a plausible-looking "fix" can break startup.

- **What the artifacts say.** `_compat/v2.py` is artifact-1's **buggiest file (48% fix rate)** and
  the "Pydantic-v2 blast-radius hub" (§4/§6); artifact-2 identifies the **only genuine top-level
  runtime import cycle** here (`_compat/__init__.py ↔ _compat/v2.py`), which *survives only by line
  ordering* — `__init__.py` must define `lenient_issubclass`/`shared` on lines 1–21 **before**
  triggering `.v2` on line 22, or startup dies with a partial-init `ImportError`.
- **Why contact is required, not optional.** That ordering constraint is **invisible and
  undocumented** — there is no comment warning a future editor not to reorder the imports. It is
  also a *moving target*: `_compat` went monolith → package (Oct 2025), grew a v1/v2-mixing feature,
  then had it ripped out (Dec 2025) (artifact-1 §8). Anyone touching it needs the *why* behind the
  current shape.
- **Ownership (git, 3y):** tiangolo 15 · **Sofie Van Landeghem 4 · Motov Yurii 4** · Victorien 1 ·
  Vincent Grafé 1. The invariant lines themselves (`_compat/__init__.py:1–25`) are **100%
  tiangolo** (commits `d34918ab`, `e3006305`, `2e7d3754`).
- **Who to contact & what to ask.**
  - **tiangolo** — the only person who can confirm whether the import order is deliberate or
    incidental, and whether the cheap decouple artifact-2 proposes (import `lenient_issubclass`/
    `shared` directly from `._compat.shared` in `v2.py:18`) is safe or was already tried.
  - **Motov Yurii / Sofie Van Landeghem** — the most active non-BDFL hands on the compat layer;
    good for reviewing a Pydantic-version-matrix change.
- **Ask before you touch:** "Is the `__init__.py` import order load-bearing? Can `v2.py` pull those
  two names from `.shared` directly?" and "Which Pydantic patch releases have broken this before?"
- **📖 Read before you change:**
  - **The invariant lines' history** — `git show d34918ab e3006305 2e7d3754` on
    `_compat/__init__.py:1–25`: shows *how* the import-ordering constraint reached its current shape
    (the single most important thing to understand before editing this file).
  - **The monolith → package split (Oct 2025)** and **the v1/v2-mixing feature added then removed
    (Dec 2025)** — see artifact-1 §8; explains why the seam looks the way it does today.
  - **The v1-drop / `pydantic.v1`-bridge endgame** — `#14575`, `#14609`, `#14583`, `#14605` and the
    compat refactors `#14856/57/60/62`: the decisions that shaped what compat still has to support.
  - **Version-compat regressions caught late** — Sofie Van Landeghem's `#15101` (3.14 / 2.12.1) and
    `#14361` (`$ref` remapping): the edge cases a Pydantic/Python bump tends to re-break.

## 2. The request-handling spine — `routing.py`

**Contact-necessity: 🔴 high (bus factor).** The sharpest single-owner risk in the repo.

- **What the artifacts say.** artifact-1 §11: **83% of `routing.py` commits are one author**
  (tiangolo) — the highest concentration on any fragile file. artifact-2 §1: it's the *true #1
  hotspot* (fan-out 12, churn 54, **6385 LOC**) and imports **private Starlette internals**
  (`starlette._exception_handler`, `starlette._utils`) that are unstable and unmockable.
- **Why contact is required.** With one person holding 83% of the history on the request lifecycle,
  and the file too large + too coupled to reason about in isolation, a reviewer with the full model
  is a hard dependency for any non-trivial change. Tests are the *only* other safety net, and
  artifact-2 flags this file as **e2e-only** (you can't unit-test it) — so you can't lean on unit
  coverage to substitute for owner review.
- **Ownership (git, 3y):** **tiangolo 41** · Timon 1 · Tamir Duberstein 1 · Sofie Van Landeghem 1 ·
  Sepehr Shirkhanlu 1 (a long tail of one-off contributors).
- **Who to contact & what to ask.**
  - **tiangolo** — non-negotiable reviewer for lifecycle/streaming/background-task/response-class
    changes.
  - **Sofie Van Landeghem** — one of the very few others who has landed here; a realistic second
    reader.
  - Whoever last touched the *specific* concern (e.g., Starlette-internals coupling) — `git blame`
    the exact lines before proposing a change.
- **Ask before you touch:** "Is the coupling to `starlette._*` intentional / is there a public
  alternative?" and "What's the intended split boundary if `routing.py` is ever decomposed?"
- **📖 Read before you change:**
  - **The Starlette-internals coupling** — `git blame` the imports of `starlette._exception_handler`
    / `starlette._utils` (artifact-2 §1): understand *why* private APIs are used before proposing a
    public swap.
  - **The recent lifecycle/streaming feature set** — SSE & JSON-Lines/binary streaming (`#15030`,
    `#15022`), the `app.frontend()` family (`#15800/863/908`), `on_event` re-implementation
    (`#14851`): the surface most likely to interact with any request-lifecycle change.
  - **Router-composition & DX edge cases** — Javier Sánchez Castro's router-includes-itself error,
    Savannah Ostrowski's endpoint-metadata-in-tracebacks (~137 LOC), and secrett2633's
    `inspect.getcoroutinefunction()` + `mock.patch` compat: small, easy-to-regress corners.

## 3. The DI resolver — `dependencies/utils.py`

**Contact-necessity: 🟠 high (broad-but-shallow knowledge).** The opposite failure mode from
`routing.py`: *many* contributors, but no one owns the whole thing.

- **What the artifacts say.** artifact-1: the **gravitational center** (46 changes, **41% fix**,
  steadiest file 8/13 months) yet the **healthiest bus factor — 18 authors, top share only 48%**.
  artifact-2: the "silent hotspot" (highest churn 74, tiny fan-in 2) that resolves dependencies by
  `inspect`-ing signatures and driving an `AsyncExitStack` — a huge mock surface, **e2e-magnet**.
- **Why contact is required.** Precisely *because* ownership is distributed, no single reviewer
  carries the end-to-end model; understanding a change's blast radius means pulling in *whichever*
  contributor last touched the relevant resolution path. The 41% fix rate means changes here
  regress easily. This is the area where "ask the right person for *this* code path" matters more
  than "ask the owner."
- **Ownership (git, 3y):** tiangolo 43 · **Motov Yurii 6 · Sofie Van Landeghem 3** · Victorien 1 ·
  Thomas LÉVEIL 1 · (…18 total).
- **Who to contact & what to ask.**
  - **tiangolo** for the overall resolution semantics.
  - **Motov Yurii** — the most active second hand on the DI engine; good for sub-dependency /
    caching / `Depends` behavior.
  - `git blame` the specific resolution branch you're changing and pull in that author.
- **Ask before you touch:** "Does this dependency-shape change interact with `AsyncExitStack`
  teardown / sub-dependency caching?" and "Which characterization tests cover this path?"
- **📖 Read before you change:**
  - **The scope / `yield` / caching core** — tiangolo's `#14262`, `#14419`, `#14448`, `#14372`:
    the decisions governing dependency scopes, `yield` exit ordering, and caching — the semantics
    most changes here have to preserve.
  - **Then `git blame` your specific resolution path** and read *that* author's PR — the long tail is
    a set of narrow, independently-fragile edge cases:
    - annotation evaluation — Mickaël Guérin (PEP 649 / 3.14), chaen (stringified annotations on
      3.10), Albin Skott (PEP 695 `TypeAliasType`);
    - security-scope propagation — Kristján Valur Jónsson;
    - `__call__`-instance & `computed_field` deps — Motov Yurii (`#14458`, `#14453`);
    - form/parse corners — Thomas LÉVEIL (`File` after `Form`), Marin Postma (empty-string → `None`).

## 4. The hand-defused cycles — undocumented structural invariants

**Contact-necessity: 🟠 high (tacit knowledge / "cleanup bait").** The most *insidious* category:
the code works because of deliberate workarounds that look like mistakes.

- **What the artifacts say.** artifact-2 §"Focused cycles": several scary-looking cycles **do not
  bite at runtime only because they were hand-defused** —
  - `utils.py:22` imports `routing.APIRoute` **only under `if TYPE_CHECKING`** (breaks the
    `routing ↔ utils` cycle);
  - `_compat/v2.py:350` (`from fastapi import params`) and `datastructures.py:148`
    (`from ._compat.v2 import …`) are **deferred function-local imports** breaking the param ring.
  Artifact-2's explicit **"do-no-harm rule": treat these as invariants; a refactor that 'cleans them
  up' into normal top-level imports will reintroduce hard cycles.**
- **Why contact is required.** A well-meaning contributor (or a linter/auto-fixer) will "tidy" a
  `TYPE_CHECKING` guard or hoist a function-local import — and reintroduce an import cycle that may
  only fail on a specific import path. **There is no comment on most of these guards explaining why
  they must stay.** The knowledge that "this ugliness is load-bearing" exists only in the reviewer's
  head. `git blame` shows all these guards are **tiangolo-authored** (e.g. `utils.py:22–23` from
  `8a0d4c79`, 2022).
- **Who to contact & what to ask.**
  - **tiangolo** — placed the guards; the authority on which are load-bearing vs. incidental.
  - Whoever proposes any `_compat` / `routing` / `params` refactor should surface these *first*.
- **Ask before you touch:** "Is this `TYPE_CHECKING` guard / deferred import safe to hoist, or does
  it break an import cycle?" — and, ideally, **push to add a `# noqa: keep-lazy — breaks
  <X↔Y> cycle` comment** so the next person doesn't have to ask. (Best long-term outcome of the
  contact: convert oral tradition into an in-tree comment.)
- **📖 Read before you change:** the four load-bearing guards themselves, in order of surprise —
  - `utils.py:22–23` `TYPE_CHECKING` import of `routing.APIRoute` — origin commit `8a0d4c79` (2022);
    breaks the `routing ↔ utils` cycle.
  - `_compat/v2.py:350` `from fastapi import params` (function-local) and `datastructures.py:148`
    `from ._compat.v2 import …` (function-local) — break the param ring.
  - artifact-2's **"do-no-harm rule"** on these (the explicit statement that hoisting them
    reintroduces hard cycles) — read it before any `_compat` / `routing` / `params` refactor.
  - Colin Watson's Pydantic-2.12.0-compat change (`params.py`) as an example of touching this knot
    safely.

## 5. Auth schemes + their schema linchpin — `security/*` and `openapi/models.py`

**Contact-necessity: 🟡 moderate but sharp-edged.** Low volume, high risk, *no clear owner*, plus a
hidden cross-package coupling.

- **What the artifacts say.** artifact-1: `security/oauth2.py` = **50% fix**, `api_key.py` /
  `dependencies/models.py` in the **low-churn / high-bug (44%)** band, auth is a "persistent
  trickle" (§10) — *and* it has the **most distributed ownership** (oauth2 top share only 38%).
  artifact-2 §A/§B: **`openapi/models.py` is the hidden linchpin — all five `security/*` modules
  import it**, and `security → openapi` (8 names, all *scheme models*) is a schema contract, so
  "a schema-model edit is a quiet way to break every auth scheme at once."
- **Why contact is required.** The combination is nasty: changes are *rare* (so no one has fresh
  context), *bug-dense* (so they regress), *distributed* (so there's no single owner to ask), and
  *invisibly coupled* (an `openapi/models.py` edit ripples into all auth schemes). You need both the
  scheme-specific author **and** awareness of the schema linchpin.
- **Ownership (git, 3y):** `oauth2.py` — tiangolo 6 · **Motov Yurii 2** · a long tail (Sun Bin,
  Salar Nosrati-Ershad, Rahul Pai, Rafal Skolasinski…). Genuinely spread out.
- **Who to contact & what to ask.**
  - **Motov Yurii** — the most consistent recent hand across security + DI + compat; the closest
    thing to a go-to for auth.
  - **tiangolo** — for the OAuth2/OpenID scheme semantics and the `openapi/models.py` contract.
  - The **PR author of the specific scheme** you're changing (`git blame` the file) — ownership is
    thin enough that the original author is often the only one who remembers the edge cases.
- **Ask before you touch:** "Does this security-scheme change require a matching `openapi/models.py`
  edit, and does that edit affect the *other* schemes?" and "What real-header/malformed-token cases
  regressed here before?" (artifact-2 recommends `TestClient` integration with real headers, not
  Request mocks — worth confirming with the reviewer).
- **📖 Read before you change:**
  - **The schema linchpin** — artifact-2 §A/§B on `openapi/models.py`: *all five* `security/*`
    modules import it, so read the `security → openapi` model contract before editing either side.
  - **Real-header / credential-parsing regressions** — Cecilia Madrid's strip-whitespace-from-
    `Authorization` fix and Motov Yurii's 401-on-missing-credentials (`#13786`, `#14266`): the
    malformed-token edge cases most likely to re-break.
  - **OpenAPI `type`-field shape** — sammasak's array-values-for `type`: the kind of quiet schema
    edit that ripples into every scheme.

---

## Cross-cutting recommendations (from the contact map)

1. **Engage `tiangolo` early on areas 1–4.** He is the sole full-context owner of the spine, the
   compat seam, and every load-bearing invariant. This is systemic bus-factor risk (artifact-1 §11),
   not a per-file quirk — plan reviews around his availability for anything in the fragile core.
2. **Treat `Sofie Van Landeghem` and `Motov Yurii` as the de-facto second owners.** They are the
   only names recurring across `routing.py`, `dependencies/utils.py`, `_compat/v2.py`, `params.py`,
   `encoders.py`, and `security/oauth2.py`. Pulling one of them into a fragile-core review is the
   cheapest way to reduce single-owner risk.
3. **The highest-leverage *outcome* of contacting contributors is documentation, not approval.**
   The recurring theme is that critical invariants (import ordering in `_compat/__init__.py`, the
   `TYPE_CHECKING`/deferred-import guards, the `openapi/models.py`→auth coupling) are **undocumented
   in the tree**. Every contact conversation should end with an in-tree comment or a `CONTRIBUTING`
   note so the next change doesn't need the same conversation.
4. **Match the contact to the failure mode.** `routing.py` = *ask the owner* (one person holds it);
   `dependencies/utils.py` & `security/*` = *ask the path author* (knowledge is distributed);
   the hand-defused cycles = *ask before you "clean up"* (the risk is a well-intentioned refactor).

---

## Method & caveats

- **Ownership numbers** are `git log --since=2023-07 --pretty=%an -- <file>`; they follow current
  paths only, so the `_compat.py` → `_compat/` rename and pre-window history are not tracked
  (same caveat as artifact-1 §11). `dependabot[bot]` counts are excluded from the "who to ask" lists.
- **Names** are taken verbatim from git author fields (de-mojibaked where needed) and used only to
  route review/questions. No email addresses are published here. Contributor availability, current
  involvement, and preferred contact channel are **not** knowable from git history — confirm via the
  project's normal channels (GitHub issues/discussions) before assuming someone is reachable.
- **This is a risk/knowledge map, not an org chart.** "Contact" means "this change benefits from a
  human with context"; it does not assign responsibility or obligate anyone named.

---

# Appendix — Key human contributors by area (last 12 months)

> **Window:** 2025-07-20 → 2026-07-20 (`git log --since=2025-07-20 --no-merges` per area path).
> **Filtering applied** (per request):
> - **Bots/automations excluded:** `dependabot[bot]` (the only bot touching `fastapi/` in the
>   window — 1 commit, a `ty` version bump). No `github-actions[bot]`, `pre-commit-ci[bot]`, or
>   translation bots appear in these code paths.
> - **AI-agent commits excluded:** a scan of author names/emails found **zero** commits authored by
>   Claude, Codex, Copilot, or any `noreply@anthropic`/`openai`/`copilot` identity in `fastapi/`
>   over the window. Nothing had to be dropped on this basis — noted so the absence is explicit.
> - **Data-quality flag (kept, not an agent):** one commit shows the author name literally as
>   `[object Object]` (a broken git display-name) but carries a real human email
>   (`lucas.wiman@gmail.com`, old squash-merged PR #5077, wrapped-function support). Treated as the
>   human contributor **Lucas Wiman**, not an automation.
> - **Identity merge:** `Motov Yurii` and `Yurii Motov` are one person (same GitHub id
>   `109919500+YuriiMotov`) — counted together.
>
> **Caveat on counts:** many high-touch commits are tiangolo's *squash-merged* PRs (e.g. "Drop
> support for `pydantic.v1`") that sweep many files, so his per-area counts overstate area-specific
> effort. Non-BDFL contributors' commits are far more area-targeted — that is what makes them the
> useful "who can help here" signal.

## The three core supporters (recurring across the fragile core)

Three humans appear again and again across areas 1–5. Their thematic profiles below are the most
reusable answer to "who can offer support."

### Sebastián Ramírez — `tiangolo` (BDFL; present in every area)
The owner of all five areas; contact of last resort everywhere. Thematically, his *recent* work
clusters into:
- **Pydantic v1→v2 endgame** — dropping v1, the temporary `pydantic.v1` bridge, deprecation
  warnings (`#14575`, `#14609`, `#14583`, `#14605`), internal compat refactors (`#14856/57/60/62`).
- **DI lifecycle & scopes** — dependency scopes, `scope="request"`/`"function"`, `yield` exit
  ordering, caching, wrapped/partial callables (`#14262`, `#14419`, `#14448`, `#14372`).
- **Request-lifecycle features** — SSE, JSON-Lines/binary streaming, the `app.frontend()` feature
  family, `on_event` re-implementation (`#15030`, `#15022`, `#15800/863/908`, `#14851`).
- **Python-version & typing modernization** — drop 3.8/3.9, upgrade internal syntax, Python 3.10
  types (`#14559`, `#14564`, `#14897/98`).

### Motov Yurii — `YuriiMotov` (15+ commits; areas 1, 2, 3, 4, 5 — the broadest non-BDFL reach)
The single most valuable **second contact** across the fragile core. Thematic groups:
- **🎯 Parameter / alias parsing (his signature area)** — parameter aliases, Query/Header/Cookie
  model aliases, extra non-body & `Form` parameter lists (`#14371`, `#14360`, `#14356`, `#14303`).
- **Security schemes** — 401 status codes when credentials missing, security schemes at the
  top-level app (`#13786`, `#14266`).
- **DI edge cases** — class-with-`__call__` dependencies, `computed_field` + separate schemas
  (`#14458`, `#14453`).
- **Tooling & docstrings** — run mypy via pre-commit, docstring links/fixes (`#14806`, `#14776`,
  `#14944`).
→ **Best for:** parameter/validation parsing, auth-scheme behavior, DI corner cases.

### Sofie Van Landeghem — `svlandeg` (6–7 commits; areas 1, 2, 3, 5)
The **Pydantic-version-compat & static-typing** specialist. Thematic groups:
- **Pydantic version compatibility** — update v2 code for deprecations, Pydantic v1-compat warnings
  for Python 3.14 / Pydantic 2.12.1, `$ref` remapping compat (`#15101`, `#14186`, `#14361`).
- **Static analysis & tooling** — add `ty` to pre-commit, install the `pydantic.mypy` plugin
  (`#15091`, `#14081`).
→ **Best for:** "does this survive the next Pydantic / Python release?" and typing/tooling review.

## Per-area contributor tables

Legend: **commits** = commits touching that area's paths in the window (bots/agents already removed).
"Thematic focus" summarizes *what* they touched there.

### Area 1 — Pydantic-v2 compat seam (`fastapi/_compat/`)

| Contributor | Commits | Thematic focus in this area | Support value |
|---|--:|---|---|
| Sebastián Ramírez | ~15 | v1 removal, compat-util refactors, JSON-schema for files, Python-3.10 types | Owner |
| **Motov Yurii** | 4 | optional-sequence serialization, param aliases, `computed_field` schemas, mypy tooling | 🔑 second owner |
| **Sofie Van Landeghem** | 4 | Pydantic v2 deprecations, 3.14/2.12.1 compat, `$ref` remapping | 🔑 version-compat |
| Victorien (`Viicos`) | 1 | optional-sequence w/ Python-3.10 union syntax (Pydantic-core maintainer) | Deep Pydantic internals |
| Vincent Grafé | 1 | computed-field OpenAPI schema w/ `separate_input_output_schemas` | Narrow: computed fields |

### Area 2 — Request-handling spine (`fastapi/routing.py`, `fastapi/utils.py`)

| Contributor | Commits | Thematic focus in this area | Support value |
|---|--:|---|---|
| Sebastián Ramírez | ~25 | SSE/streaming, `app.frontend()`, exit-stack/yield lifecycle, `on_event`, APIRoute typing | Owner (83% history) |
| **Motov Yurii** | 2 | `on_startup`/`on_shutdown` params of `APIRouter`, mypy tooling | 🔑 second reader |
| **Sofie Van Landeghem** | 2 | Pydantic-compat warnings, `ty` pre-commit | 🔑 version-compat |
| Javier Sánchez Castro | 1 | clear error on router-includes-itself | Router-composition edge cases |
| Savannah Ostrowski | 1 | endpoint metadata in tracebacks | DX / error reporting |
| Alex Colby | 1 | unformatted `{type_}` in `FastAPIError` | Narrow bugfix |
| secrett2633 | 1 | `inspect.getcoroutinefunction()` + `unittest.mock.patch` | Coroutine-detection/testing |

### Area 3 — DI resolver (`fastapi/dependencies/`)

Distributed knowledge — the long tail here is real: each one-off author is the narrow expert on a
specific resolution path.

| Contributor | Commits | Thematic focus in this area | Support value |
|---|--:|---|---|
| Sebastián Ramírez | ~20 | scopes/`yield`, caching, dataclass refactor, wrapped/partial, cyclic-recursion cleanup | Owner |
| **Motov Yurii** | 6 | param/alias & extra-param-list parsing, `__call__` deps, security schemes in OpenAPI | 🔑 second owner |
| **Sofie Van Landeghem** | 3 | Pydantic-v2 deprecations, compat warnings, `ty` tooling | 🔑 version-compat |
| Jonathan Fulton | 1 | `Response` type-hint as dependency annotation | `Response`-injection |
| Mickaël Guérin | 1 | `TYPE_CHECKING` annotations for Python 3.14 (PEP 649) | Deferred-annotation eval |
| chaen | 1 | stringified-annotation evaluation on Python 3.10 | Forward-ref evaluation |
| Albin Skott | 1 | PEP 695 `TypeAliasType` support | Modern type-alias support |
| Kanetsuna Masaya | 1 | `Json[list[str]]` handling | JSON-typed params |
| Anton (`retwish`) | 1 | error message for invalid query-param annotations | DX / error messages |
| Lie Ryan · Matthew Martin · Lucas Wiman | 1 each | `functools.partial` / wrapped-dependency support | Wrapped-callable deps |
| Kristján Valur Jónsson | 1 | hierarchical security-scope propagation | Security-scope resolution |
| luzzodev | 1 | `Depends(func, scope='function')` at top level | Dependency scopes |
| Thomas LÉVEIL | 1 | validation when `File` declared after `Form` | Form/File ordering |
| ad hoc (`Marin Postma`) | 1 | empty-string form values → `None` | Form parsing |
| Robert Hofer | 1 | `None` return type for bodiless responses | Return-type handling |
| Evgeny Bokshitsky | 1 | `dependency-cache` dict creation | DI caching |
| rmawatson | 1 | `arbitrary_types_allowed` with single arg | Arbitrary-type params |

### Area 4 — Hand-defused cycles / param knot (`params.py`, `datastructures.py`, + area 1/2 files)

| Contributor | Commits | Thematic focus in this area | Support value |
|---|--:|---|---|
| Sebastián Ramírez | ~10 | `Depends`/`Security` hashability, scopes, dataclass refactor, `annotated_doc` migration | Owner of the invariants |
| **Motov Yurii** | 3 | param aliases, `max_digits`/`decimal_places` & docstring links | 🔑 param specialist |
| Colin Watson | 1 | Pydantic 2.12.0 compatibility | Version-compat |

> The load-bearing `TYPE_CHECKING` / deferred-import guards themselves are **100% tiangolo** — no
> other contributor has authored them, so for cycle-invariant questions the second-owner bench is
> effectively empty. This reinforces area 4's "ask before you clean up" finding.

### Area 5 — Auth schemes & schema linchpin (`fastapi/security/`, `fastapi/openapi/models.py`)

| Contributor | Commits | Thematic focus in this area | Support value |
|---|--:|---|---|
| Sebastián Ramírez | ~5 | v1 drop, Python-3.10 types, `annotated_doc` migration, compat refactor | Owner |
| **Motov Yurii** | 2 | 401 status codes for missing credentials, docstring links | 🔑 auth-behavior |
| **Sofie Van Landeghem** | 2 | `ty` pre-commit, `pydantic.mypy` plugin | 🔑 tooling/typing |
| Cecilia Madrid | 1 | strip whitespace from `Authorization` header credentials | 🎯 header/credential parsing |
| sammasak | 1 | array values for OpenAPI schema `type` field | OpenAPI `type` schema |
| Kadir Can Ozden · alv2017 · Ahsan Sheraz | 1 each | docstring/typo fixes (OAuth2 forms) | Docs only — not code owners |

**Reads:**
- **Motov Yurii is the closest thing FastAPI has to a second maintainer of the fragile core** —
  the only non-BDFL present in *all five* areas, concentrated on parameter parsing, auth behavior,
  and DI edge cases. He is the highest-value single person to loop in on areas 1–5.
- **Sofie Van Landeghem is the version-compat safety net** — engage her (or Victorien for deep
  Pydantic-core questions) specifically when a change risks breaking a future Pydantic/Python
  release.
- **Area 3's expertise is genuinely distributed** — for a DI change, the *right* contact is often
  the narrow-topic author from the long tail (e.g. Mickaël Guérin/chaen for annotation evaluation,
  Kristján Valur Jónsson for security-scope propagation), not a single owner. `git blame` the exact
  resolution path.
- **Area 4 has no second owner for the invariants** — the cycle-breaking guards are tiangolo-only,
  so that specific knowledge cannot be sourced from anyone else in the window.
- **Area 5's non-BDFL, non-Motov activity is mostly docs/typo fixes**, not scheme logic — auth
  logic changes realistically need tiangolo or Motov Yurii, matching area 5's "no clear owner,
  high-bug" risk.

---

# Appendix B — Top 10 human contributors to `fastapi/` (last 12 months)

> **Window:** 2025-07-20 → 2026-07-20 (`git log --since=2025-07-20 --no-merges -- fastapi/`),
> **scoped to the `fastapi/` package only** (docs, tests, translations, tooling excluded) and
> ranked by commit count.
> **Excluded bots/automations:** `dependabot[bot]` (the only bot touching `fastapi/` in the
> window). **No** Claude/Codex/Copilot-authored commits exist. `Motov Yurii` + `Yurii Motov`
> merged (same GitHub id `109919500+YuriiMotov`). The repo owner of this map (`szaroket`) is
> **excluded**, and in any case never touches `fastapi/` (those commits live under `context/map/`).
>
> **Tie-break:** a clear top 4 by commit count, then a large cluster of single-commit authors —
> ranks 5–10 are ordered by **lines changed within `fastapi/`** (insertions + deletions).

| # | Contributor | Commits | Lines (fastapi/) | Most-updated `fastapi/` files (count) | Nature of the work |
|--:|---|--:|--:|---|---|
| 1 | **Sebastián Ramírez** (`tiangolo`) | 137 | 16,490 | `__init__.py` (76)\*, **`routing.py` (30)**, `dependencies/utils.py` (22), `_compat/v2.py` (15) | BDFL — every area; real-code center is `routing.py` + DI |
| 2 | **Motov Yurii** (`YuriiMotov`) | 16 | 608 | **`dependencies/utils.py` (6)**, `_compat/v2.py` (4), `security/oauth2.py` (2), `param_functions.py` (2) | DI & parameter-parsing, auth schemes |
| 3 | **Sofie Van Landeghem** (`svlandeg`) | 6 | 577 | **`_compat/v2.py` (4)**, `encoders.py` (3), `dependencies/utils.py` (3), `_compat/__init__.py` (3) | Pydantic-compat + static-typing |
| 4 | **Jonathan Fulton** | 2 | 7 | `openapi/utils.py` (1), `dependencies/utils.py` (1) | `Response`-as-dependency annotation |
| 5 | **Savannah Ostrowski** | 1 | 137 | `routing.py`, `exceptions.py` | Traceback / endpoint-metadata DX |
| 6 | **Colin Watson** | 1 | 34 | `params.py`, `_compat.py`† | Pydantic 2.12.0 compatibility |
| 7 | **Carlos Mario Toro** | 1 | 31 | `openapi/utils.py`, `applications.py` | OpenAPI generation |
| 8 | **Matthew Martin** | 1 | 22 | `dependencies/models.py` | Wrapped-dependency handling |
| 9 | **secrett2633** | 1 | 19 | `routing.py`, `dependencies/utils.py` | `inspect.getcoroutinefunction()` + `mock.patch` compat |
| 10 | **Vincent Grafé** | 1 | 14 | `_compat/v2.py` | Computed-field OpenAPI schema |

\* **Noise flag:** `tiangolo`'s `__init__.py` churn is `__version__` bumps (§5/§7) — process
artifact, not real editing. His top *real code* file is `routing.py` (30), consistent with his
83% ownership of the spine (§11).
† `_compat.py` was the pre-split monolith (retired to the `_compat/` package Oct 2025, §8); the
still-current file from that commit is `params.py`.

**Reads:**
- **Scoping to `fastapi/` strips the docs/i18n layer entirely.** The translators and web-frontend
  contributors that led the repo-wide list vanish — every name here is a **code** contributor. The
  top 3 are exactly the fragile-core trio from Appendix A (tiangolo, Motov Yurii, Sofie Van
  Landeghem); the ranking method and the risk analysis agree on who works the core.
- **The cliff is even steeper here.** tiangolo has **~8.5×** the commits of the #2 human and ~23× the
  #3; only four humans made more than one `fastapi/` commit all year. This is the BDFL concentration
  (§11) seen at the package level.
- **Ranks 5–10 are single-commit specialists** whose one change still lands on the hot files —
  `routing.py`, `dependencies/utils.py`, `_compat/v2.py`, `openapi/utils.py`. They match the
  long-tail "narrow expert" pattern in Appendix A's per-area tables: the right contact for a
  specific path, not general owners.
