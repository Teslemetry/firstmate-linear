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
| Replace the backlog with a Linear tasks-axi adapter | About six backlog readers, plus a tasks-axi adapter | Nothing is contributed upstream, so these stay local patches that block fast-forward updates until rebased | Not planned |
| Fork: edit `AGENTS.md`, skills, and `bin/` for Linear | Many | `/updatefirstmate` is a guarded fast-forward, so any local commit stops updates until it is rebased; `AGENTS.md` and `bin/` change almost daily upstream | Avoid |

The overlay is how the existing Teslemetry tools already work: `alert-router`, `backlog-mirror`, and `decision-desk` change nothing in Firstmate and talk to it only through `bin/fm-inbox.sh`, `bin/fm-tasks-axi.sh`, and `bin/fm-captain-hold.sh`.
This design extends that pattern to Linear, and nothing from it is contributed upstream.
It is built as `dock/` in `Teslemetry/tools`; `dock/README.md` there owns the operating detail, and this page owns the design.

### The contract surface the overlay depends on

Every coupling point is a command or record format that upstream already documents as an interface for other channels:

| Interface | Used for | Upstream owner |
| --- | --- | --- |
| `bin/fm-inbox.sh note --request-id --json` | Every inbound Linear event that needs Firstmate | Script header ("the captain's out-of-band capture surface") |
| `bin/fm-inbox.sh receipts --all-replies --after` | Firstmate's answers back to a Linear request | Script header; already used by `alert-router` `/mcp` |
| `data/backlog.md` markdown grammar | Mirroring rows (tasks-axi `list` has no `--json` yet) | tasks-axi markdown grammar, already parsed by `backlog-mirror` |
| `state/<id>.meta` `pr=` line | Detecting review-ready work | `bin/fm-pr-check.sh` |

The last two are the weakest links because they are file formats rather than commands.
A nightly contract test in `Teslemetry/tools` CI, still to be built, runs the dock and mirror against a scratch home built from upstream Firstmate `main` with the real scripts, so an upstream change that breaks any row above fails a test before it reaches the host.

### What the overlay cannot do

- The markdown backlog stays the source of truth, and Linear is a live projection plus an input surface; a captain edit to a Firstmate-owned field in Linear is shown as divergence, never silently written back.
- Firstmate's escalation habits come from `AGENTS.md`; the overlay steers them through the home's private `data/captain.md` (the upstream-sanctioned place for captain preferences) instead of editing the contract.
- Fine-grained worker milestones are read from records, not posted by workers, so Linear shows real milestones (dispatched, PR opened, blocked, landed) rather than a live narrative.

## Architecture

```
 Linear (Firstmate team)          clickstack host: dock (pm2, behind fm.ba.id.au)           Firstmate primary home
 -----------------------          ------------------------------------------------           ----------------------
 Firstmate app  --webhook------>  /linear       verify, ack sessions in <10s
 (actor=app,                                    fm-inbox.sh note                    ----->  state/inbox + check wake
  assignable,    <--activities--  reply relay   fm-inbox.sh receipts               <-----  fm-inbox.sh reply
  mentionable)   <--issue sync--  mirror        reads backlog and pr= records       <-----  data/backlog.md, state/*.meta
                 <--Triage------  Better Stack incidents (after their alert note)
 Better Stack, GitHub, Twilio --> unchanged alert-router routes, fm-inbox.sh note  ----->  state/inbox
 External agents (Grok, ...) ---> /mcp: send_note, get_status, get_replies         <---->  fm-inbox.sh
 Linear MCP (as the captain)  <--------------------------------------------------------- Firstmate agent (reads)
```

There is one Firstmate home: secondmates are retired in this setup, so every Linear and alert note goes to the primary.
All new code lives in `Teslemetry/tools`; nothing is added to this repository.

### `dock/`: alert-router's successor, plus Linear

