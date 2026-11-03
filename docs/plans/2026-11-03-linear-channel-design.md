---
title: Linear as Backlog and Captain Channel - Design
type: feat
date: 2026-11-03
artifact_readiness: design-proposal
execution: design
---
# Linear as Backlog and Captain Channel - Design

## Goal

Make Linear the place where Firstmate raises captain calls, blockers, and review-ready work exactly once, and where the captain files, dispatches, steers, stops, answers, and approves work from the Linear UI.
Linear webhooks keep Firstmate current within seconds, and Linear's Reviews tab becomes the single queue of PRs waiting on the captain.

The overriding constraint is upstream compatibility: Firstmate must keep fast-forwarding through `/updatefirstmate` with no local patches.

## Compatibility verdict

| Approach | Firstmate files changed | Effect on upstream updates | Verdict |
| --- | --- | --- | --- |
| Overlay: new services beside Firstmate that use only its public commands | None | Updates keep fast-forwarding; breakage is limited to a small, tested contract surface | Recommended |
| Replace the backlog with a Linear tasks-axi adapter | None in Firstmate if the adapter lands in tasks-axi; a few fixes otherwise | Depends on upstream accepting the adapter and keeping non-markdown backends working | Later, as an upstream contribution |
| Fork: edit `AGENTS.md`, skills, and `bin/` for Linear | Many | `/updatefirstmate` is a guarded fast-forward, so any local commit stops updates until it is rebased; `AGENTS.md` and `bin/` change almost daily upstream | Avoid |

The overlay is how the existing Teslemetry tools already work: `alert-router`, `backlog-mirror`, and `decision-desk` change nothing in Firstmate and talk to it only through `bin/fm-inbox.sh`, `bin/fm-tasks-axi.sh`, and `bin/fm-captain-hold.sh`.
This design extends that pattern to Linear.

### The contract surface the overlay depends on

Every coupling point is a command or record format that upstream already documents as an interface for other channels:

| Interface | Used for | Upstream owner |
| --- | --- | --- |
| `bin/fm-inbox.sh note --request-id --json` | Every inbound Linear event that needs Firstmate | Script header ("the captain's out-of-band capture surface") |
| `bin/fm-inbox.sh receipts --all-replies --after` | Firstmate's answers back to a Linear request | Script header; already used by `alert-router` `/mcp` |
| `bin/fm-captain-hold.sh answers --source` | A captain's selected answer in Linear | "One keyed-answer intake, fed by every channel" |
| `bin/fm-tasks-axi.sh add` and `show` | Filing a delegated issue as a queued row; reading rows | `docs/configuration.md` "Backlog backend" |
| `data/backlog.md` markdown grammar | Mirroring rows (tasks-axi `list` has no `--json` yet) | tasks-axi markdown grammar, already parsed by `backlog-mirror` |
| `state/<id>.meta` `pr=` line | Detecting review-ready work | `bin/fm-pr-check.sh` |

The last two are the weakest links because they are file formats rather than commands.
A nightly contract test in `Teslemetry/tools` CI runs the dock and mirror against a scratch home built from upstream Firstmate `main` with the real scripts, so an upstream change that breaks any row above fails a test before it reaches the host.
Upstreaming `tasks-axi list --json` (its issues 33 and 55) would remove the markdown dependency.

### What the overlay cannot do

- The markdown backlog stays the source of truth, and Linear is a live projection plus an input surface; a captain edit to a Firstmate-owned field in Linear is shown as divergence, never silently written back.
- Firstmate's escalation habits come from `AGENTS.md`; the overlay steers them through the home's private `data/captain.md` (the upstream-sanctioned place for captain preferences) instead of editing the contract.
- Fine-grained worker milestones are read from records, not posted by workers, so Linear shows real milestones (dispatched, PR opened, blocked, landed) rather than a live narrative.

## Architecture

```
 Linear (Teslemetry workspace)          clickstack host (pm2, behind Caddy at fm.ba.id.au)          Firstmate homes
 -----------------------------          ---------------------------------------------------          ---------------
 Firstmate app user  --webhook------->  dock        verify, dedupe, ack sessions in <10s
 (actor=app,                                         fm-tasks-axi.sh add (delegated issues)   ----->  data/backlog.md
  assignable,                                        fm-inbox.sh note                         ----->  state/inbox + check wake
  mentionable)                                       fm-captain-hold.sh answers               ----->  captain-held rows
                     <--activities---                fm-inbox.sh receipts (replies)           <-----  fm-inbox.sh reply
                                                     /mcp: app-identity tools                 <-----  Firstmate agent
                     <--issue sync---   linear-mirror  reads backlogs and pr= records         <-----  backlog, state/*.meta
 Linear MCP (as the captain)  <------------------------------------------------------------- Firstmate agent (reads)
```

