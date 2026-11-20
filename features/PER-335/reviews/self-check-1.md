## What would go wrong

### C1
- file: specs/1.md
- section: ## Schema/API Contracts
- quote: `customer_id`: user ID from Keycloak (UUID string), extracted from JWT token via `AuthenticationUtils.extractUserId()` (verify in `common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java`)
- match: repo-outline/app › common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java
- severity: blocking
- blocks because: Dev would implement `AuthenticationUtils.extractUserId()` to extract the user ID, but this method does not exist in the codebase. The actual method is `AuthenticationUtils.getCurrentUserId()` (shown in the outline), so the code would not compile.
- problem: The spec names a method `extractUserId()` that does not exist. The outline shows `getCurrentUserId()` as the actual method in `AuthenticationUtils`.
- change: Replace all references to `AuthenticationUtils.extractUserId()` with `AuthenticationUtils.getCurrentUserId()` throughout the spec.

### C2
- file: breakdown.md
- section: ### Story 1: Backend Review Submission API and Database Schema
- quote: Verification Approach: **9 tests** (unit tests for controller and service, integration test for full flow)
- match: stories/01-app.md › ## Tests
- severity: blocking
- blocks because: The breakdown promises 9 tests but the story only plans 10 tests (6 controller tests + 3 service tests + 1 integration test), so Dev would write 10 tests and the count would not match the breakdown's promise of 9.
- problem: The breakdown says 9 tests, but the story's Tests section lists 10 tests (6 in ReviewControllerTest + 3 in ReviewServiceTest + 1 in ReviewControllerIT).
- change: Update the breakdown to say "**10 tests**" or consolidate the story tests to match 9.

### C3
- file: specs/1.md
- section: ## Schema/API Contracts
- quote: **New table: `review`**
- match: breakdown.md › ### Story 1: Backend Review Submission API and Database Schema › #### Scope
- severity: blocking
- blocks because: The spec defines the `review` table schema with a `CHECK` constraint `CHECK (rating >= 1 AND rating <= 5)`, but the breakdown and story do not mention this database-level constraint. Dev would create the table without it, relying only on application validation, so invalid ratings could be inserted directly into the database.
- problem: The spec's SQL schema includes a database `CHECK` constraint on the `rating` column, but the breakdown and story do not specify this. The story only mentions application-level validation (`@Min(1)`, `@Max(5)` annotations), not the database constraint.
- change: Add to the story's Liquibase changeset section: "Include a `CHECK` constraint on the `rating` column: `CHECK (rating >= 1 AND rating <= 5)`."

### C4
- file: stories/01-app.md
- section: ## File Plan
- quote: `rating/src/main/java/com/yas/rating/model/Review.java` — JPA entity mapping `review` table
- match: specs/1.md › ## Schema/API Contracts › **Entity:** `Review.java`
- severity: blocking
- blocks because: The spec says the `Review` entity "Extends `com.yas.commonlibrary.model.AbstractAuditEntity` for audit fields", but the story does not mention this. Dev would create a plain entity without the audit fields (`createdAt`, `updatedAt`), so the entity would not match the spec's design and the timestamps would be missing.
- problem: The spec specifies that `Review` extends `AbstractAuditEntity` (from common-library), but the story's entity snippet does not include this inheritance. The story shows `Review` with `createdAt` and `updatedAt` fields defined directly, not inherited.
- change: In the story's `Review.java` snippet, change the class declaration to `public class Review extends AbstractAuditEntity` and remove the `createdAt` and `updatedAt` field declarations (they are inherited).

### C5
- file: stories/01-app.md
- section: ## Tests
- quote: **9 tests** (unit tests for controller and service, integration test for full flow)
- match: stories/01-app.md › ## Tests
- severity: blocking
- blocks because: The story's coverage description says "9 tests" but the test list that follows contains 10 tests (numbered 1-9 for the first two test files, plus 1 for the integration test file), so the count is inconsistent with the description.
- problem: The test coverage description says "**9 tests**" but the story actually specifies 10 tests (6 in ReviewControllerTest, 3 in ReviewServiceTest, 1 in ReviewControllerIT).
- change: Change the coverage description to "**10 tests**" to match the actual list.

### C6
- file: stories/02-app.md
- section: ## Tests
- quote: **Coverage:** 6 tests
- match: stories/02-app.md › ## Tests › ### Unit Tests (ReviewForm)
- severity: blocking
- blocks because: The story says "6 tests" for ReviewForm but the test list shows 6 tests for ReviewForm and then 4 more tests for ReviewList, totaling 10 tests for the story. Dev would write all 10 tests (as the story specifies them), but the coverage description of "6 tests" would be misleading.
- problem: The story's test coverage says "**Coverage:** 6 tests" but the story specifies 6 tests for ReviewForm + 4 tests for ReviewList + 3 E2E tests = 13 tests total.
- change: Update the coverage description to "**13 tests** (6 unit tests for ReviewForm, 4 unit tests for ReviewList, 3 E2E tests)".

