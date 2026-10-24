# Claude-to-Codex Agentic Migration

This guide walks through migrating Claude Code skills and agents into
Codex-facing skill and custom-agent artifacts using the Codex skills in this
repository.

> ⚠ This toolkit is experimental. Transforms are best-effort and the output is
> expected to be reviewed before use.

## Audience and prerequisites

You are an operator with:

- Python 3.11+ available on `PATH`.
- OpenAI Codex CLI installed locally when you want to exercise migrated outputs.
- A target repository with Claude source inputs under `.claude/skills/` and/or
  `.claude/agents/`. Optional project-level inputs may also exist under
  `.claude/commands/` and `.claude/settings.json` hook settings.
- This repository checked out, with `requirements.txt` installed:

  ```bash
  python3 -m pip install -r requirements.txt
  ```

## Conceptual model

```text
Claude source artifacts -> Codex migration skills -> Codex-facing outputs
```

The `.claude/skills/` and `.claude/agents/` directories are source input only.
For batch skill migration, `.claude/commands/**/*.md` and project-local Claude
settings hooks are also source input. The migration reads them, preserves them,
and writes converted artifacts under `.codex/`.

A Codex skill package is a directory under `.codex/skills/<name>/` containing
`SKILL.md` plus optional `scripts/`, `references/`, `assets/`, or `tests/`.
A Codex custom agent is a TOML file under `.codex/agents/<name>.toml`.
Command-derived skills are written as `.codex/skills/command-*/SKILL.md`.
Converted command hooks are written to `.codex/hooks.json` and must be checked
before use.

The core migration rule is preservation-first: keep the source workflow's
intent and constraints, then map to Codex-native primitives only when the
behavior actually requires them.

Minimal reference artifacts:

- [`examples/hello-world-skill/SKILL.md`](../../examples/hello-world-skill/SKILL.md)
  follows the official Agent Skill format but is not auto-discovered from
  `examples/`.
- [`examples/hello-world-agent/hello-world-agent.toml`](../../examples/hello-world-agent/hello-world-agent.toml)
  follows the custom-agent TOML shape but is not auto-loaded from `examples/`.

## Install the Codex migration skills

To invoke the migration workflows as Codex skills in a target repository, copy
the operational packages into that repository's `.codex/skills/` directory:

```bash
mkdir -p /path/to/target-repo/.codex/skills
cp -R .codex/skills/migrate-to-codex /path/to/target-repo/.codex/skills/
cp -R .codex/skills/migrate-agents-to-codex /path/to/target-repo/.codex/skills/
cp -R .codex/skills/verify-skill-migration /path/to/target-repo/.codex/skills/
cp -R .codex/skills/migration-dashboard /path/to/target-repo/.codex/skills/
```

Direct Python scripts can still be run from this checkout without copying the
skills first. Do not install these operational workflows into `.claude/skills/`;
that tree is migration input.

## End-to-end workflow

1. **Stage Claude inputs.** Put source skills under
   `.claude/skills/<name>/SKILL.md` and source agents under
   `.claude/agents/<name>.md`. Optionally stage slash commands under
   `.claude/commands/**/*.md` and project-local hooks in `.claude/settings.json`
   or `.claude/settings.local.json`.

2. **Confirm Codex migration skills are installed.** The target repository
   should have `migrate-to-codex`, `migrate-agents-to-codex`,
   `verify-skill-migration`, and `migration-dashboard` under `.codex/skills/`.

3. **Preview or assess before writing.** For skills, run `migrate-to-codex` in
   preview or analysis mode. Named `--skill <name>` runs stay skill-only. Batch
   runs (`--all` or zero-argument execution) also report command and hook
   surfaces. For agents, run `migrate-agents-to-codex` in preview or dry-run
   mode. Resolve blockers before live writes.

4. **Run the live migration.** Skill outputs are written to `.codex/skills/`.
   Agent outputs are written to `.codex/agents/`, with TOML as the preferred
   custom-agent format.

5. **Validate and refresh state.** Run the validator, refresh
   `migration-state.json`, and inspect progress with the dashboard.

6. **Complete nativeness review.** Generated artifacts are Codex-compatible
   first. Treat them as native only after the review flow records a native
   outcome.

7. **Run tests.** Use the direct test files listed below rather than broad
   `unittest discover` against the hidden `.codex` tree.

## Preview and direct script examples

```bash
python3 .codex/skills/migrate-to-codex/scripts/run_migration_workflow.py \
  --preview --skill <skill-name> --json

python3 .codex/skills/migrate-agents-to-codex/scripts/run_agent_migration_workflow.py \
  --preview --agent <agent-name> --json

python3 .codex/skills/migrate-to-codex/scripts/migrate_claude_workflows_to_codex.py \
  --all --dry-run --json

python3 .codex/skills/migrate-to-codex/scripts/migrate_claude_workflows_to_codex.py \
  --analysis-only --json

python3 tools/migration_support/validate_names.py --scan-dir .codex/skills
python3 tools/migration_support/validate_names.py --check environment-readiness --codex-home .codex
python3 tools/migration_support/validate_names.py --scan-dir examples
python3 tools/migration_support/tracker.py --write
python3 .codex/skills/migration-dashboard/scripts/analyze_migration.py --status

python3 .codex/skills/migrate-to-codex/tests/test_migrate_claude_workflows_to_codex.py
python3 .codex/skills/migrate-agents-to-codex/tests/test_migrate_claude_agents_to_codex.py
```

## Running against a target repo

The migration tooling is designed to operate against service-shaped target
repositories where migrated skills and agents will eventually live. This
repository ships an `example-service/` placeholder; equivalents like
`example-admin/`, `example-worker`, and `example-app/` are recognized layouts as
well.

Point the migration workflows at your target tree only after source inputs are
staged and the operational Codex migration skills are available under
`.codex/skills/`.

## What the validator checks

`tools/migration_support/validate_names.py` enforces, at minimum:

- Skill and agent identifiers are lowercase kebab-case.
- Skill packages have readable `SKILL.md` files with valid frontmatter.
- Optional `.codex/hooks.json` parses and referenced hook commands look
  available, through the `environment-readiness` check.
- Paths stay within recognized roots such as `.codex/skills/`, `.codex/agents/`,
  `examples/`, and supported target-repo prefixes.
- Stale paths, unresolved references, executable primitive remnants, and other
  migration drift are surfaced as warnings or failures.

The script is authoritative; read it for the full set of checks.

## Limitations

- `.claude/skills/` and `.claude/agents/` are preserved as source input. Do not
  delete or rewrite them as part of migration unless you are doing a separate
  cutover task.
- Generated artifacts are not automatically native. Review the result and
  complete nativeness review before relying on migrated workflows.
- Command-generated skills and converted hook config preserve Claude behavior
  visibly but remain review-required. Hook conversion preserves command
  handlers only; async hooks, conditionals, non-command handlers, prompt
  mutation, and matcher semantics need manual review.
- Non-standard Claude input layouts may need to be staged manually before
  migration.
- The tracker state schema (`migration-state.json`) is not stable; do not build
  external automation against it yet.
