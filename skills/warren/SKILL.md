---
name: warren
description: Use when organizing repository work with Warren's workspace manager—projects, branch worktrees, durable sessions, and agent context—and when choosing safe lifecycle practices beyond CLI syntax.
---

# Warren workspace practice

Warren is the Host's source of truth for development context. Use it to keep a repository, its branch checkouts, and their running processes identifiable across clients and reconnects.

## Resource semantics

| Resource | Meaning | Good default |
| --- | --- | --- |
| `project` | A registered repository identity | Register the repository once; do not make one project per branch. |
| `workspace` / `worktree` | A Warren-managed checkout for one branch and intent | Put code changes and their sessions here. |
| `session` | A durable running terminal or agent process | Reuse or move it; do not recreate it just because the UI changed. |
| `terminal-group` / `group` | A standalone terminal context without a repository workspace | Use for host-level or exploratory work. |
| `endpoint` | The Host that owns the preceding resources | Treat identical names on different endpoints as unrelated. |

A Warren session ID, provider thread ID, and transcript path are different identities. A display name, cwd, branch name, or timestamp is context—not an ID.

## Best-practice workflow

1. **Choose the scope before the command.** Decide whether the request concerns a repository, a branch checkout, a running process, or a host-level shell. This prevents putting a repository task in a terminal group or registering every checkout as a project.
2. **Resolve, then mutate.** Select the endpoint explicitly when needed. Inspect JSON listings (`project`, `workspace`, and `session`) and use an exact ID. For the current shell, use `warren session current`; trust `WARREN_SESSION_ID`, never cwd or title. Stop on a missing or ambiguous match.
3. **Let Warren own worktrees.** Register an existing repository with `project add`; create a new branch checkout with `workspace create <PROJECT_ID> --branch <branch>`. Prefer Warren's managed default path. Use its project import flow for an existing external worktree instead of creating a duplicate record.
4. **Keep process and context together.** Create a session in the workspace that owns its work. When context changes, move the session instead of restarting it; the process, output history, and session identity survive the move. Preflight a current-session move before applying it:

   ```sh
   warren session move --current --workspace <WORKSPACE_ID> --dry-run
   ```

5. **Verify state transitions.** Re-list the affected resource after a mutation. Keep the operation ID returned by a session move; undo only while its recorded post-move context is still valid.

## Anti-patterns

- Using `git worktree add/remove`, `mkdir`, `mv`, or direct state-file edits to manage Warren workspaces. These bypass the registry and can orphan sessions or duplicate checkouts.
- Choosing a target from a partial name, cwd, transcript filename, or stale session row. Human names are for display; IDs and endpoint are the authority.
- Creating a second session because an existing one is hard to find, or sending concurrent input to one agent. Inspect first; wait for an active turn and serialize sends.
- Moving an explicit session without a preflight/confirmation or expected source context, or deleting a workspace without deciding whether its checkout must be kept. Workspace deletion has no automatic undo.
- Treating a terminal group as a workspace, casually forcing a custom `--path`, mixing endpoints, or exposing endpoint tokens in logs and prompts.

Use `--json` when another tool will consume output. Read command help for syntax, but apply the model and safeguards above even when the command itself is familiar.
