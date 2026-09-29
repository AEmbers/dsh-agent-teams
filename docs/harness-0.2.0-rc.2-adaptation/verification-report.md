# Verification report — host bump to `0.2.0-rc.2`

Task: `task-4` — "Verify the adapted fork: static suites + packed artifact inspection".
Author: `verifier` (adversarial verifier, not the author of the change).
Repo under test: `C:\Sophia\dsh-agent-teams` @ `0.1.22-rc.2`.
Host under test: DeepSeek Harness desktop app `0.2.0-rc.2`.
Date of run: 2026-09-29 (local), scratch logs in `C:\Sophia\_verify`.

**This report covers two rounds.** Round 1 (first verification) found 14 PASS / 1 FAIL.
The failure was diagnosed, reported, and fixed by the Lead with a rebuild; round 2
(re-verification) re-ran the affected items and **all 15 items now PASS**. Round 1's
failing evidence is preserved below as history.

## 0. Environment note — the one thing you must know to reproduce this

The bundled pnpm on this machine is **11.7.0**, which enforces a **built-in
`minimumReleaseAge` supply-chain policy of 1440 minutes (24 h)**. The
`0.2.0-rc.2` packages were published ~22 h before this run, so they fall inside
the cutoff and the policy rejects them:

```
[ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION] 278 lockfile entries failed verification:
  @deepseek-ai/dsh-acp-app@0.2.0-rc.2 was published at 2026-09-29T09:39:39.211Z,
  within the minimumReleaseAge cutoff (2026-09-28T11:30:26.191Z)
```

This is a **local toolchain policy, not a repo defect**. The repo's CI uses
pnpm@10.33.0, which has no such policy.

Critically, the neutralisation given in the task brief works for
`pnpm install` but **does not work for `pnpm run`**:

| Form | Effect |
| --- | --- |
| `$env:npm_config_minimum_release_age = "0"` | **IGNORED** — `pnpm config get minimum-release-age` still returns `undefined`; policy stays active. |
| `--config.minimum-release-age=0` | **WORKS** — the flag is honoured by the child `pnpm install`. |

`pnpm run <script>` triggers a pre-run dependency check
(`runDepsStatusCheck` → `spawnSubprocessSync` → `pnpm install`). With only the
env var set, **every** `pnpm`-based suite dies at ~3.2 s with exit 1 before the
suite's own code ever runs:

```
[ERROR] Command failed with exit code 1: pnpm install
pnpm: Command failed with exit code 1: pnpm install
    at runDepsStatusCheck (.../pnpm/dist/pnpm.mjs:246833:7)
```

**All results below were therefore produced with the CLI flag:**

```powershell
node "<pnpm.cjs>" --config.minimum-release-age=0 run <script>
```

This first-run false failure is recorded here because a naive re-run without the
flag will reproduce 11 bogus failures and must not be mistaken for the change
breaking things.

`pnpm-workspace.yaml` was inspected and is **valid JSON** — 273 overrides, all
`0.2.0-rc.2`, no `minimumReleaseAge` / `minimumReleaseAgeExclude` key, not
collapsed to one line. No tampering to report. (Re-confirmed in round 2; see §R2.3.)

---

## 1. `verify:compatibility` — **PASS** (exit 0)

```powershell
node "<pnpm.cjs>" --config.minimum-release-age=0 run verify:compatibility
```

Exit **0** (12.3 s). `tests 10 / pass 10 / fail 0`, plus `Dependency inspection passed.`
Notable: `✔ source candidates pass the peer gate without entering the published download matrix`,
`✔ doctor detects peer-only drift and a mismatched installed plugin`.
(`scripts/compatibility.mjs` JSON.parses `pnpm-workspace.yaml` — it succeeded, which independently
confirms the workspace file is valid JSON.)

*Re-run after the rebuild in round 2 — still exit 0, 10/10; see §R2.4.*

## 2. `verify:readme` — **PASS** (exit 0)

Exit **0** (0.7 s). `tests 7 / pass 7 / fail 0`.
`README version references match 0.1.22-rc.2`.

## 3. `verify:release` — **PASS** (exit 0)

Exit **0** (0.6 s). `tests 6 / pass 6 / fail 0`.

## 4. `verify:quality-gates` — **PASS** (exit 0)

Exit **0** (0.9 s). `tests 17 / pass 17 / fail 0`; every `PASS  tdd.*` gate printed.

## 5. `verify:web-routes` — **PASS** (exit 0)

