# Research Notes

Use these as evidence themes when grounding reviews or writing rationale.

## Long-run coding-agent issues

Observed in public research, benchmarks, and field reports:

- AI-generated code often passes local checks while worsening maintainability.
- Long-horizon repository tasks remain much harder than isolated coding tasks.
- Agentic coding can increase verbosity, duplicate code, and structural erosion.
- AI-heavy codebases may show higher churn and duplication.
- Generated code can introduce security findings.
- Developers often distrust AI output enough to require careful review.

## Python-specific evidence themes

- AI-generated Python refactoring PRs can introduce new linter and security findings.
- Python flexibility makes “plausible but structureless” code easy to write.
- Common failure areas: broad exceptions, unsafe defaults, duplicate serialization, weak tests, dependency sprawl, and poor boundaries.

## Anecdotal but common field reports

Treat these as directional signals:

- Agents create `utils`/`helpers` despite existing project utilities.
- Agents add wrappers and fake service layers.
- Agents write bad mocks and tests that only prove mocks work.
- Agents stub around missing environment instead of fixing setup.
- Agents overuse comments/docs.
- Agents create `v2`/compat layers to avoid replacing code.

## How to cite in a review

Do not over-cite. Use citations only when writing external-facing reports.

For code review, concrete repo evidence matters more: file path, function/class, import graph, test failure, duplicated implementation, unsafe destructive command, missing transition, profile/benchmark.
