# Merged plan — PER-335

Execution order (2 ticket(s)):

1. `1`
2. `2`

Specs (2): `specs/1.md`, `specs/2.md`

---

## Spec `specs/1.md`

# specs/1.md

**Ticket:** Customer Can write and Submit Product Reviews  
**Epic:** _(none)_  
**Initiative:** _(none)_

---

## Stack

**Backend (Java + Spring Boot):**
- Framework: Spring Boot 3.x (observed in `cart`, `customer`, `order`, `inventory` services)
- Persistence: JPA + Hibernate, PostgreSQL
- Schema migration: Liquibase (observed in `cart/src/main/resources/db/changelog/`, `customer/src/main/resources/db/changelog/`, `inventory/src/main/resources/db/changelog/`, `location/src/main/resources/db/changelog/`, `media/src/main/resources/db/changelog/`)
- REST API: Spring Web, `@RestController`, `@RequestMapping`, `@PostMapping`, `@GetMapping`
- Validation: `@Valid`, `@NotNull`, `@Min`, `@Max`, Jakarta Validation API
- Security: Spring Security + Keycloak (JWT authentication, observed in `cart/src/main/java/com/yas/cart/config/SecurityConfig.java`, `customer/src/main/java/com/yas/customer/config/SecurityConfig.java`, `order/src/main/java/com/yas/order/config/SecurityConfig.java`)
- Error handling: `@ControllerAdvice`, `ApiExceptionHandler` (observed in `common-library/src/main/java/com/yas/commonlibrary/exception/ApiExceptionHandler.java`)
- Inter-service calls: `RestClient` (observed in `cart/src/main/java/com/yas/cart/config/RestClientConfig.java`, `customer/src/main/java/com/yas/customer/config/RestClientConfig.java`, `order/src/main/java/com/yas/order/config/RestClientConfig.java`)
- Maven multi-module project (observed in root `pom.xml`, `cart/pom.xml`, `customer/pom.xml`, etc.)

**Frontend (Next.js + TypeScript):**
- Framework: Next.js 12+ (observed in `backoffice/next.config.js`, `backoffice/package.json`)
- Language: TypeScript (observed in `backoffice/tsconfig.json`, `.tsx` files)
- Forms: `react-hook-form` (observed in `backoffice/common/items/Input.tsx`, `backoffice/modules/catalog/components/ProductGeneralInformation.tsx`)
- HTTP client: `fetch` via `ApiClientService` (observed in `backoffice/common/services/ApiClientService.ts`)
- Toast notifications: `react-toastify` (observed in `backoffice/common/services/ToastService.ts`, `backoffice/pages/reviews/index.tsx`)
- Routing: Next.js file-based routing (`pages/` directory, observed in `backoffice/pages/`)

**Database:**
- PostgreSQL (observed in `docker-compose.yml`, `postgres_init.sql`, service `application.properties` files)
- Tables owned by their respective services (no cross-service foreign keys)

**Service Architecture:**
- Microservices: `cart`, `customer`, `order`, `inventory`, `location`, `media`, `product`, `rating` (observed in repo tree)
- Each service: standalone Spring Boot app, own database, REST APIs under `/api/<service>/storefront/` (customer-facing) or `/api/<service>/backoffice/` (admin-facing)
- Security: JWT token from Keycloak, extracted via `AuthenticationUtils.getCurrentUserId()` (observed in `common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java`, `inventory/src/main/java/com/yas/inventory/utils/AuthenticationUtils.java`)

---

## Schema/API Contracts

### New `rating` Service

**Service does not currently exist.** This ticket requires creating a new microservice following the existing YAS structure:
- Maven module `rating/` under root `pom.xml`
- Structure mirrors `cart/`, `customer/`, `order/` services:
  - `rating/src/main/java/com/yas/rating/RatingApplication.java` (Spring Boot main class)
  - `rating/src/main/java/com/yas/rating/config/` (SecurityConfig, DatabaseAutoConfig, SwaggerConfig, RestClientConfig, ServiceUrlConfig)
  - `rating/src/main/java/com/yas/rating/controller/ReviewController.java`
  - `rating/src/main/java/com/yas/rating/service/ReviewService.java`
  - `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java`
  - `rating/src/main/java/com/yas/rating/model/Review.java` (JPA entity)
  - `rating/src/main/java/com/yas/rating/viewmodel/` (DTOs: `ReviewPostVm`, `ReviewVm`)
  - `rating/src/main/resources/application.properties` (database config, service port)
  - `rating/src/main/resources/db/changelog/db.changelog-master.yaml`
  - `rating/src/main/resources/db/changelog/ddl/changelog-0001.sql` (create `review` and `product_aggregate` tables)
  - `rating/src/main/resources/messages/messages.properties`
  - `rating/src/test/java/com/yas/rating/controller/ReviewControllerTest.java`
  - `rating/Dockerfile`

### Database Schema (PostgreSQL)

**New table: `review`**
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
- `rating`: integer 1–5 (database constraint: `CHECK (rating >= 1 AND rating <= 5)`)
- `content`: review text (non-empty, enforced at application level)
- `created_at`, `updated_at`: audit timestamps
- **Unique constraint on `(product_id, customer_id)`** prevents duplicate reviews (idempotency enforcement)

**New table: `product_aggregate`**
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

**Migration:**
- Liquibase changeset `rating/src/main/resources/db/changelog/ddl/changelog-0001.sql` creates both tables
- `rating/src/main/resources/db/changelog/db.changelog-master.yaml` references the changeset

### API Contract: POST /api/rating/storefront/products/{productId}/reviews

**Endpoint:** `POST /api/rating/storefront/products/{productId}/reviews`

**Request Headers:**
- `Authorization: Bearer <JWT_TOKEN>` (required, Keycloak JWT)

**Request Body (JSON):**
```json
{
  "rating": 5,
  "content": "Great product!"
}
```

**Request DTO:** `ReviewPostVm.java`
```java
package com.yas.rating.viewmodel;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public record ReviewPostVm(
    @NotNull(message = "Rating is required")
    @Min(value = 1, message = "Rating must be at least 1")
    @Max(value = 5, message = "Rating must be at most 5")
    Integer rating,

    @NotBlank(message = "Review content is required")
    String content
) {}
```