Exit **0** (0.8 s).
`PASS Web routes: real host authentication, bounded bodies, trusted origins, fail-closed startup, disposal and reload`

*Re-run after the rebuild in round 2 — still exit 0; see §R2.4.*

## 6. `verify:capabilities` — **PASS** (exit 0)

Exit **0** (1.6 s). `tests 18 / pass 18 / fail 0`.

## 7. `verify:member-spawn-recovery` — **PASS** (exit 0)

Exit **0** (0.7 s). `tests 10 / pass 10 / fail 0`.

## 8. `test:git-install` — **PASS** (exit 0)

Exit **0** (0.7 s). `tests 6 / pass 6 / fail 0`.

## 9. `verify:git-artifact` — **PASS** (exit 0) — was FAIL, resolved by rebuild

> **STATUS: RESOLVED — PASS on re-verification.** Round 1 failed with exit 1. The
> failing evidence is kept verbatim below as history, followed by the root cause
> and the round-2 result. **Nothing in the failure ever referenced a host version.**

### 9a. Original result, round 1 — **FAIL** (exit 1)

```powershell
node "<pnpm.cjs>" --config.minimum-release-age=0 run verify:git-artifact
```

Exit **1** (1.6 s). Reproducible. Exact output:

```
$ node scripts/git-artifacts.mjs
file:///C:/Sophia/dsh-agent-teams/scripts/git-artifacts.mjs:37
    throw new Error('Git artifacts are stale or modified; run pnpm build and include lib/ with the source changes')
          ^

Error: Git artifacts are stale or modified; run pnpm build and include lib/ with the source changes
    at verifyGitArtifacts (file:///C:/Sophia/dsh-agent-teams/scripts/git-artifacts.mjs:37:11)
    at file:///C:/Sophia/dsh-agent-teams/scripts/git-artifacts.mjs:44:10
[ELIFECYCLE] Command failed with exit code 1.
```

**Exact assertion text (file:line):**
`scripts/git-artifacts.mjs:37` —
`throw new Error('Git artifacts are stale or modified; run pnpm build and include lib/ with the source changes')`

### 9b. Root cause (independently derived, not summarised away)

I re-derived the stamp components in scratch (`C:\Sophia\_verify\stamp-diag.cjs`,
replaying the digest logic of `scripts/git-artifacts.mjs:18-28`):

| Component | Saved | Actual | Verdict |
| --- | --- | --- | --- |
| `schema` | `1` | `1` | OK |
| `source` | `29bc3d1a93694865e5701d7ac61a2d6e70d37dbdf19cf56e82416f877609c03a` | `653882d11584e6e4f1d8620cbbbca26c49362fc3991a480735f00089851bbd21` | **MISMATCH** |
| `output` | `fbfcc40d4c97378b3c16bb0a070926dc9a73600d45a187ab05bf1b0e25410ed0` | `fbfcc40d4c97378b3c16bb0a070926dc9a73600d45a187ab05bf1b0e25410ed0` | OK |

`lib/` is **intact and consistent** — the output digest matched exactly, so the
build output was not stale and no source file changed after the build. The
mismatch was entirely in the **source** digest, whose inputs are `src/**` plus
`CONFIG = [tsconfig.json, tsconfig.client.json, tsdown.config.ts, pnpm-lock.yaml,
pnpm-workspace.yaml, scripts/clean-build.mjs, scripts/git-artifacts.mjs]`
(`scripts/git-artifacts.mjs:9`).

Timestamp correlation made the culprit unambiguous — exactly one digested input
was newer than the stamp:

```
11:23:08Z  package.json            (version/devDeps bump)
11:23:14Z  compatibility.json
11:26:37Z  pnpm-lock.yaml          (regenerated by install)
11:27:20Z  lib/git-artifact-stamp.json   <-- build wrote the stamp here
11:28:43Z  pnpm-workspace.yaml     <-- ONLY digested input modified AFTER the stamp
```

All other `CONFIG` inputs and all of `src/**` carried mtimes older than the stamp,
and the `lib/` output digest matched — therefore **`pnpm-workspace.yaml` was
necessarily the sole cause**: it was edited 83 s after the build wrote its stamp.

The edit was the Lead's deliberate post-build restore of `pnpm-workspace.yaml` to
valid JSON (documented in the task brief). The restore is correct and must stay;
it simply happened after the build, and `pnpm-workspace.yaml` is inside the
stamp's digested input set, so it invalidated the stamp.

**Classification: build-ordering artifact, NOT a `0.2.0-rc.2` incompatibility.**
No wrong-version assertion was found anywhere in this run.

