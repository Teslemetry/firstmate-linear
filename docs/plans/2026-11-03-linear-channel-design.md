---
title: Linear as Backlog and Captain Channel - Design
type: feat
date: 2026-11-03
artifact_readiness: design-proposal
execution: design
---
# Linear as Backlog and Captain Channel - Design

## Goal

Replace the markdown `tasks-axi` backlog with Linear, and make Linear a two-way, real-time channel between Firstmate and the captain.
Firstmate raises every captain call, blocker, and review-ready PR in Linear exactly once, as a durable record the captain can act on from the Linear UI.
The captain files, dispatches, steers, stops, answers, and approves work from the Linear UI, and Firstmate learns about each action within seconds through webhooks.
Linear's Reviews tab becomes the single queue of PRs waiting on the captain.

## Non-goals

- Workers never get Linear access; hard rule 4 (crewmates never address the captain) is unchanged, so only Firstmate writes to Linear.
- No change to delivery modes, no-mistakes ownership, merge guards, or teardown's landed-work test.
- Linear's own coding sessions (Linear Agent running Claude Code or Codex in Linear's cloud) are not used; Firstmate remains the dispatcher.
- Chat, the captain inbox, mail, voice, and Relay keep working; Linear is added as the preferred durable surface, not as the only one.

## Design summary

```
 Linear workspace                 Dock (always-on, captain-owned)          Firstmate home
 ----------------                 -------------------------------          --------------
 Firstmate app user  --webhook--> verify signature + timestamp
 (actor=app,                      dedupe by Linear-Delivery
  assignable,                     append to ordered event log  <--long-poll-- linear source (procevent)
  mentionable)                    ack AgentSession in <10s                     -> check: linear <seq> wakes
                                  hold OAuth refresh token     --token------> bin/fm-linear.sh (GraphQL)
                  <---------------------------- GraphQL writes ------------- tasks-axi linear adapter
                                                                               fm-captain-hold.sh answers
```

Four new parts carry the design:

1. **A Linear app** named Firstmate, installed with `actor=app` and the `app:assignable` and `app:mentionable` scopes, so it can be delegated issues and @mentioned while the captain stays the human owner.
2. **A dock**: a tiny always-on webhook receiver the captain owns, because Linear requires a webhook response within 5 seconds and an Agent Session acknowledgement within 10 seconds, while a Firstmate home is behind NAT and is not always running.
3. **A `linear` tasks-axi adapter**, selected by `.tasks.toml` `backend = "linear"`, so every existing backlog caller (`fm-spawn.sh`, `fm-teardown.sh`, `fm-captain-hold.sh`, Bearings, handoff) keeps using tasks-axi verbs unchanged.
4. **A Linear channel on the Firstmate side**: a process-event source that turns dock events into durable `check: linear ...` wakes, a `bin/fm-linear.sh` client for outbound activities, and a `linear-channel` skill that owns the semantics.

The existing seams make this mostly additive: tasks-axi already supports non-markdown adapters (Beads), `fm-captain-hold.sh answers` is already the single keyed-answer intake every channel feeds, and `fm-procevent.sh` already turns long-polling sources into durable wakes.

## Why a dock instead of webhooks straight to the home

Linear delivers a webhook only if the endpoint answers 200 within 5 seconds, retries just three times (after 1 minute, 1 hour, and 6 hours), and may disable a webhook that keeps failing.
An Agent Session that gets no activity or `externalUrls` update within 10 seconds of `created` is shown to the captain as unresponsive.
A Firstmate home is a laptop or desktop session that sleeps, restarts, and holds a session lock, so pointing Linear straight at it would lose events and show unresponsive sessions whenever the home is down.

The dock solves both with no Firstmate logic of its own:

