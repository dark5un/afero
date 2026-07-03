## Reconnaissance Report: MemMapFs.Rename mutates open file handles

**Target:** Issue #327 -- Renaming a file using MemMapFs behaves differently to OsFs
**Codebase:** spf13/afero, ~14,700 LOC Go
**Blast Radius:** Affects memmap.go (Rename, renameDescendants) and mem/file.go (ChangeFileName, FileData). All other Fs implementations delegate to underlying Fs.
**Root Cause:** MemMapFs.Rename calls mem.ChangeFileName(fileData, newname) which mutates the FileData.name field in-place through a pointer. Since afero.File handles hold the same *FileData pointer, any handle opened before the rename reports the new name after.
**Downstream Impact:** Limited to MemMapFs. OsFs and all delegating Fs types unaffected.
**Surprises:** Existing rename tests only verify old path is gone and new path exists. None test behavior of open file handles after rename.
**Recommended Fix Direction:** Create a new FileData with the new name instead of mutating the existing one. Leave the old FileData alive for already-open handles.
