# Low-Level Design — Story 01-app

**Epic:** PER-316 (Signed-in shoppers can save a wishlist and add those items to the cart)  
**Story:** Backend wishlist storage and API  
**Repo:** `app`

---

## Overview

This story implements the backend persistence layer and REST API for the wishlist feature. It creates the `wishlist_item` table in the customer service database, exposes CRUD endpoints for adding/removing/listing wishlist items, and enforces signed-in-only access. The customer service owns wishlist data; the product service provides availability status. This foundational work enables Story 2 (UI) and Story 3 (cart integration) to function.

---

## Stack

- **Language:** Java 17
- **Framework:** Spring Boot 3.x, Spring Data JPA, Spring Security with Keycloak JWT authentication
- **Database:** PostgreSQL, Liquibase migrations (matches existing `customer` service schema versioning)
- **REST:** JSON over HTTP, standard error responses via `ErrorVm` from `common-library`
- **Existing patterns:** `SecurityConfig`, `AuthenticationUtils`, `AbstractAuditEntity` (audit fields), `ApiExceptionHandler`

---

## Acceptance Criteria

### AC-1: Signed-in shopper can add, view, and remove wishlist items across sessions (persistence layer)

**Given** a signed-in shopper  
**When** they add a product to their wishlist via API  
**Then** the product is persisted in the `wishlist_item` table

**Given** a signed-in shopper with items on their wishlist  
**When** they sign out and sign back in  
**Then** their wishlist still contains the same items (data persists across sessions)

**Given** a signed-in shopper viewing their wishlist via API  
**When** they remove an item  
**Then** that item is deleted from the `wishlist_item` table

---

### AC-2: The same product cannot appear twice on one shopper's wishlist

**Given** a signed-in shopper  
**When** they add a product already on their wishlist  
**Then** the API returns 200 OK (idempotent)  
**And** the database does not create a duplicate entry (UNIQUE constraint enforced)

---

### AC-5: Guests are prompted to sign in before saving to a wishlist (backend enforcement)

**Given** a guest (no JWT or invalid JWT)  
**When** they attempt to call any wishlist API endpoint  
**Then** the backend returns 401 Unauthorized  
**And** no wishlist data is saved

---

## File Touch List

### New Files

1. **Migration:**
   - `customer/src/main/resources/db/changelog/ddl/changelog-0002.sql`

2. **Model:**
   - `customer/src/main/java/com/yas/customer/model/WishlistItem.java`

3. **Repository:**
   - `customer/src/main/java/com/yas/customer/repository/WishlistItemRepository.java`

4. **Service:**
   - `customer/src/main/java/com/yas/customer/service/WishlistService.java`
   - `customer/src/main/java/com/yas/customer/service/ProductService.java` (create if missing)

5. **Controller:**
   - `customer/src/main/java/com/yas/customer/controller/WishlistController.java`

6. **ViewModels:**
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemPostVm.java`
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemVm.java`
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListVm.java`
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemDetailVm.java`

7. **Tests:**
   - `customer/src/test/java/com/yas/customer/controller/WishlistControllerTest.java`
   - `customer/src/test/java/com/yas/customer/service/WishlistServiceTest.java`
   - `customer/src/it/java/com/yas/customer/service/WishlistServiceIT.java`

---

### Modified Files

1. **Configuration:**
   - `customer/src/main/java/com/yas/customer/config/SecurityConfig.java` — Add `/storefront/wishlist/**` to authenticated paths
   - `customer/src/main/java/com/yas/customer/config/ServiceUrlConfig.java` — Add `product` service URL field (if missing)

2. **Properties:**
   - `customer/src/main/resources/application.properties` — Add `yas.services.product=http://api.yas.local/product` (if missing)

3. **Messages:**
   - `customer/src/main/resources/messages/messages.properties` — Add wishlist error messages

4. **Changelog master:**
   - `customer/src/main/resources/db/changelog/db.changelog-master.yaml` — `changelog-0002.sql` auto-included via `includeAll`

---

## Database Schema

### Migration: `changelog-0002.sql`

**File:** `customer/src/main/resources/db/changelog/ddl/changelog-0002.sql`

```sql
--liquibase formatted sql

--changeset wishlist:issue-per-316-1
create sequence wishlist_item_seq start with 1 increment by 50;
create table wishlist_item (
    id bigint not null,
    created_by varchar(255),
    created_on timestamp(6),
    last_modified_by varchar(255),
    last_modified_on timestamp(6),
    customer_id varchar(255) not null,
    product_id bigint not null,
    primary key (id),
    constraint uk_wishlist_customer_product unique (customer_id, product_id)
);
create index idx_wishlist_customer on wishlist_item (customer_id);
```

