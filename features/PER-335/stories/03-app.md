# Story 03-app: Backoffice Review Management Endpoints

**Repo:** `app`

---

## Context

The existing backoffice UI (`backoffice/pages/reviews/index.tsx`) expects backend endpoints for listing and deleting reviews. This story implements the missing backend endpoints (`GET /api/rating/backoffice/reviews` for paginated review list with search filters, `DELETE /api/rating/backoffice/reviews/{id}` for admin deletion) to support the existing UI. Without these endpoints, the backoffice review management page cannot function.

---

## Acceptance Criteria

1. The backoffice review list page (`backoffice/pages/reviews/index.tsx`) must fetch paginated reviews from `GET /api/rating/backoffice/reviews` with query parameters `pageNo`, `pageSize`, `createdFrom`, `createdTo`. (No explicit ticket AC, but required for existing UI)
2. The backoffice review list page must display customer name (`firstName`, `lastName`), product name, rating, content, and created date for each review. (No explicit ticket AC, but required for existing UI)
3. Admins must be able to delete a review via the "Delete" button, which calls `DELETE /api/rating/backoffice/reviews/{id}`. (No explicit ticket AC, but required for existing UI)
4. After a review is deleted, the product's average rating and review count must be recalculated. (Implied by AC 7, AC 8)
5. Deleted reviews must not appear in the review list after deletion. (Standard delete behavior)

---

## File Plan

### New Files

1. **`rating/src/main/java/com/yas/rating/controller/BackofficeReviewController.java`** — REST controller for backoffice review endpoints
   - Exposes `GET /api/rating/backoffice/reviews` for paginated review list with search filters
   - Exposes `DELETE /api/rating/backoffice/reviews/{id}` for admin deletion

2. **`rating/src/main/java/com/yas/rating/viewmodel/ReviewListGetVm.java`** — DTO for paginated review list response
   - Structure matches what `backoffice/pages/reviews/index.tsx` expects
   - Fields: `ratingList` (list of `RatingVm`), `totalPages` (int), `totalElements` (long)

3. **`rating/src/main/java/com/yas/rating/viewmodel/RatingVm.java`** — DTO for individual review in list response
   - Fields: `id`, `content`, `productName`, `createdBy` (customer ID), `firstName`, `lastName`, `createdOn` (timestamp)

4. **`rating/src/main/java/com/yas/rating/viewmodel/ReviewSearchCriteria.java`** — DTO for search query parameters
   - Fields: `pageNo`, `pageSize`, `createdFrom`, `createdTo`

5. **`rating/src/test/java/com/yas/rating/controller/BackofficeReviewControllerTest.java`** — Unit tests for backoffice endpoints

### Modified Files

1. **`rating/src/main/java/com/yas/rating/service/ReviewService.java`** — Add methods:
   - `getReviews(ReviewSearchCriteria criteria) -> ReviewListGetVm`
   - `deleteReviewById(Long id) -> void`

2. **`rating/src/main/java/com/yas/rating/repository/ReviewRepository.java`** — Add custom query method:
   - `Page<Review> findByCreatedAtBetween(Instant createdFrom, Instant createdTo, Pageable pageable)`

3. **`rating/src/main/java/com/yas/rating/service/AggregateService.java`** — Add method:
   - `updateAggregatesAfterReviewDeleted(Long productId, Integer deletedRating) -> void`

4. **`rating/src/main/java/com/yas/rating/config/SecurityConfig.java`** — Add security rules:
   - Require `ROLE_ADMIN` for `/api/rating/backoffice/reviews/**`

5. **`backoffice/modules/rating/services/RatingService.ts`** — Verify existing methods:
   - `getRatings(queryString: string)` calls `GET /api/rating/backoffice/reviews?${queryString}`
   - `deleteRatingById(id: number)` calls `DELETE /api/rating/backoffice/reviews/${id}`

---

## Interfaces

### REST API Endpoints

