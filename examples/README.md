# Examples

This directory contains sample artifacts illustrating the pipeline's input/output format.

## Files

| File | Description |
|------|-------------|
| `sample_input_test_cases.csv` | Example CSV output from `qa-test-case-generator` (Gate 1 output) |
| `sample_output_testcase.yaml` | Example Maestro atomic testcase YAML (Phase 3 output) |
| `sample_output_scenario.yaml` | Example Maestro scenario YAML orchestrating testcases (Phase 3 output) |

## Pipeline Walkthrough

```
1. User provides: Jira ticket QON-1234 (Login feature)
       ↓
2. qa-test-case-generator → sample_input_test_cases.csv
       ↓
3. 🚦 GATE 1: Human reviews CSV, approves
       ↓
4. maestro-script-creator reads CSV, produces mapping table
       ↓
5. 🚦 GATE 2: Human confirms mapping table
       ↓
6. maestro-testcase-writer → sample_output_testcase.yaml (×N files)
   maestro-scenario-composer → sample_output_scenario.yaml
       ↓
7. Scripts are generated and execution starts automatically
       ↓
8. mcp_maestro_mcp_run executes tests
       ↓
9. maestro-script-debugger auto-fixes failures (self-healing)
```
