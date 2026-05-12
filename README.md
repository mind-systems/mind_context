# Mind — monorepo context layer

This repository is the coordination layer for the Mind project. It holds cross-project plans, roadmaps, and AI context. The application code lives in separate sub-repositories cloned inside this directory.

## Onboarding a new developer

Copy the prompt below and send it to Claude Code in this directory. It will clone all sub-repositories into the right places automatically.

---

```
Clone the Mind monorepo and all sub-projects into the correct directory structure.

Run these commands in order:
1. git clone https://github.com/mind-systems/mind_context.git mind
2. cd mind
3. git clone https://github.com/mind-systems/mind-awake-api.git mind_api
4. git clone https://github.com/mind-systems/mind_mobile.git mind_mobile
5. git clone https://github.com/mind-systems/mind_landing.git mind_landing
6. git clone https://github.com/mind-systems/mind_mcp.git mind_mcp
7. git clone https://github.com/mind-systems/neiry_kit.git neiry_kit

Directory structure must be preserved exactly — build scripts and proto schemas reference files via relative paths from the root.

After cloning, for each repository (root, mind_api, mind_mobile, mind_landing, mind_mcp, neiry_kit) find the most recently committed branch and switch to it:
- Run `git branch -r --sort=-committerdate` to list remote branches by recency
- Check out the top result (skip HEAD and main/master if a feature branch is more recent)
- If the most recent branch is already main/master, stay on it

After switching branches everywhere, read CLAUDE.md in the root and in each sub-project to understand the project structure and development workflow.
```

---

## Sub-repositories

| Directory | GitHub | Stack | Purpose |
|-----------|--------|-------|---------|
| `mind_api/` | [mind-awake-api](https://github.com/mind-systems/mind-awake-api) | NestJS + TypeORM + PostgreSQL | Backend REST API |
| `mind_mobile/` | [mind_mobile](https://github.com/mind-systems/mind_mobile) | Flutter + Riverpod + Drift | iOS/Android mobile app |
| `mind_landing/` | [mind_landing](https://github.com/mind-systems/mind_landing) | Plain HTML/CSS/JS | Static landing page |
| `mind_mcp/` | [mind_mcp](https://github.com/mind-systems/mind_mcp) | TypeScript + MCP stdio | MCP server for Claude Code |
| `neiry_kit/` | [neiry_kit](https://github.com/mind-systems/neiry_kit) | Flutter plugin | Neiry neurofeedback SDK wrapper |

## Repository layout

```
mind_context/          ← you are here (coordination layer)
├── .ai-factory/       — cross-project plans, roadmaps, AI context
├── .claude/           — shared Claude Code skills
├── mind_api/          — backend (separate git repo)
├── mind_mobile/       — mobile app (separate git repo)
├── mind_landing/      — landing page (separate git repo)
├── mind_mcp/          — MCP server (separate git repo)
└── neiry_kit/         — Flutter plugin (separate git repo)
```

Each sub-directory is an independent git repository. Run `git` commands from inside the sub-directory, not from the root.
