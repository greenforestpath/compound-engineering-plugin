# Insights Ledger: EveryInc Compound Engineering Plugin

Generated: 2026-06-09

Local path: `/Users/personal/Projects2/FORKS/everyinc__compound-engineering-plugin`

Upstream: `https://github.com/EveryInc/compound-engineering-plugin`

Local fork: `https://github.com/greenforestpath/compound-engineering-plugin`

Inspected commit: `b6250490bec4c0488d68ad66d72bd99f6edb95fd`

## Evidence Contract

This ledger only makes claims backed by one of these evidence types:

- Source provenance: a local file path and line range.
- Command provenance: an exact command plus summarized output.
- GitHub metadata: output from `gh repo view`.
- Inference: explicitly labeled and bounded.

It does not claim that the plugin is safe, complete, useful for a given team, or ready for production. The verified surface is the repository structure, static source behavior, tests run locally, release metadata validation, and two isolated conversion proofs.

## Command Provenance

| Command | Result | Evidence |
|---|---:|---|
| `gh repo view everyinc/compound-engineering-plugin --json ...` | Passed | Upstream is `EveryInc/compound-engineering-plugin`, MIT licensed, default branch `main`, created `2025-10-09T19:43:46Z`, pushed `2026-06-07T18:04:05Z`, updated `2026-06-09T03:21:19Z`. |
| `gh repo fork everyinc/compound-engineering-plugin --clone=false` | Passed | Created/confirmed GitHub fork under `greenforestpath`. |
| `gh repo clone greenforestpath/compound-engineering-plugin /Users/personal/Projects2/FORKS/everyinc__compound-engineering-plugin` | Passed | Local clone created at the path above. |
| `git remote -v` | Passed | `origin` is `greenforestpath/compound-engineering-plugin`; `upstream` is `EveryInc/compound-engineering-plugin`. |
| `git rev-parse HEAD` | Passed | `b6250490bec4c0488d68ad66d72bd99f6edb95fd`. |
| `bun test` before `bun install` | Failed | Environmental failure: dependencies such as `js-yaml` and `citty` were missing. This is not counted as a code failure. |
| `bun install` | Passed | Installed `citty`, `js-yaml`, semantic-release packages, and 310 total packages. |
| `bun test` after install | Passed | `1406 pass`, `0 fail`, `3733 expect() calls`, 52 files. |
| `bun run release:validate` | Passed | `Release metadata is in sync. compound-engineering currently has 43 agents, 39 skills, and 0 MCP servers.` |
| `bun run src/index.ts list` | Passed | Listed `coding-tutor` and `compound-engineering`. |
| `CODEX_HOME=/tmp/ce-codex-proof bun run src/index.ts convert plugins/compound-engineering --to codex --include-skills` | Passed | Wrote Codex artifacts under `/tmp/ce-codex-proof`. |
| `bun run src/index.ts convert plugins/compound-engineering --to opencode --output /tmp/ce-opencode-proof` | Passed | Wrote OpenCode artifacts under `/tmp/ce-opencode-proof/.opencode`. |

## Operator Incident During Study

One exploratory command was run with `--to codex --output /tmp/ce-codex-convert-proof --include-skills`. Source inspection later showed that Codex output ignores `--output` and resolves to `CODEX_HOME` or the default Codex home. The command therefore wrote Compound Engineering artifacts into `/Users/personal/.codex`, not `/tmp`.

Observed live side effects:

- `/Users/personal/.codex/agents/compound-engineering`
- `/Users/personal/.codex/skills/compound-engineering`
- `/Users/personal/.codex/compound-engineering/install-manifest.json`
- write timestamp evidence: `2026-06-09 10:32:11` local time on generated files
- config backup: `/Users/personal/.codex/config.toml.bak.2026-06-09T03-32-11-940Z`
- diff between that backup and current `/Users/personal/.codex/config.toml`: empty

Source provenance for this behavior: `src/utils/resolve-output.ts:14-16` returns `codexHome` for the `codex` target regardless of explicit output. This is an operator gotcha, not necessarily a repository bug, because the install docs describe Codex installation as profile/home based.

No global cleanup was performed after this incident because removing live Codex plugin artifacts could delete a pre-existing user install or otherwise mutate the active agent environment.

## Executive Readout

1. The repository is both a plugin content package and a conversion/install CLI.

   Evidence: `package.json:2-8` names the npm package and binary, while `PRIVACY.md:3-6` says the repo contains a markdown/config plugin package and a CLI that converts and installs plugin content.

2. The core product model is "compound engineering": use planning, review, and reusable learning artifacts to make future engineering work easier.

   Evidence: `README.md:6-21` states the philosophy and workflow purpose. This is a self-description, not independent validation that the workflow achieves those outcomes.

3. Measured component counts are 43 agents, 39 skills, and 0 MCP servers at the inspected commit.

   Evidence: `bun run release:validate` output. This conflicts with older or approximate README counts: root `README.md:75-77` says 37 skills and 51 agents, while `plugins/compound-engineering/README.md:11-15` says 50+ agents and 38+ skills.

