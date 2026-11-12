---
name: warren
description: Use when working through Warren—creating or driving Agents, running shell Sessions, managing projects, worktrees, tasks, and pane layouts, or addressing another Warren Host. Covers which command to use and which habits are safe; flags live in `warren <command> --help`.
---

# Warren

This skill covers which Warren command fits a job and which habits keep work
safe. For flags and defaults, run `warren <command> --help` and
`warren <command> <action> --help`; the installed build's help is authoritative.

## Resources

| Resource | What it is | Use it for |
| --- | --- | --- |
| `project` | A registered repository | Register each repository once, not once per branch. |
| `workspace` (alias `worktree`) | A Warren-managed Git worktree for one branch | Repository changes and the processes that work on them. |
| `task` | A unit of work that groups Workspaces, possibly across repositories | Work that spans repositories or has an external ticket. |
| `agent` | A Session running a supported coding Agent, with a parsed transcript | Starting and talking to Codex, Claude, and the other providers in `agent create --help`. |
| `session` | A durable terminal process | Shells and other interactive programs. |
| `pane` | A saved split layout of Sessions in one Workspace | Arranging what is on screen. |
| `endpoint` | A Warren Host the CLI talks to | Choosing which Host a command acts on. |

Every Session, including every Agent, belongs to exactly one Workspace.

## Find the exact ID first

A Warren Session ID, a provider thread ID, and a transcript path are three
different things. Names, cwd, branch, and timestamps are not IDs.

1. When more than one endpoint is configured, pass `--endpoint NAME`. An ID
   from one endpoint means nothing on another.
2. Filter the list, then read the ID from `-q` or `--json` output:

   ```sh
   warren workspace list -q --project PROJECT_ID
   warren session list --search auth --status running
   warren agent list --provider codex --activity working
   ```

3. If nothing matches, or more than one thing does, stop and ask. Do not guess.
4. List again after every change.

List commands return at most 10 rows. `--json` changes the format but keeps
that limit, so 10 results may be only part of the list.

Avoid:

- Piping `list --all` through `grep` or `rg` when `--search` exists.
- Treating a 10-row result as the complete list.
- Pasting a full `--all` list into an Agent's context.
- Working out the current Agent from cwd, window title, or transcript file
  name. Use `--current` only when `WARREN_SESSION_ID` is set for the target.

## Agents

Use `agent` commands for every supported provider:

```sh
warren agent create WORKSPACE_ID --provider codex --prompt "Run the relevant tests"
warren agent create WORKSPACE_ID --provider claude --no-prompt
warren agent send AGENT_ID "Now fix the failing test"
warren agent wait AGENT_ID --timeout 30m
warren agent cancel AGENT_ID
```

`agent send` submits a turn, `agent wait` waits for the current or next turn to
finish, and `agent cancel` stops a running turn the way the provider expects.

Read the transcript with `agent read`, starting with the smallest view and
widening only when needed:

```sh
warren agent read AGENT_ID --text-only            # user and assistant text
warren agent read AGENT_ID --recent 50 --tools    # add tool calls
warren agent read AGENT_ID --tool-output          # add tool results
```

`--all` and `--full` can fill a context window; reach for them last.

Avoid:

- Driving, reading, or interrupting an Agent with `session send` or
  `session read`. Raw terminal input bypasses Warren's turn handling and
  corrupts the Agent's input box.
- Putting a print or non-interactive mode, a prompt, or shell operators in
  `--command`. It names only the executable or wrapper and its options.
- Passing `--agent-handler` without a specific reason. Each handler gives the
  Agent different capabilities.
- Treating a provider started in a shell Session as an Agent. Only Sessions
  created with `agent create` have a transcript and Agent commands. The `trae`
  preset is a plain shell Session.

## Shell Sessions

Use `session` for shells and programs that are not Agents:

```sh
warren session create WORKSPACE_ID --kind shell --command bash
warren session send SESSION_ID "make test"
warren session read SESSION_ID --timeout 8s --contains "PASS"
```

`session send` types the text and presses Enter; it does not wait for
anything. `session read` returns raw terminal output, which may include control
sequences.

## Projects, worktrees, and tasks

Let Warren create and track worktrees:

```sh
warren project add REPOSITORY_PATH --auto-import-worktrees
warren workspace create PROJECT_ID --branch BRANCH
warren task workspace create TASK_ID PROJECT_ID --branch BRANCH
```

`--auto-import-worktrees` adopts worktrees that already exist on disk.
`task workspace create` registers the project, creates the worktree, and adds
it to the task in one step.

Keep each Session in the Workspace whose work it is doing. Move it with
`session move` or `agent move` only when the work clearly belongs to one other
Workspace; a move keeps the process, its output, and its ID. Preview the move
and guard against a list that changed since you read it:

```sh
warren session move SESSION_ID --workspace WORKSPACE_ID --dry-run
warren session move SESSION_ID --workspace WORKSPACE_ID \
  --expected-workspace CURRENT_WORKSPACE_ID
warren session undo OPERATION_ID
```

Avoid:

- Running `git worktree add/remove`, `mkdir`, or `mv` on worktrees, or editing
  Warren's state files.
- Creating one task per branch. A task is the work; a Workspace is a checkout.
- Registering the same repository twice.
- Moving a Session whose work spans several Workspaces.

## Removing things

Only a Session move can be undone, with `session undo`. Removing a Workspace,
project, or task cannot be undone. Check what will go before removing:

- `session remove --dry-run` and `agent remove --dry-run` show what would end.
- `workspace remove --keep-worktree` unregisters the Workspace and leaves the
  Git checkout on disk.
- Removing a task, or detaching a Workspace from one, leaves the Workspaces and
  their Sessions untouched. Remove the Workspace itself to clean it up.
- `pane close` takes a pane off screen and `pane remove` deletes a saved
  layout. Neither ends a Session. To tidy the screen, edit the layout instead
  of ending Sessions.

## Credentials

Endpoint tokens, Relay enrollment keys, pairing links, `remote share` links,
and any QR code that encodes one each grant access to a Host. Never put them in
logs, prompts, transcripts, commits, or an Agent's context. Do not save a Relay
enrollment key anywhere; Warren uses it once and deliberately does not store it.
