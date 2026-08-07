# Open Source Contribution Log — AI301

## Status: Cycle 2 — Phase IV In Progress (PR Pending) 🔄

---

## Phase I — Issue Selection

### Issue

**[astroid #600 — numpy: add brain tips for `numpy.fromfile`](https://github.com/pylint-dev/astroid/issues/600)**

Project: [pylint-dev/astroid](https://github.com/pylint-dev/astroid) | Language: Python | Label: `numpy`

---

### Problem Summary

`astroid` is the Python AST framework that powers pylint and other static analysis tools. It uses "brain plugins" — files in `astroid/brain/` — to teach its type inference engine how to handle external libraries like numpy. Currently, `numpy.fromfile` is not recognized by astroid's inference: calls to `np.fromfile(...)` cannot be resolved to their correct return type (`numpy.ndarray`). The fix is to add a brain tip entry for `fromfile` in `astroid/brain/brain_numpy_core_multiarray.py`, following the same pattern already used for `zeros`, `ones`, `empty`, and other numpy functions.

---

### Why I Chose This Issue

I chose astroid issue #600 because it sits at the intersection of Python static analysis and cybersecurity tooling. Astroid powers pylint, Bandit, and similar SAST tools — accurate type inference in astroid means fewer false positives when those tools scan real-world code for vulnerabilities. That connection to security makes this contribution genuinely relevant to the career I am building.

The task is appropriately scoped for a first contribution: one entry added to a dictionary in `brain_numpy_core_multiarray.py` and one test case added to the corresponding test file. The pattern is already established in the same file for a dozen other numpy functions, so the implementation path is clear. The project (pylint-dev) is active and well-maintained, with a CONTRIBUTING.md, a Code of Conduct, and a history of accepting outside PRs. The issue had no assignee at the time I claimed it.

---

### Planned Fix

Add `"fromfile"` to the `METHODS_TO_BE_INFERRED` dictionary in `astroid/brain/brain_numpy_core_multiarray.py`:

```python
"fromfile": """def fromfile(file, dtype=float, count=-1, sep='', offset=0):
        return numpy.ndarray([0, 0])""",
```

Add a corresponding test case in `tests/brain/numpy/test_core_multiarray.py`:

```python
("fromfile", '"data.bin"'),
```

---

## Phase II — Reproduce & Plan

### Local Environment Setup

Cloned my fork of pylint-dev/astroid to `~/Desktop/PROJECTS/astroid` and set up the development environment:

```bash
git clone https://github.com/Genny-oo/astroid.git
cd astroid
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
pip install pytest numpy
```

Environment: Python 3.14, astroid 4.2.0b3, numpy 2.4.6, pytest 9.0.3, macOS (ARM64). Setup completed without errors.

Working branch: `add-numpy-fromfile-brain-tip`

---

### Steps to Reproduce

The issue is that `numpy.fromfile` is missing from astroid's brain plugin, so astroid cannot infer its return type.

1. Set up the local dev environment as above and activate the virtual environment.
2. Run the following Python script from the repo root:

```python
from astroid import builder

# Demonstrate that fromfile is NOT inferred (the bug)
node = builder.extract_node("""
import numpy as np
func = np.fromfile
func("data.bin") #@
""")
print("fromfile infers as:", list(node.infer()))

# Demonstrate that zeros IS inferred correctly (the system works for other functions)
node2 = builder.extract_node("""
import numpy as np
func = np.zeros
func([1, 2]) #@
""")
print("zeros infers as:", [i.pytype() for i in node2.infer()])
```

3. Observe the output:
   - **Before fix:** `fromfile infers as: [Uninferable]`
   - **After fix:** `fromfile infers as: ['.ndarray']`

**Expected:** `fromfile` should infer as `['.ndarray']`, the same as `zeros` and every other numpy function in the brain plugin.

**Actual (before fix):** `fromfile` returns `[Uninferable]` — astroid has no knowledge of this function and cannot determine its return type.

---

### Solution Approach (UMPIRE)

**Understand:** `numpy.fromfile` reads binary or text data from a file and returns a `numpy.ndarray`. It is absent from the `METHODS_TO_BE_INFERRED` dictionary in `astroid/brain/brain_numpy_core_multiarray.py`, which means astroid's inference engine has no stub for it. Any code that calls `np.fromfile(...)` will produce `Uninferable` when analyzed by pylint or other astroid-backed tools.

**Match:** The exact same pattern is already implemented for ~20 other numpy functions in the same file. For example:

```python
"zeros": """def zeros(shape, dtype=float, order='C'):
        return numpy.ndarray([0, 0])""",
```

**Plan:**
1. In `astroid/brain/brain_numpy_core_multiarray.py`: add a `"fromfile"` entry to `METHODS_TO_BE_INFERRED` between `"empty"` and `"bincount"`, using `numpy.fromfile`'s real signature.
2. In `tests/brain/numpy/test_core_multiarray.py`: add `("fromfile", '"data.bin"')` to the `numpy_functions_returning_array` tuple.
3. Run `pytest tests/brain/numpy/test_core_multiarray.py -v` to confirm the new test passes and no existing tests regress.

**Implement:** See Phase III.

**Review:** Changes follow the existing code conventions exactly — no new imports, no new functions, no structural changes.

**Evaluate:** After the fix, the reproduction script outputs `fromfile infers as: ['.ndarray']`. All existing tests pass.

---

## Phase III — Build

### Testing Strategy

The fix required changes to two files:

- **`astroid/brain/brain_numpy_core_multiarray.py`**: Added `"fromfile"` to the `METHODS_TO_BE_INFERRED` dictionary using `numpy.fromfile`'s real signature (`file, dtype=float, count=-1, sep='', offset=0`), returning `numpy.ndarray([0, 0])` consistent with all other entries.
- **`tests/brain/numpy/test_core_multiarray.py`**: Added `("fromfile", '"data.bin"')` to the `numpy_functions_returning_array` tuple, which feeds into `test_numpy_function_calls_inferred_as_ndarray`. This test runs both with `np.fromfile` (aliased) and `numpy.fromfile` (unaliased) automatically, matching how every other function in the file is tested.

### Manual Verification

Ran the test suite after implementing the fix:

```bash
pytest tests/brain/numpy/test_core_multiarray.py -v
```

Results:
- All existing tests passed (no regressions)
- New `fromfile` subtest passed for both aliased (`np.fromfile`) and unaliased (`numpy.fromfile`) usage
- CI checks (codecov, codspeed, readthedocs) all passed on the PR

### Implementation Notes

**What I built:**

Added one entry to `METHODS_TO_BE_INFERRED` in `astroid/brain/brain_numpy_core_multiarray.py`:
```python
"fromfile": """def fromfile(file, dtype=float, count=-1, sep='', offset=0):
        return numpy.ndarray([0, 0])""",
```

Added one line to `numpy_functions_returning_array` in `tests/brain/numpy/test_core_multiarray.py`:
```python
("fromfile", '"data.bin"'),
```

**Files modified:**
- `astroid/brain/brain_numpy_core_multiarray.py`
- `tests/brain/numpy/test_core_multiarray.py`

**Challenges faced:** None significant. The pattern was fully established — matched `numpy.fromfile`'s official signature from the NumPy docs and inserted it after `"empty"` for logical grouping.

**Code Changes:**

Branch: [add-numpy-fromfile-brain-tip](https://github.com/Genny-oo/astroid/tree/add-numpy-fromfile-brain-tip)

| Commit | Description |
|--------|-------------|
| `15acbc1` | Add brain tip for numpy.fromfile |
| `4e616ba` | Add test case for numpy.fromfile brain tip |
| `f9cb8fd` | Add ChangeLog entry for numpy.fromfile brain tip |

---

## Phase IV — Submit & Iterate

### Pull Request

**PR Link:** [pylint-dev/astroid#3121](https://github.com/pylint-dev/astroid/pull/3121)

**Status: ✅ MERGED into pylint-dev:main** (July 7, 2026 — commit `0053ffc`)

**Milestone:** astroid 4.2.0

### Acceptance Criteria

- [x] `"fromfile"` entry added to `METHODS_TO_BE_INFERRED` with correct NumPy signature
- [x] Test case added to `numpy_functions_returning_array` (both aliased and unaliased coverage)
- [x] All existing tests pass — no regressions
- [x] ChangeLog entry added under "What's New in astroid 4.2.0?"
- [x] CI checks pass (codecov 93.62%, codspeed no regressions, docs build succeeded)
- [x] Maintainer reviewed, approved, and merged

### Maintainer Feedback Log

| Date | Reviewer | Feedback | Action Taken | Commit |
|------|----------|----------|--------------|--------|
| Jul 4, 2026 | @Pierre-Sassoulas | "Thank you, it works for both numpy 1 and numpy 2. Would you mind adding a changelog for it?" | Added ChangeLog entry under "What's New in astroid 4.2.0?" | `f9cb8fd` |
| Jul 7, 2026 | @Pierre-Sassoulas | Updated ChangeLog entry format, approved, and merged PR | — | `1ae67ff` → merged `0053ffc` |

### Summary of Contribution

This PR adds inference support for `numpy.fromfile` in astroid's NumPy brain plugin. It fixes an issue where astroid previously returned `Uninferable` when analyzing `np.fromfile(...)`, by adding a missing brain entry so it correctly infers `numpy.ndarray`. The fix works for both NumPy 1.x and 2.x (confirmed by the maintainer).

---

## Learnings & Reflections

### Technical Learnings

**How astroid's brain plugin system works:** Before this project I had no idea how static analysis tools like pylint actually understand external libraries like NumPy without running the code. I learned that astroid uses "brain plugins" — Python files that define stub functions returning fake return-type objects. When astroid encounters `np.fromfile(...)` in user code, it looks up "fromfile" in its plugin registry and uses the stub to determine the return type. This was a completely new concept to me and explains why tools like pylint can warn about type mismatches even in code that imports third-party libraries.

**The inference pipeline:** I learned the difference between "Uninferable" (astroid couldn't determine the type) and a real type like `.ndarray`. The whole `METHODS_TO_BE_INFERRED` dictionary is essentially a hand-crafted type registry for NumPy — a fascinating workaround for the fact that NumPy is largely implemented in C and can't be statically analyzed the normal way.

**How open source contributions actually work:** The PR process was more involved than I expected. Pierre-Sassoulas (a core maintainer) tested the fix against both NumPy 1.x and 2.x before requesting a ChangeLog entry. I learned that established projects have release notes conventions (RST-formatted ChangeLog files) and that maintainers may reformat your entry before merging. The PR going through milestones (4.2.0), labels (Brain 🧠), and CI checks (codecov, codspeed, readthedocs) taught me what a real production-quality review process looks like.

**Git workflow for open source:** I learned how to fork a repo, create a feature branch, make targeted commits, and respond to review feedback by pushing new commits to the same branch. I also experienced merge conflicts firsthand when upstream main diverged from my branch.

### What I Would Do Differently

If I were starting over, I would set up the upstream remote (`git remote add upstream ...`) from the beginning so I could `git fetch upstream && git rebase upstream/main` cleanly instead of having to resolve conflicts via GitHub's web UI.

### Impact

The fix is now live in pylint-dev/astroid and will ship in astroid 4.2.0. Anyone using pylint to analyze Python code that calls `np.fromfile()` will now get accurate type inference instead of "Uninferable" warnings. This directly improves the accuracy of static analysis for the entire Python ecosystem that depends on pylint and astroid.

---

# Cycle 2

## Phase I — Issue Selection

### Issue

**[astroid #1847 — Using auto enum values provides incorrect type](https://github.com/pylint-dev/astroid/issues/1847)**

Project: [pylint-dev/astroid](https://github.com/pylint-dev/astroid) | Language: Python | Labels: `Brain 🧠`, `Bug 🐛`

Fork: [Genny-oo/astroid](https://github.com/Genny-oo/astroid) | Branch: `fix-enum-auto-type-inference`

Setup Docs: [CONTRIBUTING.md](https://github.com/pylint-dev/astroid/blob/main/CONTRIBUTING.rst) | [README](https://github.com/pylint-dev/astroid/blob/main/README.rst)

---

### Problem Summary

When an `IntEnum` class uses `enum.auto()` to assign values, pylint/astroid incorrectly infers the `.value` type as `auto` instead of `int`. For example:

```python
import enum

class EnumWithAuto(enum.IntEnum):
    A = enum.auto()
    B = 10

auto_enum = EnumWithAuto.A
print(auto_enum.value.bit_length())  # E1101: Instance of 'auto' has no 'bit_length' member
```

The `A` member uses `enum.auto()`, so pylint raises a false-positive E1101 error saying `.value` is of type `auto` and has no `bit_length`. But since `EnumWithAuto` inherits from `IntEnum`, all `.value` properties should be `int`.

The root cause is in `infer_enum_class` in `astroid/brain/brain_namedtuple_enum.py`. When building a stub class for the enum, the code checks if `stmt.value` is a `nodes.Const`. If it is, it uses the raw value (an integer). But `enum.auto()` is a `nodes.Call` node, not a `nodes.Const`, so the code falls into the `else` branch and calls `stmt.value.as_string()` — which returns the string `"enum.auto()"`. This string becomes the stub's return value, so astroid infers `.value` as type `auto` instead of `int`.

---

### Why I Chose This Issue

After completing Cycle 1 (adding numpy.fromfile brain support), I wanted to stay in the same codebase (`pylint-dev/astroid`) and go deeper. Issue #1847 is a bug in the enum brain plugin — a completely different part of the codebase from the numpy brain work — so it lets me expand my understanding of how brain plugins handle Python's standard library, not just third-party packages.

The bug affects real-world code: `enum.auto()` is widely used in production Python codebases, and false-positive E1101 errors on `IntEnum` members are a genuine pain point. The issue has been open since 2022, the labels confirm it's a known bug, and no one has claimed it — making it a good opportunity to contribute a meaningful fix.

The scope is well-defined: one function (`infer_enum_class`) in one file (`brain_namedtuple_enum.py`), with a clear fix strategy (detect `nodes.Call` representing `enum.auto()` and substitute an integer return value).

---

### Planned Fix

In `astroid/brain/brain_namedtuple_enum.py`, inside the `infer_enum_class` function, update the `else` branch (lines ~430-433) to detect `enum.auto()` calls and substitute an integer value:

```python
# Before (buggy):
else:
    inferred_return_value = stmt.value.as_string()  # gives "enum.auto()"

# After (fix):
else:
    # Check if this is an enum.auto() call — if so, use int 1 so the
    # stub correctly infers int for IntEnum members
    if (
        isinstance(stmt.value, nodes.Call)
        and isinstance(stmt.value.func, nodes.Attribute)
        and stmt.value.func.attrname == "auto"
    ):
        inferred_return_value = 1
    else:
        inferred_return_value = stmt.value.as_string()
```

A regression test will be added to `tests/brain/test_brain.py` covering the case where an `IntEnum` member assigned via `enum.auto()` correctly infers its `.value` as `int`.

---

### Acceptance Criteria

The fix is complete when all of the following are true:

- [ ] Running pylint on code with `enum.auto()` in an `IntEnum` no longer raises a false-positive `E1101` error on `.value`
- [ ] `Color.RED.value` (where `RED = enum.auto()`) infers as `nodes.Const` with an integer value, not type `auto`
- [ ] Both `auto()` members and literal-value members in the same enum infer correctly
- [ ] All existing enum-related tests in `tests/brain/test_brain.py` still pass (no regressions)
- [ ] New `BrainEnumAutoTest` tests pass

**Files to modify:**
- `astroid/brain/brain_namedtuple_enum.py` — `infer_enum_class` function, ~line 432
- `tests/brain/test_brain.py` — add `BrainEnumAutoTest` class

**Related:**
- Original issue: [#1847](https://github.com/pylint-dev/astroid/issues/1847)
- Same function previously fixed for namedtuple inference: [brain_namedtuple_enum.py](https://github.com/pylint-dev/astroid/blob/main/astroid/brain/brain_namedtuple_enum.py)

---

### Claim Comment

Posted on [issue #1847](https://github.com/pylint-dev/astroid/issues/1847) on August 5, 2026:

> Hi, I'd like to work on this issue! I've traced the bug to `infer_enum_class` in `astroid/brain/brain_namedtuple_enum.py`. When an enum member's value is `enum.auto()`, `stmt.value` is a `nodes.Call` node (not a `nodes.Const`), so the code falls into the `else` branch and calls `stmt.value.as_string()`, which returns the string `"enum.auto()"`. This string is then used as the return value in the generated stub class, causing pylint to infer the `.value` type as `auto` instead of `int`.
>
> The fix would be to detect when `stmt.value` is a `nodes.Call` representing `enum.auto()` and substitute an integer (e.g. `1`) so the type is correctly inferred as `int` for `IntEnum` members. I'll open a PR with the fix and a regression test soon.

---

## Phase II — Reproduce & Plan

### Local Environment Setup

Reused the existing fork and local dev workspace at `~/Desktop/PROJECTS/astroid`. Created a new branch for Cycle 2:

```bash
cd ~/Desktop/PROJECTS/astroid
git checkout main
git checkout -b fix-enum-auto-type-inference
```

Environment: Python 3.14, astroid 4.2.0b3, pytest 9.0.3, macOS (ARM64). No new dependencies needed — the enum module is part of Python's standard library.

---

### Steps to Reproduce

The bug: when an `IntEnum` member uses `enum.auto()`, pylint raises a false-positive `E1101: Instance of 'auto' has no 'bit_length' member`.

1. Activate the local dev environment and run this script from the repo root:

```python
from astroid import builder

node = builder.extract_node("""
import enum

class Color(enum.IntEnum):
    RED = enum.auto()
    BLUE = 10

Color.RED.value #@
""")
inferred = next(node.infer())
print("Type:", type(inferred))
print("Value:", inferred.value if hasattr(inferred, 'value') else "N/A")
```

2. **Before fix:** astroid infers `Color.RED.value` as returning the string `"enum.auto()"` — type `auto`, not `int`.
3. **After fix:** astroid correctly infers `Color.RED.value` as a `nodes.Const` with an integer value.

**Root cause confirmed:** In `infer_enum_class` (line ~432 of `brain_namedtuple_enum.py`), when `stmt.value` is not a `nodes.Const`, the code calls `stmt.value.as_string()`. For `enum.auto()`, which is a `nodes.Call`, this returns the literal text `"enum.auto()"`. That string is embedded into the generated stub class as the return value of `.value`, so astroid infers `.value` as type `auto` instead of `int`.

---

### Solution Approach (UMPIRE)

**Understand:** `enum.auto()` is a function call, represented in astroid's AST as a `nodes.Call` node. The existing code only handles `nodes.Const` (literal values like `10` or `"hello"`) correctly. Any non-Const value falls into an `else` branch that calls `.as_string()` — which works fine for most expressions but returns the wrong thing for `enum.auto()`.

**Match:** The fix follows the same conditional-isinstance pattern already used above it in the same function. No new imports needed — `nodes.Call` and `nodes.Attribute` are already used elsewhere in the file.

**Plan:**
1. In `astroid/brain/brain_namedtuple_enum.py`: add an `elif` branch before the catch-all `else` to detect `enum.auto()` calls and set `inferred_return_value = 1`.
2. In `tests/brain/test_brain.py`: add a new `BrainEnumAutoTest` class with two tests — one for a single `auto()` member, one for mixed `auto()` and literal members.
3. Run `pytest tests/brain/test_brain.py::BrainEnumAutoTest -v` to confirm both pass.

**Implement:** See Phase III.

**Review:** The fix is minimal and targeted — one `elif` block, no new imports, no structural changes. All existing enum tests should continue to pass.

**Evaluate:** After the fix, `.value` on an `enum.auto()` IntEnum member correctly infers as `nodes.Const` with an integer value.

---

## Phase III — Build

### Implementation

**File modified: `astroid/brain/brain_namedtuple_enum.py`**

Added an `elif` branch inside `infer_enum_class` to detect `enum.auto()` calls:

```python
# Before (buggy):
else:
    inferred_return_value = stmt.value.as_string()

# After (fix):
elif (
    isinstance(stmt.value, nodes.Call)
    and isinstance(stmt.value.func, nodes.Attribute)
    and stmt.value.func.attrname == "auto"
):
    # enum.auto() is a Call node, not a Const.
    # Substituting 1 so the stub correctly infers int
    # for IntEnum members instead of the type `auto`.
    inferred_return_value = 1
else:
    inferred_return_value = stmt.value.as_string()
```

**File modified: `tests/brain/test_brain.py`**

Added `BrainEnumAutoTest` class with two regression tests:

```python
class BrainEnumAutoTest(unittest.TestCase):
    """Tests for correct type inference of enum members assigned via enum.auto()."""

    def test_int_enum_auto_value_inferred_as_int(self) -> None:
        """enum.auto() members in IntEnum should have .value inferred as int, not auto."""
        node = builder.extract_node("""
        import enum

        class Color(enum.IntEnum):
            RED = enum.auto()
            GREEN = enum.auto()
            BLUE = 10

        Color.RED.value #@
        """)
        inferred = next(node.infer())
        self.assertIsInstance(inferred, nodes.Const)
        self.assertIsInstance(inferred.value, int)

    def test_int_enum_mixed_auto_and_literal(self) -> None:
        """IntEnum with both auto() and literal values should not raise false E1101."""
        node = builder.extract_node("""
        import enum

        class Status(enum.IntEnum):
            PENDING = enum.auto()
            ACTIVE = 2
            CLOSED = enum.auto()

        Status.PENDING.value #@
        """)
        inferred = next(node.infer())
        self.assertIsInstance(inferred, nodes.Const)
        self.assertIsInstance(inferred.value, int)
```

### Testing

```bash
pytest tests/brain/test_brain.py::BrainEnumAutoTest -v
```

Results:
- `test_int_enum_auto_value_inferred_as_int` — PASSED
- `test_int_enum_mixed_auto_and_literal` — PASSED
- No regressions in existing enum tests

### Code Changes

Branch: [fix-enum-auto-type-inference](https://github.com/Genny-oo/astroid/tree/fix-enum-auto-type-inference)

| Commit | Description |
|--------|-------------|
| `a6cc9d8b` | Fix incorrect type inference for enum.auto() in IntEnum members |

---

## Phase IV — Submit & Iterate

### Pull Request

**PR Link:** [pylint-dev/astroid#3202](https://github.com/pylint-dev/astroid/pull/3202)

**Status:** 🔄 Open — awaiting review

**Target branch:** `pylint-dev/astroid:main`

### Acceptance Criteria

- [ ] `elif` branch added to detect `enum.auto()` in `infer_enum_class`
- [ ] Two regression tests added in `BrainEnumAutoTest`
- [ ] All existing tests pass — no regressions
- [ ] PR open on pylint-dev/astroid
- [ ] Maintainer reviewed and merged

### Maintainer Feedback Log

| Date | Reviewer | Feedback | Action Taken | Commit |
|------|----------|----------|--------------|--------|
| Aug 7, 2026 | @kdelay | 1. Fix doesn't cover `from enum import auto` / `auto()` unqualified form — use `_looks_like()` helper instead. 2. Tests belong in `tests/brain/test_enum.py` not `test_brain.py`. 3. Comment says "correctly infers int" but value `1` is only correct for type, not runtime value for later members. | Used `_looks_like(stmt.value, "auto")` to cover both spellings; moved tests to `EnumBrainTest` in `test_enum.py` and added unqualified `auto()` test; updated comment to clarify type-only inference | `aae66066` |
