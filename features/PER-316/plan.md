# Plan — PER-316 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Wishlist feature with persistence across sessions, requiring customer storage schema changes, API endpoints, and UI pages across multiple domains (customer, product, cart), so it needs an HLD and breakdown into stories.

## Steps

1. [x] **analyst (requirements)** — Turn the ticket into structured requirements with Given/When/Then acceptance criteria.
   - focus: Focus on the persistence contract (what identifies a saved item), the cross-domain interactions (customer owns the list, product provides the item data, cart receives items), and the guest vs signed-in distinction.
2. [x] **architect (hld)** — This adds database schema (wishlist table), new API endpoints (add/remove/list wishlist items, add to cart from wishlist), and contracts between customer/product/cart domains.
   - focus: Design the wishlist storage schema, the REST API contract for wishlist operations, how discontinued products are handled in queries, and the integration points between customer service (owns wishlist), product service (provides item details), and cart service (receives items from wishlist).
3. [x] **scrum_master (breakdown)** — The ticket explicitly states this splits into several stories: persistence layer, storefront UI, and cart integration.
   - focus: Break down into: (1) backend wishlist storage and API, (2) UI wishlist page with add/remove/view, (3) add-to-cart integration, ensuring each story is independently shippable.
4. [x] **designer (lld)** — Each story needs concrete file plans for database migrations, API controllers, service layer, and UI components.
   - focus: Provide file-level implementation plans: which entity classes, repository interfaces, controller endpoints, service methods, and UI components each story needs.
5. [x] **test_architect (tests)** — The acceptance criteria include important edge cases: duplicate prevention, persistence across sessions, discontinued product handling, and guest access prevention.
   - focus: Map each acceptance criterion to specific test cases, especially for: idempotent adds, cross-session persistence, discontinued product display-but-no-add-to-cart, and guest sign-in redirection.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [x] **critic (consistency_check)** — Requirements, HLD, breakdown, and story LLDs must agree on the wishlist schema, API contract, and cross-service integration points.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket clearly defines scope and acceptance criteria; no product ambiguity that needs pre-technical sign-off.
