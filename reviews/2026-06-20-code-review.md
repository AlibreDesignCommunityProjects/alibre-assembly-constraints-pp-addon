# alibre-assembly-constraints-pp-addon — Code Review (Correctness)

**Date:** 2026-06-20
**Scope:** Second-opinion review, code only (correctness bugs).

**Summary: 1 bug — 1 Medium**

## Medium

**`source/scripts/Template.py:297`** — Integer-division bug in the internal-gear `external_arc` loop: `t = i / (num_points[3] - 1)`.
Under IronPython 2.7.10 (confirmed via `IronPython Version="2.7.10"` in the .vbproj, with no `from __future__ import division`), dividing two ints performs floor division, so `t` is `0` for every iteration except the last (where it is `1`). Every other curve-generation loop in the file correctly uses `float(i) / (num_points[...] - 1)` (e.g. lines 121, 141, 165, 178, 255, 270, 282, 288). The result is that the internal gear's external arc is generated with all interior points collapsed onto the start angle instead of being distributed across the arc, producing an incorrect/degenerate external arc geometry. Fix: use `float(i)` like the sibling loops.

## Notes (lower-confidence / not flagged as confirmed bugs)

- `create_external_gear_in_alibre` / `create_internal_gear_in_alibre` (lines 384–391, 429–436) return a dict on success but a `(False, None)` tuple on the exception paths; the caller (`create_gear_with_unique_name`, line 851) treats the result as a dict and would raise `TypeError` on `parameters['pitch_radius']`, but this is swallowed by the surrounding try/except, so a failed gear silently degrades to a printed warning rather than a user-visible error.
- In `TimerEventProcessor` (lines 624–628), `self.Items.Add(obj_name)` and `self.PreviousSelection.Add(item)` are guarded by independent try/except blocks, so they can desync; later `create_gear_click` checks `Items.Count` and `PreviousSelection is None` but not `PreviousSelection.Count`, so a desync could make `PreviousSelection.Item(0)` (line 798) throw — caught by `@safe_try`, so non-fatal.

No other correctness bugs found. The math/geometry, event wiring, and VB add-on host code otherwise appear sound. The default `num_points` lists are never overridden by callers, so the `(num_points[i]-1)` denominators cannot become zero in practice.

---

## Fixes applied — 2026-06-20

- **[Medium] `source/scripts/Template.py:297`** — `t = i / (num_points[3] - 1)` → `t = float(i) / (num_points[3] - 1)`, matching the sibling loops. Confirmed this was the only unwrapped int/int instance.
- The two lower-confidence notes (dict-vs-tuple return mismatch; Items/PreviousSelection desync) were left as-is, per their not-confirmed status.

*Caveat: change applied to source; not verified by build.*
