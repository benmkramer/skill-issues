# skill-issues

Agent skills I use with Claude Code and Codex, packaged as a plugin marketplace
named `bmk`. One repo serves both tools.

## Install

Claude Code:

```
/plugin marketplace add benmkramer/skill-issues
/plugin install bmk@bmk
```

Codex:

```sh
codex plugin marketplace add benmkramer/skill-issues
codex plugin add bmk@bmk
```

Skills only, with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```sh
npx skills add benmkramer/skill-issues -g
```

To update, run `/plugin marketplace update bmk` in Claude Code or
`codex plugin marketplace upgrade bmk` in Codex.

## Skills

| Skill | What it does |
| --- | --- |
| [`new-work-thread`](plugins/bmk/skills/new-work-thread/SKILL.md) | Starts each new thread of work in its own Rex session with its own [worktrunk](https://worktrunk.dev) worktree, and can hand the task to a fresh agent there. |

In Claude Code, plugin skills are namespaced, for example `/bmk:new-work-thread`.

## Layout

```
.claude-plugin/marketplace.json    Claude Code marketplace
.agents/plugins/marketplace.json   Codex marketplace
plugins/bmk/
  .claude-plugin/plugin.json
  .codex-plugin/plugin.json
  skills/<name>/SKILL.md           each skill lives here once
skills -> plugins/bmk/skills       for the skills CLI
```

To add a skill, create `plugins/bmk/skills/<name>/SKILL.md` and add it to the
table above.

## Developing locally

Add the marketplace from your checkout instead of GitHub
(`/plugin marketplace add ~/code/skill-issues`,
`codex plugin marketplace add ~/code/skill-issues`). Both tools install a cached
copy, so after editing a skill:

- Codex: run `codex plugin add bmk@bmk` again. It copies the working tree,
  uncommitted changes included.
- Claude Code: commit, then run `claude plugin update bmk@bmk` and restart.
  It caches by commit, so uncommitted edits aren't picked up.
