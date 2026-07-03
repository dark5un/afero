## Work Item: MemMapFs.Rename should not mutate open file handles

**Type:** Bug
**Theme:** Experiment 2: Go tech debt refactor on afero (formal process)
**Labels:** experiment:2, language:go, lsp:gopls, codebase:existing, scope:tech-debt

**Description:**

When a user opens a file on MemMapFs and then renames it via `fs.Rename()`, the already-open file handle's `Name()` method returns the new name instead of the original. On OsFs, `os.Rename` does not mutate the state of already-open handles; they report the original name until closed.

This happens because `MemMapFs.Rename` calls `mem.ChangeFileName()` which mutates the `FileData` in-place through a pointer. All open `afero.File` handles hold the same pointer and thus see the updated name.

**Acceptance Criteria:**

- [ ] Open file handle's `Name()` returns original path after rename (matches OsFs)
- [ ] New opens on renamed path return the new path
- [ ] Opens on old path fail with ErrNotExist
- [ ] Directory rename also preserves open handles to descendant files
- [ ] Full test suite passes

**Tasks:**

- [ ] Reconnaissance: trace Rename + renameDescendants + ChangeFileName
- [ ] Write RED test: open handle name preserved after rename
- [ ] Add CopyFileData to mem/file.go
- [ ] Replace ChangeFileName with CopyFileData in Rename
- [ ] Fix registerWithParent to use new FileData
- [ ] Full suite verification
- [ ] Write meditation