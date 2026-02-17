# Review: Skills Engine v0.1 — Doc vs. Implementation Inconsistencies

Reviewed commit: `c8735cd` (feat: skills engine v0.1)

---

## Bug (Functional)

### 1. CI matrix uses skill names instead of directory names

`scripts/generate-ci-matrix.ts:22-31` extracts `manifest.skill` (e.g., `"telegram"`, `"discord"`) as the skill identifier in matrix entries. Both the GitHub Actions workflow (`.github/workflows/skill-tests.yml:58`) and the local test runner (`scripts/run-ci-tests.ts:64`) use these names to construct directory paths:

```bash
npx tsx scripts/apply-skill.ts ".claude/skills/$skill"
```

This constructs `.claude/skills/telegram`, but the actual directories are `.claude/skills/add-telegram` and `.claude/skills/add-discord`. The CI would fail because these paths don't exist. The matrix generator should emit directory names (or a mapping), not raw manifest skill names.

---

## Doc-Code Mismatches

### 2. `skills-system-status.md` "What's Next" lists features that are implemented

Lines 136-143 list uninstall, rebase, CI test matrix, and path remapping as future Phase 3+ work. All four exist as working code in this commit:

- `skills-engine/uninstall.ts` (221 lines)
- `skills-engine/rebase.ts` (292 lines)
- `skills-engine/replay.ts` (302 lines)
- `skills-engine/path-remap.ts` (19 lines)
- `scripts/generate-ci-matrix.ts` + `.github/workflows/skill-tests.yml`

The status doc is stale — it reflects the state before these were built.

### 3. Architecture doc uses `skills/` paths; implementation uses `.claude/skills/`

Throughout `nanoclaw-architecture-final.md`, skill packages are referenced as `skills/add-whatsapp/`, `skills/add-telegram/`, etc. (lines 102-118, 335, 343, 1002-1017). The actual implementation places skills in `.claude/skills/` — confirmed by the manifests, `replay.ts:46`, and the CI workflow. The status doc (line 148) correctly says `.claude/skills/`, but the architecture doc is consistently wrong.

### 4. Required manifest fields differ between docs and code

The architecture doc manifest example (lines 155-160) lists `skill`, `version`, `description`, and `core_version` under `# --- Required fields ---`. The code (`manifest.ts:20-26`) validates:

```typescript
const required = ['skill', 'version', 'core_version', 'adds', 'modifies'] as const;
```

- `description`: documented as required, **not validated** by code.
- `adds` and `modifies`: validated as required, **not listed** as required in the doc.

### 5. Resolution cache key format: versioned in docs, unversioned in code

The architecture doc (line 394) shows versioned resolution keys: `whatsapp@1.2.0+telegram@1.0.0/`. The code (`resolution-cache.ts:13-15`) builds keys from plain skill names:

```typescript
function resolutionKey(skills: string[]): string {
  return [...skills].sort().join('+');
}
```

Actual keys are `discord+telegram`, not `discord@1.0.0+telegram@1.0.0`.

### 6. Shipped resolutions location

The architecture doc (line 389) says resolutions ship from `.nanoclaw/resolutions/`. But `.nanoclaw/` is gitignored (`.gitignore:33`).

The code correctly uses two tiers (`constants.ts:7-8`):

- `SHIPPED_RESOLUTIONS_DIR = '.claude/resolutions'` — in git, ships with the project
- `RESOLUTIONS_DIR = '.nanoclaw/resolutions'` — per-installation, gitignored

The architecture doc doesn't reflect this split.

### 7. Uninstall with custom patch: doc shows interactive choice, code blocks

The architecture doc (lines 846-853) says the user gets:

```
[1] Continue (discard custom patch)
[2] Abort
```

The implementation (`uninstall.ts:40-46`) returns `success: false` with a `customPatchWarning`. The CLI (`scripts/uninstall-skill.ts:14-17`) prints the warning and exits — no interactive choice. The user must manually edit `state.yaml` to proceed.

### 8. Command injection prevention is inconsistent

The status doc (line 59) highlights `execFileSync` in merge.ts to prevent command injection. `merge.ts:28` does use `execFileSync` correctly. But other modules use `execSync` with string interpolation on file paths:

- `apply.ts:230`: `` execSync(`git add "${resolvedPath}"`) ``
- `replay.ts:217`: `` execSync(`git add "${resolvedPath}"`) ``
- `rebase.ts:213`: `` execSync(`git add "${relPath}"`) ``
- `update.ts:186`: `` execSync(`git add "${relPath}"`) ``
- `uninstall.ts:135`: `` execSync(`git apply --3way "${patchPath}"`) ``
- `update.ts:224`: `` execSync(`git apply --3way "${patchPath}"`) ``

Paths with shell metacharacters could be exploited. The `execFileSync` pattern from merge.ts should be applied throughout.

### 9. Init doesn't create `.gitattributes`

Architecture doc Section 15 (lines 976-988) specifies `.gitattributes` content. Implementation guide Phase 1 says "Set up `.gitattributes`". `init.ts:initNanoclawDir()` does not create `.gitattributes`.

### 10. Update flow is much simpler than documented

Architecture doc Section 10 (lines 589-688) describes:

- `migrations.yaml` parsing and migration skill application
- Automatic re-application of updated skills against new base
- Full file operations from update packages
- Multi-step user-facing preview

The implementation (`update.ts`) handles:

- Three-way merge of changed core files
- Re-applying custom patches via `git apply --3way`
- Re-running structured operations from existing state
- Running existing skill tests
- Reading `path_remap.yaml` from update metadata

Not implemented: `migrations.yaml`, migration skills, automatic skill re-application. These are significant architectural features described in detail but absent from code.
