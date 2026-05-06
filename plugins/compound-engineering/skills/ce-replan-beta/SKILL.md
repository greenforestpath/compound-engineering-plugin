---
name: ce-replan-beta
description: "[BETA] Replan from an existing PR after new learnings have emerged. Re-grounds at the brainstorm tier — re-walks the user story and re-questions original requirements rather than patching the existing plan in place. Produces a fresh plan doc naming what to discard, what to cherry-pick from the existing PR, and the revised approach. Original PR is preserved as a superseded artifact; no Git execution. Use when a PR's approach has been outgrown by review back-and-forth, code reading, or a new brainstorm. Invoke with /ce-replan-beta [PR number, or blank for current branch's PR]."
disable-model-invocation: true
argument-hint: "[PR number, or blank for current branch's PR]"
---

# Replan from an Existing PR (Beta)

`ce-brainstorm` defines **WHAT** to build. `ce-plan` defines **HOW** to build it. `ce-work` executes. `ce-replan-beta` is for the moment when an existing PR's approach has been outgrown by new learnings — review back-and-forth, code reading, a new brainstorm, or a "this could be much simpler" realization — and the original plan is grounded in assumptions that no longer hold.

This skill produces a fresh plan doc. It does **not** execute Git operations. The original PR and original plan remain untouched on disk and on GitHub; the new plan supersedes them by reference. The user starts a fresh branch from `main` themselves.

The core move is **re-grounding at the brainstorm tier**, not patching the existing plan in place. Treat the original plan and PR as evidence to interrogate, not as authoritative framing to inherit.

## Input Argument

<input> #$ARGUMENTS </input>

| Argument | Mode |
|----------|------|
| Blank | **Auto-detect** — target the current branch's PR |
| PR number (e.g., `1234`) | **Explicit** — target the named PR |

**If no PR can be found** (blank argument and no PR for current branch, or explicit PR number does not exist or is inaccessible), do not write a plan. Explain the constraint and route the user to `ce-plan` for fresh planning or `ce-brainstorm` if the work is upstream of planning.

## Interaction Method

When asking the user a question, use the platform's blocking question tool: `AskUserQuestion` in Claude Code (call `ToolSearch` with `select:AskUserQuestion` first if its schema isn't loaded), `request_user_input` in Codex, `ask_user` in Gemini, `ask_user` in Pi (requires the `pi-ask-user` extension). Fall back to numbered options in chat only when no blocking tool exists in the harness or the call errors. Never silently skip the question.

## Pipeline Mode

When invoked from an automated workflow such as LFG or any `disable-model-invocation` context, run non-interactively: skip the synthesis confirmation prompt (Phase 3) and the handoff menu (Phase 5). Inferred bets route to a `## Assumptions` section in the output doc instead of being user-confirmed.

## Phase 0: Mode Detection

Read `<input>` above to determine the mode:

| Argument | Mode | Action |
|----------|------|--------|
| Blank | **Auto-detect** | Continue to Phase 1; PR is the current branch's PR |
| PR number (e.g., `1234`) | **Explicit** | Continue to Phase 1; PR is the explicit number |

If the user provided something that is not a PR number and not blank (e.g., a URL, a branch name, a path), surface the input and ask which PR they meant before continuing.

## Phase 1: Discovery

Run the three discovery scripts. Use the platform's parallel execution capability when independent calls are supported.

### 1.1 PR detection

```bash
bash scripts/detect-pr.sh "$PR_NUMBER"
```

- Argument is the explicit PR number, or empty for auto-detect.
- Exit code `0` with JSON on stdout: PR found. Capture the PR's `number`, `url`, `title`, `body`, and `headRefName` (branch).
- Exit code `2`: no PR found for current branch. **Route to F3 — No-PR redirect**: do not write a plan. Tell the user that `ce-replan-beta` is anchored to existing PRs, then point them to `ce-plan` for fresh planning or `ce-brainstorm` if the work is upstream of planning. End the skill.
- Exit code `1`: gh CLI error. Surface the error verbatim and end the skill — do not produce a degraded plan against missing data.

### 1.2 PR context fetch

```bash
bash scripts/fetch-pr-context.sh "$PR_NUMBER"
```

- Returns a single JSON object with PR metadata, review threads, review bodies, top-level comments, and commits.
- Save the output to a temporary file under `mktemp -d -t ce-replan-XXXXXX` and pass the path (not the content) to subagents per `docs/solutions/skill-design/pass-paths-not-content-to-subagents-2026-03-26.md`.

