# Attempt 1: Fix presentation mode with multiple windows (Issue #4233)

## The Bug

When `enter_presentation_mode([window1, window2])` is called, only the last
window ends up in presentation mode. The second window's `set_window_state`
call kicks the first window out of presentation.

## Root Cause

All four desktop backends (cocoa, gtk, qt, winforms) have a guard in their
`set_window_state` method that checks if any other window is in PRESENTATION
mode, and if so, calls `app.exit_presentation_mode()` before proceeding. When
the core's `enter_presentation_mode` loops through windows calling
`set_window_state(PRESENTATION)` on each one, the second call triggers this
guard and undoes the first.

## What This Attempt Does

1. **Moved the guard from backends to core.** Removed the guard from all four
   backends' `set_window_state` methods and added a smarter version to the
   core's `window.state` property setter (`core/src/toga/window.py`). The smart
   guard checks `state != WindowState.PRESENTATION` so it doesn't fire when
   entering presentation mode.

2. **Added `_should_exit_presentation_mode` helper** on `Window` to encapsulate
   the guard logic.

3. **Changed `enter_presentation_mode` to use the setter** instead of calling
   `window._impl.set_window_state()` directly, so all state changes go through
   the same code path.

4. **Added a regression test** that asserts both windows end up in PRESENTATION
   state after `enter_presentation_mode([window1, window2])`.

## Concerns / Why We're Trying a New Approach

- The `state` setter has validation (visibility, resizability) that
  `enter_presentation_mode` previously bypassed by calling `_impl` directly.
  Routing through the setter changes behavior — unclear if that's desired.
- The relationship between `window.state = X` (core setter) and
  `window._impl.set_window_state(X)` (backend method) is confusing — they
  sound like they do the same thing but are at different layers. This attempt
  doesn't resolve that fundamental design tension.
- Tests haven't been run yet to validate this approach.
