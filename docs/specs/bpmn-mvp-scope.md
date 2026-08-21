# BPMN diagramming tool — MVP scope spec

Handoff spec for implementation planning. Produced from the Wayfinder map
[BPMN diagramming tool for software devs — MVP scope map](https://github.com/MariosIgkiempor/bpmn/issues/1);
each section below links back to the ticket that resolved it, where the full rationale and
rejected alternatives live.

This is a feature-set and architecture spec, not pixel-level UI design.

## Product framing

A multi-tenant SaaS tool for software developers to sketch BPMN-style process flows and
collaborate on them in real time — closer to "Figma/Miro for dev-audience flowcharts" than a
BPMN-execution-engine modeling tool. There is no execution engine underneath: diagrams are for
communicating a flow between people, not driving a process runtime. This framing (from
[Canvas engine approach](https://github.com/MariosIgkiempor/bpmn/issues/3)) shapes every
scoping call below toward simplicity over spec-completeness.

## Tenancy & sharing model

([Multi-tenant data & sharing model](https://github.com/MariosIgkiempor/bpmn/issues/4))

- **Tenant = `Team`.** The app's existing `Team` model (`app/Models/Team.php`) is the tenant —
  no separate Org layer above it. Personal teams (Jetstream-style, `is_personal` flag) work
  identically to shared teams; no special-casing. Every diagram belongs to exactly one team.
- **Diagram roles**: Owner / Editor / Commenter / Viewer, stored on a `diagram_members`-style
  table holding **overrides only**. A team member's effective role is the explicit override row
  if one exists, else implicit Editor by virtue of team membership (no fan-out write on
  diagram creation).
- **Capabilities**:
  - *Owner* — edit content, delete diagram, manage the sharing link, add/remove/promote/demote
    members (including other Owners).
  - *Editor* — edit diagram content only.
  - *Commenter* — own comments only, no content edits.
  - *Viewer* — read-only.
- **Ownership**: creator becomes Owner on creation; a diagram can have multiple Owners; an
  Owner cannot be demoted/removed while they're the last remaining Owner.
- **No team-admin override** — a team's own Owner/Admin role has no special power over a
  diagram; only the diagram's own Owner(s) manage it.
- **Orphan prevention**: leaving or being kicked from a team is blocked if it would leave any
  diagram without a remaining Owner.
- **Guest access**: one regenerable link per diagram, role selectable as Commenter or Viewer
  only (never Editor/Owner), usable without team membership. Regenerating revokes the old link.
- **Fortify**: no changes needed — it stays auth-only; the existing Team/Membership/Invitation
  layer is the substrate diagram sharing hangs off.

## Canvas engine & data model

([Canvas library licensing](https://github.com/MariosIgkiempor/bpmn/issues/2),
[Canvas engine approach](https://github.com/MariosIgkiempor/bpmn/issues/3))

- **Engine: React Flow (`@xyflow/react`)**, single engine — no hybrid, no
  Excalidraw/tldraw/bpmn-js.
  - Licensing ruled out the alternatives: tldraw requires a paid commercial license for any
    production deployment (sync included, same license); bpmn-js is free but forces a
    permanent, non-removable "powered by bpmn.io" watermark. React Flow is clean MIT, free,
    no watermark.
  - Excalidraw (also clean MIT) was ruled out once freehand ink drawing was confirmed
    unnecessary — annotations can just be another node type on a structured graph, removing
    the main reason to reach for a freeform whiteboard.
- **Everything is a node on one structured graph**: BPMN shapes, text/shape annotations
  (sticky notes, arrows, highlight boxes — no freehand ink), and pasted screenshots are all
  node types on the same React Flow graph. Nodes carry a type + business data; edges carry
  source/target refs and an optional label.
- **XML interoperability is a long-term goal, not MVP** — but the data model is deliberately
  shaped so BPMN XML export/import later is a mapping exercise, not a rewrite. Annotations and
  screenshots are explicitly excluded from that future XML export surface; only BPMN shapes
  themselves need to round-trip.

## BPMN node taxonomy

([BPMN node taxonomy for MVP toolbar](https://github.com/MariosIgkiempor/bpmn/issues/8))

Simplified, dev-focused subset — not the full 60+-element BPMN 2.0 spec. A small, memorable
toolbar beats spec completeness for this audience.

| Category | MVP types |
|---|---|
| Tasks | Generic **Task**, **Service Task** (system/API), **User Task** (a human), **Manual Task** (human, off-system) |
| Events | **Start**, **End**, one **Intermediate Message** event (e.g. "webhook received") |
| Gateways | **Exclusive** (XOR) and **Parallel** (AND) only |
| Sequence flows | Edges carry an optional text label (e.g. "if valid" / "if invalid") |

- **Node data**: every node carries a `label` + optional free-text `description`. No structured
  `assignee` field for MVP — without an execution engine, "who does this" naturally lives in
  prose.
- **Explicitly excluded from MVP**: Script/Business-Rule/Send/Receive tasks, Timer/Error/Signal/
  Conditional intermediate events, Inclusive/Complex gateways, and a separate conditional-flow
  node type (a label on the edge covers it).
- **Swimlanes/pools are deferred post-MVP** — they're a distinct canvas-layout feature (a
  container other nodes drag into/out of), not just another node type. Revisit once the base
  taxonomy is implemented.

## Screenshot paste storage & UX

([Screenshot paste storage & UX](https://github.com/MariosIgkiempor/bpmn/issues/7))

- **Storage**: S3-compatible object storage via **Cloudflare R2** (reuses the Cloudflare
  account already needed for the collaboration infra below; no egress fees; slots into
  Laravel's existing, currently-unused `s3` disk driver in `config/filesystems.php`). Chosen
  over local disk since Forge deployments don't share a filesystem across app servers if
  scaled horizontally, and per-tenant quota/isolation is easier with namespaced object keys.
- **Backend processing**: **Spatie's `laravel-medialibrary`** handles storage/conversions
  rather than hand-rolled upload processing (an implementation detail to carry into planning).
- **Paste UX**: Cmd/Ctrl+V lands the image at the current viewport center, scaled down
  proportionally to a ~600px max dimension if larger, then resizable via the same corner-drag
  handle as other node types. No crop tool for MVP.
- **Limits & processing**: max ~10MB per upload; PNG/JPEG/WebP accepted; always
  resized/compressed on upload (bounds canvas render cost and storage spend as diagrams
  accumulate screenshots).
- **Image node data**: storage key/path (resolved to a URL at render time, not hardcoded),
  `width`/`height`, optional `caption`.

## Comment model & notification UX

([Comment model & notification UX](https://github.com/MariosIgkiempor/bpmn/issues/9))

- **Pin attachment**: Miro-style — pinned in-line on the canvas, not a side panel. Every pin
  stores an (x, y) canvas position as ground truth, plus an optional nullable node reference
  for "this comment concerns this node." No live re-anchoring if the referenced node moves at
  MVP.
- **@-mentions**: notify via **both email and a new in-app notification inbox**, using
  Laravel's existing `Notification` system (same pattern as
  `app/Notifications/Teams/TeamInvitation.php`). The in-app inbox is new work but avoids the
  collaboration feature feeling like an email-only afterthought.
- **Threading**: flat — a root comment plus a simple list of replies, no nested reply-trees.
  Resolved/unresolved is a boolean on the thread, toggleable by any Editor+ (routine
  housekeeping, not gated to Owners like diagram rollback below).
- **Live-sync transport**: kept **separate from the PartyKit/Yjs CRDT document**. Comments are
  append-only relational data (author, text, thread, resolved-state, timestamps) — stored in
  Postgres, broadcast live via a lightweight pub/sub channel (Laravel's own event broadcasting,
  potentially over the same self-hosted PartyKit/Cloudflare setup, or plain Laravel Echo). This
  keeps the CRDT document scoped to diagram content only, never conversation data.

## Real-time collaboration infra

([Real-time collaboration infra research](https://github.com/MariosIgkiempor/bpmn/issues/5),
full sourcing on branch `research/realtime-collab-infra` at
`docs/research/realtime-collaboration-infra.md`;
[infra decision](https://github.com/MariosIgkiempor/bpmn/issues/10))

- **Decision: PartyKit + the official `y-partykit` Yjs CRDT provider**, self-hosted on the
  team's own Cloudflare account.
- **Why over the alternatives**:
  - *Liveblocks* (hosted SaaS, ships an official `@liveblocks/react-flow` package —
    turnkey CRDT sync, live cursors, undo/redo, pinned comments) was the most turnkey option,
    but is hosted-only with no production self-host timeline, and a strong self-host
    preference plus low tolerance for recurring SaaS cost ruled it out beyond its free tier
    (documented as a fallback below).
  - *Laravel Reverb + hand-rolled Yjs* is fully self-hosted and MIT-licensed, matching the
    Forge-first deployment preference, but is genuinely greenfield engineering with no
    reference implementation anywhere — too much MVP engineering risk on the hardest technical
    piece of the product.
  - *PartyKit* is the sweet spot: not Forge, but not a black-box SaaS vendor relationship
    either (deploys to the team's own Cloudflare account — logs visible, account owned,
    ejectable later), and the official Yjs provider already does the CRDT wiring, a fraction of
    the Reverb+Yjs build effort.
- **Scale**: realistic concurrent-editors-per-diagram is small (2–6) — no connection-cap
  pressure; Cloudflare's free tier is expected to comfortably cover MVP usage.
- **Known, accepted risk**: PartyKit is mid-transition into Cloudflare's Durable Objects
  ecosystem — real product-maturity risk, accepted knowingly. **Liveblocks' free tier is the
  documented fallback** if PartyKit's stability becomes a problem post-launch.

## Diagram versioning/history

([Diagram versioning/history under real-time co-editing](https://github.com/MariosIgkiempor/bpmn/issues/11))

In scope for MVP — Yjs's in-session undo/redo alone isn't a sufficient safety net for a shared
multi-editor canvas.

- **Granularity**: periodic snapshots (~every 5–10 min of active editing, plus one on session
  end) persisted to Postgres. Not every-save (too many rows against a continuously-changing
  CRDT doc) and not manual-checkpoints-only (no automatic safety net).
- **Retention**: last **N snapshots per diagram** (e.g. 30) — bounds storage predictably
  regardless of diagram lifespan; exact N is an implementation tuning knob.
- **Rollback**: **Owner-only** (consistent with the capability matrix above — rollback is a
  destructive, shared-state action affecting everyone currently in the diagram, not a personal
  undo). Rollback replaces the live CRDT doc immediately for everyone connected, discarding
  in-progress uncommitted edits — consistent with how any overlapping CRDT edit can already
  clobber another's today. Rollback itself first snapshots the pre-rollback state, so a bad
  rollback can be rolled forward again rather than being a dead end.
- **UI**: minimal version-history list at MVP (timestamp only, author attribution deferred)
  with a rollback action gated to Owners.

## GitHub/Jira link association

([GitHub/Jira link-association model](https://github.com/MariosIgkiempor/bpmn/issues/6))

Manual URL association only for MVP — no live API sync.

- **Attachment target**: an explicit **multi-node selection** (a list of node IDs), not a
  persistent drawn frame/region — a single ticket rarely maps to a single node, but a real
  frame/container primitive would reopen the same complexity already deferred for
  swimlanes/pools. Membership is by node ID, not canvas geometry.
- **Metadata**: just the raw URL, no one-time API fetch (would require per-team OAuth/PAT
  storage and API error handling for what's meant to be dead-simple linking). Human-readable
  label is derived by parsing the URL client-side with a regex (e.g. `owner/repo#123`,
  `PROJ-456`).
- **Multiplicity**: a node-group can have multiple links attached (a simple list) — e.g. tracked
  against both a GitHub PR and a Jira ticket simultaneously.
- **UI**: a badge/icon on the first-selected ("anchor") node in the group, showing a count if
  multiple links exist. Click opens a popover listing each link with its parsed label; hovering
  the badge highlights all nodes in that link's group.

## Deferred / not yet specified

Explicitly out of MVP scope, tracked on the map for later:

- **Figma live-embed** — post-MVP per the user ("eventually"); revisit once this MVP scope
  locks and ships.
- **SaaS billing/plan tiers** — not yet specified.
- **Swimlanes/pools** — deferred out of the BPMN node taxonomy decision; needs its own future
  ticket once the base node taxonomy is implemented.
