# agentkit: a reusable environment for trustworthy coding agents

> `agentkit` is a placeholder name. Rename it freely.
> Sections marked **(proposed)** are designs, not yet proven. Sections marked **(proven)** come from the agent-causality trial.

---

## 1. What this is

A toolkit you drop into any Python or TypeScript repo so that coding agents (OpenCode, Claude Code, Codex) can work unattended and you can trust the result.

The idea comes from Lauren Tan's "trust ladder" talk. You stop babysitting agents by making their mistakes impossible or detectable. In order of strength:

1. **Codebase as memory:** one blessed pattern per task; delete bad patterns.
2. **Static analysis:** lint, types, import boundaries, CI. Hard enforcement.
3. **Guidance:** `AGENTS.md`, commands, skills. Soft enforcement.
4. **Human review:** last resort, and the source of new rules for steps 1-3.

Verification (the agent proves its own work) runs through all of it.

**Core habit:** every time you correct an agent, ask "how do I make this mistake impossible?" before just fixing it.

---

## 2. Principles

- **Deterministic over AI.** Lint, tests and hash checks first. Model judgment only where nothing else works. This matters most on a free model.
- **Suppress, don't mass-refactor.** Add a rule, baseline existing violations, list them in `CLEANUP.md`, fix them later in small PRs.
- **Structural guards beat prompt rules.** A free model will break a prompt rule. Make the violation fail a check.
- **One source of truth.** Shared logic lives in the toolkit, project facts live in the project.
- **Keep it small.** No options for languages or tools you don't use.

---

## 3. Architecture (proposed)

Some files must live inside each project (agents and CI read the repo they work in). Everything else is shared.

| Lives in the toolkit repo (versioned, updated once) | Lives in each project (generated, then owned by it) |
|---|---|
| Verify runner (reads `verify.toml`) | `verify.toml` |
| Suppression-budget checker | Layer rules and boundary baseline |
| Import-boundary checker | `CLEANUP.md` (the baseline ledger) |
| Generated-copy sync check (hash manifest) | `AGENTS.md`, `.opencode/commands/` |
| `spawn.py`, worktree cleanup | CI workflow, pre-commit config |
| Bootstrap skill and file templates | Project-specific checks (browser smoke test, `FEATURE_MAP.md`) |

Projects consume the runners as a **pinned package** (`uvx agentkit==X.Y.Z verify`) so a fix reaches every project through a version bump, and a toolkit change can't silently break one.

---

## 4. Toolkit repo layout (proposed)

```
agentkit/
  pyproject.toml
  src/agentkit/
    cli.py                  # agentkit verify | init | spawn | worktrees | budget | boundaries | sync-check
    verify.py               # reads verify.toml, runs steps, writes logs and evidence
    budget.py               # suppression budget
    boundaries.py           # import-boundary checker (AST for Python)
    syncheck.py             # hash-manifest check for generated copies
    spawn.py                # worktree-per-task agent runner
    worktrees.py            # list and clean worktrees
  templates/
    AGENTS.md.tmpl
    commands/{debug,feature,review}.md.tmpl
    ci.yml.tmpl
    pre-commit-config.yaml.tmpl
    verify.toml.tmpl
    CLEANUP.md.tmpl
  skills/
    bootstrap/SKILL.md      # "set up this repo" (your Phase 0-5 prompts)
  tests/
  GUIDE.md
```

Start with Python projects only. Add TypeScript support (ESLint, `tsc`, dependency-cruiser) when your second project needs it.

---

## 5. Config formats (proposed)

### `verify.toml`

```toml
[project]
name = "my-project"
artifacts_dir = "tools/verify/artifacts"

[[step]]
name = "pytest"
cmd = "uv run pytest -q"
modes = ["quick", "full"]

[[step]]
name = "frontend build"
cmd = "npm run build"
cwd = "frontend"
modes = ["full"]            # slow steps are full-only

[[step]]
name = "browser smoke"
cmd = "npx playwright test"
cwd = "frontend"
modes = ["full"]
evidence = ["frontend/test-results/**"]
```

- `quick` (about 15-20s) runs after every edit. `full` runs before an agent finishes and in CI.
- Every step writes a log, plus evidence files where listed. The summary prints PASS or FAIL per step with timing.

### Import boundaries

```toml
[boundaries]
layers = ["sdk", "core", "storage", "adapters", "explorer", "cli"]   # each may import only from layers to its left
forbid = [{ from = "explorer", to = "benchmark" }]
baseline = "tools/verify/import_boundaries_baseline.json"
```

### Suppressions

```toml
[suppressions]
patterns = ["# noqa", "# type: ignore", "eslint-disable", "@ts-ignore"]
ledger = "CLEANUP.md"
```

