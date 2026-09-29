# Gate acceptance proof — adapted package ACCEPTED on dsh 0.2.0-rc.2, published 0.1.21 REFUSED

**Task:** `task-5` — Prove host gate acceptance on 0.2.0-rc.2 using the host's own code path
**Author:** teammate `gate-proof`
**Snapshot taken:** 2026-09-29

> **Headline:** the host's own `evaluatePluginCompatibility()` — the exact function the installer
> calls, extracted verbatim from the user's installed `app.asar` and executed here — returns
> **no issue** (ACCEPTED) for our adapted artifact and **an issue** (REFUSED) for the published
> `@nanmicoder/dsh-agent-teams@0.1.21`. Both outcomes were observed, not inferred. **No
> exact-version exemption was active in either run, and none was ever created.**

---

## 1. What was executed

Not a re-implementation and not a simulation of the algorithm — the host's real bundled module,
relocated out of the `app.asar` container so Node's normal ESM/CJS resolution can load it.

| Fact | Value |
| --- | --- |
| Container | `C:\Users\Administrator\AppData\Local\Programs\DeepSeek Harness\resources\app.asar` |
| Packed path of the module under test | `/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js` |
| Relocated to | `C:\Sophia\_gate\host\dsh\node_modules\@deepseek-ai\dsh-app-boot\lib\index.js` |
| Real imports satisfied from | `C:\Sophia\_gate\host\dsh\node_modules\` (10,913 packed files relocated, preserving layout) |
| Exports used | `evaluatePluginCompatibility`, `pluginCompatibilityWarning`, `getDshRuntimeVersion` |

Line numbers in the relocated copy match the audit report
([host-gate-and-references.md](host-gate-and-references.md)) exactly — this is the same file:

| Line | Symbol |
| --- | --- |
| `dsh-app-boot/lib/index.js:271` | `function getDshRuntimeVersion()` |
| `dsh-app-boot/lib/index.js:286` | `function evaluatePluginCompatibility(manifest, exemptions = {}, runtimeVersion = getDshRuntimeVersion())` |
| `dsh-app-boot/lib/index.js:300` | `if (requirement.trim() === "" \|\| !semver.satisfies(runtimeVersion, requirement, { includePrerelease: true })) peers[name] = range;` |
| `dsh-app-boot/lib/index.js:320` | `function pluginCompatibilityWarning(issue)` |

### Why this is the right call site

The user's refusal came from the install pre-flight in
`/dsh/node_modules/@deepseek-ai/dsh-plugin-manager/lib/index.js:500-514`. That block's decisive line
is **`:506`** — read from the relocated copy:

```js
	const preflight = [];
	const exemptions = readProfileVersionExemptions(dir);
	const registryFlags = args.filter((argument) => argument.startsWith("--registry="));
	for (const raw of namedSpecs(args)) try {
		const manifest = await namedSpecManifest(dir, anchorPathSpec(raw, context.cwd), options, environment, registryFlags);
		if (manifest === void 0) continue;
		const issue = evaluatePluginCompatibility(manifest, exemptions);
		if (issue !== void 0 && !issue.exempted) {
			preflight.push(pluginCompatibilityWarning(issue));
			incompatible.push(incompatiblePlugin(issue));
		}
	} catch (error) {
		continue;
	}
	if (preflight.length > 0) return rejected(preflight, "nothing was installed");