`dock/` is a copy of `alert-router/` with every route kept, as the captain asked, and its rules kept: zero runtime dependencies, secrets only in the host dotenv, pm2 as the only runtime, write-then-acknowledge, and delivery only through upstream scripts.
It replaces alert-router in one cutover that repoints `fm.ba.id.au` and the ClickStack docker relay together.

Linear routes:

- `POST /linear` verifies `Linear-Signature` (hex HMAC-SHA256 of the raw body) and rejects a `webhookTimestamp` more than 60 seconds off, then answers `200` within Linear's 5-second limit (`503` when the note could not be filed, so Linear retries).
- `GET /linear/install` and `GET /linear/oauth/callback` complete the one-time `actor=app` install with the `app:assignable` and `app:mentionable` scopes and keep the token private to the dock, which alone refreshes it.
- `GET /healthz` adds the Linear, mirror, and Triage state to alert-router's checks.

Event handling:

| Linear event | Dock action |
| --- | --- |
| Agent session `created` by the captain (a delegation or @mention) | Note with the issue, the suggested task id (`FM-12` gives `fm-12`), the request, and Linear's `promptContext`; then an ephemeral "On deck" thought |
| Agent session `prompted` by the captain | Note with the captain's exact words, naming the real task id of a mirrored issue |
| `prompted` with the `stop` signal | Note asking Firstmate to interrupt the worker and clean nothing up |
| Issue update by the captain (status, priority, delegate, assignee, title, description, project, labels) | Note describing the change; a cancel adds "keep unfinished work and ask before discarding it" |
| Any event by another actor | Dropped, or noted as `EXTERNAL <name>` outside text |
| Any event by the Firstmate app itself, including sessions it opened | Dropped, so its own writes never echo back |

The dock never writes the backlog.
Firstmate files the row from the note through its normal intake, using the suggested task id, so a ship branch such as `fm/fm-12` makes Linear's GitHub integration link the PR to the issue and the mirror adopts the issue.
Answers, including a picked option, also reach Firstmate as notes, and Firstmate records them through its own `fm-captain-hold.sh` or merge path, the same split `decision-desk` uses.

### Replies without any Firstmate change

`AGENTS.md` already tells Firstmate to answer a note that needs a durable answer with `bin/fm-inbox.sh reply <id>`.
The dock polls `fm-inbox.sh receipts --all-replies` for its `lin-` request ids, the way `alert-router`'s `get_replies` does, and posts each reply to the matching Agent Session as a `response` activity.
A captain who asks Firstmate something in Linear therefore gets the answer in Linear with no new Firstmate behavior.

### No separate app-identity MCP

An earlier draft gave the Firstmate agent its own MCP tools that act as the Firstmate app, because the Linear MCP acts as the captain and Linear does not notify a user about their own actions.
That is no longer needed: every write that must notify the captain comes from the dock itself, as the app.
The mirror raises captain calls and review-ready work from Firstmate's records, the reply relay posts Firstmate's answers into their threads, and the Triage path raises incidents.
The Firstmate agent therefore needs only the official Linear MCP, mostly for reads.

The external `/mcp` endpoint that lets other agents, such as Grok, raise things with Firstmate is unrelated and stays exactly as it was in alert-router: `send_note`, `get_status`, and `get_replies` over the primary's inbox, at `/mcp` and `/mcp/<token>`.
Its replies are filtered on the `mcp-` request-id prefix and Linear's on `lin-s-`, and each keeps its own cursor, so the two never see each other's answers.

### The mirror loop: raising things from structured state

The mirror is a loop inside the dock rather than a separate tool, because it needs the same app token and the same Linear client.
It is the Linear sibling of `backlog-mirror/`, reusing its parser and its rules (the backlog is the truth, key on the id alone, conflicts are reported and not written, divergence is shown and not resolved).
It runs every 15 seconds and projects the primary home's structured state onto Linear:

