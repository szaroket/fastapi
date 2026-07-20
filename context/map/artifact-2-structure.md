# Artifact 2 — Structural Map of `fastapi/` (pydeps analysis)

> **Tool:** `pydeps 3.0.6` (import-graph extraction via `--show-deps` JSON dump).
> **Target:** the `fastapi/` package (48 internal modules, 152 internal import edges).
> **Change data:** `git log` file-touch counts over the last ~3 years (since ~2023-07).
> **Note on rendering:** Graphviz (`dot`) is **not installed** on this machine, so SVG/PNG
> graphs were not generated. All findings below come from the machine-readable dependency
> dump, which needs no Graphviz. To also get visual graphs, install Graphviz and re-run the
> commands in the last section.

This map opens with three whole-package questions (§1–§3):
1. **What are the load-bearing / change-sensitive modules?** (hub + churn analysis)
2. **Where is the structural technical debt?** (circular dependencies)
3. **How clean is the layering?** (strongly-connected components + cross-package edges)

It then narrows to the change-hotspots from `artifact-1-territory.md` to answer three more (§6–§9):
4. **Which import cycles actually bite at runtime in the active areas?** (focused cycle analysis)
5. **How testable is each hot module — unit, integration, or e2e?** (testability risks)
6. **Who depends on each hot module, and what crosses each layer boundary?** (reverse deps + layer contracts)

---

## TL;DR

