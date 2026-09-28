# One candidate: optional review-lens selection

**Do not enable this replacement.** A real engine probe found no accepted Laya
replacement on the two public PRs examined. This is a development experiment,
not a production integration or evidence of end-to-end savings.

## Where this runs

Factory delegates to the shared Archon SDLC. At factory's pin
`24796870605b0fd576734b79a8d5c781f2b1c1e1`, delivery has a separate
`classify-review-scope` agent call, followed by `resolve-review-scope.py`.
The classifier chooses optional errors/docs reviews; code, seams, simplify and
tests are mandatory. The resolver applies an explicit errors override and
validates the selected value. It cannot establish that the review judgment is
correct. The docs flag passes directly to the review workflow.

The generator extracts the original classifier and resolver into an isolated
workflow. It removes the classifier's dependency on opening a new PR, because
we use an existing public PR. A second workflow runs Laya inside an actual
Archon code node, followed by the unchanged resolver, plus copies exercising
forced-on and forced-off settings. No review, implementation, publishing, or
merge nodes run. This is **a real workflow slice, not the complete factory
lifecycle**. No provider responses or resolver outputs were stubbed.

The local workflow deliberately passes low-confidence predictions to the
resolver **for observation only**. `requires_fallback` is a diagnostic, not a
production gate; the experiment never starts the original agent as a fallback.
An oversized input skips all resolver nodes. The workflow can complete while
rejecting a classification; completion alone is not a successful replacement.

## Observations

| Public input | Original agent | Laya typed checkpoint | Consequence |
| --- | --- | --- | --- |
| PR #11: README hash correction | errors=false, docs=true | errors=true, docs=true; selected probability 0.3592 | Unnecessary error review; below the preselected 0.9 threshold |
| PR #9: runtime credentials, recovery and restart changes | Not run | Full input rejected at the configured 1,024-token budget | Original classifier would still be required |

For PR #11 the original classifier used **30,168 input tokens**, including
**14,592 cached input tokens**, and **244 output tokens**, as reported by Archon.
This is one run with this operator's native Codex configuration, explicitly
binding medium to `gpt-6-astra` with high effort. It is not Cole's default model
cost, a billing estimate, or a representative average. The agent did its own
PR retrieval; Laya's code node retrieved the title, description and complete
diff through GitHub. No model summarized Laya's input.

Laya processed **502 encoder tokens** for PR #11. These tokens are not comparable
billing units to Codex tokens. The original resolver adopted Laya's incorrect
errors=true value under auto, while forced-on and forced-off worked. This proves
that the existing deterministic code handles overrides, not semantic errors.
The docs judgment is ambiguous under the original prompt's mechanical-change
exception; the agent's docs=true is a comparator, not established ground truth.
The absence of error-handling changes in the one-line README diff is direct
source evidence for the error-review false positive.

Both examined cases would require the original agent under the diagnostic policy.
**Accepted replacement coverage: 0/2. Verified avoided agent calls: zero.**
Fallback was not executed, so no combined fallback latency or cost is claimed.
A deployed version would add local processing before the original call on these
cases. Lowering the threshold to accept PR #11 would accept its false positive.
The threshold is uncalibrated and cannot guarantee safety at any value.

Other limitations:

- Laya provides labels and probabilities, not the original classifier's
  diff-specific reason sentences. This probe does not pretend to satisfy that
  explanation contract.
- The 1,024-token limit is this adapter's configured budget, not a claim about
  every Laya configuration. Input loss is refused rather than silently trimmed.
- Local setup requires Python, model weights and PyTorch. Inference runs on CPU
  with two threads, FP32 and a pinned checkpoint. Timings include retrieval and
  cached-weight loading; first-time downloads and engine setup are excluded.
- Two deliberately selected public cases are not a held-out quality evaluation.
  No real mandatory/optional review or delivery outcome was measured.
- No comparison with a smaller agent model, no throughput benchmark, and no
  measured deterministic-cache saving is established here.
- Observed engine runs used Bun 1.4.0; the pinned checkout declares 1.4.2.
  The exercised slices completed, but this is not full supported-runtime
  compatibility validation. Python, runner and engine versions are in the receipt.

The saved Laya reports are final replays after fixing the probe generator's
virtual-environment path handling. Earlier successful local runs gave the same
decisions. Setup/validation failures and development retries are not included in
the reported per-node timings; those timings are not the total cost of this
investigation.

## Reproduce

Use a disposable Archon clone at the exact pin above and install its dependencies
with the package manager/version specified by that checkout. Keep its runtime
state outside repositories with `ARCHON_HOME`. Keep the provider's native
configuration and authentication; do not replace its home directory.

Install `requirements-probe.txt` in an isolated Python environment (install the
CPU PyTorch wheel separately as described in README.md). In the factory clone:

```bash
/tmp/factory-laya/bin/python experiments/laya/prepare_review_probe.py \
  --archon /tmp/archon-review-probe --python /tmp/factory-laya/bin/python
```

This refuses a wrong Archon revision, changed source nodes, and existing output
folders. It creates two experiment folders only in the disposable Archon clone;
it installs nothing into a factory consumer. The generated local node refers to
this factory clone's `review_probe.py`, so keep both clones available.

Set `PATH` to include that environment's `uv` and your Bun executable. Set
`HF_HOME` to the model cache and `LAYA_CPU_AMP=fp32`; after the pinned weights
are cached, use `HF_HUB_OFFLINE=1` to exclude model downloads from timing.
GitHub reads still need network access and `gh` authentication. Check out the
public PR branch in another disposable factory clone for the baseline; the
original command discovers the PR for that branch.

Run `laya-review-baseline` through `archon workflow run` with
`--workflow-source /tmp/archon-review-probe --cwd /tmp/factory-pr-target
--no-worktree`. Explicitly bind medium to the model/effort being measured with
Archon's supported configuration. Record that binding in the result.

Run `laya-review-local` with the same source/cwd flags, plus:

```text
--input repo=coleam00/ai-software-factory --input pr=11
--input report=/tmp/laya-pr11-new.json
```

Repeat with PR 9 and a different report path. Every run's message must state the
problem, value, timing, desired outcome, invariants and acceptance. For example:
“Problem: optional review classification uses model tokens. Value: measure a local
substitute. Timing: before integration. Outcome: observe this public PR's review
scope. Invariants: read-only; no reviews, edits or publishing; predictions are
experimental. Acceptance: full input or explicit rejection, measured usage and
resolver outputs.” Do not use `--dry-run` as execution evidence.

The reports preserve the captured public input, checkpoint, prompt, package
versions, probabilities and timings. The reader checks base/head revisions and
PR text again after diff capture. Reports refuse overwrite and unexpected failures
fail the node. Initial downloads and all agent usage remain separate costs.
For engine evidence, inspect the run and transcript through `workflow get` and
retain only relevant public outputs and usage, not native configuration or
unrelated logs. `results/review-probe-engine.json` is a minimal receipt from the
observed runs. Its Laya reports contain full public input snapshots.

## Decision

Keep the existing classifier. This experiment supports testing candidates inside
Archon, but does not justify replacing this step with Laya. The next useful test
would need held-out examples whose complete evidence fits the local model, an
acceptable explanation contract, and measured quality and total cost including
fallbacks. Keeping every optional review could avoid missed reviews, but would
add review work; that is a separate deterministic alternative to measure, not a
proven Laya benefit.