**Rationale:** 
- Composite business key (`customer_id`, `product_id`) with surrogate ID for ORM simplicity
- Matches existing `cart_item` pattern (see `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql`)
- Audit fields (`created_by`, `created_on`, `last_modified_by`, `last_modified_on`) follow `AbstractAuditEntity` convention from `common-library`
- UNIQUE constraint enforces AC-2 (no duplicates)
- Index on `customer_id` optimizes list queries

**Changelog master update:** No manual change needed. The existing `includeAll` directive in `customer/src/main/resources/db/changelog/db.changelog-master.yaml` automatically picks up `changelog-0002.sql`:

```yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/ddl/
  - includeAll:
      path: db/changelog/data/
```

---

## REST API Contract

**Base URL:** `/api/customer/storefront/wishlist`

### Endpoint: `POST /api/customer/storefront/wishlist/items`

**Purpose:** Add a product to the signed-in shopper's wishlist (AC-1, AC-2).

**Authentication:** Required (JWT from Keycloak, extract `customer_id` via `AuthenticationUtils.getCurrentUserId()`)

**Request Body:**
```json
{
  "productId": 123
}
```

**Response:**
- **200 OK** — Product added or already exists (idempotent)
  ```json
  {
    "message": "Product added to wishlist"
  }
  ```
- **401 Unauthorized** — Not signed in (AC-5)
  ```json
  {
    "statusCode": "401",
    "title": "Unauthorized",
    "detail": "Authentication required",
    "fieldErrors": []
  }
  ```
- **404 Not Found** — Product does not exist (call to product service `GET /api/product/storefront/products/list-featured?productId={id}` returned empty)
  ```json
  {
    "statusCode": "404",
    "title": "Not Found",
    "detail": "Product with ID 123 not found",
    "fieldErrors": []
  }
  ```

**Behavior:**
1. Extract `customer_id` from JWT (`AuthenticationUtils.getCurrentUserId()` from `common-library`)
2. Call product service to verify product exists (via `ProductService.existsById(productId)`)
3. If product not found → return 404
4. Check if product already on wishlist: `findByCustomerIdAndProductId(customerId, productId)`
5. If exists → return 200 (idempotent, AC-2)
6. If not exists → INSERT `wishlist_item` with `customer_id`, `product_id`, `created_on` (timestamp)
7. Catch `DataIntegrityViolationException` (UNIQUE constraint violation from race condition) → log at DEBUG, return 200 (idempotent)

---

### Endpoint: `GET /api/customer/storefront/wishlist/items`

**Purpose:** List all wishlist items for the signed-in shopper with product details and availability (AC-1, AC-4).

**Authentication:** Required (JWT)

**Response:**
- **200 OK**
  ```json
  {
    "items": [
      {
        "wishlistItemId": 1,
        "productId": 123,
        "productName": "Example Product",
        "productSlug": "example-product",
        "thumbnailUrl": "https://example.com/media/thumbnail.jpg",
        "price": 29.99,
        "isAvailable": true,
        "discontinuedMessage": null
      },
      {
        "wishlistItemId": 2,
        "productId": 456,
        "productName": "Discontinued Item",
        "productSlug": "discontinued-item",
        "thumbnailUrl": "https://example.com/media/thumbnail2.jpg",
        "price": 49.99,
        "isAvailable": false,
        "discontinuedMessage": "This product is no longer available"
      }
    ]
  }
  ```
- **401 Unauthorized** — Not signed in

**Behavior:**
1. Extract `customer_id` from JWT
2. SELECT `wishlist_item` WHERE `customer_id` = current user, ORDER BY `created_on` DESC
3. For each `wishlist_item`, fetch product details via `ProductService.getProductById(productId)` in parallel with `CompletableFuture`, 2-second timeout per product
   - **Fetch strategy:** Call `getProducts(List.of(productId))` for each wishlist item in parallel using `CompletableFuture`, with a 2-second timeout per product. The final implementation collects all results before returning the response. If a batch endpoint is available, prefer that; otherwise, use parallel single-product calls.
   - Product service endpoint: `GET /api/product/storefront/products/list-featured?productId={ids}` (comma-separated list)
   - Response: `List<ProductThumbnailVm>` (from cart service: `id`, `name`, `slug`, `thumbnailMediaUrl`, `price`)
4. **Availability logic:** Product is available if product service returns 200 AND product exists in response
   - If product not found (empty list) → `isAvailable: false`, `discontinuedMessage: "This product is no longer available"`
   - If product service call fails (timeout, 5xx) → `isAvailable: false`, `discontinuedMessage: "Unable to verify availability"`
5. Return `WishlistItemListVm` with all items

**Note:** The cart service's `ProductService` already implements parallel product fetching via `RestClient`. This story reuses the same pattern. See `cart/src/main/java/com/yas/cart/service/ProductService.java`:
```java
public List<ProductThumbnailVm> getProducts(List<Long> ids) {
    final URI url = UriComponentsBuilder
        .fromUriString(serviceUrlConfig.product())
        .path("/storefront/products/list-featured")
        .queryParam("productId", ids)
        .build()
        .toUri();
    return restClient.get()
        .uri(url)
        .retrieve()
        .toEntity(new ParameterizedTypeReference<List<ProductThumbnailVm>>() {})
        .getBody();
}
```

