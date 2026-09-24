# CLAUDE.md

## 1. Think Before Coding

Don't assume. Don't hide confusion. Surface tradeoffs.

- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

Minimum code that solves the problem. Nothing speculative.

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If 200 lines could be 50, rewrite it.

## 3. Surgical Changes

Touch only what you must. Clean up only your own mess.

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- Mention unrelated dead code; don't delete it.
- Remove imports/variables your changes orphaned.

The test: every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

Define success criteria. Loop until verified.

- "Add validation" → write tests for invalid inputs, then make them pass.
- "Fix the bug" → write a test that reproduces it, then make it pass.
- "Refactor X" → ensure tests pass before and after.

For multi-step tasks, state a brief plan with verifiable checks per step.

---

## Workspace: terry-papers (paper posts + knowledge graph)

Paper posting, knowledge graph, references. Homepage code/infra changes happen in `terryum-ai`, not here.

| Do here | Don't do here |
|---|---|
| Paper posts (`/post`), essays, memos | Homepage code/components, Next.js routes |
| Knowledge graph (sync-papers, sync-references) | Build scripts, Cloudflare deploy config |
| R2 image uploads, post deletion (`/del`), share (`/share`) | Supabase schema, ACL/auth |
| Paper recommendation (`/paper-search`), Obsidian sync | |

### Layout (note the symlinks)
- `posts/`, `scripts/`, `node_modules`, `package.json`, `content.config.json` → symlinks to `terryum-ai`.
- `papers/<slug>.json` and `knowledge-index.json` are **owned by this repo** (AI insight cache produced by `scripts/export-knowledge.mjs`). Don't confuse with `posts/papers/<slug>/` (the MDX content under the symlink) — they share slugs.

### Knowledge base
`/post` and `/del` auto-run `export-knowledge.mjs`. Manual rebuild: `cd ~/Codes/personal/terryum-ai && node scripts/export-knowledge.mjs` (no args → outputs to this repo).

### Key commands
- `/post https://arxiv.org/abs/...` — paper post
- `/post --type=essays --from="<draft path>"` — essay/memo from Obsidian draft
- `/post synthesis URL1 URL2` — multi-source synthesis
- `/paper-search` — recommendation
- `/share #N` — social share
- `/del #N` — delete

### Git / private
- Always `git pull --rebase origin main` before pushing to `terryum-ai`.
- Separate content commits from code commits.
- `--visibility=group --group=snu` posts: Supabase only (no Git trace). `posts/global-index.json` is gitignored.

### Pointers
- Site: https://www.terryum.ai
- Homepage code: `terryum-ai`. Obsidian ops: `terry-obsidian`. Surveys: `terry-surveys`.
- Env keys: same `.env.local` shape as `terryum-ai` (Supabase + R2). See `terryum-ai/.env.example`.
- Knowledge structure: `ONTOLOGY.md`.
