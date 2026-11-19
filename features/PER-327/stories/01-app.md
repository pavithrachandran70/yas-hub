# Story 01-app LLD: Backend review submission endpoint with validation and rating recalculation

**Repo:** `app`

---

## Overview

This LLD details the implementation of the backend review submission feature for story 01-app. It covers the REST endpoints, service layer validation and rating aggregation, database schema changes, and integration between the `rating` and `product` services. The implementation enables customers to submit product reviews (1-5 star rating + written content) with server-side validation, duplicate prevention via database constraint, and automatic recalculation of product aggregate statistics (average rating, total review count).

---

## Acceptance Criteria (from requirements/breakdown)

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

---

## File Plan

### New Files

#### Rating Service (Backend)

1. **`rating/src/main/java/com/yas/rating/controller/StorefrontReviewController.java`**
   - New controller for storefront review endpoints
   - Endpoints: `POST /api/rating/storefront/reviews`, `GET /api/rating/storefront/reviews`
   - Depends on: `RatingService`, `AuthenticationUtils`, `ReviewPostVm`, `ReviewVm`, `ReviewListVm`

2. **`rating/src/main/java/com/yas/rating/viewmodel/ReviewPostVm.java`**
   - Request DTO for review submission
   - Fields: `Long productId`, `Integer rating`, `String content`
   - Validation annotations: `@NotNull`, `@Min(1)`, `@Max(5)`, `@NotBlank`

3. **`rating/src/main/java/com/yas/rating/viewmodel/ReviewVm.java`**
   - Response DTO for single review
   - Fields: `Long id`, `Long productId`, `String customerId`, `Integer rating`, `String content`, `ZonedDateTime createdOn`

4. **`rating/src/main/java/com/yas/rating/viewmodel/ReviewListVm.java`**
   - Paginated response DTO for review list
   - Fields: `List<ReviewVm> reviews`, `int totalPages`, `long totalElements`, `int pageNo`, `int pageSize`

5. **`rating/src/main/resources/db/changelog/ddl/changelog-rating-unique-constraint.sql`**
   - Liquibase migration to add unique constraint `(customer_id, product_id)` to `rating` table
   - Also adds indexes: `idx_rating_product_id` on `product_id`, `idx_rating_customer_product` on `(customer_id, product_id)`

#### Product Service (Backend)

6. **`product/src/main/java/com/yas/product/controller/ProductAggregateController.java`**
   - New backoffice controller for aggregate updates
   - Endpoint: `PUT /api/product/backoffice/products/{productId}/rating-aggregates`
   - Depends on: `ProductService`, `RatingAggregatePostVm`

7. **`product/src/main/java/com/yas/product/viewmodel/RatingAggregatePostVm.java`**
   - Request DTO for aggregate update
   - Fields: `BigDecimal averageRating`, `Long reviewCount`
   - Validation annotations: `@NotNull`, `@DecimalMin("0.0")`, `@DecimalMax("5.0")`, `@Min(0)`

8. **`product/src/main/resources/db/changelog/ddl/changelog-product-rating-aggregates.sql`**
   - Liquibase migration to add `average_rating` DECIMAL(3,1) and `review_count` BIGINT columns to `product` table (only if missing)

#### Tests

9. **`rating/src/test/java/com/yas/rating/service/RatingServiceTest.java`**
   - Unit tests for `RatingService`: validation, duplicate detection, aggregate calculation
   - 7 new test methods

10. **`rating/src/test/java/com/yas/rating/controller/StorefrontReviewControllerTest.java`**
    - Unit tests for `StorefrontReviewController`: endpoint contract, request/response format, status codes
    - 1 new test method

11. **`rating/src/it/java/com/yas/rating/controller/StorefrontReviewControllerIT.java`**
    - Integration tests for full flow: POST → save → aggregate update → GET retrieval
    - 5 new test methods

12. **`rating/src/it/java/com/yas/rating/migration/LiquibaseMigrationIT.java`**
    - Integration tests for Liquibase migrations: unique constraint, indexes, product aggregate columns
    - 2 new test methods

13. **`product/src/test/java/com/yas/product/service/ProductServiceTest.java`**
    - Unit tests for `ProductService.updateRatingAggregates` method
    - 1 new test method

### Modified Files

