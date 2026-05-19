# Refactor discipline

**When this applies:** before starting any non-trivial refactor, or when an AI agent proposes one.

A refactor is justified only if it improves *user experience* or *maintainer experience* in a measurable way. Cosmetic, style, or "modernization-for-its-own-sake" refactors are rejected.

---

## The test

Before starting, answer:

1. **What user-facing or developer-facing pain does this solve?** (latency, bug class, on-call frequency, time-to-ship)
2. **What is the measure of success?** (numbers, not vibes)
3. **What is the blast radius?** (which files / surfaces will change)
4. **What's the rollback plan if it goes wrong?**

If any answer is "none / I don't know / vibes" — don't start.

---

## Refactor green-lights

Worth doing:

- Eliminating a class of bugs (e.g., replacing `useEffect` data-fetching with TanStack Query removes race-condition bugs).
- Removing a dead dependency that is in the critical bundle.
- Fixing a measured performance regression (a flame graph shows it).
- Consolidating duplicate logic that has caused bugs from drift.
- Strict TypeScript flags that have already been enabled — fixing the residual errors.
- Replacing a deprecated API still in active code paths.

---

## Refactor red flags

Almost never worth it:

- "Modernize" axios → fetch / ky / native. The user does not feel it. Risk is high (every page touches the API layer).
- "Convert all class components to hooks" if the class components are stable.
- "Move from CSS Modules to Tailwind" mid-project, with no styling problem to solve.
- "Replace Redux with Zustand" because Zustand is fashionable.
- "Rewrite the form library" without naming the bug.
- "Split this component" because it's "too long" — length alone is not a defect.

The TkDodo *axios → fetch* example is the canonical case: 6 kB saving, no maintainability win, every page touched. Don't.

---

## Sequencing within a refactor

When a refactor is justified:

1. **Land the new pattern alongside the old.** Don't delete first.
2. **Migrate one feature at a time.** Each migration is a separate PR.
3. **Keep tests passing on every commit.**
4. **Delete the old pattern only after every call site is migrated.**

This is the "boy scout" approach. The opposite — a big-bang rewrite — fails predictably.

---

## What to do with code that is bad but not worth refactoring

Mark and move on:

- Add a `// TODO(typescript-code-quality): see <skill-file>` comment with a link.
- Add a lint rule (escalating to `error` later) so new code doesn't compound the problem.
- Track in a tech-debt register if your team has one.

The point is: bad existing code is not a license to write more bad code. New code follows the new rules even if old code violates them.

---

## A refactor budget heuristic

For most product teams, a sustainable cadence is roughly:

- 80% feature work
- 15% small-and-co-located refactor (rename, extract, simplify — within the file you're already changing)
- 5% planned, scoped, measured larger refactor

This is a heuristic, not a contract. A team coming out of a freeze, or in stabilization, may need much higher refactor share — but that should be an explicit decision.

---

## Symptoms that a refactor was a mistake

If during or after, you see:

- The PR has touched 100+ files.
- Reviewers can't tell what changed semantically.
- Tests had to be rewritten (not just updated).
- The dev branch keeps diverging from main, requiring repeated rebase.
- Bugs are surfacing in unrelated areas.

Stop. Revert. Re-plan with smaller scope.

---

## References

- TkDodo — *Refactor impactfully*, *Always provide customer value*
- Sandi Metz — "Prefer duplication over the wrong abstraction"
- Dan Abramov — *The WET Codebase*
