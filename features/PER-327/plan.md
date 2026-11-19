# Plan — PER-327 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Customers can submit product reviews with 1-5 star ratings; this requires a new API endpoint, stored review data, and schema changes for review storage, so architecture design is needed.

## Steps

1. [ ] **analyst (requirements)** — Convert the ticket into structured requirements with Given/When/Then format and acceptance criteria.
   - focus: Focus on the review submission flow, validation rules (rating 1-5, non-empty content), duplicate prevention, and the rating recalculation mechanism.
2. [ ] **architect (hld)** — Design the API endpoint for review submission, database schema for reviews table, and the rating aggregation mechanism.
   - focus: Define POST /storefront/reviews endpoint contract, review entity schema (productId, customerId, rating, content, timestamp), aggregate rating calculation strategy, and idempotency handling for duplicate prevention.
3. [ ] **scrum_master (breakdown)** — Break down into implementable stories: backend API, frontend form, rating display updates.
   - focus: Split into: (1) backend review submission endpoint with validation and rating recalculation, (2) frontend review form component with star selector and content input, (3) reviews section display updates.
4. [ ] **designer (lld)** — Provide file-by-file implementation plans for each story covering backend entities, controllers, services, and frontend components.
   - focus: Detail controller methods, service layer for validation and rating aggregation, repository interfaces, React components for review form and display, and state management.
5. [ ] **test_architect (tests)** — Map test cases to each acceptance criterion including edge cases for validation, duplicate prevention, and rating calculations.
   - focus: Cover boundary rating values (0, 1, 5, 6), empty/whitespace content, concurrent submission prevention, rating recalculation accuracy, and persistence after page refresh.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Ensure consistency between requirements, HLD API contracts, story acceptance criteria, and test coverage.
   - focus: Verify rating validation rules match across all documents, duplicate prevention mechanism is consistently specified, and rating calculation logic aligns in architecture and implementation stories.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket clearly defines scope and acceptance criteria for review submission; no product ambiguity exists.