- It verifies `Linear-Signature` (hex HMAC-SHA256 of the raw body with the webhook signing secret) and rejects a `webhookTimestamp` more than 60 seconds from its clock.
- It deduplicates by the `Linear-Delivery` header and appends each accepted event to an ordered log with a monotonic sequence number, then answers 200.
- On an `AgentSessionEvent` `created`, it immediately emits one ephemeral `thought` activity, worded by whether a home has polled recently: "On deck, reviewing this now" or "Firstmate is offline; this is queued and will be picked up when the session resumes".
- It drops events whose actor is the Firstmate app user itself, so Firstmate's own writes never echo back as wakes.
- It owns the OAuth refresh token and lends short-lived access tokens to an authenticated home, so no long-lived Linear credential sits in a home's `.env`.
- It exposes `GET /events?after=<seq>&wait=<s>` (long-poll) and `POST /ack` to a home authenticated by a per-home secret.

Recommended hosting is a Cloudflare Worker with a Durable Object (or D1) for the log, deployed from the captain's own account, because Linear content is captain-private.
Two alternatives are worth naming:

- **Extend the existing myfirstmate.io Relay** with a Linear connector, which reuses the pairing-token and poll machinery already in `bin/fm-x-poll.sh`, at the cost of private Linear content transiting a shared service.
- **A tunnel straight to the home** (for example `cloudflared`), which is fine for development but reintroduces the offline gaps above.

## Linear workspace shape

### Team, homes, and projects

- One Linear team, `Firstmate` (key `FM`), holds every Firstmate work item.
- Each registered Firstmate project maps to a Linear Project of the same name, recorded in `data/projects.md` as a `linear_project=<id>` field, and this is how intake resolves the project for a Linear request.
- Each Firstmate home owns its items through a label `fm-home:<home-id>` (`fm-home:main` for the primary), rather than through separate teams, so a secondmate handoff is a relabel and issue identifiers stay stable.
- An issue with no Project is resolved by the ordinary intake rule (registry, work under way, code), and a genuinely ambiguous one gets a single `select` elicitation listing candidate projects.

### Workflow states

| Linear state | Linear type | Firstmate meaning |
| --- | --- | --- |
| Triage | triage | Captain filed it, not yet authorized for dispatch. |
| Backlog | backlog | Queued, unprioritized, or deferred until a date. |
| Todo | unstarted | Queued and dispatch-eligible once dependencies and capacity clear. |
| In Progress | started | In flight: a worker is live (set by `fm-spawn.sh`). |
| Captain's Call | started | Held for the captain: a decision, blocker, or credential is waiting on the captain. |
| In Review | started | A PR or ready local branch is waiting on the captain's review or merge word. |
| Done | completed | Landed and cleaned up (set by `fm-teardown.sh`). |
| Canceled | canceled | Withdrawn by the captain or closed as moot with evidence. |

Captain's Call and In Review are both assigned to the captain, so they land in the captain's My Issues and Inbox, and two saved views (`Captain's Call`, `Review queue`) give one-click lists.
The team's GitHub workflow automations must be configured so a merged PR does not move an issue to Done on its own, because `fm-teardown.sh` owns completion only after the landed-work test; the adapter still treats an already-Done issue as an idempotent close.

### Labels

- `fm-home:<id>`: owning home.
- `fm:scout`: deliverable is a report, not a PR; absent means ship (the default).
- `mode:no-mistakes`, `mode:direct-PR`, `mode:local-only`, `yolo:on`, `yolo:off`: an explicit per-task captain instruction for delivery mode and merge posture, overriding the registry for that task only, exactly as a chat instruction would.
- `fm:captain` marks a row Firstmate minted purely to carry a captain call (tasks-axi `--kind captain`).

## Backlog adapter: tasks-axi verbs on Linear

The adapter lives with tasks-axi (the package that already hosts the markdown and Beads adapters), and Firstmate selects it with:

```toml
backend = "linear"

[linear]
team = "FM"
home_label = "fm-home:main"
token_command = "bin/fm-linear.sh token"
```

### Identity

