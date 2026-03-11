# Vibefile Spec

> **Status: Draft.** This spec is being actively discussed. Open an issue or join the [Discord](https://discord.gg/3TsMMvK8tV) to participate.

---

## Overview

A Vibefile is a task runner configuration file. It describes **what** tasks do and **when** they run — not **how** they are implemented. The how is figured out at runtime by an LLM that reads the Vibefile alongside relevant context from your repo.

The format is inspired by Makefiles. The dependency model is identical. The only thing that changes is the recipe: instead of shell commands, you write a plain-English description of intent.

---

## File name

The file must be named exactly `Vibefile` with no extension, placed at the root of the repository.

```
my-project/
└── Vibefile
```

---

## Syntax

A Vibefile is a plain text file. It contains:

- **Variable assignments** — key/value pairs available throughout the file
- **Target definitions** — named tasks with optional dependencies and a recipe
- **Directives** — modifiers that change how a target is executed

### Variables

```makefile
KEY = value
```

Variables are substituted anywhere in the file using `$(KEY)`.

```makefile
ENV = production

deploy:
    "deploy to $(ENV) on fly.io"
```

### Targets

```
target-name [: dep1 dep2 ...]:
    recipe
    [directives]
```

- **target-name** — lowercase, hyphens allowed, must be unique within the file
- **dependencies** — space-separated list of other target names that must complete first
- **recipe** — a double-quoted string describing what this task should do in plain English
- **directives** — optional modifiers prefixed with `@`

### Dependencies

Dependencies are declared after the target name, separated by a colon:

```makefile
deploy: test build
    "deploy to production"
```

`deploy` will not run until `test` and `build` have both completed successfully. This is identical to Makefile behaviour.

### Recipe

The recipe is a double-quoted plain English string. It is sent to the LLM along with repo context to generate the implementation at runtime.

```makefile
seed:
    "populate the database with realistic fake data for 10 users"
```

Multi-line recipes use continuation indentation:

```makefile
release: test build
    "bump the version, update the changelog,
     tag the commit, and push to origin"
```

---

## Directives

Directives are optional modifiers on a target. They appear indented below the recipe, one per line, prefixed with `@`.

### `@require`

Asserts a condition that must be true before the target runs. If the condition is not met, the task fails with a clear error rather than attempting execution.

```makefile
ship: test build
    "deploy to production on fly.io"
    @require clean git status
    @require passing tests
```

`@require` values are evaluated by the LLM as part of the pre-flight check. They are intentionally plain English — the runtime determines how to verify them given the repo context.

### `@mcp`

Declares one or more MCP servers the task should use. When `@mcp` is present, the target runs as an **agent** rather than generating shell commands. The LLM receives tool access to the declared servers and executes the task through tool calls rather than via a generated script.

```makefile
deploy: build
    "deploy to production and verify the app came up healthy"
    @mcp fly-mcp

review:
    "open a PR, write a summary of changes, request review from the team"
    @mcp github-mcp, linear-mcp
```

Multiple servers are comma-separated. MCP server identifiers map to configured server definitions (see [MCP Configuration](#mcp-configuration)).

Without `@mcp`, the target runs in **codegen mode**: the LLM generates a shell script which the CLI executes directly.

### `@skill`

Delegates the task to a published Skill rather than a plain-text recipe. A Skill is a directory containing a `SKILL.md` file — structured LLM instructions maintained by the skill author.

```makefile
test:
    @skill python-test

deploy: test build
    @skill fly-deploy
    @mcp fly-mcp
```

`@skill` and a plain recipe string can be combined. The skill's instructions are used as the base; the additional recipe text is appended as supplementary context:

```makefile
release: test
    @skill python-publish
    "also update the changelog and tag the commit"
```

Skills are resolved in this order:

1. `skills/` directory in the repo root (project-specific)
2. `~/.vibe/skills/` (user-global)
3. Community registry at `vibefile.dev/skills`

---

## Model configuration

The model used for LLM calls is declared in the Vibefile. It is not a secret and belongs in the file.

```makefile
model = claude-sonnet-4-6

deploy:
    "deploy to production"
```

Individual targets can override the top-level model:

```makefile
model = claude-haiku-4-5

# this target needs more capability
release: test build
    model = claude-opus-4-6
    "bump version, update changelog, tag, push, open release PR"
```

### API key resolution

The API key is **never** declared in the Vibefile. It is resolved at runtime in this order:

1. `--api-key` CLI flag
2. `VIBE_API_KEY` environment variable
3. Provider-specific env var inferred from the declared model:
   - `ANTHROPIC_API_KEY` for Claude models
   - `OPENAI_API_KEY` for OpenAI models
4. `~/.vibeconfig` global config file

```yaml
# ~/.vibeconfig — never committed to version control
default_model: claude-sonnet-4-6
anthropic_key: sk-ant-...
openai_key: sk-...
```

---

## Configuration

Vibe configuration is split across three layers with different trust and visibility properties:

| Layer | Location | Committed to git? | Purpose |
|-------|----------|:-----------------:|---------|
| Project config | `.vibe/config.yaml` | ✅ Yes | MCP server definitions, skill sources, project defaults |
| User config | `~/.vibeconfig` | ❌ No | API keys, server credentials, personal defaults |
| Inline | `Vibefile` | ✅ Yes | Model selection, per-target overrides |

### Project configuration — `.vibe/config.yaml`

A `.vibe/` directory at the repo root holds project-level configuration. It is checked into version control. It never contains secrets.

```
my-project/
├── Vibefile
└── .vibe/
    ├── config.yaml     ← MCP server definitions, skill sources
    └── skills/         ← local skill definitions (project-specific)
```

```yaml
# .vibe/config.yaml

# MCP server definitions — name → how to reach it
servers:
  fly-mcp:
    url: https://mcp.fly.io/sse
  github-mcp:
    url: https://mcp.github.com/sse
  postgres-mcp:
    command: npx @modelcontextprotocol/server-postgres
    args: ["--connection-string", "$(DATABASE_URL)"]

# Where to look for skills (in addition to built-in resolution order)
skill_sources:
  - ./skills                      # project-local (always checked first)
  - registry: vibefile.dev/skills # community registry
```

Server credentials (`url` is not a secret, but auth tokens are) belong in `~/.vibeconfig`:

```yaml
# ~/.vibeconfig — never committed
server_tokens:
  fly-mcp: fo1_...
  github-mcp: ghp_...
```

### Registry integration

Rather than manually declaring every MCP server URL in `.vibe/config.yaml`, a registry can be used as the resolution backend. When the CLI encounters an `@mcp fly-mcp` directive it doesn't recognise locally, it can query a registry to discover the server's URL, authentication requirements, and available tools.

The current proposed approach is **registry-as-fallback**: local `.vibe/config.yaml` definitions take priority; unknown server names are looked up in the configured registry.

```yaml
# .vibe/config.yaml

# use a registry as the fallback resolver
registry:
  url: https://aregistry.ai          # or a self-hosted instance
  # no API key needed for public registry lookups

# only declare servers that differ from registry defaults
servers:
  postgres-mcp:
    command: npx @modelcontextprotocol/server-postgres
    args: ["--connection-string", "$(DATABASE_URL)"]
```

This means `@mcp fly-mcp` in a Vibefile just works without any local definition — the registry knows what `fly-mcp` is, where it lives, and what credentials shape it expects. The user only has to supply the actual credential in `~/.vibeconfig`.

#### The AI-native angle

Registries like [agentregistry](https://aregistry.ai) expose an MCP server themselves — meaning the CLI agent can query the registry using the same tool-call mechanism it uses for everything else. Instead of a static lookup, the agent can ask the registry questions:

- *"What MCP servers are available for interacting with Fly.io?"*
- *"Does the fly-mcp server support a `deploy` tool?"*
- *"Find a skill for publishing a Python package to PyPI."*

This would make server and skill resolution fully dynamic rather than declarative — the agent figures out what tools it needs based on the task description, queries the registry, and assembles the right server set. No explicit `@mcp` directive required.

This is a significant design direction that has real tradeoffs:

| | Declarative `@mcp` | Dynamic registry discovery |
|--|--|--|
| **Predictability** | ✅ Explicit, auditable | ⚠️ Agent-decided at runtime |
| **Ease of use** | Requires knowing server names | Just describe the task |
| **Security** | Reviewed in code | Requires registry trust |
| **Reproducibility** | Same servers every run | Registry contents may change |

Both models can coexist: explicit `@mcp` declarations remain supported; dynamic discovery is opt-in at the project or target level.

#### Registry lock-in

Vibefile does not intend to require any specific registry. The registry interface should be treated as a protocol, not a vendor. Options under consideration:

- Define a minimal registry query API that any implementation can satisfy
- Support multiple registries simultaneously (with priority order)
- Allow fully registry-free operation (all servers declared locally)
- Support self-hosted registries for enterprise use

This is deliberately unresolved. The goal is that `vibefile.dev/skills` and `aregistry.ai` and a private corporate registry can all be used interchangeably, or not used at all.

---

## Execution modes

Each target runs in one of three modes, determined by its directives:

| Mode | Trigger | Behaviour |
|------|---------|-----------|
| **Codegen** | plain recipe, no `@mcp` | LLM generates a shell script; CLI executes it |
| **Agent** | `@mcp` present | LLM executes via tool calls to declared MCP servers |
| **Skill** | `@skill` present | LLM instructions come from `SKILL.md`; mode is then codegen or agent depending on other directives |

---

## Execution sandbox

AI-generated shell commands and agentic tool calls never execute directly on the user's host machine. Every target runs inside an isolated sandbox — a container that is created for the task, given controlled access to the repo, and destroyed when the task completes.

This is a hard guarantee, not a configuration option. The user's machine is never the execution environment.

### What the sandbox looks like

The sandbox is an OCI container (Docker-compatible) spun up by the CLI before execution:

```
host machine
├── vibe CLI              ← orchestrates everything, never executes generated code
│   ├── collect context
│   ├── call LLM API
│   └── stream output ◄── sandbox stdout/stderr
│
└── sandbox container     ← all generated shell / agent tool calls run here
    ├── /repo             ← bind-mount of the project directory (read-write)
    ├── /tmp              ← ephemeral scratch space
    └── network           ← controlled (see below)
```

The repo directory is bind-mounted read-write into the sandbox, so changes the task makes (compiled artifacts, generated files, etc.) persist on the host. Everything else in the container is ephemeral and discarded after the task finishes.

### Network access

Network access inside the sandbox is **off by default** and must be explicitly declared:

```makefile
build:
    "compile and bundle for production"
    # no @network — sandbox has no network access

deploy: build
    "deploy to fly.io and verify health"
    @network outbound
    @mcp fly-mcp

seed: build
    "populate the database with realistic fake data"
    @network local       # can reach localhost / docker network, not the internet
```

Network modes:

| Mode | Behaviour |
|------|-----------|
| *(none)* | No network access. Filesystem only. |
| `@network local` | Can reach `localhost` and services on the local Docker network. No external internet. |
| `@network outbound` | Full outbound internet access. Required for deploy-type tasks. |

Declaring `@network outbound` is intentionally explicit — it signals to the reader that this task reaches the internet, and it's visible in code review.

### Sandbox requirements

The CLI requires a container runtime to be available. Docker is the default; other OCI-compatible runtimes (Podman, etc.) will be supported.

If no container runtime is found, `vibe run` fails with a clear error rather than falling back to executing on the host.

### What about MCP agent tasks?

For targets that use `@mcp`, the execution model is slightly different:

- The **LLM agent loop** runs outside the sandbox (inside the CLI process), since it needs to call MCP server APIs
- Any **shell commands** the agent generates as intermediate steps run inside the sandbox
- MCP tool calls that affect external services (e.g. `fly deploy`, `gh pr create`) are authenticated via MCP server credentials, not host credentials

This keeps the sandbox boundary clear: the sandbox is where filesystem and process execution happens. External service calls go through MCP servers, which have their own authentication.

### Escape hatch

For advanced users who understand the risks, sandboxing can be disabled per-target:

```makefile
legacy-script:
    "run the old deploy script that needs host docker socket access"
    @no-sandbox
```

Using `@no-sandbox` emits a visible warning at runtime and requires explicit confirmation unless `--yes` is passed. It should be rare.

---

## Context collection

When a target runs in codegen or agent mode, the CLI collects repo context before making the LLM call. The collector is task-aware: it sends only what's relevant for the declared intent.

**Always included:**
- Top-level file tree (depth 2)
- Contents of `Vibefile`

**Conditionally included** (detected from task name and intent):
- `package.json` / `pyproject.toml` / `go.mod` — framework and script detection
- `fly.toml`, `Dockerfile`, `railway.json` — for deploy-type tasks
- Test config files (`jest.config.*`, `pytest.ini`) — for test-type tasks
- Schema files — for seed/migration tasks
- Existing `Makefile` or shell scripts — to inherit known-good patterns
- `git status` output — uncommitted changes, current branch

**Never included:**
- `node_modules/`, `.venv/`, build output directories
- Full source file contents unless directly relevant
- Secret or credential files

The goal: the LLM generates correct commands on the first attempt because it understands the repo, without being sent the entire codebase.

---

## Compiled targets

Running an LLM on every `vibe run` would be expensive and slow. Vibefile avoids this through **target compilation**: the first time a codegen target runs, the generated shell is saved to disk. Subsequent runs execute the saved shell directly — no LLM call, no latency, no API cost.

The mental model: the first `vibe run build` is the compile step. Every run after that executes the compiled output. The Vibefile is source code. The generated shell script is the binary.

```
first run                          subsequent runs
──────────────────────────────     ───────────────────────────
vibe run build                     vibe run build
  → collect repo context             → load .vibe/compiled/build.sh
  → call LLM API          (cost)     → run in sandbox           (free)
  → save .vibe/compiled/build.sh
  → run in sandbox
```

### Cache location

Compiled scripts live in `.vibe/compiled/`. This directory **must be committed to version control.**

Why:

- **CI runs are free and fast.** CI environments run `vibe run test` using the cached script — no LLM call, no API key needed, no latency. Without committed compiled output, every CI run would require an API key and incur LLM costs.
- **One person pays the cost.** When a recipe changes, one developer runs `vibe run` (or `--recompile`), the LLM generates the script, and it's committed alongside the Vibefile change. Every subsequent run — by any teammate, in any CI pipeline — uses the cached script for free.
- **Auditability.** Code review on a Vibefile change shows both the intent change (the recipe) and the implementation change (the compiled shell). Reviewers can catch problems before they reach production.
- **Reproducibility.** The same script runs everywhere. No variance from different LLM responses across machines or API calls.

```
my-project/
├── Vibefile
└── .vibe/
    ├── config.yaml
    ├── compiled/
    │   ├── build.sh          ← compiled codegen output
    │   ├── build.lock        ← checksum of inputs that produced build.sh
    │   ├── test.sh
    │   └── test.lock
    └── skills/
```

Projects should **not** add `.vibe/compiled/` to `.gitignore`. The `vibe init` command (when implemented) will generate an appropriate `.gitignore` that excludes the binary but keeps the compiled output tracked.

### Cache invalidation

A target's compiled output is invalidated when any of its inputs change. The `.lock` file stores a checksum of everything that was sent to the LLM for that target:

```yaml
# .vibe/compiled/build.lock
recipe: "compile and bundle the project for production"
model: claude-sonnet-4-6
context_files:
  package.json: sha256:a1b2c3...
  tsconfig.json: sha256:d4e5f6...
  Vibefile: sha256:7g8h9i...
variables:
  env: production
generated_at: 2025-03-08T14:22:00Z
```

On each run, the CLI recomputes the checksum of current inputs and compares it to the `.lock` file. If anything has changed — the recipe, a variable value, a relevant config file, or the model — the target is recompiled. Otherwise the cached script runs directly.

This is intentionally similar to how build tools like Turborepo and Bazel handle input hashing, applied to LLM-generated code.

### What triggers a recompile

| Change | Recompiles? |
|--------|:-----------:|
| Recipe string edited in Vibefile | ✅ Yes |
| Variable value changed | ✅ Yes |
| Relevant context file changed (e.g. `package.json`) | ✅ Yes |
| Model version changed | ✅ Yes |
| Skill updated to a new version | ✅ Yes |
| Unrelated source files changed | ❌ No |
| Running on a different machine (same inputs) | ❌ No |

### Codegen targets only

Compilation applies to **codegen mode** only. Agent targets (`@mcp`) are inherently dynamic — they interact with live external services, inspect state, and adapt at runtime. Caching their output would be meaningless and potentially dangerous.

```makefile
build:
    "compile and bundle for production"
    # ✅ compiled — same commands every time

deploy: build
    "deploy to production on fly.io and verify health"
    @mcp fly-mcp
    # ❌ not compiled — agent adapts to live state each run
```

### Force recompile

Users can force a target to recompile regardless of cache state:

```sh
vibe run build --recompile       # force LLM call for this target
vibe run build --recompile-all   # force LLM call for this target and all dependencies
```

### Reviewing compiled output

Because compiled scripts are committed to git, they are auditable. Code review on a Vibefile change will show both the intent change (the recipe) and the implementation change (the compiled shell). This is a feature: it gives teams visibility into what the AI generated and the opportunity to catch problems before they reach production.

The compiled script can also be hand-edited if needed. The CLI will detect that the file was manually modified (checksum mismatch without a recipe change) and warn, but will still execute the edited version. To regenerate from the recipe, run with `--recompile`.

---

## Failure handling

Not all failures are equal. When a generated script exits with a non-zero code, the CLI needs to know: was the script wrong, or did the script correctly detect a real problem? The answer determines whether to retry, cache, or just report the failure.

### Exit code convention

Generated scripts use exit codes as a protocol between the script and the CLI:

| Exit code | Meaning | CLI behaviour |
|-----------|---------|---------------|
| `0` | Success | Cache the script, report success |
| `1` | **Task failed legitimately** | Report failure, do **not** retry — the script is correct but the task found a real problem (tests fail, lint errors, etc.) |
| `2` | **Precondition not met** | Report the missing requirement, do **not** retry — the environment isn't ready |
| `≥ 3` | **Script error / generation bug** | Auto-retry with error context (up to `max_retries` attempts) |

The system prompt instructs the LLM to follow this convention when generating scripts. Exit 1 means "I ran the task correctly and it found a problem." Exit 2 means "I checked and a required tool or version is missing." Any other non-zero exit (3+) typically means the script itself is broken — a wrong command, bad flags, or a misunderstanding of the project.

### Preflight checks

The system prompt instructs the LLM to emit a **preflight section** at the top of every generated script. This section verifies that required tools and versions are available before doing anything, and exits with code 2 if something is missing:

```bash
#!/bin/bash
set -euo pipefail

# --- preflight ---
command -v go >/dev/null 2>&1 || { echo "error: go is required but not installed"; exit 2; }
go_version=$(go version | grep -oP 'go\K[0-9]+\.[0-9]+')
[[ "$(printf '%s\n' "1.25" "$go_version" | sort -V | head -1)" == "1.25" ]] || { echo "error: go >= 1.25 required (found $go_version)"; exit 2; }

# --- task ---
go test -race -v ./...
```

The LLM infers what to check from the project context — `go.mod` tells it the Go version, `package.json` tells it the Node version, and so on. Preflight checks **verify but never install** — the script should not run `apt-get install` or `brew install` for system-level dependencies. If a tool is missing, the script reports what's needed and exits.

This gives the user a clear, actionable error message ("go >= 1.25 required") instead of a cryptic failure halfway through execution.

### Auto-retry on generation errors

When a script exits with code 3 or higher, the CLI assumes the script itself is wrong and retries:

```
vibe run build
  → generate script (attempt 1)
  → execute → exit code 127 (command not found)
  → retry: send error output back to LLM
    "The script failed with: line 5: pnpm: command not found
     The project uses npm, not pnpm. Regenerate the script."
  → generate script (attempt 2)
  → execute → exit code 0
  → cache the fixed script
```

The retry prompt includes:
- The original task recipe
- The script that failed
- The full stderr/stdout from the failed execution
- An instruction to fix the problem

Retry behaviour is configurable:

```makefile
max_retries = 2     # default: 1 (one retry after the initial attempt)
```

```sh
vibe run build --no-retry   # disable retry, fail immediately
```

If all retries are exhausted, the CLI fails with the last error and does **not** cache the broken script.

### Caching and failure interaction

How failure interacts with the compiled target cache:

| Outcome | Cache the script? | Why |
|---------|:-----------------:|-----|
| Exit 0 — success | ✅ Yes | Script works |
| Exit 1 — legitimate failure | ✅ Yes | Script is correct; the task found a real problem (user fixes their code and reruns) |
| Exit 2 — precondition not met | ✅ Yes | Script is correct; the environment needs setup (user installs the tool and reruns) |
| Exit 3+ — retry succeeds | ✅ Yes | Cache the **fixed** version |
| Exit 3+ — retries exhausted | ❌ No | Script is broken; don't persist it |

The key insight: exit 1 and exit 2 scripts are **correct scripts**. Caching them means the next `vibe run` skips the LLM call entirely — the user fixes their code or installs the missing tool, and reruns instantly.

### Dependency failure

When a target's dependency fails, the default behaviour is to **stop the chain**. Remaining dependencies are skipped and the dependent target does not run.

```
vibe run deploy        # depends on: test, build
  → run test → exit 1 (tests fail)
  → skip build (dependency test failed)
  → skip deploy
  ✗ deploy failed: dependency "test" failed
```

This is the safe default. A `--continue-on-error` flag may be added in the future for CI scenarios where you want to run all independent targets and collect all failures.

---

## CLI interface

```sh
vibe init                         # detect project type and generate a Vibefile
vibe init --language <lang>       # use a specific language template
vibe init --empty                 # create a minimal skeleton Vibefile (no detection)
vibe init --force                 # overwrite an existing Vibefile
vibe run <target>                 # run a target and its dependencies
vibe run <target> --dry           # print what would be executed without running
vibe run <target> --recompile     # force LLM recompile for this target
vibe run <target> --recompile-all # recompile this target and all deps
vibe list                         # list all targets with their descriptions
vibe check                        # validate the Vibefile without running anything
vibe status                       # show compiled/uncompiled state of all targets
```

### `vibe init`

Bootstraps a new Vibefile by detecting the project's language, framework, and infrastructure from manifest files (`go.mod`, `package.json`, `Dockerfile`, etc.) and generating targets appropriate for the detected stack. No LLM call is required — templates are preconfigured.

Detection uses a pluggable registry of detectors. Built-in detectors are compiled into the CLI; community detectors can be added as YAML template files:

- `.vibe/templates/<lang>.yaml` — project-local template override
- `~/.vibe/templates/<lang>.yaml` — user-global template override
- Built-in templates — always available as fallback

Language detectors and infrastructure detectors run independently and their results are merged. A Go project with a Dockerfile gets Go targets plus a Docker target.

#### `--empty` mode

When `--empty` is passed, `vibe init` skips all detection and creates a minimal skeleton Vibefile containing only the `model` variable, a `name` variable derived from the directory name, and commented-out examples showing the target syntax. This is useful for:

- Projects where auto-detection produces targets that don't match the desired workflow
- Codebases with unconventional structures that detectors don't recognize
- Users who prefer to define their targets from scratch

```sh
vibe init --empty       # creates a skeleton Vibefile with no targets
vibe init --empty --force  # overwrite an existing Vibefile with a skeleton
```

The generated skeleton:

```makefile
model = claude-sonnet-4-6
name  = my-project

# Add your targets below. Each target has a name, an optional dependency
# list, and a plain-English recipe describing what the task should do.
#
# Example:
#
# build:
#     "compile the project for production"
#
# test:
#     "run all tests with verbose output"
#
# deploy: test build:
#     "deploy to production and verify health"
#     @require clean git status
```

---

## Complete example

```makefile
# Vibefile

model   = claude-sonnet-4-6
project = my-saas-app
env     = production

# ── tasks ────────────────────────────────────────────────

build:
    "compile and bundle the project for $(env)"

test:
    @skill python-test

seed: build
    "populate the database with realistic fake data for 10 users"

migrate: build
    "run any pending database migrations safely"

deploy: test build
    "deploy to $(env) on fly.io and verify the app came up healthy"
    @require clean git status
    @mcp fly-mcp

review:
    "open a PR, write a summary of changes, request review from the team"
    @mcp github-mcp

release: test build
    @skill python-publish
    "also update the changelog and tag the commit"
    @require clean git status
```

---

## Open questions

These are actively undecided. Open an issue or join the Discord.

1. **Dynamic vs. declarative server resolution** — should `@mcp` always require an explicit server definition, or can the agent discover servers dynamically from a registry based on the task description alone?
2. **Registry protocol** — should Vibefile define a minimal registry query API that any implementation satisfies, or adopt an existing standard (e.g. the official MCP registry API)?
3. **Registry-free operation** — how much should work with zero registry configured? Ideally everything, with registry being an enhancement not a requirement.
4. **`.vibe/config.yaml` vs inline** — is a separate config file the right call, or is there a way to keep everything in the Vibefile without it becoming noisy?
5. ~~**`@require` evaluation**~~ — resolved: LLM-generated preflight checks use exit code 2 for precondition failures. CLI-side `@require` checks may be added later as an optimization.
6. **Variable syntax** — `$(VAR)` Make-style, or `${VAR}` shell-style, or something else?
7. **Target naming** — should hyphens and underscores both be valid, or pick one?
8. ~~**Failure behaviour**~~ — resolved: dependency failures stop the chain. See [Failure handling](#failure-handling).
9. ~~**Codegen transparency**~~ — resolved: generated scripts are always shown before execution. The `--dry` flag shows them without executing.
10. ~~**Compiled output in git**~~ — resolved: `.vibe/compiled/` must be committed. CI runs use cached scripts with zero LLM cost. See [Compiled targets — Cache location](#cache-location).
11. ~~**Context file tracking**~~ — resolved: the `.lock` file records checksums of all context files used for generation. On each run the CLI recomputes and compares. See [Cache invalidation](#cache-invalidation).
12. ~~**Manual edits to compiled scripts**~~ — resolved: supported with a warning. The CLI detects hand-edits via script hash mismatch and warns, but still executes. Use `--recompile` to regenerate from the recipe.
13. **Sandbox runtime** — Docker is the obvious default, but it's a heavy dependency. Should a lighter-weight option (e.g. a WASM sandbox, `bubblewrap` on Linux) also be supported?
14. **Sandbox image** — what base image does the sandbox use? A fixed minimal image, or one inferred from the repo's detected stack (e.g. a Node image for a JS project)?
15. **Repo mount granularity** — should the entire repo be mounted read-write, or should write access be restricted to specific directories the task declares it needs?

---

*Vibefile is an open spec. Contributions welcome.*