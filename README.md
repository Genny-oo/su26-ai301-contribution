# Open Source Contribution Log — AI301

## Status: Phase III Complete

---

## Phase I — Issue Selection

### Issue

**[astroid #600 — numpy: add brain tips for `numpy.fromfile`](https://github.com/pylint-dev/astroid/issues/600)**

Project: [pylint-dev/astroid](https://github.com/pylint-dev/astroid) Language: Python
Label: `numpy`

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

```
"fromfile": """def fromfile(file, dtype=float, count=-1, sep='', offset=0):
        return numpy.ndarray([0, 0])""",
```

Add a corresponding test case in `tests/brain/numpy/test_core_multiarray.py`:

```
("fromfile", '"data.bin"'),
```

---

## Phase II — Reproduce & Plan

### Local Environment Setup

Cloned my fork of pylint-dev/astroid to `~/Desktop/astroid` and set up the development environment:

```
git clone https://github.com/Genny-oo/astroid.git
cd astroid
python3 -m venv .venv
source .venv/bin/activate
pip install -e .
pip install pytest numpy
```

Environment: Python 3.14, astroid 4.2.0b3, numpy 2.4.6, pytest 9.0.3, macOS (ARM64). Setup completed without errors.

Working branch: [add-numpy-fromfile-brain-tip](https://github.com/Genny-oo/astroid/tree/add-numpy-fromfile-brain-tip)

---

### Steps to Reproduce

The issue is that `numpy.fromfile` is missing from astroid's brain plugin, so astroid cannot infer its return type.

1. Set up the local dev environment as above and activate the virtual environment.
2. Run the following Python script from the repo root:

```
from astroid import builder

# Demonstrate that fromfile is NOT inferred (the bug)
node = builder.extract_node("""
import numpy as np
func = np.fromfile
func("data.bin")  #@
""")
print("fromfile infers as:", list(node.infer()))

# Demonstrate that zeros IS inferred correctly (the system works for other functions)
node2 = builder.extract_node("""
import numpy as np
func = np.zeros
func([1, 2])  #@
""")
print("zeros infers as:", [i.pytype() for i in node2.infer()])
```

3. Observe the output:

```
fromfile infers as: [Uninferable]
zeros infers as: ['.ndarray']
```

**Expected:** `fromfile` should infer as `['.ndarray']`, the same as `zeros` and every other numpy function in the brain plugin.

**Actual:** `fromfile` returns `[Uninferable]` — astroid has no knowledge of this function and cannot determine its return type.

---

### Solution Approach

**Understand:** `numpy.fromfile` reads binary or text data from a file and returns a `numpy.ndarray`. It is absent from the `METHODS_TO_BE_INFERRED` dictionary in `astroid/brain/brain_numpy_core_multiarray.py`, which means astroid's inference engine has no stub for it. Any code that calls `np.fromfile(...)` will produce `Uninferable` when analyzed by pylint or other astroid-backed tools.

**Match:** The exact same pattern is already implemented for ~20 other numpy functions in the same file. For example:

```
"zeros": """def zeros(shape, dtype=float, order='C'):
        return numpy.ndarray([0, 0])""",
```

This is the pattern to follow.

**Plan:**

1. In `astroid/brain/brain_numpy_core_multiarray.py`: add a `"fromfile"` entry to `METHODS_TO_BE_INFERRED` between `"empty"` and `"bincount"`, using `numpy.fromfile`'s real signature.
2. In `tests/brain/numpy/test_core_multiarray.py`: add `("fromfile", '"data.bin"')` to the `numpy_functions_returning_array` tuple.
3. Run `pytest tests/brain/numpy/test_core_multiarray.py -v` to confirm the new test passes and no existing tests regress.

**Review:** Changes follow the existing code conventions exactly — no new imports, no new functions, no structural changes. Commit message will follow the project's imperative style.

**Evaluate:** After the fix, running the reproduction script above should output:

```
fromfile infers as: ['.ndarray']
zeros infers as: ['.ndarray']
```

All existing tests must continue to pass.

---

## Phase III — Build

### Testing Strategy

The fix required changes to two files:

- **`astroid/brain/brain_numpy_core_multiarray.py`**: Added `"fromfile"` to the `METHODS_TO_BE_INFERRED` dictionary using `numpy.fromfile`'s real signature (`file, dtype=float, count=-1, sep='', offset=0`), returning `numpy.ndarray([0, 0])` consistent with all other entries.
- **`tests/brain/numpy/test_core_multiarray.py`**: Added `("fromfile", '"data.bin"')` to the `numpy_functions_returning_array` tuple, which feeds into `test_numpy_function_calls_inferred_as_ndarray`. This test runs both with `np.fromfile` (aliased) and `numpy.fromfile` (unaliased) automatically, matching how every other function in the file is tested.

To verify:
```bash
pytest tests/brain/numpy/test_core_multiarray.py -v
```

All existing tests should continue to pass. The new `fromfile` subtest should now pass where it previously would have failed with `Uninferable`.

### Implementation Notes

**What I built:**
- Added one entry to `METHODS_TO_BE_INFERRED` in `astroid/brain/brain_numpy_core_multiarray.py`:
  ```python
  "fromfile": """def fromfile(file, dtype=float, count=-1, sep='', offset=0):
          return numpy.ndarray([0, 0])""",
  ```
- Added one line to `numpy_functions_returning_array` in `tests/brain/numpy/test_core_multiarray.py`:
  ```python
  ("fromfile", '"data.bin"'),
  ```

**Files modified:**
- `astroid/brain/brain_numpy_core_multiarray.py`
- `tests/brain/numpy/test_core_multiarray.py`

**Challenges faced:**
None significant. The pattern was fully established — matched `numpy.fromfile`'s official signature from the NumPy docs and inserted it in the same position (after `"empty"`) for logical grouping. No new imports, no structural changes, no style deviations.

**Code Changes:**

Branch: [add-numpy-fromfile-brain-tip](https://github.com/Genny-oo/astroid/tree/add-numpy-fromfile-brain-tip)

Commits:
- Add brain tip for numpy.fromfile — added `"fromfile"` entry to `METHODS_TO_BE_INFERRED`
- Add test case for numpy.fromfile brain tip — added test to `numpy_functions_returning_array`

---

## Phase IV — Submit & Iterate

*(To be filled in during Weeks 4+)*

### Pull Request

### PR Summary

### Maintainer Feedback Log