The checker **computes** the count from source and compares it to the rows in the ledger table. Nobody hand-edits a total. On overflow it fails with: `add a row to CLEANUP.md or fix the code`.

### Generated copies

```toml
[[generated]]
name = "explorer static"
inputs = ["frontend/src", "frontend/package-lock.json", "frontend/next.config.*"]
output = "explorer/static"
manifest = "explorer/static/.inputs.sha256"
rebuild = "python scripts/sync-explorer-static.py"
```

The check hashes `inputs`, compares to the manifest, and fails if they differ. See section 9.

---

## 6. Prerequisites

- Python 3.12+, `uv`, Node 20+ (for TypeScript projects), `git`
- OpenCode CLI (`opencode`) and VS Code
- pstack skills (see section 11)
- A GitHub repo with Actions enabled

---

## 7. Build order

1. **Finish the fixes in agent-causality first.** The path-independent static sync, scope guard and computed budget are generic problems. Solving them there builds the toolkit's logic.
2. **Create the toolkit repo** and move the generic scripts in, replacing hardcoded paths with `verify.toml` config.
3. **Make agent-causality the first consumer.** Delete its local copies. Do not keep two sources of truth.
4. **Run `init` on a second, different project** (e.g. Vigil) and fix whatever breaks.
5. **Publish** (or pin by git tag) only after step 4.

---

## 8. The verify runner

Behavior (proven in agent-causality, to be generalized):

- Reads `verify.toml`, runs steps in order, continues after a failure, and reports all failures at the end.
- Writes logs and evidence under `artifacts_dir/<timestamp>-<pid>/`, gitignored.
- Exit code is non-zero if any step fails.
- Stdlib-only and cross-platform (Windows is a first-class target; no shell scripts).
- Uses temp ports for any server step, never fixed ports, so parallel worktrees don't collide.

---

## 9. Checkers

### 9.1 Suppression budget (proven)
Counts suppression directives and boundary baseline entries, then compares to `CLEANUP.md`. Stops agents from silencing rules without a record.

### 9.2 Import boundaries (proven for Python)
AST-based. Fails on any import that crosses a layer the wrong way or hits a `forbid` pair, unless it is in the baseline file. Also ban direct imports of optional drivers (e.g. `psycopg`) outside one guard module.

### 9.3 Generated-copy sync (lesson learned)
**Do not compare built output across checkouts.** A Next.js build embeds the checkout path in module IDs, so any worktree produces a different build and fails the comparison. A `git status` exception is also unsafe: in CI the checkout is always clean, so it would pass even when the copy is stale.

Use a **source hash manifest** instead:

1. Hash the inputs (source, lockfile, configs) into `manifest`.
2. `check` recomputes and compares. Changed inputs with no regeneration fail.
3. Regenerate only on the main checkout, never from a worktree.

First try making the build deterministic (timebox it; for Next.js, set `generateBuildId`). Fall back to the hash if module IDs still depend on the path.

---

## 10. Parallel agents (`spawn`) (proven, with fixes)

```
agentkit spawn --task "Move scoring helpers into core" \
               --branch agent/cleanup-item-8 \
               --allow-paths "core/**,benchmark/**,explorer/**,tests/**,CLEANUP.md"
```

What it does:

1. Creates a git worktree for the branch under a gitignored folder.
2. Runs setup there (`uv sync`, `npm ci`).
3. Runs `opencode run` with the task.
4. Runs full verify. The run passes only if the agent finished **and** verify passed.
5. Writes `summary.json` and per-task logs.

Rules:

- Max 3 concurrent agents (use 1-2 on a free model); queue the rest.
- **Scope guard:** `--allow-paths` is required. If the diff touches anything else, the run is FAIL. This catches scope creep (an agent changed a public API it was told to leave alone).
- **Worktree guard:** a pre-commit check fails if generated output (e.g. `explorer/static/`) is staged from a linked worktree.
- No auto-push and no auto-merge without an explicit flag. Agents push branches, you review.
- Use `agentkit worktrees` to list and prune merged or abandoned worktrees, with a confirmation prompt.

---

## 11. The bootstrap skill (`init`) (proposed)

One skill that sets up a new repo. It works in phases and stops for your approval after each.

