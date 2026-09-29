# Host gate semantics + repo version-reference inventory

**Task:** `task-1` — Host gate semantics + repo version-reference inventory (read-only)
**Author:** teammate `gate-audit`
**Snapshot taken:** 2026-09-29T19:25:46+08:00

> **This repository was being mutated concurrently while this audit ran.** See
> [Part 3](#part-3--concurrent-mutation-warning). Every line number below is stated
> against the snapshot above; re-verify before acting on a specific line number.

---

## 0. Evidence base and method

| Fact | Value | Source |
| --- | --- | --- |
| Host app dir | `C:\Users\Administrator\AppData\Local\Programs\DeepSeek Harness` | filesystem |
| Packed code | `resources/app.asar` (paths inside begin `/dsh/...`) | filesystem |
| `resources/runtime/primary-runtime/runtime.json` → `desktopVersion` | `0.2.0-rc.2` | read directly |
| `runtime.json` → `node` / `pnpm` | `24.21.0` / `11.7.0` | read directly |
| Local `node` | `C:\Program Files\nodejs\node.exe`, v24.19.0 | `process.execPath` |
| semver used for the empirical battery | 7.8.4 (npm-bundled at `C:\Program Files\nodejs\node_modules\npm\node_modules\semver`) | required at runtime |
| semver version pinned by the host | `"semver": "^7.8.5"` (dependency of `dsh-app-boot`) | `dsh/node_modules/@deepseek-ai/dsh-app-boot/package.json` |

**Correction to the task brief:** `resources/runtime/runtime.json` does **not** exist. The
real file is `resources/runtime/primary-runtime/runtime.json`. Its `desktopVersion` agrees
with the value the gate actually reads (see §1.1), but it is **not** the gate's source.

**Method note — the bundled helper `C:\Sophia\_asar_get.cjs --grep <s>` matches file *paths*
only, not file *contents*.** A `--grep "compatible with this dsh runtime"` therefore returns
nothing and must not be read as "the string is absent". Content searching was done with an
inline `node -e` walker over the asar header. Two traps in that walker, both hit during the
audit: asar `offset` values are **strings** (so `Number.isFinite(node.offset)` silently
skips every packed file — it must be `Number.isFinite(parseInt(String(node.offset), 10))`),
and unpacked entries have no usable offset at all.

---

# Part 1 — Host gate semantics

## 1.1 Where the runtime version comes from

`/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js:271-275` — verbatim:

```js
function getDshRuntimeVersion() {
	const filename = fileURLToPath(new URL("../package.json", import.meta.url));
	const manifest = objectOf$1(JSON.parse(fs.readFileSync(filename, "utf8")), "app-boot package.json");
	return runtimeVersionOf(Object.hasOwn(manifest, "version") ? manifest.version : void 0);
}
```

The runtime version is **the version of `@deepseek-ai/dsh-app-boot` itself**, resolved relative
to `lib/index.js` → `/dsh/node_modules/@deepseek-ai/dsh-app-boot/package.json`. Read directly
from the asar, that manifest declares:

```json
"name": "@deepseek-ai/dsh-app-boot",
"version": "0.2.0-rc.2",
```

`runtimeVersionOf` (`:257-260`) only validates that it is a semver — it does not normalise it.
Prerelease spelling is preserved exactly (the doc comment at `:268` says "preserving its exact
spelling"), which is precisely why the prerelease suffix matters.

## 1.2 The gate function — verbatim

`/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js:286-313`:

```js
function evaluatePluginCompatibility(manifest, exemptions = {}, runtimeVersion = getDshRuntimeVersion()) {
	runtimeVersionOf(runtimeVersion);
	const fields = objectOf$1(manifest, "Plugin manifest");
	if (!Object.hasOwn(fields, "peerDependencies")) return void 0;
	const dependencies = objectOf$1(fields.peerDependencies, "Plugin manifest peerDependencies");
	const peers = {};
	for (const [name, range] of Object.entries(dependencies)) {
		if (typeof range !== "string") throw new Error(`Plugin manifest peerDependencies[${JSON.stringify(name)}] must be a string`);
		if (name !== "@deepseek-ai/dsh" && !name.startsWith("@deepseek-ai/dsh-")) continue;
		const requirement = [
			"workspace:^",
			"workspace:~",
			"workspace:*"
		].includes(range) ? runtimeVersion : range;
		if (requirement.trim() === "" || !semver.satisfies(runtimeVersion, requirement, { includePrerelease: true })) peers[name] = range;
	}
	if (Object.keys(peers).length === 0) return void 0;
	const name = identityField(fields, "name");
	const version = identityField(fields, "version");
	const key = `${name}@${version}`;
	return {
		name,
		version,
		runtimeVersion,
		peers,
		exempted: (Object.hasOwn(exemptions, key) ? exemptions[key] : void 0)?.includes(runtimeVersion) === true
	};
}
```

### The five decisive facts in that function

1. **The comparison is `semver.satisfies(runtimeVersion, requirement, { includePrerelease: true })`** (`:300`).
   Not exact string equality, not a custom comparison.
2. **`includePrerelease: true` is set.** This widens prerelease matching, but it still does *not*
   make a prerelease satisfy the equivalent release range (verified empirically in §1.7).
3. **A "DSH peer" is exactly `@deepseek-ai/dsh` or any name starting with `@deepseek-ai/dsh-`** (`:294`).
   `@deepseek-ai/cordis`, `@deepseek-ai/schemastery` and `react` are **not** checked.
4. **Only `peerDependencies` is read.** `(:289)` returns `undefined` — no gate at all — when the
   manifest has no `peerDependencies` key. `devDependencies` is never consulted.
5. **`peerDependenciesMeta` / `optional: true` is completely ignored.** The loop at `:292-301`
   never reads it. Doc comment `:277` confirms: *"Check every @deepseek-ai/dsh or
   @deepseek-ai/dsh-* peer against the runtime."* There is no required-vs-optional distinction.
   This fork marks all 21 DSH peers `optional: true`, and that buys **no** exemption.

`workspace:^`, `workspace:~` and `workspace:*` are rewritten to `runtimeVersion`, i.e. a peer
declared that way compares the runtime against itself and always passes. See §1.10.

## 1.3 The exact error message — verbatim

`/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js:320-323`:

```js
function pluginCompatibilityWarning(issue) {
	const key = `${issue.name}@${issue.version}`;
	return `Plugin ${key} is incompatible with dsh ${issue.runtimeVersion}: peerDependencies ${JSON.stringify(issue.peers)}. Running it may cause crashes or data loss. Update the plugin or install a plugin version compatible with this dsh runtime. To accept this risk explicitly, grant the exact-version exemption for ${key} on dsh ${issue.runtimeVersion} with \`dsh plugin allow-version\` or the plugin manager, then retry the installation or restart dsh. Exact-version exemption: ${issue.exempted ? "active" : "not active"}.`;
}
```

This is byte-for-byte the message the user saw, including the trailing
`Exact-version exemption: not active.`

The user's message ended with `dsh: nothing was installed.` That tail is produced at the
install path, `/dsh/node_modules/@deepseek-ai/dsh-plugin-manager/lib/index.js:500-514` — verbatim:

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

So the plugin is refused **before pnpm is invoked at all** — this is a pre-flight manifest check,
not an install failure. Note `catch (error) { continue; }` at `:511-513`: a manifest that cannot
be fetched is silently skipped rather than failing.

## 1.4 Every call site of the gate

Content search across all `.js`/`.cjs`/`.mjs` in the asar:

| Location | Context |
| --- | --- |
| `.../dsh-app-boot/lib/index.js:930-931` | boot-time guard — `throw new Error(pluginCompatibilityWarning(issue))` |
| `.../dsh-app-boot/lib/index.js:2079-2080` | non-throwing reporter |
| `.../dsh-plugin-manager/lib/index.js:506-509` | **install pre-flight (the user's failure)** |
| `.../dsh-plugin-manager/lib/index.js:628-630` | candidate scan |
| `.../dsh-plugin-manager/lib/index.js:1490`, `:1787`, `:1986` | inventory / profile listing |
| `.../dsh-plugin-manager/lib/types/index.js:304`, `:582`, `:763` | same, types build |
| `.../dsh-plugin-manager/lib/types/operations.js:307-309`, `:458-460` | same, operations build |

The gate therefore fires on **install, on boot, and on listing**. Passing it once is not a
one-time event.

## 1.5 Which peers are actually checked in this repo

`package.json:109-129` declares 21 `@deepseek-ai/dsh-*` peers; all 21 are checked, and all 21
currently carry an identical range string. `@deepseek-ai/dsh` itself appears only under
`devDependencies` (`package.json:209`) and is therefore **not** checked.

Not checked (name does not match the prefix): `@deepseek-ai/cordis`, `@deepseek-ai/schemastery`,
`react`.

## 1.6 Empirical semver battery

semver 7.8.4, `runtimeVersion = "0.2.0-rc.2"`:

| requirement | `satisfies(..., {includePrerelease:true})` | plain `satisfies` |
| --- | --- | --- |
| `"0.2.0"` | **false** | false |
| `"^0.2.0"` | **false** | false |
| `"~0.2.0"` | **false** | false |
| `"0.2.0-rc.2"` | **true** | true |
| `"^0.2.0-rc.2"` | **true** | true |
| `"~0.2.0-rc.2"` | **true** | true |
| `">=0.2.0-rc.2"` | **true** | true |
| `"^0.2.0-rc.1"` | **true** | true |
| `"0.2.0-rc.1"` | false | false |
| `">=0.1.7-rc.2"` | **true** | false |
| `"^0.1.7-rc.2"` | false | false |
| `"0.1.7-rc.2"` | false | false |
| `"^0.1.5-rc.3"` | false | false |
| `"^0.1.2-rc.1"` | false | false |
| `">=0.2.0-0"` | **true** | true |
| `"*"` | true | false |

Applied to the two real range strings:

| range | result |
| --- | --- |
| published `0.1.21` style (…`\|\| 0.2.0`) | **false → gate refuses** |
| working tree style (…`\|\| 0.2.0 \|\| 0.2.0-rc.2`) | **true → gate passes** |

**Confirmed: `semver.satisfies('0.2.0-rc.2', '0.2.0', { includePrerelease: true }) === false`.**
`includePrerelease` does not rescue it, because `0.2.0-rc.2` sorts strictly *below* `0.2.0` and
the bare range `0.2.0` is the comparator `=0.2.0`.

**This is exactly why `@nanmicoder/dsh-agent-teams@0.1.21` was refused on this host:** its peer
ranges ended in `|| 0.2.0` (the plain release) with no `0.2.0-rc.2` alternative. The gate did not
reject the plugin for being "old" — it rejected an exact-token mismatch.

## 1.7 Conclusion — the minimal declaration change

**(a) Minimal change that satisfies the HOST GATE alone:** append ` || 0.2.0-rc.2` to each of
the 21 DSH peer ranges at `package.json:109-129`. Nothing else in the manifest affects the gate.
`devDependencies`, `peerDependenciesMeta`, `engines` and the `dsh` field are all invisible to it.

**(b) Minimal *complete* change (host gate + this repo's own verifier):** (a) **plus** declare
the same version in the compatibility policy — add `{"version": "0.2.0-rc.2", "track":
"recommended"}` to `compatibility.json` `supportedHosts` and repoint `recommendedHost` to it.
This coupling is *not optional*; it is enforced by `scripts/compatibility.mjs` and explained in
§1.11. Both halves are already present in the working tree (§3).

A change of the **range operator alone cannot work**: rewriting `0.2.0` → `^0.2.0` still fails
(§1.6), because `^0.2.0` does not reach a prerelease of `0.2.0` either.

## 1.8 The exemption path (`dsh plugin allow-version`)

Implementation: `/dsh/node_modules/@deepseek-ai/dsh/lib/plugin-BGnVfe_D.js:14-56`.

- Parses only `allow-version`, `revoke-version`, `version-exemptions`; anything else is passed to pnpm (`:16`).
- Requires an **exact** `package@version` and an **exact** `--dsh-version`, plus `--accept-risk`
  for `allow-version` (`:22`, `:30`).
- Writes into the **profile's `compatibility.json`** via `setProfileVersionExemption(dir, packageVersion, runtimeVersion, command === "allow-version", acceptRisk)` (`:47`), under a file lock on the profile `package.json` (`:39`).
- `:36` prints: `dsh: warning: allowing incompatible plugin versions can break the application or corrupt data. Approval applies only to the exact package and DSH versions.`
- `:95` is the reminder printed after a failed install: `dsh: to accept the risk, run: dsh plugin --profile ${profile} allow-version ${name}@${version} --dsh-version ${runtimeVersion} --accept-risk`

Exemption validation is in `.../dsh-app-boot/lib/index.js:328-351`: `PROFILE_COMPATIBILITY_FILENAME = "compatibility.json"` (`:328`), keys must be canonical exact `name@version` (`:339-342`, `:350`), values must be lists of exact versions (`:332-334`).

**Difference between "properly adapted" and "bypassed":** an exemption is a runtime grant stored in
the *consumer's profile*, keyed to one exact plugin version and one exact runtime version. It does
not change the plugin, does not ship, and evaporates on any version change on either side. The
declaration change in §1.7 is what makes the plugin self-consistent and distributable.

Note that this profile-level `compatibility.json` is a **different file** from the repository's
shipped `C:\Sophia\dsh-agent-teams\compatibility.json`; they share only a filename.

## 1.9 Non-viable shortcut, recorded so it is not rediscovered

Declaring a peer as `workspace:^` / `workspace:~` / `workspace:*` makes the requirement equal
`runtimeVersion`, so the gate compares the runtime to itself and **always passes** (`:295-299`).
It is a one-token-per-peer bypass of the version gate. **Do not use it here:** the repo's own
verifier splits the peer range on `||` and requires one entry per declared host
(`scripts/compatibility.mjs:97-100`), so a bare `workspace:^` yields a single entry and fails the
length check. It would also silently drop all real compatibility information from the published
manifest.

---

# Part 2 — Repo reference inventory

Directory convention: paths are relative to `C:\Sophia\dsh-agent-teams`.

## Class A — version-gate surface (must change to adapt)

### A1. `package.json`

| Lines | Content | Baseline (HEAD `af32325`) | Working tree |
| --- | --- | --- | --- |
| `3` | `"version"` | `0.1.22-rc.1` | `0.1.22-rc.2` |
| `109-129` | 21 × `@deepseek-ai/dsh-*` in `peerDependencies` | …`\|\| 0.1.2-alpha.2 \|\| 0.2.0` | …`\|\| 0.1.2-alpha.2 \|\| 0.2.0 \|\| 0.2.0-rc.2` |
| `137-197` | 21 × `peerDependenciesMeta` entries | unchanged (no version strings) | unchanged |
| `209-233` | 25 × `@deepseek-ai/dsh*` in `devDependencies` | all `0.1.7-rc.2` | all `0.2.0-rc.2` |

Exact peer line numbers: `109` dsh-agent · `110` api-session-controller · `111` client-connection ·
`112` client-locale · `113` client-store · `114` client-ui-chat · `115` client-ui-conversation ·
`116` client-ui-layout · `117` client-ui-model-selection · `118` client-ui-primitives ·
`119` client-ui-renderer · `120` client-ui-session · `121` client-ui-slots · `122` commands ·
`123` llm · `124` session · `125` session-projection · `126` subagent · `127` system-prompt ·
`128` tools · `129` util-values.

`devDependencies` span `209-233` (25 entries: dsh, dsh-agent, dsh-api-session-controller,
dsh-client-connection, dsh-client-locale, dsh-client-store, dsh-client-ui-chat,
dsh-client-ui-conversation, dsh-client-ui-layout, dsh-client-ui-model-selection,
dsh-client-ui-primitives, dsh-client-ui-renderer, dsh-client-ui-session,
dsh-client-ui-sidebar-right, dsh-client-ui-slots, dsh-commands, dsh-llm, dsh-llm-retry,
dsh-session, dsh-session-projection, dsh-subagent, dsh-system-prompt, dsh-tools,
dsh-util-values, dsh-workspace).

> `@deepseek-ai/dsh-client-ui-sidebar-right` and `@deepseek-ai/dsh-llm-retry` and
> `@deepseek-ai/dsh-workspace` appear in `devDependencies` but **not** in `peerDependencies`.
> That is fine for the host gate. It matters for `scripts/compatibility.mjs:83-86`, which
> requires a workspace override for every DSH `devDependency` — all three are present.

### A2. `compatibility.json` (47 lines)

| Line | Field | Baseline | Working tree |
| --- | --- | --- | --- |
| `3` | `recommendedHost` | `0.1.7-rc.2` | `0.2.0-rc.2` |
| `7` | `supportedHosts[0]` | `0.1.7-rc.2` / `recommended` | `0.2.0-rc.2` / `recommended` |
| `11` | `supportedHosts[1]` | `0.1.5-rc.3` | `0.1.7-rc.2` / `legacy` (inserted) |
| `15` | `supportedHosts[2]` | `0.1.5-rc.2` | `0.1.5-rc.3` |
| `19` | `supportedHosts[3]` | `0.1.5-rc.1` | `0.1.5-rc.2` |
| `23` | `supportedHosts[4]` | `0.1.2-rc.1` | `0.1.5-rc.1` |
| `27` | `supportedHosts[5]` | `0.1.2-alpha.5` | `0.1.2-rc.1` |
| `31` | `supportedHosts[6]` | `0.1.2-alpha.2` | `0.1.2-alpha.5` |
| `35` | `supportedHosts[7]` | — | `0.1.2-alpha.2` |
| `41` | `sourceCandidates[0].version` | `0.2.0` | `0.2.0` (unchanged) |
| `4` | `previewTag` | `next` | `next` (must stay `next` — `scripts/compatibility.mjs:30`) |

**`sourceCandidates[0].version = "0.2.0"` must NOT be changed to `0.2.0-rc.2`.** It is a distinct
concept (see §1.11) and the peer range is required to enumerate it separately.

### A3. `pnpm-workspace.yaml` (288 lines)

| Lines | Content | Baseline | Working tree |
| --- | --- | --- | --- |
| `6-282` | `overrides` block, **274** `@deepseek-ai/dsh*` keys | all `0.1.7-rc.2` | all `0.2.0-rc.2` |

`packages` (`:1-5`), `autoInstallPeers`, `allowBuilds` are unaffected.

### A4. Shipped end-user docs carrying host versions (class A by effect)

These are listed in `package.json` `"files"` and are therefore published; `scripts/readme-version.mjs`
checks them. At snapshot time they had **already been updated** by a concurrent teammate.

`README.md`: lines `33`, `54`, `58`, `59`, `60`, `61`, `62`, `63`, `80`
`README_ZH.md`: lines `33`, `53`, `57`, `58`, `59`, `60`, `61`, `62`, `79`

Both READMEs also link `./docs/harness-0.2.0-rc.2-adaptation/README.md`, which **did not exist**
at snapshot time (see Part 4).

### A5. CI workflows — no host-version pins

`.github/workflows/verify.yml` and `.github/workflows/publish.yml` contain no host-version
literal. Their only version pins are tool versions (`pnpm@10.33.0` at `verify.yml:42,93,120` and
`publish.yml:56`; `npm@11.19.0`). The host matrix is derived at runtime from
`node scripts/compatibility.mjs --github-output`, so CI follows `compatibility.json` automatically.

## Class B — generated / shipped code (regenerated by build; no manual edit)

**None. Zero occurrences.**

- `lib/**` — searched for `0.1.7-rc.2`, `0.2.0-rc.2`, `0.1.2-alpha.2`, `0.1.5-rc.3`: **no matches**.
- `src/**` — same patterns: **no matches**.

The host version reaches shipped code only through the manifest-derived policy files, never as a
literal in `lib`/`src`. This is a genuinely useful property: after the class-A change, the only
required action is `pnpm build`.

## Class C — tests and scripts asserting expected host versions

### Verdict first

**With the class-A mutation as it stands in the working tree, NO class-C assertion fails.**
The suite is well designed in this respect: every host-version assertion either uses synthetic
fixture versions, or derives its expectation from `policy` / `policy.recommendedHost` rather than
hardcoding a literal. Each item below therefore carries an explicit "fails now?" verdict.

### C1. `scripts/compatibility.mjs` — the repo's own gate (not a test, but load-bearing)

| Line | Content | Fails now? |
| --- | --- | --- |
| `35` | host-version regex `/^\d+\.\d+\.\d+(?:-(?:alpha\|beta\|rc)\.\d+)?$/` | no — `0.2.0-rc.2` matches |
| `43` | `recommended[0].version !== input.recommendedHost` → throw | no — consistent |
| `44` | rejects alpha/beta as `recommendedHost` | no — `rc` is allowed |
| `49` | same regex for `sourceCandidates` | no |
| `62-64` | `declaredHostVersions()` = supportedHosts **+** sourceCandidates | no — returns 9 |
| `97-100` | **peer range must enumerate declared versions, with equal length** | no — 9 vs 9 |

### C2. `scripts/compatibility.test.mjs` (177 lines)

| Line | Assertion | Fails now? |
| --- | --- | --- |
| `13` | `declaredHostVersions().includes('0.2.0')` | no — `0.2.0` is still a source candidate |
| `14` | `!validatePackageCompatibility(pkg).includes('0.2.0')` | no — return value is supportedHosts only |
| `15-17` | CLI `--github-output` exits 0 and omits `0.2.0` | no |
| `18-24` | 5 malformed `sourceCandidates` must throw `/Source candidates/` — incl. `:20` `'^0.2.0'`, `:21` `'0.2.0\n'`, `:23` `version: policy.recommendedHost` | no — all still throw |
| `25-26` | strip ` \|\| 0.2.0` from the dsh-agent peer, then expect a throw | no — **but see the fragility note** |
| `32-39` | doctor classifies installed `@deepseek-ai/dsh` `0.2.0` as `source-preview`, `supportedHosts` excludes `0.2.0` | no |
| `47` | doctor OK when host version is `policy.recommendedHost` | no |
| `62-65` | `0.1.0-rc.8` must be reported unsupported | no |
| `69-71` | floating targets rejected | no |
| `73` | duplicate host rejected | no |
| `74` | `devDependencies['@deepseek-ai/dsh'] = '^0.1.2-rc.1'` rejected | no |
| `84-95` | cohort-mixing fixtures (synthetic) | no |
| `104-111` | profile duplicate-identity fixture (synthetic) | no |
| `117` | **`assert.doesNotThrow(() => validatePackageCompatibility(pkg))` on the real `package.json`** | no — this is the assertion that *would* fail on an unbalanced edit; see §1.11 |
| `118-122` | missing / mismatched workspace override rejected | no |
| `127-131` | every `requiredHostPeers` entry must exist as a peer | no |
| `132-137` | range-qualified override selectors rejected | no |
| `147-156` | doctor reports missing cordis / required plugin imports despite `optional: true` | no |
| `166-176` | peer-only drift, mismatched installed plugin (`0.1.15`) | no |

**Fragility worth fixing (not a current failure).** `compatibility.test.mjs:25` does
`pkg.peerDependencies['@deepseek-ai/dsh-agent'].replace(' || 0.2.0', '')`. The range now contains
both `|| 0.2.0` and `|| 0.2.0-rc.2`. It currently deletes the *right* token only because `0.2.0`
is enumerated before `0.2.0-rc.2`. Reorder the range (e.g. put `0.2.0-rc.2` first) and this test
silently deletes the wrong token and stops testing what it claims to test. Prefer an exact-string
anchor such as `.replace(' || 0.2.0 || 0.2.0-rc.2', ' || 0.2.0-rc.2')`.

### C3. `scripts/release-metadata.mjs`

| Line | Content | Fails now? |
| --- | --- | --- |
| `7` | release-version regex (`alpha\|beta\|rc`) | no — `0.1.22-rc.2` matches |
| `10` | dist-tag = `previewTag` when prerelease | no — `next` |
| `14` | `if (!match[4] && pkg.devDependencies['@deepseek-ai/dsh'] !== support.recommendedHost) throw 'latest must use the recommended host…'` | no — **currently dormant** because `0.1.22-rc.2` is a prerelease |

> `release-metadata.mjs:14` becomes live the moment a **stable** release is cut. It will then
> require `devDependencies['@deepseek-ai/dsh'] === recommendedHost`, i.e. both `0.2.0-rc.2`.
> They agree in the working tree, so the constraint is satisfiable — but note it forces the
> stable release to keep a prerelease host as its baseline, and `recommendedHost` can never be
> moved to a plain `0.2.0` while §1.7 applies.

### C4. `scripts/release-metadata.test.mjs` (53 lines)

| Line | Assertion | Fails now? |
| --- | --- | --- |
| `13-15` | stable `0.1.16`/`latest` metadata | no |
| `18`, `22` | synthetic `alphaOnly` devDeps + overrides → all `0.1.2-alpha.2` | no |
| `24` | expects `/recommended host/` | no — `0.1.2-alpha.2 !== 0.2.0-rc.2` still throws |
| `25-28` | synthetic `recommendedHost: '0.1.2-alpha.2'` → `/Alpha\/beta/` | no |
| `31-38` | alpha/beta/rc must use `next` | no |
| `42-44` | malformed release versions rejected | no |
| `45-47` | `^0.1.2-alpha.2` peer → `/enumerate/` | no |
| `51-52` | real `package.json`: `dist_tag === publishConfig.tag` | no — `next` === `next` |

### C5. `scripts/harness-source-verify.mjs` / `.test.mjs`

| Line | Content | Fails now? |
| --- | --- | --- |
| `harness-source-verify.mjs:8` | usage comment `[--candidate-version 0.2.0]` | no |
| `harness-source-verify.mjs:31` | allowed flag list | no |
| `harness-source-verify.mjs:68-72` | `policy.sourceCandidates` lookup → `No matching sourceCandidates entry…` | no — `0.2.0` still present |
| `harness-source-verify.mjs:169` | `selectSourceCandidate(policy, flags.get('--candidate-version'), …)` | no |
| `harness-source-verify.test.mjs:10` | fixture `{ version: '0.2.0', … }` | no |
| `harness-source-verify.test.mjs:13` | `--host-version` rejected as unknown | no |
| `harness-source-verify.test.mjs:22-28` | synthetic policy `{supportedHosts:[{version:'0.1.7-rc.2'}], sourceCandidates:[candidate]}` | no — local, independent of the repo policy |
| `harness-source-verify.test.mjs:40` | synthetic host manifest `version: '0.1.7-rc.2'` | no |

### C6. `scripts/harness-runtime-verify.mjs`

| Line | Content | Fails now? |
| --- | --- | --- |
| `3`, `13`, `27`, `29` | takes `--host-version <exact>` and validates it is an exact version | no — no host-version literal; caller-supplied |

### C7. `scripts/doctor.mjs` (the shipped user-facing diagnostic)

| Line | Content | Fails now? |
| --- | --- | --- |
| `71` | `policy.sourceCandidates?.find(c => c.version === host.version)` | no |
| `76` | `Unsupported host ${host.version}; recommended target is ${policy.recommendedHost}` | no — reads policy, no literal |
| `91` | reports `supportedHosts: allowed` | no |

`doctor.mjs` contains **no** host-version literal. It is fully policy-driven.

### C8. `scripts/readme-version.mjs` / `.test.mjs`

| Line | Content | Fails now? |
| --- | --- | --- |
| `readme-version.mjs:33` | whole-version regex built from `release-notes/*.md` names | no |
| `readme-version.mjs:42` | pins `@nanmicoder/dsh-agent-teams@<v>` must equal `package.json` version | no |
| `readme-version.test.mjs:41` | literal fixture naming `0.1.5-rc.1`, `0.1.2-alpha.5`, `0.1.2-alpha.2` | no — self-contained string |

> `readme-version.mjs:30-37` only compares against **plugin** versions derived from
> `release-notes/v*.md` filenames. It does **not** check host versions, so a stale
> `@deepseek-ai/dsh@0.1.7-rc.2` install line in a README is **silently unverified**. This was
> a real stale reference in the baseline; see Part 3.

### C9. Scripts with no host-version references at all

`verify.mjs` (85 KB), `lifecycle-verify.mjs` (85 KB), `stress-verify.mjs`, `quality-gates-tdd.mjs`,
`harness-compat-tdd.mjs`, `stability-tdd.mjs`, `member-failure-tdd.mjs`, `web-routes-verify.mjs`,
`clean-build.mjs`, `git-artifacts.mjs`, `verify-package.mjs`, `sync-skill.mjs`, `update-skills.mjs`
and the remaining `*.test.mjs` files contain **no** host-version literal and are unaffected.

## Class D — historical frozen records (do not touch)

`docs/` contains **194 files / 8599** host-version matches; `release-notes/` contains **13 files / 28**.
All of it is a historical record and none of it is read by the version gate or by CI.
Representative heaviest files (match counts):

- `docs/compatibility-audit-2026-09-05/harness-npm-snapshot.json` — 1373
- `docs/compatibility-audit-2026-09-05/dependency-resolution-evidence.json` — 462
- `docs/harness-0.1.7-rc.2-audit-2026-09-25/global-doctor.json` — 281
- `docs/harness-0.1.7-rc.2-audit-2026-09-25/cohort-0.1.7-rc.2.json` — 274
- `docs/harness-0.1.7-rc.2-audit-2026-09-25/cohort-0.1.5-rc.{1,2,3}.json` — 232 each
- `docs/issue-159-audit-2026-09-24/host-cohort.json` and `fix-verification/host-cohort.json` — 232 each
- `docs/releases/v0.1.19/cohort-0.1.5-rc.1.json` — 232
- `docs/session-latency-audit-2026-09-12/verification/0.1.5-rc.1-cohort.json` — 232
- `docs/maintenance-2026-09-06/**` — 215-216 per cohort file
- `docs/releases/v0.1.16-rc.3/**`, `docs/releases/v0.1.17/**`, `docs/releases/v0.1.17-rc.1/**`, `docs/releases/v0.1.18/**` — evidence bundles
- `docs/harness-0.1.5-rc.1-audit-2026-09-10/**`, `docs/compatibility-audit-2026-09-05/**`, `docs/theme-support-2026-09-11/**`, `docs/workspace-181-verification/**`, `docs/upgrade-skill-study-2026-09-06/**`
- `release-notes/v0.1.15.md` … `v0.1.22-rc.1.md` — 28 matches across 13 files

**Frozen — do not touch.** Note that the directory name `docs/harness-0.1.7-rc.2-audit-2026-09-25`
embeds a host version in the *path*; the release-notes filename convention `v<plugin-version>.md`
does **not** track host versions.

## Class E (out of scope, listed for completeness) — vendored skills

`skills/**` contains **63 files / 342** matches. These are vendored upstream skill documents and
reference fixtures, not part of the build or the gate, and `AGENTS.md` requires preserving upstream
skill files. `skills/upstream-lock.json` (6 matches) is a lock record. Notable:
`skills/plugin-upgrade/references/**` (up to 41 matches in `v0.1.6-alpha.1.md`),
`skills/plugin-write/scripts/query-registry.check.mjs` (9, incl. `harnessVersion: '0.1.2-alpha.2'`),
`skills/plugin-test/scripts/docker-release-smoke.test.mjs:16` (`dshVersion: '0.1.2-alpha.2'`),
`skills/plugin-upgrade/scripts/inject-lint.test.mjs:49,70` (`'^0.1.2-alpha.2'` peers).
**No manual edit required**; do not "fix" them as part of the adaptation.

---

# Part 3 — Concurrent-mutation warning

The repository was being edited by another teammate during this audit. Two concrete observations
bracket the change:

| Observation | First seen | Re-checked |
| --- | --- | --- |
| `git status --porcelain` | `M compatibility.json`, `M package.json`, `M pnpm-workspace.yaml` | additionally `M README.md`, `M README_ZH.md`, `?? release-notes/v0.1.22-rc.2.md` |
| `README.md:80` | `npm install --global @deepseek-ai/dsh@0.1.7-rc.2` (stale) | `npm install --global @deepseek-ai/dsh@0.2.0-rc.2` (fixed) |
| `release-notes/v0.1.22-rc.2.md` | absent | present (untracked) |

The lead's briefing stated the repo was at `0.1.22-rc.1`; that matches **HEAD** (`af32325`,
`git log` entry `e26cdb2 Prepare desktop 0.2.0 compatibility preview 0.1.22-rc.1`), not the
working tree, which is at `0.1.22-rc.2`.

**Consequence for the mutation pass:** the class-A change described in §1.7 is *already applied
in the working tree*. Verified by simulating the exact gate algorithm against the working-tree
`package.json` with the runtime version `0.2.0-rc.2`:

```
WORKING-TREE package.json version = 0.1.22-rc.2
incompatible DSH peers count = 0
{}
GATE PASSES (no warning) = true
```

So the remaining work is **not** "make the gate pass" — it is to verify the change is complete and
internally consistent, and to finish the artifacts that are still missing (Part 4).

---

# Part 4 — Open questions / unresolved items

1. **`docs/harness-0.2.0-rc.2-adaptation/README.md` does not exist**, yet both shipped READMEs link
   to it from their v0.1.22-rc.2 entry (`README.md:33`, `README_ZH.md:33`). It is a published
   broken link. My write scope covers only `host-gate-and-references.md`; **someone must decide
   whether to author the README or drop the link.** I did not create it.
2. **The peer range must stay exactly balanced with `compatibility.json`** (9 entries, see §1.11).
   Any future edit that touches one side without the other breaks `pnpm verify:compatibility`.
   This is a design constraint, not a bug — but it is easy to trip.
3. **`scripts/compatibility.test.mjs:25`** should be made order-independent (fragility note in C2).
4. **`release-metadata.mjs:14`** is dormant while the plugin is a prerelease. Before cutting a
   stable release, confirm that keeping `recommendedHost = 0.2.0-rc.2` (which §1.7 forces) is
   acceptable as a "stable latest" baseline.
5. **Not verified by me:** whether the adapted plugin *runs* correctly on 0.2.0-rc.2 beyond the
   version gate, and whether the `0.2.0` source candidate is still meaningful now that the desktop
   release is the recommended host. Both are outside this read-only task.

## §1.11 — The repo-internal coupling constraint (referenced above)

`scripts/compatibility.mjs:95-102` — verbatim:

```js
  for (const [name, range] of Object.entries(pkg.peerDependencies ?? {})) {
    if (name === '@deepseek-ai/dsh' || name.startsWith('@deepseek-ai/dsh-')) {
      const declared = range.split('||').map(value => value.trim())
      if (declared.length !== declaredVersions.length || declaredVersions.some(version => !declared.includes(version))) {
        throw new Error(`${name} peer range must enumerate the supported host targets and source candidates`)
      }
    }
  }
```

`declaredVersions` = `validatePolicy(input)` (the `supportedHosts` versions) **plus** the
`sourceCandidates` versions (`:62-64`). At snapshot time that is 8 hosts + 1 candidate = **9**, and
each of the 21 peer ranges enumerates exactly those 9 tokens — hence `validatePackageCompatibility`
succeeds.

Failure modes this creates, all of which surface at `compatibility.test.mjs:117`
(`assert.doesNotThrow`) and at the `verify:compatibility` script:

- Add `0.2.0-rc.2` to peers **without** declaring it in `compatibility.json` → peer range has 9,
  declared has 8 → throw.
- Declare it in `compatibility.json` **without** adding it to peers → declared has 9, range has 8 → throw.
- **Replace** `0.2.0` with `0.2.0-rc.2` in peers instead of appending → same length mismatch → throw.
- Move `recommendedHost` without adding a matching `supportedHosts` entry → `:43`
  "Exactly one recommended host is required" → throw.

This is why the answer to "what is the minimal declaration change" is a **pair** of coordinated
edits, not a single edit.