4. The CLI currently exposes implemented target handlers for OpenCode, Codex, Pi, Gemini, and Kiro.

   Evidence: `src/targets/index.ts:50-84`. Tests also assert that native marketplace-only targets such as Copilot, Droid, and Qwen are rejected by the CLI install path; see `tests/cli.test.ts:103-127`.

5. Codex support is intentionally hybrid: native plugin install for skills and Bun conversion for custom agents.

   Evidence: `README.md:100-118` says Codex needs marketplace registration, Bun agent install, and TUI plugin install. Source comments and behavior in `src/converters/claude-to-codex.ts:21-27` and `src/converters/claude-to-codex.ts:76-109` make agents-only the default to avoid double-registering skills.

6. The strongest engineering investment is not only content authoring; it is managed artifact conversion, ownership, cleanup, path safety, and cross-tool drift control.

   Evidence: parser and target layers in `src/parsers/claude.ts:16-37`, target registry in `src/targets/index.ts:50-84`, Codex writer in `src/targets/codex.ts:30-144`, manifest filtering in `src/targets/codex.ts:187-240`, and path-safety utility in `src/utils/files.ts:88-123`.

## Architecture Map

### 1. Package and CLI Entrypoint

The package is `@every-env/compound-plugin` version `3.11.2`. It exposes a `compound-plugin` binary at `src/index.ts` and uses Bun/TypeScript module semantics.

Provenance: `package.json:2-8`, `package.json:15-23`, `package.json:25-35`.

### 2. Source Plugin Loader

The loader treats a Claude plugin package as the source format. It resolves `.claude-plugin/plugin.json`, then loads agents, commands, skills, hooks, and MCP servers.

Provenance: `src/parsers/claude.ts:14-37`.

Agent files are Markdown with frontmatter/body extraction.

Provenance: `src/parsers/claude.ts:57-75`.

Command files support frontmatter fields including `allowed-tools` and `disable-model-invocation`.

Provenance: `src/parsers/claude.ts:77-99`.

Skill discovery looks for `SKILL.md`, reads frontmatter, and carries a `ce_platforms` field when present.

Provenance: `src/parsers/claude.ts:101-122`.

### 3. Target Conversion Layer

The target registry maps supported output targets to converter and writer functions. Implemented handlers are:

- `opencode`
- `codex`
- `pi`
- `gemini`
- `kiro`

Provenance: `src/targets/index.ts:50-84`.

The presence of converter files for additional platforms should not be interpreted as CLI install support. The install path uses the target registry, and tests verify that some marketplace-native targets are rejected as unknown.

Provenance: `src/targets/index.ts:50-84`, `tests/cli.test.ts:103-127`.

### 4. Codex Conversion Model

Codex default conversion emits custom agents but suppresses skills, prompts, generated skills, and MCP servers. The stated reason in source comments is that Codex native plugin install should own those assets, while the Bun converter fills the custom-agent gap.

Provenance: `src/converters/claude-to-codex.ts:21-27`, `src/converters/claude-to-codex.ts:76-109`, `tests/codex-converter.test.ts:49-73`.

Full standalone Codex mode exists behind `--include-skills`, where prompts, copied skills, generated command skills, agents, MCP servers, and hooks can flow through conversion.

Provenance: `src/converters/claude-to-codex.ts:112-131`, `tests/codex-converter.test.ts:96-120`.

### 5. Writer and Cleanup Layer

The Codex writer writes prompts, skills, agents, install manifests, config blocks, and hooks. It also performs cleanup for removed managed prompts, skills, agents, known legacy artifacts, and legacy skill directories.

Provenance: `src/targets/codex.ts:30-144`, `src/targets/codex.ts:146-171`.

Install manifests are filtered before cleanup to avoid unsafe paths. The path-safety helper rejects non-strings, empty strings, absolute paths, traversal segments, and resolved paths outside the intended root.

Provenance: `src/targets/codex.ts:187-240`, `src/utils/files.ts:88-123`, `tests/manifest-path-safety.test.ts:20-67`, `tests/manifest-path-safety.test.ts:69-118`, `tests/manifest-path-safety.test.ts:121-169`.

### 6. Privacy and Security Posture

The repo states that the plugin package does not include telemetry or analytics code, does not run a background uploader, and only sends data through host tooling or explicitly invoked integrations.

Provenance: `PRIVACY.md:7-12`.

The same document lists external-service exposure through model providers, optional integrations such as Context7 and Proof, and package manager infrastructure.

Provenance: `PRIVACY.md:13-34`.

Security reporting is private email to `kieran@every.to`; supported fixes apply to latest `main`.

Provenance: `SECURITY.md:3-12`, `SECURITY.md:22-29`.