**GET /api/rating/backoffice/reviews**

- **Purpose:** Fetch paginated list of reviews with search filters
- **Authentication:** Required (admin role)
- **Request:**
  - Query parameters:
    - `pageNo` (int, optional, default 0)
    - `pageSize` (int, optional, default 10)
    - `createdFrom` (ISO 8601 timestamp, optional)
    - `createdTo` (ISO 8601 timestamp, optional)
- **Response:**
  - **200 OK:** `ReviewListGetVm` (JSON)
    ```json
    {
      "ratingList": [
        {
          "id": 123,
          "content": "Great product!",
          "productName": "Dell Laptop",
          "createdBy": "uuid-of-customer",
          "firstName": "John",
          "lastName": "Doe",
          "createdOn": "2025-01-15T10:30:00Z"
        }
      ],
      "totalPages": 5,
      "totalElements": 42
    }
    ```
  - **401 Unauthorized:** Not authenticated
  - **403 Forbidden:** Not an admin
  - **500 Internal Server Error:** Server error

**DELETE /api/rating/backoffice/reviews/{id}**

- **Purpose:** Delete a review by ID and recalculate product aggregates
- **Authentication:** Required (admin role)
- **Request:**
  - Path parameter: `id` (Long)
- **Response:**
  - **204 No Content:** Review deleted successfully
  - **401 Unauthorized:** Not authenticated
  - **403 Forbidden:** Not an admin
  - **404 Not Found:** Review not found
  - **500 Internal Server Error:** Server error

---

### Service Layer Interfaces

**ReviewService**
- `getReviews(ReviewSearchCriteria criteria) -> ReviewListGetVm`
  - Queries `ReviewRepository.findByCreatedAtBetween(createdFrom, createdTo, pageable)` or `ReviewRepository.findAll(pageable)` if filters not provided
  - For each `Review`, fetches customer details from `customer` service (`GET /api/customer/backoffice/customers/profile/{customerId}`) to get `firstName` and `lastName`
  - For each `Review`, fetches product details from `product` service (`GET /api/product/storefront/products/{productId}`) to get `productName`
  - Maps each `Review` to `RatingVm` and returns `ReviewListGetVm` with pagination metadata

- `deleteReviewById(Long id) -> void`
  - Fetches `Review` by id via `ReviewRepository.findById(id)` — throws `NotFoundException` if not found
  - Extracts `productId` and `rating` from `Review`
  - Deletes via `ReviewRepository.deleteById(id)`
  - Calls `AggregateService.updateAggregatesAfterReviewDeleted(productId, rating)`

**AggregateService**
- `updateAggregatesAfterReviewDeleted(Long productId, Integer deletedRating) -> void`
  - Fetches `ProductAggregate` row via `ProductAggregateRepository.findById(productId)` — if not found, log error and return (data inconsistency)
  - Recalculates `averageRating`:
    ```
    new_avg = ((old_avg * old_count) - deleted_rating) / (old_count - 1)
    ```
  - Decrements `reviewCount` by 1
  - If `reviewCount` becomes 0, set `averageRating` to 0.0
  - Saves via `ProductAggregateRepository.save()`
  - **Transaction:** entire method runs in a database transaction to ensure atomicity

**ProductService** (inter-service call)
- `getProductName(Long productId) -> String`
  - Calls `GET /api/product/storefront/products/{productId}` via `RestClient`
  - Extracts product name from response
  - Throws `NotFoundException` if product not found
  - Wraps call with circuit breaker (Resilience4j, follow `cart/src/main/java/com/yas/cart/service/ProductService.java`)

**CustomerService** (inter-service call)
- `getCustomerDetails(String customerId) -> CustomerVm`
  - Calls `GET /api/customer/backoffice/customers/profile/{customerId}` via `RestClient`
  - Extracts `firstName` and `lastName` from response
  - Throws `NotFoundException` if customer not found
  - Wraps call with circuit breaker (Resilience4j)

