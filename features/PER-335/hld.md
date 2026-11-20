# hld.md

**Ticket:** Customer Can write and Submit Product Reviews  
**Epic:** _(none)_  
**Initiative:** _(none)_

---

## Summary

Customers submit product reviews (1–5 star rating and content) from the product detail page. The system validates the review at the UI and API levels, stores it in the database, recalculates the product's average rating and total review count, prevents duplicate submissions, and displays the review persistently.

---

## Components

### 1. Rating Frontend UI (Next.js + TypeScript)

**Responsibility:**  
Render the review submission form on the product detail page; validate rating (1–5) and content (non-empty, no whitespace-only) client-side; submit reviews to the backend; display confirmation or error messages; display the updated reviews list and product aggregates.

**Files:**
- **New:** `storefront/pages/products/[slug]/reviews.tsx` — Product review submission form and display (or inline in the existing product detail page).
- **New:** `storefront/modules/product/components/ReviewForm.tsx` — Form component for star rating selection, content input, and submission.
- **New:** `storefront/modules/product/components/ReviewList.tsx` — Display list of reviews (rating, content, timestamp).
- **New:** `storefront/modules/product/services/ReviewService.ts` — API client wrapper for `POST /products/{id}/reviews` and `GET /products/{id}/reviews`.
- **New:** `storefront/modules/product/models/Review.ts` — TypeScript type for review data (`productId`, `customerId`, `rating`, `content`, `timestamp`).

**Validation:**
- Rating: must be an integer between 1 and 5 (dropdown or star selector).
- Content: must not be empty or whitespace-only.
- Both fields required before enabling submit button.

**Idempotency:**
- Disable submit button while request is in-flight to prevent double-clicks.

**Error Handling:**
- Display API validation errors (e.g. rating out of range, missing fields) returned from backend.
- Display generic error message on network or server failure.
- Do not update local state (rating, count) unless server confirms success.

---

### 2. Rating Backend API (Java + Spring Boot)

**Responsibility:**  
Expose `POST /products/{id}/reviews` to create reviews; validate request (rating 1–5, non-empty content); save review to database; recalculate product aggregate rating and count; return success or error response.

**Files:**
- **New:** `rating/src/main/java/com/yas/rating/controller/ReviewController.java` — REST controller for review endpoints.
- **New:** `rating/src/main/java/com/yas/rating/service/ReviewService.java` — Business logic: validation, persistence, aggregate updates.
- **New:** `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java` — JPA repository for `Review` entity.
- **New:** `rating/src/main/java/com/yas/rating/model/Review.java` — JPA entity mapping `review` table (columns: `id`, `product_id`, `customer_id`, `rating`, `content`, `created_at`, `updated_at`).
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java` — DTO for review creation request (`productId`, `rating`, `content`).
- **New:** `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java` — DTO for review response (`id`, `productId`, `customerId`, `rating`, `content`, `createdAt`).
- **New:** `rating/src/main/resources/db/changelog/ddl/changelog-review-table.sql` — Liquibase changeset to create `review` table.

**API Contract:**

**POST /api/rating/storefront/products/{productId}/reviews**

Request (JSON):
```json
{
  "rating": 5,
  "content": "Great product!"
}
```

Response (201 Created):
```json
{
  "id": 123,
  "productId": 1,
  "customerId": "uuid-of-customer",
  "rating": 5,
  "content": "Great product!",
  "createdAt": "2025-01-15T10:30:00Z"
}
```

Response (400 Bad Request — validation failure):
```json
{
  "status": 400,
  "title": "Validation Failed",
  "detail": "Rating must be between 1 and 5",
  "fieldErrors": [
    { "field": "rating", "message": "must be between 1 and 5" }
  ]
}
```

Response (500 Internal Server Error):
```json
{
  "status": 500,
  "title": "Internal Server Error",
  "detail": "An error occurred while saving the review"
}
```

**Validation:**
- `rating`: must be an integer between 1 and 5 (inclusive) — reject with 400 if not.
- `content`: must not be null, empty, or whitespace-only — reject with 400 if invalid.
- `productId`: must exist in the product service (verify via REST call) — reject with 404 if product not found.
- `customerId`: extracted from authenticated user token (Keycloak JWT) — reject with 401 if not authenticated.

**Idempotency:**
- Enforce a unique constraint on `(product_id, customer_id)` in the database to prevent duplicate reviews from the same customer.
- If a customer submits a second review for the same product, return 409 Conflict with message "You have already reviewed this product."
- **Alternative:** Accept multiple reviews but return the most recent one (requires removing uniqueness constraint and changing display logic).

---

### 3. Rating Database Schema (PostgreSQL)

**Responsibility:**  
Store review records; enforce uniqueness (one review per customer per product); store product aggregate rating and count.

**Schema:**

**Table: `review`**
```sql
CREATE TABLE review (
  id BIGSERIAL PRIMARY KEY,
  product_id BIGINT NOT NULL,
  customer_id VARCHAR(255) NOT NULL,
  rating INT NOT NULL CHECK (rating >= 1 AND rating <= 5),
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT unique_customer_product_review UNIQUE (product_id, customer_id)
);

