# PR #523: review handoff

This is review material for the personal fork, not part of the proposed semantic
conventions. It explains the follow-up fixes, including the ADK example, so another
reviewer can assess them without the original conversation. Leave this file out
when porting the implementation fixes upstream.

## What to compare

- [Original upstream PR #523](https://github.com/open-telemetry/semantic-conventions-genai/pull/523):
  snapshot `ed599428067e89cfa7c0fe07df7853be1dbfd419`, preserved as `codex/pr523-original`.
- [Implementation fixes](https://github.com/bleedinghands/semantic-conventions-genai/commit/1faaacfb2d0112c9c8ef27d1b6fb0628fc613f14):
  commit `1faaacfb2d0112c9c8ef27d1b6fb0628fc613f14` on `codex/pr523-workflow-call-counts`.
- [Implementation-only diff, excluding this handoff](https://github.com/bleedinghands/semantic-conventions-genai/compare/ed599428067e89cfa7c0fe07df7853be1dbfd419...1faaacfb2d0112c9c8ef27d1b6fb0628fc613f14).

The fixes touch 18 files: 280 additions and 267 deletions. The upstream PR has not
been updated, and Copilot has not reassessed these fixes on that PR.

## Intent and scope

The original PR adds `gen_ai.invoke_workflow.inference_calls` and
`gen_ai.invoke_workflow.tool_calls`: histograms recording call totals per workflow
execution. Nested executions contribute to their own and their enclosing
workflow's totals; summing across workflow names can therefore double count.

Keep that proposal small. The fixes do **not** change metric names, units, buckets,
registry definitions, or the recommended `gen_ai.workflow.name` dimension. They
do not add hierarchy attributes, redesign agent metrics, or introduce a shared
instrumentation framework. The original metric documentation, reporting-model
registration, and changelog entry are retained.

## Changes and why

### 1. Count workflow calls before execution and record totals on failure

The affected workflow examples now increment counters at SDK model/tool start
boundaries and record both histograms in `finally`, including zero values.
Counting returned responses or events with usage metadata misses calls that fail
without a response. Recording only after a successful workflow loses the entire
measurement when execution raises. Exceptions still propagate normally.

These are framework-visible call counts, not a claim to count every hidden HTTP
retry inside an underlying client. That distinction remains worth checking in
review.

### 2. OpenAI Agents: simplify the example and fix handoff counting

In [the scenario](reference/scenarios/openai-agents/scenario.py):

- Use `RunHooks.on_llm_start` instead of `len(result.raw_responses)` for workflow
  inference counts, and `on_tool_start` for local tool execution.
- Wrap the SDK handoff object's `on_invoke_handoff` before delegation. Handoffs
  bypass the local-tool hook, and counting only successful handoffs would miss
  failures.
- Remove the response-item classification helper and provider-hosted tool-type
  exclusion list. Counting local execution avoids maintaining a potentially
  incomplete list of hosted item types.
- Remove the PR-added nested-agent example and agent-metric emissions. A single
  agent inside another agent's tool was a weak demonstration of a distinct
  workflow boundary. Its shared `inner_result` also retained only the last inner
  run. Nested-workflow coverage now lives in LangGraph instead.

The existing single-agent/tool example and two-agent handoff workflow remain.
Removing the PR-added agent metrics keeps this change focused on workflow metrics;
it does not remove agent metric definitions from the conventions.

### 3. Google ADK: retain coverage with a real workflow

In [the scenario](reference/scenarios/google-adk/scenario.py), add
`run_workflow_reference`: a `SequentialAgent` runs a researcher with a weather tool,
then a writer. Both agents use `before_model_callback`; the researcher also uses
`before_tool_callback`. Totals are scoped to the runner invocation and use the
workflow name supplied to ADK.

This replaces the PR's workflow-metric recordings based on the old single-agent
run's usage-event counters. It demonstrates a library-owned multi-agent workflow
and does not depend on response usage metadata. Existing agent/memory examples
and their pre-existing agent counters are deliberately not refactored here.

### 4. CrewAI: cover the existing workflow

In [the scenario](reference/scenarios/crewai/scenario.py), register before-model
and before-tool hooks around `Crew.kickoff`. Filter hook events to the current
crew, then unregister the hooks and record totals in `finally`.

CrewAI already had a workflow that could emit both metrics. Adding coverage there
closes the omission without introducing another scenario; filtering and cleanup
keep these counters from including unrelated crew activity.

### 5. LangChain/LangGraph: cover workflows and repeated nesting

In [the scenario](reference/scenarios/langchain/scenario.py), use an
`AsyncCallbackHandler` for model/tool starts. A compiled outer `Weather graph`
invokes the compiled `Weather research` subgraph twice. Each invocation gets its
own counter; merged callback configuration preserves the outer counter too.

This gives nesting concrete graph-invocation boundaries and demonstrates that
repeated inner executions are accumulated, not overwritten. Workflow names come
from the compiled graphs. Each inner invocation and the outer invocation record
their own totals, including on failure. A graph-node invocation is not itself
counted as a tool call.

### 6. Documentation, coverage data, reports, and tests

- Rewrite [the nested-workflow explanation](docs/gen-ai/non-normative/examples-workflow-call-counts.md)
  to match the runnable LangGraph example. Explain overlapping totals and remove
  the speculative claim that a future `gen_ai.main_agent.name` would identify
  nesting. These metrics currently have no nesting discriminator.
- Update all four scenario READMEs to describe the actual callback mechanisms,
  workflow boundaries, and failure behavior.
- Refresh CrewAI, LangChain, and OpenAI Agents `data.json`, the four call-count
  coverage reports, and the reference README index. Workflow coverage now lists
  all four frameworks; the removed OpenAI agent-metric additions no longer appear.
  ADK's committed coverage metadata already contained the workflow metrics, so
  its `data.json` did not need a diff despite the changed runtime example.
- Update [the metric tests](reference/tests/test_metrics.py) to require both
  workflow metrics and `gen_ai.workflow.name` across all four frameworks. Remove
  the requirement for the now-removed OpenAI agent-metric additions.

## Copilot comments and disposition

The original review contains three inline comments and one related suppressed
comment in its summary. All four concerns were accepted:

1. [Missing CrewAI/LangChain inference coverage](https://github.com/open-telemetry/semantic-conventions-genai/pull/523#discussion_r4034517579):
   added workflow-scoped start callbacks in both, refreshed committed coverage,
   and made the metadata test require all four frameworks.
2. [ADK counts only events with usage metadata](https://github.com/open-telemetry/semantic-conventions-genai/pull/523#discussion_r4034517627):
   moved the new workflow metrics to the `SequentialAgent` example, counting
   model starts before a response exists and recording totals on failure.
   The older agent-metric implementation remains outside this fix's scope.
3. [OpenAI `raw_responses` omits failed calls](https://github.com/open-telemetry/semantic-conventions-genai/pull/523#discussion_r4034517654):
   replaced workflow result-length counting with start hooks and `finally`.
   Other PR-added result-based recordings were removed with the unnecessary
   agent/nested-example additions, rather than left using the flawed approach.
4. [Missing CrewAI/LangChain tool coverage (review summary)](https://github.com/open-telemetry/semantic-conventions-genai/pull/523#pullrequestreview-5232817922):
   added tool-start counting alongside inference counting in both frameworks,
   including refreshed data/reports and assertions for both metrics.

These are implementation responses, not resolved upstream review threads.
Copilot did not request the ADK multi-agent restructuring or the nesting-example
replacement; those are additional review fixes explained above.

## Validation already performed

On implementation commit `1faaacf`, before adding this handoff:

- All 28 reference scenarios completed with `uv run run-scenario --all --keep-going`.
- The reference unit suite passed: 9 tests.
- Ruff lint and formatting checks passed; `git diff --check` passed.
- `make generate-all` and `make check-policies` completed successfully.
- Existing conformance warnings remain. The committed `findings` for all 28
  scenarios were compared with the original PR and were unchanged; passing the
  scenario command does not mean the repository is warning-free.

Exported histogram values were also checked, not just metric presence:

| Workflow execution | Inference calls | Tool calls |
| --- | --- | --- |
| OpenAI Agents handoff workflow | 2 | 1 |
| ADK `weather_report` | 3 | 1 |
| CrewAI crew | 2 | 1 |
| LangGraph `Weather research`, each of two invocations | 2 | 1 |
| LangGraph `Weather graph`, enclosing both | 4 | 2 |

The LangGraph run makes 4 model calls and 2 tool calls, while sums across inner
and outer workflow series are 8 and 4. This is intentional overlapping scope.

Additional local probes used the pinned SDKs, mocked model/tool boundaries, and
an in-memory metric reader. They checked failed model attempts and zero-call
failures in all four frameworks; a failed OpenAI handoff; an ADK tool failure
without usage metadata; and failed LangGraph calls counted in both nested scopes.
CrewAI's exercised retry path produced three model-start counts.

Those failure probes were temporary local scripts, **not committed regression
tests**. The checked-in metric tests validate coverage metadata, not exact runtime
counts or all failure paths. A reviewer should not treat them as equivalent.

For a fresh check, run these from the indicated directories:

```sh
# Repository root
make generate-all
make check-policies

# reference/ (after installing its development dependencies)
uv run run-scenario --all --keep-going
uv run --extra dev pytest -q
uv run --extra dev ruff check src scenarios tests
uv run --extra dev ruff format --check src scenarios
```

The main remaining review questions are whether the framework-level callback
boundaries match the intended definition of a call, whether the nested graph
example is sufficiently small, and whether permanent runtime regression tests
are needed in this PR or should be a focused follow-up. No claim is made that
these reference examples are production-ready instrumentation for every SDK path.
