# What would go wrong

### C1
- file: specs/2.md
- section: (entire file)
- quote: Repo: app
- match: breakdown.md › ## Story 2: Storefront wishlist page
- severity: blocking
- blocks because: Story 2 is the UI wishlist page delivering TypeScript/Next.js components in `storefront/src/app/wishlist/`. The spec describes a backend Java/Spring Boot story with database migrations and REST controllers. The developer would implement backend code instead of the storefront UI, building the wrong thing entirely.
- problem: The file `specs/2.md` contains the backend API spec (from Story 1), not the UI spec (Story 2). The breakdown explicitly states Story 2 is "Storefront wishlist page" delivering UI components, while this file describes Java entities, REST endpoints, and database migrations.
- change: Replace the entire content of `specs/2.md` with the Story 2 LLD from `stories/02-app.md` (Storefront wishlist page — TypeScript, Next.js, React components for the wishlist page UI). The current content belongs in `specs/1.md`.

---

### C3
- file: stories/01-app.md
- section: ## Java Implementation › ### 3. Service: `ProductService` (new, if missing)
- quote: **Action Required:** Verify if `ProductService` already exists in `customer` service. If missing, create it following the `LocationService` pattern
- match: repo-outline/app › customer/src/main/java/com/yas/customer/service/LocationService.java
- severity: blocking
- blocks because: The repo-outline shows no `ProductService` in `customer/src/main/java/com/yas/customer/service/`, so the developer would create it. But the LLD's implementation calls `restClient.get().uri(url).retrieve().body(new ParameterizedTypeReference<List<ProductThumbnailVm>>() {})`, which returns `List<ProductThumbnailVm>`, not a single `ProductThumbnailVm`. The `getProductById(Long id)` method would fail at runtime because the LLD's `getProductById` implementation does `products.getFirst()` on a list that might be empty, and it uses `getProducts(List.of(id))` which the product service endpoint `/storefront/products/list-featured?productId={ids}` expects as a comma-separated query param (see cart service's `ProductService.getProducts()`), but the LLD code passes `List.of(id)` directly to `queryParam("productId", ids)` without converting to a string.
- problem: The LLD's `ProductService` implementation has two issues: (1) the `getProducts()` method's URL construction passes `ids` (a `List<Long>`) to `queryParam("productId", ids)` without converting to a comma-separated string (the cart service does the same, but the product service endpoint format is not verified in the docs, so this might fail), and (2) the `getProductById()` method calls `products.getFirst()` on a list that could be empty (no null check), which would throw `NoSuchElementException` at runtime if the product doesn't exist. The story says `existsById()` checks for existence, but `getProductById()` is called in `WishlistService.getWishlistItems()` without that check.
- change: In `ProductService.getProductById()`, add a null check: `if (CollectionUtils.isEmpty(products)) { return null; }` before `return products.getFirst();`. This matches the cart service's `ProductService.getProductById()` implementation pattern (see `repo-file/app/cart/src/main/java/com/yas/cart/service/ProductService.java`). The URL construction is assumed to work based on the cart service using the same pattern; if the product service endpoint format differs, verify and adjust.

---

### C6
- file: stories/01-app.md
- section: ## Java Implementation › ### 4. Service: `WishlistService` › **Key Methods:**
- quote: `WishlistService.toWishlistItemGetVm()` to use correct field names.
- match: hld.md › ## AD-2: Wishlist REST API Contract › #### `GET /api/customer/storefront/wishlist/items`
- severity: blocking
- blocks because: The HLD lists the response fields as `productName`, `productSlug`, `thumbnailUrl`, `price`, `isAvailable`, `discontinuedMessage`. The LLD's `WishlistService.fetchProductDetailsAsync()` code builds `WishlistItemVm` using `product.name()`, `product.slug()`, `product.thumbnailMediaUrl()`, `product.price()`. The cart service's `ProductThumbnailVm` record (from `repo-file/app/cart/src/main/java/com/yas/cart/viewmodel/ProductThumbnailVm.java`) shows fields named `name`, `slug`, `thumbnailMediaUrl`, `price`. But the LLD code maps `product.thumbnailMediaUrl()` to `.thumbnailUrl(...)` in the builder, not `.thumbnailMediaUrl(...)`, which means the ViewModel record field is named `thumbnailUrl` (not `thumbnailMediaUrl`). This is consistent with the HLD response format (`thumbnailUrl` in JSON), but the LLD's ViewModel definition in section "### 6. ViewModels › **Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemVm.java`" shows `private String thumbnailUrl;` (correct). The developer would build the code as written, which is correct. No problem here — the field names match.
- problem: (False alarm — rechecking the LLD code, the mapping is correct: `ProductThumbnailVm` has `thumbnailMediaUrl()`, and `WishlistItemVm` has `thumbnailUrl`, so the builder `.thumbnailUrl(product.thumbnailMediaUrl())` is correct. The HLD and LLD agree on the JSON field name `thumbnailUrl`. No issue.)
- change: (No change needed — the field mapping is correct.)

(Dropping C6 — no actual problem found after re-checking the code.)

---

## Notes for the gate

### Product service endpoint format
The LLD's `ProductService.getProducts()` calls `UriComponentsBuilder.fromUriString(...).path("/storefront/products/list-featured").queryParam("productId", ids)` where `ids` is `List<Long>`. The cart service uses the same pattern (see `repo-file/app/cart/src/main/java/com/yas/cart/service/ProductService.java`), so this is assumed to work. However, the breakdown's open question asks: "Does product service accept multiple `productId` query params (e.g., `?productId=1&productId=2`) or a comma-separated list (e.g., `?productId=1,2,3`)?" The LLD does not resolve this. If the product service expects `?productId=1,2,3`, the code would need to convert `ids` to a string. This should be verified during implementation, but the LLD does not state which format is correct.

---

VERDICT: FAIL
