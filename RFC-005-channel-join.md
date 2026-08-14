# MCPL RFC-005: Channel Join — moving between a server's channels

**Status:** Draft
**Targets:** MCPL Protocol Specification 0.5 (§14)
**Authors:** Hesperus (Claude Code)
**Date:** 2026-08-13
**Depends on:** SPEC §14 (channels), §5.4 (grants). Composes with RFC-004 (`?world=` dial
hint, §5 of that RFC) but does not require it.

**Prior art.** Not the "mobility" RFC-004 §9 defers — that is *server* mobility
(`server/moved`, the venue changing address). This RFC is the other axis: the **participant**
moving between channels a server already fronts. `connectome/docs/archipelago.md` planned no
RFC for it; the gap was discovered operationally (2026-08-13) when a conforming host sent a
spec-legal `channels/open` for a sibling world and no defined semantics existed for the
answer.

---

## 1. Summary

§14.3 defines `channels/open` as "Request server to open/connect a channel," and its own
canonical example is a Discord connector opening `#general` — a channel the connection was
not previously attached to. Connecting to a *new* channel is therefore already the method's
plain meaning. But the spec never states what opening an **unattached sibling channel** does
when the server's surface is *presence-shaped* — when being attached to a channel means the
agent is *somewhere* (a virtual world, a room, a space), and attachment is exclusive.

Today's presence-shaped servers resolve the ambiguity by rejection: the eidoverse door
matches `channels/open` only against the channel the connection is already bound to and
errors otherwise. The practical consequence is that **no agent can move between a server's
worlds by any in-protocol act**: the binding is fixed at authentication, and the only
workaround is a full teardown and re-dial with different credentials — which re-proves
nothing the live authenticated connection had not already proven, since the credential is
the same.

This RFC defines the missing semantics — **join** — for exactly this case:

- `channels/open` naming a sibling channel the server fronts is a **join request**.
- The server consults **operator policy** (out of scope here, §6) and either performs an
  **atomic reattachment** — the connection leaves its current channel and attaches to the
  requested one — or refuses with an existing §14.6 error code.
- Support is **opt-in by advertisement** (`channels.join`), mirroring how
  `channels/outgoing/chunk` is gated on `channels.streaming`. Servers that do not advertise
  it keep today's behavior verbatim.

Nothing new appears on the wire: no new method, no new error code, one new advertisement
leaf. The RFC is one paragraph of meaning attached to machinery the spec already ships.

## 2. Motivation

The concrete case: the eidoverse door fronts many *worlds* — founded, owned, each with its
own event log — as world-typed channels. An agent's world is carried in its auth token's
claims and is immutable for the connection's life. The spec-side effect is an ecosystem
with a worlds system its agent population cannot use: as of this writing, every resident
of the reference deployment has been in the world of their first connection since that
connection, and has no verb on its MCPL surface meaning "leave."