### 9c. Resolution (applied by the Lead, re-verified by me in round 2)

The prescribed remedy was to re-run the build after the workspace file was frozen,
since `build` re-stamps as its last step (`node scripts/git-artifacts.mjs --write`).
The Lead did exactly that, then I independently re-ran the gate:

```powershell
node "<pnpm.cjs>" --config.minimum-release-age=0 run build            # Lead, EXIT=0
node "<pnpm.cjs>" --config.minimum-release-age=0 run verify:git-artifact   # verifier
```

**Round-2 result — exit 0:**

```
$ node scripts/git-artifacts.mjs
Git artifacts match source and build settings.
```

Ordering is now consistent, which is what the fix predicts:

```
11:28:43Z  pnpm-workspace.yaml          (unchanged since round 1)
11:34:07Z  lib/git-artifact-stamp.json  <-- re-stamped AFTER the workspace file
```

Independently confirmed the rebuild was surgical: `git diff --stat -- lib` is
still exactly **2 files** (`lib/client.js`, `lib/git-artifact-stamp.json`), and
the re-stamped `source` digest is now
`9ebbc04f6f058bad474cbbcd57596f4c9a6e541036db1e419a240088516b079e`.

## 10. `verify:http-body` — **PASS** (exit 0)

Exit **0** (0.8 s). `tests 2 / pass 2 / fail 0`.

## 11. `node --test scripts/issue-159.test.mjs` — **PASS** (exit 0)

Exit **0** (0.3 s). `tests 9 / pass 9 / fail 0`.

## 12. `verify:skill` — **PASS** (exit 0)

Exit **0** (0.7 s). `DSH skill mirrors are up to date: 11 skills, 128 files.`

---

## 13. `pnpm pack` — **PASS** (exit 0)

```powershell
node "<pnpm.cjs>" pack --out C:\Sophia\_verify\agentteams-0.1.22-rc.2.tgz
```

Exit **0**. Produced `@nanmicoder/dsh-agent-teams@0.1.22-rc.2`.

**Artifact under test (round 2, current):**

| Property | Value |
| --- | --- |
| Path | `C:\Sophia\_verify\agentteams-0.1.22-rc.2.tgz` |
| **SHA-256** | **`A1B33FB739B226E60CECE8B5BB81AF69C8DA323A110AB533744C4C00677B3A0C`** |
| Size | 2 099 869 bytes |
| Files in tarball | 113 |
| Produced | re-pack after the rebuild, 2026-09-29T11:34:18Z |

**`pnpm pack` is deterministic.** I packed twice — once to the canonical path and
once to a second path — and both produced the **identical** SHA-256 and byte size:

```
SHA256(canonical)   = A1B33FB739B226E60CECE8B5BB81AF69C8DA323A110AB533744C4C00677B3A0C
SHA256(second pack) = A1B33FB739B226E60CECE8B5BB81AF69C8DA323A110AB533744C4C00677B3A0C
DETERMINISTIC: True
```

This matters for identity: the digest is a **stable content identity**, so the
digest movement observed during the session (`065BE4E3…` → `A1B33FB7…`) is
attributable to the **rebuild changing `lib/` content**, not to pack
nondeterminism. The Lead's re-packed tarball and my independent re-pack are the
same bytes, so both observers were testing the same artifact.

## 14. Ship-time manifest inspection — **PASS**

