---
date: 2026-07-22T00:00:00Z
researcher: szaroket
git_commit: c637954f07b3a4bd81c493e7610ce5c3036eb0bc
branch: master
repository: fastapi
topic: "Refactor opportunities from the routing analysis: candidate classification, current shape, intentionality, migration feasibility, and a ranked shortlist"
tags: [research, codebase, refactor, technical-debt, routing, openapi, starlette-vendoring, _APIRouteLike, feasibility, blast-radius]
status: complete
last_updated: 2026-07-22
last_updated_by: szaroket
---

# Research: Refactor Opportunities from the Routing Analysis

**Date**: 2026-07-22T00:00:00Z
**Researcher**: szaroket
**Git Commit**: c637954f07b3a4bd81c493e7610ce5c3036eb0bc
**Branch**: master
**Repository**: fastapi

## Research Question

The prior analysis `context/changes/routing-analysis/research.md` recorded the technical debt and
structural risks of the request spine but deliberately left open **which** problems are worth
fixing, in what target shape, and in what order. This change answers that — as an **exploration
only**: no code changes, no decision. Method:

1. Enumerate every problem the report records (any label), and classify each: **CANDIDATE** = a fix
   that would change *code structure*; everything else is retained as a feasibility/cost input.
2. Investigate each candidate through three read-only lenses: **current shape** (confirmed in code,
   `file:line`, evidence/inference/unknown), **history & intentionality** (deliberate constraint vs
   accidental complexity vs unknown), and **migration feasibility** (existing vs new abstraction,
   blast radius from the report, existing guards/CI, first prerequisite step).
3. Close with a ranked shortlist of the 2–3 strongest opportunities — a *proposal* for a later
   planning session, not a decision.

Findings from the routing-analysis report are treated as collected evidence and built upon, not
re-derived. The report's priors — `context/map/repo-map.md` and its three artifacts — were read as
priors.

> **Boundaries honored.** No code changed. Evidence precedes interpretation. No target architecture
> is designed beyond naming an adequate target shape per candidate. Where data is missing it is
> marked **unknown** rather than filled with a plausible guess.

---

## Summary

Six problems the report records reduce to **three structural candidates**; the other three (plus two
coverage/hygiene items) are not code-structure fixes and are retained as feasibility inputs.

Investigation reshapes the picture the report left:

- **C1 (`_APIRouteLike` seam)** is the strongest opportunity. The abstraction is already correct
  and deliberate (introduced whole in one commit, `#15745`, because a second non-`APIRoute` type
  must also conform). Its only defect is that it is **bypassed by `cast()`** at every producer site,
  so the strict mypy/ty already running in CI never verifies conformance. The first step is a
  ~one-line, additive, fully-reversible `TYPE_CHECKING` conformance assertion that converts the
  report's #1 *silent-runtime* seam into a *compile-time* error with **zero runtime change**. Highest
  debt-to-cost ratio on the board.

- **C3 (`routing.py` monolith / fused dispatch)** is second. The four spine functions are **already
  module-level and importable** — they simply have no unit tests; the four-way dispatch is fused into
  one nested closure. The cheap, standalone first step (direct-call characterization tests, **zero
  code change**) fills the report's exact "e2e-only, no unit tests" gap and is the prerequisite for
  any later extraction. A proven in-repo pattern exists (`test_router_include_context.py`).

- **C2 (vendored Starlette internals)** drops out as a near-term *structural* refactor. The vendoring
  is deliberate, documented, and **forced by upstream** (Kludex/starlette#3117 removed symbols FastAPI
  depends on); two of the copies are customized to splice FastAPI's exit-stacks into the ASGI path and
  therefore *cannot* be un-vendored. Its genuine improvement is **drift-detection tooling (a test)**,
  not restructuring — so per the boundary rule it is named and set aside.

A decisive cross-cutting finding: **CI enforces 100% *line* coverage** (`coverage report
--fail-under=100`, no `branch = true`) plus strict `mypy`+`ty`. So the report's "UNCOVERED" defensive
branches are in fact line-executed (else CI is red); the real gap is **failure-localization
granularity**, not line execution — which is exactly what C3's first step addresses.

---

## Candidate classification (audit list)

Every problem the routing-analysis report records, classified. **CANDIDATE** = fixing it changes
*code structure*.