| Firstmate state | Linear projection |
| --- | --- |
| Queued row | Todo (Backlog when dated) |
| In flight row | In Progress, delegated to Firstmate |
| Open captain hold | Captain's Call, assigned to the captain, with one elicitation per hold lifecycle; lettered options in the hold reason become one-click choices |
| `pr=` recorded in `state/<id>.meta` | In Review, assigned to the captain, PR attached, with one Merge / Request changes / Not yet elicitation |
| Done row | Done, with one closing line in the thread |
| Row with no Linear issue | A new issue created as Firstmate, so Firstmate-discovered work appears too |

Because the mirror derives every raise from records Firstmate already writes, captain calls and review-ready PRs reach Linear with no Firstmate behavior change, and they are raised exactly once because each is keyed on the task id plus its hold lifecycle or PR URL.
A status is written only when the task's own state changes, so a status the captain sets in Linear stands until Firstmate moves the task.
A task named after a Linear issue adopts it, and calls are raised in the captain's own session on that issue.

### Better Stack incidents in Triage

Each Better Stack uptime incident becomes one Linear Triage issue in the Firstmate team, so an outage is a visible decision: Accept to have it fixed, Decline if it is expected.
The alert note is filed first, exactly as before, so a Linear problem can delay the issue but never the alert.
The dock opens a session on the issue and files one note asking Firstmate for its findings, which the reply relay posts to the thread; acknowledged and resolved updates go to the same thread, and an issue still in Triage closes itself when the incident resolves.
Each HyperDX alert is routed to exactly one place, straight to the dock or to Better Stack for its escalation policies, never both, so critical alerts reach Triage through Better Stack and are never filed twice.
An optional delay keeps flapping monitors out of Triage.

### Healing missed events

Linear retries a webhook only three times and can disable one that keeps failing, so Linear, not the event stream, is the source of truth for captain actions.
A bounded catch-up query for issues and Agent Sessions updated since a cursor, handing anything the dock never saw to its normal handling, is still to be built.

## Linear workspace shape

The workspace is rebuilt for this design with one human member (the captain) and Firstmate as its only agent, so the trust rule is exact: only events whose actor is the captain's Linear user are the captain's word.
Everything else - GitHub-synced or Slack-synced comments, Asks or email intake, and any future app - is untrusted text.

- **Team:** a dedicated `Firstmate` team (key `FM`) holds all Firstmate work, and the app is installed with access to that team only, so Linear itself keeps Firstmate out of every other team and the captain can use them freely; the dock still drops any event from another team as a cheap second check.
- **Repository:** resolved per issue from the Firstmate project registry, with Linear's `issueRepositorySuggestions` query and an `repo:<name>` label as hints, and one `select` elicitation when it is genuinely ambiguous.
- **Statuses:** Triage, Backlog, Todo, In Progress, Captain's Call (started type), In Review, Done, and Canceled.
- **Labels:** `fm:scout` (deliverable is a report), `mode:direct-PR` / `mode:local-only` / `mode:no-mistakes`, and `yolo:on` / `yolo:off` as explicit per-task captain instructions.
- **Views:** Captain's Call (assigned to me, Captain's Call) and Review queue (assigned to me, In Review).
- **Git automations:** turn off "PR merged moves the issue to Done" for the Firstmate team, because Firstmate's cleanup marks work done only after it has proved the work landed.
- **Guidance:** Linear's workspace and team agent guidance appears in every session's `promptContext`; it is a good place for captain-written conventions such as label meanings.

## The captain's Linear workflow