---

### Repository Interfaces

**ReviewRepository** (extends `JpaRepository<Review, Long>`)
- `Page<Review> findByCreatedAtBetween(Instant createdFrom, Instant createdTo, Pageable pageable)`
  - Custom query to filter reviews by creation date range
  - If `createdFrom` and `createdTo` are both provided, filter by `created_at BETWEEN createdFrom AND createdTo`
  - If not provided, use `findAll(pageable)` instead

**ProductAggregateRepository** (extends `JpaRepository<ProductAggregate, Long>`)
- Standard CRUD methods from `JpaRepository`

---

### DTOs (ViewModels)

**ReviewListGetVm** (response DTO for paginated list)
```java
package com.yas.rating.viewmodel;

import java.util.List;

public record ReviewListGetVm(
    List<RatingVm> ratingList,
    int totalPages,
    long totalElements
) {}
```

**RatingVm** (individual review in list)
```java
package com.yas.rating.viewmodel;

import java.time.ZonedDateTime;

public record RatingVm(
    Long id,
    String content,
    String productName,
    String createdBy, // customer ID
    String firstName,
    String lastName,
    ZonedDateTime createdOn
) {}
```

**ReviewSearchCriteria** (request DTO for search parameters)
```java
package com.yas.rating.viewmodel;

import java.time.Instant;

public record ReviewSearchCriteria(
    Integer pageNo,
    Integer pageSize,
    Instant createdFrom,
    Instant createdTo
) {
    public ReviewSearchCriteria {
        // Default values
        if (pageNo == null) pageNo = 0;
        if (pageSize == null) pageSize = 10;
    }
}
```

**CustomerVm** (response from customer service, internal use)
```java
// Not stored in rating service; used for mapping only
public record CustomerVm(
    String id,
    String firstName,
    String lastName,
    String email
) {}
```

**ProductDetailVm** (response from product service, internal use)
```java
// Not stored in rating service; used for mapping only
public record ProductDetailVm(
    Long id,
    String name,
    String slug
) {}
```

---

## Algorithm Notes

### Review List Fetch and Mapping

1. **Parse Query Parameters:**
   - Extract `pageNo`, `pageSize`, `createdFrom`, `createdTo` from request query string
   - Create `ReviewSearchCriteria` DTO
   - Defaults: `pageNo = 0`, `pageSize = 10`

2. **Query Reviews:**
   - Create `Pageable` object: `PageRequest.of(pageNo, pageSize, Sort.by("createdAt").descending())`
   - If `createdFrom` and `createdTo` are provided:
     ```java
     Page<Review> page = reviewRepository.findByCreatedAtBetween(createdFrom, createdTo, pageable);
     ```
   - Otherwise:
     ```java
     Page<Review> page = reviewRepository.findAll(pageable);
     ```

3. **Fetch Customer and Product Details:**
   - For each `Review` in `page.getContent()`:
     - Fetch customer details:
       ```java
       CustomerVm customer = customerService.getCustomerDetails(review.getCustomerId());
       ```
     - Fetch product details:
       ```java
       ProductDetailVm product = productService.getProductDetails(review.getProductId());
       ```
     - Map to `RatingVm`:
       ```java
       new RatingVm(
           review.getId(),
           review.getContent(),
           product.name(),
           review.getCustomerId(),
           customer.firstName(),
           customer.lastName(),
           review.getCreatedAt()
       )
       ```

4. **Return Response:**
   - Wrap list of `RatingVm` in `ReviewListGetVm`:
     ```java
     new ReviewListGetVm(
         ratingVmList,
         page.getTotalPages(),
         page.getTotalElements()
     )
     ```

**Performance Consideration:**
- Fetching customer and product details for each review is expensive (N+1 problem)
- **Optimization 1:** Batch fetch customer and product details (collect all customer IDs and product IDs, make bulk API calls if supported)
- **Optimization 2:** Cache customer and product names in `review` table (denormalize) — updated on review creation and product/customer name changes
- **For initial implementation:** Use simple per-review fetch (follow existing service patterns, e.g. `cart/src/main/java/com/yas/cart/service/ProductService.java`)

