# Contributing to NeuralRelay

NeuralRelay grows through community contributions — new agents, pipeline templates, and schema improvements. Here's how to contribute.

## Adding a Custom Agent

1. **Create the agent file** at `agents/your-agent.md` following this structure:

   ```markdown
   ---
   model: sonnet    # or opus for high-reasoning tasks
   tools:
     - Read
     - Glob
     - Grep
     - Bash
   maxTurns: 20
   skills:
     - handoff-protocol
   ---

   You are the {Role} agent in the NeuralRelay pipeline. {2-3 sentence role definition.}

   ## Responsibilities
   ## Input
   ## Output
   ## Workflow
   ## Quality Standards
   ## Edge Cases
   ## When You're Stuck
   ```

2. **Define which handoff your agent reads and produces.** Every agent must:
   - Specify the input handoff schema and file path
   - Specify the output handoff schema and file path
   - Write output to `.neuralrelay/handoffs/{chain-id}/{NN}-{stage-name}.json`

3. **Add your agent to a pipeline config.** Either modify `neuralrelay.yaml` or create a new template in `examples/templates/`:

   ```yaml
   pipeline:
     - name: your-stage-name
       agent: your-agent
       model_preference: sonnet
   ```

4. **Test your agent** by running a pipeline with it:
   ```
   /neuralrelay start "a task that exercises your agent"
   ```

5. **Quality requirements:**
   - Agent file must be under 200 lines
   - Must include Edge Cases and When You're Stuck sections
   - Must use imperative language ("Do X", "Never Y")

## Creating a Pipeline Template

1. Create a YAML file in `examples/templates/` with a descriptive name (e.g., `api-development.yaml`).

2. Include a comment header explaining:
   - What the template is for
   - When to use it vs other templates
   - What trade-offs it makes

3. Only reference agents that exist in `agents/`. If your template needs a new agent, create it first.

4. Test the template by copying it to `neuralrelay.yaml` and running a pipeline.

## Improving Handoff Schemas

Adding a new field to a schema has a high bar. Every field must justify its existence:

1. **Which downstream agent uses this field?** If no agent reads it, it's dead weight.
2. **What tokens does it save?** A field that saves the test agent from re-exploring code is valuable. A field that's "nice to have" is not.
3. **Is it always available?** Fields that are empty or "N/A" most of the time add noise.

To propose a schema change:
1. Open an issue describing: the field, which agent produces it, which agent consumes it, and the token savings estimate.
2. If accepted, update the schema, all example handoffs, and any agent definitions that read or write the field.
3. Ensure all example handoffs still validate against the updated schema.

## Reporting Issues

When filing an issue, include:

1. **The task description** you used with `/neuralrelay start`
2. **The handoff chain** — run `/neuralrelay chain` and paste the output
3. **Which agent failed** and how (crash, invalid handoff, wrong output)
4. **Your pipeline config** — paste your `neuralrelay.yaml`
5. **Environment** — OS, Claude Code version

The handoff chain is the most useful diagnostic. It shows exactly what each agent produced and where the chain broke.

## Code of Conduct

- Be constructive. If something doesn't work, describe what happened and what you expected.
- Share real examples. Concrete handoff chains are more useful than abstract descriptions.
- Respect everyone's time. Keep issues focused and PRs small.
- Test your contributions. Every schema change needs updated examples. Every agent needs a pipeline run.