The Firstmate task id is the lowercased Linear identifier, for example `fm-142`.
It satisfies the task-id and branch rules, and a ship branch such as `fm/fm-142` lets Linear's GitHub integration auto-attach the PR to the issue with no extra step.
The adapter also accepts the uppercase identifier and the issue UUID, and a renamed or moved issue keeps resolving because Linear redirects old identifiers.

### Field ownership

Linear is shared state, so every field has exactly one writer, which removes merge conflicts between the captain and Firstmate:

| Field | Owner | Notes |
| --- | --- | --- |
| Title, description | Captain for captain-filed issues; Firstmate for Firstmate-filed ones | Captain edits to a live task's description arrive as a mid-task ask. |
| Priority, project, cycle, estimate | Captain | Firstmate reads priority to order dispatch. |
| State | Firstmate, except Triage-to-Todo, Canceled, and the captain's answer moves listed below | Captain moves are inputs, not conflicts. |
| Assignee | Firstmate sets the captain for Captain's Call and In Review; otherwise captain | |
| Delegate | Captain delegates to Firstmate; Firstmate sets itself when it files work | |
| Relations (blocks / blocked by) | Either | Maps directly to tasks-axi blockers. |
| Firstmate record attachment | Firstmate only | See below. |

### The Firstmate record attachment

Each issue carries one attachment with the stable URL `firstmate://task/<task-id>` whose `metadata` JSON holds every machine field tasks-axi stores today: kind, hold reason (already encoded by `bin/fm-hold-reason-lib.sh`), hold-set stamp, hold origin, `hold_until`, delivery mode, yolo posture, report link, deliverable, and the legacy-id migration marker.
`attachmentCreate` with the same URL is an upsert, which makes every adapter write idempotent and keeps machine state out of the captain-readable description.
The considered task body lives in the description below a `---` rule when Firstmate owns the issue, and `update --archive-body` posts the superseded body as a collapsed comment before replacing it.

### Verb mapping

| tasks-axi verb | Linear operation |
| --- | --- |
| `add` | `issueCreate` in the team with the home label, state Todo (or Backlog), plus the record attachment. |
| `show`, `list`, `ready` | Issue query filtered by team and home label; `ready` excludes blocked, held, and dated rows. |
| `start` | State In Progress, delegate Firstmate. |
| `done --report/--evidence` | State Done, record attachment updated, report attached (see scout reports). |
| `hold --until` | State Captain's Call, assignee captain; with `--until`, state Backlog plus `hold_until` in the record. |
| `unhold` | State Todo (or back to In Progress when a live worker owns it), assignee cleared. |
| `update` | Description and record attachment update. |
| `mv` (secondmate handoff) | Swap the `fm-home:` label on every listed issue in one `issueBatchUpdate`. |
| `rm` | Archive the issue. |
| `public-followup` | Stays in the record attachment; Relay obligations are unchanged. |

The adapter must pass the existing compatibility probe in `bin/fm-tasks-axi-lib.sh` (`update --archive-body`, multi-id `mv`) before bootstrap treats it as usable.

### Migration

A one-time `tasks-axi migrate --from markdown --to linear` imports `data/backlog.md` and `data/done-archive.md`, writing the existing `migrated from data/backlog.md id <legacy id>` marker that `fm-captain-hold.sh complete` already resolves for the Beads migration.
Live `state/<id>.meta` records keep their old ids; the migration writes an alias into each record attachment so `show <old-id>` resolves until those tasks are torn down.

## Firstmate to captain: raising things once

"Once" has a precise meaning: each genuine gate produces one Linear record and one notification per hold lifecycle, and neither a restart, a re-hold, nor a repeated wake raises it again.

### Captain calls (decisions, blockers, credentials)

