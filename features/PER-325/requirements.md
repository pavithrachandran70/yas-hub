# Requirements — PER-325

**Story:** The product page shows one rating summary for the reviews it already loads

**Epic:** (standalone story)

**Covers:** (no epic requirements)

---

## Overview

On a product page, the rating in the header and the Reviews tab must describe the same reviews. Both already come from the average and the review list the page loads today. Nothing new is stored, and the rating, product, and order services stay as they are. No new endpoint, query parameter, response field, or setting.

---

## Acceptance Criteria

### AC1: Header displays average rating and review count

**Given** a product page is loaded  
**When** the page has reviews  
**Then** the header shows:
- The average rating as stars
- The average rating as a number with one decimal place
- The count formatted as "1 review" when count is 1, or "N reviews" when count is greater than 1

### AC2: Header and Reviews tab show "No reviews yet" when product has no reviews

**Given** a product page is loaded  
**When** the product has zero reviews  
**Then**:
- The header displays "No reviews yet"
- The header does not show a filled star
- The header does not show "(0 ratings)"
- The Reviews tab displays "No reviews yet"

### AC3: Reviews tab title uses the same count as header

**Given** a product page is loaded  
**When** reviews are displayed  
**Then** the Reviews tab title shows the same count as the header

### AC4: Anonymous reviews display correctly

**Given** a product page has reviews  
**When** a review has no first name and no last name  
**Then** that review displays:
- "Anonymous" as the reviewer name
- The stars for that review

### AC5: Named reviews display correctly

**Given** a product page has reviews  
**When** a review has a name  
**Then** that review displays:
- The reviewer's name
- The stars for that review
- The review text
- How long ago it was posted

### AC6: Header and tab update after successful review submission without reload

**Given** a shopper submits a review successfully  
**When** the submission completes  
**Then**:
- The header average updates to include the new review
- The header count updates to include the new review
- The Reviews tab updates to include the new review
- No page reload occurs
- Reviews remain paginated at 5 per page

### AC7: Restricted shoppers see existing message

**Given** a shopper is not allowed to review this product, or has already reviewed it  
**When** the product page is loaded  
**Then**:
- The shopper sees the message the page shows today
- The review form is not cleared

---

## Notes

- **Scope:** UI-only update. No API changes, no schema changes, no new storage.
- **Data source:** Average and review list already loaded by the product page today.
- **Services unchanged:** Rating service, product service, order service remain as they are.
- **Pagination:** Reviews remain in pages of 5 after submission.
- **Review form state:** Form is not cleared when shopper cannot submit (already reviewed or not allowed).

---

## Open Questions

None.

---

## Verification

Verify each AC by:
1. Loading a product page with reviews and confirming header and tab display match.
2. Loading a product page with zero reviews and confirming "No reviews yet" appears in both header and tab, with no stars or "(0 ratings)".
3. Checking the Reviews tab title matches the header count.
4. Viewing a review with no name and confirming "Anonymous" and stars display.
5. Viewing a review with a name and confirming name, stars, text, and timestamp display.
6. Submitting a review successfully and confirming header average, header count, and Reviews tab all update without reload, while reviews stay paginated at 5.
7. Attempting to submit as a restricted shopper (already reviewed or not allowed) and confirming the existing message is shown and the form is not cleared.
