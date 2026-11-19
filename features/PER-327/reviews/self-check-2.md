## What would go wrong

### C1
- file: breakdown.md
- section: ### Story 1: Backend review submission endpoint with validation and rating recalculation › Scope
- quote: `rating/src/main/java/com/yas/rating/controller/StorefrontReviewController.java` (new controller for storefront review endpoints)
- match: repo-outline/app › rating/src/main/java/com/yas/rating/controller/RatingController.java
- severity: blocking
- blocks because: Dev would create a new `StorefrontReviewController.java` when the repo already has `RatingController.java`. Before writing the new file, Dev must check whether `RatingController` is the backoffice controller and can be extended with storefront endpoints, or whether a separate storefront controller is needed. The breakdown does not show this check, so Dev would proceed with the new file without verifying it.
- problem: The breakdown lists a new file `StorefrontReviewController.java` but the repo outline shows an existing `RatingController.java`. The breakdown must state whether this is a backoffice controller (meaning the new storefront controller is correct) or a shared controller (meaning the breakdown should modify it instead).
- change: Add a note in the Scope: "Verify that `rating/src/main/java/com/yas/rating/controller/RatingController.java` is a backoffice-only controller before creating `StorefrontReviewController.java`. If `RatingController` already serves storefront endpoints, modify it instead of creating a new file."

---

### C2
- file: breakdown.md
- section: ### Story 1: Backend review submission endpoint with validation and rating recalculation › Scope
- quote: `product/src/main/java/com/yas/product/controller/ProductAggregateController.java` (new backoffice controller for aggregate updates — **provisional, pending resolution of HLD Open Question 3**)
- match: breakdown.md › ## Risks
- severity: blocking
- blocks because: The breakdown marks this controller as "provisional" but Story 1's AC list includes AC-7 (average rating recalculation) and AC-8 (review count increment), which require this endpoint to work. Dev would build Story 1 with a provisional controller, meaning the aggregate update either fails or is skipped, and the ACs cannot pass. The breakdown must either confirm the controller design as final or block Story 1 until the question is resolved.
- problem: The controller is marked provisional in the Scope but required by AC-7 and AC-8 in the same story's Acceptance Criteria. The "decided" note in `critic-items.md` says this is the final design, but the breakdown still calls it provisional.
- change: Remove "— **provisional, pending resolution of HLD Open Question 3**" from the Scope. The controller design is final per the decided note in `critic-items.md`.

---

### C3
- file: breakdown.md
- section: ### Story 2: Frontend review form component with star selector and content input › Scope
- quote: `storefront/modules/review/components/ReviewForm.tsx` (form component: star selector, content textarea, submit button, validation, error/success messages)
- match: repo-tree/app
- severity: blocking
- blocks because: The repo tree shows no `storefront` directory under `app`. Dev would attempt to create `storefront/modules/review/components/ReviewForm.tsx` and fail because the path does not exist. The breakdown must state where the storefront files belong: in a separate repo (not provided), under a different path in `app` (such as `storefront-ui`), or somewhere else.
- problem: The breakdown lists `storefront/modules/review/components/ReviewForm.tsx` as a new file, but `repo-tree/app` shows no `storefront` directory. The breakdown acknowledges this in the Modified Files note but does not resolve it in the Scope, so Dev would not know where to create the file.
- change: Add to the Scope: "The storefront appears to be a separate Next.js application not included in the `app` repo provided in the context. Story 2 and Story 3 belong in the **storefront repository**, which must be provided separately. The file paths will be determined once the storefront repo is available (likely `pages/products/[id].tsx` or `app/products/[id]/page.tsx` depending on Next.js version)."

---

### C4
- file: stories/01-app.md
- section: Story 01-app LLD: Backend review submission endpoint with validation and rating recalculation › File Plan › Modified Files
- quote: `rating/src/main/java/com/yas/rating/model/Rating.java` (verify existing schema matches spec; no changes expected unless fields are missing)
- match: repo-outline/app
- severity: blocking
- blocks because: The LLD says "verify existing schema matches spec; no changes expected unless fields are missing" but does not show what the existing schema is. The repo outline lists `rating/src/main/java/com/yas/rating/model/Rating.java` but does not show its fields. Dev would have to guess whether `createdBy` and `lastModifiedBy` fields exist, and if they do not, whether to add them or leave them out. The LLD must either confirm the fields exist (by quoting from `repo-file/app/rating/src/main/java/com/yas/rating/model/Rating.java` if available) or state that Dev must check and add them if missing.
- problem: The LLD says "verify existing schema matches spec" but provides no way to verify because the repo outline does not show the `Rating` entity's fields, and no `repo-file` for it is provided.
- change: Add a verification step in the Implementation Pointers: "Before implementing `RatingService.createReview`, read `rating/src/main/java/com/yas/rating/model/Rating.java` to confirm it has these fields: `id`, `productId`, `customerId`, `star`, `content`, `createdOn`, `lastModifiedOn`, `createdBy`, `lastModifiedBy`. If `createdBy` or `lastModifiedBy` are missing, add them as shown in the 'Modified Files' section below."

---

### C5
- file: stories/02-app.md
- section: Story 02-app LLD: Frontend review form component with star selector and content input › File Plan › Modified Files
- quote: **Note:** The `repo-tree/app` shows no `storefront` or `storefront-ui` directory. The storefront appears to be a separate Next.js application not included in the `app` repo provided in the context. This story's implementation belongs in the **storefront repository**, which must be provided separately.
- match: breakdown.md › ### Story 2: Frontend review form component with star selector and content input › Scope
- severity: blocking
- blocks because: Story 2's LLD says the storefront is a separate repo not provided, but the breakdown's Scope still lists `storefront/modules/review/components/ReviewForm.tsx` as a new file without noting that the repo is missing. Dev reading the breakdown would start Story 2 and fail because the path does not exist. The breakdown must state that Story 2 is blocked until the storefront repo is provided.
- problem: The LLD correctly notes that the storefront repo is missing, but the breakdown does not reflect this in the Scope or Dependencies, so Dev would not know Story 2 is blocked.
- change: Add to Story 2's Scope in the breakdown: "**Blocked:** This story requires the storefront repository, which is not included in the `app` repo. The file paths and integration points will be determined once the storefront repo is available."

---

### C6
- file: stories/03-app.md
- section: Story 03-app LLD: Reviews section display updates › File Plan › Modified Files
- quote: `storefront/pages/products/[id].tsx` (or exact path — verify in repo)
- match: breakdown.md › ### Story 3: Reviews section display updates › Scope
- severity: blocking
- blocks because: Story 3 lists `storefront/pages/products/[id].tsx` as a modified file, but the repo tree shows no `storefront` directory, and Story 2's LLD notes that the storefront is a separate repo not provided. Dev would start Story 3 and fail for the same reason as Story 2. Story 3 must also be blocked until the storefront repo is provided.
- problem: Story 3 lists storefront file paths but does not note that the storefront repo is missing, so Dev would not know the story is blocked.
- change: Add to Story 3's Scope in the breakdown: "**Blocked:** This story requires the storefront repository, which is not included in the `app` repo. The file paths and integration points will be determined once the storefront repo is available."

---

## Notes for the gate

None.

---

VERDICT: FAIL