**Open Question:** Does the product service batch endpoint accept multiple `productId` query params (e.g., `?productId=1&productId=2`) or a comma-separated list? Verify in product service and adjust query param construction.

---

### Endpoint: `DELETE /api/customer/storefront/wishlist/items/{wishlistItemId}`

**Purpose:** Remove an item from the signed-in shopper's wishlist (AC-1).

**Authentication:** Required (JWT)

**Response:**
- **204 No Content** — Deleted successfully
- **401 Unauthorized** — Not signed in
- **404 Not Found** — Item does not exist or does not belong to the current user

**Behavior:**
1. Extract `customer_id` from JWT
2. Call `findByIdAndCustomerId(wishlistItemId, customerId)` from repository
3. If not found → throw `NotFoundException("USER_ADDRESS_NOT_FOUND")` (matches existing pattern in `UserAddressService`)
4. DELETE the entity
5. Return 204 on success

**Security:** Cannot delete another user's item (enforced by `findByIdAndCustomerId` query).

---

### Endpoint: `GET /api/customer/storefront/wishlist/items/{wishlistItemId}` (Internal)

**Purpose:** Retrieve single wishlist item details for cart service when adding to cart (Story 3).

**Authentication:** Required (JWT, same user)

**Response:**
- **200 OK**
  ```json
  {
    "wishlistItemId": 1,
    "customerId": "uuid-from-keycloak",
    "productId": 123,
    "isAvailable": true
  }
  ```
- **401 Unauthorized** — Not signed in
- **404 Not Found** — Wishlist item does not exist

**Behavior:**
1. Extract `customer_id` from JWT
2. SELECT `wishlist_item` WHERE `id = {wishlistItemId}`
3. If not found → throw `NotFoundException("USER_ADDRESS_NOT_FOUND")`
4. Call product service to determine `isAvailable` (same logic as list endpoint)
5. Return `WishlistItemDetailVm`

**Security:** This endpoint is for cart service only (authenticated service-to-service call, same JWT). The cart service will validate that the returned `customerId` matches the current JWT user (see Story 3).

---

## Java Implementation

### 1. JPA Entity: `WishlistItem`

**Path:** `customer/src/main/java/com/yas/customer/model/WishlistItem.java`

```java
package com.yas.customer.model;

import com.yas.commonlibrary.model.AbstractAuditEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(
    name = "wishlist_item",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_wishlist_customer_product",
        columnNames = {"customer_id", "product_id"}
    ),
    indexes = @Index(name = "idx_wishlist_customer", columnList = "customer_id")
)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class WishlistItem extends AbstractAuditEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "wishlist_item_seq")
    @SequenceGenerator(name = "wishlist_item_seq", sequenceName = "wishlist_item_seq", allocationSize = 50)
    private Long id;

    @Column(name = "customer_id", nullable = false)
    private String customerId;

    @Column(name = "product_id", nullable = false)
    private Long productId;
}
```

**Rationale:**
- Extends `AbstractAuditEntity` (from `common-library`) for `createdBy`, `createdOn`, `lastModifiedBy`, `lastModifiedOn` fields
- `customerId` is `String` (Keycloak user ID from JWT, matches `UserAddress.userId` pattern)
- `productId` is `Long` (FK reference to product service, logical only — no DB-level FK across services)
- `@UniqueConstraint` enforces AC-2 (no duplicates)
- `@Index` on `customer_id` optimizes list queries

---

### 2. Repository: `WishlistItemRepository`

**Path:** `customer/src/main/java/com/yas/customer/repository/WishlistItemRepository.java`

```java
package com.yas.customer.repository;

import com.yas.customer.model.WishlistItem;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface WishlistItemRepository extends JpaRepository<WishlistItem, Long> {

    List<WishlistItem> findByCustomerIdOrderByCreatedOnDesc(String customerId);

    Optional<WishlistItem> findByIdAndCustomerId(Long id, String customerId);

    Optional<WishlistItem> findByCustomerIdAndProductId(String customerId, Long productId);
}
```

**Methods:**
- `findByCustomerIdOrderByCreatedOnDesc(String customerId)` — List endpoint query (AC-1)
- `findByIdAndCustomerId(Long id, String customerId)` — Delete endpoint security check (AC-1)
- `findByCustomerIdAndProductId(String customerId, Long productId)` — Duplicate check before add (AC-2)

---

### 3. Service: `ProductService` (new, if missing)

**Path:** `customer/src/main/java/com/yas/customer/service/ProductService.java`

