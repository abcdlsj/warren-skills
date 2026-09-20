---
name: warren
description: Use when organizing repository work with Warren's workspace manager—tasks, projects, branch worktrees, pane arrangements, durable sessions, and Agent context—and when choosing safe lifecycle practices beyond CLI syntax.
---

# Warren workspace practice

Warren is the source of truth for development context. Resolve the resource and
its exact ID before mutating it. A Warren Session ID, provider thread ID, and
transcript path are different identities; names, cwd, branch, and timestamps
are not IDs.

Run `warren <resource> --help` and `warren <resource> <action> --help` for the
authoritative flag set of the installed build. This document covers what each
resource means and which habits are safe, not the full syntax.

## Resource semantics

| Resource | Meaning | Default use |
| --- | --- | --- |
| `task` | Host-owned unit of work that may span repositories | Create when work touches more than one repository or has an external work-item identity. |
| `project` | Registered repository identity | Register once; do not create one per branch. |
| `workspace` / `worktree` | Warren-managed checkout for a branch and intent | Put repository changes and their processes here. |
| `agent` | Warren-integrated Agent process with a normalized transcript | Primary interface for supported providers. |
| `session` | Durable Warren-managed PTY process | Use for shells and other interactive programs. |
| `pane` | Host-owned split arrangement of Sessions inside one scope | Read it to know how a Session is displayed; edit only to arrange. |
| `terminal-group` / `group` | Terminal scope without a repository workspace | Use for host-level or exploratory work. |
| `endpoint` | Host that owns the resources above | Treat the same name on different endpoints as unrelated. |

A Session's scope is either a Workspace or a Terminal Group, never both and
never absent. An Agent is a Session with a provider binding, so it has a scope
in the same way.

## Tasks across repositories

A Task groups Workspaces from different repositories without flattening
repository boundaries, and can retain a provider-neutral external identity:

```sh
warren task create --name "Cross-repository delivery" --source tapd --external-id 12345
warren task workspace create TASK_ID PROJECT_ID --branch release/feature
warren task workspace attach TASK_ID WORKSPACE_ID
warren task workspace list TASK_ID
```

`task workspace create` registers the Project, creates the Git worktree, and
attaches it in one request. `task detach` and `task remove` drop the grouping
only; the Workspace, its checkout, and its Sessions survive.

Anti-patterns:

- Creating one Task per branch. A Task is the work, not the checkout.
- Registering the same repository twice to fake cross-repository work.
- Removing a Task to clean up a Workspace. Remove the Workspace instead.

## Agent workflow

Use Agent commands for any supported provider. `agent` IDs are Warren Session
IDs; `agentThreadId` and the transcript path are separate fields in roster
output.

Create with an explicit provider and exactly one initial-prompt mode:

```sh
warren agent create WORKSPACE_ID \
  --provider codex --command codex-alias --prompt "Run the relevant tests"
warren agent create WORKSPACE_ID --provider claude --no-prompt
warren agent create --group GROUP_ID --provider opencode --prompt "Survey this host"
```

- `--provider` selects transcript and prompt semantics. The installed build's
  accepted set is printed by `warren agent create --help`; it currently covers
  `codex`, `claude`, `opencode`, `pi`, `qoder`, and `antigravity`. Confirm from
  help rather than assuming a provider is present.
- `--prompt` is appended using that provider's own startup convention
  (positional for Codex, Claude, Pi, and Qoder; `--prompt` for OpenCode; `-i`
  for Antigravity). Warren applies the convention; pass plain text.
- `--no-prompt` explicitly creates an idle Agent. One of the two is required.
- `--wait` is only valid with `--prompt`; use it with `--timeout` when the first
  turn must finish before the command returns.
- `--command` is the executable, alias, or wrapper entry point plus its options.
  It may not contain shell operators/substitutions, a positional prompt, a
  prompt option, or a provider's non-interactive/print mode. Omit it to use the
  provider's default executable.