| Phase | What happens |
|---|---|
| 0. Recon | Detect stack, package manager, test runner, linter, CI, how to run the app. List anti-patterns. No changes. |
| 1. Skills | Install pstack (`npx skills add backnotprop/pstack`, or the port you already use; don't install both). Run `/setup-pstack` and map every role to your available model. |
| 2. Verification | Generate `verify.toml`, a browser or API smoke test, and `FEATURE_MAP.md`. Run once and show the output. |
| 3. Enforcement | CI runs full verify; pre-commit runs quick verify; add lint rules, import boundaries and the budget. Baseline existing violations into `CLEANUP.md`. |
| 4. Guidance | Generate `AGENTS.md` (under 60 lines) and the `/debug`, `/feature`, `/review` commands. |
| 5. Parallelism | Add `spawn` config and the advisory CI review job. |

Notes:

- pstack's `create-verification-skill` already generates a verify skill with a feature map. Reuse it where it fits.
- `poteto-mode` is not sticky in OpenCode. Reference it from `AGENTS.md` or type it at the start of a task.
- Ask before installing global tools, downloading browsers or deleting code.

---

## 12. CI template

```yaml
name: verify
on: [push, pull_request]
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uvx agentkit==0.1.0 verify --mode full
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: verify-artifacts
          path: tools/verify/artifacts

  ai-review:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    continue-on-error: true          # advisory only, never a required check
    steps:
      - uses: actions/checkout@v4
      - run: uvx agentkit==0.1.0 review-pr
        env:
          MODEL_API_KEY: ${{ secrets.MODEL_API_KEY }}
```

Rules for the AI review job:

- **Same-repo PRs only.** Never give secrets to fork PRs.
- **Advisory only.** A flaky free model must never block merges.
- **Treat the diff as untrusted data.** A diff can contain prompt-injection text.
- Skip cleanly if the secret is missing.
- Keep database or integration tests (e.g. Postgres) in their own job.

---

## 13. Working with a free model

- Use 1-2 parallel agents, not 3 or more. The first parallel trial likely used up the quota.
- One numbered task per session, so a cutoff doesn't leave half-done work.
- Lower effort for mechanical edits; max effort for design decisions.
- Agents should use `verify --quick`, not re-read files.
- Keep `AGENTS.md` short. Long rule lists get ignored by smaller models.
- Review risky changes yourself (public API changes, anything touching security).
- Paste diffs into a chat assistant for review when your agent quota is gone.

---

## 14. The cleanup workflow

`CLEANUP.md` lists every baselined violation, grouped by anti-pattern. The rule: **never add a suppression without adding a row, and removing a row is the cleanup signal.**

Work in small PRs, cheapest and safest first:

1. Layering violations (move shared helpers into the right layer).
2. Optional-import guards (one module that returns the driver or skips).
3. Mechanical simplification fixes (behavior-preserving).
4. Narrowing broad `except` blocks, one file at a time (riskiest).
5. Design changes (global runtime, protocols, single env loader), one per PR.
6. Replace source-substring tests with behavior tests.

Every cleanup PR: full verify before and after, no behavior change, and the budget number goes down.

---

## 15. Lessons from the first trial (proven)

| Problem | Fix |
|---|---|
| Static build differs per checkout path, so every worktree fails sync | Source hash manifest, regenerate only on main |
| `git status` exception would pass in CI even when stale | Don't use it; hash the inputs |
| Agent did more than asked and changed a public API | `--allow-paths` scope guard; review public API changes by hand |
| Agent regenerated generated files in a worktree and got a false PASS | Worktree guard in pre-commit |
| Parallel branches conflict on a hand-edited total in `CLEANUP.md` | Compute the total from the rows |
| Fixed ports collide across worktrees | Temp ports everywhere |
| Quick mode wasn't quick (it still built the frontend) | Slow steps are full-only |
| A passing browser test can still pass on weak assertions | Open the screenshot yourself once; assert real content |

---

## 16. Versioning and releases

- Semantic versions; breaking changes to `verify.toml` bump the minor version while pre-1.0.
- Projects pin the exact version in CI and pre-commit.
- Keep a `CHANGELOG.md`. Note every break explicitly.
- Upgrade one project first, run full verify, then the rest.

---

## 17. Checklist

**Toolkit**
- [ ] Verify runner reads `verify.toml`
- [ ] Budget, boundaries and sync checkers are config-driven
- [ ] `spawn` has the scope guard and concurrency cap
- [ ] Worktree guard in pre-commit
- [ ] Bootstrap skill and templates
- [ ] Tests for each checker (matching hash passes, changed source fails, and so on)

**Per project**
- [ ] Phase 0 recon reviewed
- [ ] `verify.toml` runs green in quick and full
- [ ] CI runs full verify; pre-commit runs quick
- [ ] Rules added; violations baselined in `CLEANUP.md`
- [ ] `AGENTS.md` and commands in place
- [ ] One trial of two small parallel tasks, results reviewed by you

**You, always**
- [ ] Review public API changes
- [ ] Turn every repeated correction into a rule