All new code lives in `Teslemetry/tools`; nothing is added to this repository.

### `dock/`: the Linear receiver

`dock/` starts as a copy of `alert-router/`, as the captain asked, keeping its rules: zero runtime dependencies, secrets only in the host dotenv, pm2 as the only runtime, write-then-acknowledge, and delivery only through upstream scripts.
It runs as its own pm2 app on its own port, behind a new Caddy route, so a Linear problem can never take alerting down.

Routes:

- `POST /linear/webhook` verifies `Linear-Signature` (hex HMAC-SHA256 of the raw body), rejects a `webhookTimestamp` more than 60 seconds off, and keys every write on `Linear-Delivery`, then answers within Linear's 5-second limit.
- `GET /linear/oauth/callback` completes the one-time `actor=app` install with the `app:assignable` and `app:mentionable` scopes and stores the refresh token in the dock's private state; the dock alone refreshes access tokens.
- `POST /mcp` (bearer key, same rules as `alert-router`) exposes app-identity tools to the Firstmate agent, described below.
- `GET /healthz` reports webhook configuration, token freshness, each home's inbox writability, and the last event and last catch-up times.

Event handling:

| Linear event | Dock action |
| --- | --- |
| `AgentSessionEvent` `created` from a delegation or @mention | Emit an ephemeral `thought` at once ("On deck, queued for Firstmate"), then file the row and note below. |
| `AgentSessionEvent` `prompted` on a session tied to a row | Note carrying the captain's exact words, keyed `lin-<activity-id>`. |
| `prompted` with the `stop` signal | Note marked as a stop request for that task; Firstmate interrupts through `bin/fm-control.sh`. |
| `prompted` answering a `select` elicitation | Pipe `<task-id>\t<words>\t<label>\t<mode>` into `fm-captain-hold.sh answers --source linear:<session-id>`, then note so Firstmate wakes and acts on the answer. |
| `Issue` update by the captain to Canceled, Todo, priority, or relations | Note describing the change for that task. |
| Any event whose actor is not the captain's Linear user | Dropped, or noted as `EXTERNAL <actor>:` text when it touches a Firstmate issue, the same convention `alert-router` uses for GitHub. |
| Any event whose actor is the Firstmate app itself | Dropped, so Firstmate's own writes never echo back. |

A delegated issue becomes a queued row before Firstmate is woken: the dock runs `fm-tasks-axi.sh add eng-123 "<title>" --repo <repo> --body "Linear: <url>"`, which is what the captain would do by hand.
The task id is the lowercased Linear identifier, so a ship branch such as `fm/eng-123` makes Linear's GitHub integration link the PR to the issue automatically.
The row add and the note are both keyed and idempotent, so a webhook retry replays rather than duplicates.
The note body follows `alert-router`'s style: a short plain-text summary with the issue URL, the captain's description verbatim, and the delivery id, never the raw payload.

Routing between homes reuses `alert-router`'s model: delegated work goes to the primary home, which routes to a secondmate by scope as it does today, and an issue labeled for a secondmate's area can be filed straight into that home.

### Replies without any Firstmate change

`AGENTS.md` already tells Firstmate to answer a note that needs a durable answer with `bin/fm-inbox.sh reply <id>`.
The dock polls `fm-inbox.sh receipts --all-replies` for its `lin-` request ids, the way `alert-router`'s `get_replies` does, and posts each reply to the matching Agent Session as a `response` activity.
A captain who asks Firstmate something in Linear therefore gets the answer in Linear with no new Firstmate behavior.

### `/mcp`: Firstmate acting as itself in Linear

The Linear MCP acts as the captain, and Linear does not notify a user about their own actions, so a call raised through it would never notify the captain.
Raising things once with the captain therefore needs the Firstmate app identity, which the dock holds.
Its MCP server gives the Firstmate agent a few narrow tools, all executed with the app token:

| Tool | Effect |
| --- | --- |
| `linear_raise` | Move a task's issue to Captain's Call, assign the captain, and emit one `elicitation` (with `select` options when the call has discrete choices), keyed on the task's hold lifecycle so a repeat call is a no-op. |
| `linear_update` | Emit a `thought`, `response`, or `error` on a task's session, or replace its `plan` and `externalUrls`. |
| `linear_review` | Move a task's issue to In Review with its PR or ready-branch link and a Merge / Request changes / Not yet elicitation. |
| `linear_file` | Create an issue as Firstmate for work Firstmate discovers, delegated to itself. |

None of them can merge, close a captain call, or edit the backlog; those stay with Firstmate's own guarded commands.
A CLI wrapper over the same code serves scripts and harnesses without MCP support.

### `linear-mirror/`: raising things from structured state

`linear-mirror/` is the Linear sibling of `backlog-mirror/`, reusing its parser and its rules (the backlog is the truth, key on the id alone, conflicts are reported and not written, divergence is shown and not resolved).
It runs every minute under a systemd timer and projects each home's structured state onto Linear:

| Firstmate state | Linear projection |
| --- | --- |
| Queued row | Todo (Backlog when dated) |
| In flight row | In Progress, delegated to Firstmate |
| Open captain hold | Captain's Call, assigned to the captain, with one elicitation per hold lifecycle |
| `pr=` recorded in `state/<id>.meta` | In Review, assigned to the captain, PR attached |
| Done row | Done, with the deliverable link |
| Row with no Linear issue | A new issue created as Firstmate, so Firstmate-discovered work appears too |

Because the mirror derives every raise from records Firstmate already writes, captain calls and review-ready PRs reach Linear even if the agent never calls `/mcp`, and they are raised exactly once because each is keyed on the task id plus its hold lifecycle or PR URL.
The `/mcp` tools are for richer wording and timing; the mirror is the safety net.

### Healing missed events

Linear retries a webhook only three times and can disable one that keeps failing, so Linear, not the event stream, is the source of truth for captain actions.
The mirror's run also performs a bounded catch-up query for issues and Agent Sessions updated since its last cursor, and hands anything the dock never saw to the dock's normal handling, keyed so a late real webhook deduplicates.

## Linear workspace shape

The workspace has one human member (the captain) and two installed apps (Cyrus and Codex), so the trust rule is exact: only events whose actor is the captain's Linear user are the captain's word.
Everything else - other apps, GitHub-synced or Slack-synced comments, Asks or email intake, and anything from the Support team - is untrusted text.

- **Team:** Firstmate works in the existing Engineering team (`ENG`) rather than a separate team, so the captain's real backlog and Firstmate's backlog are one list; the dock ignores other teams.
- **Repository:** resolved per issue from the Firstmate project registry, with Linear's `issueRepositorySuggestions` query and an `repo:<name>` label as hints, and one `select` elicitation when it is genuinely ambiguous.
- **Statuses:** Engineering already has Triage, Backlog, Todo, In Progress, In Review, Done, Canceled, and Duplicate; add one started-type status, Captain's Call.
- **Labels:** `fm:scout` (deliverable is a report), `mode:direct-PR` / `mode:local-only` / `mode:no-mistakes`, and `yolo:on` / `yolo:off` as explicit per-task captain instructions.
- **Views:** Captain's Call (assigned to me, Captain's Call) and Review queue (assigned to me, In Review).
- **Git automations:** turn off "PR merged moves the issue to Done" for Engineering, because Firstmate's cleanup marks work done only after it has proved the work landed.
- **Other agents:** Cyrus and Codex can also be delegated issues; delegating one issue to two agents would start two independent efforts, so Firstmate treats an issue delegated elsewhere as not its own.
- **Guidance:** Linear's workspace and team agent guidance appears in every session's `promptContext`; it is a good place for captain-written conventions such as label meanings.

## The captain's Linear workflow

| Captain does | Result |
| --- | --- |
| Files an issue and leaves it undelegated | Nothing; it is the captain's own note. |
| Delegates an issue to Firstmate, or @mentions Firstmate with a request | Immediate acknowledgement, a queued row, and normal Firstmate intake and dispatch. |
| @mentions Firstmate with a question | An answer from Firstmate's records, posted back in the thread. |
| Replies in the thread of live work | The words are passed to the worker as a mid-task instruction. |
| Picks an option on a Captain's Call | The call closes through the keyed-answer intake and the work proceeds. |
| Sends a stop request | The worker is interrupted, nothing is deleted, and Firstmate confirms what was left. |
| Moves an issue to Canceled | The worker stops; discarding unfinished work is asked for separately. |
| Picks Merge on an In Review item | Firstmate merges through its guarded merge command and confirms with the PR URL. |
| Approves or merges in Linear Reviews | A merge by the captain is seen by Firstmate's existing merge monitoring; an approval alone is review evidence, not a merge order. |