```

`evaluatePluginCompatibility(manifest, exemptions)` is therefore the *whole* of the decision. The
harness feeds it the real manifests and reports what it returns plus the verbatim
`pluginCompatibilityWarning()` text, which is what `:508` would push into `preflight`.

**Limit, stated plainly:** the surrounding `install` routine is not importable in isolation — the
module's runtime exports are only `PluginManager`, `parseInstallSpec`, `classifyInstallFailure`,
`InvalidInstallSpecError` and `default`, and `namedSpecManifest` (`:317`) is a module-private
function. So the pre-flight was **not** driven end-to-end; what was driven is the exact function it
calls, with the exact argument shape (`exemptions` = an empty object, as
`readProfileVersionExemptions()` returns on a profile that has never granted one).

---

## 2. Exact invocations

Everything below was run from `C:\Sophia\_gate`. No `pnpm install` was run in
`C:\Sophia\dsh-agent-teams`; no file outside `_gate` and this document was written.

**Step 1 — relocate the host's own module (once):**

```
node C:\Sophia\_gate\extract.cjs --tree "/dsh/node_modules"
# -> extracted 10913 files, 109798677 bytes from /dsh/node_modules -> C:/Sophia/_gate/host (skipped 1497 unpacked)
```
(1497 skipped entries are unpacked native binaries with no asar offset; they are not needed by the
compatibility path.)

**Step 2 — fetch the published artifact under test (the version the user tried):**

```
cd C:\Sophia\_gate
npm pack @nanmicoder/dsh-agent-teams@0.1.21
# -> nanmicoder-dsh-agent-teams-0.1.21.tgz
tar -xzf nanmicoder-dsh-agent-teams-0.1.21.tgz -C old-0.1.21
```

**Step 3 — drive the gate:**

```
node C:\Sophia\_gate\gate-proof.mjs "C:/Sophia/_gate/our-tarball/package/package.json" "C:/Sophia/_gate/old-0.1.21/package/package.json"
```

### Which declaration source was fed to the gate

**The packed tarball built by task-4: `C:\Sophia\_verify\agentteams-0.1.22-rc.2.tgz`**, unpacked to
`C:\Sophia\_gate\our-tarball\package\package.json`. This is the published shape of our artifact, so
the evidence is about what we would actually ship, not about the working tree.

The repository's own `C:\Sophia\dsh-agent-teams\package.json` was run first (same harness, same
result: ACCEPTED) and carries the identical peer range. Both are recorded below.

| Artifact fed to the gate | Path | SHA-256 of the manifest | Verdict |
| --- | --- | --- | --- |
| Our build, **from the packed tarball** | `C:\Sophia\_gate\our-tarball\package\package.json` | `C961587DFE0504CE9F0902EFB4D47E3B84793D5BC01A8DCFAE6F2D828A9F9845` | **ACCEPTED** |
| Our build, repo working tree | `C:\Sophia\dsh-agent-teams\package.json` | `73F45766E570D01D996618241527E9C863E3AFC5046197C31808743B5D24CFD6` | **ACCEPTED** |
| Published old version | `C:\Sophia\_gate\old-0.1.21\package\package.json` | `17F9F51BD1FCD2AE9E1399F79FD16063E9B3B1166E5E892D26F4957345EFDDBC` | **REFUSED** |

Tarball digests, for traceability:

- `C:\Sophia\_verify\agentteams-0.1.22-rc.2.tgz` — `A1B33FB739B226E60CECE8B5BB81AF69C8DA323A110AB533744C4C00677B3A0C`
- `C:\Sophia\_gate\nanmicoder-dsh-agent-teams-0.1.21.tgz` — `C4D4A2658B7F6A88FE0EA5DAC30DEE9E9204BCDEA73B50F262ADBBB86988DA3D`

> **Concurrent-repack note.** `agentteams-0.1.22-rc.2.tgz` was rebuilt by teammate `builder`
> while this proof was being written (its digest moved from `065BE4E3…` to `A1B33FB7…`). This was
> checked rather than assumed: unpacking the current tarball and comparing it file-by-file against
> the one originally tested shows **113 files compared, 1 differing** — only
> `lib/git-artifact-stamp.json`, which the gate never reads. The manifest that decides the verdict,
> `package.json`, is **byte-identical** (`C961587DFE0504CE9F0902EFB4D47E3B84793D5BC01A8DCFAE6F2D828A9F9845`)
> in both. The recorded ACCEPTED verdict therefore applies to both revisions of the tarball.

Full transcript: `C:\Sophia\_gate\gate-proof-output.txt` (59 lines, 10,136 bytes).

---

## 3. Verbatim output — our artifact is ACCEPTED

```
HOST MODULE UNDER TEST : C:/Sophia/_gate/host/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js
  extracted from       : C:\Users\Administrator\AppData\Local\Programs\DeepSeek Harness\resources\app.asar
  packed path          : /dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js
  getDshRuntimeVersion(): 0.2.0-rc.2
  running node         : v24.19.0

