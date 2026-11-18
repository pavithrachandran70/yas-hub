# Merged plan — PER-316

Execution order (3 ticket(s)):

1. `1`
2. `2`
3. `3`

Specs (3): `specs/1.md`, `specs/2.md`, `specs/3.md`

---

## Spec `specs/1.md`

```markdown
# Technical Specification — Ticket 1

**Title:** Wishlist persistence and API endpoints  
**Repo:** app  
**Epic:** PER-316 (Signed-in shoppers can save a wishlist and add those items to the cart)

---

## Overview

This story implements the backend persistence layer and REST API for the wishlist feature. It creates the `wishlist_item` table in the customer database, exposes CRUD endpoints for adding/removing/listing wishlist items, and enforces the signed-in-only access requirement. The customer service owns wishlist data; product service provides availability status.

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

## Stack

- **Language/Framework:** Java 17, Spring Boot 3.x, Maven (matches existing `cart`, `customer` services)
- **Database:** PostgreSQL with Liquibase migrations (matches `customer/src/main/resources/db/changelog/`)
- **Security:** Spring Security with Keycloak JWT authentication (matches `customer/src/main/java/com/yas/customer/config/SecurityConfig.java`)
- **REST Client:** RestTemplate or WebClient for calling product service (matches `customer/src/main/java/com/yas/customer/service/LocationService.java` pattern)
- **Testing:** Integration tests in `customer/src/it/`, unit tests in `customer/src/test/`

---

## Schema Changes

### Database Migration

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

**Update:** `customer/src/main/resources/db/changelog/db.changelog-master.yaml`

```yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/ddl/
  - includeAll:
      path: db/changelog/data/
```

(The `includeAll` directive already picks up `changelog-0002.sql` automatically; no additional entry needed.)

---

## API Contract

### Endpoints

**Base URL:** `/api/customer/storefront/wishlist`

#### 1. POST `/api/customer/storefront/wishlist/items`

**Purpose:** Add a product to the signed-in shopper's wishlist (AC-1, AC-2)

**Request:**
```json
{
  "productId": 123
}
```

**Response:**
- `200 OK` — Product added or already exists (idempotent)
- `401 Unauthorized` — Not signed in (AC-5)
- `404 Not Found` — Product does not exist (product service returned 404)

**Behavior:**
- Extract `customer_id` from JWT (`AuthenticationUtils.getCurrentUserId()`)
- Call product service `GET /api/product/storefront/products/{productId}` to verify product exists
- INSERT into `wishlist_item` if not exists (handle UNIQUE constraint violation silently → return 200)
- Return 200 even if product already on list (AC-2 idempotency)

---

#### 2. GET `/api/customer/storefront/wishlist/items`

**Purpose:** List all wishlist items for the signed-in shopper (AC-1)

**Response:**
- `200 OK`
```json
{
  "items": [
    {
      "wishlistItemId": 1,
      "productId": 123,
      "productName": "Example Product",
      "productSlug": "example-product",
      "thumbnailUrl": "https://example.com/image.jpg",
      "price": 29.99,
      "isAvailable": true,
      "discontinuedMessage": null
    },
    {
      "wishlistItemId": 2,
      "productId": 456,
      "productName": "Discontinued Item",
      "productSlug": "discontinued-item",
      "thumbnailUrl": "https://example.com/image2.jpg",
      "price": 49.99,
      "isAvailable": false,
      "discontinuedMessage": "This product is no longer available"
    }
  ]
}
```
- `401 Unauthorized` — Not signed in

**Behavior:**
- SELECT `wishlist_item` WHERE `customer_id` = current user
- For each row, call product service `GET /api/product/storefront/products/{id}` (or batch endpoint if available — see Open Questions)
- Populate `isAvailable` from product's `published` and `stockQuantity` fields (verify exact field names in product service — see Open Questions)
- If product service returns 404 or product is unavailable, set `isAvailable: false` and `discontinuedMessage: "Product information unavailable"` or "This product is no longer available"

---

#### 3. DELETE `/api/customer/storefront/wishlist/items/{wishlistItemId}`

**Purpose:** Remove an item from the wishlist (AC-1)

**Response:**
- `204 No Content` — Deleted successfully
- `401 Unauthorized` — Not signed in
- `404 Not Found` — Item does not exist or does not belong to the current user

**Behavior:**
- DELETE WHERE `id = {wishlistItemId}` AND `customer_id = current user`
- Return 404 if no rows affected (security: cannot delete another user's item)

---

#### 4. GET `/api/customer/storefront/wishlist/items/{wishlistItemId}` (Internal endpoint)

**Purpose:** Retrieve single wishlist item details for cart service (Story 3 will call this)

**Response:**
- `200 OK`
```json
{
  "wishlistItemId": 1,
  "customerId": "uuid-from-keycloak",
  "productId": 123,
  "isAvailable": true
}
```
- `401 Unauthorized` — Not signed in
- `404 Not Found` — Item does not exist

**Behavior:**
- SELECT `wishlist_item` WHERE `id = {wishlistItemId}`
- Call product service to check availability
- Return `customerId`, `productId`, `isAvailable`

---

## File Touch List

### New Files

1. **Model:**
   - `customer/src/main/java/com/yas/customer/model/WishlistItem.java`

2. **Repository:**
   - `customer/src/main/java/com/yas/customer/repository/WishlistItemRepository.java`

3. **Service:**
   - `customer/src/main/java/com/yas/customer/service/WishlistService.java`

4. **Controller:**
   - `customer/src/main/java/com/yas/customer/controller/WishlistController.java`

5. **ViewModels:**
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemPostVm.java`
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemGetVm.java`
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListGetVm.java`
   - `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemInternalVm.java`

6. **Migration:**
   - `customer/src/main/resources/db/changelog/ddl/changelog-0002.sql`

7. **Messages:**
   - Update `customer/src/main/resources/messages/messages.properties` (add error messages)

8. **Tests:**
   - `customer/src/test/java/com/yas/customer/controller/WishlistControllerTest.java`
   - `customer/src/test/java/com/yas/customer/service/WishlistServiceTest.java`
   - `customer/src/it/java/com/yas/customer/service/WishlistServiceIT.java`

---

### Modified Files

1. **Service URL Config:**
   - `customer/src/main/java/com/yas/customer/config/ServiceUrlConfig.java` (verify `product()` service URL exists; add if missing)

2. **Security Config:**
   - `customer/src/main/java/com/yas/customer/config/SecurityConfig.java` (add `/api/customer/storefront/wishlist/**` to authenticated paths)

---

## Implementation Details

### WishlistItem Entity

**File:** `customer/src/main/java/com/yas/customer/model/WishlistItem.java`

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

**Notes:**
- Extends `AbstractAuditEntity` (from `common-library`) for `createdBy`, `createdOn`, `lastModifiedBy`, `lastModifiedOn` fields (matches `UserAddress` pattern)
- `customerId` is `String` (Keycloak user ID from JWT, matches `customer/src/main/java/com/yas/customer/model/UserAddress.java`)
- `productId` is `Long` (FK reference to product service, logical only — no DB-level FK across services)

---

### WishlistItemRepository

**File:** `customer/src/main/java/com/yas/customer/repository/WishlistItemRepository.java`

```java
package com.yas.customer.repository;

import com.yas.customer.model.WishlistItem;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface WishlistItemRepository extends JpaRepository<WishlistItem, Long> {
    List<WishlistItem> findByCustomerId(String customerId);
    Optional<WishlistItem> findByIdAndCustomerId(Long id, String customerId);
    boolean existsByCustomerIdAndProductId(String customerId, Long productId);
}
```

---

### WishlistService

