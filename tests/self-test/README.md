# NeuralRelay Self-Test

NeuralRelay eating its own dogfood. This test runs the full 4-stage pipeline on a real task: adding a `/neuralrelay history` command to NeuralRelay itself.

## Purpose

1. **Proves the protocol works** end-to-end on a real task, not a contrived example
2. **Provides realistic example data** — a feature built on this actual codebase
3. **Serves as a regression test** — if future changes break the protocol, this chain stops validating

## The Task

> Add a `/neuralrelay history` command that shows the last 5 pipeline runs with their quality scores and total token costs.

This task was chosen because it:
- Touches the NeuralRelay codebase itself (commands/, .neuralrelay/handoffs/)
- Is small enough to trace the full handoff chain mentally
- Has real edge cases (empty directory, fewer than 5 runs, corrupted handoff JSON)
- Produces a plausible bug the test agent would actually find

## How to Run

```bash
# 1. Initialize NeuralRelay in a test project (or this repo)
/neuralrelay init

# 2. Run the pipeline
/neuralrelay start "Add a /neuralrelay history command that shows the last 5 pipeline runs with their quality scores and total token costs"

# 3. Compare output chain against expected-handoffs/
/neuralrelay chain
```

## Expected Handoff Chain

The `expected-handoffs/` directory contains the full handoff chain that a well-functioning pipeline should produce for this task. Key things to verify:

### 01-ba-agent.json
- BA correctly identifies `commands/history.md` as a new file and `.neuralrelay/handoffs/` as the data source
- Acceptance criteria are testable (not "shows history nicely")
- Out-of-scope includes filtering, search, and export

### 02-dev-agent.json
- Dev honestly lists coverage gaps: "no test for corrupted JSON in handoff files", "behavior with >100 chains untested"
- properties_believed includes the sorting claim at medium confidence
- known_risks mentions the filesystem performance concern

### 03-test-agent.json
- Test agent FALSIFIES the "output includes all 5 runs when exactly 5 exist" property — reveals an off-by-one bug
- Counterexample is minimal: exactly 5 chain directories, output shows only 4
- Priority ordering is visible: coverage gaps tested first, then properties

### 04-review-agent.json
- Review agent catches the off-by-one and rates it medium severity
- chain_quality scores reflect the honest dev handoff (high completeness score)
- Recommendations include a process improvement for the plugin itself

## Validation

```bash
# Verify all expected handoffs match their schemas
python -c "
import json
pairs = [
    ('../../schemas/requirements.schema.json', 'expected-handoffs/01-ba-agent.json'),
    ('../../schemas/dev-to-test.schema.json', 'expected-handoffs/02-dev-agent.json'),
    ('../../schemas/test-to-review.schema.json', 'expected-handoffs/03-test-agent.json'),
    ('../../schemas/review-final.schema.json', 'expected-handoffs/04-review-agent.json'),
]
for schema_path, data_path in pairs:
    schema = json.load(open(schema_path))
    data = json.load(open(data_path))
    required = schema['required']
    missing = [r for r in required if r not in data]
    extra = set(data.keys()) - set(schema.get('properties', {}).keys())
    status = 'PASS' if not missing and not extra else 'FAIL'
    print(f'{status}: {data_path}')
    if missing: print(f'  Missing: {missing}')
    if extra: print(f'  Extra: {extra}')
"
```
