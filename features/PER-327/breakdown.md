# Breakdown — PER-327

**Customers can write and submit product reviews**

---

## Overview

This breakdown splits the review submission feature into three sequential implementation stories. The first story establishes the backend submission endpoint with validation, duplicate prevention, and aggregate rating recalculation. The second story builds the frontend form component with star selector and content input. The third story updates the reviews section display to show submitted reviews and updated statistics. Each story is independently buildable and testable; together they deliver all acceptance criteria.

---

### Story 1: Backend review submission endpoint with validation and rating recalculation

**Repo:** `app`

#### Context
This story establishes the server-side foundation for review submission. It adds the REST endpoint, server-side validation (rating range, content non-empty), duplicate prevention via database constraint, review persistence, aggregate rating calculation, and the product service integration to update average rating and review count. Without this story, no reviews can be submitted or stored.

#### Scope

### Story 1: Backend review submission endpoint with validation and rating recalculation › Scope

**Note:** Verify that `rating/src/main/java/com/yas/rating/controller/RatingController.java` is a backoffice-only controller before creating `StorefrontReviewController.java`. If `RatingController` already serves storefront endpoints, modify it instead of creating a new file.

- **New files:**
  - `rating/src/main/java/com/yas/rating/controller/StorefrontReviewController.java` (new controller for storefront review endpoints)
  - `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java` (request DTO: `productId`, `rating`, `content`)
  - `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java` (response DTO: `id`, `productId`, `customerId`, `rating`, `content`, `createdOn`)
  - `rating/src/main/java/com/yas/rating/viewmodel/ReviewListVm.java` (paginated response DTO: `reviews`, `totalPages`, `totalElements`, `pageNo`, `pageSize`)
  - `product/src/main/java/com/yas/product/controller/ProductAggregateController.java` (new backoffice controller for aggregate updates)
  - `product/src/main/java/com/yas/product/viewmodel/RatingAggregatePostVm.java` (request DTO: `averageRating`, `reviewCount`)
  - `rating/src/main/resources/db/changelog/ddl/changelog-rating-unique-constraint.sql` (new Liquibase changelog: add unique constraint `(customerId, productId)` to `rating` table, add indexes on `productId` and `(customerId, productId)`)
  - `product/src/main/resources/db/changelog/ddl/changelog-product-rating-aggregates.sql` (new Liquibase changelog: add `average_rating` DECIMAL and `review_count` BIGINT columns to `product` table, only if missing)
- **Modified files:**
  - `rating/src/main/java/com/yas/rating/service/RatingService.java` (add methods: `createReview(ReviewPostVm, String customerId)`, `getReviewsByProductId(Long productId, int pageNo, int pageSize)`, `calculateAggregates(Long productId)` — returns `{averageRating, reviewCount}`)
  - `product/src/main/java/com/yas/product/service/ProductService.java` (add method: `updateRatingAggregates(Long productId, BigDecimal averageRating, Long reviewCount)`)
  - `product/src/main/java/com/yas/product/model/Product.java` (add fields: `@Column(name = "average_rating") private BigDecimal averageRating;`, `@Column(name = "review_count") private Long reviewCount;` — only if missing, verify in repo)
  - `rating/src/main/java/com/yas/rating/model/Rating.java` (verify existing schema matches spec; no changes expected unless fields are missing)

#### Implementation Pointers
- **StorefrontReviewController.java:**
  - `POST /api/rating/storefront/reviews` → `createReview(@RequestBody @Valid ReviewPostVm request)` → extract `customerId` from `AuthenticationUtils.extractUserId()` → call `RatingService.createReview(request, customerId)` → return `201 Created` with `ReviewVm` body, or `400/409/500` on error
  - `GET /api/rating/storefront/reviews?productId={productId}&pageNo={pageNo}&pageSize={pageSize}` → `getReviews(@RequestParam Long productId, @RequestParam(defaultValue = "0") int pageNo, @RequestParam(defaultValue = "10") int pageSize)` → call `RatingService.getReviewsByProductId(productId, pageNo, pageSize)` → return `200 OK` with `ReviewListVm` body
