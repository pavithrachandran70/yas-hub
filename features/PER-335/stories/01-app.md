# Story 01-app: Backend Review Submission API and Database Schema

**Repo:** `app`

---

## Context

Create the new `rating` microservice to store and serve product reviews. This story establishes the backend foundation: REST API endpoint for review creation, database schema (review records and denormalized aggregates), validation logic (rating range 1–5, non-empty content), product existence verification via the `product` service, aggregate calculation (average rating and review count), and idempotency enforcement (one review per customer per product). Without this backend, the frontend cannot submit or display reviews.

---

## Acceptance Criteria

1. Customers must be able to select a star rating between 1 and 5. (Backend validates rating range 1–5)
2. Review content must be mandatory and must not contain only whitespace.
3. The system must prevent submission if the rating or review content is missing.
4. The system must reject invalid ratings, including values below 1 or above 5, through API validation.
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (Backend saves review and returns success response)
6. The product's average rating must be recalculated based on all valid reviews.
7. The total review count must increase by one after successful submission.
8. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (Backend returns 400/404/409/500 responses)
9. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (Backend enforces unique constraint on `(product_id, customer_id)`)
10. Submitted reviews must remain available after refreshing or revisiting the product page. (Backend persists reviews in database)

---

## File Plan

### New Files

**Service Module (new `rating` service):**
- `rating/pom.xml` — Maven POM inheriting from parent POM, declares dependencies on `common-library`, Spring Boot, JPA, Liquibase, Keycloak adapter, Swagger
- `rating/Dockerfile` — Docker image definition (follow pattern from `product/Dockerfile`)
- `rating/src/main/java/com/yas/rating/RatingApplication.java` — Spring Boot main class with `@SpringBootApplication`
- `rating/src/main/java/com/yas/rating/config/DatabaseAutoConfig.java` — JPA config (follow `product/src/main/java/com/yas/product/config/DatabaseAutoConfig.java`)
- `rating/src/main/java/com/yas/rating/config/RestClientConfig.java` — REST client config for inter-service calls to `product` service
- `rating/src/main/java/com/yas/rating/config/SecurityConfig.java` — Spring Security config (Keycloak JWT authentication, follow `product/src/main/java/com/yas/product/config/SecurityConfig.java`)
- `rating/src/main/java/com/yas/rating/config/SwaggerConfig.java` — Swagger API docs config
- `rating/src/main/java/com/yas/rating/config/ServiceUrlConfig.java` — Service URL config bean for product service base URL (follow `customer/src/main/java/com/yas/customer/config/ServiceUrlConfig.java`)
- `rating/src/main/java/com/yas/rating/controller/ReviewController.java` — REST controller for review endpoints
- `rating/src/main/java/com/yas/rating/service/ReviewService.java` — Business logic: validation, persistence, aggregate updates
- `rating/src/main/java/com/yas/rating/service/ProductService.java` — Circuit-breaker-wrapped client for product service (verify product exists, follow `cart/src/main/java/com/yas/cart/service/ProductService.java`)
- `rating/src/main/java/com/yas/rating/service/AggregateService.java` — Aggregate calculation and persistence
- `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java` — JPA repository for `Review` entity
- `rating/src/main/java/com/yas/rating/repository/ProductAggregateRepository.java` — JPA repository for `ProductAggregate` entity
- `rating/src/main/java/com/yas/rating/model/Review.java` — JPA entity mapping `review` table
- `rating/src/main/java/com/yas/rating/model/ProductAggregate.java` — JPA entity mapping `product_aggregate` table
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java` — DTO for review creation request (`rating`, `content`)
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java` — DTO for review response (`id`, `productId`, `customerId`, `rating`, `content`, `createdAt`)
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewListVm.java` — DTO for review list response (wraps list of `ReviewVm` + `averageRating`, `totalReviews`)
- `rating/src/main/resources/application.properties` — Spring Boot config (database URL, Keycloak config, service port 8095)
- `rating/src/main/resources/logback-spring.xml` — Logging config (follow pattern from `product/src/main/resources/logback-spring.xml`)
- `rating/src/main/resources/db/changelog/db.changelog-master.yaml` — Liquibase master changelog
- `rating/src/main/resources/db/changelog/ddl/changelog-review-table.sql` — Liquibase changeset to create `review` table
- `rating/src/main/resources/db/changelog/ddl/changelog-product-aggregate-table.sql` — Liquibase changeset to create `product_aggregate` table
- `rating/src/main/resources/messages/messages.properties` — Internationalization messages (error messages)
- `rating/src/test/java/com/yas/rating/controller/ReviewControllerTest.java` — Unit tests for `ReviewController`
- `rating/src/test/java/com/yas/rating/service/ReviewServiceTest.java` — Unit tests for `ReviewService`
- `rating/src/test/resources/application.properties` — Test-specific Spring Boot config
- `rating/src/it/java/com/yas/rating/controller/ReviewControllerIT.java` — Integration test for full flow (review persistence)

### Modified Files

- `pom.xml` — Add `<module>rating</module>` to `<modules>` section
- `docker-compose.yml` — Add `rating` service container (follow pattern of other services)
- `postgres_init.sql` — Add `CREATE DATABASE rating;`

---

## Interfaces

### REST API Endpoints

**POST /api/rating/storefront/products/{productId}/reviews**

- **Purpose:** Create a new review for a product
- **Authentication:** Required (Keycloak JWT token)
- **Request:**
  - Path parameter: `productId` (Long)
  - Body: `ReviewPostVm` (JSON)
    ```json
    {
      "rating": 5,
      "content": "Great product!"
    }
    ```
- **Response:**
  - **201 Created:** `ReviewVm` (JSON)
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
  - **400 Bad Request:** Validation failure (rating out of range, empty content, missing fields)
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
  - **401 Unauthorized:** Not authenticated
  - **404 Not Found:** Product does not exist
  - **409 Conflict:** Duplicate review (customer already reviewed this product)
    ```json
    {
      "status": 409,
      "title": "Conflict",
      "detail": "You have already reviewed this product"
    }
    ```
  - **500 Internal Server Error:** Server error

**GET /api/rating/storefront/products/{productId}/reviews**

- **Purpose:** Retrieve all reviews for a product with aggregates
- **Authentication:** Optional (public read)
- **Request:**
  - Path parameter: `productId` (Long)
- **Response:**
  - **200 OK:** `ReviewListVm` (JSON)
    ```json
    {
      "reviews": [
        {
          "id": 123,
          "customerId": "uuid-of-customer",
          "rating": 5,
          "content": "Great product!",
          "createdAt": "2025-01-15T10:30:00Z"
        }
      ],
      "averageRating": 4.56,
      "totalReviews": 12
    }
    ```
  - **404 Not Found:** Product does not exist (optional; can return empty list instead)

---

### Database Schema (PostgreSQL)

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

**Fields:**
- `id`: auto-generated primary key
- `product_id`: foreign reference to product (enforced at application level, as `product` service owns the product table)
- `customer_id`: user ID from Keycloak (UUID string), extracted from JWT token via `AuthenticationUtils.getCurrentUserId()` (verify in `common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java`)
- `rating`: integer 1–5 (database constraint)
- `content`: review text (non-empty, enforced at application level)
- `created_at`, `updated_at`: audit timestamps (inherited from `AbstractAuditEntity`)
- **Unique constraint on `(product_id, customer_id)`** prevents duplicate reviews (idempotency enforcement)

**Table: `product_aggregate`**
```sql
CREATE TABLE product_aggregate (
  product_id BIGINT PRIMARY KEY,
  average_rating DECIMAL(3,2) NOT NULL,
  review_count INT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Fields:**
- `product_id`: foreign reference to product (application-level, no FK constraint)
- `average_rating`: pre-computed average rating (e.g. 4.56)
- `review_count`: total number of reviews for the product
- `updated_at`: last aggregate update timestamp

---

### Service Layer Interfaces

**ReviewService**
- `createReview(Long productId, ReviewPostVm reviewPostVm) -> ReviewVm`
  - Extracts `customerId` from JWT token using `AuthenticationUtils.getCurrentUserId()`
  - Validates `rating` (1–5) and `content` (non-empty, non-whitespace)
  - Calls `ProductService.verifyProductExists(productId)` — throws `NotFoundException` if product not found
  - Creates `Review` entity and saves via `ReviewRepository.save(review)`
  - Catches `DataIntegrityViolationException` for duplicate review (unique constraint) and throws `DuplicatedException` (HTTP 409)
  - Calls `AggregateService.updateAggregatesAfterReviewCreated(productId, rating)`
  - Returns `ReviewVm` DTO

- `getReviewsByProductId(Long productId) -> ReviewListVm`
  - Queries `ReviewRepository.findByProductIdOrderByCreatedAtDesc(productId)`
  - Fetches aggregates from `ProductAggregateRepository.findById(productId)` — if not found, returns `averageRating = 0.0, totalReviews = 0`
  - Returns `ReviewListVm` (list of `ReviewVm` + aggregates)

**AggregateService**
- `updateAggregatesAfterReviewCreated(Long productId, Integer newRating) -> void`
  - Fetches `ProductAggregate` row via `ProductAggregateRepository.findById(productId)`
  - If not found, creates new row with `averageRating = newRating, reviewCount = 1`
  - If found, recalculates:
    ```
    new_avg = ((old_avg * old_count) + new_rating) / (old_count + 1)
    new_count = old_count + 1
    ```
  - Saves via `ProductAggregateRepository.save()` (upsert)
  - **Transaction:** entire method runs in a database transaction to ensure atomicity

**ProductService**
- `verifyProductExists(Long productId) -> void`
  - Calls `GET /api/product/storefront/products/{productId}` via `RestClient`
  - Throws `NotFoundException` (HTTP 404) if product not found
  - Wraps call with circuit breaker (Resilience4j, follow `cart/src/main/java/com/yas/cart/service/ProductService.java`)
  - Constructor injects `RestClient` and `ServiceUrlConfig` to get product service base URL: `public ProductService(RestClient restClient, ServiceUrlConfig serviceUrlConfig)` and uses `serviceUrlConfig.getProduct()` to build the URL

---

### Repository Interfaces

**ReviewRepository** (extends `JpaRepository<Review, Long>`)
- `List<Review> findByProductIdOrderByCreatedAtDesc(Long productId)`

**ProductAggregateRepository** (extends `JpaRepository<ProductAggregate, Long>`)
- Standard CRUD methods from `JpaRepository`

---

### DTOs (ViewModels)

**ReviewPostVm** (request DTO)
```java
package com.yas.rating.viewmodel;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

public record ReviewPostVm(
    @NotNull(message = "Rating is required")
    @Min(value = 1, message = "Rating must be at least 1")
    @Max(value = 5, message = "Rating must be at most 5")
    Integer rating,

    @NotBlank(message = "Review content is required")
    @Size(max = 1000, message = "Review content must not exceed 1000 characters")
    String content
) {}
```

**ReviewVm** (response DTO)
```java
package com.yas.rating.viewmodel;

import java.time.ZonedDateTime;

public record ReviewVm(
    Long id,
    Long productId,
    String customerId,
    Integer rating,
    String content,
    ZonedDateTime createdAt
) {}
```

**ReviewListVm** (response DTO for list endpoint)
```java
package com.yas.rating.viewmodel;

import java.util.List;

public record ReviewListVm(
    List<ReviewVm> reviews,
    Double averageRating,
    Integer totalReviews
) {}
```

---

### JPA Entities

**Review**
```java
package com.yas.rating.model;

import com.yas.commonlibrary.model.AbstractAuditEntity;
import jakarta.persistence.*;

@Entity
@Table(name = "review", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"product_id", "customer_id"}, name = "unique_customer_product_review")
})
public class Review extends AbstractAuditEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_id", nullable = false)
    private Long productId;

    @Column(name = "customer_id", nullable = false, length = 255)
    private String customerId;

    @Column(name = "rating", nullable = false)
    private Integer rating;

    @Column(name = "content", nullable = false, columnDefinition = "TEXT")
    private String content;

    // Getters and setters omitted for brevity
}
```

**ProductAggregate**
```java
package com.yas.rating.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "product_aggregate")
public class ProductAggregate {

    @Id
    @Column(name = "product_id")
    private Long productId;

    @Column(name = "average_rating", nullable = false, precision = 3, scale = 2)
    private BigDecimal averageRating;

    @Column(name = "review_count", nullable = false)
    private Integer reviewCount;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @PreUpdate
    protected void onUpdate() {
        updatedAt = Instant.now();
    }

    // Constructors, getters, and setters omitted for brevity
}
```

---

## Algorithm Notes

### Review Creation Flow

1. **Extract Customer ID:**
   - Use `AuthenticationUtils.getCurrentUserId()` from `common-library` to extract `customer_id` from JWT token
   - If token does not contain `customer_id`, fall back to `sub` claim (standard JWT subject claim)

2. **Validate Input:**
   - `rating`: must be non-null, integer, 1–5 (validation annotations: `@NotNull`, `@Min(1)`, `@Max(5)`)
   - `content`: must be non-blank (validation annotation: `@NotBlank`) — rejects empty string and whitespace-only
   - Additional validation: trim content and check if empty after trimming:
     ```java
     if (reviewPostVm.content() == null || reviewPostVm.content().trim().isEmpty()) {
         throw new BadRequestException("Review content is required");
     }
     ```

3. **Verify Product Exists:**
   - Call `ProductService.verifyProductExists(productId)`
   - If product not found, throw `NotFoundException` (HTTP 404)

4. **Create Review Entity:**
   ```java
   Review review = new Review();
   review.setProductId(productId);
   review.setCustomerId(customerId);
   review.setRating(reviewPostVm.rating());
   review.setContent(reviewPostVm.content().trim());
   ```

5. **Save Review (with Duplicate Handling):**
   ```java
   try {
       review = reviewRepository.save(review);
   } catch (DataIntegrityViolationException e) {
       // Unique constraint violation on (product_id, customer_id)
       throw new DuplicatedException("You have already reviewed this product");
   }
   ```

6. **Update Aggregates:**
   - Call `AggregateService.updateAggregatesAfterReviewCreated(productId, rating)`
   - Wrapped in same transaction as review save to ensure atomicity

7. **Return Response:**
   - Convert `Review` entity to `ReviewVm` DTO and return with HTTP 201 status

### Aggregate Update Algorithm

**Scenario 1: First Review for Product**
```java
// No existing aggregate row
ProductAggregate aggregate = new ProductAggregate();
aggregate.setProductId(productId);
aggregate.setAverageRating(new BigDecimal(newRating));
aggregate.setReviewCount(1);
aggregate.setUpdatedAt(Instant.now());
productAggregateRepository.save(aggregate);
```

**Scenario 2: Subsequent Reviews**
```java
ProductAggregate aggregate = productAggregateRepository.findById(productId)
    .orElseThrow(() -> new NotFoundException("Aggregate not found")); // Should not happen

BigDecimal oldAvg = aggregate.getAverageRating();
int oldCount = aggregate.getReviewCount();

// Calculate new average: ((old_avg * old_count) + new_rating) / (old_count + 1)
BigDecimal newAvg = oldAvg.multiply(new BigDecimal(oldCount))
    .add(new BigDecimal(newRating))
    .divide(new BigDecimal(oldCount + 1), 2, RoundingMode.HALF_UP); // Round to 2 decimal places

aggregate.setAverageRating(newAvg);
aggregate.setReviewCount(oldCount + 1);
aggregate.setUpdatedAt(Instant.now());

productAggregateRepository.save(aggregate);
```

**Race Condition Handling:**
- Entire method runs in a database transaction (`@Transactional`)
- Use row-level locking: add `@Lock(LockModeType.PESSIMISTIC_WRITE)` to `ProductAggregateRepository.findById()` to prevent concurrent updates
- Alternative: use optimistic locking with `@Version` column in `ProductAggregate` entity and retry on conflict

### Review Retrieval Flow

1. **Fetch Reviews:**
   - Query `ReviewRepository.findByProductIdOrderByCreatedAtDesc(productId)`
   - Returns list of `Review` entities ordered by `created_at` descending (most recent first)

2. **Fetch Aggregates:**
   - Query `ProductAggregateRepository.findById(productId)`
   - If not found (no reviews yet), return `averageRating = 0.0, totalReviews = 0`

3. **Convert to DTOs:**
   - Map each `Review` entity to `ReviewVm`:
     ```java
     new ReviewVm(
         review.getId(),
         review.getProductId(),
         review.getCustomerId(),
         review.getRating(),
         review.getContent(),
         review.getCreatedAt()
     )
     ```

4. **Return Response:**
   - Wrap list of `ReviewVm` + aggregates in `ReviewListVm` and return with HTTP 200 status

---

## Tests

### Unit Tests (ReviewControllerTest)

**Test File:** `rating/src/test/java/com/yas/rating/controller/ReviewControllerTest.java`

**Coverage:** REST controller behavior, request validation, response status codes

1. **testCreateReview_Success** — POST with valid rating (5) and content returns 201 and `ReviewVm` with correct fields
2. **testCreateReview_MissingRating** — POST with no rating returns 400 with field error "Rating is required"
3. **testCreateReview_InvalidRating_Zero** — POST with rating 0 returns 400 with field error "Rating must be at least 1"
4. **testCreateReview_InvalidRating_Six** — POST with rating 6 returns 400 with field error "Rating must be at most 5"
5. **testCreateReview_MissingContent** — POST with empty content returns 400 with field error "Review content is required"
6. **testCreateReview_ProductNotFound** — POST for non-existent product (id 999) returns 404 with detail "Product not found"

```java
package com.yas.rating.controller;

import com.yas.rating.service.ReviewService;
import com.yas.rating.viewmodel.ReviewPostVm;
import com.yas.rating.viewmodel.ReviewVm;
import com.yas.commonlibrary.exception.NotFoundException;
import com.yas.commonlibrary.exception.DuplicatedException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.time.ZonedDateTime;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ReviewController.class)
class ReviewControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ReviewService reviewService;

    @Test
    void testCreateReview_Success() throws Exception {
        // Given
        Long productId = 1L;
        ReviewPostVm request = new ReviewPostVm(5, "Great product!");
        ReviewVm response = new ReviewVm(123L, productId, "customer-uuid", 5, "Great product!", ZonedDateTime.now());

        when(reviewService.createReview(eq(productId), any(ReviewPostVm.class))).thenReturn(response);

        // When & Then
        mockMvc.perform(post("/api/rating/storefront/products/{productId}/reviews", productId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"rating\": 5, \"content\": \"Great product!\"}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(123))
            .andExpect(jsonPath("$.rating").value(5))
            .andExpect(jsonPath("$.content").value("Great product!"));

        verify(reviewService, times(1)).createReview(eq(productId), any(ReviewPostVm.class));
    }

    @Test
    void testCreateReview_MissingRating() throws Exception {
        // Given
        Long productId = 1L;

        // When & Then
        mockMvc.perform(post("/api/rating/storefront/products/{productId}/reviews", productId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"content\": \"Great product!\"}")) // Missing rating
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors[0].field").value("rating"))
            .andExpect(jsonPath("$.fieldErrors[0].message").value("Rating is required"));

        verify(reviewService, never()).createReview(anyLong(), any(ReviewPostVm.class));
    }

    @Test
    void testCreateReview_InvalidRating_Zero() throws Exception {
        // Given
        Long productId = 1L;

        // When & Then
        mockMvc.perform(post("/api/rating/storefront/products/{productId}/reviews", productId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"rating\": 0, \"content\": \"Great product!\"}")) // Invalid rating
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors[0].field").value("rating"))
            .andExpect(jsonPath("$.fieldErrors[0].message").value("Rating must be at least 1"));

        verify(reviewService, never()).createReview(anyLong(), any(ReviewPostVm.class));
    }

    @Test
    void testCreateReview_InvalidRating_Six() throws Exception {
        // Given
        Long productId = 1L;

        // When & Then
        mockMvc.perform(post("/api/rating/storefront/products/{productId}/reviews", productId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"rating\": 6, \"content\": \"Great product!\"}")) // Invalid rating
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors[0].field").value("rating"))
            .andExpect(jsonPath("$.fieldErrors[0].message").value("Rating must be at most 5"));

        verify(reviewService, never()).createReview(anyLong(), any(ReviewPostVm.class));
    }

    @Test
    void testCreateReview_MissingContent() throws Exception {
        // Given
        Long productId = 1L;

        // When & Then
        mockMvc.perform(post("/api/rating/storefront/products/{productId}/reviews", productId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"rating\": 5}")) // Missing content
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors[0].field").value("content"))
            .andExpect(jsonPath("$.fieldErrors[0].message").value("Review content is required"));

        verify(reviewService, never()).createReview(anyLong(), any(ReviewPostVm.class));
    }

    @Test
    void testCreateReview_ProductNotFound() throws Exception {
        // Given
        Long productId = 999L;
        when(reviewService.createReview(eq(productId), any(ReviewPostVm.class)))
            .thenThrow(new NotFoundException("Product not found"));

        // When & Then
        mockMvc.perform(post("/api/rating/storefront/products/{productId}/reviews", productId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"rating\": 5, \"content\": \"Great product!\"}"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.detail").value("Product not found"));

        verify(reviewService, times(1)).createReview(eq(productId), any(ReviewPostVm.class));
    }
}
```

### Unit Tests (ReviewServiceTest)

**Test File:** `rating/src/test/java/com/yas/rating/service/ReviewServiceTest.java`

**Coverage:** Business logic, aggregate calculations, duplicate handling

7. **testCreateReview_DuplicateReview** — Second review from same customer for same product throws `DuplicatedException`, caught and returns 409
8. **testUpdateAggregates_FirstReview** — Creating first review sets `average_rating` = rating and `review_count` = 1
9. **testUpdateAggregates_SubsequentReview** — Creating second review recalculates `average_rating` correctly (from 5.00 to 4.50 when adding rating 4) and increments `review_count` to 2

```java
package com.yas.rating.service;

import com.yas.rating.model.Review;
import com.yas.rating.model.ProductAggregate;
import com.yas.rating.repository.ReviewRepository;
import com.yas.rating.repository.ProductAggregateRepository;
import com.yas.rating.viewmodel.ReviewPostVm;
import com.yas.rating.viewmodel.ReviewVm;
import com.yas.commonlibrary.exception.DuplicatedException;
import com.yas.commonlibrary.exception.NotFoundException;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.dao.DataIntegrityViolationException;

import java.math.BigDecimal;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ReviewServiceTest {

    @Mock
    private ReviewRepository reviewRepository;

    @Mock
    private ProductAggregateRepository productAggregateRepository;

    @Mock
    private ProductService productService;

    @InjectMocks
    private ReviewService reviewService;

    @Test
    void testCreateReview_DuplicateReview() {
        // Given
        Long productId = 1L;
        ReviewPostVm reviewPostVm = new ReviewPostVm(5, "Great product!");

        doNothing().when(productService).verifyProductExists(productId);
        when(reviewRepository.save(any(Review.class))).thenThrow(new DataIntegrityViolationException("Unique constraint violation"));

        // When & Then
        assertThrows(DuplicatedException.class, () -> reviewService.createReview(productId, reviewPostVm));

        verify(productService, times(1)).verifyProductExists(productId);
        verify(reviewRepository, times(1)).save(any(Review.class));
        verify(productAggregateRepository, never()).save(any(ProductAggregate.class));
    }

    @Test
    void testUpdateAggregates_FirstReview() {
        // Given
        Long productId = 1L;
        Integer newRating = 5;

        when(productAggregateRepository.findById(productId)).thenReturn(Optional.empty());
        when(productAggregateRepository.save(any(ProductAggregate.class))).thenAnswer(invocation -> invocation.getArgument(0));

        // When
        reviewService.updateAggregatesAfterReviewCreated(productId, newRating);

        // Then
        verify(productAggregateRepository, times(1)).findById(productId);
        verify(productAggregateRepository, times(1)).save(argThat(aggregate ->
            aggregate.getProductId().equals(productId) &&
            aggregate.getAverageRating().equals(new BigDecimal(newRating)) &&
            aggregate.getReviewCount() == 1
        ));
    }

    @Test
    void testUpdateAggregates_SubsequentReview() {
        // Given
        Long productId = 1L;
        Integer newRating = 4;

        ProductAggregate existingAggregate = new ProductAggregate();
        existingAggregate.setProductId(productId);
        existingAggregate.setAverageRating(new BigDecimal("5.00"));
        existingAggregate.setReviewCount(1);

        when(productAggregateRepository.findById(productId)).thenReturn(Optional.of(existingAggregate));
        when(productAggregateRepository.save(any(ProductAggregate.class))).thenAnswer(invocation -> invocation.getArgument(0));

        // When
        reviewService.updateAggregatesAfterReviewCreated(productId, newRating);

        // Then
        // Expected new average: ((5.00 * 1) + 4) / 2 = 4.50
        verify(productAggregateRepository, times(1)).findById(productId);
        verify(productAggregateRepository, times(1)).save(argThat(aggregate ->
            aggregate.getProductId().equals(productId) &&
            aggregate.getAverageRating().compareTo(new BigDecimal("4.50")) == 0 &&
            aggregate.getReviewCount() == 2
        ));
    }
}
```

### Integration Test (ReviewControllerIT)

**Test File:** `rating/src/it/java/com/yas/rating/controller/ReviewControllerIT.java`

**Coverage:** Full flow with database, persistence across service restarts

10. **testCreateReviewAndFetch_Persistence** — POST review (rating 5, content "Excellent product!"), then GET reviews — review is still present with correct data, `averageRating = 5.0`, `totalReviews = 1`

```java
package com.yas.rating.controller;

import com.yas.commonlibrary.IntegrationTestConfiguration;
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.TestPropertySource;
import org.testcontainers.junit.jupiter.Testcontainers;

import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ContextConfiguration(classes = IntegrationTestConfiguration.class)
@TestPropertySource("classpath:application.properties")
class ReviewControllerIT {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
    }

    @Test
    void testCreateReviewAndFetch_Persistence() {
        // Given
        Long productId = 1L;
        String requestBody = "{\"rating\": 5, \"content\": \"Excellent product!\"}";

        // When: Create review
        given()
            .contentType(ContentType.JSON)
            .body(requestBody)
        .when()
            .post("/api/rating/storefront/products/{productId}/reviews", productId)
        .then()
            .statusCode(201)
            .body("id", notNullValue())
            .body("productId", equalTo(productId.intValue()))
            .body("rating", equalTo(5))
            .body("content", equalTo("Excellent product!"));

        // Then: Fetch reviews (simulate service restart by using GET endpoint)
        given()
        .when()
            .get("/api/rating/storefront/products/{productId}/reviews", productId)
        .then()
            .statusCode(200)
            .body("reviews", hasSize(1))
            .body("reviews[0].rating", equalTo(5))
            .body("reviews[0].content", equalTo("Excellent product!"))
            .body("averageRating", equalTo(5.0f))
            .body("totalReviews", equalTo(1));
    }
}
```

---

## Dependencies

None (foundational backend story).

---

## Out of Scope

- Frontend UI for review submission (covered in Story 2)
- Frontend display of reviews and aggregates (covered in Story 2)
- Review editing or deletion by customers (not in ticket scope)
- Review moderation workflow (not in ticket scope; reviews are immediately visible)
- Backoffice endpoints for admin review management (covered in Story 3)

---

## Notes

- The `rating` service does not currently exist in the YAS codebase. This story creates a new Spring Boot service following the existing structure (`product`, `customer`, `order`, `inventory`).
- The backoffice UI (`backoffice/pages/reviews/index.tsx`) already exists and expects a reviews API. This story must implement the backend endpoints to support it (covered in Story 3 for backoffice-specific endpoints).
- The API follows the YAS convention: `/api/<service>/storefront/<resource>` for customer-facing endpoints.
- Aggregate calculation uses denormalized `product_aggregate` table for fast reads (pre-computed average rating and count), updated transactionally after each review insert.
- Idempotency is enforced by database unique constraint on `(product_id, customer_id)` — the simplest and most reliable approach.
- The `AuthenticationUtils.getCurrentUserId()` method (in `common-library`) assumes the JWT token contains a user ID claim. Verify this in the Keycloak token structure before implementation.
