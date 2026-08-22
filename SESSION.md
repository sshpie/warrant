# Warrant — Session State

Last updated: 2026-05-24

**Project.** Warrant is a book-grounded autonomous coding agent: a Claude Code
skill that runs a fully autonomous coding agent grounded in O'Reilly engineering
books, given a single direction. Built as separate artifacts, one at a time:
**A** the skill, then follow-on artifacts (**B** a CLI / the "wild Bill"
no-sandbox variant Nick floated, **C** a shareable kit). Design spec:
`docs/superpowers/specs/2026-05-21-warrant-design.md`.

**Artifact A is complete.** The Librarian (retrieval engine) and the Agent
(autonomous loop, all 4 plans) are done and on `main`. The skill is invokable
as `/warrant <direction>` in any Claude Code session with a `.warrant/config.json`.

## Done — the Librarian (Artifact A's retrieval engine)

The Librarian indexes reusable engineering principles from the `colophon` book
library into a HybridRAG index: a semantic vector index plus a principle graph,
every principle carrying a citation (book/chapter/section) and a checkability
tier (1 mechanical, 2 measurable, 3 judgment).

- Built via `superpowers:subagent-driven-development` from a 12-task plan
  (`docs/superpowers/plans/2026-05-21-warrant-librarian.md`). Each task got an
  implementer + a spec review + a code-quality review. A final
  whole-implementation review caught a corpus-parser bug every per-task review
  missed (real chapter Markdown opens with an H2, not an H1).
- On `main` (branch `librarian` merged fast-forward, then deleted). 33 tests
  pass. Package at `librarian/`; modules: models, corpus, extract, llm, edges,
  embedding, store, indexer, query, cli.
- CLI: `librarian index <book-library>` builds the index; `librarian query
  <text>` retrieves ranked principles with citations and tiers.
- Live-verified: indexed a real book (*AI for Mass-Scale Code Refactoring and
  Analysis*) end to end — 190 principles, correct citations, working query.
  The smoke test exposed two crash bugs the stub-LLM tests hid: an unwrapped
  edge-extraction call and a principle-id collision. Both fixed (`45269b7`,
  `d0d700f`).
