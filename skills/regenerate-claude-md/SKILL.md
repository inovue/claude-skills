---
name: regenerate-claude-md
description: Use when user asks to regenerate, rebuild, refresh, create, or fix CLAUDE.md files based on current codebase state. Also use after major refactors, architecture changes, or when CLAUDE.md is outdated, missing, or incorrect. Trigger on any mention of CLAUDE.md creation, update, or maintenance — even indirect requests like "Claude doesn't know about our codebase" or "set up Claude for this repo".
---

# Regenerate CLAUDE.md

Rebuild CLAUDE.md files from scratch by exploring the current codebase, preserving user-specified sections, and validating quality.

## Process

### Step 1: Discover Existing Files

Glob `**/CLAUDE.md` and `**/.claude.local.md` to find all existing files.

`.claude.local.md` files are personal overrides that belong to the individual developer, not the project — note them but never regenerate them.

Also check `.claude/settings.json` and `.claude/settings.local.json`. If `customInstructions` exist, check for contradictions with existing CLAUDE.md content — but do NOT copy them into CLAUDE.md, because `customInstructions` are already loaded separately by Claude and duplicating them causes double-application. If `allowedTools` restrictions exist, note them in CLAUDE.md so Claude understands its constraints.

Record the discovered file structure — this determines output structure.

### Step 2: Select Target Files

Present the discovered CLAUDE.md files as a numbered list and ask the user which to regenerate:

> "Found the following CLAUDE.md files:
> 1. `./CLAUDE.md`
> 2. `apps/web/CLAUDE.md`
> 3. `packages/shared/CLAUDE.md`
>
> Which files do you want to regenerate? (all / numbers e.g. 1,3 / paths)"

- If only one CLAUDE.md exists (or none), skip this step and proceed with that file (or create `./CLAUDE.md`).
- If the user already specified target files in their original request, skip this step.
- "all" regenerates every discovered file.

Only selected files proceed through the remaining steps. Non-selected files are left untouched.

### Step 3: Ask About Preserved Sections

If the user explicitly said "regenerate everything" or "from scratch" with no caveats, skip this step. Otherwise, ask:

> "Are there any sections you want to preserve? (e.g., workflow instructions, personal preferences)"

If yes, read and extract those sections verbatim for reinsertion.

### Step 4: Explore Codebase

**Detect repository type** from root config files:

| Signal | Type |
|--------|------|
| `workspaces` in package.json, `pnpm-workspace.yaml`, `lerna.json`, `Cargo.toml` with `[workspace]`, `go.work`, multi-module `build.gradle` | **Monorepo** |
| `main`/`exports` in package.json, `[lib]` in Cargo.toml, `pyproject.toml` with library metadata, no `apps/` directory | **Library** |
| None of the above | **Single app** |

Launch parallel Explore subagents based on the detected type. Each subagent focuses on one area so no single agent has to understand the entire codebase. Also read `README.md` (if present) in this step — its content informs what CLAUDE.md should complement rather than duplicate.

**Subagent allocation by type:**

- **Monorepo** (4 subagents):
  1. Root structure, config files, CI/CD, and README.md
  2. Each app under `apps/` (or equivalent) — entry points, routes, dependencies
  3. Shared packages under `packages/` (or equivalent) — public APIs, exports
  4. Infrastructure, deployment, environment setup

- **Single app** (3 subagents):
  1. Root structure, config files, README.md, and entry points
  2. Routes/API layer and data/DB layer
  3. Frontend/UI components and test patterns

- **Library** (3 subagents):
  1. Public API surface, exports, and README.md
  2. Internal implementation and architecture
  3. Build/publish config, examples, and tests

Each subagent should return a structured markdown report covering:
- Directory structure and key files
- Entry points and configuration
- Patterns, conventions, and dependencies
- Commands (build, test, dev, deploy, lint)
- Non-obvious gotchas

### Step 5: Regenerate CLAUDE.md Files

Match the selected files from Step 2. If no CLAUDE.md exists, create a single `./CLAUDE.md`.

When multiple subagents report on the same topic, cross-reference their findings. If reports conflict (e.g., one finds Jest, another finds Vitest), check the actual config files and dependencies to determine which is current.

**Content guidelines by role:**

Any CLAUDE.md at the repository root is a **Project root** file. Any CLAUDE.md in a subdirectory (regardless of path — `apps/`, `packages/`, `services/`, `libs/`, `src/`, or any other) is a **Scoped context** file.

