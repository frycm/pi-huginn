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
| CLI | — | No `pi serve` / daemon mode. |

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
│                 │ wss://host.tailnet:7314/pi   (CBOR)       │
└─────────────────┼───────────────────────────────────────────┘
                  │  HTTP upgrade: auth (Tailscale identity | Ed25519)
┌─────────────────▼────────── pi-palantir daemon ─────────────┐
│  WebSocketListener ──► PiServer (pi-server)                 │
│                             │ PiServerService               │
│                      AgentSessionService  (Part A, upstream)│
│                             │                               │
│      AgentSession #1   AgentSession #2   AgentSession #n    │
│      (coding-agent SDK: real tools, extensions, skills)     │
│                                                             │
│  Static PWA assets · /interp proxy (optional) · /health     │
└─────────────────────────────────────────────────────────────┘
```

### Trust boundaries

- **Daemon host** holds credentials for the *coding* models (pi's normal `auth.json`). It may hold none at all — "pi on the server without model access" is a supported configuration.
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
  createSession(o: CreateSessionOptions): Promise<PiSessionRuntime>;
  openSession(id: string): Promise<PiSessionRuntime>; // resume; one runtime per id, refcounted
}
```

`AgentSessionRuntime implements PiSessionRuntime` wraps an `AgentSession`:

- **`snapshot()`** — projects `AgentSession` state into a `SessionSnapshot`: `phase` from agent state, `transcript` through the same tree-to-flat mapping the HTML exporter uses, plus `queuedSteer`.
- **`subscribe()`** — adapts `AgentSession` events (`message_start/update/end`, `tool_execution_*`, `turn_*`) into `{ type: "progress", progress: TranscriptProgress }`, and emits `{ type: "snapshot" }` on authoritative changes: turn end, model change, compaction.
- **`prompt` / `steer` / `abort` / `setModel` / `setThinking`** — direct delegation, rejecting with `PiServerError("busy")` when the phase is not idle, matching the contract the test runtime already documents.
- **`dispose()`** — releases a refcount. The `AgentSession` shuts down when the last lease is released *and* an idle-unload timer (default 30 minutes) fires, so a phone dropping off Wi-Fi mid-turn does not kill the run.

### Protocol additions (v2, additive)

All gated behind the handshake version, so v1 clients keep working.

| Command / event | Purpose |
| --- | --- |
| `ui_request` event · `ui_respond` command | Surface extension UI — `confirm`, `select`, `input`, tool-approval prompts — to remote clients. Mirrors the RPC-mode extension UI protocol already specified in pi's `docs/rpc.md`, so this transports an existing contract rather than inventing one. **Non-negotiable**: without it a remote client cannot answer a permission prompt. |
| `fork`, `switch` | Session-tree navigation. |
| `read_file { path, range }`, `diff { ref? }` | Read-only views for review on a phone. The server enforces cwd-rooted paths. |
| `slash { input }` | Routes `/commands` through the same input pipeline the TUI uses, with `event.source = "rpc"`. |
| `session_name` | Equivalent of `/name`. |

Terminal access and git mutation stay out of scope.

### `pi serve`

`pi serve --unix ~/.pi/agent/server.sock [--listener <module>]` starts `AgentSessionService` plus its listeners. pi-palantir contributes the WebSocket listener through `--listener`, or simply embeds the same wiring itself — simpler for v1.

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

### Authentication

Two mechanisms, tried in order. Both resolve to an operator identity; anything else is a 401 on upgrade.

1. **Tailscale identity — zero-click.** With the daemon bound to the Tailscale interface, the upgrade handler asks the Tailscale LocalAPI `whois` for the peer IP and allows the connection when `UserProfile.LoginName` is in `allowedLogins`. No token, no pairing: the phone just opens the URL. Being IP-based rather than cookie-based, it survives PWA relaunch.
2. **Device keys — the SSH-ID feel.** The browser generates a non-extractable Ed25519 keypair via WebCrypto and stores it in IndexedDB. Pairing: `/palantir pair` in the TUI shows a QR code and short code carrying a one-time secret; the PWA posts `{ pubkey, deviceName, pairingSecret }` to `/pair`; the daemon appends the key to `~/.pi/palantir/authorized_keys` in OpenSSH format (`ssh-ed25519 AAAA… phone-martin`). Login signs a server nonce plus host and timestamp, sent as `Authorization: Palantir-Ed25519 <pubkey>.<sig>`, and the daemon issues a 30-day HttpOnly cookie so reconnects are silent.

`/palantir authorize github:<user>` imports keys from `https://github.com/<user>.keys`. Those serve SSH-capable desktop clients — a browser cannot use an existing private key, which is why browsers get their own generated device key. Revocation is `/palantir revoke <name>` or editing `authorized_keys`.

### Web client (PWA)

- **Stack** — Vite and Preact, small enough to stay quick on a phone; `pi-client` for the protocol and the upstream `RemoteSession` plus transcript reducer for state. Markdown and diff rendering; no terminal emulator.
- **Transport** — WebSocket binary frames, wrapped in an exponential-backoff reconnect loop around `PiClient.reconnect()`, since the client deliberately does not reconnect on its own. Reconnect re-attaches previously attached sessions; snapshots are authoritative, so no replay logic is needed.
- **Screens** — Sessions (grouped by cwd, phase badge, last activity) · Session (transcript with collapsible tool calls, queued steers, approval cards from `ui_request`) · Composer (text, image paste and camera, `/` palette) · Settings (device keys, voice providers, interpreter).
- **Offline** — a service worker precaches the shell, and the last snapshot per session is persisted to IndexedDB, so opening the app with no connectivity still shows state and lets you queue a prompt that sends on reconnect. Nothing else is meaningfully offline.
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
| Protocol additions are additive v2 | `ui_request` is required for real use; everything else can slip a phase. |

---

## Phasing

| Phase | Deliverable | Done when |
| --- | --- | --- |
| 0 | `AgentSessionService` + `pi serve --unix` (Part A) | `PiClient` over a Unix socket can create, attach to, and prompt a real session; the upstream `testing/` suite passes against it. Upstreamable PR. |
| 1 | WebSocket listener, Tailscale auth, minimal transcript view | Open `https://host.tailnet:7314` on a phone and watch a live session. |
| 2 | Composer, `ui_request` approvals (protocol v2), PWA install, push | Drive a full session from a phone, including tool approvals. |
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
5. **Tailscale LocalAPI availability.** Socket permissions differ across platforms; the device-key path is always present as a fallback.
6. **Interpreter cost.** Proactive narration can fire often — rate-limit to once per turn end and debounce `ui_request` bursts.

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
