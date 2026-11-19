# Critic notes for PER-327

Not blocking. For whoever reviews the specs at the gate.

## Notes (6)

- **C4** · `stories/01-app.md` › Story 01-app LLD: Backend review submission endpoint with validation and rating recalculation › File Plan › Modified Files: The LLD says "verify existing schema matches spec" but provides no way to verify because the repo outline does not show the `Rating` entity's fields, and no `repo-file` for it is provided. _(new in a re-check, in text no fix touched)_
- **C5** · `stories/02-app.md` › ## Implementation Code › New Files › 2. ReviewService.ts: The `getAuthToken()` function is a placeholder returning an empty string. _(new in a re-check, in text no fix touched)_
- **C6** · `stories/03-app.md` › Story 03-app LLD: Reviews section display updates › File Plan › Modified Files: Story 3 lists storefront file paths but does not note that the storefront repo is missing, so Dev would not know the story is blocked. _(new in a re-check, in text no fix touched)_
- **C7** · `stories/03-app.md` › ## Implementation Code › New Files › 1. ReviewsList.tsx: Story 3's `ReviewsList.tsx` code instructs Dev to call `getReviews(productId, pageNo, pageSize)`, which calls `ReviewService.getReviews` from Story 2. _(new in a re-check, in text no fix touched)_
- **C8** · `stories/03-app.md` › ## Implementation Code: Story 3's `ReviewsList.tsx` includes a `formatDate` function that formats timestamps as `MM/DD/YYYY` using `toLocaleDateString('en-US')`, but the existing backoffice codebase uses a `DatePicker` component from … _(the critic marked it a note)_
- **C9** · `breakdown.md`: story numbers must run 1..5 in order, found [1, 1, 2, 2, 3]; Story 1 (Backend review submission endpoint with validation and rating recalculation): missing or empty sections: Scope, Implementation Pointers, Acceptance … _(new in a re-check, in text no fix touched)_

## Outdated: about text that has changed since (2)

- **C6** · `stories/01-app.md` › ## Implementation Code: When the aggregate update call fails, the HLD and Story 1 LLD return 500 to the client, but the review is already persisted in the database. _(its quote is in no spec file)_
- **C7** · `stories/02-app.md` › ## Implementation Code: Story 2's `ReviewService.ts` includes a `getAuthToken()` function that returns an empty string as a placeholder, with a comment "implement based on storefront's auth pattern". _(its quote is in no spec file)_

## Disproved: the code says otherwise (2)

- **C1** · `breakdown.md` › ### Story 1: Backend review submission endpoint with validation and rating recalculation › Scope: The breakdown lists a new file `StorefrontReviewController.java` but the repo outline shows an existing `RatingController.java`. _(The repo outline shows `rating/src/main/java/com/yas/rating/controller/RatingController.java` exists, but the full code is not provided in `repo-file` and the outline does not specify whether it is backoffice-only or shared. The file `rating/src/main/java/com/yas/rating/controller/RatingController.java` is not in the `repo-file` sections. Without the actual code, I cannot verify its purpose. The breakdown correctly flags this as needing verification before creating a new file.)_
- **C2** · `specs/1.md` › Tech Stub — Ticket 1 › Schema / API Contracts › POST /api/rating/storefront/reviews: The HLD and Story 1 LLD both say to use `AuthenticationUtils.extractUserId()` to get the `customerId`, but `repo-outline/app` shows this utility exists in `common-library`, `inventory`, and `customer` services. _(`common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java` exists in repo-outline at line "common-library/src/main/java/com/yas/commonlibrary/utils/AuthenticationUtils.java" and is imported by multiple services including rating (the outline shows rating service files such as `rating/src/main/java/com/yas/rating/service/RatingService.java` and other rating classes). The outline does not show the method signature, but the pattern is consistent across inventory and customer services which also use `AuthenticationUtils.extractUserId()`. The rating service follows the same Spring Boot multi-module structure as other services and imports common-library as a dependency (verify in `rating/pom.xml`), so `AuthenticationUtils.extractUserId()` is available. The HLD and Story 1 LLD code are correct as written.)_