Bounded claim: these are repository-stated policies and static source observations. This ledger did not audit all skill bodies for every possible external command, network call, prompt-injection surface, or data exfiltration vector.

## Drift and Mismatch Ledger

| Finding | Evidence | Interpretation |
|---|---|---|
| Root README count says 37 skills and 51 agents. | `README.md:75-77` | Stale or different counting basis at inspected commit. |
| Plugin README count says 50+ agents and 38+ skills. | `plugins/compound-engineering/README.md:11-15` | Approximate marketing/reference count. |
| Release validation says 43 agents, 39 skills, 0 MCP servers. | `bun run release:validate` | Treat this as strongest current count because it is generated by repo validation logic. |
| Codex `--output` did not sandbox output. | `src/utils/resolve-output.ts:14-16` plus operator incident | For Codex proofs, set `CODEX_HOME=/tmp/...`; do not rely on `--output`. |
| CLI target support is narrower than all named ecosystems in docs/manifests. | `src/targets/index.ts:50-84`, `tests/cli.test.ts:103-127` | Distinguish native marketplace installation from Bun CLI conversion targets. |

## What The Tests Prove

The local test suite passed after dependency installation.

Verified by command: `bun test` -> `1406 pass`, `0 fail`, 52 files.

The tests cover converter behavior, target writers, release metadata, prefix conventions, manifest path safety, CLI behavior, and multiple skill helper scripts.

Bounded claim: passing tests prove the checked test contracts under the local environment. They do not prove that every generated agent/skill is semantically effective, that every external integration is safe, or that installs work in every host version.

## What The Isolated Conversion Proofs Prove

Codex proof:

- Command used `CODEX_HOME=/tmp/ce-codex-proof`.
- Output contained `AGENTS.md`, `agents/compound-engineering/*.toml`, and `compound-engineering/install-manifest.json`.
- With `--include-skills`, Codex proof also wrote skills under the temporary Codex root.

OpenCode proof:

- Command used explicit `--output /tmp/ce-opencode-proof`.
- Output was nested under `/tmp/ce-opencode-proof/.opencode`.
- Artifacts included agents, skills, and a managed install manifest.

Bounded claim: these are file-generation proofs. They do not prove that Codex or OpenCode loaded and executed the artifacts in an actual interactive session.

## Practical Use Notes

- For local study, keep this fork under `/Users/personal/Projects2/FORKS/everyinc__compound-engineering-plugin`.
- For Codex conversion experiments, always set `CODEX_HOME` to a temporary directory unless the intent is to modify the live Codex profile.
- Treat `bun run release:validate` as the quickest count and metadata drift check.
- Treat `bun test` as a meaningful local integrity check after `bun install`.
- Treat README component counts as less authoritative than release validation unless the docs are updated.

## Open Questions For Further Study

1. Audit individual skill bodies for external command usage, optional integrations, credential expectations, and data movement.
2. Run generated Codex artifacts in a disposable Codex profile and verify actual runtime discovery, not only file generation.
3. Compare native marketplace install output against Bun converter output to identify duplicate, missing, or conflicting artifacts.
4. Inspect release automation and CI workflow history from GitHub Actions if release trust matters.
5. Evaluate the semantic quality of high-impact skills such as `ce-code-review`, `ce-plan`, `ce-work`, and `ce-compound` with real project tasks.
6. Decide whether the live `/Users/personal/.codex` artifacts written during this study should be kept as an install or cleaned using a deliberate uninstall plan.

## Source Index

| Surface | Path | Why it matters |
|---|---|---|
| Package/CLI metadata | `package.json:2-35` | npm package, binary, scripts, dependencies. |
| User-facing philosophy/install docs | `README.md:6-21`, `README.md:100-118` | Product framing and Codex install model. |
| Component docs | `plugins/compound-engineering/README.md:11-18` | Approximate counts and skill reference entry. |
| Parser | `src/parsers/claude.ts:16-122` | Source plugin ingest behavior. |
| Target registry | `src/targets/index.ts:50-84` | Implemented CLI targets. |
| Codex converter | `src/converters/claude-to-codex.ts:21-131` | Hybrid install logic. |
| Codex writer | `src/targets/codex.ts:30-240` | Write, manifest, backup, cleanup, hook behavior. |
| Output resolution | `src/utils/resolve-output.ts:14-16` | Codex output root behavior. |
| File/path utilities | `src/utils/files.ts:44-60`, `src/utils/files.ts:78-123` | secure writes, path sanitization, managed path safety. |
| Path-safety tests | `tests/manifest-path-safety.test.ts:20-169` | Cleanup traversal defense contracts. |
| Codex converter tests | `tests/codex-converter.test.ts:49-120` | Agents-only default and include-skills behavior. |
| CLI target tests | `tests/cli.test.ts:103-127` | Native marketplace-only targets rejected by CLI install. |
| Privacy | `PRIVACY.md:7-34` | Stated data handling model. |
| Security | `SECURITY.md:3-29` | Support and reporting policy. |
