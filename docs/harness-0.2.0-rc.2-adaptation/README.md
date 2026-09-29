# Harness desktop 0.2.0-rc.2 adaptation

This fork (`AEmbers/dsh-agent-teams`) is adapted to the published **DeepSeek Harness desktop `0.2.0-rc.2`** release.
Upstream `NanmiCoder/dsh-agent-teams@0.1.22-rc.1` declared exact peer compatibility with `0.2.0` only, and
**`0.2.0-rc.2` does not satisfy `0.2.0`**, so the host refused to install it.

Date: 2026-09-29 · Host: DeepSeek Harness desktop 0.2.0-rc.2 (Windows x64) · Plugin: `0.1.22-rc.2`

---

## 1. Why the plugin was refused

The refusal the user saw came from the host's plugin compatibility gate, verbatim from
`/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js` inside `resources/app.asar`:

```js
// L294 — what counts as a DSH peer
if (name !== "@deepseek-ai/dsh" && !name.startsWith("@deepseek-ai/dsh-")) continue;
// L300 — the comparison that decides
if (requirement.trim() === "" || !semver.satisfies(runtimeVersion, requirement, { includePrerelease: true })) peers[name] = range;
```

`runtimeVersion` is the version of `@deepseek-ai/dsh-app-boot` **itself** (L271-275 resolves `../package.json`),
which on this host is `0.2.0-rc.2`. Four consequences decided the whole design:

| Fact | Consequence |
| --- | --- |
| The comparison is `semver.satisfies`, not string equality. | `satisfies('0.2.0-rc.2', '0.2.0', {includePrerelease:true}) === false` — a prerelease never satisfies the equivalent release comparator. Neither does `^0.2.0` nor `~0.2.0`. Changing the range *operator* cannot fix it. |
| `includePrerelease: true` is set. | It does not rescue the case above; `0.2.0-rc.2` sorts strictly below `0.2.0`. |
| Only `peerDependencies` is read, and only names matching `@deepseek-ai/dsh` / `@deepseek-ai/dsh-*`. | `@deepseek-ai/cordis`, `@deepseek-ai/schemastery` and `react` are never checked. `devDependencies` is never checked. |
| `peerDependenciesMeta` / `optional: true` is **not consulted**. | This fork marks all 21 DSH peers optional, and that buys **no** exemption. All 21 are checked regardless. |

The install pre-flight lives at `/dsh/node_modules/@deepseek-ai/dsh-plugin-manager/lib/index.js:500-514` and refuses
**before pnpm is invoked** — which is why the message ended with `dsh: nothing was installed.` The same gate also fires
at boot and when listing plugins, so passing it once is not a one-time event.

An **exact-version exemption** (`dsh plugin allow-version <pkg>@<ver> --dsh-version <ver> --accept-risk`) is not a fix:
it is a grant stored in the *consumer's* profile, keyed to one package version and one runtime version. It does not
change the plugin, does not ship, and expires on any version change. This adaptation passes the gate on its own merits
and uses **no exemption**.

## 2. What changed

Declaration-only. **No `src/**` file was modified.**

| File | Change |
| --- | --- |
| `package.json` | `version` → `0.1.22-rc.2`. All 21 `@deepseek-ai/dsh-*` peer ranges gain ` || 0.2.0-rc.2`. All 25 `@deepseek-ai/dsh*` devDependencies → `0.2.0-rc.2`. |
| `compatibility.json` | `recommendedHost` → `0.2.0-rc.2`; `{"version":"0.2.0-rc.2","track":"recommended"}` added to `supportedHosts`; `0.1.7-rc.2` demoted to `legacy`; `sourceCandidates[0] = "0.2.0"` deliberately unchanged. |
| `pnpm-workspace.yaml` | All 273 `@deepseek-ai/dsh*` overrides → `0.2.0-rc.2`. Format kept as valid JSON (see §5). |
| `README.md`, `README_ZH.md` | Recommended pair, support matrix, install pins and CLI host version moved to `0.2.0-rc.2` / `0.1.22-rc.2`. |
| `release-notes/v0.1.22-rc.2.md` | New release note. |
| `scripts/compatibility.test.mjs` | Made the peer-range assertion order-independent (see §6). |
| `lib/client.js`, `lib/git-artifact-stamp.json` | Rebuilt outputs (see §4). |

The two halves — the peer ranges and `compatibility.json` — are a **coordinated pair, not two independent edits**.
`scripts/compatibility.mjs:97-100` splits each peer range on `||` and requires it to enumerate exactly
`declaredHostVersions()` (the `supportedHosts` versions **plus** the `sourceCandidates` versions) with equal length.
That is now 8 hosts + 1 candidate = 9 tokens on both sides. Adding one side without the other fails
`pnpm verify:compatibility`; so does *replacing* `0.2.0` with `0.2.0-rc.2` instead of appending it.

A range-operator-only fix is impossible (see §1), and declaring a peer as `workspace:^` / `workspace:~` / `workspace:*`
would make the gate compare the runtime against itself and always pass — a bypass, not an adaptation, and it would
break the equal-length check. It is not used here.

## 3. Evidence that the target version is the right one

