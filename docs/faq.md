# Frequently Asked Questions

## Does NeuralRelay replace my existing testing workflow?

No. NeuralRelay adds a structured communication layer between agents. Your existing tests, linters, CI/CD pipeline, and type checker remain unchanged. The test agent writes additional tests targeting specific coverage gaps identified by the dev agent — it doesn't replace your test suite. After the pipeline runs, you still have your normal workflow for merging, deploying, and monitoring.

## How much does it cost to run a pipeline?

Typical full pipeline (4 stages): $2-8 per run. Quick mode (2 stages): $1-3 per run. These vary significantly with task complexity and codebase size.

NeuralRelay adds slight overhead for writing and reading structured handoff JSON, but is designed to reduce the larger overhead of agents re-exploring the codebase. The net effect depends on task complexity. For medium-complexity features, the estimated reduction in redundant token spend is 30-50% compared to running the same agents without handoffs. This estimate is based on architectural analysis, not independent benchmarking — actual savings vary. For trivial tasks, the pipeline overhead may cost more than it saves — use Claude Code directly for those.

## Can I use NeuralRelay with Sonnet-only to save costs?

Yes. Change all `model_preference` values in `neuralrelay.yaml` to `sonnet`, or use the `startup-fast.yaml` template which uses Sonnet for all stages. The pipeline still works — schemas are still validated, handoffs are still written, priority ordering is still followed.

The trade-off: coverage_gaps from the dev stage will be less specific, and review depth will decrease. Opus is notably better at self-reflection ("here's what I didn't test") and nuanced quality assessment. For cost-sensitive workflows, using Sonnet for BA and test stages while keeping Opus for dev and review is a reasonable middle ground.

## Does NeuralRelay work with languages other than JavaScript/TypeScript?

Yes. The handoff protocol is language-agnostic. It describes files, functions, properties, and risks — not language-specific constructs. The schemas don't reference any particular language, framework, or testing tool. NeuralRelay works with any language that Claude Code supports, which includes Python, Go, Rust, Java, C++, Ruby, and many others.

The example handoffs in the repo happen to use TypeScript because the example task (OAuth implementation) is in a Next.js project, but the protocol itself is language-neutral.

## Can I add my own agents to the pipeline?

Yes. Create a markdown file in `agents/`, define what it reads and produces, and add it to `neuralrelay.yaml`. See [Pipeline Configuration](pipeline-configuration.md#creating-custom-agents) for a step-by-step guide.

Custom agents can reuse existing handoff schemas (e.g., a compliance agent could produce a `test-to-review` type handoff) or you can create new schemas for specialized output.

## What happens if an agent fails mid-pipeline?

The process manager saves whatever handoffs have been completed. Completed stages are preserved — only the failed stage and subsequent stages are affected.

You can see partial progress with `/neuralrelay chain`. The completed handoffs contain useful information even if the pipeline didn't finish. You can use that context to debug the failure, fix the issue, or re-run the pipeline.

The process manager does not automatically retry failed agents (that's different from rework loops, which are triggered by the reviewer). If an agent crashes, investigate why rather than re-running blindly.

## Is NeuralRelay better than just using Agent Teams?

Different tools for different problems. Agent Teams give you parallel execution with peer-to-peer messaging — useful when you want multiple agents investigating different approaches simultaneously. NeuralRelay gives you sequential execution with structured handoff contracts — useful when you want a defined process where each stage builds on the previous one.

NeuralRelay is better when you want: requirements analysis -> implementation -> testing -> review, in that order, with each stage informed by the previous one's output.

Agent Teams are better when you want: 3 agents exploring 3 different implementation approaches in parallel, then picking the best one.

They can be complementary — you could use Agent Teams within a NeuralRelay stage (e.g., the dev agent spawns sub-agents for parallel implementation of independent modules).

## Does NeuralRelay guarantee it will find all bugs?

No. NeuralRelay improves bug detection by directing testing toward known coverage gaps and unverified properties. This structured approach catches more bugs than undirected testing because the test agent knows where to look. But it cannot find all bugs.

The quality depends on two factors: how honestly the dev agent reports its gaps (Opus is better at this), and how thoroughly the test agent pursues them. Some bugs only manifest in production environments, under specific load, or through interaction patterns that no test anticipated.

## Can I use this in a CI/CD pipeline?

Not in the current version. NeuralRelay runs inside interactive Claude Code sessions and requires agent spawning capabilities. Automated CI/CD integration (triggering pipelines from PRs, running in headless mode) is on the roadmap but not yet available.

## How is this different from other multi-agent approaches?

Most multi-agent tools solve orchestration — managing who runs when. NeuralRelay solves information transfer — managing what each agent knows.

The difference is the handoff protocol. When a dev agent finishes in a typical multi-agent setup, the next agent starts from scratch, re-reading the codebase to understand what changed. With NeuralRelay, the next agent reads a structured JSON contract that tells it exactly what was built, what was tested, what wasn't tested, and where the developer thinks bugs might be.

The specific innovation is the `coverage_gaps` and `properties_believed` fields in the dev-to-test handoff. These give the test agent actionable targets instead of making it guess. The review agent's `chain_quality` scoring then creates a feedback loop that improves handoff quality over time.

## What if I disagree with the review agent's verdict?

The verdict is advisory, not binding. NeuralRelay doesn't enforce the verdict — it doesn't block merges, delete code, or prevent you from shipping. You can read the review, disagree with specific findings, and proceed as you see fit.

If the reviewer is consistently too strict or too lenient for your needs, you can: adjust the pipeline (remove the review stage, switch the review model), modify the review agent's instructions in `agents/review-agent.md`, or use quick mode which has no review.

## Is the handoff data sensitive? Should I worry about it?

Handoff files contain structured information about your code: file paths, function signatures, test results, and coverage assessments. They don't contain your actual source code, but they do describe its structure.

By default, `.neuralrelay/handoffs/` is added to `.gitignore` during init, so handoff data stays local and is not committed to your repository. If you're working on sensitive projects, verify this gitignore entry is in place. The data is stored as plain JSON files on your local filesystem — it's not sent to any external service beyond the Claude API calls that power the agents.