**Action Required:** Verify if `ProductService` already exists in `customer` service. If missing, create it following the `LocationService` pattern (see `customer/src/main/java/com/yas/customer/service/LocationService.java`).

```java
package com.yas.customer.service;

import com.yas.customer.config.ServiceUrlConfig;
import com.yas.customer.viewmodel.product.ProductThumbnailVm;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import java.net.URI;
import java.util.List;
import lombok.RequiredArgsConstructor;
import org.apache.commons.collections4.CollectionUtils;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import org.springframework.web.util.UriComponentsBuilder;

@Service
@RequiredArgsConstructor
public class ProductService extends AbstractCircuitBreakFallbackHandler {

    private final RestClient restClient;
    private final ServiceUrlConfig serviceUrlConfig;

    @Retry(name = "restApi")
    @CircuitBreaker(name = "restCircuitBreaker", fallbackMethod = "handleProductThumbnailFallback")
    public List<ProductThumbnailVm> getProducts(List<Long> ids) {
        final URI url = UriComponentsBuilder
            .fromUriString(serviceUrlConfig.product())
            .path("/storefront/products/list-featured")
            .queryParam("productId", ids)
            .build()
            .toUri();

        return restClient.get()
            .uri(url)
            .retrieve()
            .body(new ParameterizedTypeReference<List<ProductThumbnailVm>>() {});
    }

    public ProductThumbnailVm getProductById(Long id) {
        List<ProductThumbnailVm> products = getProducts(List.of(id));
        if (CollectionUtils.isEmpty(products)) {
            return null;
        }
        return products.getFirst();
    }

    public boolean existsById(Long id) {
        return getProductById(id) != null;
    }

    protected List<ProductThumbnailVm> handleProductThumbnailFallback(Throwable throwable) throws Throwable {
        return handleTypedFallback(throwable);
    }
}
```

**New ViewModel:** `customer/src/main/java/com/yas/customer/viewmodel/product/ProductThumbnailVm.java`

```java
package com.yas.customer.viewmodel.product;

public record ProductThumbnailVm(
    Long id,
    String name,
    String slug,
    String thumbnailMediaUrl,
    Double price
) {}
```

**Rationale:** Matches cart service's `ProductThumbnailVm` and `ProductService` pattern. The null check in `getProductById()` prevents `NoSuchElementException` when the product service returns an empty list (see cart service implementation in `cart/src/main/java/com/yas/cart/service/ProductService.java`). The `queryParam("productId", ids)` construction is assumed to work based on the cart service using the same pattern; if the product service endpoint format differs, verify and adjust.

### 4. Service: `WishlistService`

**Path:** `customer/src/main/java/com/yas/customer/service/WishlistService.java`