---

### Review Deletion and Aggregate Update

1. **Fetch Review:**
   - Query `ReviewRepository.findById(id)`
   - If not found, throw `NotFoundException` (HTTP 404)

2. **Extract Data:**
   - `productId = review.getProductId();`
   - `deletedRating = review.getRating();`

3. **Delete Review:**
   - `reviewRepository.deleteById(id);`

4. **Update Aggregates:**
   - Call `AggregateService.updateAggregatesAfterReviewDeleted(productId, deletedRating)`
   - Inside `updateAggregatesAfterReviewDeleted`:
     ```java
     ProductAggregate aggregate = productAggregateRepository.findById(productId)
         .orElseThrow(() -> new NotFoundException("Aggregate not found for product " + productId));
     
     BigDecimal oldAvg = aggregate.getAverageRating();
     int oldCount = aggregate.getReviewCount();
     
     if (oldCount == 1) {
         // Last review deleted
         aggregate.setAverageRating(BigDecimal.ZERO);
         aggregate.setReviewCount(0);
     } else {
         // Recalculate average: ((old_avg * old_count) - deleted_rating) / (old_count - 1)
         BigDecimal newAvg = oldAvg.multiply(new BigDecimal(oldCount))
             .subtract(new BigDecimal(deletedRating))
             .divide(new BigDecimal(oldCount - 1), 2, RoundingMode.HALF_UP);
         
         aggregate.setAverageRating(newAvg);
         aggregate.setReviewCount(oldCount - 1);
     }
     
     aggregate.setUpdatedAt(Instant.now());
     productAggregateRepository.save(aggregate);
     ```

5. **Transaction:**
   - Entire method (`deleteReviewById`) runs in a database transaction (`@Transactional`) to ensure atomicity
   - If aggregate update fails, roll back review deletion

**Edge Case: Orphan Aggregates:**
- If `ProductAggregate` row is not found (data inconsistency), log error and throw `NotFoundException`
- In production, add monitoring/alerting for this case

---

### Backoffice UI Integration

**Current UI Behavior (from `backoffice/pages/reviews/index.tsx`):**

1. **On Page Load:**
   - Calls `getRatings(queryString)` with `pageNo=0`, `pageSize=10`, `createdFrom`, `createdTo` from search form
   - Expects response:
     ```typescript
     {
       ratingList: Rating[],
       totalPages: number
     }
     ```
   - Displays reviews in table with columns: ID, Content, Product Name, Customer ID, Customer Name, Date Post, Actions

2. **On Delete Button Click:**
   - Calls `deleteRatingById(ratingId)`
   - On success, shows toast "Delete rating successfully" and refetches review list
   - On error, logs error to console (no toast shown)

**Required Backend Behavior:**
- `GET /api/rating/backoffice/reviews` must return exactly the structure expected by UI
- `DELETE /api/rating/backoffice/reviews/{id}` must return 204 No Content on success, 404 if review not found

**Data Mapping:**
- UI expects `Rating` type with fields: `id`, `content`, `productName`, `createdBy`, `lastName`, `firstName`, `createdOn`
- Backend returns `RatingVm` with same fields (see DTO section)
- Timestamps: UI uses `moment` to format `createdOn` as "MMMM Do YYYY, h:mm:ss a" (e.g. "January 15th 2025, 10:30:00 am")
- Backend returns `createdOn` as ISO 8601 timestamp (e.g. "2025-01-15T10:30:00Z")

---

## Tests

### Unit Tests (BackofficeReviewControllerTest)

**Test File:** `rating/src/test/java/com/yas/rating/controller/BackofficeReviewControllerTest.java`

**Coverage:** 5 tests mapping to Verification Approach in breakdown.md