1. **`rating/src/main/java/com/yas/rating/service/RatingService.java`**
   - Add methods:
     - `ReviewVm createReview(ReviewPostVm request, String customerId)`
     - `ReviewListVm getReviewsByProductId(Long productId, int pageNo, int pageSize)`
     - `Map<String, Object> calculateAggregates(Long productId)`

2. **`rating/src/main/java/com/yas/rating/model/Rating.java`**
   - Verify existing schema matches spec
   - Add audit fields if missing: `createdBy`, `lastModifiedBy`

3. **`rating/src/main/java/com/yas/rating/repository/RatingRepository.java`**
   - Add query methods:
     - `Page<Rating> findByProductIdOrderByCreatedOnDesc(Long productId, Pageable pageable);`
     - `List<Rating> findByProductId(Long productId);`

4. **`product/src/main/java/com/yas/product/service/ProductService.java`**
   - Add method:
     - `Map<String, Object> updateRatingAggregates(Long productId, BigDecimal averageRating, Long reviewCount)`

5. **`product/src/main/java/com/yas/product/model/Product.java`**
   - Add fields (only if missing):
     - `@Column(name = "average_rating") private BigDecimal averageRating;`
     - `@Column(name = "review_count") private Long reviewCount;`

---

## Tests

### Unit Tests (8)

**File:** `rating/src/test/java/com/yas/rating/service/RatingServiceTest.java`

1. **`createReview_validInput_savesReview()`**
   - Given valid `ReviewPostVm` and `customerId`
   - When `createReview` called
   - Then `Rating` saved with correct fields, aggregate updated, `ReviewVm` returned

2. **`createReview_ratingBelowOne_throwsBadRequestException()`**
   - Given `rating = 0`
   - When `createReview` called
   - Then `BadRequestException` thrown with message "Rating must be between 1 and 5."

3. **`createReview_ratingAboveFive_throwsBadRequestException()`**
   - Given `rating = 6`
   - When `createReview` called
   - Then `BadRequestException` thrown

4. **`createReview_emptyContent_throwsBadRequestException()`**
   - Given `content = ""` or `content = "   "`
   - When `createReview` called
   - Then `BadRequestException` thrown with message "Review content cannot be empty."

5. **`createReview_duplicateReview_throwsDuplicatedException()`**
   - Given `(customerId, productId)` already exists in database
   - When `createReview` called
   - Then `DataIntegrityViolationException` caught, `DuplicatedException` thrown with message "You have already submitted a review for this product."

6. **`calculateAggregates_multipleReviews_returnsCorrectAverage()`**
   - Given 3 reviews with ratings 3, 4, 5
   - When `calculateAggregates(productId)` called
   - Then returns `{averageRating: 4.0, reviewCount: 3}`

7. **`calculateAggregates_roundingToOneDecimal_roundsCorrectly()`**
   - Given 2 reviews with ratings 4, 5
   - When `calculateAggregates(productId)` called
   - Then returns `{averageRating: 4.5, reviewCount: 2}`
   - Additional cases: 3 reviews (4,4,5) → 4.3, (4,5,5) → 4.7

8. **`calculateAggregates_noReviews_returnsZero()`**
   - Given no reviews for productId
   - When `calculateAggregates(productId)` called
   - Then returns `{averageRating: 0.0, reviewCount: 0}`

**File:** `product/src/test/java/com/yas/product/service/ProductServiceTest.java`

9. **`updateRatingAggregates_validInput_updatesProduct()`**
   - Given `productId=1`, `averageRating=4.3`, `reviewCount=42`
   - When `updateRatingAggregates` called
   - Then `Product` updated with correct values, saved, map returned

---

### Integration Tests (5)

**File:** `rating/src/it/java/com/yas/rating/controller/StorefrontReviewControllerIT.java`

10. **`postReview_validInput_returns201()`**
    - POST valid review
    - Verify `201 Created`, correct `ReviewVm` body, review saved in database, aggregate update called

11. **`postReview_invalidRating_returns400()`**
    - POST with `rating = 0`
    - Verify `400 Bad Request` with error message

12. **`postReview_duplicateReview_returns409()`**
    - POST same review twice (same `customerId`, `productId`)
    - Verify second request returns `409 Conflict`, only 1 review in database

