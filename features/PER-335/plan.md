# Plan — PER-335 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Customer product review submission with rating, validation, and display—changes the API contract (new review endpoint), schema (review storage), and UI state (average rating, review count), so architecture work is needed.

## Steps

1. [x] **analyst (requirements)** — Convert ticket into structured Given/When/Then requirements for review submission, validation, storage, and display logic.
   - focus: Focus on the full flow: UI interaction, API contract for POST /products/{id}/reviews, validation rules (rating 1-5, non-empty content), storage (new review record), aggregate updates (average rating, review count), and idempotency to prevent duplicates.
2. [x] **architect (hld)** — Design the review submission API endpoint, review schema, aggregate calculation strategy, and idempotency mechanism.
   - focus: Define POST /products/{id}/reviews contract (request/response schemas), review entity schema (product_id, customer_id, rating, content, timestamp), how average rating and count are calculated and stored (product aggregates table or computed on read), and idempotency approach (request token or database constraint). Address validation at API level and persistence layer.
3. [x] **scrum_master (breakdown)** — Break down into stories: backend API and schema, frontend UI and validation, aggregate calculation logic.
   - focus: Split by vertical slice: (1) backend review submission endpoint and storage, (2) frontend review form and submission, (3) display updated reviews and aggregates, (4) validation and error handling across layers.
4. [x] **designer (lld)** — Each story spans UI, API, database, and validation layers—file-by-file plans ensure correct implementation.
   - focus: For backend stories: specify controller, service, repository, entity, and DTO files. For frontend stories: specify component, API client, and state management files. Include validation logic locations and test file mappings.
5. [x] **test_architect (tests)** — Acceptance criteria include edge cases: invalid ratings, empty content, duplicate prevention, aggregate recalculation, persistence across refreshes.
   - focus: Map each AC to test cases: rating validation (boundary values 0, 1, 5, 6), empty/whitespace content, missing fields, successful submission with confirmation, review display, aggregate updates (initial review, subsequent reviews), idempotency (double-click), persistence (page refresh), and error scenarios.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [x] **critic (consistency_check)** — Requirements, HLD, stories, and tests must agree on API contract, validation rules, schema, aggregate calculation, and idempotency strategy.
   - focus: Cross-check that the API contract in HLD matches request/response in stories, validation rules are consistent across requirements/HLD/tests, aggregate calculation appears in all relevant artifacts, and idempotency mechanism is specified and testable.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket clearly states scope and acceptance criteria for review submission feature; no product ambiguity to resolve.
