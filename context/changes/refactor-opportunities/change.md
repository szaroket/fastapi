---
change_id: refactor-opportunities
title: Rank and sequence refactor opportunities from the routing analysis
status: preparing
created: 2026-07-22
updated: 2026-07-22
archived_at: null
---

## Notes

We have an analysis of this repository documenting technical debt and structural
risks: `context/changes/routing-analysis/research.md`. This change answers the
question that analysis deliberately left open: WHICH of those problems are worth
fixing, in what target shape, and in what order.

We explore each recorded problem in the code and in the history, then organize
them as refactor opportunities. The change proceeds in stages:
exploration → decision and plan → implementation. During the exploration stage
no refactor happens and no decision is made.

Exploration output: this change's `research.md`, concluding with a ranking of
options and their trade-offs. First read the report; the decision on what to
implement is made in the planning stage, and the refactor starts only according
to the accepted plan.

Results should be written in English.