1. **testGetReviews_Success** — GET with valid pagination params returns 200 and `ReviewListGetVm`
2. **testGetReviews_WithFilters** — GET with `createdFrom` and `createdTo` filters returns filtered results
3. **testDeleteReview_Success** — DELETE existing review returns 204
4. **testDeleteReview_NotFound** — DELETE non-existent review returns 404
5. **testGetReviews_Forbidden** — GET without admin role returns 403

```java
package com.yas.rating.controller;

import com.yas.rating.service.ReviewService;
import com.yas.rating.viewmodel.ReviewListGetVm;
import com.yas.rating.viewmodel.RatingVm;
import com.yas.rating.viewmodel.ReviewSearchCriteria;
import com.yas.commonlibrary.exception.NotFoundException;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import java.time.Instant;
import java.time.ZonedDateTime;
import java.util.List;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(BackofficeReviewController.class)
class BackofficeReviewControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ReviewService reviewService;

    @Test
    @WithMockUser(roles = "ADMIN")
    void testGetReviews_Success() throws Exception {
        // Given
        RatingVm ratingVm = new RatingVm(
            1L,
            "Great product!",
            "Dell Laptop",
            "customer-uuid",
            "John",
            "Doe",
            ZonedDateTime.now()
        );
        ReviewListGetVm response = new ReviewListGetVm(List.of(ratingVm), 1, 1);

        when(reviewService.getReviews(any(ReviewSearchCriteria.class))).thenReturn(response);

        // When & Then
        mockMvc.perform(get("/api/rating/backoffice/reviews")
                .param("pageNo", "0")
                .param("pageSize", "10"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.ratingList").isArray())
            .andExpect(jsonPath("$.ratingList[0].id").value(1))
            .andExpect(jsonPath("$.ratingList[0].content").value("Great product!"))
            .andExpect(jsonPath("$.ratingList[0].productName").value("Dell Laptop"))
            .andExpect(jsonPath("$.totalPages").value(1))
            .andExpect(jsonPath("$.totalElements").value(1));

        verify(reviewService, times(1)).getReviews(any(ReviewSearchCriteria.class));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void testGetReviews_WithFilters() throws Exception {
        // Given
        String createdFrom = "2025-01-01T00:00:00Z";
        String createdTo = "2025-01-31T23:59:59Z";

        RatingVm ratingVm = new RatingVm(
            2L,
            "Good quality",
            "HP Laptop",
            "customer-uuid-2",
            "Jane",
            "Smith",
            ZonedDateTime.parse(createdFrom)
        );
        ReviewListGetVm response = new ReviewListGetVm(List.of(ratingVm), 1, 1);

        when(reviewService.getReviews(any(ReviewSearchCriteria.class))).thenReturn(response);

        // When & Then
        mockMvc.perform(get("/api/rating/backoffice/reviews")
                .param("pageNo", "0")
                .param("pageSize", "10")
                .param("createdFrom", createdFrom)
                .param("createdTo", createdTo))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.ratingList").isArray())
            .andExpect(jsonPath("$.ratingList[0].id").value(2))
            .andExpect(jsonPath("$.totalPages").value(1));

        verify(reviewService, times(1)).getReviews(argThat(criteria ->
            criteria.createdFrom().equals(Instant.parse(createdFrom)) &&
            criteria.createdTo().equals(Instant.parse(createdTo))
        ));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void testDeleteReview_Success() throws Exception {
        // Given
        Long reviewId = 123L;
        doNothing().when(reviewService).deleteReviewById(reviewId);

        // When & Then
        mockMvc.perform(delete("/api/rating/backoffice/reviews/{id}", reviewId))
            .andExpect(status().isNoContent());

        verify(reviewService, times(1)).deleteReviewById(reviewId);
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void testDeleteReview_NotFound() throws Exception {
        // Given
        Long reviewId = 999L;
        doThrow(new NotFoundException("Review not found")).when(reviewService).deleteReviewById(reviewId);

        // When & Then
        mockMvc.perform(delete("/api/rating/backoffice/reviews/{id}", reviewId))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.detail").value("Review not found"));

        verify(reviewService, times(1)).deleteReviewById(reviewId);
    }

    @Test
    @WithMockUser(roles = "USER") // Not an admin
    void testGetReviews_Forbidden() throws Exception {
        // When & Then
        mockMvc.perform(get("/api/rating/backoffice/reviews")
                .param("pageNo", "0")
                .param("pageSize", "10"))
            .andExpect(status().isForbidden());

        verify(reviewService, never()).getReviews(any(ReviewSearchCriteria.class));
    }
}
```