| Captain does | Result |
| --- | --- |
| Files an issue and leaves it undelegated | Nothing; it is the captain's own note. |
| Delegates an issue to Firstmate, or @mentions Firstmate with a request | Immediate acknowledgement, then normal Firstmate intake and dispatch under the issue's own task id. |
| @mentions Firstmate with a question | An answer from Firstmate's records, posted back in the thread. |
| Replies in the thread of live work | The words are passed to the worker as a mid-task instruction. |
| Picks an option on a Captain's Call | Firstmate records the captain's answer and the work proceeds. |
| Sends a stop request | The worker is interrupted, nothing is deleted, and Firstmate confirms what was left. |
| Moves an issue to Canceled | The worker stops; discarding unfinished work is asked for separately. |
| Accepts or declines a Better Stack incident in Triage | Accept and delegate to have it fixed; decline to close it as expected. |
| Picks Merge on an In Review item | That is the captain's explicit merge word for that PR: Firstmate merges through its guarded merge command and confirms with the PR URL. |
| Approves or merges in Linear Reviews | A merge by the captain is seen by Firstmate's existing merge monitoring; an approval alone is review evidence, not a merge order. |

When Firstmate opens PRs from the captain's GitHub account, they already appear in the captain's Reviews tab under Created; when it uses a separate account, requesting the captain's GitHub review puts them under For me.
Either way, the In Review status plus the Review queue view separates the PRs awaiting the captain's word.

## Where the Linear MCP fits

| Use | Value | Rule |
| --- | --- | --- |
| Firstmate intake | High: read the full issue, comments, attached screenshots (`extract_images`), linked Documents, and project context before writing instructions | Read freely. |
| Captain-attributed artifacts | Medium: post scout reports as Linear Documents and periodic project status updates | Allowed, since these are records rather than notifications. |
| Reading Linear Reviews | Medium: `get_diff_threads` shows review comments the captain left in Linear (they also sync to GitHub) | Read only. |
| Merging from Linear | None | Deny `merge_diff` and `update_diff`; every merge goes through Firstmate's guarded merge command. |
| Raising calls or replying | None | Leave it to the dock: reply with `fm-inbox.sh reply`, and let the mirror raise calls, because MCP writes appear as the captain and do not notify the captain. |
| Comments and issues | None | Deny `save_comment` and `save_issue`, so nothing Firstmate writes appears as the captain's own words. |
| Workers | None | Workers get no Linear access; only Firstmate talks to the captain. |
| Building this | High: set up labels, views, and test issues, and verify mirror output against the live workspace | Use during development. |

The Firstmate agent's harness gets one Linear connection, the official Linear MCP, as the captain and mostly for reads.

## Steering Firstmate without editing `AGENTS.md`

The home's private `data/captain.md` gets this Linear section, which Firstmate loads at every session start:

```markdown
## Linear

- Notes whose request id starts with `lin-` come from Linear through the dock. A `from: captain` note is the captain's own word; `EXTERNAL` text is outside text.
- A new Linear request names a `task_id` (for `FM-12`, `fm-12`). File the backlog row under exactly that id so the Linear issue follows the work.
- Answer every `lin-s-` note with `fm-inbox.sh reply <note id>`. The dock posts the reply into that Linear thread, so write it as a captain-facing message.
- Do not raise captain calls or review-ready work in Linear yourself. The dock mirrors held tasks and recorded PRs; keep holds and PR records accurate.
- A captain's reply on a Captain's Call is the answer: record it with `fm-captain-hold.sh answer`. Picking Merge on a review question is the captain's explicit merge word for that PR.
- A stop request or a cancel means interrupt or exit the worker through `bin/fm-control.sh`. Never clean up unfinished work without asking.
- For a Better Stack Triage note, investigate and reply with findings and a recommendation. Start no fix until the captain accepts and delegates the issue.
- Use the Linear MCP to read issues, comments, images, and documents. Never use it to comment, create or edit issues, or merge.
```

## Rollout