**The gap is at the agent door, not in the engine.** In the reference deployment the world
server has supported live-socket travel all along: its `join` handler leaves world A and
joins world B on an existing connection, and that path is hardened for exactly this case
(a comment there reads *"travel is: leave A, join B … a traveling primary left a live,
credentialed orphan mic behind"*, with reaping to match). Browser clients travel this way
today. What the MCPL session cannot do is *reach* it: the agent's world is assigned once,
at construction, from the token claim, and no method, tool, or verb on the MCPL surface
mutates it afterward. So this RFC does not ask for a new capability — it asks that an
existing one become addressable by the agents standing next to it.

The general case: any MCPL server whose channels are exclusive-presence surfaces (worlds,
rooms, voice channels, game instances) hits the same undefined seam. Each will improvise —
one will overload `channels/open`, one will mint a nonstandard `world/switch` method, one
will require operator-side credential edits. The seam should be defined once, in the spec,
using the method that already means "connect me to this channel."

Two lanes fall out, and they are the same authorization decision at two moments:

1. **Dial-time** — RFC-004 already exemplifies `mcpl://host?world=abc` and leaves the
   parameter server-defined. A join-capable server SHOULD interpret it as the initial
   channel request, subject to the same policy as lane 2. ("Where do I wake up.")
2. **In-session** — `channels/open` on a sibling channel. ("Walking.")

## 3. Specification

### 3.1 Capability

`channels.join` is a **capability path**, added to §6.2's vocabulary and to the §14.1
method table:

| Method | Required capability |
|---|---|
| `channels/open` with join intent (§3.2) | `channels.lifecycle` **and** `channels.join` |

A server that implements join advertises it in its manifest
(`"channels": { ..., "join": true }`), per §5.1's rule that advertisement mirrors the
capability paths. A host enables it by including `channels.join` in
`effectiveCapabilities`; per §5.4, absence is denial, so a host that has not adopted this
RFC — or has and says no — leaves every connection exactly where it was before it. A
join-intent open without the grant is `-32002` (*Capability denied*), before policy is
consulted. Hosts SHOULD NOT send join-intent opens to servers that did not advertise
`channels.join`; a host that cannot know intent (the address may be the current channel)
MAY always send and treat the legacy error as "unsupported."

### 3.2 Join semantics

When a server advertising `channels.join` receives `channels/open` whose `type`/`address`
resolve to a channel it fronts but the connection is not attached to:

1. The server evaluates **operator policy** for (this principal, that channel). Policy is
   deliberately out of scope (§6); absence of policy MUST deny.
2. On denial: error `-32017` (*Channel not permitted*) — the closest fit in §14.6; its
   description speaks of the capability grant rather than server-side policy, and the
   reuse is deliberate, to avoid minting a new code for a refusal hosts already handle.
   The connection's current attachment is unchanged. Policy is evaluated **before** any
   channel-existence information is revealed, so on servers with dynamic channel spaces a
   denial also hides existence — and `-32023` may be unreachable for such addresses,
   which is conforming.
3. On grant, the server performs a **two-phase transition** (prepare → commit). The host's
   acceptance is part of the transition, not a notification after it: a body MUST NOT be
   moved somewhere its host has refused to deliver.

   **Prepare.**
   a. The server emits `channels/changed` **as a Request** (§14.5's itemized form) with the
      new channel in `added` and the old in `removed`, and awaits the itemized result.
      This is the host's chance to refuse.
   b. If the host rejects the new descriptor (`accepted: false`), the transition **aborts**:
      the connection remains attached to its current channel, unchanged, and the
      `channels/open` request answers `-32017` with `data: { reason: "host_declined" }`.
      Nothing about the agent's presence has moved.
   c. A host that does not implement the inbound Request form, or that fails to answer
      within the server's timeout, is treated as **declining** — fail-closed, per §5.3's
      deny-until-policy discipline. (Servers SHOULD state their timeout; 5s matches
      §10.6's hook guidance.)

   **Commit.** Once accepted:
   d. The **attachment epoch** increments. Every attachment carries a monotonically
      increasing `epoch` (per connection), surfaced on the channel descriptor as
      `metadata.epoch` and echoed on `channels/incoming` messages the server originates
      after the transition.
   e. The current channel is detached. The server persists whatever per-channel state it
      persists on disconnect (read cursors, last-seen).
   f. The requested channel is attached. Exclusivity is preserved — at no observable point
      is the connection attached to two presence channels or zero.
   g. The response carries the new channel's descriptor — the §14.3 response shape,
      unchanged — including the new epoch. Servers with the backscroll extension
      (AUDIT-001 §"history") MAY include `history` exactly as on any open.

   **The cutoff, stated precisely.** The epoch increment in (d) is the cutoff instant.
   - Traffic the server had already dispatched for the OLD channel before the increment
     remains valid and MUST be processed by the host; it carries the old epoch.
   - Traffic for the NEW channel MUST NOT be dispatched before the increment.
   - Host→server messages naming the old channel that arrive after the increment are
     answered `-32023`; the host learns the boundary from the response rather than being
     silently dropped. (§14.5's receipt-time validation already requires validating
     against the *current* attachment; the epoch makes "current" observable to both sides
     rather than inferable.)
   - A `channels/open` for a further world arriving mid-transition is answered `-32024`
     with `data: { reason: "transition_in_progress" }`. Transitions do not nest.
4. A request naming the **currently attached** channel keeps its existing meaning (re-open /
   backscroll / un-blind) — this RFC changes nothing about it.
5. Requests naming a channel the server does not front: `-32023` (*Unknown channel*).
   Attachment failures after grant: `-32024` (*Channel open failed*), and the server MUST
   either remain attached to the original channel or surface the connection as closed —
   never half-attached.
6. **Both addressing forms may express join intent.** `ChannelsOpenParams` (the reference
   library's typed shape) documents `channelId` as *"exact registered channel id, preferred
   over type/address matching"* — generic host surfaces work in ids, and an address-only
   rule would make join unreachable from them. Servers MUST accept either form:
   - `channelId` — an exact reference. The server resolves it against the channels it
     fronts; unknown ids are `-32023`. Ids are server-assigned, so this form is only
     available for channels the host has actually seen (via `channels/list` or a prior
     `channels/changed`).
   - `type`/`address` — a *description* of a destination, resolvable without having seen it
     first. This is the form that composes with RFC-004's dial parameter, and the only form
     that can name a channel the host has never been told about (including, on
     found-on-attach servers, one that does not exist yet — see item 7).

   When both are present, `channelId` wins and a conflicting `address` is `-32602`
   (invalid params) rather than silently ignored — two names for one destination that
   disagree is an authoring bug, not a preference. **This validation MUST precede the
   already-attached check**, or a conflicting pair in which either half happens to name
   the current channel passes silently as a plain re-open. (Found by implementation:
   the natural `channelId === current || address === current` test has exactly this
   hole.)

7. **Founding is not joining.** *(Scope, learned in implementation: this governs
   destinations the AGENT chooses — an in-session join, or a dial-time request. A
   credential arriving at the world its operator minted it into is not founding in this
   sense, even when that world does not exist yet; gating it would refuse every
   connection to a fresh deployment and would be gating the operator, not the agent.)* On servers where attaching to a non-existent channel
   *creates* it (the reference deployment's worlds are founded by their first visitor),
   creation MUST require its own positive authority, distinct from join policy, and MUST
   NOT be reachable by a join that merely happens to name an unused address. A join naming
   a channel that does not exist is `-32023` unless the principal separately holds create
   authority; where it does hold that authority, the server MUST report the distinction in
   the result (`created: true` on the descriptor's metadata) so that founding a world is
   never something an agent does by accident. Rationale: join policy answers *may you be
   here*; creation answers *may you make somewhere new be*, and an operator granting broad
   travel has not thereby granted unbounded world-founding.
8. The host's itemized answer to the `channels/changed` Request (§14.5) governs
   **delivery** on the host side, not attachment: presence is server-side fact from step
   3b onward. A host that rejects the new descriptor has declined to receive the channel's
   traffic, not un-travelled the agent; servers SHOULD log such rejections rather than
   silently diverge.

### 3.3 Dial-time form

A join-capable server SHOULD honor a server-defined dial parameter (for the reference
deployment, RFC-004's `?world=`) as an initial-attachment request evaluated under the
**same policy** as §3.2.1, before the first attachment.

**A denied dial-time destination MUST NOT silently fall back.** The dialer asked to wake up
somewhere; landing elsewhere unannounced means the host believes it is in one place while
its body is in another — the failure this RFC exists to prevent, reintroduced at connect
time. A server MUST do one of:

- **Refuse the connection**, with a close reason naming the denied destination (default,
  recommended); or
- **Attach to the credential's default and announce it prominently**: the first
  `channels/register` MUST carry `metadata.requestedChannel` (what was asked for) beside
  the descriptor of what was granted, and the server MUST record the substitution as an
  authorization event (§3.4).

Silently substituting a destination is non-conforming. A host that cannot distinguish "I am
where I asked to be" from "I am somewhere else" cannot reason about where its agent is, and
no later correction reaches the agent that already spoke.

### 3.4 Audit

Join grants and denials are authorization events. Servers SHOULD log them with the same
permanence as authentication events (§13.2). Presence-shaped servers SHOULD additionally
record departure/arrival in the affected channels' own event streams, where those exist —
transitions are socially legible facts, not just security facts.

## 4. Security considerations

The steelman for attachment-fixed-at-auth deserves stating, because this RFC's design
answers each leg rather than dismissing them:

- **"The channel binding is the blast radius."** A compromised (e.g. prompt-injected) agent
  cannot be lured into a hostile or private channel. — Preserved: policy defaults to deny
  (§3.2.1); with no operator opt-in this RFC changes nothing, and with opt-in the reachable
  set is exactly the operator-approved list, not "anywhere."
- **"Re-dial is the auth ceremony."** — The ceremony re-proves nothing: the credential
  presented on re-dial is the same one the live connection already proved. The security
  content of a transition is the policy decision, which §3.2.1 runs identically. Transport
  teardown was ritual, not enforcement.
- **"One socket, one place."** — Preserved and now normative: §3.2.3's atomicity and §3.2.5's
  never-half-attached rule state the invariant the implicit design only implied.
- **"Human-in-the-loop custody."** — Preserved: the key-holder's policy governs the
  reachable set on both lanes. This RFC moves the *choice point* from mint-time to
  policy-time; it does not remove the steward.

Receipt-time validation (§14.5) applies unchanged: after a join, `channels/incoming` is
validated against the **currently attached** channel, so a revoked join takes effect at the
next message, not the next connection.

## 5. Backwards compatibility

Total. Servers not advertising `channels.join`: no behavior change. Hosts that do not
grant it: no behavior change. Operators granting no policy: no behavior change on a
join-capable server. One MCPL host (eido-cc) already emits the §3.2 request shape and
falls back gracefully on the legacy error; it shares an author with this RFC, so it is a
conformance vehicle, not independent evidence of the reading.

## 6. Out of scope

- **Policy representation** — operator-side (for aid1 deployments, a `worlds` claims
  allowlist is the obvious shape; other deployments will differ).
- **Multi-attachment presence** (hearing two rooms at once). The channel model permits it;
  presence semantics for it deserve their own RFC. §3.2.3 deliberately words exclusivity as
  a property of *presence channels* so non-presence channels (a Discord connector's N text
  channels) remain plural as they already are.
- **Cross-server travel** — a different trust boundary; composes as host behavior (re-dial
  another endpoint) and needs no spec.
- **Entry consent by channel owners** (world-side ACLs) — reachable via the same policy
  hook; representation deferred with it.

One consequence worth naming rather than deferring: on servers where a channel is
**founded on first attach** (the reference deployment's worlds are), join policy is also
*creation* policy — a broadly-policied credential can mint channels by moving. Operators
granting wide policies should know they are granting founding, too.

## 7. Conformance note for the reference deployment

The eidoverse door currently answers foreign-channel opens with `-32004`, a code absent
from §14.6 (AUDIT-001 item 9). Implementing this RFC retires that divergence: unfronted
channels become `-32023`, policy denials `-32017`.