When Firstmate opens PRs from the captain's GitHub account, they already appear in the captain's Reviews tab under Created; when it uses a separate account, `linear_review` also requests the captain's GitHub review so they appear under For me.
Either way, the In Review status plus the Review queue view separates the PRs awaiting the captain's word.

## Where the Linear MCP fits

| Use | Value | Rule |
| --- | --- | --- |
| Firstmate intake | High: read the full issue, comments, attached screenshots (`extract_images`), linked Documents, and project context before writing instructions | Read freely. |
| Captain-attributed artifacts | Medium: post scout reports as Linear Documents and periodic project status updates | Allowed, since these are records rather than notifications. |
| Reading Linear Reviews | Medium: `get_diff_threads` shows review comments the captain left in Linear (they also sync to GitHub) | Read only. |
| Merging from Linear | None | Deny `merge_diff` and `update_diff`; every merge goes through Firstmate's guarded merge command. |
| Raising calls or replying | None | Use the dock's `/mcp`, because MCP writes appear as the captain and do not notify the captain. |
| Workers | None | Workers get no Linear access; only Firstmate talks to the captain. |
| Building this | High: set up labels, views, and test issues, and verify mirror output against the live workspace | Use during development. |

The Firstmate agent's harness gets two Linear connections: the official Linear MCP (as the captain, mostly reads) and the dock's `/mcp` (as the Firstmate app, writes that notify the captain).

## Steering Firstmate without editing `AGENTS.md`

The home's private `data/captain.md` gets a short Linear section, which Firstmate loads at every session start:

- A note whose request id starts with `lin-` came from the captain in Linear; answer it with `fm-inbox.sh reply`, and use the Linear task id the dock already filed.
- Raise captain calls and review-ready work with the dock's `linear_raise` and `linear_review` tools, worded to the usual captain-facing standard; chat gets at most a one-line pointer with the issue URL.
- A Linear stop or cancel note means interrupt or exit the worker through `bin/fm-control.sh`, never clean up unfinished work.

## Rollout

1. **Workspace and app:** the captain creates the Firstmate OAuth app (webhooks on, Agent session events, Inbox notifications, Permission changes, and Issue, Comment, and Attachment data changes) and adds the Captain's Call status; labels and views can be created through the Linear MCP.
2. **Dock:** copy `alert-router` to `dock/`, add the Linear routes and the OAuth install, deploy under pm2, and confirm a test delegation is acknowledged within 10 seconds and filed as a row and a note.
3. **Replies:** add the receipts poll so answers to Linear requests appear in their threads.
4. **Mirror:** start `linear-mirror` in dry-run mode against the live backlogs, then apply.
5. **Captain calls and reviews:** add the `/mcp` tools and the select-answer pipe, and add the `data/captain.md` section.
6. **Contract test:** add the nightly CI job that runs the dock and mirror against upstream Firstmate `main`.

Each step is independent and reversible: stopping the pm2 app and the timer returns Firstmate to exactly its current behavior.

## Later: Linear as the backlog itself

If the overlay proves out, the step from projection to source of truth is a Linear adapter in tasks-axi, which Firstmate already supports for Beads through `.tasks.toml` `backend = ...`.
It would map tasks-axi verbs to Linear operations, store machine fields in one `firstmate://task/<id>` attachment, and import the markdown backlog with the existing `migrated from data/backlog.md id <legacy id>` marker.
That work belongs upstream, in tasks-axi and in any small Firstmate fixes the adapter exposes, so it should be proposed there rather than carried as a local patch.

## Open questions

- **Captain's Call as a status or a label:** a status gives a clean column and view; a label avoids changing the Engineering workflow.
- **Direct answer pipe:** feeding `fm-captain-hold.sh answers` from the dock is the documented channel path, while `decision-desk` deliberately left answer recording to Firstmate; this design pipes only `select` answers, whose keys are exact, and leaves free-text answers to Firstmate.
- **Cyrus:** whether to keep it installed alongside Firstmate or uninstall it so there is one agent to delegate to.
- **Concurrent backlog writes:** the dock's `fm-tasks-axi.sh add` must be confirmed safe while Firstmate is also writing the backlog, or the add moves into Firstmate's intake and the dock only notes.
