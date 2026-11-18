# Plan — PER-315 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Wishlist feature with database schema changes (new wishlist table), new API endpoints, and UI that spans multiple stories across customer, product and cart areas.

## Steps

1. [ ] **analyst (requirements)** — Turn ticket into structured requirements with clear boundaries between wishlist persistence, UI, and cart integration.
   - focus: Break down into separate concerns: database/API for wishlist storage, storefront wishlist page, cart integration. Clarify the guest-to-signed-in flow and discontinued product display rules.
2. [ ] **architect (hld)** — Design new wishlist table schema, REST endpoints for CRUD operations, and how customer/product/cart modules interact with the wishlist contract.
   - focus: Define wishlist data model (customer_id, product_id, added_at), API contracts for add/remove/list operations, and integration points with existing cart and product services. Address idempotency for duplicate adds.
3. [ ] **scrum_master (breakdown)** — Split into implementable stories: backend persistence and API, wishlist UI page, cart integration with 'add saved item' action.
   - focus: Sequence stories so database and API come first, then UI can consume them, then cart integration. Each story should be independently testable.
4. [ ] **designer (lld)** — Each story needs file-level implementation detail: which controllers, services, repositories, and UI components to create or modify.
   - focus: Map each story to specific files: new WishlistController, WishlistService, WishlistRepository and entity for backend; new wishlist page components for UI; CartService modifications for 'add from wishlist' feature.
5. [ ] **test_architect (tests)** — Acceptance criteria include edge cases worth explicit test coverage: duplicate prevention, discontinued products, guest vs signed-in behavior.
   - focus: Map tests 1:1 to AC: duplicate add returns existing item, adding to cart preserves wishlist entry, discontinued product shows but blocks cart-add, guest redirects to sign-in.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Requirements, HLD, breakdown, and stories must all agree on wishlist schema, API contracts, and cross-module integration points.
   - focus: Verify consistency between wishlist table design in HLD, API endpoint definitions, story acceptance criteria, and test cases. Check that cart and product references match.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket clearly defines scope and acceptance criteria; no ambiguity in product requirements or cross-team dependencies.
