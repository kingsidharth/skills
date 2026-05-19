# Refactor Safety Checklist

Before changing code:

- [ ] Explain current behavior.
- [ ] Identify intended behavior.
- [ ] Decide if behavior tests are needed first.
- [ ] Verify test DB/object-store safety.
- [ ] Avoid public behavior changes unless planned.
- [ ] Avoid `v2`, shim, compat, or legacy paths unless required.
- [ ] Avoid formatting churn mixed with logic.
- [ ] Avoid new dependencies unless approved.
- [ ] Avoid useless docs/comments.

During refactor:

- [ ] Change one issue class at a time.
- [ ] Keep route/API behavior stable unless intended.
- [ ] Preserve transaction boundaries or improve them deliberately.
- [ ] Delete dead code.
- [ ] Remove duplicate path after replacement.
- [ ] Run focused tests after each meaningful change.

After refactor:

- [ ] Run focused tests.
- [ ] Run broader tests/checks.
- [ ] Search for duplicate old/new implementation.
- [ ] Search for dead imports.
- [ ] Report remaining risks.