1. Firstmate holds the task exactly as today with `bin/fm-captain-hold.sh hold <id> --reason ...`; the adapter moves it to Captain's Call and assigns the captain, which is the single Linear notification.
2. Firstmate opens one Agent Session on that issue (`agentSessionCreateOnIssue`) and emits one `elicitation` activity carrying the question, the evidence, the consequence, and the recommendation, worded under AGENTS.md section 9.
3. When the call has discrete options, the elicitation uses the `select` signal with `signalMetadata.options`, each option's `value` being the keyed-answer mode (`done`, `release`, or an option token) and its `label` the plain-language choice.
4. The session id and the hold lifecycle identity from `fm-captain-hold.sh open <id> --identity` are recorded in the record attachment, and Firstmate never emits a second elicitation for the same identity; a fresh re-hold starts a new identity and therefore exactly one new elicitation.
5. A dated deferral resurfaces once on its date by moving back to Captain's Call, which is the only repeat notification.

When Linear is enabled, chat carries at most a one-line pointer with the issue URL for a present captain, so the same call is not raised twice in two places.

### Review-ready work

When `ship-landing` records a PR (`bin/fm-pr-check.sh`):

- The issue moves to In Review, assigned to the captain, and the PR is attached (automatically through the branch name, and explicitly through `attachmentLinkGitHubPR` as a fallback).
- With yolo off, Firstmate requests the captain's GitHub review on the PR, which puts it in the captain's Linear Reviews tab under "For me"; the captain's GitHub login lives in `config/linear.json` as `captain_github_login`.
- The issue's Agent Session gets `externalUrls` for the PR and a `select` elicitation with Merge, Request changes, and Not yet.
- A local-only ready branch has no PR, so its elicitation names the branch and offers Land and Not yet.
- With yolo on, Firstmate merges green work through `bin/fm-pr-merge.sh` as today and posts a `response` with the full PR URL instead of asking.

Linear's Reviews tab therefore lists every PR awaiting the captain, the In Review view lists every ready local branch too, and both clear themselves when the work lands.

### Progress, results, and failures

- `fm-spawn.sh` start sets the session `plan` from the brief's units and `externalUrls` for the branch; plan updates replace the whole plan, are sparse, and track real milestones only, never routine progress.
- A finished scout posts its report as a Linear Document linked from the issue (the `data/<id>/report.md` copy stays authoritative locally) and a `response` relaying the findings, not just a completion notice.
- A real failure after the relevant playbook is exhausted is an `error` activity with the evidence; automatic retries and supervision mechanics are never posted.
- Agent activities are captain-facing chat, so they follow the section 9 translation rules and the captain-address rule, while issue titles, descriptions, Documents, and attachment metadata are non-chat artifacts and carry no direct address.

## Captain to Firstmate: acting from Linear