- **RatingService.java:**
  - `createReview(ReviewPostVm request, String customerId)`:
    1. Validate: `request.rating` between 1 and 5, `request.content` non-null and non-empty after trim → throw `BadRequestException` if violated
    2. Create `Rating` entity: `productId = request.productId`, `customerId = customerId`, `star = request.rating`, `content = request.content.trim()`, `createdOn = now`, `lastModifiedOn = now`, `createdBy = customerId`, `lastModifiedBy = customerId`
    3. Save via `RatingRepository.save(rating)` → catch `DataIntegrityViolationException` (unique constraint violation) → throw `ConflictException("You have already submitted a review for this product.")`
    4. Call `calculateAggregates(request.productId)` → returns `{averageRating, reviewCount}`
    5. Call `ProductService.updateRatingAggregates(request.productId, averageRating, reviewCount)` via REST client (use existing pattern in `rating` service, verify `ProductService` REST client setup) → if call fails, log error and throw `InternalServerErrorException("Failed to update product aggregates.")`
    6. Return `ReviewVm` from saved `Rating`
  - `getReviewsByProductId(Long productId, int pageNo, int pageSize)`:
    1. Query `RatingRepository.findByProductIdOrderByCreatedOnDesc(productId, PageRequest.of(pageNo, pageSize))` → returns `Page<Rating>`
    2. Map `Page<Rating>` to `ReviewListVm` (convert each `Rating` to `ReviewVm`, include pagination metadata)
  - `calculateAggregates(Long productId)`:
    1. Query `RatingRepository.findByProductId(productId)` → returns `List<Rating>`
    2. Calculate `averageRating = SUM(star) / COUNT(*)`, round to one decimal place (use `BigDecimal.setScale(1, RoundingMode.HALF_UP)`)
    3. Calculate `reviewCount = COUNT(*)`
    4. Return `{averageRating, reviewCount}`
- **ProductAggregateController.java:** (provisional — see note above)
  - `PUT /api/product/backoffice/products/{productId}/rating-aggregates` → `updateRatingAggregates(@PathVariable Long productId, @RequestBody @Valid RatingAggregatePostVm request)` → call `ProductService.updateRatingAggregates(productId, request.averageRating, request.reviewCount)` → return `200 OK` with `{productId, averageRating, reviewCount}`
- **ProductService.java:**
  - `updateRatingAggregates(Long productId, BigDecimal averageRating, Long reviewCount)`:
    1. Find `Product` by `id = productId` via `ProductRepository.findById(productId)` → throw `NotFoundException` if missing
    2. Set `product.averageRating = averageRating`, `product.reviewCount = reviewCount`, `product.lastModifiedOn = now`, `product.lastModifiedBy = current user (from security context if available, else "system")`
    3. Save via `ProductRepository.save(product)`
    4. Return `{productId, averageRating, reviewCount}`
- **Liquibase changelogs:**
  - `changelog-rating-unique-constraint.sql`:
    - `ALTER TABLE rating ADD CONSTRAINT uq_customer_product UNIQUE (customer_id, product_id);`
    - `CREATE INDEX idx_rating_product_id ON rating (product_id);` (if not already present)
    - `CREATE INDEX idx_rating_customer_product ON rating (customer_id, product_id);` (if not already present)
  - `changelog-product-rating-aggregates.sql`:
    - `ALTER TABLE product ADD COLUMN average_rating DECIMAL(3,1);` (only if column does not exist)
    - `ALTER TABLE product ADD COLUMN review_count BIGINT;` (only if column does not exist)

