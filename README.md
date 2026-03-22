# NeuralRelay

Structured handoff protocols for multi-agent development. Designed to reduce the tokens agents waste re-discovering what the previous agent already knew.

## Quick Start

```bash
# Initialize in your project
/neuralrelay init

# Run a full pipeline (4 stages: BA → Dev → Test → Review)
/neuralrelay start "Add OAuth2 login with Google and GitHub providers"

# Or run quick mode for simple tasks (2 stages: Dev → Test, ~50% cheaper)
/neuralrelay quick "Fix the off-by-one in the history command"
```

## Documentation

- [Getting Started](docs/getting-started.md) — Installation, first run, cost expectations
- [How It Works](docs/how-it-works.md) — The problem, the pipeline, the key insight
- [Handoff Protocol](docs/handoff-protocol.md) — Deep dive into structured handoffs
- [Pipeline Configuration](docs/pipeline-configuration.md) — Customizing your pipeline, templates, custom agents
- [Commands Reference](docs/commands-reference.md) — All slash commands with examples
- [Agents Reference](docs/agents-reference.md) — What each agent does and its limitations
- [Benefits](docs/benefits.md) — What NeuralRelay does for you, with honest caveats
- [Project Setup](docs/project-setup.md) — Making NeuralRelay work well with your project
- [When to Use](docs/when-to-use.md) — When NeuralRelay helps and when it doesn't
- [Troubleshooting](docs/troubleshooting.md) — Common issues and solutions
- [FAQ](docs/faq.md) — Honest answers to common questions

## The Problem