==============================================================================
CASE: ADAPTED PACKAGE (our fork, prerelease-adapted peer ranges)
manifest fed to the gate : C:/Sophia/_gate/our-tarball/package/package.json
  name                   : @nanmicoder/dsh-agent-teams
  version                : 0.1.22-rc.2
  DSH peers checked      : 21
  distinct peer ranges   : 1
  range (all identical)  : 0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2 || 0.2.0 || 0.2.0-rc.2
------------------------------------------------------------------------------
RESULT: ACCEPTED — evaluatePluginCompatibility() returned undefined,
        therefore the install pre-flight pushes nothing and never warns.
        pluginCompatibilityWarning() output: (none — there is no issue object)
        exemption active in this run: no
        runtimeVersion used by the gate: 0.2.0-rc.2
```

All 21 DSH peers are checked (the same 21 the audit enumerated) and all 21 clear. The literal
`0.2.0-rc.2` alternative appended to the pre-existing `|| 0.2.0` token is what makes the difference:
`semver.satisfies('0.2.0-rc.2', '0.2.0', { includePrerelease: true })` is `false`, because
`0.2.0-rc.2` sorts strictly below `0.2.0` and the bare range `0.2.0` is the comparator `=0.2.0`.

---

## 4. Verbatim output — published 0.1.21 is REFUSED

```
==============================================================================
CASE: PUBLISHED @nanmicoder/dsh-agent-teams@0.1.21 (the version the user tried)
manifest fed to the gate : C:/Sophia/_gate/old-0.1.21/package/package.json
  name                   : @nanmicoder/dsh-agent-teams
  version                : 0.1.21
  DSH peers checked      : 21
  distinct peer ranges   : 1
  range (all identical)  : 0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2
------------------------------------------------------------------------------
RESULT: REFUSED — evaluatePluginCompatibility() returned an issue:
        {"name":"@nanmicoder/dsh-agent-teams","version":"0.1.21","runtimeVersion":"0.2.0-rc.2","peers":{"@deepseek-ai/dsh-agent":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-api-session-controller":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-connection":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-locale":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-store":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-chat":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-conversation":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-layout":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-model-selection":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-primitives":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-renderer":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-session":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-client-ui-slots":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-commands":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-llm":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-session":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-session-projection":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-subagent":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-system-prompt":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-tools":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2","@deepseek-ai/dsh-util-values":"0.1.7-rc.2 || 0.1.5-rc.3 || 0.1.5-rc.2 || 0.1.5-rc.1 || 0.1.2-rc.1 || 0.1.2-alpha.5 || 0.1.2-alpha.2"},"exempted":false}
```

The verbatim `pluginCompatibilityWarning()` text for that issue — the string the user saw, with the
full `peers` JSON elided here only for width, reproduced in
`C:\Sophia\_gate\gate-proof-output.txt` line for line:

```
Plugin @nanmicoder/dsh-agent-teams@0.1.21 is incompatible with dsh 0.2.0-rc.2: peerDependencies {…21 peers…}. Running it may cause crashes or data loss. Update the plugin or install a plugin version compatible with this dsh runtime. To accept this risk explicitly, grant the exact-version exemption for @nanmicoder/dsh-agent-teams@0.1.21 on dsh 0.2.0-rc.2 with `dsh plugin allow-version` or the plugin manager, then retry the installation or restart dsh. Exact-version exemption: not active.
```

Note the tail: **`Exact-version exemption: not active.`** — produced by the host itself from
`issue.exempted === false`, matching the user's original report byte for byte.

---

## 5. The test discriminates

A check that cannot tell the old failure from the new pass is not evidence, so the harness compares
the two outcomes in the same process, against the same host module, at the same runtime version:

```
DISCRIMINATION CHECK
  adapted package    : evaluatePluginCompatibility() returned undefined  => ACCEPTED
  published 0.1.21   : evaluatePluginCompatibility() returned an issue  => REFUSED
  gate distinguishes : true