**Project root** — always include:
- Project overview (what it does, key tech stack)
- All commands — copy-paste ready (dev, build, test, lint, deploy)
- Code standards (formatting, imports, naming, type patterns)

**Project root** — include when applicable:
- Architecture overview (key modules, data flow)
- API routes / endpoints
- Environment variables and setup
- CI/CD pipeline notes
- Test patterns (mock strategy, fixtures, file naming)

**Scoped context** — include only what differs from root:
- Directory purpose and public API / key exports
- Local commands (if different from root)
- Patterns unique to this scope (defer shared patterns upward to root)

For monorepos, the root file covers project-wide overview + shared standards. Subdirectory files cover only what differs from or extends the root.

**Relationship with README.md:** CLAUDE.md complements README.md, not duplicates it. README.md targets human developers (setup guides, contribution guidelines, project history). CLAUDE.md targets Claude (commands in exact executable form, code patterns with examples, architectural decisions that affect code generation). If README.md already documents something well, don't copy it — instead, add the Claude-specific angle (e.g., README says "we use React" → CLAUDE.md says "components use forwardRef pattern, see `src/components/Button.tsx` for example").

**Size guideline:** Aim for 50-200 lines per file. Under 50 likely misses important context; over 200 likely includes noise that dilutes signal. Large monorepo root files may exceed 200 lines — that's fine if every line earns its place. Density matters more than length.

**Principles — and why they matter:**
- **Dense > verbose.** Tables and bullet points over paragraphs — Claude processes structured content more reliably than prose
- **Actionable > descriptive.** Commands should be copy-paste ready — vague instructions like "run the tests" force Claude to guess the exact command
- **Project-specific > generic.** Only include what's true for THIS codebase — generic advice (e.g., "use meaningful variable names") wastes context window and can conflict with actual project conventions
- **Preserve user sections** exactly where they were (typically at the top) — these represent deliberate choices the developer made
- **Omit unused frameworks/tools.** Mentioning frameworks not in the project (e.g., Next.js advice in a Vite project) actively misleads Claude into generating wrong code

**Example** — a well-written root CLAUDE.md for a TypeScript API project:

```markdown
# Project

Task management API built with Hono + Drizzle ORM on Cloudflare Workers.

## Commands

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start local dev server on port 8787 |
| `npm test` | Run vitest (unit + integration) |
| `npm run db:migrate` | Apply pending Drizzle migrations |
| `npm run deploy` | Deploy to Cloudflare Workers |

## Architecture

src/
├── routes/       # Hono route handlers (one file per resource)
├── services/     # Business logic (no HTTP concerns)
├── db/
│   ├── schema.ts # Drizzle schema (source of truth)
│   └── migrations/
└── middleware/    # Auth, logging, error handling

## Code Patterns

- Routes return `c.json()`, never throw — errors go through `middleware/error.ts`
- Services accept and return plain objects, not Hono context
- All DB queries use Drizzle query builder, no raw SQL
- Zod schemas for request validation, colocated with route files

## Test Patterns

- Unit tests: `*.test.ts` next to source files
- Integration tests: `tests/integration/*.test.ts` using test DB
- Mock pattern: `vi.mock('../services/foo')` at top of test file
```

### Step 6: Quality Check

If `claude-md-management:claude-md-improver` skill is available, invoke it to score each file, identify gaps, and apply targeted fixes. If unavailable, skip this step — the content guidelines in Step 5 are sufficient to produce quality output.

## Common Mistakes

| Mistake | Why it's a problem | Fix |
|---------|-------------------|-----|
| Including generic coding advice | Wastes context window and can contradict actual project patterns | Only document patterns actually found in the codebase |
| Listing unused frameworks (e.g., Next.js in a Vite project) | Claude generates code for the wrong framework | Check actual dependencies before writing |
| Overwriting user workflow instructions | Destroys deliberate developer choices that may not be recoverable | Always ask about preserved sections first |
| Writing long prose paragraphs | Claude extracts structured data more reliably than prose | Use tables, bullets, code blocks |
| Documenting every file | Noise dilutes the important signals | Focus on entry points, patterns, and non-obvious structure |
| Skipping the exploration step | You'll hallucinate outdated architecture | Parallel subagents ground content in actual code |
| Copying README.md content verbatim | Creates duplication that goes stale independently | Complement README with Claude-specific angles instead |
| Skipping Step 3 when user said "regenerate" | "Regenerate" may still imply preserving workflow sections | Confirm scope if ambiguous |