### C7
- file: stories/01-app.md
- section: ## Interfaces › ### Service Layer Interfaces › **ReviewService**
- quote: Extracts `customerId` from JWT token using `AuthenticationUtils.extractUserId()`
- match: repo-outline/app › common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java
- severity: blocking
- blocks because: The story instructs Dev to use `AuthenticationUtils.extractUserId()`, but this method does not exist. The actual method is `getCurrentUserId()` (shown in the outline), so the code would not compile.
- problem: The story says to use `AuthenticationUtils.extractUserId()`, but the outline shows the method is `getCurrentUserId()`.
- change: Replace `AuthenticationUtils.extractUserId()` with `AuthenticationUtils.getCurrentUserId()` in the story's service interface description and code snippets.

### C8
- file: specs/1.md
- section: ## File Touch List (from repo-tree)
- quote: `rating/src/main/java/com/yas/rating/config/ServiceUrlConfig.java` — Service URL config bean for product service base URL
- match: specs/1.md › ## Schema/API Contracts › **Inter-service call:** `ProductService.java` (in `rating` service)
- severity: blocking
- blocks because: The spec says to create `ServiceUrlConfig.java` for injecting the product service URL, but the code snippet for `ProductService.java` does not show how the URL is injected. Dev would create the file but not know how to wire it, or might hardcode the URL instead.
- problem: The spec lists `ServiceUrlConfig.java` as a new file and says "injected from application.properties" in the `ProductService` snippet, but does not show the actual injection mechanism (constructor parameter, `@Value` annotation, or config bean).
- change: In the `ProductService.java` snippet, add a constructor parameter: `public ProductService(RestClient restClient, ServiceUrlConfig serviceUrlConfig)` and use `serviceUrlConfig.getProduct()` to get the URL. Add a snippet for `ServiceUrlConfig.java` showing the config bean structure (follow the pattern from `customer/src/main/java/com/yas/customer/config/ServiceUrlConfig.java` if shown in the outline).

### C9
- file: stories/01-app.md
- section: ## File Plan › ### New Files
- quote: `rating/src/main/resources/db/changelog/ddl/changelog-0001.sql` — Liquibase changeset to create `review` and `product_aggregate` tables
- match: breakdown.md › ### Story 1: Backend Review Submission API and Database Schema › #### Scope
- severity: blocking
- blocks because: The story says to create a single changeset file `changelog-0001.sql` for both tables, but the breakdown lists two separate files: `changelog-review-table.sql` and `changelog-product-aggregate-table.sql`. Dev following the story would create one file, but the breakdown expects two, so the structure would not match.
- problem: The story says to create `changelog-0001.sql` (one file for both tables), but the breakdown says to create `changelog-review-table.sql` and `changelog-product-aggregate-table.sql` (two separate files).
- change: In the story, change the file name to match the breakdown: "Create `rating/src/main/resources/db/changelog/ddl/changelog-review-table.sql` for the `review` table and `rating/src/main/resources/db/changelog/ddl/changelog-product-aggregate-table.sql` for the `product_aggregate` table."

### C10
- file: stories/02-app.md
- section: ## File Plan › ### Modified Files
- quote: `storefront/pages/products/[slug].tsx` — Product detail page
- match: repo-tree/app
- severity: blocking
- blocks because: The story says to modify `storefront/pages/products/[slug].tsx`, but this file does not exist in the repo tree. Dev would search for it, not find it, and get stuck trying to integrate the components.
- problem: The story says to modify `storefront/pages/products/[slug].tsx`, but the repo tree does not show this file (the tree stops at 1000 files and does not include any `storefront/pages/` directory).
- change: Add a note in the story: "If `storefront/pages/products/[slug].tsx` does not exist, create it following the Next.js pages pattern (dynamic `[slug]` route) and integrate the `ReviewForm` and `ReviewList` components inline."

## Notes for the gate

### N1
The backoffice UI (`backoffice/pages/reviews/index.tsx`) already exists and expects `getRatings(queryString)` and `deleteRatingById(id)` endpoints (shown in the file content). Story 3 is correctly designed to implement these backend endpoints, so the existing UI will work once Story 3 is complete. The story's API contract matches the UI's expectations (paginated list, delete by ID).

### N2
The `rating` service does not exist yet. The plan is sound: create the service following the existing pattern (observed in `cart`, `customer`, `order`, `inventory` services). The pom.xml already has a parent POM and dependency management structure that the new `rating` module will inherit from. The story correctly specifies adding `<module>rating</module>` to the root pom.xml.

### N3
The spec and story refer to `ProductService.verifyProductExists(productId)` calling the product service, but do not specify the exact endpoint. From the repo outline, the product service likely exposes `GET /api/product/storefront/products/{id}`. The story should verify this endpoint exists before implementation. If the endpoint returns 404 for non-existent products, the story's design is correct. If it throws a different exception, the story's error handling may need adjustment.

VERDICT: FAIL
