# API compatibility delta: host `@deepseek-ai/*` 0.1.7-rc.2 → 0.2.0-rc.2

Scope: `C:\Sophia\dsh-agent-teams` (checkout `0.1.22-rc.1`).
Question answered: when the plugin's host pins move from `0.1.7-rc.2` to `0.2.0-rc.2`, is that a **pure version-declaration change**, or must plugin **source** under `src/**` also change?

## Verdict

**Declaration-only adaptation.** No `src/**` edit is required for the 0.1.7-rc.2 → 0.2.0-rc.2 host bump.

Every symbol the plugin value-imports or type-imports from the host is present in 0.2.0-rc.2 with a compatible declaration. Every package `exports` map is byte-identical between the two versions, so no import subpath changes. The host's own peer-dependency *name* sets are unchanged, so the host introduces no new required peer. The only work the bump requires is rewriting the version strings in `package.json` (`peerDependencies`, `devDependencies`).

Two deliberate qualifications, stated up front:

1. The plugin's `peerDependencies` today declare `0.2.0` (the final release), **not** `0.2.0-rc.2`. Under npm semver a prerelease cannot satisfy a plain `0.2.0` comparator, so `0.2.0-rc.2` must be *added* to those ranges. This is a declaration edit, and it is the one substantive thing the bump must not forget (see [§6](#6-the-one-declaration-action-required)).
2. This verdict is a **declaration-surface** result: it comes from comparing published `.d.ts` artifacts, not from a successful `tsc` run of the plugin against the new types. See [§8 Limitations](#8-limitations-and-explicit-partial-coverage).

## 1. Method and compared artifacts

Both versions of every consumed package were fetched from the public npm registry as tarballs into a scratch directory outside the repository (`C:\Sophia\_apiprobe\_extracted\<pkg>\<version>\package\`). No `pnpm install` was run in the repo and no repo file was modified except this report.

- Old side: `https://registry.npmjs.org/@deepseek-ai/<name>/-/<name>-0.1.7-rc.2.tgz`
- New side: `https://registry.npmjs.org/@deepseek-ai/<name>/-/<name>-0.2.0-rc.2.tgz`
- `cordis`: `4.0.2` → `4.0.4`; `schemastery`: `3.18.2` → `3.18.4` (the repo's `devDependencies` already pin `schemastery@3.18.4` **and** `cordis@4.0.4`, see [§7](#7-packages-whose-delta-is-already-absorbed)).

Compared per package: (a) the full file list of the tarball, (b) byte-equality of every `.d.ts`/`.d.mts`, (c) the `package.json` `exports` map, (d) peer/dependency key sets. Consumed-symbol set derived from `package.json` plus every `@deepseek-ai/...` specifier in `src/**` (71 import statements across 20 source files).

Sub-paths imported by the plugin: `.`, `./client`, `./types`. All three remain declared everywhere they are used.

## 2. Exports maps: no entry-point change

**All 27 compared packages have byte-identical `exports` maps across the bump.** Verified by structural JSON comparison, not by hand.

Plugin-relevant subpaths and their presence in both versions:

| Package | Subpaths (identical old and new) |
| --- | --- |
| `dsh-agent` | `.`, `./invariant`, `./package.json`, `./src/*`, `./types` |
| `dsh-api-session-controller` | `.`, `./client`, `./package.json`, `./remote`, `./remote-events`, `./src/*`, `./typert`, `./types` |
| `dsh-client-connection` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-locale` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-store` | `.`, `./package.json`, `./src/*` |
| `dsh-client-ui-chat` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-ui-conversation` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-ui-layout` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-ui-model-selection` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-ui-primitives` | `.`, `./package.json`, `./src/*` |
| `dsh-client-ui-renderer` | `.`, `./client`, `./invariant`, `./package.json`, `./src/*` |
| `dsh-client-ui-session` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-ui-sidebar-right` | `.`, `./client`, `./package.json`, `./src/*` |
| `dsh-client-ui-slots` | `.`, `./package.json`, `./src/*` |
| `dsh-commands` | `.`, `./brand`, `./invariant`, `./package.json`, `./remote`, `./src/*`, `./typert`, `./types` |
| `dsh-llm` | `.`, `./assistant-stream`, `./brand`, `./invariant`, `./message`, `./package.json`, `./remote`, `./src/*`, `./typert`, `./types` |
| `dsh-llm-retry` | `.`, `./invariant`, `./package.json`, `./types` |
| `dsh-session` | `.`, `./fork`, `./invariant`, `./package.json`, `./src/*`, `./surface`, `./types` |
| `dsh-session-projection` | `.`, `./package.json`, `./src/*`, `./types` |
| `dsh-subagent` | `.`, `./client`, `./internal`, `./invariant`, `./package.json`, `./remote`, `./src/*`, `./typert` |
| `dsh-system-prompt` | `.`, `./invariant`, `./package.json`, `./src/*` |
| `dsh-tools` | `.`, `./invariant`, `./package.json`, `./presentation`, `./src/*`, `./types` |
| `dsh-util-values` | `.`, `./package.json`, `./src/*` |
| `dsh-workspace` | `.`, `./invariant`, `./package.json`, `./src/*`, `./types` |
| `cordis` | `.`, `./package.json`, `./src/*` |
| `schemastery` | `.`, `./package.json`, `./src/*` |

Note in particular `@deepseek-ai/dsh-session/types` (used at `dsh-agent-teams\src\events.ts:19`, `dsh-agent-teams\src\client\session-navigation.ts:3`, `dsh-agent-teams\src\client\AgentTeamsCard.tsx:15`, `dsh-agent-teams\src\client\ActivityPanel.tsx:32`, `dsh-agent-teams\src\client\agent-teams-card-definition.ts:23`) and `@deepseek-ai/dsh-subagent/client` (used at `dsh-agent-teams\src\client\session-navigation.ts:4`) — both survive unchanged.

## 3. Peer/dependency key sets: no new required peer

For all 27 packages the **set of peer-dependency names is identical** old→new. The only peer-range movements are:

- Every `dsh-*` package re-pins its `@deepseek-ai/*` peers from `0.1.7-rc.2` to `0.2.0-rc.2`, and its `dsh-*` runtime `dependencies` likewise. Same names, new pins.
- Every `dsh-*` package keeps `@deepseek-ai/cordis` at `~4.0.4` in **both** versions — the host's cordis peer range did not move.
- `cordis` itself tightens its own two peers: `cordis-plugin-loader` `^1.0.3` → `~1.0.5` and `cordis-plugin-include` `^1.0.7` → `~1.0.9`. Same names, narrower ranges.

No host package added or removed a peer. Therefore raising the plugin's pins introduces **no new required peer** for the plugin to declare.

## 4. Per-package declaration coverage

`Δ.d.ts` = number of changed declaration files. All 27 compared packages are listed; nothing is omitted.

| # | Package | Old → New | Δ.d.ts | Δ.js | Plugin consumes it? | Verdict for the plugin |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `dsh-agent` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (value: `installModelSelection`; types: `Agent`, `AgentStatus`, `PreStepDecision`, `ModelSelection`) | unchanged |
| 2 | `dsh-api-session-controller` | 0.1.7-rc.2 → 0.2.0-rc.2 | 2 | 4 | yes (`SessionListState`) | additive only |
| 3 | `dsh-client-connection` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (`HostConnectionService`) | unchanged |
| 4 | `dsh-client-locale` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (side-effect `./client`) | unchanged |
| 5 | `dsh-client-store` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (`ObservableSnapshot`) | unchanged |
| 6 | `dsh-client-ui-chat` | 0.1.7-rc.2 → 0.2.0-rc.2 | 7 (+2 new) | 2 | yes (`ChatConversationViewNode`, augments `ChatNodeDataMap`) | no consumed symbol affected — see §5.1 |
| 7 | `dsh-client-ui-conversation` | 0.1.7-rc.2 → 0.2.0-rc.2 | 7 (+1 new) | 1 | yes (`ConversationNodeContext`, `ConversationNodeDefinition`) | no consumed symbol affected — see §5.2 |
| 8 | `dsh-client-ui-layout` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 1 | yes (`UsePanelInfo`) | unchanged |
| 9 | `dsh-client-ui-model-selection` | 0.1.7-rc.2 → 0.2.0-rc.2 | 3 (+1 new) | 1 | yes (`ModelDirectory`, `ModelDirectoryResolver`) | compat preserved — see §5.3 |
| 10 | `dsh-client-ui-primitives` | 0.1.7-rc.2 → 0.2.0-rc.2 | 8 (+2 new) | 1 | yes (`Menu`, `MenuEntry`, `Modal`) | additive only |
| 11 | `dsh-client-ui-renderer` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 1 | yes (side-effect `./client`) | unchanged |
| 12 | `dsh-client-ui-session` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (side-effect `./client`) | unchanged |
| 13 | `dsh-client-ui-sidebar-right` | 0.1.7-rc.2 → 0.2.0-rc.2 | 3 | 1 | yes (`ISidebarRight`) | compat preserved — see §5.4 |
| 14 | `dsh-client-ui-slots` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (`PropsLocale`, `PropsRuntime`, `Translate`) | unchanged |
| 15 | `dsh-commands` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 1 | yes (`CommandInvocation`, `CommandResult`) | unchanged |
| 16 | `dsh-llm` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 1 | yes (value: `createUserMessage`, `LlmError`, `ReasoningEffortId`; types: `ContentBlock`, `MessageId`, `MessageSource`, `UserMessage`; augments `MessageSourceMap`) | unchanged |
| 17 | `dsh-llm-retry` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | declared devDep only | unchanged |
| 18 | `dsh-session` | 0.1.7-rc.2 → 0.2.0-rc.2 | 2 | 3 | yes (value: `KNOWN_SESSION_EVENT_TYPES`; types: `Session`, `SessionEvent`, `SessionId`, `SessionEventMap`; augments `SessionEventMap`) | additive only — see §5.5 |
| 19 | `dsh-session-projection` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | declared devDep only | unchanged |
| 20 | `dsh-subagent` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 1 | yes (value: `SubagentError`, `foldSubagentDescriptor`; type: `SubagentAddress`) | unchanged |
| 21 | `dsh-system-prompt` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (side-effect) | unchanged |
| 22 | `dsh-tools` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (value: `defineTool`; type: `ToolRunContext`) | unchanged |
| 23 | `dsh-util-values` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 1 | yes (`JsonValue`) | unchanged |
| 24 | `dsh-workspace` | 0.1.7-rc.2 → 0.2.0-rc.2 | **0** | 0 | yes (`WorkspaceRegistry`) | unchanged |
| 25 | `dsh` (aggregate) | 0.1.7-rc.2 → 0.2.0-rc.2 | 3 | 1 | **no** — not imported by `src/**` | unchanged for the plugin |
| 26 | `cordis` | 4.0.2 → 4.0.4 | 3 | 1 | yes (`Context`, `Service`) | already absorbed — see §5.6 |
| 27 | `schemastery` | 3.18.2 → 3.18.4 | 1 | 2 | yes (`z` default) | already absorbed — see §7 |

Packages with **zero** declaration changes: 14 of 27. Only 10 packages changed any `.d.ts` at all.

## 5. Consumed-symbol result (the load-bearing evidence)

Every symbol the plugin imports was located in the new declarations. Presence was checked by scanning the full extracted `.d.ts` set of each package's 0.2.0-rc.2 tarball.

| Package / subpath | Consumed symbol | Present in 0.1.7-rc.2 | Present in 0.2.0-rc.2 | Signature changed? |
| --- | --- | --- | --- | --- |
| `cordis` | `Context` | yes | yes | no |
| `cordis` | `Service` | yes | yes | no |
| `schemastery` | default export `z` | yes | yes | generic arity only, defaulted (see §5.6) |
| `dsh-agent` | `Agent`, `AgentStatus`, `PreStepDecision`, `ModelSelection`, `installModelSelection` | yes | yes | no — `dsh-agent` has 0 changed `.d.ts` |
| `dsh-llm` | `createUserMessage`, `LlmError`, `ReasoningEffortId`, `ContentBlock`, `MessageId`, `MessageSource`, `UserMessage` | yes | yes | no — `dsh-llm` has 0 changed `.d.ts` |
| `dsh-tools` | `defineTool`, `ToolRunContext` | yes | yes | no — 0 changed `.d.ts` |
| `dsh-subagent` | `SubagentError`, `foldSubagentDescriptor` | yes | yes | no — 0 changed `.d.ts` |
| `dsh-subagent/client` | `SubagentAddress` | yes | yes | no |
| `dsh-session` | `Session`, `SessionEvent`, `SessionId`, `KNOWN_SESSION_EVENT_TYPES` | yes | yes | no |
| `dsh-session/types` | `SessionEventMap`, `SessionId` | yes | yes | no — `lib/types/types.d.ts` unchanged |
| `dsh-commands` | `CommandInvocation`, `CommandResult` | yes | yes | no — 0 changed `.d.ts` |
| `dsh-util-values` | `JsonValue` | yes | yes | no |
| `dsh-workspace` | `WorkspaceRegistry` | yes | yes | no |
| `dsh-client-connection` | `HostConnectionService` | yes | yes | no |
| `dsh-client-store` | `ObservableSnapshot` | yes | yes | no |
| `dsh-client-ui-slots` | `PropsLocale`, `PropsRuntime`, `Translate` | yes | yes | no — 0 changed `.d.ts` |
| `dsh-api-session-controller/client` | `SessionListState` | yes | yes | no |
| `dsh-client-ui-chat/client` | `ChatConversationViewNode`, `ChatNodeDataMap` | yes | yes | no |
| `dsh-client-ui-conversation/client` | `ConversationNodeContext`, `ConversationNodeDefinition` | yes | yes | no |
| `dsh-client-ui-layout/client` | `UsePanelInfo` | yes | yes | no |
| `dsh-client-ui-model-selection/client` | `ModelDirectory`, `ModelDirectoryResolver` | yes | yes | `ModelDirectory` ctor only — unused (see §5.3) |
| `dsh-client-ui-primitives` | `Menu`, `MenuEntry`, `Modal` | yes | yes | no |
| `dsh-client-ui-sidebar-right/client` | `ISidebarRight` | yes | yes | no |

**No consumed symbol was removed, and no consumed symbol was renamed.**

The three declaration-merge targets the plugin augments also all still exist:

- `MessageSourceMap` — `dsh-llm/lib/types/message.d.ts:101` (augmented at `dsh-agent-teams\src\harness-compat.ts:53-57`).
- `SessionEventMap` — `dsh-session/lib/types/types.d.ts:255` (augmented at `dsh-agent-teams\src\event-types.ts:105`).
- `ChatNodeDataMap` — `dsh-client-ui-chat/lib/types/client/index.d.ts:30` (augmented at `dsh-agent-teams\src\client\agent-teams-card-definition.ts:38`).

### 5.1 `dsh-client-ui-chat` — 7 changed files, none consumed

Changed: `lib/types/chat-settings.d.ts`, `lib/types/client/apply.d.ts`, `lib/types/client/chat/TurnProcessNodeView.d.ts`, `lib/types/client/chat/message-chrome.d.ts`, `lib/types/client/locale.d.ts`, `lib/types/client/transcript-view.d.ts`, `lib/types/index.d.ts`. Added: `lib/types/client/chat/RunningStatus.d.ts`, `lib/types/client/chat/RunningWhaleTail.d.ts`.

Substantive changes: `ChatSettings['transcriptView']` became optional/nullable and legacy `normal` now reads as `detailed`; `TranscriptViewPolicy` gained an optional `defaultMode` ctor param; the elapsed-time vocabulary was renamed (`duration.seconds|minutes|hours` → `duration.secondUnit|minuteUnit|hourUnit`; `message.turnProcess.deepDivingFor` → `chat.deepDivingFor`); `formatRunDuration` now returns `RunDurationPart[]` instead of `string` and `formatLiveRunDuration` was removed.

**None of this is consumed.** The plugin imports only `ChatConversationViewNode` from `@deepseek-ai/dsh-client-ui-chat/client`, which lives in `lib/types/client/contract/chat-nodes.d.ts` (`ChatConversationViewNode` at line 3) — a file **not** in the changed set. The augmentation target `ChatNodeDataMap` is in `lib/types/client/index.d.ts`, also **not** changed.

### 5.2 `dsh-client-ui-conversation` — 7 changed files, none consumed

Changed: input/composer/draft-editor contracts (`lib/types/client/contract/{input,composer-submission,draft-editor}.d.ts`, `lib/types/client/input/{facade,hub}.d.ts`), `lib/types/client/apply.d.ts`, `lib/types/client/locales.d.ts`. Added `lib/types/client/input/submission-analytics.d.ts`.

The consumed symbols live in `lib/types/client/contract/conversation.d.ts` — `ConversationNodeContext` at line 131, `ConversationNodeDefinition` at line 164 — which is **not** in the changed set. `lib/types/client/index.d.ts` (the re-export surface the plugin imports through) is also unchanged.

### 5.3 `dsh-client-ui-model-selection` — constructor widened, not called by the plugin

`ModelDirectory`'s constructor changed from 5 to 7 parameters (`lib/types/client/directory.d.ts:35-62`):

```
old: constructor(sessions, sessionId, available, catalog, projected)
new: constructor(sessions, sessionId, available, catalog, projected, isBlank: () => boolean, track?: TrackProductEvent | undefined)
```

This is not a break for this plugin because **the plugin never constructs `ModelDirectory`**. It is imported `import type` only (`dsh-agent-teams\src\client\ActivityPanel.tsx:30`, `dsh-agent-teams\src\client\StagingPlanEditor.tsx:10`) and instances are obtained from the service:

- `dsh-agent-teams\src\client\ActivityPanel.tsx:1277` — `modelDirectories.directoryFor(team.captainSessionId as SessionId)`
- `dsh-agent-teams\src\client\WorkspaceActivity.tsx:124` — `modelDirectories.directoryFor(sessionId)`

`ModelDirectoryResolver.directoryFor(sessionId: SessionId): ModelDirectory` is unchanged (`lib/types/client/service.d.ts`, class at line 11, `directoryFor` at line 25), as is the `ctx.modelDirectories` module augmentation and the `./client` re-exports (`lib/types/client/index.d.ts:3,5`). The plugin's `inject` list (`dsh-agent-teams\src\client\index.tsx:40`) and usage `ctx.modelDirectories` (`dsh-agent-teams\src\client\index.tsx:80,107`) remain valid.

One knock-on worth knowing: `directory.d.ts` now imports `TrackProductEvent` from `@deepseek-ai/dsh-client-product-analytics/client`. That is a transitive dependency of `dsh-client-ui-model-selection`, resolvable from its own `node_modules` under pnpm, so it does not require a plugin-side declaration. If a `tsc` run ever reports a missing `dsh-client-product-analytics` type, the fix is in the dependency installation, not in plugin source.

### 5.4 `dsh-client-ui-sidebar-right` — public removals that the plugin does not touch

Removed from the public surface: the `SidebarRightBinding` interface (dropped from `lib/types/client/index.d.ts`'s re-export list) and `SidebarRightInjected.bindService` (`lib/types/client/shell/SidebarRight.d.ts`). `SidebarRightController`'s constructor changed to `(tabs, pin, host: SidebarRightHost, sessions: SidebarRightSessions)`, and `createSidebarRightController` gained a required third `host` parameter plus new `show`/`measure` returns.

The plugin is unaffected because it never mentions `SidebarRightBinding` or `bindService`, never constructs `SidebarRightController`, and never calls `createSidebarRightController`. It obtains the instance purely through injection — `ctx.inject(['sidebarRight', 'sidebarRightTabs'], ...)` at `dsh-agent-teams\src\client\index.tsx:88`, published via `bridge.set(native.sidebarRight)` at `dsh-agent-teams\src\client\index.tsx:109` — and uses only these `ISidebarRight` members:

- `mounted` (`lib/types/client/service.d.ts:138` interface, `:213` implementation) — read at `dsh-agent-teams\src\client\WorkspaceActivity.tsx:57,67`
- `isExpanded()` — called at `dsh-agent-teams\src\client\WorkspaceActivity.tsx:57`
- `openTab()` — called at `dsh-agent-teams\src\client\WorkspaceActivity.tsx:59,71`

All three keep identical signatures; the diff over `ISidebarRight` is documentation-only. **Behavioural caveat (not a type break):** the documented meaning of `mounted` changed from "the session whose seat is mounted — published from the seat's passive effect" to "the Session on screen — named from the selection and main panel before React renders it". The plugin's guard at `dsh-agent-teams\src\client\WorkspaceActivity.tsx:57-71` compares `sidebar.mounted.getSnapshot()` against `current` and re-checks before acting, which is consistent with either timing, but this is a runtime-semantics change that declaration comparison cannot settle. It belongs to whichever task validates the right-sidebar behaviour on the real host.

### 5.5 `dsh-session` — additive

`lib/types/index.d.ts` adds one export: `ToolCallRecovery`. `lib/types/repair.d.ts` adds the `ToolCallRecovery` class (`observe(event)`, `results()`) and its `OpenTurnCloseCause` type. `interruptedTurnClosers`, `TOOL_NOT_STARTED` and `TOOL_OUTCOME_UNKNOWN` keep their signatures; `lib/types/types.d.ts` — the home of `SessionId`, `SessionEvent`, `SessionEventMap` and the `Session` surface — is **unchanged**.

The plugin's only *value* use of this package is `KNOWN_SESSION_EVENT_TYPES`, declared in `lib/types/known-event-types.ts` and re-exported at `lib/types/index.d.ts` with an unchanged export statement:

- `dsh-agent-teams\src\events.ts:17` — `import * as dshSession from '@deepseek-ai/dsh-session'`
- `dsh-agent-teams\src\events.ts:45-47` — reads `dshSession.KNOWN_SESSION_EVENT_TYPES` behind an `as unknown as` cast, so even a hypothetical re-typing would not break compilation.

No source edit required.

### 5.6 `cordis` — one narrowing, not called by the plugin

- `lib/types/index.d.ts`: adds `export type { Volatile, VolatileSnapshot } from '@deepseek-ai/cosmokit'` — additive.
- `lib/types/events.d.ts`: `'internal/update'` changes from `(this, config, noSave, next: () => void | Promise<void>) => void | Promise<void>` to `(this, config, noSave, next: () => void) => void`.
- `lib/types/fiber.d.ts`: `Fiber.update(config: any, noSave?: boolean)` changes from `void | Promise<void>` to `void`.

This is the only genuinely *narrowing* declaration change in the whole delta. It does not affect this plugin: the plugin's only `.update(` call site is `createHash('sha256').update(name)` at `dsh-agent-teams\src\state.ts:95`, which is Node's `Hash`, not cordis's `Fiber`. The plugin never references `Fiber`, `internal/update`, or `Volatile` anywhere in `src/**`. Its cordis usage is `import type { Context }` (14 files) and `Service` via `ctx`, both unchanged.

## 6. The one declaration action required

The plugin's `peerDependencies` currently allow the `0.2.0` line as the literal string `"0.2.0"`. The installed host ships `0.2.0-rc.2`. Under npm semver, `0.2.0-rc.2` does **not** satisfy `"0.2.0"`, because a prerelease version only matches a range that carries a comparator with the same `major.minor.patch` tuple *and* a prerelease. So the existing ranges would reject the very version being targeted.

Required declaration edit — the 21 `@deepseek-ai/dsh-*` ranges in `peerDependencies` (plus `schemastery`) need the literal `0.2.0-rc.2` alternant, e.g.

```
"@deepseek-ai/dsh-agent": "0.2.0-rc.2 || 0.2.0 || 0.1.7-rc.2 || ... "
```

and set `devDependencies` to `0.2.0-rc.2`. This is a version-declaration change and is the reason the verdict is *"declaration-only"* rather than *"no change at all"*. It involves no `src/**` edit. Ownership of the exact version-gate mechanics belongs to the sibling `gate-audit` task; this report only records that the API surface itself imposes no source constraint.

## 7. Packages whose delta is already absorbed

- **`schemastery`** — the repo's `devDependencies` already pin `3.18.4`, and the installed host ships `3.18.4`, so there is no practical delta. For completeness: 3.18.2 → 3.18.4 threads a new third generic through the schema type — `Schema<S = any, T = S>` → `Schema<S = any, T = S, Mode extends SchemaMode = 'plain'>` — adds a `volatile()` builder and a `volatile?: boolean` meta flag, and widens `object()` to `Schema<ObjectS<NoInfer<X>>, ObjectT<NoInfer<X>>>`. Because `Mode` defaults to `'plain'`, existing two-argument references stay valid. The plugin's uses are `z.object({...})` at `dsh-agent-teams\src\index.ts:91,95,101,106,116,123` and the type position `export const Config: z<Config>` at `dsh-agent-teams\src\index.ts:95`; both compile against 3.18.4 already.
- **`cordis`** — the repo's `devDependencies` already pin `4.0.4`; only the `peerDependencies` range `^4.0.2` understates it. No source impact (see §5.6).

## 8. Corroboration of the repo's pre-adaptation claim

`C:\Sophia\dsh-agent-teams\docs\harness-0.2.0-pre-adaptation\backend-audit.md` (comparing host git `dsh-v0.1.7-rc.2` @ `477b4f42` against master snapshot `21638c56`) concluded 没有重写子代理、消息投递、模型与推理力度、前端槽位和会话恢复. That claim is **corroborated** here, on a different and stricter axis (published npm artifacts, not a git snapshot), by the following concrete facts:

| Claim in the pre-adaptation report | Independent evidence from this comparison |
| --- | --- |
| 子代理 (subagent) contract unchanged | `dsh-subagent@0.2.0-rc.2` has **0 changed `.d.ts`** files across all 59 files in the tarball. `SubagentError`, `foldSubagentDescriptor`, `SubagentAddress` all present unchanged. |
| 消息投递 (message delivery) unchanged | `dsh-llm@0.2.0-rc.2` has **0 changed `.d.ts`** files. `createUserMessage`, `MessageSourceMap` (augmentation target) present unchanged. |
| 模型与推理力度 (model & reasoning effort) unchanged | `dsh-agent@0.2.0-rc.2` has **0 changed `.d.ts`** files. `installModelSelection`, `ModelSelection`, `AgentStatus` present unchanged; `dsh-llm`'s `ReasoningEffortId` unchanged. |
| 前端槽位 (front-end slots) unchanged | `dsh-client-ui-slots@0.2.0-rc.2` has **0 changed `.d.ts`** files. The specific node contracts the plugin augments — `dsh-client-ui-chat/lib/types/client/contract/chat-nodes.d.ts`, `dsh-client-ui-conversation/lib/types/client/contract/conversation.d.ts`, and both packages' `client/index.d.ts` — are all absent from their packages' changed-file sets. |
| 会话恢复 (session restore) unchanged | `dsh-session/lib/types/types.d.ts` unchanged; the 0.2.0 change is confined to `lib/types/index.d.ts` (one added export) and `lib/types/repair.d.ts` (`ToolCallRecovery` added). `dsh-session-projection@0.2.0-rc.2` has **0 changed `.d.ts`** files. |

The report's own caveat — *"After 不是已发布 0.2.0 产物；结论不覆盖此 SHA 之后的变化"* — is the gap this document closes: the comparison here is between the two **published** npm artifacts, so the conclusion now covers the shipped `0.2.0-rc.2` release, not a mid-development commit.

The report also warned that the plugin *"仍依赖内部 symbol API"* (still depends on an internal symbol API). That is true and remains true — see `dsh-agent-teams\src\harness-compat.ts:28,146-171` reading `Symbol.for('dsh.subagent.deliverPrompt')`. It is unchanged in 0.2.0-rc.2, but it is a runtime symbol rather than a declared type, so **declaration comparison cannot certify it**. It must be exercised on a real host.

## 9. Limitations and explicit partial coverage

Coverage is **complete for the declaration surface** and **incomplete for runtime behaviour**:

1. **No compiled typecheck was run.** The verdict rests on comparing `.d.ts` artifacts, which is the stated method, not on `tsc -p tsconfig.json --noEmit` passing against the new types. The build environment is owned by another teammate and I was instructed not to run `pnpm install` in the repository. Item §5.3 (`dsh-client-product-analytics` resolution) is the one place where a real `tsc` run could still surface something; if it does, the fix is dependency installation, not source.
2. **Runtime-only contracts are out of scope.** The `Symbol.for('dsh.subagent.deliverPrompt')` handshake (`dsh-agent-teams\src\harness-compat.ts:28`) and the `SidebarRight.mounted` timing redefinition (§5.4) cannot be certified from declarations and need host execution.
3. **`dsh-client-ui-sidebar-right`'s removals are safe only because the plugin does not use them.** If any future source change starts importing `SidebarRightBinding` or calling `bindService`, it will fail against 0.2.0-rc.2. This is recorded as a constraint, not a present defect.
4. **The 0.2.0 aggregate package (`@deepseek-ai/dsh`) is not imported by `src/**`** and was compared for completeness only; its `args.d.ts`/`bin.d.ts`/`plugin.d.ts` changes (all optional-parameter additions) affect the CLI, not this plugin.

## 10. Raw evidence location

Scratch artifacts (outside the repo, disposable): `C:\Sophia\_apiprobe\`

- `_pack.mjs` — registry tarball fetcher/extractor (26 packages × 2 versions + cordis + schemastery, 54 tarballs)
- `_filediff.mjs` / `_filediff.txt` — per-package file-list and byte-diff summary across both versions
- `_exports.mjs` — structural `exports`-map and peer/dependency-set comparison
- `_verify.mjs` — consumed-symbol existence check against both extracted trees
- `_diffs.txt` — unified diffs of every changed `.d.ts` (135 KB)
- `_extracted/<pkg>/<version>/package/...` — the extracted declaration trees
