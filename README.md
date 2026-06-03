# Contribution README — CodePath AI301 (Su26)

**Contributor:** Thy Tran (@ThyTran1402)
**Status:** Phase I Complete

---

## Project & Issue

- **Project:** [apache/burr](https://github.com/apache/burr) — a lightweight library for building and debugging stateful AI/LLM applications as state machines.
- **Issue:** [#607 — Streaming Event type, type hint, should support union type](https://github.com/apache/burr/issues/607)
- **Labels:** `good first issue`, `kind/bug`, `priority/high`, `area/streaming`, `area/typing`
- **My fork:** https://github.com/ThyTran1402/burr  <!-- update after forking -->

### Problem Summary

The `stream_type` parameter of the `@streaming_action.pydantic(...)` decorator is annotated as
`Union[Type[BaseModel], Type[dict]]`. This rejects a Pydantic **union** type (e.g. `MyModel1 | MyModel2`)
at type-check time, even though the runtime validation logic can already handle such unions. The fix is to
widen the type annotation (and supporting dispatch logic) so a union of `BaseModel` subclasses is accepted.

**"Fixed" looks like:** passing `stream_type=MyModel1 | MyModel2` type-checks cleanly (mypy passes),
the streaming action validates/dispatches against the correct member type at runtime, and a test covers it.

### Why I Chose This Issue

- It's a `good first issue` / `priority/high` bug — well-scoped and valued by maintainers.
- The fix is bounded: a typing change plus dispatch logic plus a test, not a large refactor.
- It maps to skills I have or can ramp on quickly: Python typing (`Union`, `types.UnionType`, PEP 604) and Pydantic.
- Strong context: the issue has a concrete reproduction example and two reference PRs.

### Known Risk (tracking)

Two PRs already attempt this fix and are **open / stalled in maintainer discussion**:
- [#732](https://github.com/apache/burr/pull/732) (mvanhorn) — adds `types.UnionType` support + 3.9 compat shim + tests.
- [#754](https://github.com/apache/burr/pull/754) (Ghraven) — maintainer flagged the "loosen to `object`" approach as too permissive.

**My plan:** study why the maintainer rejected both approaches, and aim for the suggested direction
(accept `UnionType` in the annotation while doing per-instance dispatch during validation). Will confirm
with a mentor that this issue is still worth pursuing given the open PRs.

---

## Phase II — Understand & Reproduce

- **Understanding of the issue:** _TODO_
- **Local setup notes:** _TODO_ (clone fork, editable dev install, `python -m pytest tests/`)
- **Reproduction steps & evidence:** _TODO_
- **Solution approach:** _TODO_

## Phase III — Implement & Test

- **Testing strategy:** _TODO_
- **Implementation notes:** _TODO_

## Phase IV — Pull Request

- **PR link:** _TODO_
- **PR summary:** _TODO_
- **Maintainer feedback log:** _TODO_