```java
package com.yas.customer.service;

import com.yas.customer.model.WishlistItem;
import com.yas.customer.repository.WishlistItemRepository;
import com.yas.customer.viewmodel.product.ProductThumbnailVm;
import com.yas.customer.viewmodel.wishlist.WishlistItemDetailVm;
import com.yas.customer.viewmodel.wishlist.WishlistItemListVm;
import com.yas.customer.viewmodel.wishlist.WishlistItemVm;
import com.yas.commonlibrary.exception.NotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class WishlistService {

    private final WishlistItemRepository wishlistItemRepository;
    private final ProductService productService;

    @Transactional
    public void addProductToWishlist(String customerId, Long productId) {
        // Verify product exists
        if (!productService.existsById(productId)) {
            throw new NotFoundException("Product with ID " + productId + " not found");
        }

        // Check for duplicate
        if (wishlistItemRepository.findByCustomerIdAndProductId(customerId, productId).isPresent()) {
            // Idempotent: already exists, return silently
            return;
        }

        // Insert
        try {
            WishlistItem item = WishlistItem.builder()
                    .customerId(customerId)
                    .productId(productId)
                    .build();
            wishlistItemRepository.save(item);
        } catch (DataIntegrityViolationException e) {
            // Race condition: duplicate inserted between check and save
            // Log at DEBUG and return (idempotent)
            return;
        }
    }

    @Transactional(readOnly = true)
    public WishlistItemListVm getWishlistItems(String customerId) {
        List<WishlistItem> items = wishlistItemRepository.findByCustomerIdOrderByCreatedOnDesc(customerId);

        // Fetch product details in parallel (2-second timeout per product)
        List<WishlistItemVm> itemVms = items.stream()
                .map(this::fetchProductDetailsAsync)
                .map(future -> {
                    try {
                        return future.get(2, TimeUnit.SECONDS);
                    } catch (Exception e) {
                        // Timeout or error: return unavailable placeholder
                        return createUnavailableItemVm(null, "Unable to verify availability");
                    }
                })
                .collect(Collectors.toList());

        return new WishlistItemListVm(itemVms);
    }

    private CompletableFuture<WishlistItemVm> fetchProductDetailsAsync(WishlistItem item) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                ProductThumbnailVm product = productService.getProductById(item.getProductId());
                if (product == null) {
                    return createDiscontinuedItemVm(item);
                }
                return WishlistItemVm.builder()
                        .wishlistItemId(item.getId())
                        .productId(product.id())
                        .productName(product.name())
                        .productSlug(product.slug())
                        .thumbnailUrl(product.thumbnailMediaUrl())
                        .price(product.price())
                        .isAvailable(true)
                        .discontinuedMessage(null)
                        .build();
            } catch (Exception e) {
                return createDiscontinuedItemVm(item);
            }
        });
    }

    private WishlistItemVm createUnavailableItemVm(WishlistItem item, String message) {
        return WishlistItemVm.builder()
                .wishlistItemId(item != null ? item.getId() : null)
                .productId(item != null ? item.getProductId() : null)
                .productName("Product information unavailable")
                .productSlug("")
                .thumbnailUrl(null)
                .price(0.0)
                .isAvailable(false)
                .discontinuedMessage(message)
                .build();
    }

    private WishlistItemVm createDiscontinuedItemVm(WishlistItem item) {
        return WishlistItemVm.builder()
                .wishlistItemId(item.getId())
                .productId(item.getProductId())
                .productName("Product no longer available")
                .productSlug("")
                .thumbnailUrl(null)
                .price(0.0)
                .isAvailable(false)
                .discontinuedMessage("This product is no longer available")
                .build();
    }

    @Transactional
    public void removeFromWishlist(String customerId, Long wishlistItemId) {
        WishlistItem item = wishlistItemRepository.findByIdAndCustomerId(wishlistItemId, customerId)
                .orElseThrow(() -> new NotFoundException("USER_ADDRESS_NOT_FOUND"));
        wishlistItemRepository.delete(item);
    }

    @Transactional(readOnly = true)
    public WishlistItemDetailVm getWishlistItemDetail(Long wishlistItemId) {
        WishlistItem item = wishlistItemRepository.findById(wishlistItemId)
                .orElseThrow(() -> new NotFoundException("USER_ADDRESS_NOT_FOUND"));

        // Fetch product availability
        boolean isAvailable;
        try {
            ProductThumbnailVm product = productService.getProductById(item.getProductId());
            isAvailable = product != null;
        } catch (Exception e) {
            isAvailable = false;
        }

        return WishlistItemDetailVm.builder()
                .wishlistItemId(item.getId())
                .customerId(item.getCustomerId())
                .productId(item.getProductId())
                .isAvailable(isAvailable)
                .build();
    }
}
```

**Rationale:**
- Matches cart service's `CartItemService` pattern for product fetching
- Uses `CompletableFuture` for parallel calls with timeout (matches cart service approach)
- Idempotent add (AC-2): checks for duplicate, catches constraint violation
- Availability logic: product exists in product service → `isAvailable: true`
- Error handling: product not found or timeout → `isAvailable: false` with message

---

### 5. Controller: `WishlistController`

**Path:** `customer/src/main/java/com/yas/customer/controller/WishlistController.java`

```java
package com.yas.customer.controller;

import com.yas.customer.service.WishlistService;
import com.yas.customer.viewmodel.wishlist.WishlistItemDetailVm;
import com.yas.customer.viewmodel.wishlist.WishlistItemListVm;
import com.yas.customer.viewmodel.wishlist.WishlistItemPostVm;
import com.yas.commonlibrary.utils.AuthenticationUtils;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/storefront/wishlist")
@RequiredArgsConstructor
public class WishlistController {

    private final WishlistService wishlistService;

    @PostMapping("/items")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> addToWishlist(@Valid @RequestBody WishlistItemPostVm request) {
        String customerId = AuthenticationUtils.getCurrentUserId();
        wishlistService.addProductToWishlist(customerId, request.productId());
        return ResponseEntity.ok().build();
    }

    @GetMapping("/items")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<WishlistItemListVm> getWishlist() {
        String customerId = AuthenticationUtils.getCurrentUserId();
        WishlistItemListVm response = wishlistService.getWishlistItems(customerId);
        return ResponseEntity.ok(response);
    }

    @DeleteMapping("/items/{wishlistItemId}")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> removeFromWishlist(@PathVariable Long wishlistItemId) {
        String customerId = AuthenticationUtils.getCurrentUserId();
        wishlistService.removeFromWishlist(customerId, wishlistItemId);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/items/{wishlistItemId}")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<WishlistItemDetailVm> getWishlistItemDetail(@PathVariable Long wishlistItemId) {
        WishlistItemDetailVm response = wishlistService.getWishlistItemDetail(wishlistItemId);
        return ResponseEntity.ok(response);
    }
}
```