**File:** `customer/src/main/java/com/yas/customer/service/WishlistService.java`

**Key Methods:**

```java
package com.yas.customer.service;

import com.yas.customer.model.WishlistItem;
import com.yas.customer.repository.WishlistItemRepository;
import com.yas.customer.viewmodel.wishlist.*;
import com.yas.commonlibrary.exception.NotFoundException;
import com.yas.commonlibrary.utils.AuthenticationUtils;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class WishlistService {

    private final WishlistItemRepository wishlistItemRepository;
    private final ProductService productService; // verify in `customer` service; add if missing

    @Transactional
    public void addProductToWishlist(WishlistItemPostVm request) {
        String customerId = AuthenticationUtils.getCurrentUserId();
        Long productId = request.productId();

        // Verify product exists (call product service)
        productService.getProduct(productId); // throws NotFoundException if product not found

        // Idempotent: check if already exists
        if (!wishlistItemRepository.existsByCustomerIdAndProductId(customerId, productId)) {
            WishlistItem item = new WishlistItem();
            item.setCustomerId(customerId);
            item.setProductId(productId);
            wishlistItemRepository.save(item);
        }
        // Return success even if already exists (AC-2 idempotency)
    }

    public WishlistItemListGetVm getWishlistItems() {
        String customerId = AuthenticationUtils.getCurrentUserId();
        List<WishlistItem> items = wishlistItemRepository.findByCustomerId(customerId);

        List<WishlistItemGetVm> itemVms = items.stream()
            .map(this::toWishlistItemGetVm)
            .collect(Collectors.toList());

        return new WishlistItemListGetVm(itemVms);
    }

    private WishlistItemGetVm toWishlistItemGetVm(WishlistItem item) {
        // Call product service to get product details and availability
        // Handle 404 gracefully (product deleted or unavailable)
        try {
            ProductDetailVm product = productService.getProduct(item.getProductId());
            boolean isAvailable = product.published() && product.stockQuantity() > 0;
            String discontinuedMessage = isAvailable ? null : "This product is no longer available";

            return new WishlistItemGetVm(
                item.getId(),
                product.id(),
                product.name(),
                product.slug(),
                product.thumbnailMediaUrl(),
                product.price(),
                isAvailable,
                discontinuedMessage
            );
        } catch (NotFoundException e) {
            // Product not found in product service
            return new WishlistItemGetVm(
                item.getId(),
                item.getProductId(),
                "Product information unavailable",
                null,
                null,
                null,
                false,
                "Product information unavailable"
            );
        }
    }

    @Transactional
    public void removeWishlistItem(Long wishlistItemId) {
        String customerId = AuthenticationUtils.getCurrentUserId();
        WishlistItem item = wishlistItemRepository.findByIdAndCustomerId(wishlistItemId, customerId)
            .orElseThrow(() -> new NotFoundException("Wishlist item not found"));
        wishlistItemRepository.delete(item);
    }

    public WishlistItemInternalVm getWishlistItemInternal(Long wishlistItemId) {
        String customerId = AuthenticationUtils.getCurrentUserId();
        WishlistItem item = wishlistItemRepository.findByIdAndCustomerId(wishlistItemId, customerId)
            .orElseThrow(() -> new NotFoundException("Wishlist item not found"));

        // Call product service to check availability
        boolean isAvailable;
        try {
            ProductDetailVm product = productService.getProduct(item.getProductId());
            isAvailable = product.published() && product.stockQuantity() > 0;
        } catch (NotFoundException e) {
            isAvailable = false;
        }

        return new WishlistItemInternalVm(
            item.getId(),
            item.getCustomerId(),
            item.getProductId(),
            isAvailable
        );
    }
}
```

**Notes:**
- `ProductService` must be verified in `customer` service. If it does not exist, add it following the pattern of `LocationService` (see `customer/src/main/java/com/yas/customer/service/LocationService.java`).
- `ProductDetailVm` fields (`published`, `stockQuantity`) must be verified in product service's actual response (see Open Questions).
- Error handling: 404 from product service is caught and handled gracefully (product marked as unavailable).

---

### WishlistController

**File:** `customer/src/main/java/com/yas/customer/controller/WishlistController.java`

```java
package com.yas.customer.controller;

import com.yas.customer.service.WishlistService;
import com.yas.customer.viewmodel.wishlist.*;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/customer/storefront/wishlist")
@RequiredArgsConstructor
@Tag(name = "Wishlist", description = "Wishlist management APIs")
public class WishlistController {

    private final WishlistService wishlistService;

    @PostMapping("/items")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Add product to wishlist")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Product added to wishlist"),
        @ApiResponse(responseCode = "401", description = "Unauthorized"),
        @ApiResponse(responseCode = "404", description = "Product not found")
    })
    public ResponseEntity<Void> addProductToWishlist(@Valid @RequestBody WishlistItemPostVm request) {
        wishlistService.addProductToWishlist(request);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/items")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Get wishlist items")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Wishlist items retrieved"),
        @ApiResponse(responseCode = "401", description = "Unauthorized")
    })
    public ResponseEntity<WishlistItemListGetVm> getWishlistItems() {
        return ResponseEntity.ok(wishlistService.getWishlistItems());
    }

    @DeleteMapping("/items/{wishlistItemId}")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Remove item from wishlist")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "204", description = "Item removed"),
        @ApiResponse(responseCode = "401", description = "Unauthorized"),
        @ApiResponse(responseCode = "404", description = "Item not found")
    })
    public ResponseEntity<Void> removeWishlistItem(@PathVariable Long wishlistItemId) {
        wishlistService.removeWishlistItem(wishlistItemId);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/items/{wishlistItemId}")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Get single wishlist item (internal use)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Item retrieved"),
        @ApiResponse(responseCode = "401", description = "Unauthorized"),
        @ApiResponse(responseCode = "404", description = "Item not found")
    })
    public ResponseEntity<WishlistItemInternalVm> getWishlistItemInternal(@PathVariable Long wishlistItemId) {
        return ResponseEntity.ok(wishlistService.getWishlistItemInternal(wishlistItemId));
    }
}
```

---

### ViewModels

**File:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemPostVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import jakarta.validation.constraints.NotNull;

public record WishlistItemPostVm(
    @NotNull(message = "Product ID is required")
    Long productId
) {}
```

---

**File:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemGetVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import java.math.BigDecimal;

public record WishlistItemGetVm(
    Long wishlistItemId,
    Long productId,
    String productName,
    String productSlug,
    String thumbnailUrl,
    BigDecimal price,
    boolean isAvailable,
    String discontinuedMessage
) {}
```

---

**File:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListGetVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import java.util.List;

public record WishlistItemListGetVm(
    List<WishlistItemGetVm> items
) {}
```

---

**File:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemInternalVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

public record WishlistItemInternalVm(
    Long wishlistItemId,
    String customerId,
    Long productId,
    boolean isAvailable
) {}
```

---

## Security Configuration

**File:** `customer/src/main/java/com/yas/customer/config/SecurityConfig.java`

**Modification:** Add wishlist endpoints to authenticated paths.

```java
// Existing SecurityConfig.java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    // ... existing config ...

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/api/customer/storefront/wishlist/**").authenticated() // ADD THIS
                .anyRequest().authenticated()
            )
            // ... rest of config
            .build();
    }
}
```

---

## Tests

### Unit Tests

**File:** `customer/src/test/java/com/yas/customer/service/WishlistServiceTest.java`

