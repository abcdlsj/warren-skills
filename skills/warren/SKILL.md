---
name: warren
description: Use when organizing repository work with Warren's workspace manager—projects, branch worktrees, durable sessions, and Agent context—and when choosing safe lifecycle practices beyond CLI syntax.
---

# Warren workspace practice

Warren is the source of truth for development context. Resolve the resource and
its exact ID before mutating it. A Warren Session ID, provider thread ID, and
transcript path are different identities; names, cwd, branch, and timestamps
are not IDs.

## Resource semantics

| Resource | Meaning | Default use |
| --- | --- | --- |
| `project` | Registered repository identity | Register once; do not create one per branch. |
| `workspace` / `worktree` | Warren-managed checkout for a branch and intent | Put repository changes and their processes here. |
| `agent` | Warren-integrated Codex or Claude process | Primary interface for supported Agents. |
| `session` | Durable Warren-managed PTY process | Use for shells, Trae, and other interactive programs. |
| `terminal-group` / `group` | Standalone terminal context without a repository workspace | Use for host-level or exploratory work. |
| `endpoint` | Host that owns the resources above | Treat the same name on different endpoints as unrelated. |

Only Codex and Claude are supported Agent providers. A shell/custom session is
Agent-capable only after Warren records a Codex/Claude binding; a Trae preset is
still a shell session and has no Agent transcript, activity, or Agent CLI
semantics.

## Agent workflow

Use Agent commands for Codex/Claude. `agent` IDs are Warren Session IDs;
`agentThreadId` and the transcript path are separate fields in roster output.

Create with an explicit provider and exactly one initial-prompt mode:

```sh
warren agent create WORKSPACE_ID \
  --provider codex --command codex-alias --prompt "Run the relevant tests"
warren agent create WORKSPACE_ID \
  --provider claude --no-prompt
```

- `--provider` must be `codex` or `claude`; it selects transcript semantics.
- `--command` is the executable, alias, or wrapper entry point plus its options.
  It may not contain shell operators/substitutions, a positional prompt,
  `--prompt`, or a provider's non-interactive/print mode. Pass initial text only
  with `--prompt`.
- `--prompt` is appended using the provider's initial-prompt convention.
  `--no-prompt` explicitly creates an idle Agent. One of the two is required.
- `--wait` is only valid with `--prompt`; use it when the first turn must
  finish before the command returns.

Use `agent send` for later turns. It waits for the Agent binding/transcript,
writes the provider composer, and submits the separate Enter event. Do not use
`session send` to drive a Codex/Claude TUI.

Use `agent read` to read the normalized transcript, never the PTY. Its default
projection is the newest useful activities (20) with text fields capped at
2,000 characters. Use `--recent N`, `--all`, `--include`, `--filter`, `--full`,
or `--text-only` when needed. Use `agent attach` only for the live raw TTY.
`agent wait` waits for the current or next turn to finish.

Use `agent current` or `--current` only when `WARREN_SESSION_ID` identifies the
target. Never infer the current Agent from cwd, title, or transcript filename.
Prefer `agent move`, `agent rename`, `agent pin`, and `agent remove` for Agent
lifecycle operations.

## Generic session workflow

Use `session` for a generic PTY, not for creating or conversing with Codex or
Claude:

```sh
warren session create WORKSPACE_ID --kind shell --command bash
warren session send SESSION_ID "Run a shell command"
warren session read SESSION_ID --timeout 8s
warren session attach SESSION_ID
```

`session read` returns raw terminal output and may include TUI control data;
it is not a text-only Agent reader. `session send` writes raw terminal input
and has no Agent turn/wait semantics. `session attach` is the interactive TTY
operation. The `trae` preset belongs here until it has an explicit Warren
provider integration.

## Lists and context

Roster-heavy `list` commands (`project`, `workspace`, `terminal-group`,
`session`, and `agent`) return at most 10 rows by default. Use `--limit N` for a
different bounded result. Use `--all` for the complete list, preferably with a
search so a long roster is not copied into an Agent context:

```sh
warren session list --all | rg 'pattern'
warren agent list --all | rg 'pattern'
```

For `session list`, `--ended` selects ended sessions and cannot be combined
with `--all`. `--json` changes the representation only; it does not disable
the default limit. Use exact IDs from a fresh `--json` listing after filtering.

## Safe lifecycle

1. Select the endpoint explicitly when more than one is configured.
2. Inspect the relevant JSON roster and resolve an exact ID. Stop on a missing
   or ambiguous match.
3. Let Warren own projects and worktrees:

   ```sh
   warren project add REPOSITORY_PATH
   warren workspace create PROJECT_ID --branch BRANCH
   ```

   Do not use `git worktree add/remove`, `mkdir`, `mv`, or direct state-file
   edits. Omit `--path` unless a specific worktree location is required.
4. Keep the process in the workspace that owns its work. Move an existing
   resource instead of recreating it; the process, output history, and ID are
   preserved.
5. Re-list after every mutation. For session moves, preflight first and retain
   the returned operation ID:

   ```sh
   warren session move --current --workspace WORKSPACE_ID --dry-run
   warren session undo OPERATION_ID
   ```

   Explicit-ID moves require `--confirm` (or an expected source context). Undo
   only while the recorded post-move context is unchanged. Workspace deletion
   has no automatic undo.

## Endpoint and credentials

Use `warren --endpoint NAME` or `--server URL --token TOKEN` when the target is
not the default endpoint. Never put endpoint tokens in logs, prompts, or
transcripts.