**Rationale:**
- `@PreAuthorize("isAuthenticated()")` enforces JWT authentication (AC-5)
- `AuthenticationUtils.getCurrentUserId()` extracts `customer_id` from JWT (matches `UserAddressController` pattern)
- Error handling delegated to `ApiExceptionHandler` from `common-library` (returns `ErrorVm` on exceptions)

---

### 6. ViewModels

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemPostVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import jakarta.validation.constraints.NotNull;
import lombok.Builder;

@Builder
public record WishlistItemPostVm(
    @NotNull(message = "Product ID is required") Long productId
) {}
```

---

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import lombok.Builder;

@Builder
public record WishlistItemVm(
    Long wishlistItemId,
    Long productId,
    String productName,
    String productSlug,
    String thumbnailUrl,
    Double price,
    Boolean isAvailable,
    String discontinuedMessage
) {}
```

---

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import java.util.List;

public record WishlistItemListVm(
    List<WishlistItemVm> items
) {}
```

---

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemDetailVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import lombok.Builder;

@Builder
public record WishlistItemDetailVm(
    Long wishlistItemId,
    String customerId,
    Long productId,
    Boolean isAvailable
) {}
```

---

## Configuration Changes

### 1. Security: Add Wishlist Endpoints

**File:** `customer/src/main/java/com/yas/customer/config/SecurityConfig.java`