#### Acceptance Criteria
1. Customers must be able to select a star rating between 1 and 5. (AC-1 — endpoint accepts `rating` field, server validates range)
2. Review content must be mandatory and must not contain only whitespace. (AC-2 — server rejects empty/whitespace-only `content`)
3. The system must prevent submission if the rating or review content is missing. (AC-3 — server returns `400 Bad Request` if `rating` or `content` is null/empty)
4. The system must reject invalid ratings, including values below 1 or above 5, through API validation. (AC-4 — server returns `400 Bad Request` if `rating < 1` or `rating > 5`)
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (AC-5 — server returns `201 Created` with review data; confirmation message is frontend responsibility in Story 2)
6. The newly submitted review must appear in the product's Reviews section with the correct rating and review content. (AC-6 — `GET /storefront/reviews?productId=...` returns newly saved review; display is Story 3 responsibility)
7. The product's average rating must be recalculated based on all valid reviews. (AC-7 — `calculateAggregates` computes `SUM(star) / COUNT(*)`, `updateRatingAggregates` persists it in `product` table)
8. The total review count must increase by one after successful submission. (AC-8 — `calculateAggregates` computes `COUNT(*)`, `updateRatingAggregates` persists it in `product` table)
9. The rating and review count displayed in the product header must match the Reviews section. (AC-9 — both read from `product` table's `averageRating` and `reviewCount` columns, enforced by single source of truth; display consistency is Story 3 responsibility)
10. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (AC-10 — on error, server returns `400/409/500` without persisting aggregate update; error message display is Story 2 responsibility)
11. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (AC-11 — database unique constraint `(customerId, productId)` prevents duplicates; server returns `409 Conflict` on second attempt; button disable is Story 2 responsibility)
12. Submitted reviews must remain available after refreshing or revisiting the product page. (AC-12 — reviews persisted in database, `GET /storefront/reviews` retrieves them on page load; display is Story 3 responsibility)

#### Dependencies
None — this story is foundational.

#### Out of Scope
- Frontend form component (Story 2)
- Reviews section display updates (Story 3)
- Review editing or deletion (not in this feature)
- Review moderation or reporting (not in this feature)
- Customer name display in reviews (deferred decision — may require `customer` service integration)

#### Verification Approach
**Test count:** 15 tests (8 unit, 5 integration, 2 migration verification)

**Unit tests (8):**
1. `RatingServiceTest.createReview_validInput_savesReview()` — given valid `ReviewPostVm` and `customerId`, when `createReview` called, then `Rating` saved with correct fields, `ReviewVm` returned
2. `RatingServiceTest.createReview_ratingBelowOne_throwsBadRequestException()` — given `rating = 0`, when `createReview` called, then `BadRequestException` thrown with message "Rating must be between 1 and 5."
3. `RatingServiceTest.createReview_ratingAboveFive_throwsBadRequestException()` — given `rating = 6`, when `createReview` called, then `BadRequestException` thrown
4. `RatingServiceTest.createReview_emptyContent_throwsBadRequestException()` — given `content = ""` or `content = "   "`, when `createReview` called, then `BadRequestException` thrown with message "Review content cannot be empty."
5. `RatingServiceTest.createReview_duplicateReview_throwsConflictException()` — given `(customerId, productId)` already exists in database, when `createReview` called, then `DataIntegrityViolationException` caught, `ConflictException` thrown with message "You have already submitted a review for this product."
6. `RatingServiceTest.calculateAggregates_multipleReviews_returnsCorrectAverage()` — given 3 reviews with ratings 3, 4, 5, when `calculateAggregates(productId)` called, then returns `{averageRating: 4.0, reviewCount: 3}`
7. `RatingServiceTest.calculateAggregates_roundingToOneDecimal_roundsCorrectly()` — given 2 reviews with ratings 4, 5, when `calculateAggregates(productId)` called, then returns `{averageRating: 4.5, reviewCount: 2}` (verify rounding)
8. `ProductServiceTest.updateRatingAggregates_validInput_updatesProduct()` — given `productId`, `averageRating = 4.3`, `reviewCount = 42`, when `updateRatingAggregates` called, then `Product` entity updated with correct values, saved

**Integration tests (5):**
9. `StorefrontReviewControllerIT.postReview_validInput_returns201()` — POST `/api/rating/storefront/reviews` with valid body, authenticated user, verify `201 Created` response with correct `ReviewVm` body, review saved in database
10. `StorefrontReviewControllerIT.postReview_invalidRating_returns400()` — POST with `rating = 0`, verify `400 Bad Request` response with error message
11. `StorefrontReviewControllerIT.postReview_duplicateReview_returns409()` — POST same review twice (same `customerId`, `productId`), verify second request returns `409 Conflict`
12. `StorefrontReviewControllerIT.postReview_aggregateUpdateSuccess_productUpdated()` — POST review, verify `Product` table's `averageRating` and `reviewCount` updated (use real database with Testcontainers PostgreSQL, mock or use test instance of `product` service)
13. `StorefrontReviewControllerIT.getReviews_returnsReviewsOrderedByCreatedOn()` — POST 3 reviews, GET `/api/rating/storefront/reviews?productId=...`, verify reviews returned in `createdOn DESC` order, pagination metadata correct

**Migration verification (2):**
14. `LiquibaseMigrationIT.ratingUniqueConstraint_appliedCorrectly()` — verify `uq_customer_product` constraint exists on `rating` table, attempt to insert duplicate `(customerId, productId)` fails
15. `LiquibaseMigrationIT.productAggregateColumns_exist()` — verify `product` table has `average_rating` and `review_count` columns with correct types (DECIMAL, BIGINT)

### Story 2: Frontend review form component with star selector and content input

**Repo:** `app`

#### Context
This story builds the user-facing review submission form on the product detail page. It includes the star rating selector (1-5), review content textarea, client-side validation (non-empty content), submit button with disable-during-request logic, error message display, and success confirmation toast. It consumes the backend endpoint from Story 1. Without this story, customers cannot submit reviews from the UI.

#### Scope

### Story 2: Frontend review form component with star selector and content input › Scope

**Note:** The storefront appears to be a separate Next.js application not included in the `app` repo provided in the context. Story 2 and Story 3 belong in the **storefront repository**, which must be provided separately. The file paths will be determined once the storefront repo is available (likely `pages/products/[id].tsx` or `app/products/[id]/page.tsx` depending on Next.js version).

- **New files:**
  - `storefront/modules/review/components/ReviewForm.tsx` (form component: star selector, content textarea, submit button, validation, error/success messages)
  - `storefront/modules/review/services/ReviewService.ts` (API client: `postReview(productId, rating, content)` → calls `POST /api/rating/storefront/reviews`, `getReviews(productId, pageNo, pageSize)` → calls `GET /api/rating/storefront/reviews`)
  - `storefront/modules/review/models/Review.ts` (TypeScript types: `Review`, `ReviewPostRequest`, `ReviewListResponse`)
  - `storefront/modules/review/components/ReviewForm.test.tsx` (component tests for form validation, button disable, error display)
- **Modified files:**
  - `storefront/pages/products/[id].tsx` (or exact path — verify in repo: add `<ReviewForm productId={product.id} onSuccess={handleReviewSubmitted} />` component, `handleReviewSubmitted` callback refreshes reviews list and product data)

#### Implementation Pointers
- **ReviewForm.tsx:**
  - State: `rating: number | null`, `content: string`, `isSubmitting: boolean`, `error: string | null`, `success: boolean`
  - Star selector: render 5 clickable stars (UI library or custom SVG), set `rating` on click (1-5)
  - Content textarea: controlled input, `value={content}`, `onChange={e => setContent(e.target.value)}`, HTML5 `required` attribute
  - Validation (client-side, pre-submit):
    - If `rating === null`: display inline error "Please select a rating."
    - If `content.trim() === ''`: display inline error "Review content is required."
    - If both invalid, prevent submit (do not call API)
  - Submit button: `disabled={isSubmitting}` (prevents repeated clicks while request in flight)
  - On submit:
    1. Validate client-side (see above)
    2. Set `isSubmitting = true`, `error = null`, `success = false`
    3. Call `ReviewService.postReview(productId, rating, content)`
    4. On success (201 response):
       - Set `success = true`, display toast "Thank you! Your review has been submitted." (use existing toast library in repo, verify pattern)
       - Call `onSuccess()` callback (passed as prop from parent) → parent refreshes reviews list and product data
       - Reset form: `rating = null`, `content = ''`, `isSubmitting = false`
    5. On error:
       - 400: parse error response body, set `error = detail` (e.g. "Rating must be between 1 and 5.")
       - 409: set `error = "You have already submitted a review for this product."`
       - 500: set `error = "An error occurred. Please try again later."`
       - Set `isSubmitting = false`
  - Error display: if `error !== null`, render error message above form (red text, alert box, or existing pattern in repo)
- **ReviewService.ts:**
  - `postReview(productId: number, rating: number, content: string): Promise<Review>`:
    - Request: `POST /api/rating/storefront/reviews`, body `{productId, rating, content}`, include auth token (use existing auth mechanism in storefront, verify `fetch` or axios pattern)
    - Response: on `201 Created`, return `Review` object; on error, throw with status and error message from response body
  - `getReviews(productId: number, pageNo: number, pageSize: number): Promise<ReviewListResponse>`:
    - Request: `GET /api/rating/storefront/reviews?productId={productId}&pageNo={pageNo}&pageSize={pageSize}`, include auth token
    - Response: on `200 OK`, return `ReviewListResponse` object
- **Review.ts:**
  ```typescript
  export interface Review {
    id: number;
    customerId: string;
    rating: number;
    content: string;
    createdOn: string; // ISO 8601 timestamp
  }
  export interface ReviewPostRequest {
    productId: number;
    rating: number;
    content: string;
  }
  export interface ReviewListResponse {
    reviews: Review[];
    totalPages: number;
    totalElements: number;
    pageNo: number;
    pageSize: number;
  }
  ```
- **products/[id].tsx:**
  - Add state: `const [reviewsKey, setReviewsKey] = useState(0);` (force reviews list refresh)
  - Add callback: `const handleReviewSubmitted = () => { setReviewsKey(prev => prev + 1); /* also refetch product data to update header */ }`
  - Render `<ReviewForm productId={product.id} onSuccess={handleReviewSubmitted} />` (place above or within reviews section, verify UX placement with designer if needed)

#### Acceptance Criteria
1. Customers must be able to select a star rating between 1 and 5. (AC-1 — star selector UI enforces 1-5 selection)
2. Review content must be mandatory and must not contain only whitespace. (AC-2 — client-side validation prevents submit if content empty/whitespace-only)
3. The system must prevent submission if the rating or review content is missing. (AC-3 — client-side validation displays error if either missing)
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (AC-5 — success toast "Thank you! Your review has been submitted." displayed after `201 Created` response)
10. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (AC-10 — error message displayed on `400/409/500` response; no UI update on error)
11. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (AC-11 — submit button disabled while `isSubmitting = true`)

#### Dependencies
- **Story 1** — backend endpoint `POST /api/rating/storefront/reviews` must exist and return `201 Created` with `ReviewVm` body, or `400/409/500` on error.

#### Out of Scope
- Reviews section display (Story 3)
- Average rating and review count display in product header (Story 3)
- Backend validation (Story 1 owns this)
- Review persistence verification (Story 3 handles display after page refresh)

#### Verification Approach
**Test count:** 7 component tests

**Component tests (7):**
1. `ReviewForm.test.tsx: renders star selector and content textarea` — verify 5 stars rendered, textarea present with `required` attribute
2. `ReviewForm.test.tsx: star selector sets rating on click` — click star 3, verify `rating` state = 3, 3 stars highlighted
3. `ReviewForm.test.tsx: client-side validation prevents submit if rating missing` — leave rating unselected, fill content, click submit, verify error "Please select a rating." displayed, API not called
4. `ReviewForm.test.tsx: client-side validation prevents submit if content empty` — select rating, leave content empty, click submit, verify error "Review content is required." displayed, API not called
5. `ReviewForm.test.tsx: submit button disabled during request` — fill valid form, click submit, verify button `disabled` immediately (before API response), mock API delay, verify button remains disabled
6. `ReviewForm.test.tsx: success toast displayed on 201 response` — fill valid form, mock `postReview` returns `201 Created`, click submit, verify toast "Thank you! Your review has been submitted." displayed, `onSuccess` callback called
7. `ReviewForm.test.tsx: error message displayed on 409 response` — fill valid form, mock `postReview` returns `409 Conflict`, click submit, verify error "You have already submitted a review for this product." displayed, form not reset

---

### Story 3: Reviews section display updates

**Repo:** `app`

#### Context
This story completes the feature by updating the product detail page's Reviews section to display the list of submitted reviews (paginated, newest first), and updating the product header to display the aggregate rating and review count (fetched from the product service). It ensures consistency between the header and Reviews section (AC-9) and verifies persistence after page refresh (AC-12).

#### Scope
- **New files:**
  - `storefront/modules/review/components/ReviewsList.tsx` (component: renders paginated list of reviews, each showing rating stars, content, timestamp; includes pagination controls)
  - `storefront/modules/review/components/ReviewsList.test.tsx` (component tests for review rendering, pagination)
- **Modified files:**
  - `storefront/pages/products/[id].tsx` (add `<ReviewsList productId={product.id} key={reviewsKey} />` component, fetch product data on mount/refresh to update header with `averageRating` and `reviewCount`)
  - `storefront/modules/catalog/components/ProductDetailHeader.tsx` (or similar — verify exact component in repo that displays product name/price/rating: update to display `product.averageRating` as star rating graphic + numeric value, `product.reviewCount` as "(X reviews)" text)

#### Implementation Pointers
- **ReviewsList.tsx:**
  - Props: `productId: number`, `key?: number` (from parent, forces re-fetch when key changes)
  - State: `reviews: Review[]`, `pageNo: number`, `totalPages: number`, `loading: boolean`, `error: string | null`
  - On mount (or key change):
    1. Set `loading = true`, `error = null`
    2. Call `ReviewService.getReviews(productId, pageNo, pageSize)` (default `pageSize = 10`)
    3. On success: set `reviews = response.reviews`, `totalPages = response.totalPages`, `loading = false`
    4. On error: set `error = "Failed to load reviews."`, `loading = false`
  - Render:
    - If `loading`, show loading spinner
    - If `error`, show error message
    - If `reviews.length === 0`, show "No reviews yet. Be the first to review!" message
    - Else, map `reviews` to list items:
      - Each item: render rating as star graphic (filled stars count = review.rating), content text, timestamp (formatted as "MM/DD/YYYY" or relative time)
    - Pagination controls: "Previous" button (disabled if `pageNo === 0`), page indicator "Page X of Y", "Next" button (disabled if `pageNo === totalPages - 1`); on click, update `pageNo` state and re-fetch
  - Sorting: reviews already ordered by `createdOn DESC` from backend (Story 1 endpoint), no client-side sorting needed
- **products/[id].tsx:**
  - On mount and after review submission (`handleReviewSubmitted` callback from Story 2):
    1. Fetch product data: `ProductService.getProduct(productId)` (verify exact endpoint in `product` service, e.g. `GET /api/product/storefront/products/{productId}`)
    2. Update state: `setProduct(productData)` (includes `averageRating`, `reviewCount`)
  - Render `<ReviewsList productId={product.id} key={reviewsKey} />` (place in Reviews section, below `<ReviewForm />`)
- **ProductDetailHeader.tsx:**
  - Accept `averageRating?: number`, `reviewCount?: number` as props (or access from `product` object passed to component)
  - Render rating display:
    - Star graphic: render `averageRating` rounded to nearest 0.5 (e.g. 4.3 → 4.5 stars filled) or exact value with half-star icon if supported
    - Numeric value: `{averageRating.toFixed(1)}` (e.g. "4.3")
    - Review count: `({reviewCount} reviews)` or `({reviewCount} review)` if `reviewCount === 1`
  - If `averageRating` or `reviewCount` is null/undefined (no reviews yet), display "No reviews yet" or hide rating section

#### Acceptance Criteria
6. The newly submitted review must appear in the product's Reviews section with the correct rating and review content. (AC-6 — `<ReviewsList />` fetches and displays reviews, including newly submitted one after parent's `reviewsKey` changes)
9. The rating and review count displayed in the product header must match the Reviews section. (AC-9 — both read from `product.averageRating` and `product.reviewCount` fetched from product service, single source of truth)
12. Submitted reviews must remain available after refreshing or revisiting the product page. (AC-12 — on page load, `products/[id].tsx` fetches product data and reviews from backend; reviews persisted in database, displayed on every load)