| # | Problem (as recorded) | Report label | Classification |
|---|---|---|---|
| TD#1 | Silent structural coupling routing↔OpenAPI via the 41-field `_APIRouteLike` Protocol | HIGH | **CANDIDATE C1** |
| TD#2 | Vendored copies + private-import coupling to Starlette internals | HIGH | **CANDIDATE C2** |
| TD#3 | `routing.py` 6385 LOC + eager handler compile + four-way dispatch fused in `get_request_handler` | MED/HIGH | **CANDIDATE C3** |
| TD#4 | Dangling `docs_src` import to deleted `temp_pydantic_v1_params` module | LOW | Not a candidate — one-line cleanup |
| TD#5 | Untested defensive error branches on request path | MED | Not a candidate — test gap (input) |
| TD#6 | Repo-map wording reconciliation (`TYPE_CHECKING` back-edge attribution) | LOW | Not a candidate — doc gap |
| Cov. | Core handler spine has **no unit tests**, e2e-only | — | Not a candidate — test gap; folds into C3 |
| Prior | Load-bearing hand-defused cycles + `_compat` import-ordering invariant (from repo-map) | — | Not a candidate — "do not tidy"; out of spine scope |

**Three candidates investigated: C1, C2, C3.**

---

## C1 — `_APIRouteLike` structural Protocol seam (routing.py ↔ openapi/utils.py)

### Current shape (evidence)

- `_APIRouteLike(Protocol)` defined at `fastapi/routing.py:902-944`, declaring **41 fields**, one per
  line (`:903-943`). [evidence] (Prior anchor said `902-945`; class ends at `944` — minor drift, still
  41 fields.)
- The seam has a **real import edge**, not "purely structural": `fastapi/openapi/utils.py:8` does
  `from fastapi import routing`; all seven consumption sites reference it qualified as
  `routing._APIRouteLike` (`openapi/utils.py:216,230,237,262,332,483,485`), and openapi/utils.py also
  reaches into `routing.APIRoute` at `:326`. [evidence] The **name is imported**; only the *conformance
  of route objects* is duck-typed. (Corrects the prior "no import cycle" framing for this seam.)
- OpenAPI reads **24 of the 41** declared fields through the seam (`operation_id, path_format, name,
  summary, tags, description, unique_id, endpoint, deprecated, methods, response_class,
  include_in_schema, dependant, body_field, callbacks, status_code, response_description,
  is_json_stream, stream_item_field, is_sse_stream, response_field, responses, response_fields,
  openapi_extra`). [evidence] The remaining ~17 (the `response_model_*` family, `path_regex`,
  `param_convertors`, `_flat_dependant`, `_embed_body_fields`, …) are declared but consumed elsewhere.
  [inference]
- **Two first-class conformers**, not one: `APIRoute` (populated via `_populate_api_route_state`,
  cast at `routing.py:1179,1211`) **and** `_EffectiveRouteContext` (a `@dataclass`, `routing.py:1359`;
  populated at `:1420`, cast at `:1216,1421`). Callback `APIRoute`s are additionally cast at
  `openapi/utils.py:332`. [evidence] This is *why* the contract is a structural Protocol rather than a
  base class — two unrelated types must satisfy it.
- **Conformance is unchecked by the type-checker.** Every producer launders the object through
  `cast()`: `cast(_APIRouteLike, self)` at `routing.py:1179,1211,1216,1418,1421` and
  `cast(routing._APIRouteLike, …)` at `openapi/utils.py:332,485`. `cast()` suppresses structural
  checking; the only real annotation (`_populate_api_route_state(route: _APIRouteLike, …)`,
  `routing.py:947`) is reached via the cast at `:1179`. There is **no** un-cast assignment of an
  `APIRoute` into a `_APIRouteLike` slot anywhere. [evidence]

### Intentionality verdict — **deliberate constraint (load-bearing decoupling seam)**

