# Merged plan — PER-327

Execution order (1 ticket(s)):

1. `1`

Specs (1): `specs/1.md`

---

## Spec `specs/1.md`

# Tech Stub — Ticket 1

**Customers can write and submit product reviews**

---

## Stack

- **Backend:** Spring Boot 3.x, JPA/Hibernate, PostgreSQL, Liquibase
- **Frontend (Backoffice):** Next.js (TypeScript), React
- **Frontend (Storefront):** React (TypeScript)
- **Existing services:** `rating`, `product`, `customer`

---

## Schema / API Contracts

### Review Entity (Rating table)

**Location:** `rating/src/main/java/com/yas/rating/model/Rating.java`

**Fields:**
- `id` (Long, primary key, auto-generated)
- `productId` (Long, not null, indexed)
- `customerId` (String, not null, indexed)
- `star` (Integer, not null, range 1-5)
- `content` (String, not null)
- `createdOn` (ZonedDateTime, auto-populated)
- `lastModifiedOn` (ZonedDateTime, auto-updated)
- `createdBy` (String, audit field)
- `lastModifiedBy` (String, audit field)

**Constraints:**
- Unique constraint on `(customerId, productId)` to enforce one review per customer per product

**Migration required:** Verify existing Liquibase changelog at `rating/src/main/resources/db/changelog/ddl/` includes this schema. If the unique constraint or indexes on `(productId)` or `(customerId, productId)` are missing, add a new changelog.

---

### POST /api/rating/storefront/reviews

**Request Body:**
```json
{
  "productId": 123,
  "rating": 5,
  "content": "Excellent product!"
}
```

**Note:** `customerId` is extracted from the authenticated user's security context (via `AuthenticationUtils.extractUserId()`), not sent by the client.

**Success Response (201 Created):**
```json
{
  "id": 456,
  "productId": 123,
  "customerId": "user-uuid",
  "rating": 5,
  "content": "Excellent product!",
  "createdOn": "2025-05-23T14:35:00Z"
}
```

**Error Responses:**
- **400 Bad Request:** Invalid rating value (< 1 or > 5), missing or whitespace-only content, missing required fields
  ```json
  {
    "status": 400,
    "title": "Bad Request",
    "detail": "Rating must be between 1 and 5."
  }
  ```
- **401 Unauthorized:** No valid authentication token
- **409 Conflict:** Duplicate review (customer has already reviewed this product)
  ```json
  {
    "status": 409,
    "title": "Conflict",
    "detail": "You have already submitted a review for this product."
  }
  ```
- **500 Internal Server Error:** Database or aggregate calculation failure
  ```json
  {
    "status": 500,
    "title": "Internal Server Error",
    "detail": "An error occurred while processing your request."
  }
  ```

---

### GET /api/rating/storefront/reviews

**Query Parameters:**
- `productId` (required): Long
- `pageNo` (optional, default 0): Integer
- `pageSize` (optional, default 10): Integer

**Success Response (200 OK):**
```json
{
  "reviews": [
    {
      "id": 456,
      "customerId": "user-uuid",
      "rating": 5,
      "content": "Excellent product!",
      "createdOn": "2025-05-23T14:35:00Z"
    }
  ],
  "totalPages": 1,
  "totalElements": 1,
  "pageNo": 0,
  "pageSize": 10
}
```

**Sorting:** Reviews ordered by `createdOn DESC` (newest first).

---

### PUT /api/product/backoffice/products/{productId}/rating-aggregates

**Request Body:**
```json
{
  "averageRating": 4.3,
  "reviewCount": 42
}
```

**Success Response (200 OK):**
```json
{
  "productId": 123,
  "averageRating": 4.3,
  "reviewCount": 42
}
```

**Note:** This endpoint does not currently exist in the `product` service and must be added.

---

### Product Entity Updates

**Location:** `product/src/main/java/com/yas/product/model/Product.java`

**New fields required (verify if missing):**
```java
@Column(name = "average_rating")
private BigDecimal averageRating;

@Column(name = "review_count")
private Long reviewCount;
```

**Migration required:** If these columns do not exist, add a Liquibase changelog in the `product` service at `product/src/main/resources/db/changelog/ddl/`.

---

## File Touch List

### Backend (rating service)