13. **`postReview_aggregateUpdateSuccess_productUpdated()`**
    - POST 2 reviews for same product (ratings 5, 3)
    - Verify product's `averageRating=4.0`, `reviewCount=2`

14. **`getReviews_returnsReviewsOrderedByCreatedOn()`**
    - POST 3 reviews at different times
    - GET reviews
    - Verify reviews returned in `createdOn DESC` order, pagination correct

---

### Migration Tests (2)

**File:** `rating/src/it/java/com/yas/rating/migration/LiquibaseMigrationIT.java`

15. **`ratingUniqueConstraint_appliedCorrectly()`**
    - Verify unique constraint `(customerId, productId)` exists
    - Attempt duplicate insert fails with `DataIntegrityViolationException`

16. **`productAggregateColumns_exist()`**
    - Verify `product` table has `average_rating` DECIMAL(3,1) and `review_count` BIGINT columns

---

**Total:** 15 tests (8 unit, 5 integration, 2 migration verification)

---

## Implementation Code

### New Files

#### 1. StorefrontReviewController.java

```java
package com.yas.rating.controller;

import com.yas.commonlibrary.utils.AuthenticationUtils;
import com.yas.rating.service.RatingService;
import com.yas.rating.viewmodel.ReviewListVm;
import com.yas.rating.viewmodel.ReviewPostVm;
import com.yas.rating.viewmodel.ReviewVm;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/rating/storefront/reviews")
@RequiredArgsConstructor
public class StorefrontReviewController {
    private final RatingService ratingService;

    @PostMapping
    public ResponseEntity<ReviewVm> createReview(@Valid @RequestBody ReviewPostVm request) {
        String customerId = AuthenticationUtils.extractUserId();
        ReviewVm reviewVm = ratingService.createReview(request, customerId);
        return ResponseEntity.status(HttpStatus.CREATED).body(reviewVm);
    }

    @GetMapping
    public ResponseEntity<ReviewListVm> getReviews(
            @RequestParam Long productId,
            @RequestParam(defaultValue = "0") int pageNo,
            @RequestParam(defaultValue = "10") int pageSize) {
        ReviewListVm reviewListVm = ratingService.getReviewsByProductId(productId, pageNo, pageSize);
        return ResponseEntity.ok(reviewListVm);
    }
}
```

---

#### 2. ReviewPostVm.java

```java
package com.yas.rating.viewmodel;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public record ReviewPostVm(
        @NotNull(message = "Product ID is required")
        Long productId,

        @NotNull(message = "Rating is required")
        @Min(value = 1, message = "Rating must be between 1 and 5")
        @Max(value = 5, message = "Rating must be between 1 and 5")
        Integer rating,

        @NotBlank(message = "Review content cannot be empty")
        String content
) {}
```

---

#### 3. ReviewVm.java

```java
package com.yas.rating.viewmodel;

import java.time.ZonedDateTime;

public record ReviewVm(
        Long id,
        Long productId,
        String customerId,
        Integer rating,
        String content,
        ZonedDateTime createdOn
) {}
```

---

#### 4. ReviewListVm.java

```java
package com.yas.rating.viewmodel;

import java.util.List;

public record ReviewListVm(
        List<ReviewVm> reviews,
        int totalPages,
        long totalElements,
        int pageNo,
        int pageSize
) {}
```

---

#### 5. changelog-rating-unique-constraint.sql

```sql
-- Add unique constraint to prevent duplicate reviews per customer per product
ALTER TABLE rating ADD CONSTRAINT uq_customer_product UNIQUE (customer_id, product_id);

-- Add indexes for performance (only if not already present)
CREATE INDEX IF NOT EXISTS idx_rating_product_id ON rating (product_id);
CREATE INDEX IF NOT EXISTS idx_rating_customer_product ON rating (customer_id, product_id);
```

**Integration:** Add to `rating/src/main/resources/db/changelog/db.changelog-master.yaml`:

```yaml
- changeSet:
    id: add-rating-unique-constraint
    author: system
    changes:
      - sqlFile:
          path: db/changelog/ddl/changelog-rating-unique-constraint.sql
          splitStatements: true
```

---

#### 6. ProductAggregateController.java

