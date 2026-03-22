# When to Use NeuralRelay (and When Not To)

## When NeuralRelay Helps

**New feature development** where multiple concerns (code, tests, review) benefit from structured communication. The handoff protocol is most valuable when there's significant context to transfer between agents — feature scope, implementation decisions, untested edge cases.

**Complex changes touching multiple files or modules.** The more files involved, the more context the test agent would waste re-discovering. A 9-file OAuth implementation saves more through handoffs than a 1-file utility function.

**Tasks where you want an audit trail of decisions.** The handoff chain is a structured record of: what requirements were defined, what was built, what was tested, what gaps remain, what the reviewer found. This is useful for compliance, team reviews, or understanding past decisions.

**Teams standardizing their multi-agent workflow.** Instead of each developer running agents ad-hoc with different prompts, NeuralRelay provides a repeatable process. The pipeline config is committed to the repo, so everyone runs the same workflow.

**Any task where you'd normally run 3+ agent passes anyway.** If you were going to run a dev agent, then a test agent, then a review agent separately, NeuralRelay makes those passes talk to each other instead of each starting from scratch.

## When NeuralRelay Is Overkill

**Typo fixes, config changes, single-line edits.** The pipeline overhead (spawning 4 agents, writing handoff JSON, validating schemas) costs more than just making the change directly. Use Claude Code without NeuralRelay.

**Tasks where you already know exactly what to do and just need code written.** If you can describe the exact code change in a sentence, skip the BA stage at minimum. Consider quick mode or no pipeline at all.

**Exploratory coding where requirements aren't clear yet.** NeuralRelay assumes you can describe the task upfront. If you're still figuring out what to build — prototyping, spiking, investigating — the structured requirements stage will slow you down. Explore first, then use NeuralRelay when you know what you want.

**Very small projects with fewer than ~500 lines of code.** In small codebases, agents can read everything quickly. The overhead of structured handoffs exceeds the benefit of avoiding re-exploration, because re-exploration is cheap.

**Tasks that are faster to do yourself than to describe.** If writing a clear task description takes longer than writing the code, skip the pipeline.

## When NeuralRelay Is the Wrong Tool Entirely

**You're not using multi-agent workflows.** NeuralRelay is about agent-to-agent communication. If you run a single Claude Code agent for everything, NeuralRelay adds overhead with no benefit. It only helps when multiple agents need to share context.

**Your task requires real-time interaction.** NeuralRelay runs a fully automated pipeline — you launch it and wait for results. If your task requires manual QA, user testing, interactive debugging, or back-and-forth conversation, the pipeline model doesn't fit.

**You need guaranteed correctness.** NeuralRelay improves bug detection by directing testing toward known coverage gaps and unverified properties. This catches more bugs than undirected testing. But it cannot find all bugs, prove correctness, or replace formal verification. If you need mathematical guarantees, NeuralRelay is not the right tool.

**You need CI/CD integration.** NeuralRelay runs inside interactive Claude Code sessions. It cannot be triggered from a CI pipeline, a GitHub Action, or an automated build system. CI/CD integration is on the roadmap but not available in the current version.

**Your security requirements demand certified tooling.** The security agent performs useful static analysis, but it is not a certified security scanner. It cannot replace tools like Snyk, Semgrep, or a professional penetration test. Use it as an additional layer, not a replacement.

## Quick Mode vs Full Pipeline

| | Full Pipeline | Quick Mode |
|---|---|---|
| **Command** | `/neuralrelay start "task"` | `/neuralrelay quick "task"` |
| **Stages** | BA -> Dev -> Test -> Review | Dev -> Test |
| **Best for** | New features, complex changes, security-sensitive code, unfamiliar codebase areas | Bug fixes, small features, refactors, tasks with clear requirements |
| **Token cost** | Higher (4 agents) | Lower (2 agents, ~30-50% less) |
| **Output** | Review verdict + quality score + chain quality | Test results only (no review verdict) |
| **Rework loops** | Yes (max 2 by default) | No |
| **Requirements** | Independent BA analysis | You provide requirements in the task description |
| **Audit trail** | Complete (4 handoffs) | Partial (2 handoffs) |

**Rule of thumb:** If you'd feel comfortable merging the code with just tests and no code review, use quick mode. If you'd want a reviewer to look at it, use the full pipeline.

## Decision Flowchart

1. **Is this a trivial change?** (typo, config, rename) -> Skip NeuralRelay, use Claude Code directly
2. **Do you already know the exact fix?** -> Consider quick mode or no pipeline
3. **Is this exploratory?** (prototyping, investigating) -> Skip NeuralRelay, explore first
4. **Is this a bug fix or small feature with clear requirements?** -> `/neuralrelay quick`
5. **Is this a new feature, complex change, or security-sensitive?** -> `/neuralrelay start`
6. **Does this touch auth, payments, or PII?** -> Use the enterprise-secure template with the security agent
