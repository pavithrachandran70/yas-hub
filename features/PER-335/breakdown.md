# breakdown.md

**Ticket:** Customer Can write and Submit Product Reviews  
**Epic:** _(none)_  
**Initiative:** _(none)_

---

## Summary

Customers submit product reviews (1–5 star rating and content) from the product detail page. The system validates input, saves the review, updates the product's average rating and review count, prevents duplicates, and displays the review persistently.

---

## Stories

### Story 1: Backend Review Submission API and Database Schema

**Repo:** `app`

#### Context

Create the new `rating` microservice to store and serve product reviews. This story establishes the backend foundation: REST API endpoint for review creation, database schema (review records and denormalized aggregates), validation logic (rating range 1–5, non-empty content), product existence verification via the `product` service, aggregate calculation (average rating and review count), and idempotency enforcement (one review per customer per product). Without this backend, the frontend cannot submit or display reviews.

#### Scope

- **New:** `rating/pom.xml` — Maven POM inheriting from parent POM
- **New:** `rating/Dockerfile` — Docker image definition
- **New:** `rating/src/main/java/com/yas/rating/RatingApplication.java` — Spring Boot main class
- **New:** `rating/src/main/java/com/yas/rating/config/DatabaseAutoConfig.java` — JPA config
- **New:** `rating/src/main/java/com/yas/rating/config/RestClientConfig.java` — REST client config for inter-service calls
- **New:** `rating/src/main/java/com/yas/rating/config/SecurityConfig.java` — Spring Security config (Keycloak JWT authentication)
- **New:** `rating/src/main/java/com/yas/rating/config/SwaggerConfig.java` — Swagger API docs config
- **New:** `rating/src/main/java/com/yas/rating/controller/ReviewController.java` — REST controller for review endpoints (`POST /api/rating/storefront/products/{productId}/reviews`, `GET /api/rating/storefront/products/{productId}/reviews`)
- **New:** `rating/src/main/java/com/yas/rating/service/ReviewService.java` — Business logic: validation, persistence, aggregate updates
- **New:** `rating/src/main/java/com/yas/rating/service/ProductService.java` — Circuit-breaker-wrapped client for product service (verify product exists)
- **New:** `rating/src/main/java/com/yas/rating/service/AggregateService.java` — Aggregate calculation and persistence
- **New:** `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java` — JPA repository for `Review` entity
- **New:** `rating/src/main/java/com/yas/rating/repository/ProductAggregateRepository.java` — JPA repository for `ProductAggregate` entity
- **New:** `rating/src/main/java/com/yas/rating/model/Review.java` — JPA entity mapping `review` table
- **New:** `rating/src/main/java/com/yas/rating/model/ProductAggregate.java` — JPA entity mapping `product_aggregate` table
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java` — DTO for review creation request (`rating`, `content`)
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java` — DTO for review response (`id`, `productId`, `customerId`, `rating`, `content`, `createdAt`)
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewListVm.java` — DTO for review list response (wraps list of `ReviewVm` + `averageRating`, `totalReviews`)
- **New:** `rating/src/main/resources/application.properties` — Spring Boot config (database URL, Keycloak config, service port 8095)
- **New:** `rating/src/main/resources/logback-spring.xml` — Logging config
- **New:** `rating/src/main/resources/db/changelog/db.changelog-master.yaml` — Liquibase master changelog
- **New:** `rating/src/main/resources/db/changelog/ddl/changelog-0001.sql` — Liquibase changeset to create `review` and `product_aggregate` tables
- **New:** `rating/src/main/resources/messages/messages.properties` — Internationalization messages (error messages)
- **New:** `rating/src/test/java/com/yas/rating/controller/ReviewControllerTest.java` — Unit tests for `ReviewController`
- **New:** `rating/src/test/java/com/yas/rating/service/ReviewServiceTest.java` — Unit tests for `ReviewService`
- **New:** `rating/src/test/resources/application.properties` — Test-specific Spring Boot config
- **Modified:** `pom.xml` — Add `<module>rating</module>` to `<modules>` section
- **Modified:** `docker-compose.yml` — Add `rating` service container
- **Modified:** `postgres_init.sql` — Add `CREATE DATABASE rating;`

#### Implementation Pointers

- `rating/src/main/java/com/yas/rating/controller/ReviewController.java`:
  - Add method `createReview(@PathVariable Long productId, @RequestBody @Valid ReviewPostVm reviewPostVm)` — returns `ResponseEntity<ReviewVm>` with 201 status
  - Add method `getReviewsByProductId(@PathVariable Long productId)` — returns `ResponseEntity<ReviewListVm>` with 200 status
- `rating/src/main/java/com/yas/rating/service/ReviewService.java`:
  - Add method `createReview(Long productId, ReviewPostVm reviewPostVm)` — extracts `customerId` from JWT token using `AuthenticationUtils.extractUserId()`, validates rating and content, calls `ProductService.verifyProductExists(productId)`, saves `Review` entity, calls `AggregateService.updateAggregatesAfterReviewCreated(productId, rating)`, returns `ReviewVm`
  - Add method `getReviewsByProductId(Long productId)` — queries `ReviewRepository.findByProductIdOrderByCreatedAtDesc(productId)`, fetches aggregates from `ProductAggregateRepository.findById(productId)`, returns `ReviewListVm`
- `rating/src/main/java/com/yas/rating/service/ProductService.java`:
  - Add method `verifyProductExists(Long productId)` — calls `GET /api/product/storefront/products/{productId}` via `RestClient`, throws `NotFoundException` if product not found
- `rating/src/main/java/com/yas/rating/service/AggregateService.java`:
  - Add method `updateAggregatesAfterReviewCreated(Long productId, Integer newRating)` — fetches or creates `ProductAggregate` row, recalculates `averageRating` and increments `reviewCount`, saves via `ProductAggregateRepository.save()`
- `rating/src/main/java/com/yas/rating/model/Review.java`:
  - Extends `com.yas.commonlibrary.model.AbstractAuditEntity` for audit fields
  - Fields: `id` (Long, primary key), `productId` (Long), `customerId` (String, 255), `rating` (Integer, constraint 1–5), `content` (String, TEXT)
  - Unique constraint on `(productId, customerId)` via `@Table(uniqueConstraints = ...)`
- `rating/src/main/java/com/yas/rating/model/ProductAggregate.java`:
  - Fields: `productId` (Long, primary key), `averageRating` (BigDecimal, precision 3 scale 2), `reviewCount` (Integer), `updatedAt` (Instant)
- `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java`:
  - Extends `JpaRepository<Review, Long>`
  - Add method `List<Review> findByProductIdOrderByCreatedAtDesc(Long productId)`
- `rating/src/main/java/com/yas/rating/repository/ProductAggregateRepository.java`:
  - Extends `JpaRepository<ProductAggregate, Long>`
- `rating/src/main/resources/db/changelog/ddl/changelog-0001.sql`:
  - Create `review` table with columns `id`, `product_id`, `customer_id`, `rating`, `content`, `created_at`, `updated_at`, unique constraint `unique_customer_product_review` on `(product_id, customer_id)`, indexes on `product_id` and `customer_id`
  - Create `product_aggregate` table with columns `product_id` (primary key), `average_rating`, `review_count`, `updated_at`
- `rating/src/main/java/com/yas/rating/config/SecurityConfig.java`:
  - Configure Keycloak JWT authentication, require authentication for `POST /storefront/products/{productId}/reviews`, permit GET without authentication
- `rating/src/main/java/com/yas/rating/config/RestClientConfig.java`:
  - Configure `RestClient` bean for inter-service calls to `product` service

#### Acceptance Criteria

1. Customers must be able to select a star rating between 1 and 5. (AC 1)
2. Review content must be mandatory and must not contain only whitespace. (AC 2)
3. The system must prevent submission if the rating or review content is missing. (AC 3)
4. The system must reject invalid ratings, including values below 1 or above 5, through API validation. (AC 4)
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (AC 5 — backend saves review and returns success response)
6. The product's average rating must be recalculated based on all valid reviews. (AC 7)
7. The total review count must increase by one after successful submission. (AC 8)
8. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (AC 10 — backend returns 400/404/409/500 responses)
9. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (AC 11 — backend enforces unique constraint on `(product_id, customer_id)`)
10. Submitted reviews must remain available after refreshing or revisiting the product page. (AC 12 — backend persists reviews in database)

#### Dependencies

None (foundational backend story).

#### Out of Scope

- Frontend UI for review submission (covered in Story 2).
- Frontend display of reviews and aggregates (covered in Story 2).
- Review editing or deletion by customers (not in ticket scope).
- Review moderation workflow (not in ticket scope; reviews are immediately visible).
- Backoffice endpoints for admin review management (covered in Story 3).

#### Verification Approach

**10 tests** (unit tests for controller and service, integration test for full flow):

1. `ReviewControllerTest.testCreateReview_Success` — POST with valid rating and content returns 201 and `ReviewVm`
2. `ReviewControllerTest.testCreateReview_MissingRating` — POST with no rating returns 400 with field error
3. `ReviewControllerTest.testCreateReview_InvalidRating` — POST with rating 0 or 6 returns 400 with field error
4. `ReviewControllerTest.testCreateReview_MissingContent` — POST with empty content returns 400 with field error
5. `ReviewControllerTest.testCreateReview_ProductNotFound` — POST for non-existent product returns 404
6. `ReviewServiceTest.testCreateReview_DuplicateReview` — Second review from same customer for same product throws `DuplicatedException`, caught and returns 409
7. `ReviewServiceTest.testUpdateAggregates_FirstReview` — Creating first review sets `average_rating` = rating and `review_count` = 1
8. `ReviewServiceTest.testUpdateAggregates_SubsequentReview` — Creating second review recalculates `average_rating` correctly and increments `review_count`
9. `ReviewControllerIT.testCreateReviewAndFetch_Persistence` — POST review, restart service, GET reviews — review is still present (integration test with database)
10. `ReviewControllerTest.testGetReviewsByProductId_Success` — GET reviews for existing product returns 200 and `ReviewListVm` with correct aggregates

### Story 2: Frontend Review Submission Form and Display

**Repo:** `app`

#### Context

Add the UI for customers to submit reviews (star rating selector, text input for content, submit button) on the product detail page, and display the submitted reviews list with updated aggregates (average rating, review count) in the product header and Reviews section. This story connects the frontend to the backend API created in Story 1, implements client-side validation (disable submit until fields valid, prevent double-clicks), and handles success/error responses (toast notifications, error messages).

#### Scope

- **New:** `storefront/pages/products/[slug]/reviews.tsx` — Product review submission form and display (or inline in existing product detail page)
- **New:** `storefront/modules/product/components/ReviewForm.tsx` — Form component for star rating selection, content input, and submission
- **New:** `storefront/modules/product/components/ReviewList.tsx` — Display list of reviews (rating, content, timestamp)
- **New:** `storefront/modules/product/services/ReviewService.ts` — API client wrapper for `POST /api/rating/storefront/products/{productId}/reviews` and `GET /api/rating/storefront/products/{productId}/reviews`
- **New:** `storefront/modules/product/models/Review.ts` — TypeScript type for review data (`id`, `productId`, `customerId`, `rating`, `content`, `createdAt`)
- **Modified:** `storefront/pages/products/[slug].tsx` — Integrate `ReviewForm` and `ReviewList` components (verify file exists; assumed based on backoffice structure)
- **Modified:** `storefront/modules/product/components/ProductHeader.tsx` — Display updated `averageRating` and `reviewCount` from API response (verify file exists)

#### Implementation Pointers

- `storefront/modules/product/services/ReviewService.ts`:
  - Add function `createReview(productId: number, rating: number, content: string)` — calls `POST /api/rating/storefront/products/{productId}/reviews` with body `{ rating, content }`, returns promise resolving to `ReviewVm`
  - Add function `getReviews(productId: number)` — calls `GET /api/rating/storefront/products/{productId}/reviews`, returns promise resolving to `ReviewListVm`
- `storefront/modules/product/components/ReviewForm.tsx`:
  - Use `react-hook-form` for form state (observed in `backoffice/common/items/Input.tsx`, `backoffice/modules/catalog/components/ProductGeneralInformation.tsx`)
  - Add star rating input (1–5) — controlled component, disable submit button until rating selected
  - Add textarea for review content — validate non-empty and non-whitespace, max 1000 characters
  - Add submit button — disable while request in-flight (prevent double-clicks)
  - On submit: call `ReviewService.createReview(productId, rating, content)`, show success toast (`react-toastify`, observed in `backoffice/common/services/ToastService.ts`), clear form, trigger refetch of reviews
  - On error: display error message from API response (validation errors, product not found, duplicate review, server error)
- `storefront/modules/product/components/ReviewList.tsx`:
  - Fetch reviews on mount via `ReviewService.getReviews(productId)`
  - Display list of reviews: rating (stars), content, timestamp
  - Display aggregates: `averageRating` (e.g. "4.5 ★"), `totalReviews` (e.g. "(12 reviews)")
- `storefront/pages/products/[slug].tsx`:
  - Integrate `ReviewForm` and `ReviewList` components in product detail page
  - Pass `productId` prop to both components
- `storefront/modules/product/components/ProductHeader.tsx`:
  - Fetch aggregates from `ReviewService.getReviews(productId)` on mount
  - Display `averageRating` and `reviewCount` in product header (verify existing structure)

#### Acceptance Criteria

1. The newly submitted review must appear in the product's Reviews section with the correct rating and review content. (AC 6)
2. The rating and review count displayed in the product header must match the Reviews section. (AC 9)
3. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (AC 10 — frontend displays error toast, does not update local state)
4. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (AC 11 — frontend disables submit button while request in-flight)
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (AC 5 — frontend shows success toast)

#### Dependencies

- **Story 1** — Backend API endpoints must exist (`POST /api/rating/storefront/products/{productId}/reviews`, `GET /api/rating/storefront/products/{productId}/reviews`).

#### Out of Scope

- Backend validation logic (covered in Story 1).
- Review editing or deletion by customers (not in ticket scope).
- Review pagination (all reviews displayed; pagination can be added later).
- Backoffice UI for admin review management (covered in Story 3).

#### Verification Approach

**6 tests** (frontend unit tests and E2E tests):

1. `ReviewForm.test.tsx` — Render form, verify star rating selector and content textarea are present
2. `ReviewForm.test.tsx` — Submit button is disabled until rating selected and content non-empty
3. `ReviewForm.test.tsx` — Clicking submit with valid data calls `ReviewService.createReview` and shows success toast
4. `ReviewForm.test.tsx` — Clicking submit with invalid data (empty content) does not call API
5. `ReviewList.test.tsx` — Fetches reviews on mount and displays list with correct rating and content
6. E2E test (Playwright or Cypress) — Navigate to product detail page, submit review, verify review appears in Reviews section and aggregates are updated in product header

---

### Story 3: Backoffice Review Management Endpoints

**Repo:** `app`

#### Context

The existing backoffice UI (`backoffice/pages/reviews/index.tsx`) expects backend endpoints for listing and deleting reviews. This story implements the missing backend endpoints (`GET /api/rating/backoffice/reviews` for paginated review list with search filters, `DELETE /api/rating/backoffice/reviews/{id}` for admin deletion) to support the existing UI. Without these endpoints, the backoffice review management page cannot function.

#### Scope

- **New:** `rating/src/main/java/com/yas/rating/controller/BackofficeReviewController.java` — REST controller for backoffice review endpoints
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewListGetVm.java` — DTO for paginated review list response (matches structure expected by `backoffice/pages/reviews/index.tsx`)
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewSearchCriteria.java` — DTO for search query parameters (`pageNo`, `pageSize`, `createdFrom`, `createdTo`)
- **Modified:** `rating/src/main/java/com/yas/rating/service/ReviewService.java` — Add methods `getReviews(ReviewSearchCriteria criteria)` and `deleteReviewById(Long id)`
- **Modified:** `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java` — Add custom query method for search with pagination
- **Modified:** `rating/src/main/java/com/yas/rating/service/AggregateService.java` — Add method `updateAggregatesAfterReviewDeleted(Long productId, Integer deletedRating)` to recalculate aggregates on deletion
- **Modified:** `rating/src/main/java/com/yas/rating/config/SecurityConfig.java` — Require `ROLE_ADMIN` for `/backoffice/reviews/**`
- **New:** `rating/src/test/java/com/yas/rating/controller/BackofficeReviewControllerTest.java` — Unit tests for backoffice endpoints
- **Modified:** `backoffice/modules/rating/services/RatingService.ts` — Verify `getRatings` and `deleteRatingById` methods call the new backend endpoints (may already exist; check `repo-file/app/backoffice/modules/rating/services/RatingService.ts` if available)

#### Implementation Pointers

- `rating/src/main/java/com/yas/rating/controller/BackofficeReviewController.java`:
  - Add method `getReviews(@RequestParam Map<String, String> params)` — returns `ResponseEntity<ReviewListGetVm>` with 200 status
  - Add method `deleteReview(@PathVariable Long id)` — returns `ResponseEntity<Void>` with 204 status
- `rating/src/main/java/com/yas/rating/service/ReviewService.java`:
  - Add method `getReviews(ReviewSearchCriteria criteria)` — queries `ReviewRepository` with pagination and filters, fetches customer names from `customer` service (call `GET /api/customer/backoffice/customers/{customerId}`), fetches product names from `product` service (call `GET /api/product/storefront/products/{productId}`), returns `ReviewListGetVm`
  - Add method `deleteReviewById(Long id)` — fetches `Review` by id, deletes via `ReviewRepository.deleteById(id)`, calls `AggregateService.updateAggregatesAfterReviewDeleted(productId, deletedRating)`
- `rating/src/main/java/com/yas/rating/service/AggregateService.java`:
  - Add method `updateAggregatesAfterReviewDeleted(Long productId, Integer deletedRating)` — fetches `ProductAggregate` row, recalculates `averageRating` as `((old_avg * old_count) - deleted_rating) / (old_count - 1)`, decrements `review_count`, saves via `ProductAggregateRepository.save()`
- `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java`:
  - Add method `Page<Review> findByCreatedAtBetween(Instant createdFrom, Instant createdTo, Pageable pageable)` (custom query for search filters)
- `rating/src/main/java/com/yas/rating/config/SecurityConfig.java`:
  - Configure security: require `ROLE_ADMIN` for `POST /backoffice/reviews/**`, `DELETE /backoffice/reviews/{id}`

#### Acceptance Criteria

1. The backoffice review list page (`backoffice/pages/reviews/index.tsx`) must fetch paginated reviews from `GET /api/rating/backoffice/reviews` with query parameters `pageNo`, `pageSize`, `createdFrom`, `createdTo`. (No explicit ticket AC, but required for existing UI)
2. The backoffice review list page must display customer name (`firstName`, `lastName`), product name, rating, content, and created date for each review. (No explicit ticket AC, but required for existing UI)
3. Admins must be able to delete a review via the "Delete" button, which calls `DELETE /api/rating/backoffice/reviews/{id}`. (No explicit ticket AC, but required for existing UI)
4. After a review is deleted, the product's average rating and review count must be recalculated. (Implied by AC 7, AC 8)
5. Deleted reviews must not appear in the review list after deletion. (Standard delete behavior)

#### Dependencies

- **Story 1** — Backend `Review` and `ProductAggregate` entities must exist; `ReviewRepository` and `AggregateService` must be implemented.

#### Out of Scope

- Customer-facing review submission and display (covered in Story 1 and Story 2).
- Review moderation workflow (approval/rejection by admins; not in ticket scope).
- Review editing (not in ticket scope).

#### Verification Approach

**5 tests** (unit tests for backoffice controller and service):

1. `BackofficeReviewControllerTest.testGetReviews_Success` — GET with valid pagination params returns 200 and `ReviewListGetVm`
2. `BackofficeReviewControllerTest.testGetReviews_WithFilters` — GET with `createdFrom` and `createdTo` filters returns filtered results
3. `BackofficeReviewControllerTest.testDeleteReview_Success` — DELETE existing review returns 204
4. `BackofficeReviewControllerTest.testDeleteReview_NotFound` — DELETE non-existent review returns 404
5. `ReviewServiceTest.testDeleteReview_AggregatesUpdated` — Deleting a review recalculates `average_rating` and decrements `review_count`

---

## Cross-Story Dependencies

None (stories are ordered sequentially: Story 1 provides backend, Story 2 consumes it, Story 3 extends backend for backoffice).

---

## Risks

### 1. Customer Identity Extraction from JWT Token

**Risk:** The `customer_id` field in the `review` table is populated from `AuthenticationUtils.extractUserId()`, which assumes the JWT token contains a user ID claim. If the token structure does not include this claim, the service will fail.

**Mitigation:** Verify JWT token structure in `common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java`. If the token does not contain `customer_id`, call the `customer` service to fetch it via `GET /api/customer/storefront/customer/profile`.

---

### 2. Product Existence Validation

**Risk:** The `rating` service must verify that `product_id` exists before saving a review. This requires calling the `product` service. If the `product` service is unavailable or the product does not exist, the review creation fails.

**Mitigation:** Use the existing `ProductService` pattern (observed in `cart/src/main/java/com/yas/cart/service/ProductService.java`, `customer/src/main/java/com/yas/customer/service/ProductService.java`, `order/src/main/java/com/yas/order/service/ProductService.java`). Call `GET /api/product/storefront/products/{id}` to verify product existence. Return `404 Not Found` to the client if the product does not exist. Implement circuit breaker (Resilience4j, observed in `AbstractCircuitBreakFallbackHandler` in `cart`, `customer`, `order` services) to handle transient failures of the `product` service.

---

### 3. Aggregate Update Race Condition

**Risk:** If two reviews for the same product are submitted concurrently, the aggregate update (average rating, review count) could race, leading to incorrect values.

**Mitigation:** Wrap the aggregate update in a database transaction (already specified in `AggregateService.updateAggregatesAfterReviewCreated`). Use row-level locking: `SELECT FOR UPDATE` on the `product_aggregate` row before updating. Alternative: use atomic SQL updates:
```sql
UPDATE product_aggregate
SET average_rating = ((average_rating * review_count) + ?) / (review_count + 1),
    review_count = review_count + 1
WHERE product_id = ?;
```
Implement optimistic locking (`@Version` column in `ProductAggregate` entity) to detect and retry conflicts.

---

### 4. Review Content Abuse

**Risk:** Customers could submit very long content or malicious HTML/JS.

**Mitigation:** Enforce a maximum content length (e.g. 1000 characters) at the API level using `@Size(max = 1000)` on the `ReviewPostVm.content` field. Sanitize content before storing (strip HTML tags, escape special characters) or store as plain text and escape on display (handled in frontend Story 2).

---

### 5. Frontend Cache Invalidation After Review Submission

**Risk:** After a review is submitted, the product detail page must fetch the updated reviews list and aggregates (average rating, review count). If the frontend caches the product data, stale data may be displayed.

**Mitigation:** After a successful `POST /products/{id}/reviews` response, refetch the reviews list via `GET /products/{id}/reviews` and update the displayed aggregates. Invalidate any cached product data on the frontend.

---

### 6. Database Constraint Violation Handling (Duplicate Review)

**Risk:** If the unique constraint on `(product_id, customer_id)` is violated (duplicate review), the database throws a `ConstraintViolationException`.

**Mitigation:** Catch `org.springframework.dao.DataIntegrityViolationException` in the service layer and return a 409 Conflict response with message "You have already reviewed this product."

---

### 7. Initial Aggregate Record Creation

**Risk:** When the first review is submitted for a product, the `product_aggregate` row does not exist yet.

**Mitigation:** Use `INSERT ... ON CONFLICT DO UPDATE` (PostgreSQL upsert) or `MERGE` (SQL standard) to atomically create or update the aggregate row. Example:
```sql
INSERT INTO product_aggregate (product_id, average_rating, review_count)
VALUES (?, ?, 1)
ON CONFLICT (product_id) DO UPDATE SET
  average_rating = ((product_aggregate.average_rating * product_aggregate.review_count) + EXCLUDED.average_rating) / (product_aggregate.review_count + 1),
  review_count = product_aggregate.review_count + 1,
  updated_at = CURRENT_TIMESTAMP;
```

---

## Open Questions

1. **Authentication Requirement:**  
   Must customers be authenticated (logged in) to submit a review, or can anonymous users submit reviews?  
   **Decision Required:** Ticket implies authentication (customer identity required for idempotency and display). Recommend requiring authentication.

2. **Review Moderation:**  
   Are reviews immediately visible after submission, or do they go through moderation?  
   **Decision Required:** Ticket implies immediate visibility. If moderation is needed, add a `status` field (e.g. `PENDING`, `APPROVED`, `REJECTED`) to the `review` table and filter by status in queries.

3. **Review Content Maximum Length:**  
   What is the maximum length for review content?  
   **Decision Required:** Set a reasonable limit (e.g. 1000 characters) to prevent abuse. Enforce at API level and database schema.

4. **Review Editing:**  
   Can customers edit their reviews after submission?  
   **Decision Required:** Not in scope for this ticket. If required, add `PUT /reviews/{id}` endpoint and update `updated_at` timestamp on edit. Recalculate aggregates if rating changes.

5. **Review Deletion by Customers:**  
   Can customers delete their own reviews?  
   **Decision Required:** Not in scope for this ticket. Admin deletion is covered in Story 3. If customer deletion is required, add customer-facing `DELETE /storefront/products/{productId}/reviews/{id}` endpoint.

6. **Pagination for Reviews:**  
   Should the reviews list be paginated?  
   **Decision Required:** Not specified in ticket. For initial implementation, return all reviews (up to a reasonable limit, e.g. 50). Add pagination parameters (`pageNo`, `pageSize`) later if needed.

---

## Notes

- The `rating` service does not currently exist in the YAS codebase. Story 1 creates a new microservice following the existing structure (Spring Boot, JPA, Liquibase, REST API).
- The backoffice UI (`backoffice/pages/reviews/index.tsx`) already exists and expects a reviews API. Story 3 implements the backend endpoints to support it.
- The API follows the YAS convention: `/api/<service>/storefront/<resource>` for customer-facing endpoints, `/api/<service>/backoffice/<resource>` for admin endpoints.
- Aggregate calculation uses denormalized `product_aggregate` table for fast reads (pre-computed average rating and count), updated transactionally after each review insert or delete.
- Idempotency is enforced by database unique constraint on `(product_id, customer_id)` — the simplest and most reliable approach.