- `--agent-handler` selects the transport inside a provider family (`tui`,
  `cli`, `acp`). Omit it unless a specific transport is required: the default is
  the TUI handler, and a wrong handler changes which capabilities and transcript
  semantics the Agent has.
- `WORKSPACE_ID` is optional. Pass it, or `--group`, to state the scope; do not
  rely on an implicit default when more than one scope is plausible.

Use `agent send` for later turns. It waits for the Agent binding/transcript,
writes the provider composer, and submits the separate Enter event. With no
TEXT it reads the prompt from stdin.

Use `agent read` for the normalized transcript, never the PTY. Its default
projection is the newest 20 user/assistant/error activities with text capped at
2,000 characters. Widen deliberately, cheapest first:

```sh
warren agent read AGENT_ID --text-only            # user and assistant text only
warren agent read AGENT_ID --recent 50 --tools    # add tool-call summaries
warren agent read AGENT_ID --tool-output          # add bounded tool results
warren agent read AGENT_ID --chars 8000           # raise the per-field cap
```

`--all` returns every matching activity and `--full` returns the complete
canonical event history; `--full` cannot be combined with projection flags.
Both can flood a context window. Use `--include`/`--filter` to select activity
types instead of reading everything.

`agent wait` waits for the current or next turn. `agent attach` is the live raw
TTY. Use `agent current` or `--current` only when `WARREN_SESSION_ID` identifies
the target. Prefer `agent move`, `agent rename`, `agent pin`, and
`agent remove` for lifecycle operations; `agent remove --dry-run` reports what
would end.

Anti-patterns:

- Using `session send`/`session read` to drive or read a provider TUI. The
  Agent commands own turn semantics; raw PTY writes desynchronize the composer.
- Inferring the current Agent from cwd, window title, or transcript filename.
- Reaching for `--full` before `--text-only` or `--recent N`.
- Passing a print/non-interactive provider flag through `--command`.
- Assuming a provider is Agent-capable because its binary runs in a shell
  Session. A Session is Agent-capable only after Warren records a provider
  binding. A `trae` preset is a shell Session and has no Agent transcript,
  activity, or Agent CLI semantics.

## Generic session workflow

Use `session` for a generic PTY, not for creating or conversing with an Agent
provider:

```sh
warren session create WORKSPACE_ID --kind shell --command bash
warren session send SESSION_ID "Run a shell command"
warren session read SESSION_ID --timeout 8s --contains "done"
warren session attach SESSION_ID
```

`session read` returns raw terminal output and may include TUI control data; it
is not a text-only reader. `session send` writes raw terminal input and has no
turn or wait semantics; it appends a carriage return so the line is submitted,
and `--raw` sends the bytes exactly as given. `session attach` is the
interactive TTY operation.

## Pane arrangements

A pane arrangement is durable Host state, not a client layout. One Workspace or
Terminal Group may hold several arrangements; one is rendered at a time, so
switching never disturbs the others.

```sh
warren session panes SESSION_ID            # how this Session is arranged, and who displays it
warren pane list --workspace WORKSPACE_ID
warren pane split --pane PANE_ID --session SESSION_ID --axis vertical
```

In `session panes` output, GROUP/PANE come from the Host's arrangement, and
SCREEN names a client displaying the Session right now; empty means none is.

Anti-patterns:

- Treating `pane close` as ending a Session. It takes the Pane off screen; the
  Session keeps running and stays reachable as an ordinary Tab.
- Treating `pane remove` as deleting Sessions. It deletes the arrangement.
- Ending a Session to tidy a layout. Edit the arrangement instead.

## Lists and context

Roster-heavy `list` commands (`task`, `project`, `workspace`, `terminal-group`,
`session`, `agent`, `pane`) return at most 10 rows with long fields truncated.
Narrow the query instead of dumping the roster:

```sh
warren workspace list -q --project PROJECT_ID          # IDs only
warren session list --search auth --status running
warren agent list --provider codex --activity working
warren workspace list --unmerged --has-sessions
```

- `-q`/`--quiet` prints IDs only and is the cheapest form for scripting and for
  handing IDs to another Agent.
