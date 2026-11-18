# What would go wrong

### C1
- file: specs/2.md
- section: ## Stack
- quote: Repo: app
- severity: blocking
- blocks because: Story 2 is a UI story for the storefront, not a backend story in the `app` repo. The developer would implement backend code in Java/Spring Boot instead of TypeScript/Next.js storefront UI, building the wrong thing entirely.
- problem: The file states "Repo: `app`" and describes this as a backend Java/Spring Boot story with database migrations and REST controllers. The HLD and breakdown both describe Story 2 as "Storefront wishlist page" delivering UI components in `storefront/src/app/wishlist/`, not backend code in `customer/`.
- change: This is Story 2 (Storefront wishlist page), not Story 1 (Backend wishlist storage and API). Replace the entire content with the Story 2 LLD from `stories/02-app.md`, or if this file is intended to be Story 1, rename it to `specs/1.md` and create a new `specs/2.md` for the UI story.

### C2
- file: breakdown.md
- section: ## Story 1: Backend wishlist storage and API › ### Implementation Pointers › **ProductService (new, if missing):**
- quote: If does not exist, create with method: `ProductDetailVm getProduct(Long productId)`
- match: repo-file/app/cart/src/main/java/com/yas/cart/service/ProductService.java
- severity: blocking
- blocks because: Dev would create a new `ProductService` in the customer service with a `getProduct()` method returning `ProductDetailVm`, but the cart service's existing `ProductService` has no such method—it has `getProducts(List<Long> ids)` and `getProductById(Long id)` returning `ProductThumbnailVm`. Dev would use the wrong method signature and return type, breaking the API contract.
- problem: The breakdown says to create `ProductService.getProduct(Long productId)` returning `ProductDetailVm` if the service does not exist. The cart service already has `ProductService` (see `cart/src/main/java/com/yas/cart/service/ProductService.java`) with `getProductById(Long id)` returning `ProductThumbnailVm` and `getProducts(List<Long> ids)` returning `List<ProductThumbnailVm>`. The story is for the customer service, which does not have a `ProductService`.
- change: In the customer service, create `ProductService` with `getProductById(Long id)` returning `ProductThumbnailVm` (matching the cart service pattern) or `getProducts(List<Long> ids)` for batch fetching. Update `WishlistService.fetchProductDetailsAsync()` to use this method and map `ProductThumbnailVm` fields to `WishlistItemGetVm` fields. Remove reference to `ProductDetailVm` (which does not exist in the codebase shown).

### C3
- file: stories/01-app.md
- section: ## Java Implementation › ### 3. Service: `ProductService` (new, if missing)
- quote: Response includes `published: boolean`, `stockQuantity: number` (or similar fields — verify exact names in product service `ProductThumbnailVm`)
- match: repo-file/app/cart/src/main/java/com/yas/cart/viewmodel/ProductThumbnailVm.java › public record ProductThumbnailVm(Long id, String name, String slug, String thumbnailMediaUrl, Double price)
- severity: blocking
- blocks because: Dev would check for `published` and `stockQuantity` fields in `ProductThumbnailVm` to determine availability, but those fields do not exist. They would either fail to compile or invent the fields, building incorrect availability logic.
- problem: The LLD says `ProductThumbnailVm` includes `published` and `stockQuantity` fields to determine availability. The actual `ProductThumbnailVm` record (see `cart/src/main/java/com/yas/cart/viewmodel/ProductThumbnailVm.java`) has only `id`, `name`, `slug`, `thumbnailMediaUrl`, and `price`—no availability fields.
- change: Availability is determined by whether the product service returns the product (200 response) or not (404/empty list). Update `WishlistService.fetchProductDetailsAsync()` to set `isAvailable = true` if `getProductById()` returns a non-null `ProductThumbnailVm`, and `isAvailable = false` if it returns `null` or throws `NotFoundException`. Remove all references to `published` and `stockQuantity` fields in Story 1 LLD.

### C4
- file: stories/01-app.md
- section: ## REST API Contract › ### Endpoint: `GET /api/customer/storefront/wishlist/items` › **Behavior:**
- quote: For each `wishlist_item`, fetch product details via `ProductService.getProducts(List.of(productId))`
- match: breakdown.md › ## Story 1: Backend wishlist storage and API › ### Implementation Pointers › **Service (new):** › Method: `getWishlistItems(String customerId)` — SELECT wishlist items, fetch product details in parallel with `CompletableFuture` (2s timeout per product)
- severity: blocking
- blocks because: Dev would call `getProducts()` once per item (inefficient but not blocking) or in a loop, but the breakdown says to use parallel `CompletableFuture` calls. The LLD's single-call approach and the breakdown's parallel approach are different implementations; dev would have to choose, and inconsistency means they might pick the slower one.
- problem: Story 1 LLD says to call `getProducts(List.of(productId))` for each item, implying sequential or batched calls. The breakdown says to fetch product details "in parallel with `CompletableFuture`" with a 2-second timeout per product. These are two different strategies.
- change: Align both documents: state that the service calls `getProducts(List.of(productId))` for each wishlist item in parallel using `CompletableFuture`, with a 2-second timeout per product. The final implementation collects all results before returning the response. If a batch endpoint is available, prefer that; otherwise, use parallel single-product calls.

### C5
- file: specs/3.md
- section: ## Database Schema › ### Cart Service — Verification and Migration › **Decision:** Add migration `changelog-0005.sql` to enforce uniqueness.
- match: repo-file/app/cart/src/main/resources/db/changelog/ddl/changelog-0001.sql › create table cart_item (id bigint not null, quantity integer not null, parent_product_id bigint, product_id bigint, cart_id bigint not null, primary key (id));
- severity: note
- blocks because: N/A (not blocking)
- problem: The story says to add a UNIQUE constraint on `(cart_id, product_id)` in a new migration `changelog-0005.sql`. The existing schema (see `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql`) already has a primary key on `id`, not a composite key. The cart service's `CartItem` model (see `cart/src/main/java/com/yas/cart/model/CartItem.java`) uses `@IdClass(CartItemId.class)` with `@Id` on both `customerId` and `productId`, meaning the entity already enforces uniqueness at the ORM level for `(customerId, productId)`. The database schema and the entity are mismatched.
- change: Verify in the actual running database whether `cart_item` has a composite primary key on `(customer_id, product_id)` or a single `id` column. If it already has the composite PK, skip the migration. If it has only `id`, the migration is needed, but it must also align the entity's `@Id` fields with the schema. This is a schema-entity mismatch that the architect at the gate should resolve, as it cannot be fixed by rewriting the spec alone.

VERDICT: FAIL
