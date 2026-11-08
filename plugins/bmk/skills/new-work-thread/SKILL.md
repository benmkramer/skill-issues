---
name: new-work-thread
description: Use when the user asks to start a new thread of work, new task, new feature, new branch, spin off work, or "start fresh on X". Defaults to a new Rex session with a new worktrunk (wt) worktree, unless the user says to stay in the current thread or you are already in a worktree.
---

# Start a new thread of work

The user uses **Rex** as the terminal (the `rex` CLI manages sessions, tabs, and panes)
and **worktrunk** (`wt`) to manage git worktrees. Each thread of work gets its
own **Rex session** and its own **worktree**: one session per task, matching
Rex's model, where a session is labeled after the task it holds. Don't use a new
tab in the current session for a new thread.

Requirements: the `rex` CLI, `jq`, and worktrunk with shell integration
installed (`wt config shell install`), so `wt switch` can change the new shell's
directory.

For full CLI detail, load the `rex` skill (or run `rex __internal skill`) and the
`worktrunk` skill. This skill only decides *what* to do and gives the recipe.

## Decide

1. **The user says to stay** ("stay here", "in this thread", "same session", "don't
   make a worktree"): do the work in place. Stop here.
2. **Already in a worktree**: stay put and work here. Check with:
   ```sh
   d="$launch_dir"  # the directory the agent session started in (see Recipe)
   [ "$(git -C "$d" rev-parse --path-format=absolute --git-dir --git-common-dir 2>/dev/null | uniq | wc -l)" -eq 2 ] && echo linked-worktree
   ```
   (`wt list` also shows which worktree is current.) Only make a new session
   and worktree from here if the user explicitly asks for one.
3. **Not in a git repo**: ask which repo the work belongs to. Don't guess.
4. **Rex isn't reachable** (`rex whoami --json` fails): create the worktree with
   `wt switch --create BRANCH` in place and tell the user you couldn't open a session.
5. **Otherwise**: open a new session and worktree (recipe below).

## Recipe: new Rex session + new worktree

**Find the repo from the launch directory**, the directory the agent session
started in (Claude Code reports it as the primary working directory; Codex as
the `cwd` in its environment context). Don't use wherever the shell has since
`cd`'d to. Resolve it to the repo root with
`git -C "$launch_dir" rev-parse --show-toplevel`. If the launch directory isn't
in a git repo, ask (Decide, step 3).

Before creating anything, say in one line which repo and branch you're using,
for example: "Creating `fix-login-redirect` in `~/code/hatchmates`." Then go
ahead without waiting for a reply.

Pick a short kebab-case branch name from the task (for example
`fix-login-redirect`). It doubles as the session label. Check that it isn't
already taken: `git -C "$repo" branch --list BRANCH`, `wt -C "$repo" list`, and
`rex ls`. If the branch exists, ask whether to reuse it (`wt switch BRANCH`, no
`--create`). If a session with that label exists, ask whether to use it instead
of making another.

```sh
launch_dir="/abs/path/the/agent/started/in"
repo="$(git -C "$launch_dir" rev-parse --show-toplevel)"
branch="fix-login-redirect"

# 1. New session labeled after the branch, starting in the repo. Its shell is
#    interactive, so the `wt` shell function (which cds into the worktree) is loaded.
sess_json="$(rex new "$branch" -c "$repo" --json)"
session="$(printf '%s' "$sess_json" | jq -r '.session_id')"
block="$(printf '%s' "$sess_json" | jq -r '.initial_windows[0].block_ids[0]')"

# 2. Wait for the shell prompt (text sent before it appears is lost), then
#    create the worktree in that session. The pattern matches prompts ending
#    in ❯ $ % # or >; adjust it if the user's prompt ends differently.
for i in $(seq 20); do
  rex capture -s "$session" -b "$block" --trim | grep -qE '[❯$%#>] *$' && break
  sleep 0.25
done
rex send -s "$session" -b "$block" "wt switch --create $branch" &&
  rex send-key -s "$session" -b "$block" enter

# 3. Show the new session in the Rex app.
rex do session.select session_id="$session"
```

Notes:

- Base defaults to the default branch. Add `--base @` to branch off the current
  branch, or `--base OTHER` if the user names one.
- worktrunk hooks (post-create, post-start, etc.) run automatically. Capture the
  session's pane afterward to confirm the switch succeeded and hooks didn't fail.
- `rex new` doesn't show the session in the app by itself. Step 3 does, but it is
  a client action that needs Remote Control (off by default). If several apps
  are connected, pass `-C CLIENT` (see `rex client ls`). If it's refused, don't
  change the setting. Tell the user the session label so they can pick it from the
  sidebar.
- Skip step 3 if the user wants to stay put.
- Never `rex send` into your own pane (`$REX_BLOCK`).

### Hand the work to an agent in the new session

If the user described the task, start the same agent CLI you are (`claude` for Claude
Code, `codex` for Codex) in the new worktree with a handoff prompt, using
`wt switch`'s `-x`:

```sh
rex send -s "$session" -b "$block" "wt switch --create $branch -x claude -- 'Fix the login redirect loop. Context: ...'" &&
  rex send-key -s "$session" -b "$block" enter
```

For a long or quote-heavy prompt, write it to a file first and pass
`"\$(cat /abs/path/prompt.md)"` instead of the inline string. The handoff
prompt should be self-contained: the goal, relevant files, decisions already
made, and what "done" looks like. The new agent won't see this conversation.

If the user only asked for the session and worktree, skip `-x` and leave the shell
sitting in the worktree.

## Report back

Tell the user the session label, branch, and worktree path (from `wt list` or a
capture of the new pane), whether the app switched to it, and whether an agent
was started there. Then stop: the new thread's work happens in the new session,
not in this one.