```java
package com.yas.product.controller;

import com.yas.product.service.ProductService;
import com.yas.product.viewmodel.RatingAggregatePostVm;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/product/backoffice/products")
@RequiredArgsConstructor
public class ProductAggregateController {
    private final ProductService productService;

    @PutMapping("/{productId}/rating-aggregates")
    public ResponseEntity<Map<String, Object>> updateRatingAggregates(
            @PathVariable Long productId,
            @Valid @RequestBody RatingAggregatePostVm request) {
        Map<String, Object> result = productService.updateRatingAggregates(
                productId,
                request.averageRating(),
                request.reviewCount()
        );
        return ResponseEntity.ok(result);
    }
}
```

---

#### 7. RatingAggregatePostVm.java

```java
package com.yas.product.viewmodel;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record RatingAggregatePostVm(
        @NotNull(message = "Average rating is required")
        @DecimalMin(value = "0.0", message = "Average rating must be at least 0.0")
        @DecimalMax(value = "5.0", message = "Average rating must be at most 5.0")
        BigDecimal averageRating,

        @NotNull(message = "Review count is required")
        @Min(value = 0, message = "Review count must be non-negative")
        Long reviewCount
) {}
```

---

#### 8. changelog-product-rating-aggregates.sql

```sql
-- Add average_rating and review_count columns to product table (only if missing)
-- PostgreSQL syntax
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM information_schema.columns 
                   WHERE table_name = 'product' AND column_name = 'average_rating') THEN
        ALTER TABLE product ADD COLUMN average_rating DECIMAL(3,1);
    END IF;

    IF NOT EXISTS (SELECT 1 FROM information_schema.columns 
                   WHERE table_name = 'product' AND column_name = 'review_count') THEN
        ALTER TABLE product ADD COLUMN review_count BIGINT;
    END IF;
END $$;
```

**Integration:** Add to `product/src/main/resources/db/changelog/db.changelog-master.yaml`:

```yaml
- changeSet:
    id: add-product-rating-aggregates
    author: system
    changes:
      - sqlFile:
          path: db/changelog/ddl/changelog-product-rating-aggregates.sql
          splitStatements: false
```

---

### Modified Files

#### RatingService.java (additions)

```java
// Add these imports
import com.yas.commonlibrary.exception.BadRequestException;
import com.yas.commonlibrary.exception.DuplicatedException;
import com.yas.commonlibrary.exception.InternalServerErrorException;
import com.yas.rating.viewmodel.ReviewListVm;
import com.yas.rating.viewmodel.ReviewPostVm;
import com.yas.rating.viewmodel.ReviewVm;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.ZonedDateTime;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

// Add these methods to RatingService class:

public ReviewVm createReview(ReviewPostVm request, String customerId) {
    // Validate rating range
    if (request.rating() < 1 || request.rating() > 5) {
        throw new BadRequestException("Rating must be between 1 and 5.");
    }

    // Validate content
    String trimmedContent = request.content().trim();
    if (trimmedContent.isEmpty()) {
        throw new BadRequestException("Review content cannot be empty.");
    }

    // Create Rating entity
    Rating rating = new Rating();
    rating.setProductId(request.productId());
    rating.setCustomerId(customerId);
    rating.setStar(request.rating());
    rating.setContent(trimmedContent);
    rating.setCreatedOn(ZonedDateTime.now());
    rating.setLastModifiedOn(ZonedDateTime.now());
    rating.setCreatedBy(customerId);
    rating.setLastModifiedBy(customerId);

    // Save rating (catch unique constraint violation)
    try {
        rating = ratingRepository.save(rating);
    } catch (DataIntegrityViolationException ex) {
        throw new DuplicatedException("You have already submitted a review for this product.");
    }

    // Calculate aggregates
    Map<String, Object> aggregates = calculateAggregates(request.productId());
    BigDecimal averageRating = (BigDecimal) aggregates.get("averageRating");
    Long reviewCount = (Long) aggregates.get("reviewCount");

    // Update product aggregates via REST client
    try {
        productService.updateRatingAggregates(request.productId(), averageRating, reviewCount);
    } catch (Exception ex) {
        log.error("Failed to update product aggregates for productId: {}", request.productId(), ex);
        throw new InternalServerErrorException("Failed to update product aggregates.");
    }

    // Return ReviewVm
    return new ReviewVm(
            rating.getId(),
            rating.getProductId(),
            rating.getCustomerId(),
            rating.getStar(),
            rating.getContent(),
            rating.getCreatedOn()
    );
}

public ReviewListVm getReviewsByProductId(Long productId, int pageNo, int pageSize) {
    Page<Rating> page = ratingRepository.findByProductIdOrderByCreatedOnDesc(
            productId,
            PageRequest.of(pageNo, pageSize)
    );

    List<ReviewVm> reviews = page.getContent().stream()
            .map(rating -> new ReviewVm(
                    rating.getId(),
                    rating.getProductId(),
                    rating.getCustomerId(),
                    rating.getStar(),
                    rating.getContent(),
                    rating.getCreatedOn()
            ))
            .toList();

    return new ReviewListVm(
            reviews,
            page.getTotalPages(),
            page.getTotalElements(),
            page.getNumber(),
            page.getSize()
    );
}

public Map<String, Object> calculateAggregates(Long productId) {
    List<Rating> reviews = ratingRepository.findByProductId(productId);

    Map<String, Object> result = new HashMap<>();

    if (reviews.isEmpty()) {
        result.put("averageRating", BigDecimal.ZERO);
        result.put("reviewCount", 0L);
        return result;
    }

    int sum = reviews.stream().mapToInt(Rating::getStar).sum();
    int count = reviews.size();
    double average = (double) sum / count;
    BigDecimal averageRating = BigDecimal.valueOf(average)
            .setScale(1, RoundingMode.HALF_UP);

    result.put("averageRating", averageRating);
    result.put("reviewCount", (long) count);
    return result;
}
```

