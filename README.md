# pi-palantir

Remote sessions for the [pi coding agent](https://github.com/earendil-works/pi) — run pi on an always-on machine, watch and steer it from a phone, tablet, or another PC, optionally by voice.

> A palantír is a seeing-stone: you look into it and see, and speak with, what is far away.

**Status: design only.** Nothing is implemented yet. This README is the architecture proposal; it will shrink into a normal project README as phases land.

---

## Table of contents

- [Goals / non-goals](#goals--non-goals)
- [What already exists upstream](#what-already-exists-upstream)
- [Architecture](#architecture)
- [Part A — upstream server completion](#part-a--upstream-server-completion)
- [Part B — the palantir package](#part-b--the-palantir-package)
- [Key decisions](#key-decisions)
- [Phasing](#phasing)
- [Open questions and risks](#open-questions-and-risks)
- [Prior art](#prior-art)
- [Why this name](#why-this-name)

---

## Goals / non-goals

**Goals**

1. Run pi sessions on an always-on machine (workstation, home server, VM); observe and control them from any device over Tailscale.
2. Web client that installs as an offline-capable PWA on iOS and Android — no app store.
3. Simple public-key login — the "GitHub public keys / Termius SSH ID" feel — plus zero-click login when Tailscale identity is available.
4. Voice interaction that is a conversation, not dictation: a realtime **voice agent** (OpenAI Realtime, ElevenLabs Conversational AI, xAI Grok Voice — BYOK) talks with the operator, and a **manager** (any frontier or open-weight text model) stands between that conversation and the sessions — it knows what every session is doing, answers questions about them, spins up read-only investigation sessions for questions that need exploring, and turns spoken intent into prompts, steers, and approvals. Results come back asynchronously, the way a human manager would report back.
5. Sessions are organised as **projects** (one per repository), the way the Claude Code and Codex desktop apps do it: several sessions per project, created fresh or forked from an existing session's state.
6. Build on pi's own `protocol` / `client` / `server` packages rather than inventing another extension-side bridge.

**Non-goals (v1)**

- Replacing the TUI. The daemon hosts sessions; the TUI attaches to the same sessions later ([phase 7](#phasing)).
- Multi-tenant or multi-user. One operator, many devices.
- Public-internet exposure. A private network (Tailscale) is assumed; a reverse-proxy story is documented, not built.
- A native app. PWA only.

---

## What already exists upstream

pi ships three packages that are exactly the right foundation, and they are further along than the third-party remote UIs assume. Everything below describes the latest stable release, `v0.84.2` (`914cf14`), which is also the base of the fork that carries Part A.

| Layer | Exists | Gap |
| --- | --- | --- |
| [`pi-protocol`](https://github.com/earendil-works/pi/tree/main/packages/protocol) | Protocol v1: CBOR frames, `hello` handshake, correlated request/response, `session_snapshot` / `session_progress` / `server_snapshot` events. Commands: `list`, `create`, `attach`, `detach`, `prompt`, `steer`, `abort`, `setModel`, `setThinking`. | No commands for tool approval / extension UI (`confirm`, `select`, `input`), session fork/switch, file read, git diff, or slash commands. No auth messages — by design, auth belongs to the transport. |
| [`pi-server`](https://github.com/earendil-works/pi/tree/main/packages/server) | `PiServer`, the `PiServerListener` / `ByteConnection` interfaces, and a Unix-socket listener. | No WebSocket listener. **No real `PiServerService`** — only the test stub in `src/testing/service.ts`. Nothing adapts `AgentSession` to `PiSessionRuntime`. |
| [`pi-client`](https://github.com/earendil-works/pi/tree/main/packages/client) | `PiClient` is transport-neutral with no Node imports, so it runs in a browser unchanged. Leases (`exclusive` / `shared`), snapshot subscription, `PiServerError` mapping. `coding-agent/src/client/remote-session.ts` adds a `RemoteSession` state machine and transcript reducer. | No WebSocket transport factory, no reconnect policy, no UI. |
| CLI | Experimental `pi server --listen unix:///path.sock` and `pi client --connect …` commands plus `pi --listen`, with a URL-style `TransportAddress` parser (`unix:` only today). | No non-Unix transport; no daemonisation story. |

The work therefore splits cleanly into **(A) finish the server side upstream** — generally useful and upstreamable — and **(B) this package**, which is the product.

---

## Architecture

```
┌──────────────── phone / tablet / PC (PWA) ─────────────────┐
│  UI (Preact + pi-client)                                    │
│   ├─ Projects · Sessions · Transcript · Composer · Approvals│
│   ├─ Voice agent client (WebRTC / WS to the voice provider) │
│   │      audio in/out ◄──► tool calls ──► manager           │
│   └─ Manager (text model, tools over the protocol)          │
│        runs here, or behind the proxy (see below)           │
│                 │ wss://host.tailnet/pi   (CBOR over TLS)    │
└─────────────────┼────────────────────────────────────────────┘
                  │  HTTP upgrade: Origin check · auth (Tailscale whois | device-key cookie)
┌─────────────────▼────────── pi-palantir daemon ──────────────┐
│  WebSocketListener ──► PiServer (pi-server)                  │
│                             │ PiServerService                │
│                      AgentSessionService  (Part A, upstream) │
│                             │                                │
│   project ~/src/app           project ~/src/lib              │
│   ├─ AgentSession #1 (work)   └─ AgentSession #3 (work)      │
│   └─ AgentSession #2 (read-only fork of #1 — investigation)  │
│      (coding-agent SDK: real tools, extensions, skills)      │
│                                                              │
│  Static PWA assets · /auth · /pair · /health                 │
└──────────────────────────────────────────────────────────────┘
┌──────── optional: palantir proxy (same process or elsewhere) ┐
│  holds voice/manager provider keys · mints ephemeral tokens  │
│  for the voice agent · can host the manager                  │
└──────────────────────────────────────────────────────────────┘
```

Three layers, each seeing only what it needs:

| Layer | Sees | Talks to |
| --- | --- | --- |
| **Voice agent** (realtime provider) | The conversation and a small tool set. **Never** transcript data. | The operator (audio), the manager (tool calls). |
| **Manager** (text model) | Compact digests of every session, the protocol. | The voice agent (tool results, announcements), the daemon (protocol commands). |
| **Daemon** (pi) | Projects, sessions, the coding models. | Clients over the protocol. Never the voice or manager keys. |

### Trust boundaries

- **Daemon host** holds credentials for the *coding* models (pi's normal `auth.json`). It may hold none at all — "pi on the server without model access" is a supported, **observe-only** configuration: sessions can be listed, attached, and read, but `prompt` / `steer` are rejected until a model is selected. How that state is encoded on the wire is specified under [no-model sessions](#no-model-sessions).
- **Project trust** is the host's, not the client's. A remote `create` pointing at a directory with project extensions goes through the same trust gate the CLI uses ([below](#project-trust)); the daemon never executes untrusted project code because a phone asked it to.
- **Voice and manager keys** live on a machine the operator controls — the client device (IndexedDB, used straight from the browser where a provider allows it) or the optional [proxy](#the-proxy), which the operator runs alongside the daemon or anywhere else. The daemon that hosts sessions never needs them; it may run on a shared host with no model access at all.
- Everything runs on the tailnet. The daemon refuses to bind to non-private interfaces without an explicit `--insecure-bind`.

---

## Part A — upstream server completion

### How Part A reaches pi

Part A is developed in the existing fork [`frycm/pi`](https://github.com/frycm/pi), checked out as a sibling of this repo (`../pi`). The fork is **always based on the latest stable pi release** — the `vX.Y.Z` tag, currently `v0.84.2` (`914cf14`) — never on upstream `main`. The rules:

- Part A is a small, self-contained patch series (service, protocol v2, transport registration) on a `palantir` branch in the fork, carried on top of the current stable tag. On every upstream release the branch is rebased onto the new tag; anything that no longer applies is fixed or dropped, never worked around.
- pi-palantir consumes the fork's workspace packages under their real scoped names, pointed at the sibling checkout during development — a `file:` specifier changes where npm reads a package, not its name, so the dependency keys are exactly:

  ```json
  "dependencies": {
    "@earendil-works/pi-protocol":     "file:../pi/packages/protocol",
    "@earendil-works/pi-server":       "file:../pi/packages/server",
    "@earendil-works/pi-client":       "file:../pi/packages/client",
    "@earendil-works/pi-coding-agent": "file:../pi/packages/coding-agent"
  }
  ```

  This README records which stable tag the fork branch is based on. No renamed or republished packages.
- Every patch is written to be upstreamable and is submitted upstream as soon as it is stable. A patch that lands upstream is deleted from the series at the next rebase, so the fork trends toward zero diff.
- Nothing in Part B depends on unreleased upstream behaviour: whatever upstream `main` gains between releases is picked up only when it ships in a stable tag. Where this document says "upstream has X" it refers to `v0.84.2`; the contracts cited here are identical at that tag and at current `main` (`c49906e`).

### `AgentSessionService implements PiServerService`

New, in `packages/coding-agent/src/server/agent-session-service.ts` upstream:

```ts
export class AgentSessionService implements PiServerService {
  constructor(opts: {
    agentDir: string;
    createRuntime: CreateAgentSessionRuntimeFactory;
    sessionStore: SessionManager;
  });

  listSessions(): Promise<SessionMetadata[]>; // SessionManager → id/createdAt/updatedAt/name/cwd
  listModels(): Promise<ModelMetadata[]>;     // ModelRegistry → protocol ModelMetadata
  createSession(o: CreateSessionOptions): Promise<PiSessionRuntime>; // after the project-trust gate
  openSession(id: string): Promise<PiSessionRuntime>;                // resume; PiServer owns the lifetime
}
```

`AgentSessionRuntime implements PiSessionRuntime` wraps an `AgentSession`:

- **`snapshot()`** — projects `AgentSession` state into a `SessionSnapshot`: `phase` from agent state, a **bounded** `transcript` window (below) through the same tree-to-flat mapping the HTML exporter uses, `queuedSteer`, and any pending UI requests.
- **`subscribe()`** — adapts `AgentSession` events (`message_start/update/end`, `tool_execution_*`, `turn_*`) into `{ type: "progress", progress: TranscriptProgress }`, and emits `{ type: "snapshot" }` on authoritative changes: turn end, model change, compaction, UI request raised or resolved.
- **Operation guards** — per operation, matching the upstream test runtime rather than a blanket "not idle ⇒ busy" rule, so steering and cancellation stay available exactly when they are needed:

  | Operation | Allowed when | Otherwise |
  | --- | --- | --- |
  | `prompt`, `setModel`, `setThinking` | `phase === "idle"` | `PiServerError("busy")` |
  | `steer` | a prompt is active (`phase !== "idle"`) | `PiServerError("busy", "no active prompt")` |
  | `abort` | a prompt is active | `PiServerError("busy", "no active prompt")` |
  | `ui_respond` | the request is pending | `stale_request` (see [pending UI requests](#pending-ui-requests)) |

- **`dispose()`** — shuts the `AgentSession` down, nothing more. `PiServer` already owns runtime lifetime: it holds one exclusive runtime per session ID and only calls `dispose()` once the last attachment is gone **and** the runtime is idle, so a phone dropping off Wi-Fi mid-turn does not kill the run — that property comes for free. Keeping an `AgentSession` alive across an idle gap (so a resumed session does not re-read the file and re-load extensions) is an upstream `PiServer` option (`idleRetentionMs`, or a `retain`/`release` hook on `LiveSessionManager`) and is **out of phase 0**. A refcount or timer hidden behind `dispose()` would either block reconnects behind the server's `disposing` state or leave a second wrapper racing the retained one, so the adapter must not do that.

#### Bounded transcript

The protocol's default frame ceiling is 16 MiB, and a long session's transcript can exceed that cumulatively even with per-result truncation. Bounding only the initial snapshot is not enough — a history page, an "untruncated" item, or an `item_finished` progress event carrying a large tool result can each blow the frame on its own. So the budget applies to **every message on every path**:

- **`MAX_MESSAGE_BYTES`** = 8 MiB encoded (half of `DEFAULT_MAX_FRAME_LENGTH`, leaving headroom for the envelope). The server encoder asserts it before sending any event or response; exceeding it is a server bug surfaced as `PiServerError("invalid_request", "message too large")` to the requester, never a dropped connection. Clients are held to the same budget for requests (see the image limits below).
- **`MAX_PART_BYTES`** = 256 KiB per content part (text, tool output, diff). Any content part larger than that is carried **truncated** — first 256 KiB plus `{ truncated: true, totalBytes }` — wherever an item appears: the snapshot tail, `history` pages, and `item_finished` / `item_updated` progress events. Nothing in the live stream is ever sent whole if it is big.
- **Snapshot tail** — the most recent items up to 1 MiB encoded (after part truncation), plus `historyCursor` marking where the window starts.

| Command | Purpose |
| --- | --- |
| `history { sessionId, before: cursor, limit? }` | Page backwards. `limit` is a ceiling on item count; the real bound is encoded bytes — the server stops adding items at 4 MiB and returns `next` so the client continues. Parts are truncated exactly as in the snapshot. |
| `read_item { sessionId, itemId, partIndex, offset, length? }` | Fetch a **byte range** of one content part, `length` capped at 4 MiB; the response carries `{ bytes, offset, totalBytes, next? }`. Full content is obtained by range continuation, never in one message. This is what the manager's `read_more` calls (with a small `length`). |

Raising the frame limit is explicitly **not** the answer; the suite includes a test that a 50 MiB tool result round-trips through snapshot, history, progress, and `read_item` without any message above `MAX_MESSAGE_BYTES`.

#### No-model sessions

`AgentSession.model` may be undefined — no authenticated model, or a resumed session whose persisted model lost its credentials — while v1 `SessionSnapshot.model` is required. The contract:

- In **protocol v2** `model` becomes optional and the snapshot gains `modelState: "ready" | "unavailable"` with a human-readable `modelStateReason`. Such a session is observe-only: `prompt`/`steer` are rejected with `PiServerError("invalid_request", "no model")` until `setModel` succeeds with an authenticated model from `listModels()`.
- For a connection negotiated at **v1** the rule is "never emit a snapshot without `model`":
  - `create` with no authenticated model (no `model` given and no usable default) is **rejected before any runtime exists**, with `PiServerError("invalid_request", "no authenticated model; pass model or use protocol v2")`. The v1 `create` result requires a `SessionSnapshot`, so there is no schema-valid alternative.
  - `list` keeps no-model sessions visible — `SessionMetadata` has no model field — so the operator can see them and fix credentials on the host.
  - `attach` to a no-model session is rejected with the same `invalid_request`; `open` therefore never produces a v1 snapshot either.
- Tests, in phase 0 (which negotiates v1 only): v1 `create` with no credentials → `invalid_request`, no session file created; v1 `list` still shows a persisted no-model session; v1 `attach` to it → `invalid_request`; and, once v2 lands, the same two sessions attach on v2 with `modelState: "unavailable"`.

#### Project trust

The CLI resolves project trust through `ProjectTrustStore` / `resolveProjectTrusted()` **before** `createAgentSessionServices()`, and non-interactive modes fail closed when no decision exists. The service reuses that gate rather than bypassing it:

- `createSession({ cwd })` and `openSession()` call the same resolver with the daemon's `agentDir` trust store. A recorded decision (`trust.json`) is honoured as-is.
- **Phases 0–1** are non-interactive: with trust-requiring resources present and no decision, the default is to **reject** (`PiServerError("invalid_request", "project not trusted: <cwd>")`); an opt-in `untrustedProjects: "load-without-protected-resources"` service option loads the session with project extensions/skills/prompts withheld, and the snapshot says so (`projectTrusted: false`).
- **Recording trust persistently.** Upstream has no `--trust` flag, and `--approve` is a run-only override — `resolveProjectTrusted()` returns it without touching `ProjectTrustStore` — so it cannot pre-seed the daemon. The two paths that persist are: the interactive TUI prompt (`pi` in that cwd, answer *trust*), and a new `ProjectTrustStore`-backed subcommand in Part A, `pi trust <cwd>` / `pi trust --revoke <cwd>`, which writes the same `trust.json` entry the TUI would. The daemon reads that file, so either path takes effect on the next `create`.
- **Remote decision (phase 2)** is a **server-level** flow, because trust must be decided before `createAgentSessionServices()` runs and therefore before any session exists that could own a `ui_request`:
  1. `create` on a cwd with trust-requiring resources and no decision fails with the v2 error `project_untrusted`, whose payload lists `cwd` and the resource kinds found (extensions, skills, prompts, settings). v1 connections get `invalid_request`.
  2. The client shows the same text the TUI shows and, on *trust*, sends the v2 server command `trust_project { cwd, decision: "trust" | "reject" }`. The server writes through `ProjectTrustStore` — persistent, identical to the TUI's record — and answers `{ cwd, trusted }`.
  3. The client re-issues `create`. There is no pending or half-loaded session to promote; the only state is the trust file.

  `trust_project` is gated by the daemon option `remoteTrustDecisions` (default `true` for device-key and Tailscale identities, since both resolve to the single operator); with it off, step 2 answers `not_implemented` and the operator uses `pi trust`.

### Protocol additions (v2, additive)

#### Version negotiation

Today's handshake accepts only `version === PROTOCOL_VERSION`, and the strict v1 codec rejects unknown message shapes — so "gate it on the handshake" is not enough by itself: bumping the constant rejects v1 clients, and sending a v2 event to a v1 decoder breaks it. The negotiation is therefore explicit:

- **Pre-negotiation decoder.** The first frame is decoded by a tiny permissive decoder that understands two hello shapes: the unchanged legacy `{ type: "hello", version: 1 }` (which current `PiClient`s send, validated by a strict schema) and the new `{ type: "hello", versions: [1, 2] }`. Only after the first frame does the server pick a codec.
- **Legacy hello** (`version: N`): if `N === 1` the server answers the exact v1 `hello` reply (`version: 1` literal, `connectionId`, `snapshot`); otherwise it answers the exact v1 `hello_error` with the existing error code `"version"`. No new error code is introduced, because legacy clients cannot decode one.
- **Negotiating hello** (`versions: [...]`): the server replies with the highest version both support in `version`, or `hello_error` with code `"version"` when the sets do not intersect. `isSupportedProtocolVersion()` becomes a range check used only on this path.
- The server instantiates a **per-connection codec** for the negotiated version and keeps a per-connection `version` in `ConnectionState`.
- **Compatibility test**: the unchanged `PiClient` from `v0.84.2` (legacy hello, strict v1 decoder) connects to the v2 server, lists, creates, attaches, prompts, and steers through a full turn — while a v2 client is attached to the same session, a `ui_request` is raised and answered, an extension calls `setStatus` / `setWidget` / `setTitle` / `set_editor_text`, and a tool returns a 50 MiB result — without a single decode error on the v1 client, and with the v2 client seeing `ui`, `turnOwner`, and `truncated` metadata in the same session.
- **v1 projection**: the v1 encoder is a **whitelist**, not a strip list — every outgoing message is projected through the unchanged `v0.84.2` strict schemas and validated against them before framing, so any v2 addition, present or future, is removed mechanically rather than by remembering to list it. Concretely, for a v1 connection:
  - events `ui_request`, `ui_resolved`, and the `ui_state` progress event are **suppressed** entirely;
  - snapshot fields `modelState`, `modelStateReason`, `pendingUiRequests`, `historyCursor`, `projectTrusted`, `ui`, and `turnOwner` are projected out;
  - transcript content parts lose `{ truncated, totalBytes }` in snapshots, `history` is v2-only, and `item_started` / `item_updated` / `item_finished` progress items are projected to the v1 item shape — the *content* stays truncated to `MAX_PART_BYTES` (a v1 client gets the first 256 KiB, which is also what the frame ceiling demands), only the metadata is dropped;
  - `hello.snapshot` (the server snapshot) goes through the same projection.

  A projection failure (v2 message with no v1 equivalent) is a server bug surfaced to the v1 requester as `invalid_request`; it never reaches the client's decoder. v1 clients see a v1 session and simply cannot answer approvals.
- Every v2 command and event is additive; nothing in v1 changes meaning.

#### Commands and events

| Command / event | Purpose |
| --- | --- |
| `ui_request` event · `ui_respond` command · `ui_resolved` event | Surface extension UI — `confirm`, `select`, `input`, `editor`, and tool-approval prompts — to remote clients, mirroring the `extension_ui_request` / `extension_ui_response` contract in pi's `docs/rpc.md`, including the fire-and-forget methods `notify`, `setStatus`, `setWidget`, `setTitle`, `set_editor_text` (sent with `expectsResponse: false`). **Non-negotiable**: without it a remote client cannot answer a permission prompt. Semantics under [pending UI requests](#pending-ui-requests). |
| `history`, `read_item` | Byte-bounded transcript paging and ranged content-part fetch (see [bounded transcript](#bounded-transcript)). New v2 error codes used by this section and the ones below: `project_untrusted`, `stale_request`, `stale_revision`; v1 connections receive `invalid_request` in their place. |
| `create { cwd, readOnly?, forkFrom?: { sessionId, itemId } }` (v2 extension) | Creates a session in a project (`cwd`). `forkFrom` copies the source session's history up to `itemId` — the same operation as `fork` below, exposed at creation so the manager can open an investigation in one call. `readOnly: true` gives the session a read-only tool allowlist (read, grep, ls, git read commands; no write, edit, or side-effecting bash), enforced by the daemon rather than promised by a prompt; the snapshot carries `readOnly: true` and the session is listed as ephemeral (hidden from the default session list, auto-archived after a configurable idle period). Read-only sessions share the project's working tree, which is safe precisely because they cannot write. Worktree-backed sessions that *do* work in parallel are a later phase. |
| `server_snapshot` grouping (v2) | `SessionMetadata` gains `project` (the git toplevel of `cwd`, or `cwd` itself outside a repository), `readOnly`, and `forkedFrom`. Projects are a grouping key, not a separate entity — nothing is created or stored for them. |
| `fork { sessionId, itemId }`, `switch { sessionId, targetSessionId }` | Session-tree navigation as **server-level** operations. `LiveSessionManager` keys a runtime by its original ID and rejects a snapshot whose ID changes, so neither can mutate the runtime in place. Both return `{ sessionId }` of the destination; the client then performs an explicit `detach` / `attach` transition, and the source runtime is left untouched. |
| `read_file { path, range }`, `diff { ref? }` | Read-only views for review on a phone. The server enforces cwd-rooted paths and a size cap. |
| `list_commands` | Discovery metadata for the `/` palette: name, description, source (builtin / extension / skill / template). Execution stays on `prompt` with `source: "rpc"` — `AgentSession.prompt()` already recognises extension commands and expands skills and templates, so a separate `slash` command would duplicate routing and drift from the TUI. |
| `session_name` | Equivalent of `/name`. |
| `prompt` / `steer` content (v2) | `{ content: [ { type: "text", text } \| { type: "image", mimeType, data } ] }` alongside v1's `{ text }`. `data` is a CBOR **byte string**, not base64, so encoded size ≈ file size. Limits are chosen so the largest legal request is encodable under `MAX_MESSAGE_BYTES` (8 MiB): `image/png`, `image/jpeg`, `image/webp`, `image/gif`; ≤ 2 MiB per image, ≤ 3 images per message (≤ 6 MiB of image bytes), text ≤ 1 MiB. The server rejects anything over with `invalid_request` before touching `AgentSession`; the PWA re-encodes camera and pasted images to fit (longest edge 2048 px, JPEG q≈0.85 — a phone photo lands well under 1 MiB) and refuses the rest. Larger attachments would need a chunked upload plus `image_ref`, which is deliberately not in v1. This is what lets the composer's camera and paste actually reach `AgentSession`. |
| `trust_project { cwd, decision }` (server-level) | Records a persistent project-trust decision; see [project trust](#project-trust). |
| `turnOwner` in `SessionSnapshot` (v2) | Server-side turn ownership; see [turn ownership](#turn-ownership). |
| Mutation envelope (v2) | Every mutating command (`prompt`, `steer`, `abort`, `ui_respond`, `setModel`, `setThinking`, `fork`, `switch`, `session_name`) carries a client-generated `requestId` (UUID) and an `expectedRevision`. The server deduplicates `requestId` per session for a retention window (default 10 minutes, returning the original result) and rejects a mismatched `expectedRevision` with `stale_revision`. See [offline and reconnect](#offline-and-reconnect). |

Terminal access and git mutation stay out of scope.

#### Pending UI requests

An event alone is lossy — a reconnecting client gets an authoritative snapshot but would never learn about the approval it missed — and `PiServer` broadcasts events to every attachment, so two devices can race to answer. Hence:

- **In the snapshot.** `SessionSnapshot.pendingUiRequests: [{ id, method, payload, createdAt, expiresAt? }]`. Pending requests belong to the *session*, not to the connection that happened to be attached when they were raised, so they survive client disconnects and are visible to anything that attaches later.
- **First response wins, atomically.** The runtime resolves a request exactly once. A second `ui_respond` for the same `id` — from any device — returns `PiServerError("stale_request", "already answered")`; an `id` that never existed or has expired returns `stale_request` too. Every attachment receives `ui_resolved { id, by: connectionId }` so open approval cards close everywhere.
- **Expiry and cancellation.** Requests carry the extension's timeout when it has one. `abort` cancels all pending requests for the session; the extension sees the same cancellation it would in the TUI. Requests are never transferred or cancelled merely because a connection dropped.
- **Fire-and-forget methods** are delivered as `ui_request` with `expectsResponse: false` and accept no response, but only `notify` is genuinely transient. `setStatus`, `setWidget`, `setTitle`, and `set_editor_text` change visible UI state, so their *current* values live in the snapshot as `SessionSnapshot.ui: { status: Record<key, text>, widgets: Record<key, lines>, title?: string, editorText?: string }` — exactly the state the TUI keeps for them. A call mutates that map and emits a `ui_state` progress event with the delta; a reconnecting or second client gets the current values from its snapshot and never needs a replay. `notify` is event-only and is dropped for clients that are not attached at the time.

#### Turn ownership

Upstream's `exclusive` / `shared` leases are maps inside one `PiClient`; the server lets every attached connection drive the single runtime, so a phone and a TUI can each believe they hold an exclusive lease. Ownership is therefore made **server-wide** in `LiveSessionManager`:

- `SessionSnapshot.turnOwner?: { connectionId, since }` (v2). It is set **atomically** by the `prompt` that starts a turn and cleared when the phase returns to `idle`.
- While a turn is owned: `prompt` from anyone is `busy` (as today); `steer` is accepted from the owner and rejected for other connections with `session_locked`; `abort` is accepted from **any** attached connection — every surface is the same operator, and an emergency stop must not depend on which device started the turn. `ui_respond` keeps first-response-wins across all attachments.
- If the owning connection disconnects mid-turn, the run continues and ownership becomes **orphaned** (`turnOwner` cleared, phase still busy): the next `steer` from any attachment claims ownership atomically for the remainder of the turn. A client that reconnects with the same device identity does not get its ownership back automatically; it steers and re-claims like anyone else.
- Client-local leases stay as a client-side convenience for a single client's own components; they grant nothing on the server.
- Phase 7 (TUI as client) relies on this and nothing else to serialise surfaces; open question 3 is resolved by it.

### CLI: build on `pi server --listen`

Upstream already ships experimental `pi server` / `pi client` commands and `pi --listen`, with URL-style transport addresses (`--listen unix:///tmp/pi.sock`) parsed by `transport-address.ts`. Part A does **not** introduce a competing `pi serve` syntax; it completes what is there:

- `pi server --listen unix:///…` runs `AgentSessionService` behind the Unix listener (phase 0).
- `TransportAddress` gains a registration point so a package can contribute a transport: `pi-palantir` registers `ws://` / `wss://` (`pi server --listen wss://127.0.0.1:7314/pi`), or — simpler for v1 and what phase 1 actually does — the palantir daemon embeds `PiServer` + `AgentSessionService` directly and owns its own process.

---

## Part B — the palantir package

```
pi-palantir/
├─ package.json     # "pi": { "extensions": ["./extension/index.ts"] }
├─ extension/       # TUI side: /palantir start|stop|status|pair, QR, status-line segment
├─ daemon/          # node: WebSocket listener, auth, static assets
├─ proxy/           # optional: provider keys, ephemeral voice tokens, hosted manager
├─ manager/         # the manager agent: digests, tools, announcement queue (runs in web/ or proxy/)
├─ web/             # PWA (Vite). No Node imports. pi-client + the transcript reducer.
└─ shared/          # auth token format, manager tool schema, voice-agent tool schema
```

### WebSocket listener

`createWebSocketListener(opts): PiServerListener` — `ws` on Node, each socket becoming a `ByteConnection` whose `send` honours `bufferedAmount` backpressure and whose `maxPendingBytes` overflow closes the socket, matching the Unix listener's behaviour. Authentication runs in the **HTTP upgrade handler**, before the connection reaches `PiServer` — exactly what the `pi-server` README prescribes.

The upgrade handler runs these checks in order, and any failure is a plain HTTP error before a socket exists:

1. **Origin.** Browsers may open cross-origin WebSockets and the handshake is not protected by CORS, so a malicious page visited by an allowed user could otherwise drive the daemon with that user's tailnet identity. The `Origin` header must equal the daemon's configured public origin exactly (`https://<hostname>`, derived from `--hostname`; there is exactly one). A missing `Origin` is accepted **only** on the native-client path (no cookie; device-key signature over the server nonce carried in `Sec-WebSocket-Protocol`), never for Tailscale zero-click.
2. **Authentication** — one of the two mechanisms below.
3. Hand-off to `PiServer` with the resolved operator identity attached to the connection.

### Authentication

Two mechanisms, tried in order. Both resolve to an operator identity; anything else is a 401 on upgrade.

1. **Tailscale identity — zero-click.** With the daemon bound to the Tailscale interface, the upgrade handler asks the Tailscale LocalAPI `whois` for the peer IP and allows the connection when `UserProfile.LoginName` is in `allowedLogins`. No token, no pairing: the phone just opens the URL. Being IP-based rather than cookie-based, it survives PWA relaunch. `whois` authenticates the *network peer*, not the page that opened the socket — which is why the Origin check above is mandatory on this path. When TLS is terminated by `tailscale serve`, the identity arrives as `Tailscale-User-Login` headers instead; both are accepted.
2. **Device keys — the SSH-ID feel.** The browser generates a non-extractable Ed25519 keypair via WebCrypto and stores it in IndexedDB. Pairing: `/palantir pair` in the TUI shows a QR code and short code carrying a one-time secret; the PWA posts `{ pubkey, deviceName, pairingSecret }` to `/pair`; the daemon appends the key to `~/.pi/palantir/authorized_keys` in OpenSSH format (`ssh-ed25519 AAAA… phone-martin`). Login is an **HTTP bootstrap**, because a browser's `WebSocket` constructor takes only a URL and subprotocols and cannot set an `Authorization` header:
   - `GET /auth/challenge` → `{ nonce, host, issuedAt }` (single use, 60 s).
   - `POST /auth/login` (same origin) with `{ pubkey, signature }` over `nonce ‖ host ‖ issuedAt`.
   - On success the daemon sets a `Secure; HttpOnly; SameSite=Strict; Path=/` cookie (30 days). Its value is an HMAC-signed token binding the key's fingerprint and the daemon's **auth epoch**; the upgrade handler re-validates it on every connection against the current `authorized_keys` and epoch, so `/palantir revoke <name>` (which bumps the epoch for that key, or globally) invalidates already-issued cookies immediately rather than in 30 days.

**Non-tailnet exposure (documented, not built in v1).** `tailscale serve` endpoints and MagicDNS certificates exist only inside the tailnet, so device-key auth alone does not make the daemon reachable from a LAN browser. For that the operator provides the contract the daemon expects: `--bind <lan-ip>:443 --hostname <name that resolves on the LAN> --tls-cert/--tls-key` with a certificate the phone trusts (a private CA installed on the device, or a public name with a DNS-01 Let's Encrypt certificate), **or** a reverse proxy terminating TLS for that hostname and forwarding the upgrade with `X-Forwarded-For` set — the daemon then trusts only that proxy's address. In either case the allowed Origin is `https://<hostname>`, Tailscale zero-click is disabled for non-tailnet peers (`whois` is only consulted for CGNAT `100.64/10` sources), and only device-key logins are accepted. Without trusted HTTPS the PWA, WebCrypto, mic, and Push do not work, so a plain `http://` LAN mode is refused rather than degraded.

`/palantir authorize github:<user>` imports keys from `https://github.com/<user>.keys`. Those serve SSH-capable desktop clients — a browser cannot use an existing private key, which is why browsers get their own generated device key. Revocation is `/palantir revoke <name>` or editing `authorized_keys`.

### Web client (PWA)

- **Stack** — Vite and Preact, small enough to stay quick on a phone; `pi-client` for the protocol and the upstream `RemoteSession` plus transcript reducer for state. Markdown and diff rendering; no terminal emulator.
- **Transport** — WebSocket binary frames, wrapped in an exponential-backoff reconnect loop around `PiClient.reconnect()`, since the client deliberately does not reconnect on its own. Reconnect re-attaches previously attached sessions; snapshots are authoritative for *state*, so no event replay is needed — pending approvals arrive in `pendingUiRequests`, older history through `history`.
- **Screens** — Projects (one per repository, sessions beneath with phase badge and last activity; ephemeral investigation sessions collapsed by default) · Session (transcript tail with collapsible tool calls and "load earlier" paging, queued steers, approval cards from `pendingUiRequests`) · New session (fresh or forked from an item, in the project's checkout) · Composer (text, image paste and camera via the v2 image content parts, `/` palette from `list_commands`) · Manager (text chat with the manager — the same agent voice uses, usable without voice) · Settings (device keys, voice provider, manager model, proxy).
- <a id="offline-and-reconnect"></a>**Offline and reconnect** — a service worker precaches the shell, and the last snapshot per session is persisted to IndexedDB, so opening the app with no connectivity still shows state. A prompt composed offline is **queued, not auto-replayed**: every mutation carries a `requestId` and the `expectedRevision` the user was looking at when they wrote it. On reconnect the client first receives the fresh snapshot; if the revision still matches, it sends the queued mutation (the server's `requestId` dedupe makes the retry safe even if the original was accepted just before the drop); if the session has moved on — another device prompted, a turn finished — the draft stays in the composer marked *written against an older state* for the user to send or discard. Nothing else is meaningfully offline.
- **Notifications** — Web Push when a session raises a `ui_request` or finishes a turn while the app is backgrounded. VAPID keys are generated on first daemon run. iOS supports Web Push for installed PWAs from 16.4.

### Voice

Voice is a conversation with the **manager**, not a way to type prompts. The operator talks to a realtime voice agent; the voice agent talks to the manager through tool calls; the manager talks to the sessions through the protocol. Session data never reaches the voice agent.

#### Voice agent

Three realtime speech-to-speech providers, bring-your-own-key, selectable in Settings. Surveyed August 2026; all three support client-side tool calling, browser connections via short-lived tokens, and injecting text into a running conversation — the three things the manager needs. Prices are list, per provider pages, and change often.

| Provider | Model | Browser transport | Token | Tools | Inject an announcement | Rough cost |
| --- | --- | --- | --- | --- | --- | --- |
| **OpenAI Realtime** | `gpt-realtime-2.1` (flagship); `gpt-realtime-mini` for cost | WebRTC (also WebSocket, SIP) | `POST /v1/realtime/client_secrets`, minted by the proxy | Client-side function calling over the data channel; MCP servers | `conversation.item.create` with a text item, then `response.create` | $32 / $64 per 1M audio in/out tokens (≈ $0.30–0.40/min); mini ≈ $0.06–0.15/min |
| **ElevenLabs Agents** (Conversational AI) | Agent configured in ElevenLabs; LLM is the operator's choice there — hosted Claude, GPT-5, Gemini, Qwen, or a custom OpenAI-compatible endpoint | WebRTC or WebSocket via the JS SDK | Conversation token from the agents API, minted by the proxy | Client tools (run in the SDK), webhook tools, MCP | `sendContextualUpdate` (invisible to the user, visible to the agent) or `sendUserMessage` (forces a turn) | ≈ $0.08–0.10/min of conversation plus LLM tokens |
| **xAI Grok Voice** | `grok-voice-latest` (= `grok-voice-think-fast-2.0`) | **WebSocket only** — no WebRTC; the browser sends Opus/PCM frames itself | Ephemeral `xai-client-secret.…` in the WebSocket protocol header, minted by the proxy | Client-side function tools, MCP | `conversation.item.create` with a text message | $0.08/min |

Recommended order:

1. **OpenAI Realtime first.** WebRTC from the browser is the least work on a phone (the browser handles capture, playback, echo cancellation, and codec), tool calling and mid-conversation injection are the most mature, and the whole agent — prompt and tools — is defined by palantir at connect time.
2. **ElevenLabs second**, for voice quality and for operators who want Claude behind the voice. The trade-off: prompt, tools, and LLM are configured in the ElevenLabs agent, not by palantir, so palantir ships a reference agent definition and the operator applies it. The proxy holds only the ElevenLabs key; xAI is not among ElevenLabs' hosted LLMs, and a custom endpoint must be OpenAI-compatible.
3. **xAI third.** Cheapest and the only one that exposes "think" models for voice, but WebSocket-only means palantir owns audio capture and Opus encoding in the browser — more code, and weaker on iOS Safari.

The voice agent's system prompt describes the manager and its tools; it does **not** contain session content. Its tool set is small and identical across providers, executed client-side and forwarded to the manager:

```
status(sessionId?)                 → what is / was happening (answered from the manager's cached digest — fast)
ask(question, sessionId?)          → a question that needs the manager to read or investigate (async, returns a ticket)
act(intent, sessionId?)            → "tell it to also update the tests", "approve", "stop" (confirmed aloud unless "act immediately" is on)
new_session({ project, forkFrom? }) → start a working session
```

Because the tool set is the contract, providers are interchangeable, and the cascade alternative (STT + text model + TTS) remains possible as a fourth, cheaper "provider" behind the same tools — it is deliberately not in v1.

#### The manager

A text-model agent — Anthropic, OpenAI, xAI, or an open-weight model through any OpenAI-compatible endpoint, BYOK — that is the single middle-man between the operator and every session. It runs in the browser by default, or behind the proxy when the operator prefers keys server-side.

```
State it maintains (continuously, from session_progress — not on demand):
  · per session: compact digest — last N transcript items, tool calls collapsed to
    name + status + one-line result, current phase, pending ui_request, turn owner
  · announcement queue — things the operator has not heard yet

Tools over the protocol:
  prompt(text) / steer(text)               → protocol commands on a session
  respond_ui(requestId, answer)            → approvals and selects
  abort()
  read_more(sessionId, itemId)             → pull a tool result or diff into context on demand (via read_item)
  investigate(question, { project, forkFrom? })
                                           → create a read-only session (fresh in the project, or forked from
                                             an item of an existing session), prompt it with the question,
                                             and report back when its turn ends
  list_projects() / list_sessions(project)
```

- **Async by design.** `ask` and `investigate` return immediately ("I've asked — I'll tell you"); the operator keeps talking about something else. When the investigation's turn ends the manager reads its result, condenses it, and pushes an announcement to the voice agent. Turn end and new `ui_request` on any session feed the same queue, so there is exactly one mechanism for "something to tell the operator", with three sources.
- **Interrupt policy.** Announcements are delivered when the operator pauses, never mid-sentence; a `ui_request` is the only source allowed to pre-empt, and only after a configurable silence.
- **Digest, not full transcript** — mobile context and cost both matter; `read_more` fetches detail only when the conversation needs it ("what did the test output actually say?").
- **Voice-shaped output** — short spoken sentences, no code recitation unless asked, file paths spoken as basenames. The manager writes for the ear; the voice agent only repeats.
- **Text is first-class.** The Manager screen in the PWA is the same agent over a text box. It ships before voice and is how the manager is tested.
- **Scope per conversation.** The manager's own context lives for one voice call or chat tab; the session digests persist. Manager memory across calls is a later phase.

#### Investigation sessions

Questions the manager cannot answer from digests — "why is the migration failing?", "what does the scheduler actually do?" — get a session of their own so they never compete with the working session's queue or lease:

- **Fresh** in the project (`create { cwd, readOnly: true }`) when the question is about the code as it is on disk.
- **Forked** from a session item (`forkFrom`) when the question is about what a session did or knows. A fork sees history up to that item; it does not see the source session's in-flight turn.
- Always **read-only**, enforced daemon-side. That is what makes sharing the working tree safe. Parallel sessions that *write* are a later phase and will need worktrees.
- **Ephemeral**: hidden from the default session list, auto-archived after idle. The manager keeps the answer; the session is disposable.

#### The proxy

An optional component (`proxy/`) the operator runs where they like — in the daemon process, on the same host, or anywhere reachable from the PWA. It is the place for keys that should not live on a phone and for the pieces a browser cannot do:

- holds the voice provider keys and mints the short-lived tokens the voice agent needs to connect from the browser (OpenAI client secret, ElevenLabs conversation token, xAI ephemeral token) — **required** for voice, since none of the three providers should be given a long-lived key from a browser;
- optionally holds the manager's model key and hosts the manager, so the PWA is a thin client and several devices share one manager;
- stores nothing else, keeps no transcripts, and is authenticated the same way as the daemon (device key or Tailscale identity).

Without the proxy the PWA still does everything except voice; with the manager key on the device, the Manager text chat works proxy-less.

### Extension (TUI side)

Deliberately thin: `/palantir start|stop|status`, `/palantir pair` (QR overlay), `/palantir authorize <key|github:user>`, `/palantir revoke <name>`, and a status-line segment showing connected devices. The daemon is a separate process with launchd and systemd unit templates, so it outlives any one terminal.

---

## Key decisions

| Decision | Rationale |
| --- | --- |
| The daemon hosts sessions; the TUI attaches as a client later | Otherwise "remote" dies with the terminal. v1 can coexist: daemon-hosted and TUI-hosted sessions are separate session files, and the web UI sees only the former. |
| Auth in the HTTP upgrade, not in the protocol | Matches pi-server's stated design and keeps the protocol runtime-neutral. |
| Voice and manager keys are BYOK and never on the session host | The host may be a shared server; keys for *my* voice live on *my* phone or on a proxy *I* run. The daemon can run without model access while the operator still has a manager. |
| The voice agent never sees session data | Keeps its context tiny, makes the three voice providers interchangeable, and keeps transcripts away from whichever vendor renders the voice. The manager is the only component that reads sessions. |
| One manager, many sessions, async replies | A manager that reports back later — instead of a voice loop that blocks on one session — is what lets the operator keep talking while sessions work. Turn end, approvals, and investigation results all flow through one announcement queue. |
| Investigation sessions are read-only and ephemeral | Parallel exploration without worktrees, lease contention, or list clutter. Writing in parallel is a later phase. |
| PWA, not native | iOS Web Push and standalone mode are good enough since 16.4; no store friction; one codebase. The native remote apps in this space pay for that choice in maintenance. |
| No terminal or git UI in v1 | Stay inside the protocol's session abstraction. `read_file` and `diff` cover review-on-phone. |
| Protocol additions are additive v2, with explicit version negotiation | `ui_request` is required for real use; everything else can slip a phase. The handshake advertises a version range and the server filters v2 shapes off v1 connections, so v1 clients never see a frame they cannot decode. |

---

## Phasing

| Phase | Deliverable | Done when |
| --- | --- | --- |
| 0 | `AgentSessionService` behind `pi server --listen unix:///…` (Part A): bounded snapshots, per-operation guards, no-model encoding, project-trust gate (reject by default) | `PiClient` over a Unix socket can create, attach to, and prompt a real session; the upstream `testing/` suite passes against it, plus tests for no-model create/open and untrusted-cwd rejection. Upstreamable PR. |
| 1 | WebSocket listener with Origin check, **TLS**, Tailscale auth, minimal transcript view | Open `https://<host>.<tailnet>.ts.net` on a phone over a trusted certificate — service worker, WebCrypto, mic, and Push all require a secure context — and watch a live session. TLS ships in one of two modes: `tailscale serve` in front of a loopback-bound daemon (recommended: Tailscale provisions and renews the certificate and forwards identity headers), or `--tls-cert/--tls-key` from `tailscale cert`, with the daemon warning at startup when the certificate is within 14 days of expiry and refusing to start once it has expired. The served hostname is the MagicDNS name; the raw `100.x` address is not served. |
| 2 | Protocol v2 negotiation, `ui_request` approvals with `pendingUiRequests`, composer with image parts (camera/paste), mutation envelope with `requestId`/`expectedRevision`, PWA install, push | Drive a full session from a phone, including tool approvals and sending a photo, with two devices attached and no double-answered approval. |
| 3 | Device-key pairing, `authorized_keys`, revocation | A tailnet device whose Tailscale login is **not** in `allowedLogins` (a shared node, or a second account) pairs by QR and drives a session through the phase-1 endpoint; revoking its key closes its connection within one upgrade. Reachability and TLS are unchanged from phase 1 — device keys are authentication only. |
| 4 | Projects and investigation sessions (Part A): `create` with `readOnly` / `forkFrom`, project grouping in `server_snapshot`, read-only tool allowlist, ephemeral archiving | From the PWA, fork a running session into a read-only sibling, ask it a question, get the answer, and see the working session untouched. A read-only session attempting a write is rejected by the daemon. |
| 5 | Manager over text — Anthropic first, then OpenAI, xAI, OpenAI-compatible — with digests, `investigate`, and the announcement queue; Manager screen in the PWA | "What's it doing?", "approve", "also update the tests", "why does the build fail?" — typed; the last one answered later by an investigation session while the chat continues. |
| 6 | Voice agents — OpenAI Realtime first, then ElevenLabs Conversational AI and xAI Grok Voice — plus the proxy (ephemeral tokens, hosted manager) | The same four tools by voice; an announcement arrives mid-conversation without cutting the operator off. |
| 7 | TUI attaches to daemon sessions | One session visible in the terminal and on the phone at once. |

---

## Open questions and risks

1. **Transcript projection fidelity.** `SessionSnapshot.transcript` is a flattened view of a tree-shaped session; compaction and branches need a defined mapping. The HTML exporter already has one — reuse it rather than writing a second.
2. **TUI-as-client.** The TUI owns `AgentSession` in-process today. Turning it into a `PiClient` of the daemon is the largest refactor here; the existence of `coding-agent/src/client/remote-session.ts` upstream suggests that direction is already intended. Track upstream before building it.
3. **Lease semantics.** Resolved by server-side [turn ownership](#turn-ownership); client leases are local conveniences only.
4. **iOS audio.** Background microphone capture is impossible in Safari, so conversation mode is foreground-only. Acceptable.
5. **Tailscale LocalAPI availability.** Socket permissions differ across platforms; `tailscale serve` identity headers avoid the LocalAPI entirely, and the device-key path is always present as a fallback.
6. **Manager cost and latency.** Two model hops sit between the operator and a session (voice agent → manager). Status questions must be answered from the cached digest, never by a fresh read; only `ask` / `investigate` may be slow, and they are async. Announcements are rate-limited to once per turn end, with `ui_request` bursts debounced.
7. **Voice-provider tool semantics differ.** All three execute tools client-side and all three accept injected text (`conversation.item.create` on OpenAI and xAI, `sendContextualUpdate` on ElevenLabs), but whether an injected announcement makes the agent *speak* unprompted differs — OpenAI and xAI need an explicit `response.create`, ElevenLabs needs `sendUserMessage` to force a turn. The interrupt policy therefore lives in palantir, which decides *when* to inject, and the per-provider adapter only knows *how*. xAI's WebSocket-only transport also means palantir owns audio capture and encoding for that provider.
8. **Read-only enforcement.** The allowlist must cover extension-provided tools, not only the built-ins, and bash needs a conservative read-only classifier (or to be disabled in read-only sessions). Err on the side of refusing.
9. **Idle retention upstream.** Keeping an `AgentSession` warm across an idle gap belongs in `PiServer` (see `dispose()` above). Until that option exists, a session with no attachments that goes idle is disposed and re-opened from disk on the next attach — correct, just slower.

---

## Prior art

Surveyed before starting; none of these combine remote web/mobile access, public-key auth, and BYOK conversational voice with a manager layer between the operator and the sessions, and none build on pi's own protocol packages.

| Project | What it does | Why not just use it |
| --- | --- | --- |
| [jmfederico/pi-web](https://github.com/jmfederico/pi-web) | Most mature web UI; machines → projects → workspaces → sessions, multi-machine proxying | No voice; auth undocumented; own session daemon rather than the protocol packages |
| [BlackBeltTechnology/pi-agent-dashboard](https://github.com/BlackBeltTechnology/pi-agent-dashboard) | Mobile-first dashboard, OAuth2 (GitHub/Google/OIDC), mDNS and zrok, mirrors the TUI | No voice; bridge-extension architecture; heavier than needed |
| [`@firstpick/pi-package-webui`](https://pi.dev/packages/@firstpick/pi-package-webui) | Web UI plus an optional browser "Natural Conversation Mode" voice loop | Voice is plain STT→prompt→TTS with no manager in between; LAN PIN auth only |
| [remote-pi](https://github.com/jacobaraujo7/remote_pi) | Native iOS and Android apps, Ed25519 identity, QR pairing, WebSocket relay | Relay sees plaintext; no voice; native apps |
| [pi-remote-control](https://github.com/zerray/pi-remote-control) | iOS app plus relay daemon, Tailscale-friendly | iOS only, closed app, no voice |
| [`@howaboua/pi-gippity-control`](https://pi.dev/packages/@howaboua/pi-gippity-control) | WebRTC realtime voice into a LAN web UI | Requires an OpenAI Codex login; not BYOK or multi-provider; unauthenticated by design |
| [pi-voice-stt](https://pi.dev/packages/pi-voice-stt), [yukukotani/pi-voice](https://github.com/yukukotani/pi-voice) | Good BYOK STT/TTS with many providers | Desktop TUI only; no remote surface. Useful reference for the provider abstraction |

---

## Why this name

A palantír is a seeing-stone — you look into it and both *see* and *speak with* what is far away, which is precisely remote observation plus voice. It also fits the Tolkien lineage of pi's own `@earendil-works` scope.

The obvious names were taken: `pi-web`, `pi-remote`, `pi-relay`, `pi-remote-control`, and `pi-webui` all exist on npm and pi.dev. Considered alternatives: `pi-periscope` (safe, but implies watching only), `pi-parley` (voice, but loses the distance), `pi-osanwe` (Tolkien's *ósanwe*, mind-to-mind speech — perfect meaning, unspellable).

Trademark note: Palantir Technologies holds the mark for software. Many open-source projects use the word, and `pi-palantir` for a personal MIT-licensed tool is low risk — but `pi-periscope` remains the fallback if that ever becomes uncomfortable.

---

## License

MIT