- `resources/runtime/primary-runtime/runtime.json` → `"desktopVersion": "0.2.0-rc.2"`.
- The installed host ships **279** `@deepseek-ai/dsh-*` packages, **all** at exactly `0.2.0-rc.2`
  (plus independently versioned `cordis 4.0.4`, `schemastery 3.18.4`, `cosmokit 1.8.5`).
- All **273** override entries in this repo map one-to-one onto that shipped closure: 273 matched at `0.2.0-rc.2`,
  **0 conflicts, 0 missing**. The override set did not have to be invented; it was verified package by package against
  what the desktop application actually loads.

## 4. Verification performed

- **Build**: `install` → `typecheck` → `build` all exit 0 against the real `0.2.0-rc.2` packages, with **zero** `tsc`
  diagnostics on both `tsconfig.json` and `tsconfig.client.json`. Resolved `node_modules/@deepseek-ai/dsh-agent` is
  `0.2.0-rc.2`; the regenerated `pnpm-lock.yaml` contains 2938 references to `0.2.0-rc.2` and 0 to `0.1.7-rc.2`.
- **API delta**: 27 host packages compared as published npm tarballs on both sides (`.d.ts`, `exports` maps,
  peer-name sets). 14 of 27 have zero declaration changes, all 27 `exports` maps are byte-identical, and no declared
  peer name set changed. Every symbol this plugin consumes is present and unchanged. Three changes that look breaking
  (`ModelDirectory` constructor arity, `SidebarRightBinding` removal, `Fiber.update()` narrowing) were each checked
  against the plugin's actual usage and are not reachable from it. This is what justifies "declaration-only".
- **Host gate**: the adapted declarations were replayed through the host's own gate function (§1) and pass with
  **0 incompatible peers**, while the published `0.1.21` declaration is still refused — see `gate-acceptance-proof.md`.
- **Repo verifiers and packed artifact**: see `verification-report.md`.

The rebuilt `lib/` is committed on purpose. `scripts/git-artifacts.mjs` digests the build outputs *and* a settings
projection of `package.json` (which now carries a new version and new devDependencies) plus `pnpm-lock.yaml`, so
reverting `lib/` would make `verify:git-artifact` fail. The visible `lib/client.js` change is a CSS-module class-name
prefix derived from the **build machine's absolute source path** — upstream's own artifact embeds
`/Users/nanmi/.codex/worktrees/e986/...`, ours embeds `C:\Sophia\...`. It is cosmetic and self-consistent: the old
prefix no longer appears anywhere under `lib/`, and the CSS string and its class map are regenerated together in the
same file.

## 5. Local pnpm note (do not "fix" this in the repo file)

This machine's bundled pnpm enforces a `minimumReleaseAge` supply-chain policy that rejects the freshly published
`0.2.0-rc.2` packages, so a plain `pnpm install --frozen-lockfile` fails with *"The lockfile contains entries that the
active policies reject."* That is a **local** policy, not a repo property — CI pins `pnpm@10.33.0`, which has no such
policy. Neutralise it per-invocation:

```sh
pnpm install --frozen-lockfile --config.minimum-release-age=0
# or, to cover child processes too:
export npm_config_minimum_release_age=0
```

pnpm 11.22 "helpfully" writes a `minimumReleaseAgeExclude` block into `pnpm-workspace.yaml` when it resolves. **Do not
commit it:** it writes the key unquoted, which makes the file invalid JSON, and `scripts/compatibility.mjs` does
`JSON.parse` on that file — the repo's own verifier would break. `pnpm-workspace.yaml` must stay valid JSON.

## 6. Known fragility

`scripts/compatibility.test.mjs` used to strip a token with `.replace(' || 0.2.0', '')`. The range now contains both
`|| 0.2.0` and `|| 0.2.0-rc.2`, so that literal deleted whichever came first and would silently stop testing anything
if the range were reordered. It now splits, asserts the precondition, and filters — order-independent and still red
if the token is missing.

`scripts/release-metadata.mjs:14` is dormant while the plugin version is a prerelease. It becomes live for a stable
release and will then require `devDependencies['@deepseek-ai/dsh'] === recommendedHost`. They agree today, but note it
forces a stable release to keep a prerelease host as its baseline.

## 7. Not verified here

- **Runtime behaviour beyond the gate.** Compilation, types and the version gate are proven; a full run of the plugin's
  team scenarios on this host is not. Two runtime-only paths deserve an explicit smoke test on a real host:
  the `Symbol.for('dsh.subagent.deliverPrompt')` handshake, and `dsh-client-ui-sidebar-right`'s **redefined**
  `mounted` semantics (it now means "the session on screen, named before React renders" rather than "the session whose
  seat is mounted, published from the seat's passive effect"). The plugin re-checks `mounted` before acting
  (`src/client/WorkspaceActivity.tsx`), so it looks safe under either timing, but only a run settles it.
- **The `0.2.0` source candidate.** It is kept because the repo records the official source line separately from
  published hosts. Now that `0.2.0-rc.2` is the recommended host, whether that candidate is still worth carrying is a
  maintenance decision, not a correctness one.

## 8. Reproduce

```sh
git clone https://github.com/AEmbers/dsh-agent-teams.git
cd dsh-agent-teams
pnpm install --no-frozen-lockfile --config.minimum-release-age=0   # first time: re-resolve for 0.2.0-rc.2
pnpm typecheck
pnpm build
pnpm verify:compatibility
```