**Response (201 Created):**
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

**Response DTO:** `ReviewVm.java`
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

**Response (400 Bad Request — validation failure):**
```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "Validation failed",
  "fieldErrors": [
    { "field": "rating", "message": "Rating must be between 1 and 5" },
    { "field": "content", "message": "Review content is required" }
  ]
}
```

**Response (409 Conflict — duplicate review):**
```json
{
  "status": 409,
  "title": "Conflict",
  "detail": "You have already reviewed this product"
}
```

**Response (404 Not Found — product does not exist):**
```json
{
  "status": 404,
  "title": "Not Found",
  "detail": "Product not found"
}
```

**Response (401 Unauthorized — not authenticated):**
```json
{
  "status": 401,
  "title": "Unauthorized",
  "detail": "Authentication required"
}
```

**Controller:** `ReviewController.java`
```java
package com.yas.rating.controller;

import com.yas.rating.service.ReviewService;
import com.yas.rating.viewmodel.ReviewPostVm;
import com.yas.rating.viewmodel.ReviewVm;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/rating/storefront/products")
public class ReviewController {

    private final ReviewService reviewService;

    public ReviewController(ReviewService reviewService) {
        this.reviewService = reviewService;
    }

    @PostMapping("/{productId}/reviews")
    public ResponseEntity<ReviewVm> createReview(
            @PathVariable Long productId,
            @Valid @RequestBody ReviewPostVm reviewPostVm
    ) {
        ReviewVm reviewVm = reviewService.createReview(productId, reviewPostVm);
        return ResponseEntity.status(HttpStatus.CREATED).body(reviewVm);
    }
}
```

**Service:** `ReviewService.java`
```java
package com.yas.rating.service;

import com.yas.commonlibrary.exception.DuplicatedException;
import com.yas.commonlibrary.exception.NotFoundException;
import com.yas.commonlibrary.utils.AuthenticationUtils;
import com.yas.rating.model.Review;
import com.yas.rating.repository.ReviewRepository;
import com.yas.rating.viewmodel.ReviewPostVm;
import com.yas.rating.viewmodel.ReviewVm;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ReviewService {

    private final ReviewRepository reviewRepository;
    private final ProductService productService;
    private final AggregateService aggregateService;

    public ReviewService(ReviewRepository reviewRepository, ProductService productService, AggregateService aggregateService) {
        this.reviewRepository = reviewRepository;
        this.productService = productService;
        this.aggregateService = aggregateService;
    }

    @Transactional
    public ReviewVm createReview(Long productId, ReviewPostVm reviewPostVm) {
        // 1. Verify product exists (call product service)
        productService.verifyProductExists(productId);

        // 2. Extract customer ID from JWT token
        String customerId = AuthenticationUtils.getCurrentUserId();

        // 3. Create review entity
        Review review = new Review();
        review.setProductId(productId);
        review.setCustomerId(customerId);
        review.setRating(reviewPostVm.rating());
        review.setContent(reviewPostVm.content().trim());

        // 4. Save review (handle unique constraint violation)
        try {
            review = reviewRepository.save(review);
        } catch (DataIntegrityViolationException e) {
            throw new DuplicatedException("You have already reviewed this product");
        }

        // 5. Update product aggregates (average rating, review count)
        aggregateService.updateAggregatesAfterReviewCreated(productId, reviewPostVm.rating());

        // 6. Return response
        return new ReviewVm(
            review.getId(),
            review.getProductId(),
            review.getCustomerId(),
            review.getRating(),
            review.getContent(),
            review.getCreatedAt()
        );
    }
}
```

**Entity:** `Review.java`
```java
package com.yas.rating.model;

import jakarta.persistence.*;
import java.time.ZonedDateTime;

@Entity
@Table(name = "review", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"product_id", "customer_id"}, name = "unique_customer_product_review")
})
public class Review {

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

    @Column(name = "created_at", nullable = false, updatable = false)
    private ZonedDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private ZonedDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        createdAt = ZonedDateTime.now();
        updatedAt = ZonedDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = ZonedDateTime.now();
    }

    // Getters and setters
    // ...
}
```

**Repository:** `ReviewRepository.java`
```java
package com.yas.rating.repository;

import com.yas.rating.model.Review;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ReviewRepository extends JpaRepository<Review, Long> {
}
```

**Inter-service call:** `ProductService.java` (in `rating` service)
```java
package com.yas.rating.service;

import com.yas.commonlibrary.exception.NotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

@Service
public class ProductService {

    private final RestClient restClient;
    private final String productServiceUrl; // injected from application.properties

    public ProductService(RestClient restClient, String productServiceUrl) {
        this.restClient = restClient;
        this.productServiceUrl = productServiceUrl;
    }

    public void verifyProductExists(Long productId) {
        try {
            restClient.get()
                .uri(productServiceUrl + "/api/product/storefront/products/{id}", productId)
                .retrieve()
                .toBodilessEntity();
        } catch (Exception e) {
            throw new NotFoundException("Product not found");
        }
    }
}
```

**Aggregate update:** `AggregateService.java` (in `rating` service)
```java
package com.yas.rating.service;

import com.yas.rating.model.ProductAggregate;
import com.yas.rating.repository.ProductAggregateRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AggregateService {

    private final ProductAggregateRepository productAggregateRepository;

    public AggregateService(ProductAggregateRepository productAggregateRepository) {
        this.productAggregateRepository = productAggregateRepository;
    }

    @Transactional
    public void updateAggregatesAfterReviewCreated(Long productId, Integer newRating) {
        ProductAggregate aggregate = productAggregateRepository.findById(productId)
            .orElse(new ProductAggregate(productId, 0.0, 0));

        double oldAvg = aggregate.getAverageRating();
        int oldCount = aggregate.getReviewCount();

        double newAvg = ((oldAvg * oldCount) + newRating) / (oldCount + 1);
        int newCount = oldCount + 1;

        aggregate.setAverageRating(newAvg);
        aggregate.setReviewCount(newCount);

        productAggregateRepository.save(aggregate);
    }
}
```

