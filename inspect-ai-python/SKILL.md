---
name: inspect-ai-python
description: Build LLM evaluations with Inspect AI (inspect-ai Python package by UK AISI). Use this skill whenever the user mentions Inspect AI, inspect-ai, LLM evaluation frameworks, eval tasks, eval datasets, solvers, scorers, agent evals, model grading, agentic benchmarks, SWE-bench evals, or any task involving evaluating language models systematically. Also trigger for questions about sandboxing model code execution, tool-use evals, multi-agent evaluation, eval log analysis, or running benchmarks like MMLU, HumanEval, GSM8K, HellaSwag, ARC with Inspect. Even if the user just says "write an eval" or "benchmark this model", consider this skill.
---

# Inspect AI — Python Eval Framework

Open-source LLM evaluation framework by UK AI Security Institute.
`pip install inspect-ai` · [Docs](https://inspect.aisi.org.uk/) · [GitHub](https://github.com/UKGovernmentBEIS/inspect_ai)

## Table of Contents

**Read the section(s) relevant to the task. Each file is self-contained.**

### Setup
→ `references/setup.md` — Installation, API keys, `.env` config, VS Code extension, CLI basics

### Core Concepts
→ `references/core-concepts.md` — Architecture overview: Task = Dataset + Solver + Scorer. The eval() pipeline, @task decorator, TaskState lifecycle

### Components
→ `references/components.md`
- **Datasets** — Sample, CSV/JSON/HuggingFace loading, field mapping, filtering, metadata
- **Solvers** — Built-in (generate, chain_of_thought, system_message, multiple_choice, self_critique), custom solvers, @solver decorator
- **Scorers** — Built-in (exact, match, includes, model_graded_qa/fact, choice, math, f1, perplexity), custom scorers, metrics, epochs

### Models
→ `references/models.md` — Provider selection (OpenAI, Anthropic, Google, etc.), GenerateConfig, model roles, caching, batch mode, compaction, multimodal, reasoning models, structured output

### Tools
→ `references/tools.md` — @tool decorator, standard tools (bash, python, web_search, text_editor, computer, web_browser), MCP tools, custom tools, sandboxing (Docker, per-sample setup), tool approval

### Agents
→ `references/agents.md` — Agent protocol, ReAct agent, Deep Agent, multi-agent, custom agents, Agent Bridge (Claude Code, Codex CLI, Gemini CLI), Human Agent

### Analysis
→ `references/analysis.md` — Eval logs, Log Viewer, log dataframes, programmatic log access

### Advanced
→ `references/advanced.md` — Eval sets, error handling, limits (message/token/time/working), typing, tracing, parallelism, interactivity, early stopping, extensions API

### CLI Reference
→ `references/cli.md` — `inspect eval`, `inspect log`, `inspect view`, `inspect score`, task params, environment variables

### Extensions & Ecosystem
→ `references/extensions.md` — Inspect Scout (transcript analysis), Inspect Flow (workflow orchestration), Inspect Sandboxes (Daytona), inspect_costs_plugin