CREATE INDEX idx_review_product_id ON review(product_id);
CREATE INDEX idx_review_customer_id ON review(customer_id);
```

**Table: `product_aggregate`**
```sql
CREATE TABLE product_aggregate (
  product_id BIGINT PRIMARY KEY,
  average_rating DECIMAL(3,2) NOT NULL,
  review_count INT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Notes:**
- `product_id` is a foreign reference to the `product` table in the `product` service (enforced via application logic, not database FK, as services are separate).
- `customer_id` is the user ID from Keycloak (UUID string).
- `product_aggregate` stores denormalized average rating and count for fast reads; updated after each review insert.

---

### 4. Aggregate Calculation Strategy

**Approach:**  
Maintain denormalized aggregate values (`average_rating`, `review_count`) in the `product_aggregate` table, updated transactionally after each review insert.

**Algorithm:**
1. **On review insert:**
   - Calculate new average rating:
     ```
     new_avg = ((old_avg * old_count) + new_rating) / (old_count + 1)
     ```
   - Increment review count by 1.
   - Upsert into `product_aggregate` table.
2. **On review delete (if supported):**
   - Recalculate average rating:
     ```
     new_avg = ((old_avg * old_count) - deleted_rating) / (old_count - 1)
     ```
   - Decrement review count by 1.
   - Update `product_aggregate` table.

**Consistency:**
- Wrap review insert and aggregate update in a database transaction to ensure atomicity.
- If aggregate update fails, roll back review insert.

**Alternative (not chosen):**
- Compute average rating and count on every read by querying the `review` table with `AVG(rating)` and `COUNT(*)`.
- Pros: No denormalization, simpler consistency model.
- Cons: Slower reads for high-traffic product pages.

---

### 5. Idempotency Mechanism

**Chosen Approach:**  
Database-level unique constraint on `(product_id, customer_id)` to prevent duplicate reviews.

**Behavior:**
- If a customer submits a second review for the same product, the database insert fails with a unique constraint violation.
- The backend catches this exception and returns `409 Conflict` with message "You have already reviewed this product."

**Alternative (not chosen):**
- Request token-based idempotency (client generates a unique token per submission, backend stores token in a separate table, rejects duplicate tokens).
- Pros: More flexible for multi-step workflows.
- Cons: Adds complexity; not needed for a simple "one review per customer per product" rule.

---

### 6. Display and Persistence

**Display:**
- After successful submission, the new review appears in the product's Reviews section (fetched via `GET /products/{id}/reviews` or added to the client-side list from the response).
- Product header displays updated average rating (e.g. "4.5 ★ (12 reviews)") and review count.

**Persistence:**
- Reviews are stored in the database and survive page refreshes / revisits.
- Frontend fetches reviews on page load via `GET /products/{id}/reviews`.

---

## Open Questions

1. **Authentication Requirement:**  
   Must customers be authenticated (logged in) to submit a review, or can anonymous users submit reviews?  
   **Decision Required:** Ticket implies authentication (customer identity required for idempotency and display). Recommend requiring authentication.

2. **Review Moderation:**  
   Are reviews immediately visible after submission, or do they go through moderation?  
   **Decision Required:** Ticket implies immediate visibility. If moderation is needed, add a `status` field (e.g. `PENDING`, `APPROVED`, `REJECTED`) to the `review` table and filter by status in queries.

3. **Review Content Length:**  
   What is the maximum length for review content?  
   **Decision Required:** Set a reasonable limit (e.g. 1000 characters) to prevent abuse. Enforce at API level and database schema (`TEXT` type supports up to 1 GB in PostgreSQL; add application validation).

4. **Review Editing:**  
   Can customers edit their reviews after submission?  
   **Decision Required:** Not in scope for this ticket. If required, add `PUT /reviews/{id}` endpoint and update `updated_at` timestamp on edit. Recalculate aggregates if rating changes.

5. **Review Deletion:**  
   Can customers delete their reviews? Can admins delete reviews?  
   **Decision Required:** Not in scope for this ticket. If required, add `DELETE /reviews/{id}` endpoint and recalculate aggregates on deletion.

6. **Pagination for Reviews:**  
   Should the reviews list be paginated?  
   **Decision Required:** Not specified in ticket. For initial implementation, return all reviews (up to a reasonable limit, e.g. 50). Add pagination later if needed.

---

## Architecture Decisions

### AD-1: Separate `rating` Service

**Binds:**  
Review data and aggregates are owned by a dedicated `rating` microservice, not embedded in the `product` service.

**Prevents:**  
Coupling product catalog logic to rating/review logic; allows independent scaling and development of review features.

**Rule:**  
The `product` service **does not** own review data. The `rating` service exposes review APIs. The `product` service **may** cache average rating and count for display (synced via event or periodic fetch), but the `rating` service is the source of truth.

**Rationale:**  
YAS follows a microservices architecture (observed in the `product`, `customer`, `order`, `inventory` services). Separating rating into its own service aligns with this pattern and isolates bounded contexts.

---

### AD-2: Denormalized Aggregates in `product_aggregate` Table

**Binds:**  
Average rating and review count are stored in a separate `product_aggregate` table, updated transactionally after each review insert.

**Prevents:**  
Slow reads caused by on-the-fly aggregate queries against the `review` table on high-traffic product pages.

**Rule:**  
The `rating` service MUST update `product_aggregate` in the same transaction as the review insert. Frontend reads aggregates from this table (or from the product service if cached).

**Rationale:**  
Pre-computed aggregates are a standard optimization for read-heavy scenarios. The trade-off (slight complexity in write logic) is justified by the expected high read-to-write ratio for product pages.

---

### AD-3: Database Unique Constraint for Idempotency

**Binds:**  
Idempotency is enforced by a unique constraint on `(product_id, customer_id)` in the `review` table.

**Prevents:**  
Duplicate reviews from the same customer for the same product, even if the submit button is clicked multiple times or multiple requests are sent concurrently.

**Rule:**  
The backend MUST NOT implement application-level duplicate detection (e.g. checking for existing reviews before insert). The database constraint is the single source of truth.

**Rationale:**  
Database constraints are atomic and race-condition-free. Application-level checks can fail under concurrent load. The database is the simplest and most reliable enforcement point.

---

### AD-4: REST API Over GraphQL or Event-Driven

**Binds:**  
Review submission and retrieval use synchronous REST APIs (`POST /products/{id}/reviews`, `GET /products/{id}/reviews`).

**Prevents:**  
Divergence from the existing YAS API style (all services expose REST endpoints, as observed in the codebase).

**Rule:**  
The `rating` service MUST expose REST endpoints following the YAS convention: `/api/rating/storefront/products/{productId}/reviews` for customer-facing operations.

**Rationale:**  
YAS uses REST for inter-service and client-server communication. Introducing GraphQL or event-driven patterns here would create inconsistency and require new infrastructure (GraphQL gateway, event bus) not present in the codebase.

---

### AD-5: Validation at Both UI and API Levels

**Binds:**  
Rating and content are validated client-side (disable submit button until valid) **and** server-side (reject invalid requests with 400 Bad Request).

**Prevents:**  
Bypassing client-side validation via direct API calls or browser manipulation.

**Rule:**  
The API MUST NOT trust client-side validation. All validation rules (rating range 1–5, non-empty content) MUST be enforced at the API layer with explicit error responses.

**Rationale:**  
Client-side validation improves UX (immediate feedback), but only server-side validation is secure and enforceable. Both layers are required for a production-grade feature.

---

## Risks

1. **Concurrency on Aggregate Updates:**  
   If two reviews for the same product are submitted simultaneously, aggregate updates could race.  
   **Mitigation:** Use database-level atomic operations (e.g. `UPDATE product_aggregate SET average_rating = ..., review_count = review_count + 1 WHERE product_id = ?`) within the transaction. Alternatively, use row-level locking (`SELECT FOR UPDATE`) on the `product_aggregate` row before updating.

2. **Customer Identity from JWT Token:**  
   Extracting `customer_id` from the JWT token assumes the token contains a user ID claim.  
   **Mitigation:** Verify the token structure in the existing `customer` or `order` services. If the token does not contain `customer_id`, call the `customer` service to fetch it via `GET /customer/storefront/customer/profile`.

3. **Product Existence Validation:**  
   The `rating` service must verify that `product_id` exists before saving a review. This requires calling the `product` service.  
   **Mitigation:** Use the existing `ProductService` pattern (observed in `cart`, `customer`, `order` services) to call `GET /api/product/storefront/products/{id}`. If the product is not found, return 404 to the client.

4. **Review Content Abuse:**  
   Customers could submit very long content or malicious HTML/JS.  
   **Mitigation:** Enforce a maximum content length (e.g. 1000 characters) at the API level. Sanitize content before storing (strip HTML tags, escape special characters) or store as plain text and escape on display.

5. **Average Rating Precision:**  
   Storing `average_rating` as `DECIMAL(3,2)` limits precision to two decimal places (e.g. 4.56).  
   **Mitigation:** Acceptable for star ratings displayed to customers. If higher precision is needed, use `DECIMAL(5,4)` or store the sum and count separately and calculate on read.

---

## Deferred

1. **Review Editing and Deletion:**  
   Not in scope for this ticket. Can be added later via `PUT /reviews/{id}` and `DELETE /reviews/{id}` endpoints.

2. **Review Pagination:**  
   Not specified in the ticket. Initial implementation returns all reviews (up to a reasonable limit). Pagination can be added later if needed.

3. **Review Moderation Workflow:**  
   Not specified in the ticket. If required, add a `status` field to the `review` table and implement an admin approval workflow.

4. **Review Photos/Media:**  
   Not specified in the ticket. If required, integrate with the existing `media` service to allow customers to upload images with reviews.

5. **Helpfulness Voting (Thumbs Up/Down):**  
   Not specified in the ticket. Can be added later via additional fields (`helpful_count`, `unhelpful_count`) and endpoints.

6. **Admin Review Management (Backoffice UI):**  
   Not specified in the ticket. Can be added later via a new backoffice page (`backoffice/pages/reviews/index.tsx`) and backend endpoints in the `rating` service (`GET /api/rating/backoffice/reviews`, `DELETE /api/rating/backoffice/reviews/{id}`).

---

## Notes

- The `rating` service does not currently exist in the YAS codebase. This ticket requires creating a new service following the existing structure (Spring Boot, JPA, Liquibase, REST API).
- The `product` service **may** cache average rating and count for display on product list pages (not detail pages). If caching is needed, the `rating` service can publish events (via Kafka) on review insert, and the `product` service can consume these events to update its cache. This is deferred until required.
- The frontend uses Next.js and TypeScript, following the existing pattern in `storefront/pages/products/[slug].tsx` and `storefront/modules/product/`.
- The API follows the YAS convention: `/api/<service>/storefront/<resource>` for customer-facing endpoints.