---

### Unit Tests (ReviewServiceTest)

**Test File:** `rating/src/test/java/com/yas/rating/service/ReviewServiceTest.java`

**Coverage:** Business logic for review deletion and aggregate updates

```java
package com.yas.rating.service;

import com.yas.rating.model.Review;
import com.yas.rating.model.ProductAggregate;
import com.yas.rating.repository.ReviewRepository;
import com.yas.rating.repository.ProductAggregateRepository;
import com.yas.commonlibrary.exception.NotFoundException;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.Instant;
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
    private AggregateService aggregateService;

    @InjectMocks
    private ReviewService reviewService;

    @Test
    void testDeleteReview_AggregatesUpdated() {
        // Given
        Long reviewId = 123L;
        Long productId = 1L;
        Integer deletedRating = 4;

        Review review = new Review();
        review.setId(reviewId);
        review.setProductId(productId);
        review.setRating(deletedRating);

        when(reviewRepository.findById(reviewId)).thenReturn(Optional.of(review));
        doNothing().when(reviewRepository).deleteById(reviewId);
        doNothing().when(aggregateService).updateAggregatesAfterReviewDeleted(productId, deletedRating);

        // When
        reviewService.deleteReviewById(reviewId);

        // Then
        verify(reviewRepository, times(1)).findById(reviewId);
        verify(reviewRepository, times(1)).deleteById(reviewId);
        verify(aggregateService, times(1)).updateAggregatesAfterReviewDeleted(productId, deletedRating);
    }

    @Test
    void testDeleteReview_NotFound() {
        // Given
        Long reviewId = 999L;
        when(reviewRepository.findById(reviewId)).thenReturn(Optional.empty());

        // When & Then
        assertThrows(NotFoundException.class, () -> reviewService.deleteReviewById(reviewId));

        verify(reviewRepository, times(1)).findById(reviewId);
        verify(reviewRepository, never()).deleteById(anyLong());
        verify(aggregateService, never()).updateAggregatesAfterReviewDeleted(anyLong(), anyInt());
    }
}
```

---

### Unit Tests (AggregateServiceTest)

**Test File:** `rating/src/test/java/com/yas/rating/service/AggregateServiceTest.java`

**Coverage:** Aggregate recalculation logic after review deletion

