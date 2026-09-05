# pi-huginn

**Keep coding work moving when you leave your desk.**

Run [pi](https://github.com/earendil-works/pi) on an always-on machine. See what its
sessions are doing, understand what needs your attention, and direct the same work from
your terminal, phone, tablet, or another computer — through text or conversation.

**Status: design only.** No Huginn runtime or client is implemented yet. This is the
product direction; the [architecture](docs/architecture.md),
[delivery plan](docs/roadmap.md), and [upstream assessment](docs/upstream-2026-09-05.md)
describe proposed work and distinguish it from existing pi capabilities.

## The everyday workflow

1. Start a session at your desk and give it a task.
2. Leave the terminal. The host keeps working, even when every client disconnects.
3. Open your phone to see what changed, what is blocked, and what needs a decision.
4. Ask the manager a question, steer the work, or stop it directly. A longer investigation
   returns later without blocking the conversation.
5. Return to the same session from another device, with its decisions and outcomes intact.

The manager's job is to reduce the attention needed to supervise work. It maintains a
compact view of active sessions, explains the evidence behind a status, and turns the
operator's intent into bounded work. Text ships first; voice uses the same host services.

## Host-owned state, thin clients

The host owns pi sessions, manager conversations, investigation tickets, command receipts,
approvals, notification delivery, and provider credentials. Closing a browser does not
discard work or create a second manager on the next device.

Web and mobile clients render that state and send user input. The first client is an
installable PWA for desktop, iOS, and Android. It can retain drafts and an explicitly stale
view offline; execution and authoritative state remain on the host. Future native clients
can use the same services.

## Three complementary projects

| Project | Responsibility | Relationship to Huginn |
| --- | --- | --- |
| **Huginn** | Live supervision, remote access, manager, text and voice interaction | This project |
| **[pi-enclave](https://github.com/frycm/pi-enclave)** | Sandbox enforcement, deterministic policy, model review, and human escalation | Strongly recommended for unattended work and auto mode; optional |
| **[pi-muninn](https://github.com/frycm/pi-muninn)** | Searchable, provenance-rich project history | Optional historical context and bounded outcome capture |

**Enclave defines the boundaries.** Users configure them in Enclave's own format; Huginn
shows the effective policy and routes decisions through Enclave. Huginn does not introduce
a competing sandbox, policy language, or model reviewer. Actions Enclave permits within the
user's delegated task do not acquire a second Huginn approval step.

**Huginn also works on its own.** Without Enclave, remote sessions and the manager use pi's
configured tools and execution behavior. Huginn does not claim that an Enclave boundary is
enforced when the extension is absent. Selecting Enclave and then losing its enforcement
must never silently fall back to standalone execution.

**Muninn supplies history, not live control state.** The manager can explicitly retrieve
past decisions and outcomes, with their provenance and corrections. Session status,
in-flight work, command receipts, and pending approvals remain available without Muninn.
Installing or removing either optional integration does not migrate ownership of those facts.

These are intended integration contracts, not compatibility claims. The dated
[assessment](docs/upstream-2026-09-05.md#sibling-integrations) records the current version
and API gaps.

## First usable release

The first release proves the desk-to-phone-to-desk workflow: one shared session visible in
the TUI and PWA, Tailscale access, direct steering and stopping, actionable notifications,
and a small hosted text manager. Two connected devices, connection loss after sending a
command, and host restart are part of acceptance, not later polish.

The [delivery plan](docs/roadmap.md) brings that workflow forward. Enclave and Muninn
compatibility are investigated at the start; optional integrations cannot prevent a
standalone release. One voice provider follows the text workflow, with additional providers,
image attachments, and alternative authentication added only after it is useful.

## Scope

- One operator, many devices, initially one always-on host reached over Tailscale.
- Existing pi protocol and service contracts wherever they fit; small upstream changes
  only for demonstrated gaps against a tested stable release.
- One active writing run per checkout initially; bounded read-only investigations with
  explicit workspace provenance. Isolated parallel writing comes later.
- Direct session controls remain usable when the manager or voice provider is unavailable.
- No terminal emulator, git mutation UI, public relay, multi-user administration, or native
  app in the first release.

## Name

Huginn and Muninn are Odin's ravens, associated with thought and memory. Huginn attends to
ongoing work; Muninn remembers its history. Earlier proposals used the working name
`pi-palantir`; the project and proposed package/commands now use `pi-huginn` and `huginn`.

## License

[MIT](LICENSE)