#### Dependencies
- **Story 1** — backend endpoints `GET /api/rating/storefront/reviews` and `GET /api/product/storefront/products/{productId}` (or equivalent) must exist and return correct data.
- **Story 2** — `ReviewForm` component's `onSuccess` callback triggers reviews list refresh (via `reviewsKey` state change).

#### Out of Scope
- Review submission form (Story 2 owns this)
- Backend validation and persistence (Story 1 owns this)
- Review editing or deletion (not in this feature)
- Customer name display (deferred decision — if needed, requires `customer` service integration to resolve `customerId` to customer name)

#### Verification Approach
**Test count:** 6 component tests + 1 E2E test

**Component tests (6):**
1. `ReviewsList.test.tsx: renders loading spinner while fetching` — mock `getReviews` with delay, verify spinner displayed
2. `ReviewsList.test.tsx: displays error message on fetch failure` — mock `getReviews` returns error, verify "Failed to load reviews." displayed
3. `ReviewsList.test.tsx: displays "No reviews yet" if reviews array empty` — mock `getReviews` returns `{reviews: [], totalPages: 0}`, verify message displayed
4. `ReviewsList.test.tsx: renders reviews with correct rating and content` — mock `getReviews` returns 2 reviews (rating 4, 5), verify 2 list items rendered, correct star count and content text displayed
5. `ReviewsList.test.tsx: pagination buttons update pageNo and refetch` — mock `getReviews` returns `{reviews: [...], totalPages: 3, pageNo: 0}`, click "Next", verify `pageNo` state = 1, `getReviews` called again with `pageNo=1`
6. `ProductDetailHeader.test.tsx: displays averageRating and reviewCount correctly` — render component with `averageRating={4.3}`, `reviewCount={42}`, verify "4.3" displayed, "(42 reviews)" displayed, star graphic shows 4.3 rating

