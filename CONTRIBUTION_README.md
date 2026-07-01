# Contribution: Union Type Support for Streaming Actions in Apache Burr

## Project Information

- **Project:** Apache Burr (incubating)
- **GitHub:** https://github.com/apache/burr
- **Issue:** [#607 — Streaming Event type, type hint, should support union type](https://github.com/apache/burr/issues/607)
- **Working Branch:** `fix/stream_type_support_union_type`
- **Fork:** https://github.com/ThyTran1402/burr/tree/fix/stream_type_support_union_type
- **Status:** Phase IV Complete — PR submitted, awaiting review

---

## Problem Statement

The `@streaming_action.pydantic()` decorator in Apache Burr currently rejects union types (e.g., `Model1 | Model2`) for the `stream_type` parameter, even though they would work at runtime. This limits the flexibility of streaming actions that need to yield multiple different result types.

### Current Behavior (Broken)
```python
union = MyModel1 | MyModel2

@streaming_action.pydantic(
    reads=[...],
    writes=[...],
    state_input_type=..., 
    state_output_type=...,
    stream_type=union,  # Type checker error: invalid type
)
```

### Expected Behavior (After Fix)
```python
@streaming_action.pydantic(
    reads=[...],
    writes=[...],
    state_input_type=..., 
    state_output_type=...,
    stream_type=Model1 | Model2,  # Should work!
)
```

---

## Reproduction Process

### Environment Setup

**Prerequisites:**
- Python 3.9 or higher
- Apache Burr repository cloned and forked
- Virtual environment created and activated

**Setup Steps:**
1. Clone the fork: `git clone https://github.com/ThyTran1402/burr.git`
2. Create virtual environment: `python -m venv .venv`
3. Activate it: `.venv\Scripts\activate` (Windows) or `source .venv/bin/activate` (Unix)
4. Install in development mode: `pip install -e ".[pydantic]"`
5. Verify installation: `python -c "import burr; print(burr.__version__)"`

### Steps to Reproduce

**Step 1: Create Test Models**

Create a file `test_union_repro.py`:

```python
from pydantic import BaseModel
from typing import Generator, Optional, Tuple, Union
from burr.core import State
from burr.core.action import streaming_action

class TextChunk(BaseModel):
    text: str
    chunk_id: int

class StructuredResult(BaseModel):
    summary: str
    confidence: float

class InputState(BaseModel):
    prompt: str

class OutputState(BaseModel):
    response: str
```

**Step 2: Attempt Union Type Decorator**

```python
@streaming_action.pydantic(
    reads=["prompt"],
    writes=["response"],
    state_input_type=InputState,
    state_output_type=OutputState,
    stream_type=TextChunk | StructuredResult,  # Error here
)
def stream_with_union(state: State) -> Generator[Tuple[TextChunk | StructuredResult, Optional[OutputState]], None, None]:
    yield TextChunk(text="Hello", chunk_id=1), None
    yield StructuredResult(summary="Done", confidence=0.95), OutputState(response="Complete")
```

**Step 3: Run Type Checker**

```bash
mypy test_union_repro.py
```

**Expected Error (Before Fix):**
```
error: Argument "stream_type" to "pydantic" of "streaming_action" has incompatible type "Union[type[TextChunk], type[StructuredResult]]"; expected "Union[type[BaseModel], type[dict]]"
```

### Reproduction Evidence

- **Branch:** https://github.com/ThyTran1402/burr/tree/fix/stream_type_support_union_type
- **Test File:** `tests/integrations/test_streaming_union_types.py` (created with 10 comprehensive tests)
- **All Tests Pass:** 10/10 union type tests pass
- **Backward Compatibility:** 33/33 existing pydantic tests pass

---

## Root Cause Analysis

### Problem Location

The issue exists in two files:

**1. `burr/core/action.py` (line ~1510)**
- Method: `streaming_action.pydantic()`
- Current type: `stream_type: Union[Type["BaseModel"], Type[dict]]`

**2. `burr/integrations/pydantic.py` (line ~272)**
- Type alias: `PartialType = Union[Type[pydantic.BaseModel], Type[dict]]`

### Why Union Types Are Rejected

1. **Union types with `|` operator** (Python 3.10+) create `types.UnionType` objects at runtime
2. **Union types with `typing.Union`** create `typing._UnionGenericAlias` objects
3. Neither of these matches the narrow type `Union[Type[BaseModel], Type[dict]]`
4. Type checkers (mypy, Pylance) reject these as incompatible
5. **However**, at runtime, the value flows through correctly without validation

### Key Insight

The type annotation is **purely for IDE/type-checking**. At runtime:
- The union type value flows through unchanged
- No validation happens on the stream_type
- Everything works correctly, but type checkers complain

### Call Chain

```
streaming_action.pydantic()
  ↓ (calls)
pydantic_streaming_action()
  ↓ (calls)
_validate_and_extract_signature_types_streaming()
  ↓ (passes through)
FunctionBasedStreamingAction(intermediate_result_type=stream_type)
```

No runtime logic changes needed—only type annotations.

---

## Solution Approach

### Understanding the Problem

The type system is too restrictive. We need to express that `stream_type` can accept:
- A single BaseModel class
- The dict type
- Union of BaseModel types (both syntaxes)

### Matching Similar Solutions

This pattern exists throughout Python:
- Function return types: `def foo() -> Model1 | Model2:`
- Variable annotations: `result: Model1 | Model2 = ...`
- The type system needs to accept union types as valid values

### Implementation Plan

**File 1: `burr/integrations/pydantic.py` (line 272)**

Change:
```python
PartialType = Union[Type[pydantic.BaseModel], Type[dict]]
```

To:
```python
# PartialType represents the type of intermediate results in a streaming action.
# It can be:
# - Type[pydantic.BaseModel]: A Pydantic model class
# - Type[dict]: The dict type
# - A union of BaseModel types (e.g., Model1 | Model2, or Union[Model1, Model2])
# We use 'object' to accept union types since they are valid runtime values even if
# the type system cannot precisely express them. Union types created with | (Python 3.10+)
# or typing.Union are both supported.
PartialType = Union[Type[pydantic.BaseModel], Type[dict], object]
```

**File 2: `burr/integrations/pydantic.py` (lines 303-310)**

Update function signature:
```python
def _validate_and_extract_signature_types_streaming(
    fn: PydanticStreamingActionFunction,
    stream_type: Optional[PartialType],  # Changed from explicit Union
    state_input_type: Optional[Type[pydantic.BaseModel]] = None,
    state_output_type: Optional[Type[pydantic.BaseModel]] = None,
) -> Tuple[
    Type[pydantic.BaseModel], Type[pydantic.BaseModel], PartialType  # Changed return type
]:
```

**File 3: `burr/core/action.py` (line 1510)**

Update parameter type:
```python
stream_type: Union[Type["BaseModel"], Type[dict], object],
```

Update docstring:
```python
:param stream_type: The pydantic model or dictionary type that is used to represent the partial results.
    Use a dict if you want this untyped. Can also be a union of models (e.g., Model1 | Model2).
```

### Why This Approach Works

**Minimal:** Only type annotations change  
**Backward Compatible:** All existing code continues to work  
**Universal:** Works on Python 3.9+ (all supported versions)  
**Type-Safe:** More specific than `Any`, clear intent with comments  
**No Runtime Changes:** Union types already work at runtime  

---

## Verification Plan

### Test Coverage

**Created:** `tests/integrations/test_streaming_union_types.py` with 10 tests:

1. Single model type (backward compatibility)
2. Dict type (backward compatibility)
3. Union with 2 models (typing.Union syntax)
4. Union with 3 models (typing.Union syntax)
5. Union with 2 models (pipe syntax, Python 3.10+)
6. Union with 3 models (pipe syntax, Python 3.10+)
7. pydantic_streaming_action decorator with union
8. Async streaming with union types
9. Union execution with runtime behavior
10. Union execution with 3 types

### Test Results

```
pytest tests/integrations/test_streaming_union_types.py -v
============================= test session starts =============================
collected 10 items
test_streaming_action_single_model_type_backward_compat PASSED [ 10%]
test_streaming_action_dict_type_backward_compat PASSED [ 20%]
test_streaming_action_typing_union_two_models PASSED [ 30%]
test_streaming_action_typing_union_three_models PASSED [ 40%]
test_streaming_action_pipe_union_two_models PASSED [ 50%]
test_streaming_action_pipe_union_three_models PASSED [ 60%]
test_pydantic_streaming_action_union_decorator PASSED [ 70%]
test_streaming_action_async_union PASSED [ 80%]
test_streaming_action_union_execution PASSED [ 90%]
test_streaming_action_union_execution_three_types PASSED [100%]
============================= 10 passed in 3.28s ================================
```

### Backward Compatibility

```
pytest tests/integrations/test_burr_pydantic.py -v
============================= test session starts =============================
collected 33 items
[... all 33 tests pass ...]
============================= 33 passed in 2.70s ================================
```

---

## Implementation Status

### Phase II Deliverables

- Local environment set up and verified
- Issue reproduced with clear steps
- Root cause identified and documented
- Solution approach documented
- Implementation files scoped (3 changes in 2 files)
- Test strategy defined
- Type annotations updated
- Comprehensive tests written and passing
- All existing tests still pass (backward compatible)

### Files Modified

1. `burr/core/action.py` — Type hint + docstring (1 location)
2. `burr/integrations/pydantic.py` — Type alias + function signature (2 locations)
3. `tests/integrations/test_streaming_union_types.py` — New test file (10 tests)

### Code Changes Summary

```python
# Before
PartialType = Union[Type[pydantic.BaseModel], Type[dict]]

# After
PartialType = Union[Type[pydantic.BaseModel], Type[dict], object]
```

This one-line change (plus documentation) enables full union type support.

---

## Next Steps (Phase III)

- [ ] Submit Phase II check-in with this documentation
- [ ] Wait for mentor feedback
- [ ] Open Pull Request to Apache Burr with:
  - Code changes (3 files)
  - Comprehensive tests
  - Clear commit messages
- [ ] Address any code review feedback
- [ ] Merge to main Burr repository

---

## References

### Project Documentation
- **Burr GitHub:** https://github.com/apache/burr
- **Contributing Guide:** https://github.com/apache/burr/blob/main/CONTRIBUTING.rst
- **Streaming Actions:** https://github.com/apache/burr/blob/main/burr/core/action.py

### Related Code
- Streaming action implementation: [burr/core/action.py](https://github.com/apache/burr/blob/main/burr/core/action.py#L1503)
- Pydantic integration: [burr/integrations/pydantic.py](https://github.com/apache/burr/blob/main/burr/integrations/pydantic.py#L270)
- Type definitions: [burr/core/typing.py](https://github.com/apache/burr/blob/main/burr/core/typing.py)

### Python Type Hints
- [PEP 604 - Union as Type Syntax (|)](https://www.python.org/dev/peps/pep-0604/)
- [typing.Union Documentation](https://docs.python.org/3/library/typing.html#typing.Union)

---

## Questions for Review

1. Is the `object` type in the union acceptable for this use case?
2. Should we add runtime validation to ensure only valid types are passed?
3. Any concerns with the approach or implementation?

---

## Phase II Completion Checklist

- [x] Reproduced the issue with clear steps
- [x] Identified root cause in specific files and lines
- [x] Wrote detailed solution plan
- [x] Scoped changes (3 locations in 2 files)
- [x] Documented test strategy
- [x] Created comprehensive test suite
- [x] All tests passing
- [x] Backward compatibility verified
- [x] Documentation complete
- [x] Ready for mentor review

**Status: Phase II Complete **

---

## Phase IV — Submit & Iterate

> **Note:** The final implementation went slightly beyond the Phase II plan. In
> addition to broadening the type annotation, it **adds runtime validation**
> (`_is_valid_stream_type`) so that invalid `stream_type` values now fail fast with a
> clear `ValueError`. This directly answers the "Questions for Review" above
> (runtime validation *was* added). The "No Runtime Changes" note in the earlier
> sections reflects the original Phase II plan, not the final submitted code.

- **PR Link:** `<PASTE PR URL HERE AFTER OPENING — e.g. https://github.com/apache/burr/pull/XXX>`
- **Contribution:** Added union-type support to Apache Burr's
  `@streaming_action.pydantic()` `stream_type` parameter, so a streaming action can
  declare that it yields one of several Pydantic models (`Model1 | Model2` or
  `Union[Model1, Model2]`). Broadened the `PartialType` annotation, added a recursive
  runtime validator (`_is_valid_stream_type`) that accepts `dict`, `BaseModel`
  subclasses, and unions of those (both `typing.Union` and PEP 604 `|` forms) while
  rejecting invalid types with an informative error, and updated the docstring.
- **Issue:** Closes [apache/burr#607](https://github.com/apache/burr/issues/607)

### Files Changed (vs `apache/burr:main`)

| File | Change |
|---|---|
| `burr/core/action.py` | Broadened `stream_type` annotation + docstring |
| `burr/integrations/pydantic.py` | Broadened `PartialType`, added `_is_valid_stream_type`, added validation in `_validate_and_extract_signature_types_streaming` |
| `tests/integrations/test_streaming_union_types.py` | New — 10 tests (backward compat, both union syntaxes, invalid-type rejection, async, runtime execution) |

### Verification

- `python -m pytest tests/integrations/test_streaming_union_types.py` → **10 passed**
- Existing pydantic integration tests → still passing (backward compatible)
- Diff against `apache/burr:main` is limited to the 3 files above (no scope creep)

### Maintainer Feedback

_None yet — awaiting first review._

### Status

**Awaiting review**

---

**Phase IV Complete.**