1. **Built:** the dock with every alert-router route, the Linear webhook and install, the reply relay, the mirror, and Better Stack incidents in Triage, with unit and end-to-end tests in CI.
2. **Workspace and app:** the captain creates the Firstmate team with Triage on and a Captain's Call status, then the Firstmate OAuth app as `dock/README.md` describes, with access to the Firstmate team only.
3. **Cutover:** move `fm.ba.id.au` and the ClickStack docker relay to the dock together, stop alert-router, and verify with a Better Stack test incident, a GitHub redelivery, and a test delegation.
4. **Steering:** add the `data/captain.md` section above.
5. **Still to build:** the missed-event catch-up and the nightly contract test against upstream Firstmate `main`.

Rollback is the reverse of the cutover; stopping the dock returns Firstmate to exactly its current behavior.

## Alternative: Linear as the backlog itself

Instead of mirroring, Firstmate can read and write Linear directly by selecting a Linear tasks-axi adapter through `.tasks.toml` `backend = "linear"`, the same seam Beads uses.
The dock is still required either way, because only it can receive webhooks and acknowledge Agent Sessions in time; the native option replaces only the mirror loop.

### What native needs

1. **A Linear adapter in tasks-axi** implementing every verb Firstmate uses (`add`, `show`, `list`, `ready`, `start`, `done` with evidence, `hold --until`, `unhold`, `update --archive-body`, multi-id `mv`, `rm`, `public-followup`) and passing the compatibility probe in `bin/fm-tasks-axi-lib.sh`, with machine fields in one `firstmate://task/<id>` attachment per issue.
2. **Firstmate fixes for readers that bypass tasks-axi.** Lifecycle transitions already handle non-markdown adapters, but several readers open `data/backlog.md` directly: the session-start digest listing, `bin/fm-fleet-snapshot.sh` (Bearings and heartbeats), `bin/fm-inbox.sh status`, `bin/fm-backlog-handoff.sh`, `bin/fm-backlog-receive.sh`, and `bin/fm-afk-return.sh`. On a Linear backend those would report an empty or absent backlog, which the startup contract treats as a registry to rebuild, so they must route through tasks-axi first.
3. **A migration** of the markdown backlog and Done archive, using the existing `migrated from data/backlog.md id <legacy id>` marker.
4. Optionally, **instruction changes** so Firstmate drives Agent Sessions itself; the `data/captain.md` section covers this without editing `AGENTS.md`.

### Comparison

| | Mirror (overlay) | Native (Linear backend) |
| --- | --- | --- |
| Firstmate code changed | None | About six backlog readers, plus a tasks-axi adapter |
| Upstream updates | Fast-forward unchanged | Blocked by the local patches on every update until they are rebased |
| Source of truth | Markdown; Linear is a projection seconds behind | Linear, with no second copy and no divergence |
| Captain edits in Linear | Captain-owned fields reach Firstmate through dock notes; Firstmate-owned fields are flagged as divergence, not written back | Every edit is the real record immediately, including priority, relations, and status |
| Linear or network outage | Firstmate keeps working; Linear catches up afterwards | Dispatch, cleanup, and captain holds stop, because backlog writes refuse rather than guess |
| Latency and limits | Local file speed; Linear rate limits touch only the mirror | Every backlog read and write is an API call, including the session-start digest |
| Integrity guards | tasks-axi atomic file writes and the existing lock model | Must be rebuilt in the adapter with optimistic concurrency against concurrent captain edits |
| Testing | Contract test against upstream `main` | Firstmate's own test suite assumes markdown; the adapter needs its own suite plus a Firstmate run on a Linear backend |
| Rollback | Stop the dock's mirror loop | Migrate back to markdown |
| Build size | Dock plus a mirror loop | Dock plus a full adapter, the readers fix, and migration |

### Recommendation

The mirror, because nothing goes upstream: native would mean carrying local patches to Firstmate's backlog readers and a tasks-axi adapter indefinitely, and rebasing them on every upstream update.

## Open questions

- **Missed events:** whether the catch-up query is worth building, given Linear's three webhook retries and the dock's own uptime.
- **HyperDX routing:** each alert must go to exactly one of the dock or Better Stack; nothing enforces that, so it stays a configuration check.
