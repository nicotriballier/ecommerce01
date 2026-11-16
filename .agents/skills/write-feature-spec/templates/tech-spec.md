# Tech spec: <Feature name>

PRD: [prd.md](./prd.md) · Status: Draft / In review / Approved

## 1. Summary
One paragraph: what we're building and the chosen approach.

## 2. Context
Current behavior and relevant code (link files). Constraints from ADRs and the glossary.

## 3. Design
### Bounded contexts touched
| Context | Change (domain / app / adapter) |
|---|---|

### Domain model
New/changed entities, value objects, invariants, domain events.

### Data model and migrations
- `migrations/0000NN_<name>.up.sql` / `.down.sql`: tables, columns, indexes, constraints.
- Backfill strategy, if any.

### HTTP / HTMX
| Method | Route | Handler | Auth | Notes |
|---|---|---|---|---|
Templates/partials added or changed (storefront vs admin shell).

### Key flows
Sequence of calls for the main path (Mermaid diagram if non-trivial).

## 4. Alternatives considered
| Option | Pros | Cons | Why not |
|---|---|---|---|

## 5. Edge cases and failure modes
Invalid input, concurrency, money/rounding, stock, authz, idempotency, partial failure.

## 6. Security and privacy
Authz rules, CSRF, PII handling, rate limits.

## 7. Testing plan
| AC | Test type (unit / integration / e2e) | Location |
|---|---|---|
Commands: `make tests`, `make integration`, golangci-lint.

## 8. Rollout
Migration order, feature flag / demo mode, backward compatibility, rollback steps.

## 9. Observability
Logs, metrics, admin visibility.

## 10. Task breakdown
Ordered list of PR-sized tasks with estimates.

## 11. Open questions
| # | Question | Proposed default | Decision |
|---|---|---|---|

## Changelog
- YYYY-MM-DD: initial draft