**E2E test (1):**
7. `review-submission.feature: full user flow` — navigate to product detail page, fill review form (rating 5, content "Great product!"), submit, verify confirmation toast, verify new review appears in Reviews section with rating 5 and content "Great product!", verify product header shows updated average rating and review count, refresh page, verify review still displayed (persistence check)

---

## Cross-Story Dependencies

None — stories are sequenced (1 → 2 → 3) with no parallelism. Story 2 consumes Story 1's backend endpoint; Story 3 consumes Story 1's retrieval endpoint and Story 2's refresh trigger.

---

## Risks

1. **Product service aggregate update failure (Story 1):** If the `product` service is down or slow during review submission, the review will be saved but the aggregate statistics may not update. This creates temporary inconsistency between the saved review and the displayed statistics.
   - **Mitigation:** Add retry logic or circuit breaker (Spring Cloud Resilience4j pattern already in use — verify in `AbstractCircuitBreakFallbackHandler`), or accept eventual consistency with a background reconciliation job (deferred decision).
   - **Affects:** Story 1 (backend), Story 3 (display may show stale statistics until manual reconciliation).

2. **Customer name display (Story 3):** The current design stores only `customerId` in the review. If the UI should display customer names, the frontend must call the `customer` service to resolve names (or the review retrieval endpoint must join/enrich with customer names). This is unspecified and may require additional work.
   - **Mitigation:** Clarify UX requirements before Story 3 implementation. If names are required, add a separate sub-task to integrate with `customer` service.
   - **Affects:** Story 3 (UI may display only `customerId` as placeholder; customer name resolution is deferred).