- Edge extraction redesigned (`1984874`): the old version sent every principle
  in one prompt and overflowed the token limit on a real book, leaving the
  graph empty. Candidate edges are now gated by embedding similarity (each
  principle's nearest neighbors) and classified in bounded LLM batches, so the
  request size is capped regardless of corpus size. Unit-verified — the
  batching scales; a live re-verify on a real book is still pending.

## Done — the Agent, plan 1 of 4: the plan artifact

The plan artifact is the versioned decision-tree data structure the agent loop
builds, versions, amends, and persists. A self-contained Python package under
`agent/`, with zero runtime dependencies beyond the standard library.

- Built via `superpowers:subagent-driven-development` from the 9-task plan
  `docs/superpowers/plans/2026-05-21-warrant-plan-artifact.md`. Each task got an
  implementer, a spec-compliance review, and a code-quality review; a final
  whole-implementation review closed it out.
- Merged to `main` (branch `agent-plan-artifact`, fast-forward, then deleted).
  69 tests pass. HEAD `ba12a26`.
- Three modules under `agent/agent/`: `plan.py` is the schema (`ApplicableCheck`,
  `PlanNode`, `Plan` frozen dataclasses with `__post_init__` validation);
  `planstore.py` is I/O (dict round-trip plus versioned `plan.v{N}.json`
  persistence); `planops.py` is operations (`new_plan`, `add_node`, `amend_node`,
  `next_version` for mutation; `find_node`, `children`, `independent_siblings`
  for queries). The three modules form a clean stack: `planstore` and `planops`
  both import from `plan`, and neither imports the other.
- One fix landed beyond the plan's literal code. A code review caught that
  `amend_node` forwarded its `**changes` straight into `dataclasses.replace`, so
  a caller could pass `id=` and rewrite a node's identity, producing duplicate
  ids and breaking the unique-id invariant `add_node` enforces. `amend_node` now
  rejects `id`, `amended_from`, and `amended_reason` in `changes` (commit
  `df8313c`).

## Done — the Agent, plan 2 of 4: the loop package

The `loop/` package drives the Orient → Retrieve → Plan → Execute phases.

- Built via `superpowers:subagent-driven-development` from the 13-task plan
  `docs/superpowers/plans/2026-05-24-warrant-agent-loop.md`.
- Merged to `main`. All tests pass. HEAD `2ec060180b94e4785a3942bb1b14a69cae2819be`.
- Modules: `models`, `runstore`, `worktree`, `materializer`,
  `phases/{orient,retrieve,plan,execute}`, `runner`.
- `WarrantRunner.run(direction)` drove Orient → Retrieve → Plan → Execute;
  `.resume(run_state)` re-entered Execute from a checkpoint.
- `Invoker` protocol: plan 4 skill wrapper provides `ClaudeCodeInvoker`;
  tests use `FakeInvoker`.

## Done — the Agent, plan 3 of 4: Verify + citation report

Extends `loop/` with a Verify phase and CitationReport projection. `WarrantRunner.run()`
now drives Orient → Retrieve → Plan → Execute → Verify and returns
`tuple[RunState, CitationReport]`.

- Built via `superpowers:subagent-driven-development` from the 8-task plan
  `docs/superpowers/plans/2026-05-24-warrant-verify-citationreport.md`. Design spec:
  `docs/superpowers/specs/2026-05-24-warrant-verify-citationreport-design.md`.
- Merged to `main` (branch `worktree-agent-verify`, fast-forward, then deleted).
  95 tests pass. HEAD `77db7a1`.
- New modules: `verifier_materializer.py`, `citationreport.py`, `phases/verify.py`.
- Updated: `models.py` (added `VerifierCheckOutcome`, `VerifierResult`, `CitationReport`,
  `VerifierInvoker` Protocol, `pre_execution_sha` on `NodeStatus`); `phases/execute.py`
  (records `pre_execution_sha` via `_get_head_sha` before each dispatch batch);
  `runner.py` (added `_execute_verify_loop`, `verifier_invoker`, `verify_iteration_cap`).
- `VerifierInvoker` protocol: `invoke(prompt, timeout) -> VerifierResult`. Tests use
  `FakeVerifierInvoker`. Live skill wrapper provides the real invoker in plan 4.
- `integrity_failure` verdict routes node back to pending and re-executes; at
  `per_node_attempt_cap` the node is amended + marked failed. `audit_catch` and
  `clean` leave the node done. Verifier exceptions produce a synthetic clean result
  (no routing loop).
- `CitationReport` counts grounded/conflicted/ungrounded nodes, tier-1/2/3 checks,
  amendments, and sets `suspiciously_clean` flag when no stress signals appear on
  plans of >= 5 nodes. `render_citation_report()` produces formatted text output.
- Two bugs caught and fixed by reviews: `_get_diff` untracked-file quoting (use
  `git ls-files -z` not `git status --short`); exhausted-phase spin in
  `_execute_verify_loop` (break when `phase == "exhausted"`).

## Done — the Agent, plan 4 of 4: skill wrapper

Wires `WarrantRunner` to real Claude CLI invocations and delivers the `/warrant`
Claude Code skill entry point. Artifact A is now fully complete.

- Built via `superpowers:subagent-driven-development` from the 4-task plan
  `docs/superpowers/plans/2026-05-24-warrant-skill-wrapper.md`. Design spec:
  `docs/superpowers/specs/2026-05-24-warrant-skill-wrapper-design.md`.
- Merged to `main` (branch `skill-wrapper`, fast-forward, then deleted).
  109 tests pass. HEAD `bfcf95b`.
- New subpackage `loop/loop/skill/` with three modules:
  - `invokers.py`: `ClaudeCodeLLM` (callable LLM), `ClaudeCodeInvoker`
    (Invoker protocol), `ClaudeCodeVerifierInvoker` (VerifierInvoker protocol).
    All call `claude -p <prompt>` via subprocess. `_extract_json` strips markdown
    fences and falls back to brace extraction. Synthetic failed/clean results on
    parse failure prevent routing loops.
  - `factory.py`: `Config` dataclass (11 fields with defaults), `load_config()`
    (reads JSON, ignores unknown keys), `build_runner()` (assembles
    `WarrantRunner` from config).
  - `__main__.py`: `run` and `resume` subcommands via argparse. Dispatches to
    `runner.run(direction)` or `runner.resume(run_state)`, prints citation report
    + worktree path.
- `SKILL.md` at repo root: thin Claude Code skill wrapper. Checks for
  `.warrant/config.json`, dispatches `/warrant resume` to the resume subcommand
  (conditional on `$ARGUMENTS = "resume"`), otherwise runs the agent.
- `.warrant/config.example.json` documents all 9 required + optional fields.
- One integration bug caught by final whole-implementation review: SKILL.md
  was unconditionally calling `loop.skill run` even on `/warrant resume`,
  so the resume path would have started a new run with direction="resume".
  Fixed with a bash conditional before merge.

## Done — Artifact B: standalone CLI

Adds a real-API / no-sandbox variant of the agent that runs `warrant` as a
standalone binary against a live Anthropic API key, outside the Claude Code
skill context.

- Built via `superpowers:subagent-driven-development` from the 8-task plan
  `docs/superpowers/plans/2026-05-24-warrant-artifact-b.md`. On branch
  `artifact-b`.
- 240 tests pass (136 loop / 69 agent / 35 librarian).
- New subpackage `loop/loop/api/` with four modules:
  - `sandbox.py`: `WorktreeSandbox` — clones the base repo into a temp
    worktree, exposes `apply_diff` and `run_command`, and tears down cleanly.
  - `invokers.py`: `AnthropicInvoker` and `AnthropicVerifierInvoker` — call
    the Anthropic API directly (no subprocess). `_extract_json` strips fences,
    falls back to brace extraction. Synthetic failed/clean results on parse
    failure prevent routing loops.
  - `factory.py`: `ApiConfig` dataclass (12 fields with defaults, no
    `anthropic_api_key` — that comes from `ANTHROPIC_API_KEY` env), `load_api_config()`,
    `build_api_runner()`.
  - `__main__.py`: `run` and `resume` subcommands. Entry point registered as
    the `warrant` console script.
- `ExecutionMaterializer` and `VerifierMaterializer` both gained a
  `## Working directory` section so the agent and verifier always know where
  to operate.
- `.warrant/api-config.example.json` documents all 12 fields. `anthropic_api_key`
  is intentionally absent; users set `ANTHROPIC_API_KEY` in the environment.

## Done — Artifact C: shareable starter kit

Makes the repo self-contained for new users: `git clone` → `make install-api` →
`warrant init` → `warrant run --direction "..."` in under 10 minutes, no private
book corpus needed.

- Built via `superpowers:subagent-driven-development` from the 5-task plan
  `docs/superpowers/plans/2026-05-24-warrant-artifact-c.md`. Design spec:
  `docs/superpowers/specs/2026-05-24-warrant-artifact-c-design.md`.
- 249 tests pass (145 loop / 69 agent / 35 librarian). HEAD at latest main commit.
- New additions:
  - `README.md`: project face — quick start, config reference, structure.
  - `Makefile`: `install`, `install-api`, `test`, `demo` targets.
  - `warrant init` subcommand in `loop/loop/api/__main__.py`: interactive
    scaffolding of `.warrant/api-config.json` (3 prompts, defaults to
    `sample-library/index`). Default config path fixed from `.warrant/config.json`
    to `.warrant/api-config.json`.
  - `sample-library/principles.json`: 15 hand-crafted engineering principles
    (design/robustness/testing/architecture/ops). Edge kinds encoded per-entry
    (refines / contradicts / shares_topic — no LLM blanket assignment).
  - `sample-library/build_index.py`: one-time build script calling librarian
    internals directly (no LLM, no corpus parsing). Builds from JSON → embedded
    index. Run once during dev; output committed to git.
  - `sample-library/index/`: pre-built librarian index (15 principles, 384-dim
    embeddings, 18 edges). Committed so users get a working index immediately.
  - `sample-library/demo-config.json`: points at the bundled index with reduced
    caps for a fast demo run.
- New test file: `loop/tests/test_api_init.py` (8 tests).

## Open / next

1. Optional: live re-verify of Librarian edge extraction on a real book.
2. Optional: end-to-end live test of the full `/warrant <direction>` skill
   against a real codebase with a built Librarian index.
3. Optional: push to GitHub remote (`https://github.com/sshpie/warrant`)
   so Artifact C is publicly visible.

## Notes

- `colophon-library` (the book corpus) finished at 195 books — PRIVATE repo
  (full book text is copyrighted).
- The Tier-1 *check compiler* is deliberately out of scope — its own future
  sub-project; `checkability_tier` is populated but not compiled.
- `~/warrant` has a GitHub remote: `https://github.com/sshpie/warrant`
- To use the skill: build a Librarian index, copy `.warrant/config.example.json`
  to `.warrant/config.json` in your project, fill in `index_path` and
  `base_repo`, then invoke `/warrant <direction>` in Claude Code.
