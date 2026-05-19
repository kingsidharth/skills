# Analysis

## Eval Logs

Every `inspect eval` run writes a JSON log file containing full eval metadata, sample-level results, scores, and model interactions.

Default location: `./logs/` (configurable via `INSPECT_LOG_DIR` or `--log-dir`).

### Log Viewer

```bash
inspect view                          # latest log
inspect view --log-dir ./logs         # browse directory
inspect view --log ./logs/specific.json
```

Web-based UI showing: task summary, per-sample results, message transcripts, scores, token usage.

### CLI Log Commands

```bash
inspect log list                          # list recent logs
inspect log list --status error           # filter by status
inspect log read ./logs/eval.json         # print log summary
```

### Programmatic Access

```python
from inspect_ai.log import read_eval_log, list_eval_logs

# Read a single log
log = read_eval_log("./logs/eval.json")
print(log.status)       # "success" | "error" | "cancelled"
print(log.results)      # scores, metrics
for sample in log.samples:
    print(sample.score)

# List logs
logs = list_eval_logs("./logs/")
```

## Log Dataframes

Convert logs to pandas DataFrames for analysis:

```python
from inspect_ai.log import read_eval_log

log = read_eval_log("./logs/eval.json")
df = log.as_dataframe()
```

Columns include: sample ID, input, target, output, score value, metadata fields, token counts, timestamps.

### Cross-eval Comparison

```python
from inspect_ai.log import list_eval_logs, read_eval_log

logs = [read_eval_log(l.name) for l in list_eval_logs("./logs/")]
# Compare accuracy across models, prompts, etc.
```

## Eval Sets

Group related evals for systematic comparison:

```bash
inspect eval task.py --model openai/gpt-4o --model anthropic/claude-sonnet-4-0
```

Logs are tagged with eval set metadata for grouped analysis in the Log Viewer.