```java
package com.yas.rating.service;

import com.yas.rating.model.ProductAggregate;
import com.yas.rating.repository.ProductAggregateRepository;
import com.yas.commonlibrary.exception.NotFoundException;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AggregateServiceTest {

    @Mock
    private ProductAggregateRepository productAggregateRepository;

    @InjectMocks
    private AggregateService aggregateService;

    @Test
    void testUpdateAggregatesAfterReviewDeleted_MultipleReviewsRemaining() {
        // Given
        Long productId = 1L;
        Integer deletedRating = 3;

        ProductAggregate existingAggregate = new ProductAggregate();
        existingAggregate.setProductId(productId);
        existingAggregate.setAverageRating(new BigDecimal("4.00")); // (5 + 4 + 3) / 3 = 4.00
        existingAggregate.setReviewCount(3);

        when(productAggregateRepository.findById(productId)).thenReturn(Optional.of(existingAggregate));
        when(productAggregateRepository.save(any(ProductAggregate.class))).thenAnswer(invocation -> invocation.getArgument(0));

        // When
        aggregateService.updateAggregatesAfterReviewDeleted(productId, deletedRating);

        // Then
        // Expected new average: ((4.00 * 3) - 3) / 2 = (12 - 3) / 2 = 4.50
        verify(productAggregateRepository, times(1)).findById(productId);
        verify(productAggregateRepository, times(1)).save(argThat(aggregate ->
            aggregate.getProductId().equals(productId) &&
            aggregate.getAverageRating().compareTo(new BigDecimal("4.50")) == 0 &&
            aggregate.getReviewCount() == 2
        ));
    }

    @Test
    void testUpdateAggregatesAfterReviewDeleted_LastReviewDeleted() {
        // Given
        Long productId = 1L;
        Integer deletedRating = 5;

        ProductAggregate existingAggregate = new ProductAggregate();
        existingAggregate.setProductId(productId);
        existingAggregate.setAverageRating(new BigDecimal("5.00")); // Last review
        existingAggregate.setReviewCount(1);

        when(productAggregateRepository.findById(productId)).thenReturn(Optional.of(existingAggregate));
        when(productAggregateRepository.save(any(ProductAggregate.class))).thenAnswer(invocation -> invocation.getArgument(0));

        // When
        aggregateService.updateAggregatesAfterReviewDeleted(productId, deletedRating);

        // Then
        // Expected: averageRating = 0.0, reviewCount = 0
        verify(productAggregateRepository, times(1)).findById(productId);
        verify(productAggregateRepository, times(1)).save(argThat(aggregate ->
            aggregate.getProductId().equals(productId) &&
            aggregate.getAverageRating().compareTo(BigDecimal.ZERO) == 0 &&
            aggregate.getReviewCount() == 0
        ));
    }

    @Test
    void testUpdateAggregatesAfterReviewDeleted_AggregateNotFound() {
        // Given
        Long productId = 999L;
        Integer deletedRating = 4;

        when(productAggregateRepository.findById(productId)).thenReturn(Optional.empty());

        // When & Then
        assertThrows(NotFoundException.class, () ->
            aggregateService.updateAggregatesAfterReviewDeleted(productId, deletedRating)
        );

        verify(productAggregateRepository, times(1)).findById(productId);
        verify(productAggregateRepository, never()).save(any(ProductAggregate.class));
    }
}
```

---

## Dependencies

- **Story 1 (Backend Review Submission API and Database Schema)** — Backend `Review` and `ProductAggregate` entities must exist; `ReviewRepository` and `AggregateService` must be implemented.

---

## Out of Scope

- Customer-facing review submission and display (covered in Story 1 and Story 2)
- Review moderation workflow (approval/rejection by admins; not in ticket scope)
- Review editing (not in ticket scope)

---

## Notes

- The backoffice UI (`backoffice/pages/reviews/index.tsx`) already exists and expects a reviews API. This story implements the backend endpoints to support it.
- The API follows the YAS convention: `/api/<service>/backoffice/<resource>` for admin endpoints.
- Aggregate calculation uses denormalized `product_aggregate` table for fast reads (pre-computed average rating and count), updated transactionally after each review insert or delete.
- The `AuthenticationUtils.extractUserId()` method (in `common-library`) assumes the JWT token contains a user ID claim. Admin role is verified via Spring Security (`ROLE_ADMIN` in JWT).
- Inter-service calls to `customer` and `product` services follow the existing pattern (observed in `cart/src/main/java/com/yas/cart/service/ProductService.java`, `customer/src/main/java/com/yas/customer/service/ProductService.java`) with circuit breaker (Resilience4j).
- The backoffice UI does not display the rating (1–5 stars) in the table, only the review content, product name, customer name, and date. If the rating should be displayed, add a `rating` field to `RatingVm` and update the UI to display it.