---

#### RatingRepository.java (additions)

```java
// Add these query methods:
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import java.util.List;

public interface RatingRepository extends JpaRepository<Rating, Long> {
    Page<Rating> findByProductIdOrderByCreatedOnDesc(Long productId, Pageable pageable);
    List<Rating> findByProductId(Long productId);
}
```

---

#### ProductService.java (additions)

```java
// Add these imports
import com.yas.commonlibrary.exception.NotFoundException;
import java.math.BigDecimal;
import java.time.ZonedDateTime;
import java.util.HashMap;
import java.util.Map;

// Add this method to ProductService class:

public Map<String, Object> updateRatingAggregates(Long productId, BigDecimal averageRating, Long reviewCount) {
    Product product = productRepository.findById(productId)
            .orElseThrow(() -> new NotFoundException("Product not found with ID: " + productId));

    product.setAverageRating(averageRating);
    product.setReviewCount(reviewCount);
    product.setLastModifiedOn(ZonedDateTime.now());
    
    // Set lastModifiedBy from security context if available, else "system"
    try {
        String currentUser = AuthenticationUtils.extractUserId();
        product.setLastModifiedBy(currentUser);
    } catch (Exception ex) {
        product.setLastModifiedBy("system");
    }

    productRepository.save(product);

    Map<String, Object> result = new HashMap<>();
    result.put("productId", product.getId());
    result.put("averageRating", product.getAverageRating());
    result.put("reviewCount", product.getReviewCount());
    return result;
}
```

---

#### Product.java (additions - only if fields missing)

```java
// Add these fields to Product entity if they don't exist:

@Column(name = "average_rating", precision = 3, scale = 1)
private BigDecimal averageRating;

@Column(name = "review_count")
private Long reviewCount;

// Add getters and setters
public BigDecimal getAverageRating() {
    return averageRating;
}

public void setAverageRating(BigDecimal averageRating) {
    this.averageRating = averageRating;
}

public Long getReviewCount() {
    return reviewCount;
}

public void setReviewCount(Long reviewCount) {
    this.reviewCount = reviewCount;
}
```

---

#### Rating.java (additions - only if audit fields missing)

```java
// Add these fields to Rating entity if they don't exist:

@Column(name = "created_by")
private String createdBy;

@Column(name = "last_modified_by")
private String lastModifiedBy;

// Add getters and setters
public String getCreatedBy() {
    return createdBy;
}

public void setCreatedBy(String createdBy) {
    this.createdBy = createdBy;
}

public String getLastModifiedBy() {
    return lastModifiedBy;
}

public void setLastModifiedBy(String lastModifiedBy) {
    this.lastModifiedBy = lastModifiedBy;
}
```

---

**End of Story 01-app LLD**
