# bun-dep-types-probe

Mend SCA detection probe targeting dependency-group classification and install-script trust metadata in Bun projects.

## Pattern bundle

- `dep-types-basic` — exercises all four `package.json` dependency groups (`dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`) to assert that Mend classifies each with the correct `group` and `optional` flag.
- `trusted-dependencies` — adds a `trustedDependencies` array to verify that Bun's install-script allowlist field is orthogonal to dep-tree extraction: the dep tree must be identical with or without the array.

## Why bundled

Both patterns edit top-level `package.json` fields that a Mend manifest parser reads in a single pass. Combining them into one probe lets a single scan confirm:

1. The parser reads all four dependency-group fields (not just `dependencies`).
2. Unknown metadata fields (`trustedDependencies`, including a package that is not in any dep group) do not crash the parser or produce ghost deps.

A failure in this probe localizes to either group classification (wrong `group` value) or manifest-metadata tolerance (crash or ghost dep) — both in the same code layer.

## Mend config

No `.whitesource` file is emitted. Bun is NOT in the Mend `install-tool` supported list (`scanSettings.versioning` cannot pin a Bun toolchain version). Detection relies entirely on static lockfile parsing of `bun.lock`. There is no UA pre-step for Bun.

## Package inventory

| Package | Manifest field | Resolved version | Expected `group` | `optional` |
|---|---|---|---|---|
| `hono` | `dependencies` | `4.12.18` | `main` | `false` |
| `typescript` | `devDependencies` | `5.8.3` | `dev` | `false` |
| `react` | `peerDependencies` | `18.3.1` | `main` | `false` |
| `fsevents` | `optionalDependencies` | `2.3.3` | `main` | `true` |

## Group classification table

| Direct dep | `package.json` field | Expected Mend `group` | `optional` flag | Classification risk |
|---|---|---|---|---|
| `hono@4.12.18` | `dependencies` | `main` | — | Baseline — must always be `main`. |
| `typescript@5.8.3` | `devDependencies` | `dev` | — | **Primary assertion.** If emitted as `main`, group classification is broken. |
| `react@18.3.1` | `peerDependencies` | `main` | — | Bun auto-installs peers; they appear in the lockfile as ordinary packages. Mend may drop them (treated as "not in lock") or misclassify. | UPD -  EXPECTED
| `fsevents@2.3.3` | `optionalDependencies` | `main` | `true` | Must carry `optional: true`. Mend may drop entirely on a Linux host even though the lockfile lists the package. |UPD -  EXPECTED

## `trustedDependencies` note

```json
"trustedDependencies": ["fsevents", "esbuild"]
```

`trustedDependencies` is Bun's allowlist for packages permitted to run install scripts (`postinstall`, `preinstall`, etc.). It has no effect on which packages are resolved or at what version. Key assertions:

- `fsevents` is a direct dep in `optionalDependencies` AND appears in `trustedDependencies` — both facts coexist; the dep must still appear in the tree with `optional: true`.
- `esbuild` is listed in `trustedDependencies` but is NOT declared in any dependency group. Mend must NOT invent a dep entry for `esbuild`. If `esbuild` appears in Mend's output tree, that is a false positive caused by misreading `trustedDependencies` as a dependency list.

## Transitive dependency chain

```
react@18.3.1
  └─ loose-envify@1.4.0
       └─ js-tokens@4.0.0
```

`loose-envify` and `js-tokens` are transitive-only. They must appear in the full dep tree but must NOT appear as direct deps of the root.

## Known failure modes

| Failure | Root cause | Mend output symptom |
|---|---|---|
| `typescript` reported as `group: main` | devDependencies not classified separately | Wrong group label |
| `react` missing entirely | peerDependencies not resolved from lockfile | Missing dep |
| `react` reported as `group: dev` | peer deps conflated with dev deps | Wrong group label |
| `fsevents` missing on Linux scan host | Optional dep skipped based on host OS | Missing dep (should still be in tree) |
| `fsevents` missing `optional: true` | Optional flag not propagated from lockfile | Missing flag |
| Ghost `esbuild` dep in tree | `trustedDependencies` misread as dep list | False positive dep |
| Empty tree or parse error | JSONC comments/trailing commas in `bun.lock` crash npm-fallback parser | No deps reported |

## Resolver note (live UA knowledge)

Bun is not listed in the UA JavaScript resolver table. The closest analog is the npm resolver (`NpmLockCollector`), which targets `package-lock.json` / `npm-shrinkwrap.json`. The UA will attempt to parse `bun.lock` via this path. Key divergences:

- `bun.lock` is JSONC (comments + trailing commas allowed) — standard JSON parsers will fail.
- Bun's lockfile tuple format `["name@version", { metadata }, "integrity"]` differs from npm's nested `dependencies` object.
- The `workspaces` top-level key in `bun.lock` maps root manifest groups; npm's lock format does not have this key at the same level.

If the UA's npm-fallback parser cannot read `bun.lock`, this probe produces an empty tree. That empty tree is itself a detectable failure the comparator should flag.

## Tracked in

`docs/BUN_COVERAGE_PLAN.md` §11.1 entry #2