Linear content is trusted captain input, scoped to what the captain actually does: an event counts as the captain's word only when its actor is a human workspace member listed in `config/linear.json` `trusted_actors` (default: every human member, per the captain's own instruction).
Content that arrives through Linear but was authored elsewhere (GitHub-synced comments, Slack-synced threads, Linear Asks or email intake, customer requests, other integrations' bots) has a non-member or bot actor and is treated as untrusted data, exactly as a GitHub comment is today.

| Captain action in Linear | Webhook | Firstmate response |
| --- | --- | --- |
| Files an issue in Triage or Backlog without delegating | `Issue` create | Recorded as a queued item the captain owns; no dispatch. |
| Delegates an issue to Firstmate, or @mentions Firstmate asking for work | `AgentSessionEvent` `created` | Dispatch authorization: intake, brief, spawn (below). |
| Moves an issue from Triage to Todo | `Issue` update | Dispatch-eligible on the next queue re-evaluation. |
| Replies in an Agent Session on live work | `AgentSessionEvent` `prompted` | Mid-task ask: appended verbatim to the brief's `## Captain's intent` and steered to the worker with `bin/fm-send.sh`. |
| Answers an elicitation (select or free text) | `AgentSessionEvent` `prompted` | Keyed answer piped to `bin/fm-captain-hold.sh answers --source linear:<session-id>`. |
| Sends a stop request | `prompted` with `stop` signal | `bin/fm-control.sh <id> interrupt`, then a `response` confirming the stop and the state left behind; nothing is torn down. |
| Moves an issue to Canceled | `Issue` update | Worker exited through `fm-control.sh`; unlanded work is preserved and discarding it is asked for separately, per hard rule 3. |
| Approves or merges a PR in Linear Reviews | GitHub, then Linear `Attachment` update | A merge done by the captain is observed by the existing merge poll; an approval alone is review evidence, not a merge order. |
| Asks a question (@Firstmate "what's the state of X?") | `AgentSessionEvent` `created` | Answered as a `response` from durable records only, like `fm-inbox.sh status`, with no dispatch. |
| Edits priority, relations, or project | `Issue` update | Backlog re-evaluation; relations become tasks-axi blockers. |

### Intake from a delegated issue

1. The dock has already acknowledged the session; Firstmate drains the `check: linear` wake.
2. The project resolves from the issue's Linear Project, otherwise by the ordinary intake rule, otherwise by one `select` elicitation.
3. Labels supply ship or scout and any per-task mode or yolo instruction; the registry supplies the rest, exactly as AGENTS.md section 7 requires.
4. The brief's `## Captain's intent` is the issue title, description, and session prompt verbatim, built from the session's `promptContext`, without widening.
5. `bin/fm-spawn.sh` runs with the issue's task id; the adapter moves the issue to In Progress, and the session's `plan` and `externalUrls` are set.
6. If the issue is out of a secondmate's scope or capacity blocks dispatch, the session gets one `thought` saying where it was routed or why it is queued.

### Answers feed the existing single intake

The Linear channel is a captured-answer source in the sense of `captain-hold-lifecycle`: it is bound with `bin/fm-captain-hold.sh bind linear:<home-id>` before the source is armed, and it turns each elicitation reply into `<task-id>\t<answer>\t<label>\t<mode>` lines.
A `select` reply maps directly to its option `value`; a free-text reply is passed with an empty mode so it is recorded as the captain's exact words and closes nothing it cannot prove, and Firstmate then interprets it as a normal captain answer.
The reserved `reconcile` value is never offered as an option, and the attended-only `release` rule is unchanged because a Linear reply is the captain being present.
A merge-approval answer releases the hold and Firstmate then runs `bin/fm-pr-merge.sh`, so merge proof and guards stay where they are.

## Real-time loop and reconciliation

### Webhook subscriptions

- Agent session events (`created`, `prompted`).
- Inbox notifications (`AppUserNotification`: mentions, assignment to Firstmate, new comments on delegated issues, status changes).
- Data changes for `Issue`, `Comment`, `IssueLabel`, `Attachment`, `Project`, and `Reaction` in the Firstmate team.
- Permission changes and `OAuthApp` `revoked`, which become a single credential-needed escalation.

### Home side

A built-in process-event adapter `linear` long-polls the dock and normalizes each event into a durable `check: linear <seq> <kind> <task-id-or-issue>` wake, using the same exactly-once-recovering journal pattern as the mail plane.
The source is registered per home and filtered by that home's `fm-home:` label, plus unlabeled new issues for the primary home, which owns routing.
`fm-procevent.sh handled` acknowledges a sequence only after the wake is handled, so a crash replays rather than drops.

### Healing missed events

Webhooks can still be missed while the dock is down or after Linear disables a failing webhook, so Linear itself, not the event stream, is the source of truth.
Each heartbeat runs one bounded catch-up query (`issues(filter: {team, updatedAt > last_seen})` plus open Agent Sessions) and synthesizes wakes for anything the stream did not deliver, keyed so a later real event for the same change deduplicates.
A webhook the dock finds disabled, or a catch-up gap, is reported once as a monitoring problem rather than silently worked around.

### Rate limits

Linear's GraphQL API is rate limited by request count and query complexity per app.
The adapter batches reads per heartbeat, caches team states and labels, and never polls Linear for freshness when the dock stream is healthy.

## Secondmates

There is one Linear app installation and one dock per captain workspace.
The primary home routes each newly delegated issue by secondmate scope, as today, by relabeling it `fm-home:<secondmate>`; the secondmate's own `linear` source then receives its later events, and its backlog adapter sees only its label.
Secondmate outcomes still reach the captain through the parent channel, and the primary decides what is posted to Linear, so a secondmate never raises a duplicate of a call the primary already raised.

## Configuration and files

| Path | Kind | Purpose |
| --- | --- | --- |
| `.tasks.toml` or home `.tasks.toml` | Shared / private | `backend = "linear"` and the `[linear]` block. |
| `.env` `FM_LINEAR_DOCK_URL`, `FM_LINEAR_DOCK_SECRET` | Private | Opt-in and dock credential; absent means the channel is off. |
| `config/linear.json` | Private | Team key, `trusted_actors`, `captain_linear_user`, `captain_github_login`, state and label ids. |
| `bin/fm-linear.sh` | Shared | Token, GraphQL client, activity and session helpers; header owns its contract. |
| `bin/fm-procevent-linear.sh` | Shared | Built-in process-event adapter for the dock stream. |
| `.agents/skills/linear-channel/SKILL.md` | Shared | When to raise, how to word activities, how to interpret captain actions. |
| `docs/linear.md` | Shared | Operator setup: app creation, scopes, webhook categories, dock deploy. |
| Dock (separate repository) | Captain-owned | Worker source, deploy script, and secret rotation. |

`bin/fm-bootstrap.sh` gains a `LINEAR` diagnostic: off, healthy, dock unreachable, token revoked, or webhook disabled.

## Rollout

1. **Workspace and app**: create the team, states, labels, views, and the Firstmate app; deploy the dock; confirm signed delivery and the 10-second acknowledgement with a test delegation.
2. **Read-only mirror**: keep markdown authoritative and run the adapter in mirror mode, writing Linear from tasks-axi transitions, to validate the mapping on real work.
3. **Cut over the backlog**: migrate, switch `backend = "linear"`, and keep `data/backlog.md` as a frozen read-only export for one release.
4. **Inbound channel**: arm the `linear` source; delegation, prompts, stop, cancel, and questions go live.
5. **Captain calls and reviews**: elicitations bound to the keyed-answer intake, In Review flow, GitHub review requests, and merge-from-Linear answers.
6. **Default surface**: the section 9 escalation rule names Linear as the durable surface when enabled, and Bearings reads the same backlog through the adapter with no change of its own.

Each step is independently reversible by switching `.tasks.toml` back and disarming the source; the migration never deletes the markdown files.

## Risks and open questions

- **Where the adapter lives.** tasks-axi is an external package; if a Linear adapter cannot land there, a Firstmate-side shim behind `bin/fm-tasks-axi.sh` is the fallback, at the cost of a second backlog implementation to keep compatible.
- **Who else is in the workspace.** "Linear is trusted" is safe only if `trusted_actors` matches reality; if teammates join, the default must narrow to the captain's user id before they can delegate work or answer calls.
- **Merge from an approval.** This design treats a captain's PR approval as review evidence, not a merge order; a home that wants approval to mean merge can opt in with `config/linear.json` `approval_means_merge`, which applies only with yolo off.
- **Linear GitHub automations.** The team's "PR merged moves to Done" automation must be off, or the backlog will claim completion before cleanup proves the work landed.
- **Agent Plans are a technology preview.** The design uses them only for display, so a breaking change degrades the view and nothing else.
- **Dock privacy and availability.** The dock holds the OAuth token and a short event log of private content; it should retain events only until acknowledged plus a short window, and its own outage is covered by the heartbeat catch-up.
- **Cost of a second identity.** Linear installs agent apps without a billable seat, but every Agent Session activity is visible to anyone in the team, which is another reason to keep posts sparse and outcome-only.