Extracted to a fresh scratch dir (`C:\Sophia\_verify\rerun\package\`) from the
current tarball `A1B33FB7…` and inspected the **shipped** `package.json`:

| Check | Result |
| --- | --- |
| `version` is `0.1.22-rc.2` | PASS — `0.1.22-rc.2` |
| `@deepseek-ai/dsh-*` peerDependencies | PASS — exactly **21** (24 total peers; the other 3 are `@deepseek-ai/cordis`, `@deepseek-ai/schemastery`, `react`) |
| every one of the 21 ranges contains the exact token `0.2.0-rc.2` | PASS — 21/21, **0** lacking the token |
| tarball contains `lib/` | PASS (+ `lib/index.js`, `lib/client.js`, `lib/types/index.d.ts`, `lib/types/client/index.d.ts` all present) |
| tarball contains `compatibility.json` | PASS |
| tarball contains `cordis.patch.yml` | PASS |

All 21 ranges are the identical string:

```
0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2 || 0.2.0 || 0.2.0-rc.2
```

Note this range **still advertises the legacy `0.1.7-rc.2` and the `0.2.0`
line** ahead of the new token. That is backwards-compatible by design, not a
defect — but it means the token check alone cannot prove the plugin *prefers*
`0.2.0-rc.2`; semantics come from the repo's 273 overrides, which item 15 covers.

**Anti-stale check (round 2):** the `lib/git-artifact-stamp.json` shipped inside
the tarball is byte-identical to the repo's post-rebuild stamp
(`source = 9ebbc04f6f058bad474cbbcd57596f4c9a6e541036db1e419a240088516b079e`),
proving the re-pack picked up the **rebuilt** `lib/` rather than re-shipping the
pre-rebuild artifacts. Without this check, item 14 could pass while shipping
stale output.

## 15. Override closure re-derivation — **PASS** (exit 0)

```powershell
node C:\Sophia\_override_check.cjs
```

Exit **0**. Re-derived independently against the installed host's shipped
package closure read from `app.asar`:

```
overrides total          : 273
override -> host 0.2.0-rc.2 : 273
override -> host OTHER ver  : 0
override -> NOT in host     : 0
devDependencies not in host : 0

host @deepseek-ai packages: 287 | at 0.2.0-rc.2: 279
host dsh-* packages      : 279 | at 0.2.0-rc.2: 279
dsh-* covered by repo overrides: 273 / 279
```

**273/273 map to host `0.2.0-rc.2`, 0 conflicts, 0 missing — matches expectation.**
The 6 host `dsh-*` packages not pinned by repo overrides are
`@deepseek-ai/dsh-client-product-analytics`, `@deepseek-ai/dsh-client-ui-settings-session-log`,
`@deepseek-ai/dsh-desktop-host`, `@deepseek-ai/dsh-experimental-schedule-bundle`,
`@deepseek-ai/dsh-host-product-telemetry-otel`, `@deepseek-ai/dsh-otel` — all
host-internal/telemetry-desktop packages that the plugin does not depend on.

---

## Round 2 — re-verification after the rebuild

Re-verification was requested by the Lead after applying the §9c fix.

### R2.1 Affected items re-run

| # | Command | Exit | Verdict |
| --- | --- | --- | --- |
| 9 | `run verify:git-artifact` | **0** | **PASS** — "Git artifacts match source and build settings." |
| 13 | `pack --out …agentteams-0.1.22-rc.2.tgz` | 0 | **PASS** — SHA-256 `A1B33FB7…`, deterministic (two packs, identical digest) |
| 14 | shipped-manifest inspection of that tarball | n/a | **PASS** — see §14, incl. anti-stale stamp check |

### R2.2 Regression spot-checks (rebuild must not disturb passing suites)

| # | Command | Exit | Verdict |
| --- | --- | --- | --- |
| 1 | `run verify:compatibility` | **0** | **PASS** — `tests 10 / pass 10 / fail 0`, `Dependency inspection passed.` (incl. all 6 `doctor` tests) |
| 5 | `run verify:web-routes` | **0** | **PASS** — `PASS Web routes: real host authentication, bounded bodies, trusted origins, fail-closed startup, disposal and reload` |

No regression from the rebuild.

### R2.3 Workspace-file re-check (must still be valid JSON)

```
VALID JSON; overrides=273
minimumReleaseAge key present: false
minimumReleaseAgeExclude key present: false
non-0.2.0-rc.2 override values: 0
```

PASS — `pnpm-workspace.yaml` is still valid JSON with **exactly 273** overrides,
all valued `0.2.0-rc.2`, and **no** `minimumReleaseAge` / `minimumReleaseAgeExclude`
key. Not collapsed to a single line. Byte-mtime unchanged from round 1
(`2026-09-29T11:28:43Z`), i.e. the rebuild did not rewrite it.

### R2.4 Rebuild scope confirmed

`git diff --stat -- lib` = **2 files** (`lib/client.js`, `lib/git-artifact-stamp.json`) —
matches the Lead's stated scope, independently confirmed.

---

## Summary table

| # | Command | Round 1 | Round 2 | Verdict |
| --- | --- | --- | --- | --- |
| 1 | `run verify:compatibility` | 0 | 0 | **PASS** (10/10) |
| 2 | `run verify:readme` | 0 | — | **PASS** (7/7, "README version references match 0.1.22-rc.2") |
| 3 | `run verify:release` | 0 | — | **PASS** (6/6) |
| 4 | `run verify:quality-gates` | 0 | — | **PASS** (17/17) |
| 5 | `run verify:web-routes` | 0 | 0 | **PASS** |
| 6 | `run verify:capabilities` | 0 | — | **PASS** (18/18) |
| 7 | `run verify:member-spawn-recovery` | 0 | — | **PASS** (10/10) |
| 8 | `run test:git-install` | 0 | — | **PASS** (6/6) |
| 9 | `run verify:git-artifact` | **1** | **0** | **PASS** (resolved by rebuild; §9) |
| 10 | `run verify:http-body` | 0 | — | **PASS** (2/2) |
| 11 | `node --test scripts/issue-159.test.mjs` | 0 | — | **PASS** (9/9) |
| 12 | `run verify:skill` | 0 | — | **PASS** (11 skills, 128 files) |
| 13 | `pnpm pack --out …` | 0 | 0 | **PASS** — SHA-256 `A1B33FB739B226E60CECE8B5BB81AF69C8DA323A110AB533744C4C00677B3A0C`, deterministic |
| 14 | shipped `package.json` inspection | n/a | n/a | **PASS** — 0.1.22-rc.2; 21/21 dsh peers carry `0.2.0-rc.2`; `lib/`, `compatibility.json`, `cordis.patch.yml` present; shipped stamp == repo stamp |
| 15 | `node C:\Sophia\_override_check.cjs` | 0 | — | **PASS** — 273/273 → 0.2.0-rc.2, 0 conflicts, 0 missing |

Counts: **Round 1: 14 PASS / 1 FAIL / 0 not-run.** **Round 2 (final): 15 PASS / 0 FAIL / 0 not-run.**
No item was skipped or reported as a pass without executing.

## Verdict

**The `0.2.0-rc.2` adaptation is verified and sound.** All 15 items pass as of
round 2, including every check that actually exercises host-version
compatibility: the peer/dependency gate (item 1), the capability surface
(item 6), web routes (item 5), the shipped manifest (item 14) and the full
override closure against the installed host (item 15, 273/273).

The single round-1 failure (item 9) was **not** a version-bump defect. It was a
build-ordering artifact: `pnpm-workspace.yaml` was edited 83 s *after* the build
wrote `lib/git-artifact-stamp.json`, while that file is inside the stamp's
digested input set. The diagnosis was decisive — the `lib/` **output** digest
matched exactly, all `src/**` and every other `CONFIG` input predated the stamp,
and `pnpm-workspace.yaml` was the only digested input that postdated it. Re-running
the build after the workspace file was frozen re-stamped the tree, and
`verify:git-artifact` now exits 0 (§9c), with `verify:compatibility` and
`verify:web-routes` re-confirmed green and `git diff --stat -- lib` still exactly
2 files.

**Remaining caveats for the release decision — neither blocks, both are
environment-scoped:**

1. Any future edit to `pnpm-workspace.yaml`, the lockfile, `package.json`
   devDependencies, or `tsconfig*`/`tsdown.config.ts` invalidates the stamp again
   and requires a rebuild before `verify:git-artifact` will pass.
2. `pnpm run` on this machine needs `--config.minimum-release-age=0`; plain
   `pnpm run <script>` fails all pnpm suites on the local 24 h supply-chain
   policy, which is a toolchain artefact and not a repo defect.

No repo script was found asserting an old host version anywhere in this run.

---

## Correction to a round-1 secondary observation (withdrawn)

Round 1 of this report raised a "latent bug" claim that `lib/git-artifact-stamp.json`
does not cover `package.json` settings because the stamp JSON has no `settings`
key and `verifyGitArtifacts` never compares `saved.settings`.

**That claim was WRONG and is withdrawn.** The Lead corrected it, and I confirmed
it against the source: `current()` at `scripts/git-artifacts.mjs:27` passes the
`settings` projection as the **third positional argument** to `digest` —
`digest(root, [...paths], settings)` — which binds to the `prefix` parameter
(`scripts/git-artifacts.mjs:18`). `digest` hashes it first via
`hash.update(prefix)` (`:19`) and folds it into the returned `source` digest,
which `verifyGitArtifacts` (`:36`) does compare (`saved.source !== actual.source`).

So any change to `name`, `version`, `type`, `exports`, `devDependencies`,
`dependencies` or the `build` script **does** change `source` and **is** detected.
There is **no coverage gap**. The absence of a `settings` key in the stamp file is
correct and expected: the value is folded into `source`, not stored separately.
My round-1 scratch comparison of "saved.settings vs actual settings" was
meaningless because no such key exists on either side.

Corrected conclusion: `scripts/git-artifacts.mjs` has no missing-coverage defect.
