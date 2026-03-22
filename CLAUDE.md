# NeuralRelay — Process Intelligence Plugin for Claude Code

## Project Overview
NeuralRelay is a Claude Code plugin (v1.2.0) that provides structured handoff protocols and configurable agent pipelines for multi-agent development workflows. It is designed to reduce redundant token spend (estimated 30-50% for medium-to-complex tasks) by passing precise context between pipeline stages instead of agents re-discovering it from scratch. Actual savings depend on task complexity, codebase size, and handoff quality.

## What This Is
A Claude Code plugin that ships as agents + commands + skills + schemas — pure markdown and JSON, no runtime dependencies, no external services, no build step.

## Directory Structure
```
neuralrelay/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── agents/                      # 6 pipeline agents
│   ├── ba-agent.md              # Business Analyst — produces requirements handoff
│   ├── dev-agent.md             # Senior Developer — produces dev-to-test handoff
│   ├── test-agent.md            # Test Engineer — produces test-to-review handoff
│   ├── review-agent.md          # Tech Lead Reviewer — produces final review handoff
│   ├── process-manager.md       # Pipeline orchestrator — manages flow + health
│   └── security-agent.md        # Security Reviewer — focused vulnerability audit
├── commands/                    # 6 slash commands
│   ├── init.md                  # /neuralrelay init — setup in a project
│   ├── start.md                 # /neuralrelay start "task" — run full pipeline
│   ├── quick.md                 # /neuralrelay quick "task" — 2-stage pipeline
│   ├── status.md                # /neuralrelay status — show pipeline state
│   ├── chain.md                 # /neuralrelay chain — visualize handoff chain
│   └── report.md                # /neuralrelay report — cost & quality analytics
├── skills/
│   └── handoff-protocol.md      # Auto-loaded skill — SHP schema knowledge
├── schemas/
│   ├── requirements.schema.json
│   ├── dev-to-test.schema.json
│   ├── test-to-review.schema.json
│   └── review-final.schema.json
├── examples/
│   ├── neuralrelay.yaml         # Default pipeline config
│   ├── sample-handoffs/         # Complete example handoff chain (OAuth2 feature)
│   └── templates/               # 3 pipeline templates (startup, enterprise, tdd)
├── docs/                        # 9 documentation files
├── tests/self-test/             # NeuralRelay runs on itself — expected handoff chain
├── CONTRIBUTING.md
├── README.md
├── LICENSE                      # MIT
└── CHANGELOG.md
```

## Core Concepts

### Structured Handoff Protocol (SHP)
Typed JSON contracts between pipeline stages. Each agent READS the previous handoff and PRODUCES its own. Key fields:
- BA → All: acceptance_criteria, technical_constraints, out_of_scope, domain_glossary
- Dev → Test: coverage_gaps, properties_believed (with confidence), known_risks
- Test → Review: property_verification (confirmed/falsified/inconclusive), bugs_found with counterexamples
- Review → Done: review_verdict, quality_score, handoff_chain_quality meta-assessment

### Pipeline-as-Config
Agent pipeline defined in neuralrelay.yaml. Users add/remove/reorder stages. Handoff protocol adapts to any topology.

### Process Manager
Orchestrates the pipeline, validates handoffs against schemas, tracks token usage, handles rework loops (max 2 cycles when reviewer requests changes).

## Quality Standards
- Every agent file under 200 lines — concise, no fluff
- Every schema field has a description and earns its place
- Every command works standalone (no hidden dependencies)
- Handoff schemas are JSON Schema draft-07 with required fields and additionalProperties: false
- All example handoffs validate against their schemas
- No external network calls — everything is local file I/O
- The plugin name in all commands and paths is "neuralrelay" (lowercase, no spaces)

## Agent Markdown Rules
- Start with 2-3 sentence role definition
- Include: Responsibilities, Input/Output, Workflow, Quality Standards, Edge Cases
- Use imperative language ("Do X", "Never Y", "Always Z")
- Specify what handoff the agent READS and what it PRODUCES
- All handoffs write to `.neuralrelay/handoffs/{chain-id}/{NN}-{agent-name}.json`
- Agents discover prior handoffs by reading chain directory and identifying handoff_type