**Test Cases:**
1. `testAddProductToWishlist_Success` — Add a product, verify repository save called
2. `testAddProductToWishlist_Idempotent` — Add existing product, verify no duplicate save
3. `testAddProductToWishlist_ProductNotFound` — Product service returns 404, verify NotFoundException thrown
4. `testGetWishlistItems_Success` — Retrieve items, verify product service called for each item
5. `testGetWishlistItems_ProductUnavailable` — Product service returns 404 or unavailable, verify `isAvailable: false`
6. `testRemoveWishlistItem_Success` — Remove item, verify repository delete called
7. `testRemoveWishlistItem_NotFound` — Item does not exist, verify NotFoundException thrown
8. `testGetWishlistItemInternal_Success` — Retrieve internal view, verify fields

---

### Integration Tests

**File:** `customer/src/it/java/com/yas/customer/service/WishlistServiceIT.java`

**Test Cases:**
1. `testAddProductToWishlist_PersistsAcrossSessions` — Add product, flush session, reload, verify item exists (AC-1)
2. `testAddProductToWishlist_NoDuplicates` — Add same product twice, verify UNIQUE constraint (AC-2)
3. `testGetWishlistItems_CallsProductService` — Verify real HTTP call to product service (use WireMock or Testcontainers)
4. `testRemoveWishlistItem_DeletesFromDatabase` — Remove item, verify row deleted

---

### Controller Tests

**File:** `customer/src/test/java/com/yas/customer/controller/WishlistControllerTest.java`

**Test Cases:**
1. `testAddProductToWishlist_ReturnsOk` — POST `/items`, verify 200
2. `testAddProductToWishlist_Unauthorized` — POST without JWT, verify 401 (AC-5)
3. `testGetWishlistItems_ReturnsOk` — GET `/items`, verify 200 and correct response structure
4. `testRemoveWishlistItem_ReturnsNoContent` — DELETE `/items/{id}`, verify 204
5. `testRemoveWishlistItem_NotFound` — DELETE non-existent item, verify 404

---

## Error Messages

**File:** `customer/src/main/resources/messages/messages.properties`

Add:
```properties
wishlist.item.not_found=Wishlist item not found
product.not_found=Product not found
product.unavailable=This product is no longer available
```

---

## Risks

### RISK-1: Product Service Latency on Wishlist List
**Description:** Fetching product details for every wishlist item in a loop could cause slow response times if a user has many items.

**Mitigation:**
- **Decision needed in this story:** Implement batch product fetch or parallel calls?
- **Option A:** Add batch endpoint in product service: `POST /api/product/storefront/products/batch` with `{"productIds": [123, 456, ...]}`
- **Option B:** Use parallel calls with CompletableFuture or WebClient (non-blocking); timeout after 2s per product
- **Decision:** Investigate whether product service already has a batch endpoint (see Open Questions). If not, implement **Option B** (parallel calls) in this story, and propose batch endpoint as a future optimization.

**Owner:** This story

---

### RISK-2: Product Service Availability Fields
**Description:** Exact field names for `published` and `stockQuantity` in product service response are not confirmed.

**Mitigation:**
- **Action in this story:** Inspect product service code (`product/src/main/java/com/yas/product/viewmodel/product/ProductDetailVm.java` or equivalent) before implementation.
- **If fields differ:** Adjust `WishlistService.toWishlistItemGetVm()` to use correct field names.
- **Document actual field names** in this spec during implementation.

**Owner:** This story

---

## Open Questions

1. **Product Service Batch Endpoint:** Does product service already have a batch product fetch endpoint?
   - **Resolution:** Check `product/src/main/java/com/yas/product/controller/ProductStorefrontController.java`. If missing, implement parallel calls in `WishlistService.getWishlistItems()`.

2. **Product Availability Fields:** Exact field names for `published` and `stockQuantity` in product service response?
   - **Resolution:** Inspect product service ViewModels before implementation. Update `toWishlistItemGetVm()` with correct field names.

3. **ProductService in Customer Service:** Does `customer` service already have a `ProductService` for calling product service?
   - **Resolution:** Check `customer/src/main/java/com/yas/customer/service/`. If missing, create it following the pattern of `LocationService` (see `customer/src/main/java/com/yas/customer/service/LocationService.java`).

---

## Notes

- **Idempotency:** Add-to-wishlist is idempotent (AC-2). No error on duplicate; always return 200.
- **Persistence:** Wishlist items persist across sessions (AC-1). No session-based storage; all data in database.
- **Security:** All endpoints require authentication (AC-5). Spring Security enforces this; no manual JWT check needed.
- **Error Handling:** Product service 404 is handled gracefully (product marked as unavailable, not an error response to user).
- **Logging:** Use SLF4J (matches existing services). Log wishlist operations at INFO level, errors at ERROR.
- **Testing:** Integration tests must use real database (Testcontainers or H2) to verify UNIQUE constraint and persistence.

---

## Verification

After implementation, verify:
1. Database migration runs successfully: `wishlist_item` table created with correct schema
2. UNIQUE constraint enforced: Adding duplicate product returns 200 without duplicate row
3. Persistence across sessions: Item added, app restarted, item still exists
4. Guest access blocked: No JWT → 401 on all endpoints
5. Product unavailable handling: Product service 404 → `isAvailable: false` in response, no error thrown
6. Integration tests pass: All test cases in `WishlistServiceIT` and `WishlistControllerTest` pass

---

## Dependencies

- **Downstream (this story depends on):** Product service must expose `GET /api/product/storefront/products/{id}` endpoint (already exists; verify before implementation)
- **Upstream (other stories depend on this):** Story 2 (Storefront wishlist page) and Story 3 (Add-to-cart integration) depend on these APIs
```

## Spec `specs/2.md`

# Technical Specification — Ticket 2

**Epic:** Signed-in shoppers can save a wishlist and add those items to the cart  
**Ticket ID:** 2  
**Repo:** app

---

## Overview

This ticket implements the backend wishlist storage layer and REST API endpoints for a signed-in shopper's wishlist. It introduces a new `wishlist_item` table in the customer database, exposes CRUD endpoints for wishlist operations (add, list, remove), and provides an internal endpoint for the cart service to retrieve wishlist item details when adding to cart. Guest users are blocked at the API level (401 Unauthorized). The customer service owns the wishlist data, calls the product service to fetch item details and availability, and enforces idempotent adds (duplicate prevention via unique constraint).

---

## Stack

- **Language:** Java 17
- **Framework:** Spring Boot 3.x, Spring Security with Keycloak JWT authentication
- **Database:** PostgreSQL, Liquibase migrations (matches existing `customer` service schema versioning)
- **REST:** JSON over HTTP, standard error responses via `ErrorVm` from `common-library`
- **Existing patterns:** `SecurityConfig`, `AuthenticationUtils`, `AbstractAuditEntity` (audit fields), `ApiExceptionHandler`

---

## Schema Changes

### Customer Database

**Migration:** `customer/src/main/resources/db/changelog/ddl/changelog-0002.sql`

```sql
--liquibase formatted sql

--changeset wishlist:issue-316-1
CREATE TABLE wishlist_item (
    id BIGSERIAL PRIMARY KEY,
    customer_id VARCHAR(255) NOT NULL,
    product_id BIGINT NOT NULL,
    created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_modified_date TIMESTAMP,
    CONSTRAINT uk_wishlist_customer_product UNIQUE (customer_id, product_id)
);

CREATE INDEX idx_wishlist_customer ON wishlist_item (customer_id);
```

**Liquibase changeset:** Add to `customer/src/main/resources/db/changelog/db.changelog-master.yaml`:

```yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/ddl/
  - includeAll:
      path: db/changelog/data/