- **The true #1 hotspot is `routing.py`** (fan-in 4, churn 54, 6385 LOC) — the request-handling engine and the most change-sensitive real logic in the package. `__init__.py` scores higher on `fan_in × churn` but is a 25-LOC re-export barrel — API-surface volatility, not a bug hotspot.
- **The "silent" hotspot is `dependencies/utils.py`** — modest fan-in (2) hides the highest churn in the codebase (74) over 1061 lines. The DI resolver changes constantly and is dense; prime candidate for decomposition + characterization tests.
- **The Pydantic-compat seam is the fragile heart:** `_compat` (fan-in 10–11, everyone routes v1/v2 compat through it) ↔ `_compat/v2.py` is **the only genuine top-level runtime cycle**, and it survives only by import line-ordering. `v2.py` is the buggiest file (48% fix rate). Cheap fix: import `lenient_issubclass`/`shared` from `._compat.shared` directly, dropping the parent back-edge.
- **Structural debt = 30 import cycles / 10 mutual pairs**, but most route through re-export barrels (benign). The genuine logic cycles to break: `_compat ↔ _compat/v2`, `routing ↔ utils`, and the param-ring `_compat/v2 → params → datastructures → _compat/v2`. Several are already hand-defused via `TYPE_CHECKING` guards + deferred function-local imports — treat those as load-bearing invariants a refactor must not "clean up."
- **Half the package is one tangle:** 24 of 48 modules form a single strongly-connected component (app + routing + DI + params + security + openapi + compat) — no layered DAG, no "safe bottom layer" in the core. The other 24 modules are clean singletons.
- **Leaky boundaries:** `security → openapi` (5 edges) is a pure *schema* contract — all 8 crossing names are Pydantic scheme models living in `openapi/models.py`; move them to a neutral leaf and the sideways edge vanishes with no logic change. The largest layer contract is `dependencies → _compat` (20 names) — the concrete reason a `_compat/v2.py` change surfaces as a `dependencies/utils.py` bug. `middleware/*` (1 inbound edge) is the model clean boundary.
- **Testability:** the risk is inbound ASGI, not an outbound client — no active module imports a network/DB/SDK to stub. `dependencies/utils.py`, `routing.py`, `applications.py` naturally become **e2e** (drive with `TestClient`); `_compat/v2.py` is a **contract test** vs. real Pydantic (version matrix + `@lru_cache` reset); `encoders.py` and `params.py` construction are the clean **fast-unit** islands.
- **Safest first move:** `openapi/utils.py` is coupled by co-change, not by import cycle — the loosest member of the fragile-core knot and the best candidate to touch/extract first.
- **Rendering caveat:** most findings come from the machine-readable pydeps JSON dump + static AST analysis (Graphviz was initially absent, and `starlette`/`pydantic` aren't installed here). Only one image is generated: the 10-node "fragile core" subgraph at `context/map/fragile-core-cycles.svg`.

### Contents
1. [Hub & change-sensitivity analysis](#1-hub--change-sensitivity-analysis)
2. [Circular dependencies (technical-debt signal)](#2-circular-dependencies-technical-debt-signal)
3. [Layering & module tangle (strongly-connected components)](#3-layering--module-tangle-strongly-connected-components)
4. [What reports pydeps can generate](#what-reports-pydeps-can-generate)
5. [Summary — where the risk lives](#summary--where-the-risk-lives)
6. [Focused: dependency cycles in the active areas](#focused-dependency-cycles-in-the-active-areas-cross-referenced-with-artifact-1-territorymd)
7. [Focused: testability risks in the active areas](#focused-testability-risks-in-the-active-areas-cross-referenced-with-artifact-1-territorymd)
8. [Rendered subgraph — the "fragile core"](#rendered-subgraph--the-fragile-core-contextmapfragile-core-cyclessvg)
9. [Follow-up: reverse dependencies & layer-contract surface](#follow-up-reverse-dependencies--layer-contract-surface-cross-referenced-with-artifact-1-territorymd)

---

## 1. Hub & change-sensitivity analysis

"Importance" here = **fan-in** (how many internal modules depend on it) — a change to a
high-fan-in module ripples widest. "Change-sensitivity" = fan-in **crossed with git churn**
(how often the file actually changes). A `risk` score of `fan_in × churn` surfaces modules
that are both central *and* volatile — the places where regressions concentrate.

### Top risk (fan_in × churn)

| Rank | Module | fan_in | fan_out | churn (3y) | LOC | risk |
|-----:|--------|-------:|--------:|-----------:|----:|-----:|
| 1 | `__init__.py` (`fastapi`) | 21 | 9 | 126 | 25 | **2646** |
| 2 | `routing.py` | 4 | 12 | 54 | 6385 | **216** |
| 3 | `dependencies/utils.py` | 2 | 13 | 74 | 1061 | **148** |
| 4 | `exceptions.py` | 12 | 0 | 12 | 256 | **144** |
| 5 | `params.py` | 6 | 6 | 19 | 754 | **114** |
| 6 | `_compat.py` | 11 | 2 | 10 | 40 | **110** |
| 7 | `utils.py` | 5 | 6 | 19 | 136 | **95** |
| 8 | `openapi/models.py` | 8 | 3 | 11 | 435 | **88** |
| 9 | `_compat/v2.py` | 3 | 7 | 26 | 493 | **78** |
| 10 | `datastructures.py` | 7 | 2 | 11 | 186 | **77** |
| 11 | `encoders.py` | 4 | 4 | 19 | 364 | **76** |
| 12 | `types.py` | 8 | 0 | 5 | 12 | **40** |

### How to read this

- **`fastapi/__init__.py` dominates but is partly noise.** 21 dependents and 126 changes,
  yet only 25 LOC — it is a re-export barrel. Most of its churn is adding exports and version
  bumps, not logic. Treat its risk score as "API-surface volatility," not "bug hotspot."
- **`routing.py` is the true #1 hotspot.** High fan-in *and* the second-highest churn (54),
  on top of a **6385-line** file. This is the heart of request handling and the single most
  change-sensitive piece of real logic in the package. Any refactor here is high-blast-radius;
  it deserves the strongest test coverage.
- **`dependencies/utils.py` is the "silent" hotspot.** Modest fan-in (2) hides the highest
  churn in the codebase (74) over 1061 lines — the dependency-injection resolver. It changes
  constantly and is dense; a prime candidate for decomposition and characterization tests.
- **`_compat.py` / `_compat/v2.py` are load-bearing shims.** `_compat` has fan-in 11 (everyone
  routes Pydantic v1/v2 compatibility through it) and `_compat/v2.py` carries churn 26. This is
  the Pydantic-abstraction seam — fragile by nature and central to everything.
- **Stable pure hubs:** `exceptions.py`, `types.py`, `datastructures.py`, `openapi/models.py`
  have high fan-in but **zero fan-out** (or near it) and low churn — good, well-factored leaves
  that many modules lean on safely.

**Where to focus effort:** `routing.py`, `dependencies/utils.py`, `params.py`,
and the `_compat` pair — high centrality, high volatility, large size.

---

## 2. Circular dependencies (technical-debt signal)

pydeps' import graph contains **30 distinct import cycles** among internal modules and
**10 mutual (A↔B) import pairs**. Cycles are the clearest structural-debt marker: they resist
isolated testing, make refactoring risky, and prevent clean module extraction.

### Mutual import pairs (A imports B *and* B imports A)

| Pair | Nature |
|------|--------|
| `fastapi` ↔ `applications` | via `__init__` re-export barrel |
| `fastapi` ↔ `routing` | via `__init__` re-export barrel |
| `fastapi` ↔ `param_functions` | via `__init__` re-export barrel |
| `fastapi` ↔ `responses` | via `__init__` re-export barrel |
| `_compat` ↔ `_compat/v2` | **genuine logic coupling** |
| `routing` ↔ `utils` | **genuine logic coupling** |
| `security` ↔ `security/api_key` | package `__init__` ↔ submodule |
| `security` ↔ `security/http` | package `__init__` ↔ submodule |
| `security` ↔ `security/oauth2` | package `__init__` ↔ submodule |
| `security` ↔ `security/open_id_connect_url` | package `__init__` ↔ submodule |

### Interpretation — two classes of cycle

- **Re-export cycles (lower concern):** Many cycles route through `fastapi/__init__.py` or a
  package `__init__.py` (e.g. `security` ↔ its submodules). These are an artifact of the
  barrel-export style — the `__init__` re-exports names from submodules that in turn
  `from fastapi import ...`. They are conventional in FastAPI and mostly benign, but they *do*
  mean you cannot import a submodule without triggering the whole package.
- **Genuine logic cycles (real debt):**
  - **`_compat` ↔ `_compat/v2`** — the Pydantic-compat layer imports itself circularly.
  - **`routing` ↔ `utils`** — two large, high-churn modules mutually coupled.
  - **`datastructures → _compat → _compat/v2 → params → datastructures`** — a 4-hop cycle
    binding data structures, the compat shim, and parameter definitions into one knot.
  These are the cycles worth breaking (e.g. by extracting shared types into a leaf module).

### Representative deep cycle

```
fastapi → applications → exception_handlers → utils → routing
        → dependencies.models → security → security.oauth2 → param_functions → fastapi
```

A 9-hop cycle spanning the app, error handling, routing, DI, and security — evidence that these
concerns are not cleanly separable in the current structure.

---

## 3. Layering & module tangle (strongly-connected components)

> pydeps' own "bacon" depth metric is **degenerate** for a whole-package scan: pydeps injects a
> synthetic `__main__` that imports every top-level module, so all 48 modules report `bacon = 1`.
> Layering was therefore derived from **strongly-connected components (SCC)** of the real
> internal import graph instead — a far more honest picture.

### The headline finding

**24 of 48 modules (50%) form a single strongly-connected component** — one giant tangle in
which every module is reachable from every other:

```
_compat, _compat.shared, _compat.v2, applications, datastructures,
dependencies.models, dependencies.utils, encoders, exception_handlers,
fastapi (__init__), openapi.docs, openapi.models, openapi.utils,
param_functions, params, responses, routing, security,
security.api_key, security.base, security.http, security.oauth2,
security.open_id_connect_url, utils
```

The remaining 24 modules are clean singletons (middleware, `staticfiles`, `templating`,
`testclient`, `requests`, `websockets`, `background`, `concurrency`, `cli`, `logger`, etc.) —
they depend inward but nothing internal depends back on them.

**What this means:** the core of FastAPI (app + routing + DI + params + security + openapi +
compat) cannot be reduced to a layered DAG — it is one mutually-recursive cluster. Much of the
binding is the `__init__` re-export style, but the practical consequences are real: you cannot
load, test, or reason about one core module in isolation from the others, and there is no
"safe bottom layer" within the core to build on.

### Cross-subpackage coupling (leaky boundaries)

Directed import edges between subpackages (`a → b : count`):

| From | To | Edges |
|------|-----|------:|
| `openapi` | core | 18 |
| `security` | core | 17 |
| `dependencies` | core | 15 |
| `_compat` | core | 7 |
| `security` | `openapi` | 5 |
| core | `_compat` | 4 |
| core | `openapi` | 4 |
| core | `security` | 4 |
| `dependencies` | `security` | 2 |
| `openapi` | `dependencies` | 2 |
| core | `dependencies` | 2 |
| `_compat` | `openapi` | 1 |
| core | `middleware` | 1 |

- Subpackages depend heavily **inward on core** (expected and healthy).
- But core also reaches **back out** into `_compat`, `openapi`, `security`, and `dependencies`
  (4+4+4+2 edges) — this bidirectional core↔subpackage traffic is what sustains the big SCC.
- **`security → openapi` (5 edges)** is a notable sideways dependency: the security modules are
  coupled to OpenAPI schema generation, not just to core.

**Cleanest boundary:** `middleware/*` — one inbound edge from core, nothing else. A good model
for how the other subpackages could ideally sit.

---

## What reports pydeps can generate

For reference, the report types available from this tool (only the JSON-based ones were used
here since Graphviz is absent):

| Report | Command | Needs Graphviz? | Used here |
|--------|---------|:---------------:|:---------:|
| **JSON dependency dump** (fan-in/out, per-module) | `pydeps fastapi --show-deps --no-output --max-bacon 0` | No | ✅ (all 3 analyses) |
| **Cycle report** | `pydeps fastapi --show-cycles -o cycles.svg` | Yes (render) | ⛔ (cycles derived from JSON instead) |
| **Raw import list** | `pydeps fastapi --show-raw-deps` | No | — |
| **SVG / PNG dependency graph** | `pydeps fastapi --cluster -o graph.svg` | Yes | ⛔ |
| **Graphviz source** | `pydeps fastapi -T dot -o graph.dot --no-show` | No (render later) | — |
| **Reverse graph** ("who depends on X") | `pydeps fastapi --reverse -o rev.svg` | Yes | — |

Useful filters: `--max-bacon N` (depth cap), `--cluster` (group externals),
`--only fastapi.security`, `--exclude`, `--rmprefix fastapi.`, `--noshow`.

### To generate the visual graphs on this machine

```bash
# 1. install Graphviz and put dot on PATH (Windows):
#    winget install graphviz     (then restart shell)
# 2. cycles-only graph:
python -m pydeps fastapi --show-cycles --cluster -o context/map/cycles.svg --noshow
# 3. clustered overview:
python -m pydeps fastapi --cluster --max-bacon 2 --rankdir LR -o context/map/overview.svg --noshow
```

---

## Summary — where the risk lives

1. **Change-sensitive core:** `routing.py`, `dependencies/utils.py`, `params.py`, and the
   `_compat` shim pair — central, volatile, and large. Prioritize test coverage and treat any
   refactor as high-blast-radius.
2. **Real (non-barrel) circular debt:** break `_compat ↔ _compat/v2`, `routing ↔ utils`, and
   the `datastructures → _compat/v2 → params → datastructures` knot by extracting shared types
   into leaf modules.
3. **Structural coupling:** half the package is one strongly-connected tangle with no internal
   "safe bottom layer." The `middleware/*` subpackage is the model of a clean boundary; the
   `security → openapi` sideways dependency is the clearest candidate to decouple.

---

# Focused: dependency cycles in the *active* areas (cross-referenced with `artifact-1-territory.md`)

> **Scope:** only the cycles that pass through the change-hotspots that `artifact-1-territory.md`
> identified — `dependencies/utils.py`, `routing.py`, `_compat/v2.py`, `openapi/utils.py`,
> `applications.py`, `encoders.py`, `utils.py`, `params.py`, `dependencies/models.py`,
> `datastructures.py`, and the auth files `security/oauth2.py` + `security/api_key.py`. This is
> **not** a full repo cycle dump.
>
> **Tool note:** the requested "dependency-cruiser" is a JavaScript/TypeScript tool and does not
> parse Python, so the equivalent evidence here comes from **pydeps** (import graph) confirmed
> against the **actual `import` statements** in the source. The evidence column cites both.
>
> **Method:** enumerated every elementary import cycle, then kept those touching an active file.
> pydeps reports **~2,924** cycles through the active set — but **2,915 of them are the same
> handful of edges threaded through the `fastapi/__init__.py` re-export barrel** (combinatorial
> noise). Stripping the barrel leaves **9 genuine cycles**. Those 9 are analysed below.

## Most important observations (read first)

1. **Only 9 genuine cycles matter, and they land exactly on the three hotspots artifact-1 named.**
   Every genuine cycle sits in one of: the **Pydantic-v2 compat seam**, the **param/validation
   core**, or the **DI + routing spine** — the same "fragile heart" and "architectural seam"
   from artifact-1 §1/§4/§6. The dependency graph and the git history agree on where the risk is.

2. **The one genuinely fragile *runtime* cycle is `_compat/__init__.py ↔ _compat/v2.py`, and it
   survives only by line ordering.** Both directions are top-level imports: `v2.py:18` does
   `from fastapi._compat import lenient_issubclass, shared` while `_compat/__init__.py:22+` does
   `from .v2 import …`. It works **only because** `__init__.py` pulls `lenient_issubclass`/`shared`
   from `.shared` on **lines 1–21**, *before* triggering `v2` on line 22 — so the names already
   exist in the half-initialised package when `v2` reaches back for them. **Import order is
   load-bearing**: reorder `__init__.py` and startup breaks with a partial-init `ImportError`.
   This is precisely `_compat/v2.py`, artifact-1's **buggiest file (48% fix rate)** and the
   "Pydantic-v2 blast-radius hub." The clean fix is cheap (see table): both names already live in
   `shared.py`, so `v2.py` should import them from `._compat.shared` directly, not via the parent.

3. **The scariest-looking cycles are already hand-defused — which is itself the warning sign.**
   `routing ↔ utils` and `dependencies.utils → utils → routing → dependencies.utils` do **not**
   cycle at runtime: `utils.py:22` imports `routing` only under `if TYPE_CHECKING`. Likewise
   `_compat/v2 → params` (`v2.py:350`) and `datastructures → _compat/v2` (`datastructures.py:148`)
   are **deferred function-local imports**. The maintainers already worked around these knots by
   hand. Good for import safety — but those `TYPE_CHECKING` guards and lazy imports are exactly
   the friction a refactor will trip over.

4. **The param/validation knot is a 4-file ring:** `_compat/v2 → params → datastructures → _compat/v2`
   (plus `params → openapi.models → _compat`). Three of the four are top-churn files, and this is
   the same cluster behind artifact-1's winter "param-annotation wave." A change to one field/
   validation type structurally cannot stay local.

5. **The auth cycles are an unavoidable package-init artifact — benign, but they widen blast radius.**
   `security/__init__.py ↔ oauth2/api_key` is *not* caused by the submodules importing the package
   by name (they already import the sibling `security.base` directly). It exists because importing
   **any** `fastapi.security.*` submodule forces Python to run `security/__init__.py` first, and that
   `__init__` eagerly imports every scheme. So you cannot import one auth scheme without loading all
   of them — and the cycle can't be fixed by changing the submodule's import style. Relevant because
   artifact-1 flags `oauth2.py`/`api_key.py` as **low-churn / high-bug (50% / 44%)**: rare changes
   there are risky, and the coupling makes them ripple.

## Active-area cycle table

| Area | What you found | Evidence (pydeps import graph + source `import`s) | Why this matters for the change | Relation to `artifact-1-territory.md` | What to check next |
|------|----------------|---------------------------------------------------|--------------------------------|---------------------------------------|--------------------|
| **Pydantic-v2 compat seam** (`_compat` ↔ `_compat/v2`) | A **genuine top-level mutual import** — the only true runtime cycle in the hot set — that works only by import ordering. | pydeps edge `_compat ↔ _compat.v2`. `fastapi/_compat/v2.py:18` `from fastapi._compat import lenient_issubclass, shared`; `fastapi/_compat/__init__.py:1–21` re-exports those from `.shared`, **then** `:22+` `from .v2 import …`. v2 succeeds because lines 1–21 run before line 22. | Import order is load-bearing: the package `__init__` and its `v2` submodule initialise each other, and only the current line order keeps `lenient_issubclass`/`shared` defined before `v2` reads them. Reordering or splitting `_compat` risks partial-init `ImportError`s at startup, and any Pydantic-version change touches both ends at once. | `_compat/v2.py` = **buggiest file (48% fix), fan-in 11**, the "coupling hub / Pydantic-v2 blast radius" (§4, §6). `_compat` was already a moving target (monolith→package, §8). | **Cheap fix, worth doing:** change `v2.py:18` to `from fastapi._compat.shared import lenient_issubclass` + `from fastapi._compat import shared` → `import fastapi._compat.shared as shared` — both names already live in `shared.py`, so importing them directly removes the parent-package back-edge and breaks this cycle entirely. |
| **Param / validation core** (`_compat/v2 → params → datastructures → _compat/v2`; `params → openapi.models → _compat`) | A 4-file validation ring; the `_compat/v2→params` and `datastructures→_compat/v2` hops are **deferred (function-local) imports** deliberately breaking the load-time cycle. | pydeps cycles `_compat.v2 → params → datastructures → _compat.v2` and `… → openapi.models → _compat`. `params.py:8` `from fastapi.openapi.models import Example`, `:16` `from .datastructures import _Unset`, `:13` `from ._compat import …` (all top-level); `_compat/v2.py:350` `from fastapi import params` (**inside `is_scalar_field()`**); `datastructures.py:148` `from ._compat.v2 import …` (**inside a method**). | Field/validation types are defined across four mutually-dependent files. The lazy imports mean the cycle "works" but is invisible to static tooling and easy to re-break: pulling one of those deferred imports up to module level would reintroduce a hard cycle. | This is the winter **param-annotation wave** cluster (§2), and `params.py` (fan-in 6, 754 LOC) + `_compat/v2.py` are both fragile-heart files (§4). | Grep for other `from fastapi import params` / `from ._compat.v2 import` deferred inside functions — inventory the lazy-import workarounds before moving any type. Add a regression test that imports each module standalone. |
| **DI + routing spine** (`routing ↔ utils`; `dependencies.utils → utils → routing → dependencies.utils`) | Cycles exist in the graph but are **type-only / broken at runtime** by a `TYPE_CHECKING` guard. | pydeps edges `routing ↔ utils` and `dependencies.utils → utils → routing`. `routing.py:77` `from fastapi.utils import (…)` and `:51` `from fastapi.dependencies.utils import (…)` (top-level, runtime); `utils.py:22–23` `if TYPE_CHECKING:` → `from .routing import APIRoute`; `dependencies/utils.py:62` `from fastapi.utils import create_model_field, get_path_param_names`. | The request-lifecycle trio only holds together because the `utils→routing` back-edge is annotations-only. Any refactor that needs `APIRoute` at runtime in `utils.py`, or that moves a helper between these files, converts a benign type cycle into a hard import cycle. | `dependencies/utils.py` (46 changes) + `routing.py` (36) are artifact-1's **gravitational center**; both are the steadiest files (8/13 months, §10). `routing.py` also carries an **83% single-owner bus factor** (§11). | Verify `utils.py`'s only `routing` use stays under `TYPE_CHECKING`. Before splitting `routing.py` (6385 LOC), map which of the `dependencies.utils`/`utils` helpers it pulls at import time. |
| **OpenAPI schema generation** (`openapi/utils.py`, `openapi/models.py`) | `openapi/utils.py` — despite being a top-4 hotspot and part of artifact-1's "4-file knot" — is in **no genuine import cycle**; only `openapi/models.py` appears in one (`params → openapi.models → _compat`). | pydeps: `openapi.utils` appears only in barrel cycles through `fastapi/__init__.py`; `openapi.models` in the genuine `_compat/v2 → params → openapi.models → _compat` ring. `params.py:8` imports `openapi.models.Example`. | The `openapi/utils.py`↔`routing.py` coupling artifact-1 saw is **co-change coupling, not an import cycle** — safer to move than the compat/param knots. But `openapi/models.py` is pinned into the validation ring via `params`. | Reconciles §6: the "4-file knot" (`_compat/v2`+`dependencies/utils`+`openapi/utils`+`routing`) is bound by *co-commits*, not imports — `openapi/utils.py` is the loosest member structurally. | Confirm `openapi/utils.py` has no function-local imports back into `routing`/`params`. If clean, it is the safest of the four to refactor or extract first. |
| **Auth / security** (`security/__init__ ↔ oauth2`, `↔ api_key`) | **Package-init barrel** cycle — *not* an explicit back-import. The submodules already import the sibling `security.base` directly; the back-edge is the implicit parent-package init that pydeps records. (The `from fastapi.security import …` lines in these files are **docstring examples**, which pydeps does not parse.) | pydeps: `security.oauth2.imports` includes `fastapi.security` though `oauth2.py` has **no** module-level `from fastapi.security import`. Real imports are `oauth2.py:8` `from fastapi.security.base import SecurityBase` and `api_key.py:5` likewise — importing a submodule executes `security/__init__.py:1–10` (`from .api_key/.oauth2 import …`), closing the loop. | Structurally benign (Python resolves it via partial-init ordering), but importing any single auth scheme loads the whole `security` package — you can't test/import one scheme in isolation. On the rare, risky auth change this widens blast radius. | `security/oauth2.py` (50% fix) and `dependencies/models.py`/`api_key.py` are artifact-1's **low-churn / high-bug** files (§4) and auth is a "persistent trickle" (§10) — exactly where you least want extra coupling. | Nothing to "fix" in the submodules — the sibling imports are already correct. If isolation is ever needed, the only lever is making `security/__init__.py` re-exports lazy (PEP 562 `__getattr__`), which trades away eager `from fastapi.security import X`. Usually not worth it. |

## Bottom line for the change

- **Break-first target:** `_compat/__init__.py ↔ _compat/v2.py` — the only genuine *runtime* mutual
  import, sitting on the buggiest file. Everything else in the compat/param knot already relies on
  hand-placed lazy imports.
- **Do-no-harm rule:** the `TYPE_CHECKING` guard in `utils.py:22` and the deferred imports in
  `_compat/v2.py:350` / `datastructures.py:148` are load-bearing. Treat them as invariants; a
  refactor that "cleans up" these into normal top-level imports will reintroduce hard cycles.
- **Safest first move:** `openapi/utils.py` is coupled by co-change, not by import cycle — the
  loosest member of artifact-1's 4-file knot and the best candidate to touch first.

---

# Focused: testability risks in the *active* areas (cross-referenced with `artifact-1-territory.md`)

> **Scope:** the same active hot-set as above (`dependencies/utils.py`, `routing.py`, `_compat/v2.py`,
> `openapi/utils.py`, `applications.py`, `encoders.py`, `utils.py`, `params.py`,
> `dependencies/models.py`, `security/oauth2.py`, `security/api_key.py`, `datastructures.py`).
>
> **Method:** pydeps supplied (a) the **internal fan-out** per module (`deps.json`) and (b) the
> package's **external dependency surface** (`pydeps fastapi --externals`). Because pydeps'
> default trace stops at the package boundary, the per-module *external* attribution below was
> read straight from each file's `import` block and is what drives mocking difficulty.
>
> **`pydeps --externals` for `fastapi`:** `anyio`, `asyncio`, `pydantic`, `pydantic_core`,
> `pydantic_extra_types`, `starlette`, `python_multipart` / `multipart`, `email_validator`,
> `annotated_doc`, `typing_extensions`, `typing_inspection`, `re`, `collections`, `pathlib`.
> **Notably absent: any outbound HTTP/API-client library.**

## Most important observations (read first)

1. **The risk is inbound ASGI, not an outbound API client.** No active module imports a network
   client, DB driver, or SDK to stub. The "hard to isolate" dependencies are all **inbound
   platform types** — Starlette `Request`/`WebSocket`/`Response`, Pydantic models, and
   `inspect`-based signature introspection. So the dominant testability cost is **fabricating a
   realistic ASGI Request / real Pydantic model**, which pushes the hot set toward
   **`TestClient` integration/e2e**, not classic mock-the-collaborator unit tests.

2. **The two gravitational-center files are the least unit-testable in the repo.**
   `dependencies/utils.py` and `routing.py` import ASGI types (`Request`, `WebSocket`, `Response`,
   `BackgroundTasks`), `inspect` signature reflection, `contextlib.AsyncExitStack`, `anyio`, and —
   in `routing.py` — **private Starlette internals** (`starlette._exception_handler`,
   `starlette._utils`). These cannot be meaningfully mocked; a change here **naturally lands as an
   e2e test** (send a request → assert response / resolved dependency). This is artifact-1's
   "request-lifecycle spine."

3. **`_compat/v2.py` is a contract test against a real dependency, not a mockable unit.** It reaches
   into **private Pydantic internals** (`pydantic._internal._typing_extra`,
   `pydantic._internal._schema_generation_shared`, `pydantic_core.core_schema`). Mocking those
   asserts nothing and they drift across Pydantic patch releases — precisely why it is artifact-1's
   **buggiest file (48% fix)**. It must run against the **real installed Pydantic** (integration/
   contract), and its module-level `@lru_cache` (`get_cached_model_fields`) is **global state that
   leaks across tests**.

4. **The genuinely unit-testable islands are `encoders.py`, `params.py`, and parts of `utils.py`.**
   Pure functions / declaration dataclasses, no ASGI, no I/O. `encoders.py` (`jsonable_encoder`)
   is the model case — object in, JSON-able out — ideal for **parametrized golden/snapshot unit
   tests** with zero mocking. Spend fast-unit budget here and reserve integration budget for the spine.

5. **Auth sits exactly on the mock↔integration boundary.** `security/oauth2.py` / `api_key.py` are
   small callables that read a *few* `Request` attributes (`.headers` / `.query_params` /
   `.cookies`), so a Request mock is *feasible* (narrow surface) — but it under-tests real
   header/scheme parsing, and artifact-1 flags these as **50% / 44% bug-density**. Default to
   **`TestClient` integration** with a protected route.

**Testability triage legend:** 🌐 change naturally becomes **e2e** · 🔗 **integration** preferred ·
🧪 **heavy mocking** if unit-tested · ✅ cleanly **unit-testable**.

## Active-area testability table

| Area | What you found | Evidence from pydeps | Why this matters for the change | Relation to `artifact-1-territory.md` | What to check next |
|------|----------------|----------------------|--------------------------------|---------------------------------------|--------------------|
| **DI resolver** `dependencies/utils.py` 🌐🧪 | The hardest file to isolate: resolves dependencies by **`inspect`-ing callable signatures at runtime** and driving an **`AsyncExitStack`**, operating on live `Request`/`WebSocket`/`Response`. Unit-testing means faking an ASGI connection + realistic signatures — huge mock surface. | pydeps fan-out **13** (all internal); `--externals` members it pulls: `starlette.requests.Request/HTTPConnection`, `starlette.websockets.WebSocket`, `starlette.responses.Response`, `starlette.background.BackgroundTasks`, `starlette.concurrency.run_in_threadpool`, `pydantic`, plus stdlib `inspect`, `contextlib.AsyncExitStack`. | This is the #1 e2e-magnet. Any behavioral change to DI resolution should be proven end-to-end; a "unit" test here mostly tests your mocks, not the resolver. | The **gravitational center** (46 changes, 41% fix, 18 authors) — steadiest file, 8/13 months (§1, §4, §10). | Prefer a `TestClient` app exercising the changed dependency shape. If you must unit-test, isolate the *pure* helpers (`get_path_param_names`, param analysis) from the `Request`-touching resolution. |
| **Router / request lifecycle** `routing.py` 🌐 | The ASGI request/response engine. Imports **private Starlette internals** (`_exception_handler`, `_utils`) and `anyio` — brittle to mock and unstable to couple to. Behavior only manifests as a real HTTP round-trip. | pydeps fan-out **12**; `--externals`: `anyio`, `starlette.routing`, `starlette._exception_handler`, `starlette._utils`, `starlette.requests.Request`, `starlette.responses`, `starlette.datastructures`; stdlib `inspect`, `json`, `email.message`, `os`, `functools`, `contextlib`. | Changes here **naturally become e2e** — send a request, assert status/body/headers. Mocking the lifecycle is infeasible and mocking `starlette._*` privately couples tests to unstable APIs. | Request-handling spine (36 changes); **83% single-owner bus factor** (§11) — tests are the main safety net, so make them behavioral. | Add/extend `TestClient` route tests covering the changed path (streaming, background tasks, response class). Avoid asserting against `starlette._*` internals. |
| **Pydantic-v2 compat** `_compat/v2.py` 🔗🧪 | A shim over **private Pydantic APIs** — not mockable in a way that proves anything, and version-fragile. Also holds **module-level `@lru_cache`** → cross-test cache bleed. | pydeps fan-out **7**; `--externals`: `pydantic._internal._typing_extra`, `pydantic._internal._schema_generation_shared`, `pydantic.json_schema`, `pydantic_core.core_schema`, `pydantic.fields.FieldInfo`, `TypeAdapter`, `create_model`; stdlib `functools.lru_cache`. | Test against the **real installed Pydantic** (contract/integration); pin the version in CI. Reset/avoid the `lru_cache` between cases or tests interfere. Explains the 48% fix rate: breakage tracks Pydantic releases, not FastAPI logic. | Buggiest file (48% fix, §4); fan-in 11 "Pydantic-v2 blast radius" (§6); moving target (§8). | Run the compat tests across the supported Pydantic version matrix; add a cache-clear fixture for `get_cached_model_fields`; assert on *schema output*, not on internal Pydantic objects. |
| **App composition root** `applications.py` 🌐 | The `FastAPI` class instantiates **the whole Starlette app + middleware stack** and exposes `state` (shared mutable). It *is* the e2e entry point; there is no smaller unit. | pydeps fan-out **14**; `--externals`: `starlette.applications.Starlette`, six `starlette.middleware.*` classes, `starlette.datastructures.State`, `starlette.types.{ASGIApp,Receive,Scope,Send}`. | Changing app wiring/middleware order is only observable by **running the app**. `state` is global mutable app-scope → tests that touch it must isolate it. | 15 changes, top-level module (§3). | Cover via `TestClient(app)` asserting middleware/exception-handler behavior; ensure per-test app instances so `app.state` doesn't bleed. |
| **OpenAPI schema gen** `openapi/utils.py` 🔗✅ | Needs a *real assembled app/routes* as input, but its **output is a deterministic dict** → naturally a **golden/snapshot** test. Highest fan-out but no async, no request. | pydeps fan-out **17** (highest in the set); `--externals`: `starlette.routing.BaseRoute`, `starlette.responses.JSONResponse`, `pydantic.BaseModel`, stdlib `http.client`, `inspect`. | Cheaper to test than the spine: build a small app, snapshot `app.openapi()`. Diffs are readable and stable. | Top-4 hotspot (20 changes) but **feature-churn, low bug rate** (25%, §4); the *loosest* member of the 4-file knot (import-cycle-free). | Add/refresh an `openapi.json` snapshot for the routes your change affects; assert schema diff rather than mocking route objects. |
| **Encoders** `encoders.py` ✅ | A **pure function** (`jsonable_encoder`): object → JSON-able value. No ASGI, no I/O, no global state. Only breadth risk is many type branches. | pydeps fan-out **4** (lowest-but-two); `--externals`: `pydantic.{BaseModel,SecretStr,AnyUrl,NameEmail}`, `pydantic_core.PydanticUndefinedType`, `pydantic_extra_types.color`, stdlib `dataclasses`, `collections`, `enum`, `re.Pattern`. | The **best fast-unit target** in the hot set — parametrize over input types, zero mocking. Put the change's correctness burden here where feedback is cheapest. | 13 changes, moderate churn (§1). | Extend the parametrized encoder table with the new/changed type; no integration needed. |
| **Param & Dependant models** `params.py` ✅ / `dependencies/models.py` 🧪 | `params.py` = declaration dataclasses (`Query/Path/Body`) — construct-and-assert, unit-testable. `dependencies/models.py` has a **platform-conditional import** (`asyncio` vs `inspect` `iscoroutinefunction` by Python version) and a `cached_property` → branch + cache-state risk. | `params.py` fan-out **6**, externals `pydantic.{AliasChoices,AliasPath,FieldInfo}`. `dependencies/models.py` fan-out **5**, stdlib `inspect`, version-gated `from asyncio import iscoroutinefunction`, `functools.cached_property`. | `params` construction is unit-testable, but its *behavior* only appears through the DI resolver → pair unit tests with one integration test. The version-gated import in `models.py` needs coverage on **both** Python branches. | `params.py` 11 changes, 73% single-owner (§11); `dependencies/models.py` low-churn/high-bug (44%, §4). | Unit-test param construction; run `models.py` tests on the min and max supported Python; watch `cached_property` staleness in reused `Dependant` fixtures. |
| **Auth schemes** `security/oauth2.py`, `security/api_key.py` 🔗🧪 | Small callables reading a few `Request` attributes (`.headers`/`.query_params`/`.cookies`) then raising 401 or returning a credential. Narrow enough to mock a Request, but the real value is header/scheme parsing. | `oauth2.py` fan-out **8**, `api_key.py` fan-out **5**; `--externals`: `starlette.requests.Request`, `starlette.status`, `starlette.exceptions.HTTPException`. (The `from fastapi.security import …` lines are docstrings — not imports.) | Mocking a Request is feasible but under-tests parsing; given the high bug rate, prefer **integration** with a protected route sending real headers/cookies. | `oauth2.py` **50% fix**, `api_key.py` in the low-churn/high-bug band (§4); auth = "persistent trickle" (§10). | Add `TestClient` cases sending valid/missing/malformed `Authorization`/API-key headers; assert 401 vs pass-through rather than mocking `Request.headers`. |
| **Datastructures / UploadFile** `datastructures.py` 🔗 | Thin re-exports of **Starlette platform types** plus `UploadFile` wrapping a real spooled temp file → **file I/O**, best exercised with real uploads. | pydeps fan-out **2** (lowest); `--externals`: `starlette.datastructures.{URL,Headers,FormData,UploadFile,...}`, `pydantic.GetJsonSchemaHandler`. | Little standalone logic, but `UploadFile` behavior (spooling, async read/close) needs a **real multipart upload** to test meaningfully. | fan-in 6, stable-but-central (§5). | If the change touches `UploadFile`, use a `TestClient` multipart POST with a file large enough to trigger spooling; don't fake the file object. |

## Bottom line for testing the change

- **Where extensive mocking is likely (and mostly counter-productive):** `dependencies/utils.py`
  (fake ASGI Request + signatures) and `_compat/v2.py` (private Pydantic internals). In both, the
  mock surface is so large that the test would validate the mocks, not the code — prefer real
  collaborators.
- **Where an integration test is preferable:** `_compat/v2.py` (contract test vs. real Pydantic,
  version matrix), `openapi/utils.py` (build app → snapshot schema), `security/*` (protected route
  + real headers), `datastructures.py` (`UploadFile` via real multipart).
- **Where the change naturally becomes an e2e test:** `dependencies/utils.py`, `routing.py`, and
  `applications.py` — the request-lifecycle spine and composition root. Drive them with
  `TestClient` and assert observable HTTP behavior.
- **Where fast unit tests pay off:** `encoders.py` and `params.py` construction — pure, mock-free,
  cheapest feedback; concentrate correctness checks here.
- **Global-state watch-outs:** `_compat/v2.py` `@lru_cache`, `applications.py` `app.state`,
  `dependencies/models.py` `cached_property` — use fresh fixtures / cache resets so tests don't leak.

---

# Rendered subgraph — the "fragile core" (`context/map/fragile-core-cycles.svg`)

> **Rendered with Graphviz** (`dot` 15.1.0) via pydeps. This is the one place we generate an
> image; the rest of the map stays text.

## The single question this graph answers

> **"What is the minimal set of core modules you cannot change or test in isolation — and how do
> their genuine (non-barrel) import cycles bind them together?"**

Everything above points at the same knot: the Pydantic-v2 compat seam + the param/validation core
+ the DI/routing spine. The SVG is scoped to exactly those **10 modules** so the answer is legible
instead of drowned in the 48-module whole-package graph.

## Why this scope (and how the pydeps flags map to the request)

The requested `--focus` / `--include-only` / `--collapse` are **dependency-cruiser** flags; the
pydeps equivalents used here:

| Intent | dependency-cruiser | pydeps flag used | Effect here |
|--------|--------------------|------------------|-------------|
| include-only a module set | `--include-only` | **`--only fastapi._compat fastapi.params fastapi.datastructures fastapi.openapi.models fastapi.routing fastapi.utils fastapi.dependencies.utils fastapi.encoders`** | keeps just the fragile-core nodes (10 modules) |
| collapse/shorten labels | `--collapse` | **`--rmprefix fastapi.`** | strips the `fastapi.` prefix from every node label |
| focus direction | `--focus` | **`--reverse`** | flips arrows so **A → B means "A imports B"** (intuitive dependency direction) |
| full depth | — | **`--max-bacon 0`** | no depth cut |
| drop the barrel | — | *omit `fastapi` from `--only`* | the `__init__.py` re-export hub is excluded, so **only genuine module→module edges survive** (no combinatorial barrel cycles) |

Exact command (run from repo root, `dot` on `PATH`):

```bash
python -m pydeps fastapi \
  --only fastapi._compat fastapi.params fastapi.datastructures fastapi.openapi.models \
         fastapi.routing fastapi.utils fastapi.dependencies.utils fastapi.encoders \
  --rmprefix fastapi. --reverse --max-bacon 0 --rankdir LR \
  -T svg -o context/map/fragile-core-cycles.svg --noshow
```

Result: **10 nodes, 23 edges.**

## How to read it

- **Arrows mean "imports"** (`params → _compat` = `params` imports `_compat`).
- **`_compat` is the sink hub** — imported by 8 of the other 9 modules. It is the single point
  through which the whole core is coupled; it is also the buggiest area (`_compat/v2.py`, 48% fix).
- **The genuine cycles to spot in the picture:**
  - `_compat ↔ _compat.v2` — the only true top-level runtime cycle (see §"Focused: dependency cycles").
  - `routing ↔ utils` and `dependencies.utils → utils → routing → dependencies.utils` — the DI/
    routing spine (the `utils → routing` back-edge is `TYPE_CHECKING`-only).
  - `_compat.v2 → params → datastructures → _compat.v2` — the param/validation ring (the
    `_compat.v2 → params` and `datastructures → _compat.v2` hops are deferred function-local imports).
- **Takeaway:** any change touching a field type, validation, or the request lifecycle lands
  somewhere in this 10-node cluster and cannot be isolated from `_compat`. This is the subgraph to
  put under characterization/integration tests before refactoring.

---

# Follow-up: reverse dependencies & layer-contract surface (cross-referenced with `artifact-1-territory.md`)

> **Why this section exists.** The step's four questions were mostly answered above, but two
> were only partial: **Q1 "where are the contracts between layers"** (the map showed edge
> *counts*, not the names crossing) and **Q4 "who depends on X / what breaks"** (fan-in totals,
> but no per-module reverse list — pydeps' `--reverse` report was never generated). This section
> closes both.
>
> **Method.** pydeps traces by *importing* modules, but `starlette`/`pydantic` are not installed
> on this machine (see the stale `deps.err`), so a live pydeps run fails. Instead the graph here
> was built **statically from the current source AST** (`ast.ImportFrom`/`ast.Import`, relative
> imports resolved) — internal `fastapi.*` edges only. This is more current than the earlier
> pydeps dump (it sees the `_compat/` *package*, not the retired `_compat.py` monolith), so a few
> fan-in numbers differ slightly from §1.

## A. Reverse dependencies — who imports each hot module (internal only)

Internal-importer counts. Note `routing.py`'s blast radius is external + via the `__init__`
re-export barrel, so its *internal* fan-in understates it — it is a hub by fan-**out** (12), not
by importers.

| Hot module | Internal fan-in | Importers | What breaks on change |
|---|--:|---|---|
| `exceptions.py` | **12** | `__init__`, applications, dependencies.utils, encoders, exception_handlers, openapi.utils, params, responses, routing, security.http, security.oauth2, utils | Widest internal reach — confirms artifact-1 "most depended-on file." |
| `_compat` | **10** | _compat.v2, dependencies.models, dependencies.utils, encoders, openapi.models, openapi.utils, param_functions, params, routing, utils | The Pydantic blast radius (artifact-1 §6): DI, encoding, params, openapi, routing all ripple. |
| `openapi/models.py` | **8** | openapi.utils, param_functions, params, security.api_key, security.base, security.http, security.oauth2, security.open_id_connect_url | Explains the `security → openapi` sideways edge (see §B). |
| `datastructures.py` | **7** | `__init__`, applications, openapi.utils, param_functions, params, routing, utils | `UploadFile`/`Default` reach the param + routing spine. |
| `utils.py` | **5** | applications, dependencies.utils, exception_handlers, openapi.utils, routing | Half the spine; the `routing ↔ utils` cycle partner. |
| `encoders.py` | **4** | exception_handlers, openapi.docs, openapi.utils, routing | Response-serialization path. |
| `routing.py` | **2** | `__init__`, utils | Low *internal* fan-in — consumer/hub; breakage surfaces at the HTTP boundary. |
| `dependencies/utils.py` | **2** | openapi.utils, routing | The "silent hotspot": high churn, tiny fan-in; a change breaks routing + schema-gen. |
| `_compat/v2.py` | **2** | _compat, datastructures | Runtime-cycle partner of `_compat`. |
| `params.py` | **2** | applications, openapi.utils | — |
| `security/oauth2.py` | **2** | dependencies.utils, security | DI resolves security deps → a break here hits dependency resolution. |
| `openapi/utils.py` | **1** | applications | Loosest member — safest to touch first (matches "safest first move"). |
| `security/api_key.py` | **1** | security | Only the package `__init__` imports it. |

**Reads:**
- **Fan-in and churn are inversely correlated at the hot core.** The two biggest hotspots by
  churn — `routing.py` and `dependencies/utils.py` — have internal fan-in of only 2 each, while
  the *stable* leaves (`exceptions` 12, `_compat` 10, `openapi.models` 8, `datastructures` 7)
  carry the fan-in. So "what breaks" splits cleanly: change a **leaf** and many modules recompile
  their assumptions; change the **spine** and the failure shows up at the request boundary (e2e),
  not in importers — reinforcing artifact-2's testability finding.
- **`openapi/models.py` is the hidden linchpin for auth.** All five `security/*` modules depend on
  it. Combined with artifact-1's "auth is low-churn / high-bug (50%/44%)," a schema-model edit is a
  quiet way to break every auth scheme at once.

## B. Layer-contract surface — the names that cross each boundary

What actually flows across each subpackage boundary (the "contract"), largest first:

| Boundary | # names | Contract (representative names) |
|---|--:|---|
| `dependencies → _compat` | **20** | `ModelField`, `create_body_model`, `is_scalar_field`, `get_cached_model_fields`, `serialize_sequence_value`, `field_annotation_is_sequence`, `RequiredParam`, `Undefined`, `lenient_issubclass`, … |
| `openapi → core` | 14 | `Body`, `Response`, `jsonable_encoder`, `ParamTypes`, `ModelNameMap`, `generate_operation_id_for_path`, `is_body_allowed_for_status_code`, `routing`, … |
| `dependencies → core` | 10 | `BackgroundTasks`, `DependencyCacheKey`, `DependencyScopeError`, `create_model_field`, `get_path_param_names`, `params`, … |
| `core → _compat` | 9 | `ModelField`, `PydanticSchemaGenerationError`, `Url`, `annotation_is_pydantic_v1`, `is_pydantic_v1_model_instance`, `with_info_plain_validator_function`, `v2`, … |
| `core → dependencies` | 9 | `Dependant`, `solve_dependencies`, `get_dependant`, `get_body_field`, `get_flat_dependant`, `get_typed_return_annotation`, … |
| `security → openapi` | 8 | **security scheme *models*:** `APIKey`, `APIKeyIn`, `HTTPBase`, `HTTPBearer`, `OAuth2`, `OAuthFlows`, `OpenIdConnect`, `SecurityBase` |
| `openapi → _compat` | 7 | `get_definitions`, `get_flat_models_from_fields`, `get_model_name_map`, `get_schema_from_model_field`, `ModelField`, `lenient_issubclass`, … |
| `core → openapi` | 5 | `get_openapi`, `get_swagger_ui_html`, `get_redoc_html`, `Example`, … |
| `openapi → dependencies` | 5 | `Dependant`, `get_flat_dependant`, `get_flat_params`, `get_validation_alias`, `_get_flat_fields_from_params` |
| `_compat → core` | 4 | `IncEx`, `ModelNameMap`, `UnionType`, `params` |
| `dependencies → security` | 2 | `SecurityBase`, `SecurityScopes` |
| `security → core` | 2 | `Form`, `HTTPException` |
| `core → middleware` | 1 | `AsyncExitStackMiddleware` |

**Reads:**
- **The single largest layer contract is `dependencies → _compat` (20 names)** — the DI engine
  consumes 20 field/validation helpers from the Pydantic-compat shim. This is the concrete
  "contract" behind artifact-1's fragile heart: it is exactly *why* a `_compat/v2.py` change
  (artifact-1's 48%-fix file) surfaces as a `dependencies/utils.py` bug.
- **`security → openapi` is a *schema* contract, not a logic one.** All 8 crossing names are
  Pydantic **scheme models** (`OAuth2`, `HTTPBearer`, `SecurityBase`, …) that happen to live in
  `openapi/models.py`. This pinpoints artifact-2's "clearest candidate to decouple": move those
  security-scheme models to a neutral leaf module and the sideways edge disappears with no logic
  change.
- **The bidirectional pairs are the SCC glue.** `core ↔ _compat` (9 out / 4 in — plus `_compat`'s
  9 into DI) and `core ↔ dependencies` (10 out / 9 in) both cross in *both* directions. These
  reciprocal contracts are the mechanical reason the 24-module core is one strongly-connected
  component (§3) and cannot be laid out as a DAG.

**Reproduce:** `scripts`-free, no Graphviz — run `context/map/`'s static analyzer
(`ast`-based reverse-dependency + boundary-name extraction over `fastapi/`). Equivalent pydeps
form once `starlette`/`pydantic` are installed: `pydeps fastapi --reverse --show-deps`.