3. **Large review content (Story 1):** No max length currently defined. If users submit very large text (e.g. > 10KB), it may cause performance or storage issues.
   - **Mitigation:** Define and enforce a max length (e.g. 5000 characters) in requirements and Story 1's backend validation.
   - **Affects:** Story 1 (validation logic), Story 2 (client-side validation and textarea `maxLength` attribute).

4. **Average rating precision (Story 1, Story 3):** Rounding to one decimal place (e.g. `4.3`) may cause minor discrepancies if users expect higher precision.
   - **Mitigation:** Document the rounding rule (average rating = `SUM(star) / COUNT(*)`, rounded to one decimal place) and ensure consistency across all display locations (Story 3).
   - **Affects:** Story 1 (calculation logic), Story 3 (display formatting).

5. **Concurrent submission race (Story 1):** Two simultaneous requests from the same user might both pass uniqueness checks before the database constraint is enforced.
   - **Mitigation:** The database unique constraint `(customerId, productId)` is the final arbiter; one request will succeed, the other will fail with 409. No additional locking needed.
   - **Affects:** Story 1 (error handling must map `DataIntegrityViolationException` to 409 response).

## Open Questions

1. **Does the `product` service's `Product` entity already have `averageRating` and `reviewCount` columns?**
   - **Impact:** If missing, Story 1 must add a Liquibase migration to create them.
   - **Resolution needed before:** Story 1 implementation starts.

