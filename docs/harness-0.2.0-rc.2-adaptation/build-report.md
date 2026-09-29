# Build Report — fork dsh-agent-teams against DSH host 0.2.0-rc.2

**Task:** task-3 — Install deps and build the fork against host 0.2.0-rc.2
**Owner:** builder (teammate)
**Repo:** `C:\Sophia\dsh-agent-teams` (branch `main`, HEAD `af32325`)
**Date of run:** this session

## VERDICT: BUILD OK

Install, typecheck and build all completed with exit status 0 against the real
`@deepseek-ai/*` **0.2.0-rc.2** packages. The installed
`node_modules/@deepseek-ai/dsh-agent` resolves to **0.2.0-rc.2** (proven below).
No source file under `src/**` required a change.

---

## 1. Toolchain used

| Item | Value |
|---|---|
| Node on PATH | `v24.19.0` |
| pnpm launcher | `C:\Users\Administrator\AppData\Local\Programs\DeepSeek Harness\resources\runtime\pnpm\bin\pnpm.cjs` |
| pnpm version (first call) | `11.7.0` (matches repo expectation) |
| pnpm version (later calls) | `11.22.0` (pnpm self-updated via the repo's `packageManager` pin after the lockfile was regenerated) |

pnpm is not on PATH; every command below was invoked through `node <pnpm.cjs>`.

---

## 2. Step 1 — dependency install

### Command (run from `C:\Sophia\dsh-agent-teams`)

```
node "C:\Users\Administrator\AppData\Local\Programs\DeepSeek Harness\resources\runtime\pnpm\bin\pnpm.cjs" install --no-frozen-lockfile
```

`--no-frozen-lockfile` was used deliberately: the shipped `pnpm-lock.yaml` still
pinned the old 0.1.7-rc.2 graph, so a frozen install fails by design.

### Exit status: `0`

### Log tail (relevant lines only)

```
? Verifying lockfile against supply-chain policies (730 entries)...
✓ Lockfile passes supply-chain policies (730 entries in 5.2s)
[WARN] 1 deprecated subdependencies found: node-domexception@1.0.0
Packages: +660
[WARN] Issues with peer dependencies found. Run "pnpm peers check" to list them.

devDependencies:
+ @deepseek-ai/cordis 4.0.4
+ @deepseek-ai/dsh 0.2.0-rc.2
+ @deepseek-ai/dsh-agent 0.2.0-rc.2
... (all 26 @deepseek-ai/dsh* devDependencies resolved at 0.2.0-rc.2) ...
+ typescript 5.9.3 (7.0.2 is available)

Done in 13.1s using pnpm v11.7.0
```

660 packages added; no install errors, no missing-peer hard failures.

---

## 3. Step 2 — typecheck

### Command

```
node "<pnpm.cjs>" typecheck
```

Expands to `tsc -p tsconfig.json --noEmit && tsc -p tsconfig.client.json --noEmit`.

### Exit status: `0`

### Log tail

```
$ tsc -p tsconfig.json --noEmit && tsc -p tsconfig.client.json --noEmit
```

Both `tsc` passes emitted **no diagnostics at all** (no TS error codes, no
warnings). This is the key compatibility result: the fork's source typechecks
cleanly against the 0.2.0-rc.2 host type surface.

---

## 4. Step 3 — build

### Command

```
node "<pnpm.cjs>" build
```

Expands to
`node scripts/clean-build.mjs && tsc -p tsconfig.json && tsc -p tsconfig.client.json && tsdown && node scripts/git-artifacts.mjs --write`.

### Exit status: `0`

### Log tail

```
ℹ tsdown v0.22.2 powered by rolldown v1.1.5
ℹ config file: C:\Sophia\dsh-agent-teams\tsdown.config.ts
ℹ [@nanmicoder/dsh-agent-teams/client] entry: lib/client/index.js
ℹ [@nanmicoder/dsh-agent-teams/client] target: node22.19.0
 WARN  We recommend using the ESM format instead of CommonJS.
ℹ [@nanmicoder/dsh-agent-teams/client] lib\client.js      229.09 kB │ gzip: 50.14 kB
ℹ [@nanmicoder/dsh-agent-teams/client] lib\client.js.map  262.93 kB │ gzip: 60.41 kB
ℹ [@nanmicoder/dsh-agent-teams/client] 2 files, total: 492.02 kB
✔ [@nanmicoder/dsh-agent-teams/client] Build complete in 115ms
```

The only output is one advisory warning (tsdown suggesting ESM over CommonJS).
It is **not** an error and does not affect the verdict.

### Artifacts produced (`lib/`)

Entry points declared in `package.json` `exports` all exist:

- `lib/index.js` (28,149 B) and `lib/types/index.d.ts` ✔
- `lib/client.js` (229,091 B), `lib/client.js.map`, `lib/types/client/index.d.ts` ✔

Plus `lib/tools.js` (143,809 B), `lib/quality-gates.js`, `lib/members.js`,
`lib/state.js` and the rest of the module set. `scripts/git-artifacts.mjs --write`
also ran successfully and regenerated `lib/git-artifact-stamp.json`:

```json
{
  "schema": 1,
  "source": "29bc3d1a93694865e5701d7ac61a2d6e70d37dbdf19cf56e82416f877609c03a",
  "output": "fbfcc40d4c97378b3c16bb0a070926dc9a73600d45a187ab05bf1b0e25410ed0"
}
```

---

## 5. Proof that the installed `dsh-agent` is 0.2.0-rc.2

Read directly from the installed package metadata:

```
node_modules/@deepseek-ai/dsh-agent/package.json
  name    = @deepseek-ai/dsh-agent
  version = 0.2.0-rc.2
```

Full picture of the installed `@deepseek-ai/*` scope (27 packages):

| Version | Packages | Note |
|---|---|---|
| `0.2.0-rc.2` | 25 | every `@deepseek-ai/dsh*` package |
| `3.18.4` | 1 | `@deepseek-ai/schemastery` — independently versioned |
| `4.0.4` | 1 | `@deepseek-ai/cordis` — independently versioned |

### Lockfile corroboration

`pnpm-lock.yaml` (669,028 B) was fully regenerated:

- occurrences of `0.2.0-rc.2`: **2938**
- occurrences of the old `0.1.7-rc.2`: **0**

---

## 6. Side effects on files I do not own — reported for the Lead

The install step caused pnpm to **rewrite `pnpm-workspace.yaml`** on its own
initiative. Two effects:

1. pnpm re-serialized the whole file into **single-line JSON flow style**
   (1 line, was 288 lines). Comments and multi-line YAML structure are gone.
   *All 273 `overrides` entries are still present and still pinned to
   `0.2.0-rc.2`* — nothing was lost functionally, but the file is no longer
   human-readable and will produce a large, noisy git diff
   (`1 insertion(+), 288 deletions(-)` against HEAD).
2. pnpm appended **278 `minimumReleaseAgeExclude` entries** (all
   `@deepseek-ai/dsh-*@0.2.0-rc.2`), with the log line:
   `Added 278 entries to minimumReleaseAgeExclude in pnpm-workspace.yaml`.

I did not edit this file by hand; I am reporting it because the Lead authored its
contents. If the Lead wants readable YAML, the file can be re-serialized — but
note pnpm may rewrite it again on the next install.

## 7. Warnings carried forward (non-blocking)

- `[WARN] 1 deprecated subdependencies found: node-domexception@1.0.0`
- `[WARN] Issues with peer dependencies found. Run "pnpm peers check" to list them.`
- tsdown advisory: "We recommend using the ESM format instead of CommonJS."

None of these block the build or the typecheck. The peer-dependency warning was
not investigated further and is worth a look if the Lead cares about a clean
`pnpm peers check` output.

---

## 8. Honesty notes / scope limits

- I did **not** run `pnpm verify` (owned by another teammate), so this report
  attests to install + typecheck + build only — not to the runtime test suite.
- I did **not** modify any file under `src/**`; no source change was needed.
- The build proves *compilation* against 0.2.0-rc.2, not *runtime behaviour*
  against the host. Runtime/lifecycle compatibility is the verification
  teammate's concern.
- Build outputs (`lib/`, `pnpm-lock.yaml`, `node_modules/`) were regenerated by
  these commands; nothing was hand-edited to make the build pass.

## 9. Final answer

**BUILD OK** — no failing error text to report. Install (exit 0), typecheck
(exit 0, zero diagnostics), build (exit 0) all succeed against
`@deepseek-ai/dsh-agent@0.2.0-rc.2`, with the lockfile fully migrated off
`0.1.7-rc.2`.