**New files:**
- `rating/src/main/java/com/yas/rating/controller/StorefrontReviewController.java` (or add methods to existing controller if one exists)
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java` (request DTO)
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java` (response DTO)
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewListVm.java` (paginated response DTO)

**Modified files:**
- `rating/src/main/java/com/yas/rating/service/RatingService.java` (add `createReview`, `getReviewsByProductId`, `calculateAggregates` methods — verify exact class location in repo)
- `rating/src/main/resources/db/changelog/ddl/changelog-000X.sql` (add unique constraint + indexes if missing)

**Unit tests:**
- `rating/src/test/java/com/yas/rating/service/RatingServiceTest.java` (new tests for validation, duplicate detection, aggregate calculation)
- `rating/src/test/java/com/yas/rating/controller/StorefrontReviewControllerTest.java` (new tests for endpoint contract)

**Integration tests:**
- `rating/src/it/java/com/yas/rating/service/RatingServiceIT.java` (new tests for full flow)
- `rating/src/it/java/com/yas/rating/controller/StorefrontReviewControllerIT.java` (new tests for POST → save → GET retrieval)

---

### Backend (product service)

**New files:**
- `product/src/main/java/com/yas/product/controller/ProductAggregateController.java` (or add method to existing backoffice controller)
- `product/src/main/java/com/yas/product/viewmodel/RatingAggregatePostVm.java` (request DTO)

**Modified files:**
- `product/src/main/java/com/yas/product/model/Product.java` (add `averageRating`, `reviewCount` columns if missing — verify in repo)
- `product/src/main/java/com/yas/product/service/ProductService.java` (add `updateRatingAggregates` method — verify exact class location in repo)
- `product/src/main/resources/db/changelog/ddl/changelog-000X.sql` (add columns if missing)

**Unit tests:**
- `product/src/test/java/com/yas/product/service/ProductServiceTest.java` (new tests for aggregate update logic)

**Integration tests:**
- `product/src/it/java/com/yas/product/service/ProductServiceIT.java` (new tests for aggregate update flow)

---

### Frontend (storefront)

**New files:**
- `storefront/modules/review/components/ReviewForm.tsx` (form component with star selector, content input, submit button)
- `storefront/modules/review/services/ReviewService.ts` (API client for POST and GET review endpoints)
- `storefront/modules/review/models/Review.ts` (TypeScript types: `Review`, `ReviewPostRequest`, `ReviewListResponse`)

**Modified files:**
- `storefront/pages/products/[id].tsx` (or wherever product detail page lives — verify exact path in repo) — integrate `ReviewForm` component and reviews list display
- `storefront/modules/catalog/components/ProductDetailReviews.tsx` (or similar — verify in repo) — add review submission form + list rendering

**Component tests:**
- `storefront/modules/review/components/ReviewForm.test.tsx` (new tests for form validation, button disable, error message display)

**E2E tests (optional):**
- `automation-ui/storefront/src/test/resources/features/review-submission.feature` (Cucumber feature for full user flow)

---

## Risks

1. **Product service aggregate update failure:** If the `product` service is down or slow during review submission, the review will be saved but the aggregate statistics may not update. This creates temporary inconsistency.
   - **Mitigation:** Add retry logic or circuit breaker (Spring Cloud Resilience4j pattern already in use — verify in `AbstractCircuitBreakFallbackHandler`), or accept eventual consistency with a background reconciliation job (deferred decision).

2. **Concurrent submission race:** Two simultaneous requests from the same user might both pass uniqueness checks before the database constraint is enforced.
   - **Mitigation:** The database unique constraint `(customerId, productId)` is the final arbiter; one request will succeed, the other will fail with 409. No additional locking needed.

3. **Large review content:** No max length currently defined. If users submit very large text, it may cause performance or storage issues.
   - **Mitigation:** Define and enforce a max length (e.g. 5000 characters) in requirements and validation (open question).

4. **Average rating precision:** Rounding to one decimal place may cause minor discrepancies in display (e.g. `4.3` vs `4.33`).
   - **Mitigation:** Document the rounding rule (average rating = `SUM(star) / COUNT(*)`, rounded to one decimal place) and ensure consistency across all display locations.

5. **Missing aggregate update endpoint in product service:** The `PUT /api/product/backoffice/products/{productId}/rating-aggregates` endpoint does not currently exist and must be added. If this is not implemented, the review submission cannot update product statistics.
   - **Mitigation:** Prioritize this endpoint implementation in the product service as part of the same delivery cycle.

6. **Customer name display:** The current design stores only `customerId` in the review. If the UI should display customer names, the frontend must call the `customer` service to resolve names (or the review retrieval endpoint must join/enrich with customer names). This is unspecified and may require additional work.
   - **Mitigation:** Clarify UX requirements before frontend implementation (open question).

---

**End of Tech Stub**

---

# Story summaries

- `01-app`: Story 01-app LLD: Backend review submission endpoint with validation and rating recalculation
- `02-app`: Story 02-app LLD: Frontend review form component with star selector and content input
- `03-app`: Story 03-app LLD: Reviews section display updates

---

_Single-ticket pack: order is trivial (no multi-split)._
