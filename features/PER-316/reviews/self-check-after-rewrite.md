# What would go wrong

### C1
- file: specs/2.md
- section: (entire file)
- quote: # Technical Specification — Ticket 2 ... **Title:** Wishlist persistence and API endpoints
- match: breakdown.md › ## Story 2: Storefront wishlist page
- severity: blocking
- blocks because: Dev would build backend Java/Spring Boot code (entities, REST controllers, database migrations) for a story that must deliver UI components (TypeScript/Next.js, React pages in `storefront/src/app/wishlist/`). The breakdown says Story 2 is "Storefront wishlist page" delivering `page.tsx`, `WishlistItem.tsx`, and `wishlistService.ts`. This spec describes backend `WishlistItem.java`, `WishlistController.java`, and SQL migrations. Dev would build the wrong artifact in the wrong language.
- problem: The file `specs/2.md` contains the backend API spec (database schema, Java entities, REST endpoints) but the breakdown states Story 2 is "Storefront wishlist page" (UI components, Next.js pages, TypeScript API client).
- change: Replace the content of `specs/2.md` with the Story 2 UI spec from `stories/02-app.md` (sections: Overview, Stack, Acceptance Criteria, File Touch List, TypeScript Interfaces, API Client, Wishlist Page Component, Wishlist Item Component, Header Modification, Product Page Modification, Tests).

---

### C4
- file: specs/3.md
- section: ## Database Schema › ### Cart Service — Verification and Migration
- quote: **Action Required:** Verify in `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql` whether the `cart_item` table enforces UNIQUE(`customer_id`, `product_id`). ... **Decision:** Add migration `changelog-0005.sql` to enforce uniqueness.
- match: repo-file/app/cart/src/main/java/com/yas/cart/model/CartItem.java
- severity: blocking
- blocks because: The spec says to verify whether a unique constraint exists and add a migration if missing. The entity `CartItem.java` uses `@IdClass(CartItemId.class)` with composite PK `(customerId, productId)`, which enforces uniqueness at the JPA level. The SQL migration in `changelog-0001.sql` creates `cart_item` with single PK `id bigint not null` and no unique constraint. Dev would add a migration that conflicts with the entity definition (entity has composite PK, schema has single PK), causing Hibernate schema validation failures or runtime errors.
- problem: The spec asks Dev to verify and add a unique constraint on `cart_item`, but the entity already uses a composite primary key `@IdClass(CartItemId.class)` on `(customerId, productId)`, while the schema has single PK `id`. Adding a unique constraint without resolving the entity-schema mismatch would cause runtime errors.
- change: State explicitly: "The entity `CartItem.java` uses `@IdClass(CartItemId.class)` with composite PK `(customerId, productId)`, which enforces uniqueness at the JPA level. No new migration is needed. If the schema's single-PK structure causes issues, the architect at the gate must resolve the entity-schema mismatch (outside this story's scope)."

---

### C6
- file: stories/01-app.md
- section: ## Open Questions › 1. Product Service Batch Endpoint
- quote: Does product service already have a batch product fetch endpoint? If not, Story 1 must implement parallel calls or propose a batch API addition. **Resolution:** Check `product/src/main/java/com/yas/product/controller/ProductStorefrontController.java`. If missing, implement parallel calls in `WishlistService.getWishlistItems()`.
- match: repo-file/app/cart/src/main/java/com/yas/cart/service/ProductService.java › public List<ProductThumbnailVm> getProducts(List<Long> ids)
- severity: blocking
- blocks because: Dev would search for a batch endpoint or implement parallel calls, but the cart service's `ProductService.getProducts(List<Long> ids)` already shows the product service accepts a list of IDs in a single call (`queryParam("productId", ids)`). Dev would waste time implementing a parallel-fetch solution that is unnecessary, or propose a batch endpoint that already exists.
- problem: The open question asks whether a batch endpoint exists and suggests parallel calls as a fallback, but the cart service's `ProductService.getProducts()` shows the product service already accepts multiple product IDs in one call.
- change: Remove the open question. In the `WishlistService` section, state: "Call `ProductService.getProducts(List<Long> ids)` with all wishlist item product IDs in a single batch call (matches cart service pattern in `cart/src/main/java/com/yas/cart/service/ProductService.java`). No parallel fetch logic needed."

---

## Notes for the gate

### Entity-Schema Mismatch in Cart Service
- context: `cart/src/main/java/com/yas/cart/model/CartItem.java` uses `@IdClass(CartItemId.class)` with composite PK `(customerId, productId)`, but `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql` creates `cart_item` with single PK `id bigint not null`.
- issue: Story 3 spec asks Dev to verify and add a unique constraint on `cart_item`. Adding a constraint without resolving the entity-schema mismatch would cause Hibernate validation failures or runtime errors.
- action: The architect at the gate should decide whether to: (a) change the entity to match the schema (single PK `id`, add unique constraint on `customer_id, product_id`), or (b) change the schema to match the entity (composite PK on `customer_id, product_id`). This is a structural decision outside the scope of rewriting Story 3.

---

VERDICT: FAIL