- `--search TEXT` plus a scope filter beats `--all` piped through a text tool.
- `--limit N` raises or lowers the bound when a count is actually needed.
- `--json` changes the representation only; it does **not** disable the default
  limit. A 10-element JSON array may be a truncated roster, not the whole one.
- For `session list` and `agent list`, `--ended` selects ended rows and cannot
  be combined with `--all`.

Resolve exact IDs from a fresh `--json` or `-q` listing after filtering, and
stop on a missing or ambiguous match rather than guessing.

Anti-patterns:

- `warren session list --all | rg pattern` when `--search` exists.
- Reading a 10-row default as a complete roster, in table or JSON form.
- Copying a full `--all` roster into an Agent context.

## Safe lifecycle

1. Select the endpoint explicitly when more than one is configured.
2. Inspect the relevant JSON or quiet roster and resolve an exact ID.
3. Let Warren own projects and worktrees:

   ```sh
   warren project add REPOSITORY_PATH
   warren project add REPOSITORY_PATH --auto-import-worktrees
   warren workspace create PROJECT_ID --branch BRANCH
   ```

   Do not use `git worktree add/remove`, `mkdir`, `mv`, or direct state-file
   edits. When a repository already has Git worktrees on disk, adopt them with
   `project add --auto-import-worktrees` rather than recreating them. Omit
   `--path` unless a specific worktree location is required.
4. Keep the process in the workspace that owns its work. A Workspace and the
   shell or Agent handling its task are normally paired. Warren can move an
   existing Session or Agent to another Workspace or Group without recreating
   it; use that when the current responsibility clearly belongs exclusively to
   another scope. If the responsibility is ambiguous or spans multiple
   workspaces, do not move mechanically. The process, output history, and ID
   are preserved after a move.
5. Re-list after every mutation. For Session moves, preflight first and retain
   the returned operation ID:

   ```sh
   warren session move --current --workspace WORKSPACE_ID --dry-run
   warren session move SESSION_ID --workspace WORKSPACE_ID \
     --expected-workspace CURRENT_WORKSPACE_ID
   warren session undo OPERATION_ID
   ```

   Explicit-ID moves require `--confirm` or an expected-source context;
   `--expected-workspace` and `--expected-agent-session` make the move fail
   rather than act on a roster that changed under you. Undo only while the
   recorded post-move context is unchanged.
6. Removal is asymmetric, so check the blast radius before removing:
   `session undo` is the only undo. Workspace, Project, and Task removal have
   none. `workspace remove --keep-worktree` keeps the Git checkout on disk when
   only the Warren registration should go.

## Endpoints, remote hosts, and credentials

One Warren CLI can address several Hosts. Resources never cross endpoints, and
an ID from one endpoint is meaningless on another.

```sh
warren endpoint add vps --url http://127.0.0.1:8789 --token TOKEN --use
warren endpoint add box --ssh box --ssh-remote 127.0.0.1:8789
warren ssh TARGET                     # bootstrap the remote daemon over a tunnel
warren --endpoint vps workspace list -q
```

`warren ssh list` reports which SSH aliases the embedded client can resolve;
unsupported `ProxyJump`/`ProxyCommand` aliases need a direct host or an external
`ssh -L` tunnel plus `endpoint add`.

`warren display set NAME [NAME ...]` is local client configuration for which
endpoints appear in the sidebar. It is independent of the current endpoint, so
switching endpoints does not change sidebar membership, and the explicit set
cannot be empty.

Relay puts a Host behind a shared control plane:

```sh
warren relay connect SETTINGS_URL     # enroll this Host with an admin-issued key
warren relay share --qr               # issue one reusable pairing invite
```

Everything Relay-related is a bearer credential: endpoint tokens, enrollment
keys, pairing invites, and the QR PNG that encodes one. Never put any of them in
logs, prompts, transcripts, commit contents, or an Agent's context. An
enrollment key is a short-lived bootstrap credential and is deliberately not
written to the endpoint configuration; do not persist it yourself.
