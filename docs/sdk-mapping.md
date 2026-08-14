# defims/pi_agent_rust ↔ earendil-works/pi Alignment

> **English (default)** · 中文版: [sdk-mapping.zh-CN.md](./sdk-mapping.zh-CN.md)
>
> `defims/pi_agent_rust` is the Rust version of the pi engine ([earendil-works/pi](https://github.com/earendil-works/pi),
> TypeScript SDK `@earendil-works/pi-coding-agent`), continuously tracking upstream.
>
> This document records the `AgentSession` interface differences between the two, serving as
> the baseline for filling missing in_process methods and detecting changes when tracking
> upstream releases.
>
> - **Alignment target**: earendil-works/pi's `AgentSessionLike` interface (agegr/pi-web `lib/pi-types.ts`)
> - **Aligned object**: defims/pi_agent_rust's `AgentSessionHandle` (in_process path, `src/sdk.rs`)
> - **Rust crate lib name**: `pi` (Cargo package name `pi_agent_rust`)

## Core rules

- **Docs default to English**: the canonical copy is English (`X.md`); the Chinese version lives at `X.zh-CN.md`, cross-linked from the file header; update both copies together on any change.
- **Commits in English**: git commit messages in this repository are always written in English.

## Fork core requirements

The consumer (moho-mate) embeds the engine in-process (in_process path). Three core
requirements the fork must uphold while filling gaps / tracking upstream:

1. **Parallel-capable** — multiple `AgentSessionHandle` instances run concurrently (multi-session); no global lock or shared mutable state between handles may serialize them.
2. **Async parallelism** — concurrency is expressed as async tasks (the engine runs on the asupersync runtime, built by the caller) instead of blocking OS threads; events flow out through `on_event`/`subscribe` callbacks.
3. **Async throughout** — the whole surface is `async fn`, mirroring the TS SDK's Promise-returning shape; long operations (prompt/bash/compact) are cancellable at any time via `AbortHandle`/`Signal` and never block the caller.

## Status legend

- ✅ **Aligned** — present on both sides, same semantics (parameter shapes may differ)
- 🟡 **Different shape** — semantics match, but calling convention/params/return types differ (language-paradigm differences)
- ❌ **Gap** — the TS SDK has it, Rust does not (to fill)
- ⏭️ **Architectural difference** — extensions/custom-ui system; not implemented in Rust

---

## 1. Session core (✅ aligned)

| TS SDK (`AgentSessionLike`) | Rust (`AgentSessionHandle`) | Notes |
|---|---|---|
| `prompt(text, options?)` | `prompt(input, on_event)` / `prompt_with_abort(input, signal, on_event)` | 🟡 Rust splits into with/without-abort variants; options (images/streamingBehavior) go through SessionOptions |
| `abort()` | `new_abort_handle()` + `AbortHandle.abort()` | 🟡 Rust requires a pre-built AbortHandle+Signal |
| `subscribe(listener)` | `subscribe(listener)` / `unsubscribe(id)` | ✅ Rust adds an explicit unsubscribe |
| `dispose()` | `into_inner()` | 🟡 Rust consumes the handle to take the inner session |
| `reload(options?)` | `continue_turn(on_event)` / `continue_turn_with_abort(...)` | 🟡 not fully equivalent |
| `setModel(model)` | `set_model(provider, model_id)` | 🟡 TS takes an object; Rust splits provider+model_id |
| `model` (readonly) | `model() -> (String, String)` | ✅ |
| `setThinkingLevel(level)` | `set_thinking_level(level)` | ✅ |
| `setSessionName(name)` | `set_session_name(name)` | ✅ |
| `compact(customInstructions?)` | `compact(on_event)` / `compact_with_instructions(instructions, on_event)` | ✅ fork added the instructions variant |
| `steer(text, images?)` | `steer(message)` | 🟡 Rust lacks the images param |
| `followUp(text, images?)` | `follow_up(message)` | 🟡 same |
| `sessionId` (readonly) | `session().session.lock().header.id` | 🟡 needs an async lock |
| `sessionFile` (readonly) | `session().session.lock()` reads the file path | 🟡 |
| `isStreaming` (readonly) | `state().await.is_streaming` | 🟡 needs await |
| `isCompacting` (readonly) | `state().await.is_compacting` | 🟡 |

## 2. Stats / bash / compaction (✅ filled by the fork)

| TS SDK | Rust | Notes |
|---|---|---|
| `getSessionStats()` | `get_session_stats()` | ✅ fork commit e178fb48 |
| `getLastAssistantText()` | `get_last_assistant_text()` | ✅ fork commit e178fb48 |
| `setAutoCompactionEnabled(b)` | `set_auto_compaction(b)` | ✅ fork commit e178fb48 |
| `compact(customInstructions?)` | `compact_with_instructions(...)` | ✅ fork commit e178fb48 |
| `executeBash(cmd, onChunk?, opts?)` | `bash(cmd, abort_rx)` | 🟡 fork commit 52d39dc3; different param shape (abort via oneshot, no onChunk) |
| `autoCompactionEnabled` (readonly) | none (set-only, no read) | ❌ missing getter |

## 3. Queue management (❌ gap)

The TS SDK exposes reading + clearing of the (steering/followUp) queue; pi's internal queue is private in Rust.

| TS SDK | Rust | Notes |
|---|---|---|
| `clearQueue()` | none (pi's internal queue is private) | ❌ moho-mate clears via a LoopState mirror |
| `getSteeringMessages()` | none | ❌ |
| `getFollowUpMessages()` | none | ❌ |
| `pendingMessageCount` (readonly) | none | ❌ |

## 4. Tool management (❌ gap)

Rust has a `ToolRegistry`, but no query/set methods are exposed on the handle.

| TS SDK | Rust | Notes |
|---|---|---|
| `getAllTools()` | no handle method (the underlying ToolRegistry has it) | ❌ |
| `getActiveToolNames()` | none | ❌ |
| `setActiveToolsByName(names)` | none (moho-mate approximates via set_system_prompt) | ❌ PARTIAL |

## 5. Navigation (❌ gap)

| TS SDK | Rust | Notes |
|---|---|---|
| `navigateTree(targetId, opts?)` | no handle method (the underlying Session.navigate_to exists) | ❌ moho-mate goes through session_mut().session.lock().navigate_to |

## 6. Bash helper state (❌ gap)

| TS SDK | Rust | Notes |
|---|---|---|
| `abortBash()` | none (bash abort is managed by the caller via a oneshot channel) | 🟡 different mechanism |
| `isBashRunning` (readonly) | none | ❌ missing state query |

## 7. Compaction / retry / context (❌ gap)

| TS SDK | Rust | Notes |
|---|---|---|
| `abortCompaction()` | none | ❌ |
| `setAutoRetryEnabled(b)` | none | ❌ retry state is RPC-subprocess-only (RpcSharedState) |
| `autoRetryEnabled` (readonly) | none | ❌ same |
| `getContextUsage()` | none | ❌ missing context usage query |

## 7.5. SessionManager (✅ aligned)

pi-web-rust wires this via `pi::sdk::SessionIndex` + `pi::sdk::SessionMeta` + `pi::sdk::build_session_context`.

| TS SDK | Rust | Status |
|---|---|---|
| `SessionManager.listAll()` → `SessionInfo[]` | `SessionIndex::new().list_sessions(None)` → `Vec<SessionMeta>` | ✅ SessionMeta carries first_message/parent_session_path/modified_ms |
| `SessionManager.open(path).getEntries()` | `Session::open(path)` + public `session.entries` field | ✅ |
| `buildSessionContext(entries, leafId, byId)` | `pi::sdk::build_session_context(entries, leaf_id, by_id)` → `SessionContextSnapshot` | ✅ free function (leaf→root via parentId + compaction cut-off) |
| `resolveModelScopeWithDiagnostics(patterns, modelRuntime)` | `pi::sdk::resolve_model_scope_with_diagnostics(patterns, registry, allow_missing)` → `(Vec<ScopedModel>, Vec<String>)` | ✅ returns structured diagnostics |
| `getAgentDir()` | `Config::global_dir()` | ✅ |

## 8. Settings / model runtime (✅ partially aligned)

| TS SDK | Rust | Status |
|---|---|---|
| `settingsManager` (readonly) | none | ❌ missing settings manager |
| `modelRuntime` (readonly) | `agent.model_registry()` / `agent.auth_storage()` | ✅ pub getters added |

## 9. Extension system (⏭️ architectural difference)

The TS SDK has a full extension system (pi extensions); Rust has another implementation that
also goes by "extensions" but with a different interface.
pi-web's skills/plugins/custom-ui all ride on this layer.

| TS SDK | Rust | Status |
|---|---|---|
| `extensionRunner` (readonly) | `extension_manager()` / `has_extensions()` | ⏭️ Rust has it but the interface differs |
| `promptTemplates` (readonly) | none | ⏭️ |
| `resourceLoader` (readonly) | none | ⏭️ |
| `bindExtensions?` | none | ⏭️ |

---

## Alignment progress

| Category | Total | ✅ aligned | ❌ gap |
|---|---|---|---|
| Session core | 16 | 16 | 0 |
| Stats/bash/compaction | 6 | 5 | 1 (autoCompactionEnabled getter) |
| Queue management | 4 | 0 | 4 |
| Tool management | 3 | 0 | 3 |
| Navigation | 1 | 0 | 1 |
| Bash helpers | 2 | 0 | 2 |
| Compaction/retry/context | 4 | 0 | 4 |
| Settings/model | 2 | 1 | 1 |
| Extensions (architectural) | 4 | — | ⏭️ skipped |
| **Total** | **42** | **22** | **16** |

**Aligned rate: 52%** (22/42, excluding architectural-difference items).

## Fork fill log (defims/pi_agent_rust)

| commit | methods | TS SDK aligned |
|---|---|---|
| `e178fb48` | get_session_stats / get_last_assistant_text / set_auto_compaction / compact_with_instructions | getSessionStats / getLastAssistantText / setAutoCompactionEnabled / compact |
| `52d39dc3` | bash | executeBash |
| `2595adae` | (macOS type fixes, not a method fill) | — |
| `baee8763` | (removed RPC-only methods absent from the TS SDK AgentSessionLike) | — |
| `86e8cac6` | SessionMeta extension (first_message/parent_session_path/modified_ms) + build_session_context free function + sdk exports SessionIndex/SessionMeta | SessionManager.listAll / buildSessionContext |
| `adfdc96c` | resolve_model_scope_with_diagnostics + AgentSession model_registry()/auth_storage() getters | resolveModelScopeWithDiagnostics / modelRuntime readonly |
| `36fdac58` | createAgentSessionServices / FromServices split | createAgentSessionServices / createAgentSessionFromServices |

## Upstream tracking flow

```bash
# 1. When earendil-works/pi cuts a release, update the AgentSessionLike interface
#    and check §1-9 of this document for added/changed methods

# 2. Fill the gaps in this repository (defims/pi_agent_rust)
# add methods to the AgentSessionHandle impl block in src/sdk.rs
cargo check --lib  # verify

# 3. push the fork
git push origin master:main

# 4. consumer moho-mate (host repo) updates the submodule + verifies
cd <moho-mate root>
git add pi-agent-rust
cargo check
```

## Related files

- TS SDK interface: upstream agegr/pi-web `lib/pi-types.ts` (`AgentSessionLike`; a copy exists in the consumer moho-mate's pi-web-rust submodule)
- Rust handle: `src/sdk.rs` (`AgentSessionHandle` impl)
- Rust engine core: `src/agent.rs` (`AgentSession`)
- RPC path (reference): `src/rpc.rs` (`RpcTransportClient`)
- Detailed signatures: consumer moho-mate's `docs/pi-sdk-probe-notes.md`
- 中文版: [sdk-mapping.zh-CN.md](./sdk-mapping.zh-CN.md)
