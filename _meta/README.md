# _meta

This directory contains our experiment artifacts for the afero fork.

```
_meta/
  README.md                 -- this file
  meditations/              -- experiment retrospectives
  items/                    -- work item definitions
  reconnaissance/           -- blast radius reports
  skills/                   -- reusable process skills
  articles/                 -- LinkedIn posts, chapter drafts
  references/               -- external reference material
```

## Experiments

| # | Issue | Scope | Branch | Status |
|---|-------|-------|--------|--------|
| 1 | #270: MemMapFs.Create auto-creates parent dirs | Tech debt fix | `fix/memmapfs-create-consistency` | Done |
| 2 | #327: MemMapFs.Rename mutates open file handles | Tech debt fix | `fix/memmapfs-rename-consistency` | Done |

For the full methodology, see `skill_view(name='codebase-reconnaissance')` and `skill_view(name='story-composer')`.