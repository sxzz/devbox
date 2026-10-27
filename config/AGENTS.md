# Global rules

Shared by Claude Code, Codex and pi. Keep it agent-neutral and short: only what an agent wouldn't do right by default.

Source: `config/AGENTS.md` in the devbox repo (`/workspace/devbox`). On every container start it is copied to each agent's global instructions file, overwriting them; edit the repo copy, never the copies in home. Changes take effect after `scripts/rebuild` on the host.

## Environment
You are running inside the devbox container, launched by Paseo.
- Only `/workspace` and the home directory persist; nothing else from the host is mounted.
- There is no Docker in this container. Do not try to install or start one.
- Use `mise` to add tool versions a project needs (`mise use node@20` in the project).
- A shared Chromium is available. Drive it with the `agent-browser` CLI (already connected via `AGENT_BROWSER_CDP`; run `agent-browser skills get core` for usage); the `playwright` MCP server reaches the same browser as a fallback. Dev servers on `localhost` are reachable from it.
- Other agents and I use that browser too: pass your own `--session <name>`, and close the tabs you opened when done. It may already be signed in to websites. Do not sign out, change account settings or passwords, or make purchases.
- GitHub access is a bot account via `gh`.
- Google Workspace (Gmail, Calendar, Drive, ...) via the `gog` CLI, signed in to my Google account; see `gog --help`. Sending mail, accepting invites, sharing or deleting files act as me: ask first.
- Gmail is draft-only: `gog` blocks sending, replying, forwarding and trashing. To send mail, create a draft (`gog gmail drafts create`, or `drafts reply`/`drafts forward`), tell me its subject and recipients, and I send it from Gmail myself. Do not try to work around the block.

## Talking to me
- Reply in the language I write in.
- Plain, concrete words: what changed, how many, the effect. Use the system's own terms, not invented labels.
- Final reports: list the changes and why, with key file paths, how it was verified, and anything I need to do.
- If my plan or assumption looks wrong, say so with reasons before doing it, then follow my decision.

## How much to decide alone
Depends on the project. Infer which kind from the repo (published package, `"private": true`, README); ask if unclear.
- **Vibe app** (personal or experimental): decide most things yourself, including small refactors around the change.
- **Library / npm package / serious open source**: involve me more. Ask before adding dependencies, changing public APIs, exports or config formats, or anything that could break users. Keep diffs minimal; mention unrelated issues instead of fixing them.
- Irreversible actions (deleting data, touching production, publishing) always need my go-ahead.

## Git
- Never commit unless asked. Push and opening a PR each need a separate request; push to a new branch.
- Commit messages follow the repo's existing style (check `git log`).
- Write commit messages, PRs, and anything posted on GitHub (comments, reviews) in English.
- No AI attribution anywhere: no `Co-Authored-By` AI trailers, no "Generated with ..." lines in commits or PRs.

## Repositories
Local clones live directly under `/workspace` (`/workspace/<repo-name>`).

Remotes:
- Every stored `remote.*.url` / `pushurl` is `https://github.com/<owner>/<repo>.git` (HTTPS, with `.git`), including forks and upstreams.
- `origin` is the canonical repo. In someone else's project, my fork is the remote `sxzz`. A contributor's remote added to check out their PR is named after their GitHub login.
- Fix remotes that point at a renamed or transferred repo (check with `gh api repos/<owner>/<repo>`).

Cleanup (when I ask to tidy repos):
- Remove a contributor remote once all its PRs are merged or closed.
- Delete a local branch only when its tip equals the `headRefOid` of a merged or closed PR, not by name match. A branch checked out in another worktree: ask, then remove the worktree too.

Batch changes across many repos:
- Skip repos with local changes and list them in the report; `git pull --ff-only` on `main` first and report repos that can't fast-forward.
- One commit per repo, same message for all; include the regenerated lockfile in that commit.
- Package manager is declared as `devEngines.packageManager` (`name`, `version`, `onFail: "download"`), not the `packageManager` field.

## Code
- Comments in English unless the project already writes comments in another language. Comments explain why, not what.
- Docs follow the project's conventions or my instructions.
- Keep files under ~1500 lines; split into modules when one gets close.
- Before calling a task done or committing, run the project's lint/format/typecheck and relevant tests. Report failures as they are.
- Renames and cross-cutting changes: search all file types for every spelling variant first, then search again afterwards to confirm nothing is left.
- Bulk mechanical edits across many files: write a small script instead of many single edits, then verify the result.

## Remembering things
When I say "remember this": general preferences go in this file (edit the devbox repo copy and show me the change); facts about one project go in that project's memory or instructions, not here.
