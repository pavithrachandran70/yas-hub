## What would go wrong

### C5
- file: stories/02-app.md
- section: ## Implementation Code › New Files › 2. ReviewService.ts
- quote: const getAuthToken = (): string => { // Placeholder - implement based on storefront's auth pattern return ''; // Return actual token from session/cookie/localStorage };
- match: backoffice/common/services/ApiClientService.ts
- severity: blocking
- blocks because: Dev would implement `getAuthToken()` to return an empty string, so every API call would fail with 401 Unauthorized and no reviews could be submitted or retrieved.
- problem: The `getAuthToken()` function is a placeholder returning an empty string. Dev would leave it as-is, expecting the storefront to magically authenticate requests. The backoffice uses a Next.js BFF pattern where authentication is handled server-side (via cookies managed by `backoffice-bff`), so the client-side `fetch` does not explicitly send tokens. The storefront likely uses the same pattern, but Story 2's code instructs Dev to add `'Authorization': \`Bearer ${getAuthToken()}\`` headers, which contradicts the BFF architecture.
- change: Remove the `Authorization` header from `ReviewService.ts` fetch calls and add a note: "The storefront uses a Next.js BFF pattern (see `storefront-bff` or similar). Authentication is handled server-side via session cookies; the client does not send explicit tokens. If the storefront deviates from this pattern, verify the authentication mechanism before implementation." Update the fetch calls to remove the `Authorization` header line.

### C7
- file: stories/03-app.md
- section: ## Implementation Code › New Files › 1. ReviewsList.tsx
- quote: async function getAuthenticatedUser() {
- match: backoffice/common/components/AuthenticationInfo.tsx › export default function AuthenticationInfo()
- severity: blocking
- blocks because: Dev would copy the `ReviewsList.tsx` code as written, which uses `fetch` with `Authorization` headers. The storefront uses a Next.js BFF pattern (same as backoffice), so authentication is handled server-side via session cookies. The client does not send explicit tokens, and Dev would implement a broken `getAuthToken()` function (see C5), causing all review list fetches to fail with 401.
- problem: Story 3's `ReviewsList.tsx` code instructs Dev to call `getReviews(productId, pageNo, pageSize)`, which calls `ReviewService.getReviews` from Story 2. That service adds `Authorization` headers using `getAuthToken()`, which is a placeholder (see C5). Dev would implement the reviews list component with broken authentication, so no reviews would ever load.
- change: Update Story 3's `ReviewsList.tsx` to remove any authentication logic (it should simply call `getReviews` from `ReviewService`, which Story 2 must fix per C5). Add a note in Story 3's Implementation Notes: "Authentication is handled by the Next.js BFF pattern (see Story 2's `ReviewService.ts` fix). The `ReviewsList` component does not handle authentication; it relies on the service layer."

## Notes for the gate

### C4 (from last review, still relevant)
Story 1's LLD lists `Rating.java` modifications (adding `createdBy` and `lastModifiedBy` fields) in the Modified Files section, but the Implementation Pointers section does not explicitly instruct Dev to verify whether these fields already exist before adding them. The LLD says "verify existing schema matches spec; no changes expected unless fields are missing", but the Implementation Code section shows the full field definitions as if Dev should add them unconditionally. This could cause Dev to add duplicate fields if they already exist in the code today. The gate should check whether `rating/src/main/java/com/yas/rating/model/Rating.java` already has `createdBy` and `lastModifiedBy` fields, and if so, update the LLD to remove those field definitions from the Implementation Code section (or mark them as "only if missing").

### C6 (from last review, still relevant)
Story 3's LLD modifies `storefront/pages/products/[id].tsx` and `storefront/modules/catalog/components/ProductDetailHeader.tsx`, but the `repo-tree/app` shows no `storefront` directory. The breakdown already notes this in Story 2's Scope ("The storefront appears to be a separate Next.js application..."), but Story 3's breakdown does not carry this forward. The gate should verify whether the storefront repo is separate before Dev attempts to implement Story 3. If it is separate, the file paths in Story 3's File Plan and Implementation Code sections are placeholders and must be updated once the storefront repo structure is known.

### C8 (from last review, still relevant)
Story 3's `ReviewsList.tsx` uses `date.toLocaleDateString('en-US')` to format review timestamps. If the storefront already has a date formatting utility (like the backoffice's moment-based approach in `ExportProduct.js`), Dev should use that instead of reinventing it. The gate should check whether the storefront has a consistent date formatting pattern and, if so, update Story 3's Implementation Code to use it.


### C9
- file: breakdown.md
- quote: (missing)
- severity: blocking
- blocks because: Sub-issues are created from breakdown.md; with these gaps the tracker tickets miss their repo, sections or order.
- problem: story numbers must run 1..5 in order, found [1, 1, 2, 2, 3]; Story 1 (Backend review submission endpoint with validation and rating recalculation): missing or empty sections: Scope, Implementation Pointers, Acceptance Criteria, Dependencies, Out of Scope, Verification Approach; Story 1 (Backend review submission endpoint with validation and rating recalculation › Scope): missing the **Repo:** line; Story 1 (Backend review submission endpoint with validation and rating recalculation › Scope): missing or empty sections: Context, Scope; Story 2 (Frontend review form component with star selector and content input): missing or empty sections: Scope, Implementation Pointers, Acceptance Criteria, Dependencies, Out of Scope, Verification Approach; Story 2 (Frontend review form component with star selector and content input › Scope): missing the **Repo:** line; Story 2 (Frontend review form component with star selector and content input › Scope): missing or empty sections: Context, Scope.
- change: Rewrite every story to follow this format:
Each story becomes a tracker sub-issue, read by someone who has not seen requirements.md or the HLD.
Use exactly this layout for EVERY story (N starts at 1, no gaps):

### Story N: <what exists or works when this story is done>

**Repo:** `<repo-key>`

#### Context
Why this story exists and what it enables, 1-3 sentences.

#### Scope
Bullets: the concrete files this story may change, from repo-tree / repo-context (mark new files as new).

#### Implementation Pointers
Bullets: the exact functions, classes, routes, fields or GraphQL types to add or change, with file
(and line when known).

#### Acceptance Criteria
A numbered list. Each item is a complete, checkable statement of behavior (Given/When/Then or one
sentence), copied or adapted ONLY from requirements.md / the ticket. An AC id may follow in
parentheses, but never write an id alone.

#### Dependencies
`None` when this story is foundational; otherwise one bullet per earlier story it needs:
`Story K — <why>`. Only earlier stories may be named.

#### Out of Scope
What this story deliberately does not do, naming the story that does it when there is one.

#### Verification Approach
The exact test count and the tests; later LLD/tests must match this count.

Epic-level sections (cross-story dependencies, risks, open questions) go after the last story as `##`
headings.

VERDICT: FAIL