Multi-agent AI development has a dirty secret: agents waste a significant portion of their tokens re-discovering context that previous agents already established. Nicholas Carlini documented building a C compiler with 16 Claude agents for ~$20K, with significant costs attributed to agent re-orientation and coordination overhead ([source](https://nicholas.carlini.com/writing/2025/how-i-use-ai.html)). Analysis estimated 22-44% of multi-agent project costs are attributable to agent amnesia and coordination overhead ([source](https://medium.com/@mrsandelin/your-ai-agent-teams-are-burning-money-heres-the-math-939e3b3b9d88)).

The problem compounds when you add quality assurance. CodeRabbit's analysis found that AI-generated code contains more bugs than human-written code, with significantly more logic and correctness errors ([source](https://www.coderabbit.ai/blog/ai-code-review-insights)). One major reason: the same agent that writes code also reviews it, inheriting the same blind spots and assumptions. Self-grading doesn't work — you need independent verification with structured information transfer.

Current multi-agent tools (Agent Teams, /feature-dev, raw agent spawning) solve orchestration but not information loss. They manage *who runs when*, but not *what they know*. Each agent starts with a blank slate and burns tokens rebuilding context from scratch.

## How NeuralRelay Works

When you run `/neuralrelay start "Add OAuth2 login"`, NeuralRelay orchestrates a 4-stage pipeline where each agent reads the previous agent's typed handoff instead of re-exploring the codebase:

```
Task Description
       │
       ▼
┌──────────────┐     01-ba-agent.json
│   BA Agent   │────────────────────────────►
│   (sonnet)   │  acceptance_criteria,
└──────────────┘  affected_modules, glossary
                           │
                           ▼
                  ┌──────────────┐     02-dev-agent.json
                  │  Dev Agent   │────────────────────────►
                  │   (opus)     │  coverage_gaps,
                  └──────────────┘  properties_believed
                                           │
                                           ▼
                                  ┌──────────────┐     03-test-agent.json
                                  │ Test Agent   │────────────────────────►
                                  │  (sonnet)    │  property_verification,
                                  └──────────────┘  bugs_found
                                                           │
                                                           ▼
                                                  ┌──────────────┐
                                                  │Review Agent  │──► 04-review-agent.json
                                                  │   (opus)     │    verdict, quality_score
                                                  └──────────────┘
```

The key insight is the **dev-to-test handoff**. Here's what it looks like:

```json
{
  "handoff_type": "dev_to_test",
  "coverage_gaps": [
    "Race condition when two concurrent refreshAccessToken calls execute for the same user — both read the same refresh token, provider invalidates on first use, second call fails silently",
    "Token encryption key rotation — no test for reading tokens encrypted with a previous key",
    "Email case sensitivity in profile linking: 'User@Gmail.com' vs 'user@gmail.com'"
  ],
  "properties_believed": [
    {
      "property": "validateOAuthCallback never throws for any input",
      "confidence": "high",
      "evidence": "try-catch wraps entire body; 12 unit tests with null/undefined/empty inputs"
    },
    {
      "property": "refreshAccessToken is safe for concurrent calls with different users",
      "confidence": "medium",
      "evidence": "No shared mutable state per-user, but no concurrent test exists"
    },
    {
      "property": "State parameter is cryptographically random and single-use",
      "confidence": "low",
      "evidence": "crypto.randomBytes(32) used, but 5-minute expiry window allows theoretical replay"
    }
  ]
}
```

Instead of the test agent spending thousands of tokens reading every file and guessing where bugs might hide, it reads this handoff and immediately knows:
- **What to test first**: the concurrent token refresh race condition (coverage gap)
- **What to try to break**: the medium/low confidence properties
- **What to skip**: the 12 input-validation tests the developer already wrote

## The Three Innovations

### 1. Structured Handoff Protocol (SHP)

Typed JSON contracts between pipeline stages. Each schema enforces required fields with descriptions, preventing the information loss that happens with unstructured agent-to-agent communication. Four handoff types cover the full pipeline: requirements, dev-to-test, test-to-review, and review-final.

### 2. Priority-Ordered Testing

The test agent follows a strict priority order:

1. **Coverage gaps** from the dev handoff (known unknowns — highest value)
2. **Low/medium confidence properties** (try to falsify them)
3. **Uncovered acceptance criteria** from the BA
4. **Known risks** and edge cases

This ordering ensures every test token goes toward discovering *new* information, not re-confirming what the developer already tested.

### 3. Handoff Chain Quality Scoring

The review agent doesn't just review the code — it rates the handoff chain itself:

```json
{
  "chain_quality": {
    "requirements_clarity": 9,
    "dev_handoff_completeness": 8,
    "test_coverage_adequacy": 8
  }
}
```

This meta-feedback improves the system over time. If the dev agent consistently gets low `dev_handoff_completeness` scores, the `/neuralrelay report` command flags it: "Dev handoff scores averaging 4/10 — coverage_gaps may be too vague."

## Configuration

NeuralRelay uses `neuralrelay.yaml` in your project root:

```yaml
pipeline:
  - name: requirements
    agent: ba-agent
    model_preference: sonnet
  - name: implementation
    agent: dev-agent
    model_preference: opus
  - name: testing
    agent: test-agent
    model_preference: sonnet
  - name: review
    agent: review-agent
    model_preference: opus

settings:
  max_rework_cycles: 2        # Retry loops when reviewer requests changes
  handoff_validation: strict   # Validate all handoffs against schemas
  handoff_dir: .neuralrelay/handoffs
  token_tracking: true
```

Customize by adding, removing, or reordering stages. The handoff protocol adapts to any topology.

## Pipeline Templates

NeuralRelay ships with 3 pre-built pipeline configurations in `examples/templates/`:

### Startup Fast (`startup-fast.yaml`)
```
Dev (sonnet) → Test (sonnet)
```
2 stages, both sonnet, 1 rework cycle. For solo devs and startups who want speed. ~50% cheaper than the full pipeline.

### Enterprise Secure (`enterprise-secure.yaml`)
```
BA → Dev → Security Review (opus) → Test → Review
```
5 stages with a dedicated security agent between dev and test. Use for auth, payments, PII, or any public-facing API changes.

### TDD Strict (`tdd-strict.yaml`)
```
BA → Test (specs) → Dev → Test (verify) → Review
```
5 stages — tests written BEFORE code. The first test stage writes test specifications from requirements, the dev implements to pass them, the second test stage verifies. For high-reliability features.

To use a template, copy it to your project root:
```bash
cp examples/templates/enterprise-secure.yaml neuralrelay.yaml
```

## Adding Custom Agents

Create `agents/your-agent.md` following the standard structure (role, responsibilities, input/output handoffs, workflow, quality standards, edge cases), then add the stage to `neuralrelay.yaml`. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full template and requirements.

## Comparison

| Feature | NeuralRelay | Raw Agent Teams | /feature-dev | Manual Multi-Agent |
|---|---|---|---|---|
| Structured Handoffs | Typed JSON schemas | None | None | Ad-hoc |
| Priority Testing | Coverage-gap-first | N/A | Basic | Manual |
| Token Tracking | Per-agent per-run | None | None | Manual |
| Pipeline Config | YAML, any topology | Hardcoded | Fixed | Custom scripts |
| Quality Scoring | Code + chain quality | None | None | Manual review |
| Rework Loops | Automated (max 2) | None | None | Manual |
| Handoff Validation | Schema-enforced | None | None | None |

## Full vs Quick Mode

| | Full Mode | Quick Mode |
|---|---|---|
| **Command** | `/neuralrelay start "task"` | `/neuralrelay quick "task"` |
| **Stages** | BA → Dev → Test → Review | Dev → Test |
| **Best for** | New features, complex changes, security-sensitive code, unfamiliar codebase areas | Bug fixes, small features, refactors, tasks with clear requirements |
| **Token cost** | Full | ~30-50% less |
| **Output** | Review verdict + quality score | Test results (no review) |
| **Rework loops** | Yes (max 2) | No |

## Commands

| Command | Description |
|---|---|
| `/neuralrelay init` | Initialize NeuralRelay in the current project |
| `/neuralrelay start "task"` | Run a full 4-stage pipeline for the given task |
| `/neuralrelay quick "task"` | Run a lightweight 2-stage pipeline (dev + test) |
| `/neuralrelay status` | Show current pipeline status and token usage |
| `/neuralrelay chain [id]` | Visualize a handoff chain in human-readable format |
| `/neuralrelay report` | Generate cost and quality analytics across runs |

## Limitations

- **Not cost-effective for trivial tasks.** The pipeline overhead (spawning multiple agents, writing handoffs) exceeds the benefit for typo fixes, config changes, or single-line edits. Use Claude Code directly for those.
- **Savings are estimated, not independently benchmarked.** The 30-50% figure is an architectural projection based on how handoffs reduce redundant exploration. Actual savings depend on task complexity, codebase size, and handoff quality.
- **Quality depends on model compliance.** Agents are instructed to write honest coverage gaps and follow priority ordering, but model behavior is non-deterministic. Agents occasionally produce vague gaps or ignore the handoff protocol. Opus follows instructions more reliably than Sonnet.
- **Full pipeline takes 10-25 minutes and costs $2-8.** This is a time and cost investment. Quick mode is faster ($1-3, 5-12 minutes) but has fewer stages.
- **Does not guarantee finding all bugs.** NeuralRelay improves bug detection by targeting known gaps, but cannot prove correctness or catch all issues.

## Roadmap

- **v1.2**: Lifecycle-Aware Process Monitoring (LAPM) — health checks, stall detection, automatic recovery
- **v1.3**: Adaptive Cost Router — dynamically select models per stage based on task complexity and budget
- **v2.0**: Team analytics dashboard — cross-project quality trends, agent performance benchmarks, cost optimization recommendations

## Contributing

Contributions welcome — new agents, pipeline templates, and schema improvements. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License. See [LICENSE](LICENSE) for details.