2. **Does the `rating` service's `Rating` table already have a unique constraint on `(customerId, productId)`?**
   - **Impact:** If missing, Story 1 must add a Liquibase migration to create it.
   - **Resolution needed before:** Story 1 implementation starts.

3. **What is the exact endpoint for updating product aggregates in the `product` service?**
   - **Impact:** If it does not exist, Story 1 must add it. If it exists but has a different signature, Story 1 must adapt.
   - **Resolution needed before:** Story 1 implementation starts.

4. **Should the review retrieval endpoint include customer names, or is `customerId` sufficient?**
   - **Impact:** If names are required, Story 3 must integrate with `customer` service (additional API call or backend join).
   - **Resolution needed before:** Story 3 implementation starts.

5. **Is there a maximum length for review content?**
   - **Impact:** Story 1 (backend validation) and Story 2 (client-side validation, textarea `maxLength`) need the limit.
   - **Resolution needed before:** Story 1 and Story 2 implementation starts.

6. **What is the exact path and component name for the product detail page in the storefront?**
   - **Impact:** Story 2 and Story 3 modify this file; incorrect path will cause integration issues.
   - **Resolution needed before:** Story 2 implementation starts.

7. **What is the exact component name and location for the product header that displays rating and review count?**
   - **Impact:** Story 3 modifies this component; incorrect path will cause display inconsistency.
   - **Resolution needed before:** Story 3 implementation starts.

---

**End of Breakdown**
