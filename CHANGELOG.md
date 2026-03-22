# Changelog

## [1.2.0] - 2026-03-22

### Added
- Comprehensive user documentation in docs/ directory (9 files)
- Getting started guide with installation, first run walkthrough, and cost expectations
- How-it-works conceptual guide with token usage comparison
- Handoff protocol deep-dive with field explanations and examples
- Pipeline configuration guide with customization scenarios and custom agent instructions
- Commands reference for all 6 slash commands
- Agents reference with capabilities and honest limitations for each agent
- When-to-use guide with clear guidance on when NeuralRelay helps and when it doesn't
- Troubleshooting guide covering 10 common issues
- FAQ with 12 honest Q&As
- Project setup guide: CLAUDE.md tips, task description examples, template selection
- Benefits documentation: mechanism-based explanations with honest caveats for each benefit
- Limitations section added to README.md (visible, not buried in FAQ)
- Documentation links section added to README.md

### Changed
- CLAUDE.md rewritten to reflect current project state (6 agents, 6 commands, full directory structure)
- Removed fabricated $schema URLs from plugin.json (no official schema URL exists)
- Removed marketplace.json (fabricated format — will use Anthropic's actual submission process)
- Replaced fabricated installation commands with reference to official Claude Code docs
- Author info updated with full name and GitHub username
- Plugin version bumped to 1.2.0

## [1.1.1] - 2026-03-22

### Fixed
- Security agent path conflict: switched from hardcoded filenames to sequence-numbered discovery ({NN}-{agent-name}.json), enabling any pipeline topology
- Schema/agent instruction mismatch: aligned coverage_gaps minimum to 3 everywhere (schema, agent instructions, skill)
- Rework feedback mechanism: defined explicit rework flow with rework-{N}.json files, dev agent reads issues_to_address for targeted fixes
- Quick mode integration gap: dev agent explicitly handles missing BA handoff, produces inline_requirements field, process manager passes raw task description
- Inconsistent token savings claims: harmonized all percentage claims to "30-50%" with source citations in README

### Changed
- All handoff files renamed from role-based (01-requirements.json) to agent-based (01-ba-agent.json) convention
- All agents now discover prior handoffs by reading chain directory and identifying handoff_type, not hardcoded paths
- Review agent reads ALL files in chain directory regardless of stage count
- dev-to-test schema: added optional inline_requirements field for quick mode
- plugin.json version bumped to 1.1.1

## [1.1.0] - 2026-03-22

### Added
- Self-test: NeuralRelay runs on itself — complete expected handoff chain for adding a /neuralrelay history command (tests/self-test/)
- Quick mode: /neuralrelay quick for lightweight 2-stage pipelines (dev + test, ~50% cheaper)
- Chain visualization: /neuralrelay chain for human-readable handoff chain summaries
- Security agent: dedicated security review agent for enterprise pipelines (agents/security-agent.md)
- 3 pipeline templates: startup-fast, enterprise-secure, tdd-strict (examples/templates/)
- CONTRIBUTING.md with guidelines for custom agents, templates, and schema improvements

### Changed
- All 5 agent definitions now include Edge Cases sections with explicit handling for failure modes
- Process manager supports "full" and "quick" pipeline modes
- README updated with quick mode comparison, pipeline templates, chain visualization, and contributing link
- plugin.json updated with new commands (quick, chain) and security agent

## [1.0.0] - 2026-03-22

### Added
- Structured Handoff Protocol with 4 typed schemas (requirements, dev-to-test, test-to-review, review-final)
- 5 pipeline agents: BA, Developer, Test Engineer, Tech Lead Reviewer, Process Manager
- 4 slash commands: /neuralrelay init, start, status, report
- Pipeline-as-config via neuralrelay.yaml
- Auto-validation of handoffs against JSON schemas
- Rework loops (max 2 cycles) when reviewer requests changes
- Token tracking per agent per pipeline run
- Handoff chain quality scoring (meta-feedback on process)
- Complete example handoff chain for OAuth2 implementation
