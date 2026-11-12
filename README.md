# claude-skills

Claude Code skills, packaged as a plugin marketplace.

| Plugin      | What it does                                                                                                   |
| ----------- | -------------------------------------------------------------------------------------------------------------- |
| `pr-review` | Senior-dev PR review: a plain summary, only verified bugs (as JSON with criticality), and a merge verdict. `--comment #1 #3` drafts those findings as a pending GitHub review. |

## Install

This repo is private, so you need read access to it and a working git login for
GitHub (for example `gh auth login`).

In Claude Code:

```
/plugin marketplace add Muhsin-42/claude-skills
/plugin install pr-review@muhsin-skills
```

Updates: `/plugin marketplace update muhsin-skills`.

### Without the plugin system

Copy the skill folder into your personal skills directory:

```bash
mkdir -p ~/.claude/skills
cp -R plugins/pr-review/skills/pr-review ~/.claude/skills/
```

## Using `pr-review`

```
/pr-review 123                 # a PR number or URL
/pr-review my-branch           # a branch vs the default branch
/pr-review                     # the current branch + uncommitted changes
/pr-review --comment #1 #3     # put findings 1 and 3 on the PR as a pending review
```

It needs the GitHub CLI (`gh`), logged in. It reads the repo's own
`CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`, standards docs and ADRs when they
exist, so each project's rules are what it reviews against. `--comment` never
submits the review — you do that from the PR's "Files changed" tab.

## Layout

```
.claude-plugin/marketplace.json          # the marketplace: lists the plugins
plugins/<plugin>/.claude-plugin/plugin.json
plugins/<plugin>/skills/<skill>/SKILL.md
```

To add a skill, add a folder under `plugins/`, then list it in
`marketplace.json`.