```

(The existing `includeAll` entry for `ddl/` will pick up `changelog-0002.sql` automatically.)

**Rationale:** Composite business key (`customer_id`, `product_id`) with surrogate ID for ORM. Matches `cart_item` pattern (see `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql`). Audit fields follow `AbstractAuditEntity` convention from `common-library`.

---

## API Contract

### Customer Service — Wishlist Endpoints

**Base URL:** `/api/customer/storefront/wishlist`

---

#### `POST /api/customer/storefront/wishlist/items`

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
    "statusCode": 401,
    "title": "Unauthorized",
    "detail": "Authentication required",
    "fieldErrors": []
  }
  ```
- **404 Not Found** — Product does not exist (call to product service `GET /api/product/storefront/products/{id}` returned 404)
  ```json
  {
    "statusCode": 404,
    "title": "Not Found",
    "detail": "Product not found",
    "fieldErrors": []
  }
  ```

**Behavior:**

1. Extract `customer_id` from JWT (`AuthenticationUtils.getCurrentUserId()`).
2. Call product service `GET /api/product/storefront/products/{productId}` to verify product exists. If 404, return 404.
3. INSERT `wishlist_item` (`customer_id`, `product_id`, `created_date`). Handle UNIQUE constraint violation silently → return 200 (idempotent, AC-2).
4. Return 200 with success message.

**Error Handling:**

- Product service timeout or 5xx → return 500 with message "Unable to verify product availability".
- Database constraint violation (duplicate) → catch, log at DEBUG, return 200.
- Any other database error → 500 Internal Server Error.

---

#### `GET /api/customer/storefront/wishlist/items`

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

1. Extract `customer_id` from JWT.
2. SELECT `wishlist_item` WHERE `customer_id` = current user, ORDER BY `created_date` DESC.
3. For each `wishlist_item`, call product service `GET /api/product/storefront/products/{productId}`.
   - **Batch fetch strategy (preferred):** If product service has a batch endpoint `POST /api/product/storefront/products/batch` with body `{"productIds": [123, 456, ...]}`, use it. Otherwise, make parallel calls with `CompletableFuture` or `WebClient` (non-blocking), timeout after 2 seconds per product.
   - **Availability logic (verify field names in product service `ProductDetailVm` before implementation):**
     - Product is **available** if product service returns 200 AND `published == true` AND `stockQuantity > 0`.
     - Product is **discontinued** if product service returns 404 OR `published == false`.
     - If product service call fails (timeout, 5xx), treat as unavailable: `isAvailable: false`, `discontinuedMessage: "Unable to verify availability"`.
4. Populate response with product details (`name`, `slug`, `thumbnailMediaUrl`, `price`), `isAvailable`, and `discontinuedMessage` (if unavailable).
5. Return 200 with items array.

**Error Handling:**

- Product service timeout for a single product → mark that product as unavailable, continue processing remaining items.
- Database error → 500 Internal Server Error.

**Open Question:** Exact field names in product service response (`published` vs `isPublished`, `stockQuantity` vs `availableQuantity`) — verify in `product/src/main/java/com/yas/product/viewmodel/product/ProductDetailVm.java` before implementation.

---

#### `DELETE /api/customer/storefront/wishlist/items/{wishlistItemId}`

**Purpose:** Remove an item from the signed-in shopper's wishlist (AC-1).

**Authentication:** Required (JWT)

**Response:**

- **204 No Content** — Deleted successfully
- **401 Unauthorized** — Not signed in
- **404 Not Found** — Item does not exist or does not belong to the current user

**Behavior:**

1. Extract `customer_id` from JWT.
2. DELETE `wishlist_item` WHERE `id = {wishlistItemId}` AND `customer_id = current user`.
3. If no rows affected → 404 (security: cannot delete another user's item).
4. Return 204 on success.

**Error Handling:**

- Database error → 500 Internal Server Error.

---

#### `GET /api/customer/storefront/wishlist/items/{wishlistItemId}` (Internal)

**Purpose:** Retrieve wishlist item details for cart service when adding to cart (AC-3).

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

1. SELECT `wishlist_item` WHERE `id = {wishlistItemId}`.
2. If not found → 404.
3. Call product service `GET /api/product/storefront/products/{productId}` to determine availability (same logic as list endpoint).
4. Return `{ wishlistItemId, customerId, productId, isAvailable }`.

**Security:** This endpoint is for cart service only. The cart service will validate that the returned `customerId` matches the current JWT user (see cart service ticket 3).

---

## File Touch List

### New Files

- `customer/src/main/resources/db/changelog/ddl/changelog-0002.sql` — Liquibase migration for `wishlist_item` table
- `customer/src/main/java/com/yas/customer/model/WishlistItem.java` — JPA entity
- `customer/src/main/java/com/yas/customer/repository/WishlistItemRepository.java` — Spring Data JPA repository
- `customer/src/main/java/com/yas/customer/service/WishlistService.java` — Business logic (add, list, remove, getById)
- `customer/src/main/java/com/yas/customer/controller/WishlistController.java` — REST controller for storefront wishlist endpoints
- `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemPostVm.java` — Request ViewModel for POST (add product)
- `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemVm.java` — Response ViewModel for single wishlist item (list response)
- `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListVm.java` — Response ViewModel for list endpoint (`{ items: [...] }`)
- `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemDetailVm.java` — Response ViewModel for internal `GET /{id}` endpoint (for cart service)

### Modified Files

- `customer/src/main/resources/db/changelog/db.changelog-master.yaml` — Add reference to `changelog-0002.sql` (auto-included via `includeAll`)
- `customer/src/main/java/com/yas/customer/config/SecurityConfig.java` — Add `/api/customer/storefront/wishlist/**` to authenticated paths (verify exact path matchers)

---

## Implementation Details

### JPA Entity: `WishlistItem`

**Path:** `customer/src/main/java/com/yas/customer/model/WishlistItem.java`

```java
package com.yas.customer.model;

import com.yas.commonlibrary.model.AbstractAuditEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "wishlist_item", uniqueConstraints = {
    @UniqueConstraint(name = "uk_wishlist_customer_product", columnNames = {"customer_id", "product_id"})
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class WishlistItem extends AbstractAuditEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "customer_id", nullable = false)
    private String customerId;

    @Column(name = "product_id", nullable = false)
    private Long productId;
}
```

**Rationale:** Extends `AbstractAuditEntity` (from `common-library`) for `createdDate` and `lastModifiedDate` audit fields. No foreign key to product service (cross-service boundary). `@UniqueConstraint` enforces AC-2 (no duplicates).

---

### Repository: `WishlistItemRepository`

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

    List<WishlistItem> findByCustomerIdOrderByCreatedDateDesc(String customerId);

    Optional<WishlistItem> findByIdAndCustomerId(Long id, String customerId);

    Optional<WishlistItem> findByCustomerIdAndProductId(String customerId, Long productId);
}
```

**Methods:**

- `findByCustomerIdOrderByCreatedDateDesc(String customerId)` — List endpoint query (AC-1)
- `findByIdAndCustomerId(Long id, String customerId)` — Delete endpoint security check (AC-1)
- `findByCustomerIdAndProductId(String customerId, Long productId)` — Check for duplicate before add (AC-2)

---

### Service: `WishlistService`

**Path:** `customer/src/main/java/com/yas/customer/service/WishlistService.java`

```java
package com.yas.customer.service;

