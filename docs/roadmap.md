# Delivery plan

All Huginn milestones are unimplemented. Dependencies and current integration facts are in
the [September 5 assessment](upstream-2026-09-05.md); behavior is specified in the
[architecture](architecture.md).

## Release sequence

| Milestone | Deliverable | Acceptance evidence |
| --- | --- | --- |
| 0 — verify the foundation | Pin pi v0.85.0, exercise its experimental host/services/TUI, inspect package exports, and identify the smallest missing common-host controls. Probe Enclave and Muninn compatibility separately. | A real session can be created, prompted, observed from two presentations, detached, and reattached. Record which extension hooks, approvals, durable operations, and browser bindings work; document failures without assuming feature parity. No legacy v1/v2 patch series. |
| 1 — hand off one session | Always-on host; Tailscale Serve and operator authorization; thin PWA and shared TUI session; bounded transcript; direct prompt/steer/stop/takeover; persistent command receipts and outcomes. | Start work in the TUI, observe and take control on the phone while the TUI remains connected, then return to that same session. Disconnect after sending a command and recover its receipt. Restart the host and show a reconciled outcome or explicit uncertainty without replaying side effects. |
| 2 — supervise through text | Hosted text manager and durable conversations; attention inbox and opt-in push; evidence-backed digests; one bounded live investigation; host budgets and outbox; setup diagnostics. | Ask what is happening, delegate a steer, launch a question that answers later, and continue the conversation from another device. Closing every browser loses neither ticket nor outcome. Direct controls still work during manager failure. This completes the first usable standalone release. |
| 3 — integrate boundaries and history | Optional Enclave profile/status/decision bridge and Muninn retrieval/capture adapters, with compatibility gates. Work starts during milestone 0 and may ship earlier when verified. | Run within a user-defined Enclave profile with no duplicate Huginn approvals; a denial cannot be bypassed and loss of selected enforcement stops affected execution. Retrieve attributable Muninn history, export one additional observation idempotently, and keep core work functioning when Muninn is absent. Claim phone approval of persisted Enclave actions only after its supported remote API exists and passes the bridge tests. |
| 4 — converse by voice | One provider, host token broker, thin audio client, durable announcements and explicit audio handoff. | The text workflow works by voice within measured latency/cost bounds. An investigation answers later at a pause, switching devices preserves the conversation, and suspending audio preserves host work. Direct stop remains usable independently. |
| Later — expand demonstrated use | Alternative device authentication, more voice providers, images/camera, captured-workspace investigations, isolated parallel writers, additional hosts or native clients. | Each addition has a concrete observed need, its own compatibility evidence, and preserves the earlier handoff/recovery invariants. |

Shared TUI sessions are an early gate, not a final refactor. If the experimental runtime
cannot yet compose the extensions or controls needed, milestone 0 records the required
upstream change. Optional Enclave/Muninn features can remain unavailable without postponing
the standalone release; an explicitly selected but unhealthy enforcement configuration
cannot silently run standalone.

## The first-release acceptance scenario

A developer starts a task in a daemon-hosted session from the TUI, opens the PWA on a phone,
and sees the same work. They leave the TUI attached, take control on the phone, and steer the
active run. A dropped connection after sending does not cause the instruction to run twice.

They ask the text manager to investigate a failure. It returns a ticket immediately. They
close the phone and later open another client: the ticket, conversation, outcome, and unread
announcement are there. The answer names its source evidence and the workspace it observed.
After a host restart, the UI distinguishes recovered work from interrupted or unknown work.

They can stop the selected coding operation even when the manager model is unavailable.
They can complete this scenario with ordinary pi alone. In the Enclave variant, in-boundary
work proceeds under the configured policy and escalations follow Enclave's actual supported
decision channel. The UI never labels a terminal-only pending action remotely approvable.

## Product measures

These are initial targets to validate with use, not measured capabilities. Record test host,
phone/browser, network path, providers, and sample size when reporting results.

| Measure | Initial target or release gate |
| --- | --- |
| Setup effort | First connected session within 10 minutes once pi and Tailscale are installed; no routine certificate repair or manual database work |
| Time to understand work | With three active sessions, identify what changed and what needs attention within 30 seconds without opening raw tool logs |
| Warm status view | Cached attributable status visible within 2 seconds on the reference tailnet; show freshness and disconnection clearly |
| Command acceptance | Receipt visible within 1 second on the reference tailnet, independent of model response time; measure percentiles |
| Device handoff | Same session/conversation after switching devices; no lost admitted instructions or unresolved decision cards |
| Recovery correctness | Disconnect-after-send and restart-at-dispatch cases produce no blind redispatch, false success, or silently lost tickets |
| Notification quality | No lost pending cause in the recovery scenarios; measure duplicate deliveries, actionable/ignored ratio, and user muting rather than promising exactly-once delivery |
| Manager usefulness | An initial fixed set of status, steer, ambiguous-target, failure, and investigation scenarios; unsupported conclusions and wrong-session actions are release failures |
| Cost and resource use | Combined usage visible where available; bounded investigations and admission budgets; reported billing gaps and possible overshoot explicit |

## Required validation

Use focused integration scenarios around actual host/worker boundaries rather than tests
that merely mirror a view reducer:

- Two devices and a TUI race to steer, take control, answer a decision, and stop. Old control
  generations cannot steer; a delayed stop cannot cancel the next run.
- A request is accepted and its reply lost; the client reconnects after state advanced.
  The same ID retrieves the receipt, while changed input under that ID is rejected.
- Kill the host before dispatch, during dispatch, and after upstream acceptance. Reconcile
  from durable evidence; ambiguous side effects remain unknown instead of being repeated.
- A 50 MiB tool result passes through live updates, snapshots, paging, and range reads
  without exceeding configured transport or client-memory budgets.
- Missing model credentials, incompatible pi/services, stale PWA assets, and failed optional
  adapters produce useful capability/failure states while unaffected controls remain usable.
- Enclave: ordinary permitted actions, deterministic deny, human ask, expired request,
  changed policy, lost attendance, manager-authored text, and disabled/unhealthy enforcement.
  Manager text must not become direct-human authorization; no model can answer a human ask.
- Muninn: ordinary session capture is not duplicated; explicit retrieval preserves correction
  and trust labels; replayed observations deduplicate; unavailable or incompatible Muninn does
  not block live work or turn external evidence into user authority.
- A live investigation observes an externally edited dirty checkout and reports its limits.
  The one-writer guard applies at the shared runtime boundary, including TUI admission.
- Budget exhaustion and manager/voice outage leave cancellation and the attention inbox
  available. Read/display/playback acknowledgement races preserve source outcome records.

## Decisions deliberately deferred

The initial implementation must settle the exported pi composition path, extension lifecycle
compatibility, durable receipt reconciliation, and Enclave attendance/provenance bridge.
Those are evidence gaps rather than reasons to design a replacement pi framework now.

Provider pricing/model surveys, generalized compatibility with old experimental protocols,
LAN/public exposure, a second policy language, and a second project journal are not first
release work. Further automation should follow measured usefulness and the user's Enclave
boundaries, with host-owned state throughout.
