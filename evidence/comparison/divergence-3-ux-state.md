# Divergence 3: UX in-flight state

## The difference

| | Track A | Track B |
|---|---|---|
| Button state | None (always clickable) | Tracks `deletingId` state |
| User can | Double-click to send two requests | Double-click blocked; button disabled during deletion |
| Label | No change | Switches to "Deleting…" during deletion |

## Why it matters

A's double-click defect is a UX and potentially a data problem:

1. **User friction:** Double-click fires two DELETE requests. No visual feedback, so user doesn't know first request succeeded.
2. **Duplicate request behavior:** The second request is avoidable and can return an error path (for example, not found after the first request completes).
3. **Wasted work:** Even in the happy case, two identical requests is inefficient.

B prevents this by disabling the button and showing "Deleting…", so the second click does nothing.

## The mechanism

B caught this via a schema checklist item A didn't have:

1. **feature.schema.yaml checklist:** `loading_state_handled`
   - "UI must show loading state during async operations"
   - This item is required in B's schema
   - This item is absent from A's schema
   
2. **What B did:**
   - Identified DELETE as an async operation
   - Asked: "How does the user know deletion is in progress?"
   - Added per-item `deletingId` state
   - Disabled button while deletion is in flight
   - Showed "Deleting…" label
   
3. **What A did:**
   - No requirement to add loading state
   - Implemented the feature without state tracking
   - Relied on implicit success (response arrives quickly)

## Evidence

**B's compliance log:**  
[../../evidence/B/compliance-log.yaml](../../evidence/B/compliance-log.yaml), event 9:
> "Checklist items `loading_state_handled` and `error_state_handled`: per-item `deletingId` state disables the row's Delete button and shows `"Deleting…"`"

**Code diff:**  
- [../../evidence/B/git-diff-stat.txt](../../evidence/B/git-diff-stat.txt): Frontend pages changed (35 lines added for state tracking)
- [../../evidence/A/git-diff-stat.txt](../../evidence/A/git-diff-stat.txt): Frontend pages changed (36 lines added) but for basic UI wiring, not state tracking

**Test coverage:**  
- B added explicit tests for the loading state
- A's tests don't verify loading state (because it wasn't added)

## Conclusion

This divergence has nothing to do with knowledge ("how to add loading state?") or standards (REST conventions). It's about **required checklist items forcing deliberation**.

B's schema said: "You must handle loading state." A had no such requirement, so A didn't add it.

This is the most obvious example of how schemas work: they transform "nice to have" into "must do", which creates a deliberate question and a documented answer.