**Entity:** `ProductAggregate.java`
```java
package com.yas.rating.model;

import jakarta.persistence.*;
import java.time.ZonedDateTime;

@Entity
@Table(name = "product_aggregate")
public class ProductAggregate {

    @Id
    @Column(name = "product_id")
    private Long productId;

    @Column(name = "average_rating", nullable = false, precision = 3, scale = 2)
    private Double averageRating;

    @Column(name = "review_count", nullable = false)
    private Integer reviewCount;

    @Column(name = "updated_at", nullable = false)
    private ZonedDateTime updatedAt;

    public ProductAggregate() {}

    public ProductAggregate(Long productId, Double averageRating, Integer reviewCount) {
        this.productId = productId;
        this.averageRating = averageRating;
        this.reviewCount = reviewCount;
        this.updatedAt = ZonedDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = ZonedDateTime.now();
    }

    // Getters and setters
    // ...
}
```

**Repository:** `ProductAggregateRepository.java`
```java
package com.yas.rating.repository;

import com.yas.rating.model.ProductAggregate;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ProductAggregateRepository extends JpaRepository<ProductAggregate, Long> {
}
```

### Frontend API Call

**File:** `backoffice/modules/rating/services/RatingService.ts` (already exists, extends for create)

**New method:**
```typescript
export async function createReview(productId: number, rating: number, content: string) {
  const url = `/api/rating/storefront/products/${productId}/reviews`;
  const body = JSON.stringify({ rating, content });
  const response = await apiClientService.post(url, body);
  return response;
}
```

**Usage in product detail page:** verify in `storefront/pages/products/[slug].tsx` (file not shown in repo-tree; assumed to exist based on backoffice structure)

---

## File Touch List (from repo-tree)