### 1.3 Original plan discovery

Write the PR body from step 1.1 to a temporary file so `find-original-plan.sh` can scan it for explicit plan-doc links:

```bash
bash scripts/find-original-plan.sh "$HEAD_REF" "$PR_BODY_FILE"
```

- Empty stdout means no candidate cleared the heuristic. Continue without an original plan; surface the gap in the synthesis (frontmatter `original_plan: null`).
- Non-empty stdout: a repo-relative path. Read the candidate plan doc.

### 1.4 Recent brainstorm scan

Use the native file-search tool (e.g., Glob in Claude Code) to list `docs/brainstorms/*-requirements.md` files modified in the last 30 days. Read any that match the PR's topic by filename or content. New brainstorms postdating the original plan often *are* the new learning.

## Phase 2: Re-Grounding

Read `references/regrounding-workflow.md` for the four-step re-derivation pattern, anti-patterns to avoid, and a worked example. Execute the four steps against the artifacts gathered in Phase 1:

1. Read artifacts in order — PR threads first, original plan last.
2. Re-derive the problem frame from the user's discussion language.
3. Mark every original requirement `[unchanged]` / `[revise]` / `[discard]` with explicit reasoning for revisions and discards.
4. Compose a three-bucket synthesis (Stated / Inferred / Out of scope).

## Phase 3: Synthesis Checkpoint

Present the synthesis composed in Phase 2 to the user as prose (not a menu — option sets bias the answer). Wait for confirmation, revision, or redirect before writing the doc.

If the user revises, integrate the change and re-present the revised synthesis. Doc-write (Phase 4) fires only on explicit confirmation.

**Pipeline mode:** skip this phase entirely. Route Inferred bets to a `## Assumptions` section in the output doc so downstream review can scrutinize them as un-validated agent inferences.

## Phase 4: Write Plan Doc

Read `references/doc-template.md` for the canonical output template, filename pattern, and discipline checks.

1. Determine the output filename: `docs/plans/YYYY-MM-DD-NNN-replan-<topic>-beta-plan.md`. Today's date; next sequence number for the day starting at `001`; topic is a kebab-cased short label (3-5 words) derived from the new approach.
2. Create `docs/plans/` if it does not exist.
3. Compose the doc following the template — frontmatter (with `original_pr`, `original_plan`, `supersedes`), Summary, Re-Grounded Problem Frame, Requirements with `[unchanged]`/`[revise]`/`[discard]` annotations, Discarded Approaches, Cherry-Pick Guidance, Supersedes, New Learnings, Scope Boundaries, Context & Research, Key Technical Decisions, Implementation Units, Suggested Branch Name, Sources & References.
4. Run the discipline checks listed at the bottom of `references/doc-template.md` before writing.
5. Use the Write tool to save the file. Use repo-relative paths everywhere — never absolute paths.
6. Confirm the file path back to the user (absolute path so the reference is clickable in modern terminals).

The original PR and original plan are referenced but **never** edited or deleted. This skill writes one new file and stops.

## Phase 5: Handoff

**Pipeline mode:** skip this phase. Return control to the caller after the plan file is written.

**Interactive mode:** present the handoff menu using the platform's blocking question tool.

**Question:** "Replan written to `<absolute path>`. What would you like to do next?"

**Options:**

1. **Start `/ce-work` against the new plan** (recommended) — invoke the `ce-work` skill via the platform's skill-invocation primitive (`Skill` in Claude Code, `Skill` in Codex, the equivalent on other platforms), passing the new plan path as the argument. Do not merely tell the user to type a command — fire the invocation.
2. **Open in Proof (web app) for review** — load the `ce-proof` skill in HITL-review mode with the new plan path as `source file`. Follow the post-HITL resync logic: if Proof returns material edits, re-run `ce-doc-review` against the doc before handing off to `ce-work`.
3. **Done for now** — display a brief confirmation and end the turn.

For free-text revisions outside the three options, accept the input, apply the revision to the doc, and loop back to this menu.

**Completion check:** the skill is not complete until the post-write menu has been presented, the user has selected an option, and the inline routing for that option has been executed. Presenting the menu and stopping at the user's selection is not completion — fire the routed action.

---

> **Note:** This is a beta skill. The invocation contract, doc shape, and discovery heuristics may change before promotion to a stable `ce-replan`. See `docs/solutions/skill-design/beta-skills-framework.md` for the framework and promotion path.