```

What separates them is one token in the peer range. Ours ends
`… || 0.1.2-alpha.2 || 0.2.0 || 0.2.0-rc.2`; 0.1.21's ends `… || 0.1.2-alpha.2` with no
prerelease-of-0.2.0 alternative at all (its last `|| 0.2.0` was never in the published manifest —
the tarball was fetched fresh from npm for this test and inspected directly).

---

## 6. No exemption was active

Three independent confirmations, all from the host code rather than from my own bookkeeping:

1. **The harness passes `{}` as `exemptions` on every call.** With `exemptions = {}`,
   `evaluatePluginCompatibility` computes
   `exempted: (Object.hasOwn({}, key) ? … : void 0)?.includes(runtimeVersion) === true` → `false`.
2. **The host reported it.** For 0.1.21 the returned issue carries `"exempted":false`, and the
   rendered warning ends `Exact-version exemption: not active.`
3. **The accepted case raised no issue at all.** The adapted package never reached the exemption
   check: `evaluatePluginCompatibility` returned `undefined` at `:287`
   (`if (Object.keys(peers).length === 0) return void 0;`) because all 21 peers satisfied the
   runtime. A grant cannot be what produced a result that does not consult grants.

`dsh plugin allow-version` was **never invoked**, in any profile. No `compatibility.json` was
written anywhere. The adapted declaration passes on its own merits.

---

## 7. Isolation and blast radius

- The user's live profile `C:\Users\Administrator\.dsh\profiles\desktop` was **not** read for
  exemptions, **not** written, and **not** installed into. The desktop app was not touched.
- All scratch state lives under `C:\Sophia\_gate\`: `host\` (relocated host modules),
  `extract.cjs`, `gate-proof.mjs`, `gate-proof-output.txt`, `old-0.1.21\`, `our-tarball\`,
  `nanmicoder-dsh-agent-teams-0.1.21.tgz`, `probe-import.mjs`.
- `pnpm install` was never run in `C:\Sophia\dsh-agent-teams`; the packed tarball produced by
  task-4 was consumed read-only.
- The only file written inside the repository is this document.

## 8. Reproducing

```
node C:\Sophia\_gate\extract.cjs --tree "/dsh/node_modules"
cd C:\Sophia\_gate
npm pack @nanmicoder/dsh-agent-teams@0.1.21
tar -xzf nanmicoder-dsh-agent-teams-0.1.21.tgz -C old-0.1.21
node C:\Sophia\_gate\gate-proof.mjs "C:/Sophia/_gate/our-tarball/package/package.json" "C:/Sophia/_gate/old-0.1.21/package/package.json"
```

Expected: `RESULT: ACCEPTED` for the first manifest, `RESULT: REFUSED` for the second, and
`gate distinguishes : true`.

## 9. What this does and does not establish

**Established:** the compatibility gate — the sole reason the user's install was refused, per
`dsh-plugin-manager/lib/index.js:500-514` — now returns no issue for our adapted artifact and still
returns an issue for the published 0.1.21. The refusal message the adapted package would produce is
gone because there is no longer an issue to render.

**Not established by this document:** that the adapted plugin *runs* correctly on 0.2.0-rc.2 beyond
the version gate, and that `pnpm` completes an actual install of it into a real profile. Those are
outside `task-5`; the first belongs to the runtime-verification work, and the second would require
touching a profile, which this task forbids.
