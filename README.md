# Open Source Contribution Log — AI301

## Status: Cycle 2 — Phase I In Progress 🔄

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
## Phase II — Reproduce & Plan

### Local Environment Setup

Reused the existing fork and local development workspace at `~/Desktop/PROJECTS/astroid`. Since Cycle 2 focuses on a different subsystem (Standard Library Enums instead of NumPy), I updated the working branch and local dependencies to ensure a clean state:

```bash
cd ~/Desktop/PROJECTS/astroid
git checkout main
git pull upstream main
git checkout -b fix-enum-auto-type-inference
pip install -e .
pip install pytest
