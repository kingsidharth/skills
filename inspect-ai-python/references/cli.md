# CLI Reference

## inspect eval

Run evaluations.

```bash
inspect eval <task_file> [options]
```

### Key Options

| Flag | Purpose |
|---|---|
| `--model` | Model to evaluate (e.g. `openai/gpt-4o`) |
| `-T key=value` | Task parameter override |
| `--task-config file.yaml` | Task parameters from file |
| `--solver` | Override solver |
| `--scorer` | Override scorer |
| `--limit N` | Evaluate only first N samples |
| `--sample-id id1,id2` | Evaluate specific sample IDs |
| `--epochs N` | Repeat each sample N times |
| `--log-dir path` | Output directory for logs |
| `--cache` | Enable response caching |
| `--batch` | Use provider batch API |
| `--interactive` | Step through samples |

### Model/Generation Options

| Flag | Purpose |
|---|---|
| `--temperature F` | Sampling temperature |
| `--max-tokens N` | Max tokens per generation |
| `--top-p F` | Nucleus sampling |
| `--max-retries N` | API call retries |
| `--max-connections N` | Concurrent API connections |

### Resource Limits

| Flag | Purpose |
|---|---|
| `--message-limit N` | Max messages per sample |
| `--token-limit N` | Max tokens per sample |
| `--time-limit N` | Max seconds per sample |
| `--max-samples N` | Concurrent samples |
| `--max-sandboxes N` | Concurrent containers |
| `--fail-on-error T/N` | Error tolerance |

## inspect view

Launch the Log Viewer web UI.

```bash
inspect view [--log-dir path] [--log file] [--port N]
```

## inspect log

Manage eval logs.

```bash
inspect log list [--log-dir path] [--status success|error]
inspect log read <log_file>
```

## inspect score

Rescore an existing log with a new scorer.

```bash
inspect score --log <log_file> --scorer <scorer_module>
```

## Environment Variables

All `--flag` options map to `INSPECT_EVAL_*` env vars:

- `--model` → `INSPECT_EVAL_MODEL`
- `--log-dir` → `INSPECT_LOG_DIR`
- `--max-connections` → `INSPECT_EVAL_MAX_CONNECTIONS`
- `--log-level` → `INSPECT_LOG_LEVEL` (values: debug, info, warning, error)
