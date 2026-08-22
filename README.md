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
4. Voice interaction that is more than STT→prompt→TTS: an optional **interpreter** (frontier model, BYOK, multi-provider) that narrates what a session is doing, answers questions about it, and turns spoken intent into prompts, steers, and approvals.
5. Build on pi's own `protocol` / `client` / `server` packages rather than inventing another extension-side bridge.

**Non-goals (v1)**

- Replacing the TUI. The daemon hosts sessions; the TUI attaches to the same sessions later ([phase 6](#phasing)).
- Multi-tenant or multi-user. One operator, many devices.
- Public-internet exposure. A private network (Tailscale) is assumed; a reverse-proxy story is documented, not built.
- A native app. PWA only.

---

## What already exists upstream

pi ships three packages that are exactly the right foundation, and they are further along than the third-party remote UIs assume.

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
┌──────────────── phone / tablet / PC (PWA) ────────────────┐
│  UI (Preact + pi-client)                                   │
│   ├─ SessionList · Transcript · Composer · Approvals       │
│   ├─ Voice: mic capture → STT ─┐                            │
│   │          TTS playback  ◄───┤                            │
│   └─ Interpreter client ◄──────┘                            │
│        (BYOK keys live here, in IndexedDB, never on host)   │
│                 │ wss://host.tailnet/pi   (CBOR over TLS)   │
└─────────────────┼───────────────────────────────────────────┘
                  │  HTTP upgrade: Origin check · auth (Tailscale whois | device-key cookie)
┌─────────────────▼────────── pi-palantir daemon ─────────────┐
│  WebSocketListener ──► PiServer (pi-server)                 │
│                             │ PiServerService               │
│                      AgentSessionService  (Part A, upstream)│
│                             │                               │
│      AgentSession #1   AgentSession #2   AgentSession #n    │
│      (coding-agent SDK: real tools, extensions, skills)     │
│                                                             │
│  Static PWA assets · /auth · /pair · /interp proxy · /health│
└─────────────────────────────────────────────────────────────┘
```

### Trust boundaries

- **Daemon host** holds credentials for the *coding* models (pi's normal `auth.json`). It may hold none at all — "pi on the server without model access" is a supported, **observe-only** configuration: sessions can be listed, attached, and read, but `prompt` / `steer` are rejected until a model is selected. How that state is encoded on the wire is specified under [no-model sessions](#no-model-sessions).
- **Project trust** is the host's, not the client's. A remote `create` pointing at a directory with project extensions goes through the same trust gate the CLI uses ([below](#project-trust)); the daemon never executes untrusted project code because a phone asked it to.
- **Client device** holds the *voice and interpreter* BYOK keys. They go straight from the browser to ElevenLabs / OpenAI / xAI / Anthropic. The daemon never sees them unless the user opts into the `/interp` proxy, which exists only for providers that block browser calls, and which stores nothing.
- Everything runs on the tailnet. The daemon refuses to bind to non-private interfaces without an explicit `--insecure-bind`.

---

## Part A — upstream server completion

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

The protocol's default frame ceiling is 16 MiB, and a long session's transcript can exceed that cumulatively even with per-result truncation. A snapshot therefore carries a **tail window** — the most recent items up to a byte budget (default 1 MiB, configurable) — plus a `historyCursor` marking where the window starts. The rest is fetched on demand:

| Command | Purpose |
| --- | --- |
| `history { sessionId, before: cursor, limit }` | Page backwards through items older than the window; returns items plus the next cursor. |
| `read_item { sessionId, itemId }` | Fetch one item untruncated (full tool result, full diff). This is what the interpreter's `read_more` calls. |

`session_progress` events are unaffected; they are deltas on the live tail. Raising the frame limit is explicitly **not** the answer.

#### No-model sessions

`AgentSession.model` may be undefined — no authenticated model, or a resumed session whose persisted model lost its credentials — while v1 `SessionSnapshot.model` is required. The contract:

- In **protocol v2** `model` becomes optional and the snapshot gains `modelState: "ready" | "unavailable"` with a human-readable `modelStateReason`. Such a session is observe-only: `prompt`/`steer` are rejected with `PiServerError("invalid_request", "no model")` until `setModel` succeeds with an authenticated model from `listModels()`.
- For a connection negotiated at **v1**, sessions without a model are omitted from `list` and `attach` returns `invalid_request` — never a snapshot that would fail schema validation on the client.
- Both paths are exercised by tests: `create` with no credentials, and `open` of a persisted session whose model is no longer authenticated.

#### Project trust

The CLI resolves project trust through `ProjectTrustStore` / `resolveProjectTrusted()` **before** `createAgentSessionServices()`, and non-interactive modes fail closed when no decision exists. The service reuses that gate rather than bypassing it:

- `createSession({ cwd })` and `openSession()` call the same resolver with the daemon's `agentDir` trust store. A recorded decision (`trust.json`) is honoured as-is.
- **Phases 0–1** are non-interactive: with trust-requiring resources present and no decision, the default is to **reject** (`PiServerError("invalid_request", "project not trusted: <cwd>")`); an opt-in `untrustedProjects: "load-without-protected-resources"` service option loads the session with project extensions/skills/prompts withheld, and the snapshot says so (`projectTrusted: false`). Operators record trust from the TUI (`pi` in that cwd) or `pi --trust`.
- **Later**, a `ui_request` with `method: "confirm"` carries the trust question to the remote client; an affirmative answer is written through the same trust store so the decision is explicit and persistent, identical to what the TUI would record.

### Protocol additions (v2, additive)

#### Version negotiation

Today's handshake accepts only `version === PROTOCOL_VERSION`, and the strict v1 codec rejects unknown message shapes — so "gate it on the handshake" is not enough by itself: bumping the constant rejects v1 clients, and sending a v2 event to a v1 decoder breaks it. The negotiation is therefore explicit:

- `hello` carries `supportedVersions: [1, 2]` (client) and the server replies with the highest version both support, or rejects with `unsupported_version` when the ranges do not intersect. `isSupportedProtocolVersion()` becomes a range check.
- The server instantiates a **per-connection codec** for the negotiated version and keeps a per-connection `version` in `ConnectionState`.
- **Event filtering**: v2-only events (`ui_request`, `ui_resolved`) are not sent on v1 connections, and v2-only snapshot fields (`modelState`, `pendingUiRequests`, `historyCursor`, `projectTrusted`) are stripped by the v1 encoder. v1 clients see a v1 session and simply cannot answer approvals.
- Every v2 command and event is additive; nothing in v1 changes meaning.

#### Commands and events

| Command / event | Purpose |
| --- | --- |
| `ui_request` event · `ui_respond` command · `ui_resolved` event | Surface extension UI — `confirm`, `select`, `input`, `editor`, and tool-approval prompts — to remote clients, mirroring the `extension_ui_request` / `extension_ui_response` contract in pi's `docs/rpc.md`, including the fire-and-forget methods `notify`, `setStatus`, `setWidget`, `setTitle`, `set_editor_text` (sent with `expectsResponse: false`). **Non-negotiable**: without it a remote client cannot answer a permission prompt. Semantics under [pending UI requests](#pending-ui-requests). |
| `history`, `read_item` | Paged transcript history and untruncated item fetch (see [bounded transcript](#bounded-transcript)). |
| `fork { sessionId, itemId }`, `switch { sessionId, targetSessionId }` | Session-tree navigation as **server-level** operations. `LiveSessionManager` keys a runtime by its original ID and rejects a snapshot whose ID changes, so neither can mutate the runtime in place. Both return `{ sessionId }` of the destination; the client then performs an explicit `detach` / `attach` transition, and the source runtime is left untouched. |
| `read_file { path, range }`, `diff { ref? }` | Read-only views for review on a phone. The server enforces cwd-rooted paths and a size cap. |
| `list_commands` | Discovery metadata for the `/` palette: name, description, source (builtin / extension / skill / template). Execution stays on `prompt` with `source: "rpc"` — `AgentSession.prompt()` already recognises extension commands and expands skills and templates, so a separate `slash` command would duplicate routing and drift from the TUI. |
| `session_name` | Equivalent of `/name`. |
| `prompt` / `steer` content (v2) | `{ content: [ { type: "text", text } \| { type: "image", mimeType, data } ] }` alongside v1's `{ text }`. Bounded: `image/png`, `image/jpeg`, `image/webp`, `image/gif`; ≤ 5 MiB per image decoded, ≤ 4 images per message, enforced by the server and by the PWA before sending. This is what lets the composer's camera and paste actually reach `AgentSession`. |
| Mutation envelope (v2) | Every mutating command (`prompt`, `steer`, `abort`, `ui_respond`, `setModel`, `setThinking`, `fork`, `switch`, `session_name`) carries a client-generated `requestId` (UUID) and an `expectedRevision`. The server deduplicates `requestId` per session for a retention window (default 10 minutes, returning the original result) and rejects a mismatched `expectedRevision` with `stale_revision`. See [offline and reconnect](#offline-and-reconnect). |

Terminal access and git mutation stay out of scope.

#### Pending UI requests

An event alone is lossy — a reconnecting client gets an authoritative snapshot but would never learn about the approval it missed — and `PiServer` broadcasts events to every attachment, so two devices can race to answer. Hence:

- **In the snapshot.** `SessionSnapshot.pendingUiRequests: [{ id, method, payload, createdAt, expiresAt? }]`. Pending requests belong to the *session*, not to the connection that happened to be attached when they were raised, so they survive client disconnects and are visible to anything that attaches later.
- **First response wins, atomically.** The runtime resolves a request exactly once. A second `ui_respond` for the same `id` — from any device — returns `PiServerError("stale_request", "already answered")`; an `id` that never existed or has expired returns `stale_request` too. Every attachment receives `ui_resolved { id, by: connectionId }` so open approval cards close everywhere.
- **Expiry and cancellation.** Requests carry the extension's timeout when it has one. `abort` cancels all pending requests for the session; the extension sees the same cancellation it would in the TUI. Requests are never transferred or cancelled merely because a connection dropped.
- **Fire-and-forget methods** are delivered as `ui_request` with `expectsResponse: false`, are never stored in `pendingUiRequests`, and accept no response.

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
├─ daemon/          # node: WebSocket listener, auth, static assets, /interp proxy
├─ web/             # PWA (Vite). No Node imports. pi-client + the transcript reducer.
└─ shared/          # auth token format, interpreter tool schema
```

### WebSocket listener

`createWebSocketListener(opts): PiServerListener` — `ws` on Node, each socket becoming a `ByteConnection` whose `send` honours `bufferedAmount` backpressure and whose `maxPendingBytes` overflow closes the socket, matching the Unix listener's behaviour. Authentication runs in the **HTTP upgrade handler**, before the connection reaches `PiServer` — exactly what the `pi-server` README prescribes.

The upgrade handler runs these checks in order, and any failure is a plain HTTP error before a socket exists:

1. **Origin.** Browsers may open cross-origin WebSockets and the handshake is not protected by CORS, so a malicious page visited by an allowed user could otherwise drive the daemon with that user's tailnet identity. The `Origin` header must equal the daemon's own HTTPS origin exactly. A missing `Origin` is accepted **only** on the native-client path (no cookie; device-key signature over the server nonce carried in `Sec-WebSocket-Protocol`), never for Tailscale zero-click.
2. **Authentication** — one of the two mechanisms below.
3. Hand-off to `PiServer` with the resolved operator identity attached to the connection.

### Authentication

Two mechanisms, tried in order. Both resolve to an operator identity; anything else is a 401 on upgrade.

1. **Tailscale identity — zero-click.** With the daemon bound to the Tailscale interface, the upgrade handler asks the Tailscale LocalAPI `whois` for the peer IP and allows the connection when `UserProfile.LoginName` is in `allowedLogins`. No token, no pairing: the phone just opens the URL. Being IP-based rather than cookie-based, it survives PWA relaunch. `whois` authenticates the *network peer*, not the page that opened the socket — which is why the Origin check above is mandatory on this path. When TLS is terminated by `tailscale serve`, the identity arrives as `Tailscale-User-Login` headers instead; both are accepted.
2. **Device keys — the SSH-ID feel.** The browser generates a non-extractable Ed25519 keypair via WebCrypto and stores it in IndexedDB. Pairing: `/palantir pair` in the TUI shows a QR code and short code carrying a one-time secret; the PWA posts `{ pubkey, deviceName, pairingSecret }` to `/pair`; the daemon appends the key to `~/.pi/palantir/authorized_keys` in OpenSSH format (`ssh-ed25519 AAAA… phone-martin`). Login is an **HTTP bootstrap**, because a browser's `WebSocket` constructor takes only a URL and subprotocols and cannot set an `Authorization` header:
   - `GET /auth/challenge` → `{ nonce, host, issuedAt }` (single use, 60 s).
   - `POST /auth/login` (same origin) with `{ pubkey, signature }` over `nonce ‖ host ‖ issuedAt`.
   - On success the daemon sets a `Secure; HttpOnly; SameSite=Strict; Path=/` cookie (30 days). Its value is an HMAC-signed token binding the key's fingerprint and the daemon's **auth epoch**; the upgrade handler re-validates it on every connection against the current `authorized_keys` and epoch, so `/palantir revoke <name>` (which bumps the epoch for that key, or globally) invalidates already-issued cookies immediately rather than in 30 days.

`/palantir authorize github:<user>` imports keys from `https://github.com/<user>.keys`. Those serve SSH-capable desktop clients — a browser cannot use an existing private key, which is why browsers get their own generated device key. Revocation is `/palantir revoke <name>` or editing `authorized_keys`.

### Web client (PWA)

- **Stack** — Vite and Preact, small enough to stay quick on a phone; `pi-client` for the protocol and the upstream `RemoteSession` plus transcript reducer for state. Markdown and diff rendering; no terminal emulator.
- **Transport** — WebSocket binary frames, wrapped in an exponential-backoff reconnect loop around `PiClient.reconnect()`, since the client deliberately does not reconnect on its own. Reconnect re-attaches previously attached sessions; snapshots are authoritative for *state*, so no event replay is needed — pending approvals arrive in `pendingUiRequests`, older history through `history`.
- **Screens** — Sessions (grouped by cwd, phase badge, last activity) · Session (transcript tail with collapsible tool calls and "load earlier" paging, queued steers, approval cards from `pendingUiRequests`) · Composer (text, image paste and camera via the v2 image content parts, `/` palette from `list_commands`) · Settings (device keys, voice providers, interpreter).
- <a id="offline-and-reconnect"></a>**Offline and reconnect** — a service worker precaches the shell, and the last snapshot per session is persisted to IndexedDB, so opening the app with no connectivity still shows state. A prompt composed offline is **queued, not auto-replayed**: every mutation carries a `requestId` and the `expectedRevision` the user was looking at when they wrote it. On reconnect the client first receives the fresh snapshot; if the revision still matches, it sends the queued mutation (the server's `requestId` dedupe makes the retry safe even if the original was accepted just before the drop); if the session has moved on — another device prompted, a turn finished — the draft stays in the composer marked *written against an older state* for the user to send or discard. Nothing else is meaningfully offline.
- **Notifications** — Web Push when a session raises a `ui_request` or finishes a turn while the app is backgrounded. VAPID keys are generated on first daemon run. iOS supports Web Push for installed PWAs from 16.4.

### Voice

Three independently configurable slots, all bring-your-own-key, all client-side.

| Slot | Providers (v1) |
| --- | --- |
| STT | ElevenLabs Scribe (streaming), OpenAI `gpt-4o-transcribe` / Whisper, xAI, browser `SpeechRecognition` (free fallback) |
| TTS | ElevenLabs (streaming), OpenAI, xAI, browser `speechSynthesis` (free fallback) |
| Interpreter | Anthropic, OpenAI, xAI — or none, which leaves plain dictation |

Two modes:

- **Dictation** — push-to-talk, STT, text lands in the composer for review. The mobile equivalent of `pi-voice-stt`.
- **Conversation** — hold-to-talk, STT, then the interpreter either speaks an answer, submits a prompt or steer, answers a pending `ui_request`, or aborts. Side effects are confirmed aloud before execution unless "act immediately" is enabled.

### The interpreter

A small agent, running in the browser by default, with a system prompt and tools rather than free-form text:

```
Context per turn:
  · compact session digest — last N transcript items, tool calls collapsed to
    name + status + one-line result, current phase, pending ui_request
  · the user's utterance

Tools:
  say(text)                     → TTS
  prompt(text) / steer(text)    → protocol commands (exclusive lease)
  respond_ui(requestId, answer) → approvals and selects
  abort()
  read_more(itemId)             → pull a tool result or diff into context on demand
```

- **Digest, not full transcript** — mobile context and cost both matter; `read_more` fetches detail only when the conversation needs it ("what did the test output actually say?").
- **Voice-shaped output** — the system prompt enforces short spoken sentences, no code recitation unless asked, file paths spoken as basenames.
- **Proactive narration (opt-in)** — on turn end or a new `ui_request`, the interpreter runs with an empty utterance to announce what happened: *"Finished — three files edited, tests pass. It wants permission to run `git push`. Allow?"*
- **Keys stay on the device.** The `/interp` daemon proxy exists only for providers without workable CORS and forwards keys per request without storing them.

### Extension (TUI side)

Deliberately thin: `/palantir start|stop|status`, `/palantir pair` (QR overlay), `/palantir authorize <key|github:user>`, `/palantir revoke <name>`, and a status-line segment showing connected devices. The daemon is a separate process with launchd and systemd unit templates, so it outlives any one terminal.

---

## Key decisions

| Decision | Rationale |
| --- | --- |
| The daemon hosts sessions; the TUI attaches as a client later | Otherwise "remote" dies with the terminal. v1 can coexist: daemon-hosted and TUI-hosted sessions are separate session files, and the web UI sees only the former. |
| Auth in the HTTP upgrade, not in the protocol | Matches pi-server's stated design and keeps the protocol runtime-neutral. |
| Voice and interpreter BYOK is client-side | The host may be a shared server; keys for *my* voice stay on *my* phone. It also lets the host run without model access while the client still has an interpreter. |
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
| 3 | Device-key pairing, `authorized_keys`, revocation | Works from a non-tailnet browser on the LAN. |
| 4 | Dictation voice — ElevenLabs and OpenAI first | Hands-free prompt entry. |
| 5 | Interpreter — Anthropic first, then OpenAI and xAI — plus proactive narration | "What's it doing?", "approve", "also update the tests" — by voice. |
| 6 | TUI attaches to daemon sessions | One session visible in the terminal and on the phone at once. |

---

## Open questions and risks

1. **Transcript projection fidelity.** `SessionSnapshot.transcript` is a flattened view of a tree-shaped session; compaction and branches need a defined mapping. The HTML exporter already has one — reuse it rather than writing a second.
2. **TUI-as-client.** The TUI owns `AgentSession` in-process today. Turning it into a `PiClient` of the daemon is the largest refactor here; the existence of `coding-agent/src/client/remote-session.ts` upstream suggests that direction is already intended. Track upstream before building it.
3. **Lease semantics.** Phone and TUI may both want to prompt. Proposal: whichever surface prompts takes an exclusive lease for the turn and releases it at turn end; observers hold shared leases.
4. **iOS audio.** Background microphone capture is impossible in Safari, so conversation mode is foreground-only. Acceptable.
5. **Tailscale LocalAPI availability.** Socket permissions differ across platforms; `tailscale serve` identity headers avoid the LocalAPI entirely, and the device-key path is always present as a fallback.
6. **Interpreter cost.** Proactive narration can fire often — rate-limit to once per turn end and debounce `ui_request` bursts.
7. **Idle retention upstream.** Keeping an `AgentSession` warm across an idle gap belongs in `PiServer` (see `dispose()` above). Until that option exists, a session with no attachments that goes idle is disposed and re-opened from disk on the next attach — correct, just slower.

---

## Prior art

Surveyed before starting; none of these combine remote web/mobile access, public-key auth, and BYOK voice with an interpretation layer, and none build on pi's own protocol packages.

| Project | What it does | Why not just use it |
| --- | --- | --- |
| [jmfederico/pi-web](https://github.com/jmfederico/pi-web) | Most mature web UI; machines → projects → workspaces → sessions, multi-machine proxying | No voice; auth undocumented; own session daemon rather than the protocol packages |
| [BlackBeltTechnology/pi-agent-dashboard](https://github.com/BlackBeltTechnology/pi-agent-dashboard) | Mobile-first dashboard, OAuth2 (GitHub/Google/OIDC), mDNS and zrok, mirrors the TUI | No voice; bridge-extension architecture; heavier than needed |
| [`@firstpick/pi-package-webui`](https://pi.dev/packages/@firstpick/pi-package-webui) | Web UI plus an optional browser "Natural Conversation Mode" voice loop | Voice is plain STT→prompt→TTS with no interpretation; LAN PIN auth only |
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
