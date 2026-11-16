---
name: write-feature-spec
description: Write a PRD and a tech spec for a new ecommerce01 feature and get them approved before any code is written. Use when asked to spec out, plan, or write a PRD/tech spec/design doc for a feature.
---

# Write a feature spec (PRD + tech spec)

## Output
- `docs/specs/<feature-slug>/prd.md` from `templates/prd.md`
- `docs/specs/<feature-slug>/tech-spec.md` from `templates/tech-spec.md`
- Opened as a **draft PR** titled `spec: <feature>`. No implementation code in this PR.

## Procedure
1. **Understand the ask.** Restate the feature in one sentence. Identify the user (shopper, admin, ops) and the problem.
2. **Research before asking.** Read `README.md`, `docs/glossary.md`, `docs/subdomains.md`, the ADRs, and the bounded contexts the feature touches (`backend/<context>/{domain,app,adapter}`, `backend/layout` for HTMX). Check `migrations/` for the current schema and `backend/integration/` for test patterns.
3. **Ask only blocking questions** (max 5, batched in one message). Anything else: make a reasoned assumption and record it under "Assumptions".
4. **Write the PRD first**, then the tech spec. The tech spec must trace back to PRD acceptance criteria (reference them by ID, e.g. AC-3).
5. **Self-review** against the checklist below, then open the draft PR and send the user the link with the open questions listed.
6. **Iterate** on review comments: resolve each, update docs, reply on the thread. Keep a short changelog at the bottom of each doc.
7. **Stop at approval.** Do not implement until the user explicitly approves the spec.

## Depth rules (edit these to match your taste)
- PRD: problem and outcome first; ≤ 2 pages; no implementation details.
- Acceptance criteria: numbered (AC-1…), testable, Given/When/Then.
- Tech spec: be concrete where it is risky (schema, money, stock, auth, events); be brief where it follows an existing pattern (say "same as <file>").
- Every schema change lists the exact up/down migration names (`migrations/0000NN_<name>.{up,down}.sql`).
- Use the glossary's ubiquitous language; respect bounded-context boundaries (`layout` depends on contexts via small local interfaces).
- Prefer tables and bullet lists over prose. No filler.

## Self-review checklist
- [ ] Every AC is covered by the test plan.
- [ ] Out-of-scope section exists and is explicit.
- [ ] Edge cases cover: empty/invalid input, concurrency (stock, cart), money/rounding, authz (customer vs admin), idempotency.
- [ ] Migrations are backward compatible or the rollout explains why not.
- [ ] Open questions are listed with a proposed default for each.
