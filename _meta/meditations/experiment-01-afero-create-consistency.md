# Meditation: Go Tech Debt Refactor on afero

**Date:** July 3, 2026
**Experiment:** Fixing `MemMapFs.Create` inconsistency against an existing, real-world codebase
**Repo:** dark5un/afero (forked from spf13/afero)
**Branch:** fix/memmapfs-create-consistency
**Hypothesis:** The TDD pipeline holds for Go tech debt work in an existing codebase where the LSP (gopls) is solid but the code structure is unfamiliar.

## The Setup

We forked `spf13/afero`, a 14,700-line Go filesystem abstraction library with 6.7k GitHub stars, 537 commits, and 104 open issues. We targeted issue #270: `WriteFile` creates non-existing directories on `MemMapFs` but fails on `OsFs`. The issue had been open since August 2020.

No PRs filed. No maintainers contacted. This was our experiment, not their problem.

## The Work

The pipeline we used: blast radius analysis, reproduction test, fix, audit downstream effects, full suite verification.

Blast radius revealed that `MemMapFs.Create` called `registerWithParent`, which called `lockfreeMkdir` on any missing ancestor directory. `OsFs.Create` delegates to `os.Create`, which never creates parents. All other Fs implementations (BasePathFs, HttpFs, RegexpFs, ReadOnlyFs, CacheOnReadFs, CopyOnWriteFs) delegate to the underlying Fs and were affected indirectly.

The fix itself was simple: check parent directory exists before creating in `Create`. If not, return `os.ErrNotExist`. Three lines of logic.

## What Worked

The blast radius tool found all 9 Create implementations and all 5 call sites of registerWithParent in one pass. Without it, I would have missed CacheOnReadFs.Create entirely.

gopls was excellent. Go's static typing made refactoring safe. When I hit a test failure, the compiler told me exactly which import was missing (filepath in cacheOnReadFs.go).

The RED-GREEN-REFACTOR cycle still held for bug hunting, not just feature building.

## What Did Not

The tests exposed something uncomfortable. The existing test suite had a quiet dependency on the buggy behavior. `TestLstatIfPossible`, `TestRead0`, `TestUnionCacheWrite` all worked because MemMapFs silently created parent directories. Our fix revealed these tests were testing in a world that did not match real OsFs behavior.

Fixing each test was mechanical (add `MkdirAll` calls), but each one required understanding a different part of the codebase first. Not hard, just slow.

CacheOnReadFs.Create was missing a parent-directory check on its layer. This was a genuine bug masked by MemMapFs's leniency. CopyOnWriteFs.OpenFile already had the fix. The pattern inconsistency between the two composite filesystems is itself a design smell.

## What Surprised

The slow part was not the fix. The slow part was the audit trail: tracing which callers relied on the behavior, which tests would break, and which downstream composite filesystems expected the old contract. Three minutes to write the fix. Twenty minutes to track the blast radius and fix the fallout.

The existing test suite passes now, but I am not confident we found every knock-on effect. There is an unknown number of users calling `fs.Create("nonexistent/path/file.txt")` on MemMapFs who will silently get different behavior than on OsFs. Our fix makes them consistent by failing on both.

This is the real cost of behavioral inconsistency: it compounds silently until someone tries to make the abstraction actually watertight.

## Stats

| Metric | Value |
|--------|-------|
| Files changed | 4 (memmap.go, ioutil.go, cacheOnReadFs.go, lstater_test.go) |
| Lines added | 35 |
| Lines removed | 2 |
| Tests passing | 100% (before and after) |
| New test | TestMemFsCreateWithoutParentDir |
| Blast radius time | ~15 min |
| Fix time | ~3 min |
| Downstream fixes | ~20 min |
| LSP quality | Excellent |

## What This Means for Our Thesis

The pipeline works for Go tech debt refactor in an existing codebase. The LSP is good enough. The blast radius tooling is essential -- without it, CacheOnReadFs.Create would have been missed until CI failed.

The methodology question this raises: in a messy codebase, the hardest part is not the fix itself. It is knowing what you are about to break. Blast radius analysis is not optional here. It is the entire value proposition.

The next question for the Python experiment: can we replicate this when the LSP gives us hints instead of guarantees?