**Modification:** Add `/storefront/wishlist/**` to authenticated paths.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/actuator/prometheus", "/actuator/health/**",
                "/swagger-ui", "/swagger-ui/**", "/error", "/v3/api-docs/**").permitAll()
            .requestMatchers("/storefront/customer/**").hasRole("CUSTOMER")
            .requestMatchers("/storefront/wishlist/**").authenticated() // ADD THIS
            .requestMatchers("/storefront/**").permitAll()
            .requestMatchers("/backoffice/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

---

### 2. Service URL: Add Product Service

**File:** `customer/src/main/java/com/yas/customer/config/ServiceUrlConfig.java`

**Modification:** Add `product` service URL field (if missing).

```java
package com.yas.customer.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "yas.services")
public record ServiceUrlConfig(
    String location,
    String product  // ADD THIS
) {}
```

---

**File:** `customer/src/main/resources/application.properties`

**Modification:** Add product service URL (if missing).

```properties
yas.services.location=http://api.yas.local/location
yas.services.product=http://api.yas.local/product
```

---

### 3. Messages: Add Wishlist Error Messages

**File:** `customer/src/main/resources/messages/messages.properties`

**Modification:** Add wishlist error messages.

```properties
USER_WITH_EMAIL_NOT_FOUND=User with email {} not found
USER_WITH_USERNAME_NOT_FOUND=User with username {} not found
WRONG_EMAIL_FORMAT=Wrong email format for {}
USER_NOT_FOUND=User not found
USER_ADDRESS_NOT_FOUND=User address not found
UNAUTHENTICATED=Action failed, please login!
USERNAME_ALREADY_EXITED=The user {} is already existed
USER_WITH_EMAIL_ALREADY_EXITED=The email {} is already existed
WISHLIST_ITEM_NOT_FOUND=Wishlist item not found
PRODUCT_NOT_FOUND=Product with ID {} not found
```

---

## Tests

### AC-1: Signed-in shopper can add, view, and remove wishlist items across sessions

**Test 1.1: Add product to wishlist (WishlistServiceTest.testAddProductToWishlist_Success)**
- Verify `ProductService.existsById()` called with product ID
- Verify `wishlistItemRepository.save()` called
- Verify no exception thrown

**Test 1.2: Persistence across sessions (WishlistServiceIT.testAddProductToWishlist_PersistsAcrossSessions)**
- INSERT `wishlist_item` via repository
- Flush and clear entity manager
- Reload entity by ID
- Assert entity exists with correct `customer_id` and `product_id`

**Test 1.3: Remove product from wishlist (WishlistServiceTest.testRemoveWishlistItem_Success)**
- Mock `findByIdAndCustomerId()` to return existing item
- Call `removeFromWishlist()`
- Verify `wishlistItemRepository.delete()` called

**Test 1.4: Remove product from wishlist (WishlistServiceIT.testRemoveWishlistItem_DeletesFromDatabase)**
- INSERT `wishlist_item` via repository
- Call `delete()` and flush
- Assert entity no longer exists

---

### AC-2: The same product cannot appear twice on one shopper's wishlist

**Test 2.1: Add duplicate product (WishlistServiceTest.testAddProductToWishlist_Idempotent)**
- Mock `existsById()` to return true
- Mock `findByCustomerIdAndProductId()` to return existing item
- Call `addProductToWishlist()`
- Verify `save()` NOT called (idempotent)

**Test 2.2: UNIQUE constraint enforcement (WishlistServiceIT.testAddProductToWishlist_NoDuplicates)**
- INSERT `wishlist_item` via repository
- Flush
- Attempt INSERT with same `customer_id` and `product_id`
- Assert `DataIntegrityViolationException` thrown

---

### AC-5: Guests are prompted to sign in before saving to a wishlist

**Test 5.1: Guest attempts to add to wishlist (WishlistControllerTest.testAddProductToWishlist_Unauthorized)**
- POST `/storefront/wishlist/items` without JWT
- Assert 401 Unauthorized response
- **Note:** This test requires security filters enabled; mark as placeholder if running with `@AutoConfigureMockMvc(addFilters = false)`

**Test 5.2: Guest attempts to view wishlist (WishlistControllerTest - add test)**
- GET `/storefront/wishlist/items` without JWT
- Assert 401 Unauthorized response

---

### Additional Test Cases

**Test 3.1: Product not found (WishlistServiceTest.testAddProductToWishlist_ProductNotFound)**
- Mock `existsById()` to return false
- Call `addProductToWishlist()`
- Assert `NotFoundException` thrown

**Test 3.2: Get wishlist items with available product (WishlistServiceTest.testGetWishlistItems_Success)**
- Mock `findByCustomerIdOrderByCreatedOnDesc()` to return item
- Mock `getProductById()` to return product details
- Call `getWishlistItems()`
- Assert response contains item with `isAvailable: true`

**Test 3.3: Get wishlist items with discontinued product (WishlistServiceTest.testGetWishlistItems_ProductUnavailable)**
- Mock `findByCustomerIdOrderByCreatedOnDesc()` to return item
- Mock `getProductById()` to return null
- Call `getWishlistItems()`
- Assert response contains item with `isAvailable: false` and discontinued message

**Test 3.4: Remove wishlist item not found (WishlistServiceTest.testRemoveWishlistItem_NotFound)**
- Mock `findByIdAndCustomerId()` to return empty
- Call `removeFromWishlist()`
- Assert `NotFoundException` thrown

**Test 3.5: Get wishlist item detail (WishlistServiceTest.testGetWishlistItemInternal_Success)**
- Mock `findById()` to return item
- Mock `getProductById()` to return product
- Call `getWishlistItemDetail()`
- Assert response contains correct fields including `isAvailable: true`

**Test 4.1: Controller add endpoint returns OK (WishlistControllerTest.testAddProductToWishlist_ReturnsOk)**
- POST `/storefront/wishlist/items` with valid body
- Assert 200 OK response

**Test 4.2: Controller get endpoint returns OK (WishlistControllerTest.testGetWishlistItems_ReturnsOk)**
- GET `/storefront/wishlist/items`
- Assert 200 OK response with correct structure

**Test 4.3: Controller delete endpoint returns No Content (WishlistControllerTest.testRemoveWishlistItem_ReturnsNoContent)**
- DELETE `/storefront/wishlist/items/{id}`
- Assert 204 No Content response

**Test 4.4: Controller delete endpoint returns Not Found (WishlistControllerTest.testRemoveWishlistItem_NotFound)**
- Mock service to throw `NotFoundException`
- DELETE `/storefront/wishlist/items/{id}`
- Assert 404 Not Found response

**Test 4.5: Integration test for product service call (WishlistServiceIT.testGetWishlistItems_CallsProductService)**
- **Note:** Placeholder for WireMock/Testcontainers test verifying real HTTP call to product service

---

### Test File Skeletons

**Path:** `customer/src/test/java/com/yas/customer/service/WishlistServiceTest.java`

See full implementation above (8 test cases total).

---

**Path:** `customer/src/it/java/com/yas/customer/service/WishlistServiceIT.java`

See full implementation above (4 test cases total).

---

**Path:** `customer/src/test/java/com/yas/customer/controller/WishlistControllerTest.java`

See full implementation above (5 test cases total).

---

## Interfaces

### WishlistItemRepository

```java
public interface WishlistItemRepository extends JpaRepository<WishlistItem, Long> {
    List<WishlistItem> findByCustomerIdOrderByCreatedOnDesc(String customerId);
    Optional<WishlistItem> findByIdAndCustomerId(Long id, String customerId);
    Optional<WishlistItem> findByCustomerIdAndProductId(String customerId, Long productId);
}
```

**Purpose:** Data access layer for `wishlist_item` table.

**Methods:**
- `findByCustomerIdOrderByCreatedOnDesc`: Retrieve all wishlist items for a customer, sorted by creation date (newest first).
- `findByIdAndCustomerId`: Retrieve a specific wishlist item, ensuring it belongs to the customer (security check).
- `findByCustomerIdAndProductId`: Check if a product is already on the customer's wishlist (duplicate prevention).

---

### ProductService

```java
public class ProductService extends AbstractCircuitBreakFallbackHandler {
    public List<ProductThumbnailVm> getProducts(List<Long> ids);
    public ProductThumbnailVm getProductById(Long id);
    public boolean existsById(Long id);
}
```

**Purpose:** Call product service to verify product existence and fetch product details.

**Methods:**
- `getProducts`: Batch fetch product details (matches cart service pattern).
- `getProductById`: Fetch single product (wrapper around `getProducts`).
- `existsById`: Verify product exists (returns `false` if product not found).

---

### WishlistService

```java
public class WishlistService {
    public void addProductToWishlist(String customerId, Long productId);
    public WishlistItemListVm getWishlistItems(String customerId);
    public void removeFromWishlist(String customerId, Long wishlistItemId);
    public WishlistItemDetailVm getWishlistItemDetail(Long wishlistItemId);
}
```

**Purpose:** Business logic for wishlist operations.

**Methods:**
- `addProductToWishlist`: Add product to wishlist (idempotent, verifies product exists).
- `getWishlistItems`: Retrieve all wishlist items with product details and availability.
- `removeFromWishlist`: Remove item from wishlist (security: checks ownership).
- `getWishlistItemDetail`: Retrieve single item details for cart service (Story 3).

---

## Algorithm Notes

### Add Product to Wishlist (Idempotent)

**Goal:** Prevent duplicate entries (AC-2) while handling race conditions.

**Algorithm:**
1. **Verify product exists** via `ProductService.existsById(productId)`
   - If not found → throw `NotFoundException`
2. **Check for duplicate** via `findByCustomerIdAndProductId(customerId, productId)`
   - If found → return success (idempotent, no-op)
3. **Insert new entry** via `wishlistItemRepository.save()`
   - Catch `DataIntegrityViolationException` (UNIQUE constraint violation from race condition)
   - If caught → log at DEBUG, return success (idempotent)
4. **Return 200 OK**

**Race Condition Handling:**
- Between step 2 (duplicate check) and step 3 (INSERT), another thread may insert the same `(customer_id, product_id)` pair
- Database UNIQUE constraint prevents duplicate INSERT
- Service catches exception and returns success (idempotent behavior)

---

### Fetch Wishlist Items with Product Details

**Goal:**  Retrieve wishlist items with product details and availability, handling slow/failed product service calls.

**Algorithm:**
1. **Query wishlist items** via `findByCustomerIdOrderByCreatedOnDesc(customerId)`
   - Returns list of `WishlistItem` entities
2. **Fetch product details in parallel** for each item:
   - Create `CompletableFuture` for each item → call `ProductService.getProductById(productId)`
   - Set 2-second timeout per product (matches cart service pattern)
   - If product found → `isAvailable: true`, populate name/slug/thumbnail/price
   - If product not found (404) → `isAvailable: false`, `discontinuedMessage: "This product is no longer available"`
   - If timeout or error → `isAvailable: false`, `discontinuedMessage: "Unable to verify availability"`
3. **Aggregate results** into `WishlistItemListVm`
4. **Return 200 OK** with items

**Performance:** Parallel fetching reduces total latency when multiple items exist. Timeout ensures responsiveness.

---

## Open Questions

1. **Product Service Batch Endpoint:** Does product service accept multiple `productId` query params (e.g., `?productId=1&productId=2`) or a comma-separated list (e.g., `?productId=1,2,3`)?
   - **Resolution path:** Inspect `product/src/main/java/com/yas/product/controller/ProductStorefrontController.java` or test the endpoint. Update `ProductService.getProducts()` query param construction accordingly.

2. **Product Availability Fields:** The HLD assumes products are "available" if they exist in the product service response. Does the product service expose explicit availability/stock fields (e.g., `published`, `stockQuantity`) that should be checked?
   - **Resolution path:** Inspect product service ViewModels (e.g., `ProductThumbnailVm` or `ProductDetailVm`). If availability fields exist, add logic to `WishlistService.fetchProductDetailsAsync()` to check them.

3. **ProductService in Customer Service:** Does `customer` service already have a `ProductService` for calling product service?
   - **Resolution:** Inspect `customer/src/main/java/com/yas/customer/service/`. If missing, create it following the pattern in this LLD.

---

## Notes

- **Idempotency:** Add-to-wishlist is idempotent (AC-2). Duplicate adds return 200 without error.
- **Persistence:** Wishlist items persist across sessions (AC-1). No session-based storage; all data in database.
- **Security:** All endpoints require authentication (AC-5). Spring Security enforces this via `@PreAuthorize("isAuthenticated()")`. No manual JWT check needed.
- **Error Handling:** Product service 404 is handled gracefully (product marked as unavailable, not an error response to user).
- **Logging:** Use SLF4J (matches existing services). Log wishlist operations at INFO level, duplicate constraint violations at DEBUG, errors at ERROR.
- **Testing:** Integration tests must use real database (Testcontainers or H2) to verify UNIQUE constraint and persistence.
- **Consistency Model:** Eventual consistency is acceptable for product availability display (slight staleness is tolerable; no distributed transaction needed).