import com.yas.customer.model.WishlistItem;
import com.yas.customer.repository.WishlistItemRepository;
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
    private final ProductService productService; // Existing service for product API calls

    @Transactional
    public void addProductToWishlist(String customerId, Long productId) {
        // Verify product exists
        productService.getProduct(productId); // Throws NotFoundException if 404

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
        List<WishlistItem> items = wishlistItemRepository.findByCustomerIdOrderByCreatedDateDesc(customerId);

        // Fetch product details in parallel (or batch if product service supports it)
        List<WishlistItemVm> itemVms = items.stream()
                .map(item -> fetchProductDetailsAsync(item))
                .map(future -> {
                    try {
                        return future.get(2, TimeUnit.SECONDS); // 2-second timeout per product
                    } catch (Exception e) {
                        // Timeout or error: return unavailable placeholder
                        return createUnavailableItemVm(item);
                    }
                })
                .collect(Collectors.toList());

        return new WishlistItemListVm(itemVms);
    }

    private CompletableFuture<WishlistItemVm> fetchProductDetailsAsync(WishlistItem item) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                // Call product service (verify exact method signature in ProductService)
                // ProductDetailVm product = productService.getProduct(item.getProductId());
                // Map to WishlistItemVm with availability logic (see notes below)
                // Return WishlistItemVm
                throw new UnsupportedOperationException("Implement product service call and mapping");
            } catch (NotFoundException e) {
                // Product not found: return as discontinued
                return createDiscontinuedItemVm(item);
            } catch (Exception e) {
                // Other error: return as unavailable
                return createUnavailableItemVm(item);
            }
        });
    }

    private WishlistItemVm createUnavailableItemVm(WishlistItem item) {
        return WishlistItemVm.builder()
                .wishlistItemId(item.getId())
                .productId(item.getProductId())
                .productName("Product information unavailable")
                .productSlug("")
                .thumbnailUrl(null)
                .price(0.0)
                .isAvailable(false)
                .discontinuedMessage("Unable to verify availability")
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
                .orElseThrow(() -> new NotFoundException("Wishlist item not found"));
        wishlistItemRepository.delete(item);
    }

    @Transactional(readOnly = true)
    public WishlistItemDetailVm getWishlistItemDetail(Long wishlistItemId) {
        WishlistItem item = wishlistItemRepository.findById(wishlistItemId)
                .orElseThrow(() -> new NotFoundException("Wishlist item not found"));

        // Fetch product availability
        boolean isAvailable;
        try {
            // Call product service and check availability
            // ProductDetailVm product = productService.getProduct(item.getProductId());
            // isAvailable = product.isPublished() && product.getStockQuantity() > 0;
            throw new UnsupportedOperationException("Implement product availability check");
        } catch (NotFoundException e) {
            isAvailable = false;
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

**Open Implementation Notes:**

1. **Product Service Integration:** Verify method signature and return type in `customer/src/main/java/com/yas/customer/service/ProductService.java`. Replace `throw new UnsupportedOperationException(...)` with actual calls.
2. **Batch Fetch Strategy:** Check if product service has a batch endpoint. If yes, replace parallel `CompletableFuture` with batch call. If no, implement parallel fetch with timeout.
3. **Availability Field Names:** Verify exact field names in product service `ProductDetailVm` (e.g., `published` vs `isPublished`, `stockQuantity` vs `availableQuantity`). Update availability logic accordingly.

---

### Controller: `WishlistController`

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
@RequestMapping("/api/customer/storefront/wishlist")
@RequiredArgsConstructor
public class WishlistController {

    private final WishlistService wishlistService;

    @PostMapping("/items")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> addToWishlist(@Valid @RequestBody WishlistItemPostVm request) {
        String customerId = AuthenticationUtils.getCurrentUserId();
        wishlistService.addProductToWishlist(customerId, request.getProductId());
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

- `@PreAuthorize("isAuthenticated()")` enforces JWT authentication (AC-5).
- `AuthenticationUtils.getCurrentUserId()` extracts `customer_id` from JWT (matches existing `CustomerController` pattern).
- Error handling delegated to `ApiExceptionHandler` from `common-library` (returns `ErrorVm` on exceptions).

---

### ViewModels

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemPostVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import jakarta.validation.constraints.NotNull;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@NoArgsConstructor
@AllArgsConstructor
public class WishlistItemPostVm {

    @NotNull(message = "Product ID is required")
    private Long productId;
}
```

---

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class WishlistItemVm {

    private Long wishlistItemId;
    private Long productId;
    private String productName;
    private String productSlug;
    private String thumbnailUrl;
    private Double price;
    private Boolean isAvailable;
    private String discontinuedMessage;
}
```

---

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.util.List;

@Getter
@NoArgsConstructor
@AllArgsConstructor
public class WishlistItemListVm {

    private List<WishlistItemVm> items;
}
```

---

**Path:** `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemDetailVm.java`

```java
package com.yas.customer.viewmodel.wishlist;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class WishlistItemDetailVm {

    private Long wishlistItemId;
    private String customerId;
    private Long productId;
    private Boolean isAvailable;
}
```

---

## Security Configuration

**Path:** `customer/src/main/java/com/yas/customer/config/SecurityConfig.java`

**Modification:** Add `/api/customer/storefront/wishlist/**` to authenticated paths. Verify exact configuration in existing `SecurityConfig` (matches `CustomerController` and `UserAddressController` patterns).

**Example (adjust to match existing config):**

```java
http.authorizeHttpRequests(authorize -> authorize
    .requestMatchers("/api/customer/storefront/wishlist/**").authenticated()
    .anyRequest().permitAll()
);
```

---

## Risks

### RISK-1: Product Service Latency on Wishlist List

**Description:** Fetching product details for every wishlist item in a loop could cause slow response times if a user has many items.

**Mitigation:**

- **Preferred:** Implement batch product fetch endpoint in product service: `POST /api/product/storefront/products/batch` with `{"productIds": [123, 456, ...]}`.
- **Fallback (this ticket):** Parallel calls with `CompletableFuture`, 2-second timeout per product.
- **Monitoring:** Log slow wishlist list calls; alert if p95 > 1s.

**Owner:** This ticket (Story 1) must decide batch vs parallel strategy. If batch endpoint does not exist, raise as open question for product service team.

---

### RISK-2: Orphaned Wishlist Items After Product Deletion

**Description:** If a product is hard-deleted from product service, wishlist items referencing it remain.

**Mitigation:**

- Soft-delete products (set `published = false`) instead of hard-delete — product service decision (outside this feature's scope).
- Wishlist list endpoint handles 404 from product service gracefully (mark as discontinued, AC-4).
- **Future enhancement:** Periodic cleanup job to remove wishlist items for products deleted > 90 days ago.

**Owner:** Product service (existing behavior); no change needed for this ticket.

---

### RISK-3: Duplicate Check Race Condition

**Description:** Between the duplicate check (`findByCustomerIdAndProductId`) and the INSERT, another thread could insert the same item, causing a constraint violation.

**Mitigation:**

- Catch `DataIntegrityViolationException` in `addProductToWishlist()`, log at DEBUG, return success (idempotent).
- UNIQUE constraint in database is the final enforcement mechanism (AC-2).

**Owner:** This ticket (Story 1).

---

## Tests

### AC-1: Signed-in shopper can add, view, and remove wishlist items across sessions

**Test 1.1: Add product to wishlist**

- **Given:** Signed-in user, valid product ID
- **When:** POST `/api/customer/storefront/wishlist/items` with `{"productId": 123}`
- **Then:** 200 OK, product appears in GET `/api/customer/storefront/wishlist/items`

**Test 1.2: Persistence across sessions**

- **Given:** Signed-in user with wishlist items
- **When:** User signs out (JWT expires), signs back in with new JWT, GET `/api/customer/storefront/wishlist/items`
- **Then:** Same items are returned (verify `customer_id` persistence)

**Test 1.3: Remove product from wishlist**

- **Given:** Signed-in user with product ID 123 on wishlist
- **When:** DELETE `/api/customer/storefront/wishlist/items/{wishlistItemId}`
- **Then:** 204 No Content, product no longer in GET `/api/customer/storefront/wishlist/items`

---

### AC-2: The same product cannot appear twice on one shopper's wishlist

**Test 2.1: Add duplicate product**

- **Given:** Signed-in user, product ID 123 already on wishlist
- **When:** POST `/api/customer/storefront/wishlist/items` with `{"productId": 123}` again
- **Then:** 200 OK, GET `/api/customer/storefront/wishlist/items` returns only one entry for product 123

**Test 2.2: Concurrent duplicate adds**

- **Given:** Signed-in user, product ID 123 not on wishlist
- **When:** Two concurrent POST requests with `{"productId": 123}`
- **Then:** Both return 200 OK, GET `/api/customer/storefront/wishlist/items` returns only one entry for product 123 (UNIQUE constraint enforced)

---

### AC-4: Discontinued products stay on the wishlist but cannot be added to the cart

**Test 4.1: Discontinued product displayed on wishlist**

- **Given:** Signed-in user with product ID 456 on wishlist, product service returns `published: false` for product 456
- **When:** GET `/api/customer/storefront/wishlist/items`
- **Then:** Response includes product 456 with `isAvailable: false`, `discontinuedMessage: "This product is no longer available"`

**Test 4.2: Product not found (404 from product service)**

- **Given:** Signed-in user with product ID 789 on wishlist, product service returns 404 for product 789
- **When:** GET `/api/customer/storefront/wishlist/items`
- **Then:** Response includes product 789 with `isAvailable: false`, `discontinuedMessage: "This product is no longer available"`

---

### AC-5: Guests are prompted to sign in before saving to a wishlist

**Test 5.1: Guest attempts to add to wishlist**

- **Given:** No JWT (guest user)
- **When:** POST `/api/customer/storefront/wishlist/items` with `{"productId": 123}`
- **Then:** 401 Unauthorized, no wishlist data saved

**Test 5.2: Guest attempts to view wishlist**

- **Given:** No JWT (guest user)
- **When:** GET `/api/customer/storefront/wishlist/items`
- **Then:** 401 Unauthorized

---

## Open Questions

1. **Product Service Batch Endpoint:** Does product service have a batch product fetch endpoint (`POST /api/product/storefront/products/batch`)? If not, this ticket will implement parallel calls with `CompletableFuture` and 2-second timeout.
   - **Resolution path:** Check `product/src/main/java/com/yas/product/controller/ProductStorefrontController.java`. If missing, raise to product service team.

2. **Product Availability Field Names:** Exact field names for `published` and `stockQuantity` in product service `ProductDetailVm` — verify in `product/src/main/java/com/yas/product/viewmodel/product/ProductDetailVm.java` before implementation.
   - **Resolution path:** Inspect product service ViewModels and update `WishlistService.fetchProductDetailsAsync()` availability logic.

3. **ProductService Method Signature:** Verify signature of `customer/src/main/java/com/yas/customer/service/ProductService.java` method for fetching product details. Update `WishlistService` calls accordingly.
   - **Resolution path:** Inspect `ProductService` class before implementation.

---

## Notes

- **Idempotency:** Add-to-wishlist is idempotent (AC-2). Duplicate adds return 200 without error.
- **Error Handling:** All errors return `ErrorVm` from `common-library` (matches existing customer service pattern).
- **Logging:** Use SLF4J (existing pattern). Log wishlist operations at INFO level, duplicate constraint violations at DEBUG, errors at ERROR.
- **Testing:** Integration tests in `customer/src/it/` for wishlist CRUD. Unit tests for `WishlistService` logic (mocking product service calls).
- **Consistency Model:** Eventual consistency is acceptable for product availability display (slight staleness is tolerable; no distributed transaction needed).

## Spec `specs/3.md`

# Technical Stub — PER-316-3

**Ticket:** Add wishlist item to cart  
**Story:** Shopper can add a saved wishlist item to the cart; the item remains on the wishlist after being added  

---

## Stack

**Repository:** `app`  
**Language:** Java 17  
**Framework:** Spring Boot 3.x, Maven  
**Database:** PostgreSQL with Liquibase migrations  
**API Style:** REST over HTTP, JSON payloads  
**Security:** Spring Security with Keycloak JWT authentication  
**Service Architecture:** Microservices with domain-driven boundaries  

**Affected Services:**
- **Cart service** (`cart/`) — Receives wishlist items via new endpoint, adds to cart
- **Customer service** (`customer/`) — Provides wishlist item details to cart service (new internal endpoint)
- **Product service** (`product/`) — Provides product details and availability (existing endpoint)

**Existing Patterns:**
- Controllers extend security config (e.g., `cart/src/main/java/com/yas/cart/controller/CartItemController.java`)
- Service layer uses `RestClient` for inter-service calls (e.g., `cart/src/main/java/com/yas/cart/service/ProductService.java`)
- Entities follow `AbstractAuditEntity` pattern (e.g., `cart/src/main/java/com/yas/cart/model/CartItem.java`)
- ViewModels use `*Vm` suffix (e.g., `cart/src/main/java/com/yas/cart/viewmodel/CartItemPostVm.java`)
- Liquibase migrations in `src/main/resources/db/changelog/` (e.g., `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql`)

---

## Schema and API Contracts

### Database Schema Changes

**None for this story.** The `cart_item` table schema is unchanged. The story uses existing cart storage.

**Prerequisite:** Story 1 (wishlist persistence) must deliver the `wishlist_item` table in the customer service database before this story can function. This story does not create or modify that table.

**Cart Service Database** (`cart/src/main/resources/db/changelog/ddl/`) — No new migrations. Existing schema:
- `cart_item` table: `id`, `quantity`, `parent_product_id`, `product_id`, `cart_id` (from `changelog-0001.sql`)
- Composite unique constraint on `cart_item` (verify: does the existing schema enforce UNIQUE(`customer_id`, `product_id`) or equivalent? If not, this story must add it in a new migration `ddl/changelog-0005.sql`)

**Action Required:** Verify in `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql` whether `cart_item` enforces uniqueness per customer/product. If missing, add:

```sql
-- cart/src/main/resources/db/changelog/ddl/changelog-0005.sql
ALTER TABLE cart_item ADD CONSTRAINT uk_cart_item_customer_product UNIQUE (cart_id, product_id);
-- Note: Assumes `cart` table has `customer_id` or the uniqueness is enforced at cart level
```

**Customer Service Database** — Owned by Story 1; this story reads the `wishlist_item` table via API only.

---

## REST API Contracts

### Cart Service — New Endpoint

**POST** `/api/cart/storefront/items/from-wishlist`

**Purpose:** Add a wishlist item to the cart. The wishlist item remains on the wishlist after the operation (AC-3).

**Authentication:** Required (JWT, Spring Security `@PreAuthorize("isAuthenticated()")`)

**Request Body:**
```json
{
  "wishlistItemId": 1
}
```

**Response:**
- **200 OK** — Item added to cart successfully
  ```json
  {
    "id": 123,
    "productId": 456,
    "quantity": 1
  }
  ```
- **400 Bad Request** — Product is discontinued (AC-4)
  ```json
  {
    "statusCode": 400,
    "title": "Bad Request",
    "detail": "Product is no longer available and cannot be added to the cart",
    "fieldErrors": []
  }
  ```
- **401 Unauthorized** — Not signed in (JWT missing or invalid)
- **404 Not Found** — Wishlist item does not exist or does not belong to the current user
  ```json
  {
    "statusCode": 404,
    "title": "Not Found",
    "detail": "Wishlist item not found or access denied",
    "fieldErrors": []
  }
  ```

**Behavior:**
1. Extract `customer_id` from JWT (`AuthenticationUtils.getCurrentUserId()`)
2. Call customer service internal endpoint: `GET /api/customer/storefront/wishlist/items/{wishlistItemId}`
3. Validate `customerId` from response matches JWT user (403 if not)
4. If `isAvailable == false`, return 400 Bad Request (AC-4)
5. Call product service: `GET /api/product/storefront/products/{productId}` for price/details
6. Insert into `cart_item` (or update quantity if already in cart — verify existing `CartItemService` behavior)
7. Wishlist item remains unchanged (customer service does not delete it)

**Error Handling:**
- Customer service returns 404 → return 404 to client
- Product service returns 404 or `isAvailable == false` → return 400 "Product no longer available"
- Duplicate cart item (if unique constraint exists) → catch constraint violation, return 200 (idempotent)

---

### Customer Service — New Internal Endpoint

**GET** `/api/customer/storefront/wishlist/items/{wishlistItemId}`

**Purpose:** Retrieve wishlist item details for cart service (internal call only; not exposed to UI)

**Authentication:** Required (same JWT as cart service)

**Response:**
```json
{
  "wishlistItemId": 1,
  "customerId": "uuid-from-keycloak",
  "productId": 123,
  "isAvailable": true
}
```

**Behavior:**
1. SELECT from `wishlist_item` WHERE `id = {wishlistItemId}`
2. If not found or `customer_id` does not match JWT → return 404
3. Call product service to determine `isAvailable` (see below)
4. `isAvailable = true` if product service returns 200 AND product is published AND in stock (verify exact field names in product service)
5. Return `wishlistItemId`, `customerId`, `productId`, `isAvailable`

**Assumption:** The customer service must fetch product availability from the product service. The cart service's existing `ProductService` class has two methods: `getProducts(List<Long> ids)` returning `List<ProductThumbnailVm>` and `getProductById(Long id)` returning `ProductThumbnailVm`. The customer service's new `ProductService` will follow the same pattern: create a new class `customer/src/main/java/com/yas/customer/service/ProductService.java` with methods `getProducts(List<Long> ids)` returning `List<ProductThumbnailVm>` and `getProductById(Long id)` returning `ProductThumbnailVm`. The `WishlistService.getWishlistItemById()` method will call `ProductService.getProductById(productId)` and map `ProductThumbnailVm` fields to determine `isAvailable`. The exact availability logic depends on fields in `ProductThumbnailVm` (verify in product service code or Story 1 documentation).

**Security:** This endpoint is for authenticated service-to-service calls. The cart service calls it with the user's JWT; customer service validates the JWT and checks ownership.

---

### Product Service — Existing Endpoint (No Changes)

**GET** `/api/product/storefront/products/{id}`

**Purpose:** Fetch product details and availability (existing endpoint used by cart service)

**Used By:** Cart service (existing) and customer service (Story 1 and this story)

**Response Fields Needed:** Fields from `ProductThumbnailVm` (verify exact names in `product` service code). The cart service's `ProductService.getProducts()` returns `List<ProductThumbnailVm>`, so the same view model is used.

**No changes required for this story.**

---

## File Touch List

### Cart Service (`cart/`)

#### New Files
- `src/main/java/com/yas/cart/viewmodel/WishlistItemToCartVm.java` — Request ViewModel for add-from-wishlist endpoint
  ```java
  public record WishlistItemToCartVm(Long wishlistItemId) {}
  ```

- `src/main/java/com/yas/cart/viewmodel/WishlistItemDetailVm.java` — Response ViewModel from customer service (internal)
  ```java
  public record WishlistItemDetailVm(
      Long wishlistItemId,
      String customerId,
      Long productId,
      Boolean isAvailable
  ) {}
  ```

- `src/main/java/com/yas/cart/service/WishlistService.java` — New service for calling customer service wishlist API
  ```java
  @Service
  public class WishlistService {
      private final RestClient restClient;
      private final ServiceUrlConfig serviceUrlConfig;
      // getWishlistItemById(Long wishlistItemId) -> WishlistItemDetailVm
  }
  ```

#### Modified Files
- `src/main/java/com/yas/cart/controller/CartItemController.java`
  - Add endpoint: `POST /storefront/items/from-wishlist`
  - Method: `addFromWishlist(@RequestBody WishlistItemToCartVm vm)`
  - Delegates to `CartItemService.addFromWishlist(vm)`

- `src/main/java/com/yas/cart/service/CartItemService.java`
  - Add method: `CartItemGetVm addFromWishlist(WishlistItemToCartVm vm)`
  - Calls `WishlistService.getWishlistItemById()`
  - Validates `customerId` matches JWT user
  - Calls existing `ProductService.getProductById()` method
  - Checks `isAvailable` (if false → throw `BadRequestException`)
  - Calls existing `addCartItem()` or `updateCartItem()` method (depending on whether product already in cart)

- `src/main/java/com/yas/cart/config/ServiceUrlConfig.java` (verify in `cart/src/main/resources/application.properties`)
  - Add field: `private String customer;` (if not already present)
  - Add getter: `public String customer() { return customer; }`

- `src/main/resources/application.properties`
  - Add (if missing): `yas.services.customer=http://api.yas.local/customer`

- `src/main/resources/db/changelog/db.changelog-master.yaml` (if unique constraint migration is needed)
  - Include: `ddl/changelog-0005.sql`

- `src/main/resources/db/changelog/ddl/changelog-0005.sql` (if cart_item lacks unique constraint)
  ```sql
  --liquibase formatted sql
  --changeset story-3:add-cart-item-unique-constraint
  ALTER TABLE cart_item ADD CONSTRAINT uk_cart_item_customer_product UNIQUE (customer_id, product_id);
  ```
  **(Action: Verify existing schema first. Skip this file if constraint already exists. The existing `cart_item` table uses composite key `(customer_id, product_id)` per `CartItemId.java`, so uniqueness is already enforced. No new migration needed.)**

#### Test Files (New)
- `src/test/java/com/yas/cart/service/WishlistServiceTest.java` — Unit test for `WishlistService`
- `src/it/java/com/yas/cart/service/WishlistServiceIT.java` — Integration test for customer service call

#### Test Files (Modified)
- `src/test/java/com/yas/cart/controller/CartItemControllerTest.java` — Add test for `POST /storefront/items/from-wishlist`
- `src/test/java/com/yas/cart/service/CartItemServiceTest.java` — Add test for `addFromWishlist()` method

---

### Customer Service (`customer/`)

#### New Files
- `src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemInternalVm.java` — Response ViewModel for internal wishlist item endpoint
  ```java
  public record WishlistItemInternalVm(
      Long wishlistItemId,
      String customerId,
      Long productId,
      Boolean isAvailable
  ) {}
  ```

- `src/main/java/com/yas/customer/service/ProductService.java` — New service for calling product service API
  ```java
  @Service
  public class ProductService {
      private final RestClient restClient;
      private final ServiceUrlConfig serviceUrlConfig;
      // getProductById(Long productId) -> ProductThumbnailVm
      // getProducts(List<Long> ids) -> List<ProductThumbnailVm>
  }
  ```
  **(Pattern: Follow `cart/src/main/java/com/yas/cart/service/ProductService.java`. Use `ProductThumbnailVm` from common library or customer service viewmodel package.)**

- `src/main/java/com/yas/customer/viewmodel/product/ProductThumbnailVm.java` (if not in common library)
  ```java
  public record ProductThumbnailVm(
      Long id,
      String name,
      String slug,
      // Add fields from product service response as needed
  ) {}
  ```
  **(Action: Check if `ProductThumbnailVm` exists in `common-library` or copy from cart service pattern.)**

#### Modified Files
- `src/main/java/com/yas/customer/controller/UserAddressController.java` → **OR** create new `WishlistController.java`
  - Add endpoint: `GET /storefront/wishlist/items/{id}`
  - Method: `getWishlistItemById(@PathVariable Long id)`
  - Delegates to `WishlistService.getWishlistItemById(id)` (from Story 1)
  - Returns `WishlistItemInternalVm`

- `src/main/java/com/yas/customer/service/WishlistService.java` (created in Story 1)
  - Add method: `WishlistItemInternalVm getWishlistItemById(Long id)`
  - SELECT from `wishlist_item` WHERE `id = {id}` AND `customer_id = current user`
  - If not found → throw `NotFoundException`
  - Call `ProductService.getProductById(productId)` to fetch product details
  - Determine `isAvailable` from `ProductThumbnailVm` fields (verify exact logic in product service documentation)
  - Return `WishlistItemInternalVm`

- `src/main/java/com/yas/customer/config/ServiceUrlConfig.java` (verify in `customer/src/main/resources/application.properties`)
  - Add field: `private String product;` (if not already present)
  - Add getter: `public String product() { return product; }`

- `src/main/resources/application.properties`
  - Add (if missing): `yas.services.product=http://api.yas.local/product`

#### Test Files (New)
- `src/test/java/com/yas/customer/controller/WishlistControllerTest.java` — Unit test for `GET /storefront/wishlist/items/{id}`
- `src/test/java/com/yas/customer/service/ProductServiceTest.java` — Unit test for `ProductService`
- `src/it/java/com/yas/customer/service/ProductServiceIT.java` — Integration test for product service call
- `src/it/java/com/yas/customer/service/WishlistServiceIT.java` (if not created in Story 1) — Integration test

---

### Product Service (`product/`)

**No changes.** This story uses the existing storefront product detail endpoint.

**Action Required:** Verify the exact response field names in product service code. The cart service's `ProductService.getProducts()` returns `List<ProductThumbnailVm>`, so the same view model is used. Document the fields available in `ProductThumbnailVm` (e.g., `id`, `name`, `slug`, `price`, availability indicators).

---

## Risks

### RISK-1: Cart Item Uniqueness Constraint Missing
**Description:** If the `cart_item` table does not enforce UNIQUE(`customer_id`, `product_id`), adding the same product from the wishlist twice could create duplicate cart rows.

**Mitigation:**
- **Verification:** The `cart_item` entity uses composite key `(customer_id, product_id)` per `cart/src/main/java/com/yas/cart/model/CartItemId.java`, so uniqueness is already enforced at the JPA level. No new migration needed.
- **Fallback:** `CartItemService.addFromWishlist()` calls existing `addCartItem()` method, which already handles duplicate product IDs by incrementing quantity (see `CartItemService.performAddCartItem()` in `cart/src/main/java/com/yas/cart/service/CartItemService.java`)

**Owner:** This story (Story 3)

---

### RISK-2: Customer Service Product Availability Check Latency
**Description:** The customer service internal endpoint calls the product service to determine `isAvailable`. If the product service is slow, the cart add-from-wishlist operation may time out.

**Mitigation:**
- Set RestClient timeout (e.g., 2s) in `customer/src/main/java/com/yas/customer/config/RestClientConfig.java`
- If product service call fails (timeout or 5xx), return `isAvailable: false` with a generic "Unable to verify availability" message

**Owner:** This story (Story 3)

---

### RISK-3: Wishlist Item Ownership Validation Gap
**Description:** If the customer service internal endpoint does not validate that the `wishlistItemId` belongs to the current user, a malicious user could add another user's wishlist items to their cart.

**Mitigation:**
- **Mandatory:** Customer service `GET /storefront/wishlist/items/{id}` MUST check `customer_id = current JWT user` before returning the item
- If ownership check fails, return 404 (not 403, to avoid leaking existence)

**Owner:** This story (Story 3)

---

### RISK-4: Duplicate Add-to-Cart Requests (Race Condition)
**Description:** If two tabs add the same wishlist item to the cart simultaneously, the cart might have duplicate entries (if no unique constraint) or fail unexpectedly.

**Mitigation:**
- Composite key on `cart_item` enforces uniqueness (see RISK-1)
- Existing `CartItemService.addCartItem()` handles duplicate product IDs by incrementing quantity (idempotent behavior)

**Owner:** This story (Story 3)

---

## Open Questions

1. **Cart Item Uniqueness Constraint:** Does `cart_item` already enforce UNIQUE(`customer_id`, `product_id`)? Check `cart/src/main/resources/db/changelog/ddl/changelog-0001.sql` and `cart/src/main/java/com/yas/cart/model/CartItemId.java`.
   - **Resolution:** Verified. The `CartItem` entity uses `@IdClass(CartItemId.class)` with composite key `(customerId, productId)`, which enforces uniqueness. No new migration needed.

2. **Product Service Response Fields:** What are the exact field names for product availability in `ProductThumbnailVm`?
   - **Resolution Path:** Inspect product service code or cart service's `ProductService.getProducts()` return type. Document fields available in `ProductThumbnailVm` and update `customer/src/main/java/com/yas/customer/service/WishlistService.java` logic to match actual field names.

3. **Customer Service ProductService Existence:** Does `customer/src/main/java/com/yas/customer/service/ProductService.java` already exist?
   - **Resolution:** Verified in repo-outline. The customer service does not have a `ProductService` class. This story must create it, following the pattern in `cart/src/main/java/com/yas/cart/service/ProductService.java`.

4. **Cart Service Existing Behavior on Duplicate Product:** When `CartItemService.addCartItem()` is called for a product already in the cart, does it increment quantity or fail?
   - **Resolution:** Verified in `cart/src/main/java/com/yas/cart/service/CartItemService.java`. The `performAddCartItem()` method checks for existing cart items and increments quantity if found. The `addFromWishlist()` method will call `addCartItem()`, which already handles this case.

---

## Notes

- **Idempotency:** Adding the same wishlist item to the cart multiple times is idempotent. The existing `CartItemService.addCartItem()` method increments quantity if the product is already in the cart. The endpoint returns 200 in both cases.

- **Wishlist Item Persistence:** The wishlist item is **not** deleted after being added to the cart (AC-3). The customer service does not modify the `wishlist_item` table in this operation.

- **Error Response Format:** Use `ErrorVm` from `common-library/src/main/java/com/yas/commonlibrary/viewmodel/error/ErrorVm.java` for all error responses (matches existing pattern across services).

- **Logging:** Use SLF4J (existing pattern). Log wishlist-to-cart operations at INFO level, errors at ERROR.

- **Security:** All endpoints require authentication (`@PreAuthorize("isAuthenticated()")`). Extract `customer_id` from JWT using `AuthenticationUtils.extractUserId()` (from `common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java`).

- **Testing:** Integration tests in `cart/src/it/` and `customer/src/it/` must mock the inter-service calls (product service, customer service) to avoid external dependencies. Unit tests use Mockito (see existing patterns in `cart/src/test/java/com/yas/cart/service/CartItemServiceTest.java`).

---

## Dependencies

- **Story 1 (Wishlist persistence and API)** must be complete before this story can be implemented. This story depends on:
  - `wishlist_item` table in customer service database
  - `WishlistService` in customer service (creates/reads/deletes wishlist items)
  - Wishlist item model and repository classes

- **Product Service** must have a storefront endpoint that returns product availability fields. The cart service's existing `ProductService` class uses `GET /api/product/storefront/products/list-featured?productId={ids}` to fetch products. Verify the endpoint exists and document the exact response schema (`ProductThumbnailVm` fields).

---

# Story summaries

- `01-app`: Low-Level Design — Story 01-app
- `02-app`: Low-Level Design — Story 02-app
- `03-app`: Low-Level Design — Story 03-app

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