- Introduced **whole in a single commit**: `8e1d774ce` — "♻️ Refactor internals to preserve
  `APIRouter` and `APIRoute` instances (#15745)" (2026-06-14). `git log -S "_APIRouteLike"` returns
  only this commit — it did **not** grow field-by-field. [evidence]
- Before it, `openapi/utils.py` imported `routing.APIRoute` directly and branched on
  `isinstance(route, routing.APIRoute)`; the same commit replaced those with `route: routing._APIRouteLike`
  and a `_get_api_route_for_openapi(...)` helper, and added `_EffectiveRouteContext`. [evidence]
- **Why a Protocol:** OpenAPI must now consume two unrelated types — real `APIRoute`s and
  `_EffectiveRouteContext` proxies — presenting the same attribute surface. Structural typing is the
  mechanism that lets both satisfy `get_openapi_path()`. The seam is the whole point of the
  "preserve instances / include-context" machinery. [evidence/inference]

### Feasibility notes

- **Existing abstraction, not new.** `_APIRouteLike` is already the right shape. The defect is that
  it is *documentary, not enforced* — bypassed by `cast()`. Target shape = the same Protocol, made
  load-bearing to the type checker. This is a code-structure fix, not a business-concept redesign.
- **Guards that exist:** strict `mypy` (pydantic plugin, `strict=true`) and `ty` run in CI via
  `.pre-commit-config.yaml:39-51` / `.github/workflows/pre-commit.yml` — but the `cast()`s defeat
  conformance checking, so a route-field rename passes mypy/ty **green**. The 100%-line gate ensures
  OpenAPI-generation lines execute; whether any single test asserts the schema for a route exercising
  **all 41 fields** at once is **unknown** (no maximal-field test found). So a field rename can slip
  past both type-checking and line coverage. [evidence/inference]
- **First prerequisite step:** add a `TYPE_CHECKING`-only static conformance assertion that `APIRoute`
  (and `_EffectiveRouteContext`) satisfy `_APIRouteLike` **without a cast** — mypy/ty then fail the
  instant the type and Protocol diverge. Additive, one file, no import edge, no runtime change, trivially
  revertible. Only after that guard exists should the `cast()` sites be removed one at a time. [inference]
- **Blast radius (from the report):** `openapi/utils.py` co-changes with `routing.py` 13×/12mo (tied #1);
  seam consumed at 7 sites; the Protocol binds routing↔openapi. The first step touches **one file** and
  changes no runtime path, so its effective blast radius is ~zero.

---

## C2 — Vendored / private Starlette internals in routing.py

### Current shape (evidence)

- **Private-symbol imports** (`routing.py:83-85`): `starlette._exception_handler.wrap_app_handling_exceptions`
  (`:84`), `starlette._utils.get_route_path`, `starlette._utils.is_async_callable` (`:85`). `compile_path`
  is **imported** (public) at `:101` and used at `:799,1024,1680` — *not* vendored (confirms prior). [evidence]
- **Four vendored/copied local defs**, all present with "Copy of / Vendored from starlette" comments:
  `request_response` (`:113-152`, "modified to include the dependencies' AsyncExitStack"),
  `websocket_session` (`:157-178`), `_AsyncLiftContextManager` (`:185-204`, "Vendored … to avoid importing
  private symbols"), `_wrap_gen_lifespan_context` (`:208-222`). A fifth, `_DefaultLifespan` (`:242`, "copy
  of the Starlette `_DefaultLifespan` class that was removed in Starlette … Ref:
  https://github.com/Kludex/starlette/pull/3117"), is a related copy not in the prior list of four. [evidence]
- **Direct constructions** of `starlette.routing.Route` (`:1640`), `WebSocketRoute` (`:1666`), `Mount`
  (`:1688`), `Host` (`:1692`), plus a precise reach-in writing `.path_regex/.path_format/.param_convertors`
  at `:1677-1680` on a `copy.copy`'d Mount. [evidence]
- **No Starlette version pin in comments.** Only PR references are Kludex/starlette#3117 (`:250`, `:2489`).
  [evidence]

### Intentionality verdict — **deliberate constraint (documented, forced by upstream)**

Two sub-cases, both intentional:
- **(a) `request_response` / `websocket_session`** — *customized* copies, not dead vendoring. They inject
  FastAPI's dependency lifecycle (`scope["fastapi_inner_astack"]` / `fastapi_function_astack` and the
  "Response not awaited" `FastAPIError` guard, added by `e329d78f8` #14099) into the ASGI closure. They
  **cannot** be replaced by an import — splicing FastAPI behavior into Starlette's request path is their
  reason to exist. `request_response` traces to the first tracked commit; `websocket_session` to #178. [evidence]
- **(b) `_AsyncLiftContextManager` / `_wrap_gen_lifespan_context` / `_DefaultLifespan`** — *true* vendoring,
  added together in `f9f799260` — "Re-implement `on_event` … for compatibility with the next Starlette
  (#14851)" (2026-02-06). Comments state the motive: avoid importing private symbols / copy a class a newer
  Starlette **removed** (Kludex/starlette#3117), while keeping `on_startup`/`on_shutdown` working. [evidence]

All are upstream-FastAPI patterns (tiangolo commits, FastAPI PR numbers), **not** fork-local additions.

### Feasibility notes

- **The structural fix is largely infeasible/inadvisable as a near-term refactor.** Reducing the vendoring
  is low-reversibility and gated on upstream exposing public equivalents — outside this fork's control; the
  customized copies (a) cannot be un-vendored at all. Per the boundary rule, the candidate's genuine
  improvement is **observability, not restructuring**.
- **Existing guards:** Starlette is pinned (`starlette>=0.46.0` in `pyproject.toml:45`; `uv.lock` resolves
  `starlette==1.3.1`, Kludex lineage). CI already runs a **dual-source matrix** — `starlette-pypi` and
  `starlette-git` (`git+https://github.com/Kludex/starlette@main`, `test.yml:59-61,132-134`). That catches
  **import drift** of the private symbols (a removal turns the `starlette-git` leg red). It does **not** catch
  **behavioral drift** of the vendored copies — no test compares a copy against installed upstream. [evidence]
- **First prerequisite step (non-structural):** add `tests/test_starlette_vendoring.py` snapshotting upstream
  `inspect.getsource()` (or a stable property) for each vendored symbol against a checked-in expectation, so
  any upstream change forces a conscious re-vendor. Additive, runs in the existing suite, fully revertible;
  also answers the report's Open Question #1 (which upstream revision the copies track). [inference]
- **Blast radius:** under-counted by both the import graph (external) and git co-change (Starlette isn't in
  this repo) — the report's standing caveat. A detection test has ~zero blast radius.

---

## C3 — routing.py monolith / eager-compile / fused four-way dispatch

### Current shape (evidence)

- **6385 LOC** (`wc -l`, matches prior). 40 top-level `def`/`class` (14 classes, 26 functions);
  `APIRouter(routing.Router)` at `:2199` runs to EOF (~4180 LOC of the tail). [evidence]
- **Eager compile** confirmed: `self.app = request_response(self.get_route_handler())` is the last
  statement of `APIRoute.__init__` (`routing.py:1208`; `__init__` = `:1146-1208`). `get_route_handler`
  (`:1210-…`) resolves an optional effective-context override via a ContextVar (with an in-code `# TODO`
  to deprecate the no-scope hook) then calls `get_request_handler(...)`. [evidence]
- **Four-way dispatch fused in one nested closure.** `get_request_handler` is a **module-level** def
  (`:367-745`); the returned handler is the nested `async def app(request)` (`:398-743`). After dependency
  solving, inside `if not errors:` the branches are SSE `if is_sse_stream:` (`:512-635`), JSONL
  `elif is_json_stream:` (`:636-668`), raw-stream `elif …is_async_gen_callable/is_gen_callable:` (`:669-688`),
  and **normal** as the trailing `else:` (`:689-734`, calling `run_endpoint_function` `:690` then
  `serialize_response` `:711`). [evidence] (Prior listed normal "first"; it is the trailing `else`.)
- **Shared closure state** the four branches read: `dependant`, `stream_item_field`, `status_code`,
  `response_field`, the six `response_model_*` flags, `actual_response_class`, `is_coroutine`,
  `is_sse_stream`, `is_json_stream`; per-request `solved_result`, `endpoint_ctx`; and inner helpers
  `_serialize_data` (`:487-510`) / `_serialize_sse_item` (`:516`). All four assign the single closure-local
  `response`. [evidence]
- **Key structural fact for feasibility:** the four spine functions are **already module-level and
  importable** — `serialize_response` (`:293`), `run_endpoint_function` (`:336`), `_build_response_args`
  (`:349`), `get_request_handler` (`:367`), `get_websocket_app` (`:748`). The **dispatch branches are not**
  independent units — they are inline blocks inside the nested `app` closure, closing over the large captured
  set above. [evidence]

### Intentionality verdict — **deliberate feature accretion + intentional single-closure fusion; the un-split monolith itself is [unknown] intent**

- **Growth is feature accretion, not bug churn.** LOC trajectory: 785 (initial) → 673 → 4515 (#14099) →
  4657 (on_event #14851) → 4802 (JSONL #15022) → 4912 (SSE #15030) → 5596 (include-context #15745) → 6385
  (current). The repo-map corroborates: routing.py churn is "~22%… feature work, lower risk … churn from
  *features*, not bugs" (`context/map/artifact-1-territory.md`). [evidence]
- **Eager-compile is ancient upstream**, not a fork choice: `git blame` puts the line at `8c3ef76139`
  (dmontagu, 2019-10-04). [evidence]
- **Fused dispatch was deliberately bolted into the existing closure.** The SSE commit `223815584` diff
  inserts `+if is_sse_stream:` / `+elif is_json_stream:` **into the existing normal-path closure**; all four
  branches share the same fully-solved `solved_result` (DI, `background_tasks`, headers, exit-stacks) — this
  is intentional *reuse* of the solved dependency result, not incidental fusion. [evidence]
- **No split was ever attempted or forbidden.** `git log --grep=split/extract/module` on the file finds
  nothing; the project *does* split monoliths when it decides to (`_compat.py` → `_compat/` package, #14168),
  so routing.py's persistence is an **unrecorded default**, not a stated constraint — genuinely **unknown**
  intent. No `CLAUDE.md`/`AGENTS.md`/`CONTRIBUTING`/CI note forbids restructuring here. [evidence]

### Feasibility notes

- **Existing abstraction for step 1; new (small) abstraction for later steps.** The spine functions are
  already module-level; they just lack direct tests. Extracting the fused branches later needs a small
  captured-state container (a dataclass built once in `get_request_handler`). Not a business-concept redesign.
- **Guards that exist:** the **100%-line-coverage gate** (`coverage report --fail-under=100`, `test.yml:242`;
  no `branch=true`, so line- not branch-coverage) plus a dense e2e branch suite (`test_sse`, `test_stream_*`,
  `test_serialize_response`, `test_validate_response`, `test_dump_json_fast_path`, `test_validation_error_context`,
  `test_frontend`). Strong at detecting *that* something broke; weak at *localizing* it — the report's e2e-only
  concern restated precisely. [evidence]
- **Proven in-repo extraction pattern:** the include/context machinery is the best-tested part of the file
  *because it is expressed as module-level, individually-constructable units* —
  `tests/test_router_include_context.py` (~38 tests) imports internals straight from `fastapi.routing` and
  drives them by hand. Replicating that shape for the spine is exactly what unlocks the same unit-test density.
  [evidence]
- **First prerequisite step (zero code change):** write direct-call characterization unit tests for the
  already-module-level `serialize_response` / `run_endpoint_function` / `_build_response_args`, mirroring the
  `test_router_include_context.py` style, pinning current behavior *before* any extraction. This fills the
  report's exact gap (spine has no unit tests) and stands on its own value. Only then extract the shared stream
  serializer (`_serialize_data` / `_serialize_sse_item` → module-level), then, last and largest, the four-way
  dispatch. Each step is behavior-preserving and revertible per commit; the eager-compile capture-once
  semantics stay untouched. [inference]
- **Blast radius (from the report):** routing.py co-changes with `dependencies/utils.py` and
  `openapi/utils.py` 13× each and `applications.py` 9×/12mo — but **step 1 is tests-only, blast radius ~zero**;
  extraction steps stay within routing.py + its test files.

---

## Refactor opportunities (ranked proposal for a later planning session)

Ranked by debt cost vs change cost, on the evidence above. This is a proposal to be decided in the
planning stage — **not** a decision.

### #1 — C1: make the `_APIRouteLike` contract enforced instead of documentary

- **Current → target shape:** a 41-field structural Protocol that documents the routing↔OpenAPI contract
  but is bypassed by `cast()` at every producer site → the **same** Protocol, made load-bearing so strict
  mypy/ty (already in CI) verify conformance; casts removed incrementally.
- **Why #1 (debt vs change):** highest debt-to-cost ratio. Debt is the report's **#1 silent-runtime** risk
  (a route-field rename breaks generated schema at runtime, green mypy + green 100%-line-coverage). Change
  cost is near-zero: a `TYPE_CHECKING`-only conformance assertion — additive, one file, no runtime path,
  fully reversible — flips silent-runtime into compile-time immediately. Abstraction already exists and is
  deliberate; nothing to design.
- **Blast radius:** first step touches one file, no runtime change (~zero). Full cast removal touches
  `routing.py` (5 sites) + `openapi/utils.py` (2 sites); openapi/utils.py is the report's *loosest* member
  and safest to touch.
- **Incremental, reversible path:** (1) add `TYPE_CHECKING` conformance assertion for `APIRoute` **and**
  `_EffectiveRouteContext` → both must satisfy it; (2) with the guard green, remove `cast()` sites one at a
  time, letting real annotations flow; (3) optionally trim Protocol fields OpenAPI never reads, once the
  checker proves what is unused.
- **First prerequisite step:** add the un-cast `TYPE_CHECKING` conformance assertion; confirm mypy+ty stay
  green, then confirm they go **red** under a deliberate field rename (validates the guard bites).

### #2 — C2 is NOT here (see rejected); #2 is C3: give the request spine a unit-test seam, then decompose

- **Current → target shape:** four core spine functions module-level but untested, with a four-way dispatch
  (normal/SSE/JSONL/raw-stream) fused into one nested `app` closure → module-level, directly-tested spine
  functions plus branch handlers extracted behind an explicit captured-state object, mirroring the
  already-unit-tested include/context machinery.
- **Why #2 (debt vs change):** debt is real (MED/HIGH cognitive load; **coarse failure localization** —
  100%-line coverage proves lines run but the e2e-only suite can't pin a regression to a branch). Change cost
  of the *first* step is zero-code (tests only) and delivers standalone value by closing the report's exact
  "no unit tests" gap; but total scope is larger than C1 and the hardest step (splitting the fused closure)
  needs a new captured-state container, so it ranks below C1.
- **Blast radius:** step 1 tests-only (~zero); extraction steps stay within `routing.py` + its tests. Eager-
  compile semantics untouched, so the composition-time contract is preserved.
- **Incremental, reversible path:** (1) direct-call characterization tests for `serialize_response` /
  `run_endpoint_function` / `_build_response_args`; (2) promote `_serialize_data` / `_serialize_sse_item` to
  module-level, unit-test them; (3) extract each dispatch branch behind a captured-state dataclass, one branch
  per commit, each guarded by steps 1–2 + the 100%-line gate.
- **First prerequisite step:** write the direct-call unit tests for the three already-module-level functions,
  in the `test_router_include_context.py` style — no code change, immediate value, and the safety net every
  later extraction leans on.

*(A third structural opportunity is not offered: C2's honest fix is a drift-detection test, not a
restructuring — see rejected. Presenting two strong candidates rather than padding to three.)*

## Considered and rejected

- **C2 — vendored/private Starlette internals (as a *structural* refactor).** Rejected for now. The vendoring
  is deliberate, documented, and **forced by upstream** (Kludex/starlette#3117); the customized copies
  (`request_response`, `websocket_session`) *cannot* be un-vendored because they exist to splice FastAPI's
  exit-stacks into the ASGI path. Reducing the vendoring is low-reversibility and gated on upstream. The
  actionable, high-value item is **non-structural**: a `tests/test_starlette_vendoring.py` drift-detection
  test (the existing `starlette-git` CI axis already catches *import* drift; only *copy* drift is unguarded).
  Route it to the test-coverage backlog, not a refactor.
- **TD#4 — dangling `docs_src` import to deleted `temp_pydantic_v1_params`.** Not a structural fix — a one-line
  cleanup (delete/repair the stale import in `docs_src/pydantic_v1_in_v2/tutorial004_an_py310.py:4`). Hygiene.
- **TD#5 + coverage item — untested defensive branches / e2e-only spine.** Not structural on their own; the
  spine-testability part is subsumed by **C3 step 1**. Note: under the 100%-line gate these lines *are*
  executed — the gap is localization, not execution.
- **TD#6 — repo-map wording reconciliation.** Documentation clarity, not code.
- **Prior (repo-map) — load-bearing hand-defused cycles + `_compat` import-ordering invariant.** Explicitly
  flagged "do not tidy": hoisting the `TYPE_CHECKING` back-edge (`utils.py:22`) or reordering `_compat`
  imports reintroduces hard cycles / breaks boot. Out of spine scope and an anti-candidate.

---

## Code References

- `fastapi/routing.py:902-944` — `_APIRouteLike` Protocol (41 fields)
- `fastapi/routing.py:1179,1211,1216,1418,1421` — `cast(_APIRouteLike, …)` producer sites (conformance defeat)
- `fastapi/openapi/utils.py:8,216,230,237,262,326,332,483,485` — routing import + seam consumers
- `fastapi/routing.py:83-85,113-152,157-178,185-204,208-222,242` — private imports + vendored Starlette copies
- `fastapi/routing.py:1640,1666,1677-1680,1688,1692` — direct `starlette.routing.*` constructions + reach-in
- `fastapi/routing.py:293,336,349,367-745,748` — module-level spine fns + fused nested `app` closure
- `fastapi/routing.py:512-635,636-668,669-688,689-734` — SSE / JSONL / raw-stream / normal dispatch branches
- `fastapi/routing.py:1146-1208` — `APIRoute.__init__` eager compile (`:1208`)
- `.github/workflows/test.yml:59-61,132-134,242` — `starlette-pypi`/`starlette-git` axis + 100%-line gate
- `.pre-commit-config.yaml:39-51` — `mypy` (strict) + `ty` in CI
- `pyproject.toml:45,202-204,235` — starlette dep, mypy strict, coverage config (no `branch=true`)
- `uv.lock` (`starlette==1.3.1`) — pinned Starlette (Kludex lineage)
- `tests/test_router_include_context.py` — the module-level-unit-test pattern to replicate for the spine

## Architecture Insights

- **Enforcement, not abstraction, is C1's gap.** The right seam exists and was deliberately built; `cast()`
  turns a would-be compile-time contract into a runtime one. The cheapest high-value refactors here are about
  *re-enabling* the type checker the repo already runs.
- **100%-line ≠ branch/localization safety.** The strict line gate guarantees execution but not that a
  regression is *localizable* or that a *specific field's* schema is asserted — the two blind spots C1 and C3
  target respectively.
- **Deliberate constraints dominate.** Two of three candidates (C1, C2) are load-bearing decisions with clear
  provenance; only C3's un-split monolith is an unrecorded default. Refactor effort should respect the seams
  (make C1's enforced, add C3's test seam) rather than undo the vendoring the fork was forced into.

## Historical Context (from prior changes)

- `context/changes/routing-analysis/research.md` — the report this change builds on: source of the six
  technical-debt items, blast-radius co-change data, and the e2e-only concern. Confirmed and sharpened here;
  corrections: the routing↔openapi seam has a *real import edge* + a *second conformer*; "UNCOVERED" branches
  are line-executed under the 100% gate (localization gap, not execution gap).
- `context/map/repo-map.md` (+ `artifact-1-territory.md`, `-2-structure.md`, `-3-contributors.md`) — priors:
  routing.py Risk Zone (6385 LOC, 83% one owner, private-Starlette coupling, e2e-only); the load-bearing
  hand-defused cycles flagged "do not tidy"; `_compat.py → _compat/` as the in-repo precedent that the project
  splits monoliths when it decides to.

## Related Research

- `context/changes/routing-analysis/research.md` — the immediate prior; this document is its follow-on
  (which debt to fix, target shape, order).

## Open Questions

1. Is there any single test asserting the OpenAPI schema for a route exercising **all 41** `_APIRouteLike`
   fields at once? None found — **unknown**; C1's conformance assertion closes the hole regardless.
2. Which exact upstream Starlette revision do the vendored copies track? Still **unknown** from this repo;
   C2's proposed drift test would pin it.
3. Exact runtime branch-hit distribution across the four dispatch branches — not measured (no branch-coverage;
   `branch=true` is off). Confirmable only by enabling branch coverage / running the suite.
