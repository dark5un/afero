# Meditation: Go Tech Debt Refactor on afero (Formal Process)

**Date:** July 3, 2026
**Experiment:** Fixing `MemMapFs.Rename` open handle mutation using the formal reconnaissance + work item pipeline
**Repo:** dark5un/afero (forked from spf13/afero)
**Branch:** fix/memmapfs-rename-consistency
**Hypothesis:** The formal pipeline (reconnaissance -> work item -> RED -> GREEN -> REFACTOR -> meditation) produces a more thorough result than the ad-hoc approach, with fewer surprises.

## The Comparison

This is experiment 2 on the same codebase. Experiment 1 (issue #270, Create inconsistency) was done ad-hoc: I explored, found the bug, fixed it, and discovered knock-ons as tests failed. This experiment (issue #327, Rename inconsistency) used the formal process.

## The Formal Process

**Phase 1: Surface Scan** -- baseline tests pass, target identified. 2 minutes.

**Phase 2: Blast Radius** -- traced Rename, renameDescendants, ChangeFileName, FileData, and all Fs implementations. The key insight: `findParent` uses `f.Name()` to locate the parent directory, and `registerWithParent` adds the file to the parent's children list. This meant the fix had to update which FileData pointer was registered, not just how the name was changed. 10 minutes.

**Phase 3: Knock-on Inventory** -- the RED test revealed the exact behavior difference. The reconnaissance report documented the root cause and the recommended fix direction before any code was written. 5 minutes.

**Phase 4: Fix** -- added `CopyFileData`, replaced `ChangeFileName` calls in `Rename` and `renameDescendants`, fixed the `registerWithParent` call to use the new FileData. 10 minutes.

**Full suite:** Passed first time. No surprises.

## What Worked

The reconnaissance report was the key difference. It forced me to trace the full call chain before touching any code. When I wrote the fix, I already knew exactly where the parent registration issue would be -- because the report had documented how `findParent` and `registerWithParent` interact.

The RED test was also better. For experiment 1, I wrote a test that checked for `os.IsNotExist(err)`. For experiment 2, I wrote a test that checked the actual post-rename behavior of the open handle. The first test said "the fix changed the error." The second test says "the fix changed the runtime behavior." The second is more meaningful.

## What Did Not

The formal process took longer in wall clock (about 30 minutes vs 20 for experiment 1). But the time was front-loaded into the reconnaissance phase. The actual fix cycle was faster because there were no surprises.

The `TestRename` test failure was expected (I had documented it in the knock-on inventory). The fix was the `registerWithParent` pointer change, which I would have missed in the ad-hoc approach because I was focused on the Rename method, not the parent registration.

## What Surprised

The most valuable thing the formal process produced was not the fix. It was the reconnaissance report section on `findParent`/`registerWithParent` interaction. Without that analysis, I would have fixed the ChangeFileName call and not realized the `registerWithParent` needed to change too. That would have been a failed test suite and a frustrated debugging session.

The ad-hoc approach (experiment 1) found downstream effects through test failures. The formal approach (experiment 2) found them through analysis. Test failures are reactive. Analysis is proactive. The analysis was faster because it didn't require a compile-run cycle for each discovery.

## Stats

| Metric | Experiment 1 (ad-hoc) | Experiment 2 (formal) |
|--------|----------------------|----------------------|
| Time to first commit | ~20 min | ~30 min |
| Time to fix | ~3 min | ~10 min |
| Fix LOC | 3 | 2 files, ~20 lines total |
| Test failures during fix | 3 (surprises) | 1 (expected) |
| New tests | 1 | 1 |
| Reconnaissance report | None | Written |
| Work item | Retroactive | Written before fix |
| LSP quality | Excellent | Excellent |

## What This Means for Our Thesis

The formal process is slower overall but eliminates surprises. The time shifts from debugging to analysis. The tradeoff is worth it for codebases where the blast radius is non-trivial (which is most of them in practice).

The next question: can we replicate this when the LSP is weaker (Python) and the codebase is unfamiliar (tqdm)?