**New files (rating service):**
- `rating/pom.xml`
- `rating/Dockerfile`
- `rating/src/main/java/com/yas/rating/RatingApplication.java`
- `rating/src/main/java/com/yas/rating/config/SecurityConfig.java`
- `rating/src/main/java/com/yas/rating/config/DatabaseAutoConfig.java`
- `rating/src/main/java/com/yas/rating/config/RestClientConfig.java`
- `rating/src/main/java/com/yas/rating/config/ServiceUrlConfig.java`
- `rating/src/main/java/com/yas/rating/config/SwaggerConfig.java`
- `rating/src/main/java/com/yas/rating/controller/ReviewController.java`
- `rating/src/main/java/com/yas/rating/service/ReviewService.java`
- `rating/src/main/java/com/yas/rating/service/ProductService.java`
- `rating/src/main/java/com/yas/rating/service/AggregateService.java`
- `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java`
- `rating/src/main/java/com/yas/rating/repository/ProductAggregateRepository.java`
- `rating/src/main/java/com/yas/rating/model/Review.java`
- `rating/src/main/java/com/yas/rating/model/ProductAggregate.java`
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java`
- `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java`
- `rating/src/main/resources/application.properties`
- `rating/src/main/resources/db/changelog/db.changelog-master.yaml`
- `rating/src/main/resources/db/changelog/ddl/changelog-0001.sql`
- `rating/src/main/resources/messages/messages.properties`
- `rating/src/test/java/com/yas/rating/controller/ReviewControllerTest.java`
- `rating/src/test/java/com/yas/rating/service/ReviewServiceTest.java`

**Modified files (backoffice frontend):**
- `backoffice/modules/rating/services/RatingService.ts` (add `createReview` method)
- `backoffice/pages/reviews/index.tsx` (may need to fetch updated aggregates; verify if AC 7/8 require immediate UI update here)

**Modified files (storefront frontend, assumed to exist):**
- `storefront/pages/products/[slug].tsx` (add review form component)
- `storefront/modules/product/components/ReviewForm.tsx` (new component for rating/content submission)
- `storefront/modules/product/services/ReviewService.ts` (new service for `createReview` API call)
- `storefront/modules/product/models/Review.ts` (new TypeScript type for review data)

**Modified files (root):**
- `pom.xml` (add `rating` module to `<modules>` list)
- `docker-compose.yml` (add `rating` service container)

**Modified files (Kubernetes/Helm, if applicable):**
- `k8s/charts/rating/` (new Helm chart for rating service, mirroring `k8s/charts/cart/`, `k8s/charts/customer/`, etc.)

---

## Risks

### 1. `customer_id` Extraction from JWT Token

**Risk:**  
The `customer_id` field in the `review` table is populated from `AuthenticationUtils.getCurrentUserId()`, which assumes the JWT token contains a user ID claim. If the token structure does not include this claim, the service will fail.

**Mitigation:**  
- Verify JWT token structure in `common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java`.
- If the token does not contain `customer_id`, call the `customer` service to fetch it via `GET /api/customer/storefront/customer/profile` (observed in `backoffice/modules/customer/services/CustomerService.ts`).
- Fallback: extract the Keycloak user ID from the token's `sub` claim (standard JWT claim for subject/user ID).

### 2. Product Existence Validation

**Risk:**  
The `rating` service must verify that `product_id` exists before saving a review. This requires calling the `product` service. If the `product` service is unavailable or the product does not exist, the review creation fails.

**Mitigation:**  
- Use the existing `ProductService` pattern (observed in `cart/src/main/java/com/yas/cart/service/ProductService.java`, `customer/src/main/java/com/yas/customer/service/ProductService.java`, `order/src/main/java/com/yas/order/service/ProductService.java`).
- Call `GET /api/product/storefront/products/{id}` to verify product existence.
- Return `404 Not Found` to the client if the product does not exist.
- Implement circuit breaker (Resilience4j, observed in `AbstractCircuitBreakFallbackHandler` in `cart`, `customer`, `order` services) to handle transient failures of the `product` service.

### 3. Aggregate Update Race Condition

**Risk:**  
If two reviews for the same product are submitted concurrently, the aggregate update (average rating, review count) could race, leading to incorrect values.

**Mitigation:**  
- Wrap the aggregate update in a database transaction (already specified in `AggregateService.updateAggregatesAfterReviewCreated`).
- Use row-level locking: `SELECT FOR UPDATE` on the `product_aggregate` row before updating.
- Alternative: use atomic SQL updates:
  ```sql
  UPDATE product_aggregate
  SET average_rating = ((average_rating * review_count) + ?) / (review_count + 1),
      review_count = review_count + 1
  WHERE product_id = ?;
  ```
- Implement optimistic locking (`@Version` column in `ProductAggregate` entity) to detect and retry conflicts.

### 4. Review Content Validation (Whitespace-only)

**Risk:**  
The `@NotBlank` annotation in `ReviewPostVm` prevents empty strings but may not catch whitespace-only content (e.g. `"   "`).

**Mitigation:**  
- Trim the content in `ReviewService.createReview` before saving: `review.setContent(reviewPostVm.content().trim());`.
- Add a custom validation annotation `@NotBlankAfterTrim` or validate in the service layer:
  ```java
  if (reviewPostVm.content() == null || reviewPostVm.content().trim().isEmpty()) {
      throw new BadRequestException("Review content is required");
  }
  ```

### 5. Idempotency Enforcement (Unique Constraint Violation)

**Risk:**  
The unique constraint on `(product_id, customer_id)` is enforced at the database level. If the constraint violation is not caught correctly, the service may throw a generic exception instead of a `409 Conflict`.

**Mitigation:**  
- Catch `DataIntegrityViolationException` in `ReviewService.createReview` (already specified).
- Inspect the exception message to confirm it is a unique constraint violation (PostgreSQL error code `23505`).
- Return a clear `409 Conflict` response with message "You have already reviewed this product."

### 6. Frontend Idempotency (Double-click Prevention)

**Risk:**  
Disabling the submit button while a request is in-flight (UI-level idempotency) may not prevent duplicate submissions if the user opens multiple tabs or refreshes the page.

**Mitigation:**  
- Rely on the database unique constraint as the primary idempotency mechanism (already specified).
- UI-level prevention (disable button) is a UX improvement, not a security guarantee.
- If a duplicate request reaches the backend, return `409 Conflict` and display an error message.

### 7. Review Persistence Across Page Loads (Cache Invalidation)

**Risk:**  
After a review is submitted, the product detail page must fetch the updated reviews list and aggregates (average rating, review count). If the frontend caches the product data, stale data may be displayed.

**Mitigation:**  
- After a successful `POST /products/{id}/reviews` response, refetch the reviews list via `GET /products/{id}/reviews` (endpoint not specified in HLD; must be added).
- Refetch the product aggregates (average rating, count) via `GET /products/{id}` or a dedicated aggregates endpoint.
- Invalidate any cached product data on the frontend (verify caching strategy in `storefront/` frontend code, not shown in repo-tree).

### 8. Missing `GET /products/{id}/reviews` Endpoint

**Risk:**  
The HLD specifies the `POST` endpoint for creating reviews but does not define a `GET` endpoint for fetching reviews. The UI cannot display the reviews list without this endpoint.

**Mitigation:**  
- Add `GET /api/rating/storefront/products/{productId}/reviews` endpoint in `ReviewController.java`:
  ```java
  @GetMapping("/{productId}/reviews")
  public ResponseEntity<List<ReviewVm>> getReviewsByProductId(@PathVariable Long productId) {
      List<ReviewVm> reviews = reviewService.getReviewsByProductId(productId);
      return ResponseEntity.ok(reviews);
  }
  ```
- Add query method in `ReviewRepository.java`:
  ```java
  List<Review> findByProductIdOrderByCreatedAtDesc(Long productId);
  ```
- Implement in `ReviewService.java`:
  ```java
  public List<ReviewVm> getReviewsByProductId(Long productId) {
      List<Review> reviews = reviewRepository.findByProductIdOrderByCreatedAtDesc(productId);
      return reviews.stream()
          .map(review -> new ReviewVm(
              review.getId(),
              review.getProductId(),
              review.getCustomerId(),
              review.getRating(),
              review.getContent(),
              review.getCreatedAt()
          ))
          .toList();
  }
  ```

### 9. Missing `GET /products/{id}/aggregates` Endpoint

**Risk:**  
The HLD specifies the `product_aggregate` table but does not define an endpoint for fetching aggregates (average rating, review count). The UI cannot display updated aggregates without this endpoint.

**Mitigation:**  
- Add `GET /api/rating/storefront/products/{productId}/aggregates` endpoint in `ReviewController.java`:
  ```java
  @GetMapping("/{productId}/aggregates")
  public ResponseEntity<ProductAggregateVm> getProductAggregates(@PathVariable Long productId) {
      ProductAggregateVm aggregates = aggregateService.getAggregatesByProductId(productId);
      return ResponseEntity.ok(aggregates);
  }
  ```
- Add DTO `ProductAggregateVm.java`:
  ```java
  public record ProductAggregateVm(
      Long productId,
      Double averageRating,
      Integer reviewCount
  ) {}
  ```
- Implement in `AggregateService.java`:
  ```java
  public ProductAggregateVm getAggregatesByProductId(Long productId) {
      ProductAggregate aggregate = productAggregateRepository.findById(productId)
          .orElse(new ProductAggregate(productId, 0.0, 0));
      return new ProductAggregateVm(
          aggregate.getProductId(),
          aggregate.getAverageRating(),
          aggregate.getReviewCount()
      );
  }
  ```

### 10. Review Content Maximum Length

**Risk:**  
The `content` field is defined as `TEXT` in PostgreSQL, which supports up to 1 GB. Without a maximum length constraint, customers could submit extremely long reviews, causing performance issues or abuse.

**Mitigation:**  
- Set a reasonable maximum length (e.g. 1000 characters) in `ReviewPostVm.java`:
  ```java
  @Size(max = 1000, message = "Review content must not exceed 1000 characters")
  String content
  ```
- Enforce at the database level (optional):
  ```sql
  ALTER TABLE review ADD CONSTRAINT content_length_check CHECK (LENGTH(content) <= 1000);
  ```

### 11. Rating Service Database Configuration

**Risk:**  
The `rating` service requires its own PostgreSQL database. If the database is not created or the connection is misconfigured, the service will fail to start.

**Mitigation:**  
- Add `rating` database to `postgres_init.sql`:
  ```sql
  CREATE DATABASE rating;
  ```
- Configure `rating/src/main/resources/application.properties`:
  ```properties
  spring.datasource.url=jdbc:postgresql://postgres:5432/rating
  spring.datasource.username=postgres
  spring.datasource.password=postgres
  spring.jpa.hibernate.ddl-auto=validate
  spring.liquibase.enabled=true
  spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.yaml
  server.port=8095
  ```
- Add `rating` service to `docker-compose.yml`:
  ```yaml
  rating:
    build: ./rating
    ports:
      - "8095:8095"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/rating
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=postgres
    depends_on:
      - postgres
  ```

### 12. Keycloak JWT Token Validation

**Risk:**  
The `rating` service must validate JWT tokens issued by Keycloak. If the token validation is misconfigured, unauthorized requests may be accepted.

**Mitigation:**  
- Configure Spring Security in `SecurityConfig.java` (follow pattern in `cart/src/main/java/com/yas/cart/config/SecurityConfig.java`, `customer/src/main/java/com/yas/customer/config/SecurityConfig.java`, `order/src/main/java/com/yas/order/config/SecurityConfig.java`):
  ```java
  @Configuration
  @EnableWebSecurity
  public class SecurityConfig {

      @Bean
      public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
          http
              .authorizeHttpRequests(auth -> auth
                  .requestMatchers("/api/rating/storefront/**").authenticated()
                  .anyRequest().permitAll()
              )
              .oauth2ResourceServer(oauth2 -> oauth2
                  .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
              );
          return http.build();
      }

      @Bean
      public JwtAuthenticationConverter jwtAuthenticationConverter() {
          JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
          // Configure converter to extract roles/authorities from JWT
          return converter;
      }
  }
  ```
- Configure `rating/src/main/resources/application.properties`:
  ```properties
  spring.security.oauth2.resourceserver.jwt.issuer-uri=http://keycloak:8080/realms/yas
  ```

---

## Open Questions (to be resolved before implementation)

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

5. **Review Deletion:**  
   Can customers delete their reviews? Can admins delete reviews?  
   **Decision Required:** Ticket AC 10 implies failure scenarios but not deletion. Existing backoffice page `backoffice/pages/reviews/index.tsx` has a "Delete" button (line 97), so admin deletion is already supported. Customer deletion is not in scope for this ticket.

6. **Pagination for Reviews:**  
   Should the reviews list be paginated?  
   **Decision Required:** Not specified in ticket. For initial implementation, return all reviews (up to a reasonable limit, e.g. 50). Add pagination later if needed.

7. **Customer Display Name:**  
   Should reviews display the customer's full name (first name + last name) or just the customer ID?  
   **Decision Required:** Existing backoffice page `backoffice/pages/reviews/index.tsx` displays `lastName` and `firstName` (line 142). The `rating` service must fetch customer details from the `customer` service or store customer names in the `review` table. Recommend calling `GET /api/customer/backoffice/customers/profile/{customerId}` to fetch customer names for display.

8. **Product Name in Review:**  
   Should the `review` table store the product name, or should it be fetched from the `product` service when displaying reviews?  
   **Decision Required:** Existing backoffice page `backoffice/pages/reviews/index.tsx` displays `productName` (line 141). The `rating` service must fetch product details from the `product` service or store product names in the `review` table. Recommend calling `GET /api/product/storefront/products/{productId}` to fetch product names for display.

---

## Notes

- The `rating` service does not currently exist in the YAS codebase. This ticket requires creating a new service following the existing structure (Spring Boot, JPA, Liquibase, REST API).
- The `product` service **may** cache average rating and count for display on product list pages (not detail pages). If caching is needed, the `rating` service can publish events (via Kafka) on review insert, and the `product` service can consume these events to update its cache. This is deferred until required.
- The frontend uses Next.js and TypeScript, following the existing pattern in `storefront/pages/products/[slug].tsx` (assumed to exist based on backoffice structure) and `storefront/modules/product/`.
- The API follows the YAS convention: `/api/<service>/storefront/<resource>` for customer-facing endpoints.
- The backoffice page for managing reviews (`backoffice/pages/reviews/index.tsx`) already exists and displays reviews with customer names, product names, and a delete button. The new backend API must support fetching this data (customer and product details must be joined from their respective services).
- The `AuthenticationUtils.getCurrentUserId()` method (in `common-library`) assumes the JWT token contains a user ID claim. Verify this in the Keycloak token structure before implementation.

## Spec `specs/2.md`

# specs/2.md

**Ticket:** Customer Can write and Submit Product Reviews  
**Epic:** _(none)_  
**Initiative:** _(none)_  
**Story:** Implement Product Review Submission Backend API and Storage

---

## Summary

Backend implementation for product review submission: REST API endpoint, database schema, validation, aggregate calculation, and idempotency enforcement.

---

## Stack

**Repository:** `app`  
**Base Branch:** `main`  
**Languages/Frameworks:**
- Java 17+ / Spring Boot 3.x (backend services follow this pattern: `product`, `customer`, `order`, `inventory`)
- PostgreSQL (database)
- Liquibase (schema migrations)
- Spring Data JPA (persistence)
- Spring Security (authentication via Keycloak JWT)
- Maven (build)

**Existing Services (from `repo-outline/app`):**
- `product` service: manages product catalog
- `customer` service: manages customer identity and wishlist
- `order` service: manages orders
- `cart` service: manages shopping cart
- `rating` service: **does not exist yet** — this story creates it

**Conventions (from `repo-context/app.md`):**
- Test: `mvn test`
- Lint: `mvn checkstyle:check`
- Existing services follow structure: `<service>/src/main/java/com/yas/<service>/` for source, `<service>/src/main/resources/db/changelog/` for Liquibase migrations
- REST API path convention: `/api/<service>/storefront/<resource>` for customer-facing endpoints, `/api/<service>/backoffice/<resource>` for admin endpoints

---

## Schema/API Contracts

### Database Schema (PostgreSQL)

**New Tables:**

1. **`review` table** (stores individual review records):
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

2. **`product_aggregate` table** (stores denormalized average rating and review count):
   ```sql
   CREATE TABLE product_aggregate (
     product_id BIGINT PRIMARY KEY,
     average_rating DECIMAL(3,2) NOT NULL,
     review_count INT NOT NULL DEFAULT 0,
     updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

**Constraints:**
- `product_id` references the `product` table in the `product` service (enforced via application logic, not FK, as services are separate).
- `customer_id` is the user ID from Keycloak (UUID string extracted from JWT).
- `unique_customer_product_review` constraint enforces one review per customer per product.

---

### API Contract

**Endpoint:** `POST /api/rating/storefront/products/{productId}/reviews`

**Request Headers:**
- `Authorization: Bearer <JWT token>` (Keycloak-issued token containing `customer_id` claim)

**Request Body (JSON):**
```json
{
  "rating": 5,
  "content": "Great product!"
}
```

**Fields:**
- `rating` (integer, required): 1–5 inclusive
- `content` (string, required): non-empty, non-whitespace-only

**Success Response (201 Created):**
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

**Error Responses:**

**400 Bad Request (validation failure):**
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

**401 Unauthorized (not authenticated):**
```json
{
  "status": 401,
  "title": "Unauthorized",
  "detail": "Authentication required"
}
```

**404 Not Found (product does not exist):**
```json
{
  "status": 404,
  "title": "Not Found",
  "detail": "Product not found"
}
```

**409 Conflict (duplicate review):**
```json
{
  "status": 409,
  "title": "Conflict",
  "detail": "You have already reviewed this product"
}
```

**500 Internal Server Error:**
```json
{
  "status": 500,
  "title": "Internal Server Error",
  "detail": "An error occurred while saving the review"
}
```

---

**Endpoint:** `GET /api/rating/storefront/products/{productId}/reviews`

**Response (200 OK):**
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

---

## File Touch List (from `repo-tree/app`)

**New Files:**

1. **Service Module (new `rating` service):**
   - `rating/pom.xml` — Maven POM inheriting from parent POM
   - `rating/Dockerfile` — Docker image definition (follow pattern from `product/Dockerfile`)
   - `rating/src/main/java/com/yas/rating/RatingApplication.java` — Spring Boot main class
   - `rating/src/main/java/com/yas/rating/config/DatabaseAutoConfig.java` — JPA config (follow `product/src/main/java/com/yas/product/config/DatabaseAutoConfig.java`)
   - `rating/src/main/java/com/yas/rating/config/RestClientConfig.java` — REST client config for inter-service calls
   - `rating/src/main/java/com/yas/rating/config/SecurityConfig.java` — Spring Security config (Keycloak JWT authentication, follow `product/src/main/java/com/yas/product/config/SecurityConfig.java`)
   - `rating/src/main/java/com/yas/rating/config/SwaggerConfig.java` — Swagger API docs config
   - `rating/src/main/java/com/yas/rating/controller/ReviewController.java` — REST controller for review endpoints
   - `rating/src/main/java/com/yas/rating/service/ReviewService.java` — Business logic: validation, persistence, aggregate updates
   - `rating/src/main/java/com/yas/rating/service/ProductService.java` — Circuit-breaker-wrapped client for product service (verify product exists, follow `cart/src/main/java/com/yas/cart/service/ProductService.java`)
   - `rating/src/main/java/com/yas/rating/repository/ReviewRepository.java` — JPA repository for `Review` entity
   - `rating/src/main/java/com/yas/rating/model/Review.java` — JPA entity mapping `review` table
   - `rating/src/main/java/com/yas/rating/model/ProductAggregate.java` — JPA entity mapping `product_aggregate` table
   - `rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java` — DTO for review creation request (`productId`, `rating`, `content`)
   - `rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java` — DTO for review response (`id`, `productId`, `customerId`, `rating`, `content`, `createdAt`)
   - `rating/src/main/java/com/yas/rating/viewmodel/ReviewListVm.java` — DTO for review list response (wraps list of `ReviewVm` + aggregates)
   - `rating/src/main/resources/application.properties` — Spring Boot config (follow `product/src/main/resources/application.properties`)
   - `rating/src/main/resources/logback-spring.xml` — Logging config
   - `rating/src/main/resources/db/changelog/db.changelog-master.yaml` — Liquibase master changelog
   - `rating/src/main/resources/db/changelog/ddl/changelog-review-table.sql` — Liquibase changeset to create `review` table
   - `rating/src/main/resources/db/changelog/ddl/changelog-product-aggregate-table.sql` — Liquibase changeset to create `product_aggregate` table
   - `rating/src/main/resources/messages/messages.properties` — Internationalization messages (error messages, follow `product/src/main/resources/messages/messages.properties`)
   - `rating/src/test/java/com/yas/rating/controller/ReviewControllerTest.java` — Unit tests for `ReviewController`
   - `rating/src/test/java/com/yas/rating/service/ReviewServiceTest.java` — Unit tests for `ReviewService`
   - `rating/src/test/resources/application.properties` — Test-specific Spring Boot config

2. **Docker Compose (if not already present):**
   - Verify `rating` service is added to `docker-compose.yml` (follow pattern of other services)

3. **Kubernetes (if deploying to K8s):**
   - Verify `k8s/charts/rating/` Helm chart is created (follow pattern of other services, e.g. `k8s/charts/product/`)

**Modified Files:**

1. **Parent POM (if `rating` is a new Maven module):**
   - `pom.xml` — Add `<module>rating</module>` to `<modules>` section

2. **Backoffice UI (admin can delete reviews):**
   - `backoffice/modules/rating/services/RatingService.ts` — Already exists (see `repo-outline/app`); verify `deleteRatingById` calls the new backend endpoint
   - `backoffice/pages/reviews/index.tsx` — Already exists (see `repo-file/app/backoffice/pages/reviews/index.tsx`); verify it fetches reviews from the new `rating` service

3. **Storefront UI (customers submit and view reviews):**
   - See Story 2 (frontend implementation) — no changes in this backend story

---

## Risks

1. **Concurrency on Aggregate Updates:**
   - **Risk:** Two reviews for the same product submitted simultaneously could race when updating `product_aggregate`.
   - **Mitigation:** Use database-level atomic operations (`UPDATE product_aggregate SET average_rating = ..., review_count = review_count + 1 WHERE product_id = ?`) within the transaction. Alternatively, use row-level locking (`SELECT FOR UPDATE`) on the `product_aggregate` row before updating.

2. **Customer Identity from JWT Token:**
   - **Risk:** Extracting `customer_id` from the JWT token assumes the token contains a user ID claim.
   - **Mitigation:** Verify the token structure in the existing `customer` or `order` services (see `customer/src/main/java/com/yas/customer/service/CustomerService.java` and `order/src/main/java/com/yas/order/service/CustomerService.java` for how they extract user identity). If the token does not contain `customer_id`, call the `customer` service to fetch it via `GET /api/customer/storefront/customer/profile`.

3. **Product Existence Validation:**
   - **Risk:** The `rating` service must verify that `product_id` exists before saving a review. This requires calling the `product` service.
   - **Mitigation:** Use the existing `ProductService` pattern (observed in `cart/src/main/java/com/yas/cart/service/ProductService.java`, `customer/src/main/java/com/yas/customer/service/ProductService.java`, `inventory/src/main/java/com/yas/inventory/service/ProductService.java`, `order/src/main/java/com/yas/order/service/ProductService.java`) to call `GET /api/product/storefront/products/{id}`. If the product is not found, return 404 to the client.

4. **Review Content Abuse:**
   - **Risk:** Customers could submit very long content or malicious HTML/JS.
   - **Mitigation:** Enforce a maximum content length (e.g. 1000 characters) at the API level using `@Size(max = 1000)` on the `ReviewPostVm.content` field. Sanitize content before storing (strip HTML tags, escape special characters) or store as plain text and escape on display (handled in frontend Story 2).

5. **Average Rating Precision:**
   - **Risk:** Storing `average_rating` as `DECIMAL(3,2)` limits precision to two decimal places (e.g. 4.56).
   - **Mitigation:** Acceptable for star ratings displayed to customers. If higher precision is needed, use `DECIMAL(5,4)` or store the sum and count separately and calculate on read.

6. **Initial Aggregate Record Creation:**
   - **Risk:** When the first review is submitted for a product, the `product_aggregate` row does not exist yet.
   - **Mitigation:** Use `INSERT ... ON CONFLICT DO UPDATE` (PostgreSQL upsert) or `MERGE` (SQL standard) to atomically create or update the aggregate row. Example:
     ```sql
     INSERT INTO product_aggregate (product_id, average_rating, review_count)
     VALUES (?, ?, 1)
     ON CONFLICT (product_id) DO UPDATE SET
       average_rating = ((product_aggregate.average_rating * product_aggregate.review_count) + EXCLUDED.average_rating) / (product_aggregate.review_count + 1),
       review_count = product_aggregate.review_count + 1,
       updated_at = CURRENT_TIMESTAMP;
     ```

7. **Database Constraint Violation Handling:**
   - **Risk:** If the unique constraint on `(product_id, customer_id)` is violated (duplicate review), the database throws a `ConstraintViolationException`.
   - **Mitigation:** Catch `org.springframework.dao.DataIntegrityViolationException` in the service layer and return a 409 Conflict response with message "You have already reviewed this product."

8. **Backoffice UI Dependency:**
   - **Risk:** The existing `backoffice/pages/reviews/index.tsx` (see `repo-file/app/backoffice/pages/reviews/index.tsx`) assumes a `getRatings` API endpoint exists.
   - **Mitigation:** Implement `GET /api/rating/backoffice/reviews` endpoint in the new `rating` service to support the backoffice UI. This endpoint should:
     - Accept query parameters: `pageNo`, `pageSize`, `createdFrom`, `createdTo`, etc. (see `RatingSearch` component in `backoffice/modules/rating/components/RatingSearch.tsx`).
     - Return paginated list of reviews with customer name (join with `customer` service or store denormalized `firstName`/`lastName` in `review` table).
     - Support the existing `deleteRatingById` endpoint: `DELETE /api/rating/backoffice/reviews/{id}`.
   - **Note:** The backoffice endpoints are out of scope for the customer-facing acceptance criteria, but are needed for the existing UI to work. Add them to this story or defer to a follow-up task.

---

## Implementation Notes

1. **Liquibase Migrations:**
   - Create `rating/src/main/resources/db/changelog/ddl/changelog-review-table.sql`:
     ```sql
     --liquibase formatted sql
     --changeset yas:create_review_table
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
   - Create `rating/src/main/resources/db/changelog/ddl/changelog-product-aggregate-table.sql`:
     ```sql
     --liquibase formatted sql
     --changeset yas:create_product_aggregate_table
     CREATE TABLE product_aggregate (
       product_id BIGINT PRIMARY KEY,
       average_rating DECIMAL(3,2) NOT NULL,
       review_count INT NOT NULL DEFAULT 0,
       updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
     );
     ```
   - Reference these changesets in `rating/src/main/resources/db/changelog/db.changelog-master.yaml`:
     ```yaml
     databaseChangeLog:
       - include:
           file: db/changelog/ddl/changelog-review-table.sql
       - include:
           file: db/changelog/ddl/changelog-product-aggregate-table.sql
     ```

2. **JPA Entities:**
   - `Review` entity (follow pattern from `product/src/main/java/com/yas/product/model/Product.java`):
     - Extends `com.yas.commonlibrary.model.AbstractAuditEntity` for `createdAt`/`updatedAt` audit fields (see `common-library/src/main/java/com/yas/commonlibrary/model/AbstractAuditEntity.java`).
     - Maps to `review` table.
     - Fields: `id` (Long), `productId` (Long), `customerId` (String), `rating` (Integer), `content` (String).
   - `ProductAggregate` entity:
     - Does not extend `AbstractAuditEntity` (simpler entity).
     - Maps to `product_aggregate` table.
     - Fields: `productId` (Long, primary key), `averageRating` (BigDecimal), `reviewCount` (Integer), `updatedAt` (Instant).

3. **Service Layer Logic (ReviewService):**
   - **createReview(ReviewPostVm):**
     1. Extract `customerId` from SecurityContext (follow pattern in `customer/src/main/java/com/yas/customer/service/CustomerService.java` using `com.yas.commonlibrary.utils.AuthenticationUtils.getCurrentUserId()`).
     2. Validate `rating` (1–5) and `content` (non-empty, non-whitespace).
     3. Call `ProductService.getProduct(productId)` to verify product exists (throws 404 if not found).
     4. Create `Review` entity and save via `ReviewRepository.save(review)`. Catch `DataIntegrityViolationException` for duplicate review (unique constraint) and throw 409 Conflict.
     5. Update `ProductAggregate`: use upsert query (see Risk 6) to atomically insert or update the aggregate row.
     6. Return `ReviewVm` DTO.

4. **REST Controller (ReviewController):**
   - Annotate with `@RestController` and `@RequestMapping("/storefront/products")`.
   - `@PostMapping("/{productId}/reviews")`:
     - Accept `@PathVariable Long productId` and `@RequestBody @Valid ReviewPostVm reviewPostVm`.
     - Validate `reviewPostVm` using Bean Validation annotations (`@Min(1)`, `@Max(5)`, `@NotBlank`, `@Size(max = 1000)` on fields).
     - Call `reviewService.createReview(productId, reviewPostVm)`.
     - Return `ResponseEntity<ReviewVm>` with 201 status.
     - Handle exceptions: return 400 for validation errors, 404 for product not found, 409 for duplicate review, 500 for internal errors (follow pattern in `product/src/main/java/com/yas/product/controller/ProductController.java`).

5. **Inter-Service Communication (ProductService):**
   - Follow pattern from `cart/src/main/java/com/yas/cart/service/ProductService.java`:
     - Extends `AbstractCircuitBreakFallbackHandler` (see `cart/src/main/java/com/yas/cart/service/AbstractCircuitBreakFallbackHandler.java`).
     - Use `RestClient` to call `GET /api/product/storefront/products/{id}`.
     - Apply `@CircuitBreaker` annotation for resilience (follow Spring Cloud Circuit Breaker pattern).

6. **Security Configuration:**
   - Follow pattern from `product/src/main/java/com/yas/product/config/SecurityConfig.java`:
     - Configure Keycloak JWT authentication.
     - Permit `/storefront/products/{productId}/reviews` (GET) without authentication (public read).
     - Require authentication for `/storefront/products/{productId}/reviews` (POST) — only authenticated users can submit reviews.
     - Require `ROLE_ADMIN` for `/backoffice/reviews/**` (admin delete endpoint).

7. **Testing:**
   - Unit tests:
     - `ReviewControllerTest` (follow `cart/src/test/java/com/yas/cart/controller/CartItemControllerTest.java`): mock `ReviewService`, test request validation, response status codes.
     - `ReviewServiceTest` (follow `cart/src/test/java/com/yas/cart/service/CartItemServiceTest.java`): mock `ReviewRepository`, `ProductService`, test business logic (aggregate calculation, duplicate handling, product validation).
   - Integration tests (optional for this story, can be in Story 3):
     - `ReviewControllerIT` (follow `cart/src/it/java/com/yas/cart/controller/CartItemControllerIT.java`): test full flow with database and security context.

---

## Acceptance Criteria Coverage

This story implements backend support for the following acceptance criteria from the ticket:

- **AC 1:** Backend validates `rating` is between 1 and 5 (database constraint + API validation).
- **AC 2:** Backend validates `content` is mandatory and non-whitespace (`@NotBlank` annotation).
- **AC 3:** Backend prevents submission if `rating` or `content` is missing (validation at API level).
- **AC 4:** Backend rejects invalid ratings via API validation (400 Bad Request).
- **AC 5:** Backend saves review and returns success response (201 Created).
- **AC 7:** Backend recalculates product average rating based on all valid reviews (aggregate update in `product_aggregate` table).
- **AC 8:** Backend increments total review count by one (aggregate update in `product_aggregate` table).
- **AC 10:** Backend returns appropriate error messages on failure (400, 404, 409, 500 responses).
- **AC 11:** Backend prevents duplicate reviews via unique constraint on `(product_id, customer_id)` (idempotency).
- **AC 12:** Backend persists reviews in database (reviews survive restarts and are queryable).

**Frontend-focused ACs (AC 6, AC 9) are covered in Story 2 (frontend implementation).**

---

## Open Questions (from hld.md)

1. **Authentication Requirement:** Ticket implies authentication (customer identity required). Recommend requiring authentication — `customer_id` is extracted from JWT token.
2. **Review Moderation:** Ticket implies immediate visibility. No moderation workflow in this story. Can be added later via `status` field (`PENDING`, `APPROVED`, `REJECTED`) in `review` table.
3. **Review Content Length:** Set maximum 1000 characters (enforced at API level and database schema).
4. **Review Editing:** Not in scope for this story. Can be added later via `PUT /reviews/{id}` endpoint.
5. **Review Deletion:** Backoffice UI already has delete functionality (`backoffice/pages/reviews/index.tsx` calls `deleteRatingById`). Implement `DELETE /api/rating/backoffice/reviews/{id}` endpoint in this story to support the existing UI. Recalculate aggregates on deletion.
6. **Pagination for Reviews:** Not specified in ticket. For initial implementation, return all reviews (up to a reasonable limit, e.g. 50) in `GET /api/rating/storefront/products/{productId}/reviews`. Add pagination parameters (`pageNo`, `pageSize`) later if needed.

---

## Notes

- The `rating` service does not currently exist in the YAS codebase. This story creates a new Spring Boot service following the existing structure (`product`, `customer`, `order`, `inventory`).
- The backoffice UI (`backoffice/pages/reviews/index.tsx`) already exists and expects a reviews API. This story must implement the backend endpoints to support it.
- The API follows the YAS convention: `/api/<service>/storefront/<resource>` for customer-facing endpoints, `/api/<service>/backoffice/<resource>` for admin endpoints.
- Aggregate calculation uses denormalized `product_aggregate` table for fast reads (pre-computed average rating and count), updated transactionally after each review insert.
- Idempotency is enforced by database unique constraint on `(product_id, customer_id)` — the simplest and most reliable approach.

---

# Story summaries

- `01-app`: Story 01-app: Backend Review Submission API and Database Schema
- `02-app`: Story 02-app: Frontend Review Submission Form and Display
- `03-app`: Story 03-app: Backoffice Review Management Endpoints

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
