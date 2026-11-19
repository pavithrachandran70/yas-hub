# Story — PER-325

**Type:** story  
**ID:** 325  
**Title:** The product page shows one rating summary for the reviews it already loads  
**Epic:** (standalone story)  
**Status:** (set by plan)  
**Assignee:** (set by plan)

---

## Description

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

## Technical Specification

### Scope

UI-only update in the `app` repository. No API changes, no schema changes, no new storage. The product page already loads review data; this story synchronizes the header display with that data and handles the post-submission state update.

### Affected Components

Verify exact paths and component names in `app` codebase:

- **Product page component** (see LLD) — displays product details, header rating summary, and Reviews tab
- **Header rating display** (see LLD) — shows stars, average number, and count text
- **Reviews tab** (see LLD) — shows review list and tab title with count
- **Review submission handler** (see LLD) — triggered after successful review post

### Implementation Guidance

#### 1. Rating Summary Display (AC1, AC2)

**Header rating calculation:**
- Compute average from the review list already loaded by the product page
- Format average to one decimal place (e.g., `4.3`)
- Render stars based on average (verify star rendering logic in existing component)
- Render count as `"1 review"` when count is 1, `"N reviews"` when count > 1

**Zero-review case:**
- When review list is empty or count is 0:
  - Display "No reviews yet" in header
  - Do not render filled stars
  - Do not render "(0 ratings)" text
  - Display "No reviews yet" in Reviews tab

#### 2. Reviews Tab Title (AC3)

The Reviews tab title must display the same count value used in the header. Use a shared function or state variable to ensure consistency.

#### 3. Review Display (AC4, AC5)

**Anonymous reviews:**
- When `firstName` and `lastName` are both null/empty, display "Anonymous"
- Render stars for the review

**Named reviews:**
- Display `firstName` and `lastName` (or combined name field, verify in existing review component)
- Render stars
- Display review text
- Display relative time (e.g., "2 days ago") — verify existing time formatting logic

#### 4. Post-Submission State Update (AC6)

**After successful review submission:**
- The review submission handler receives the new review data (or refetches the review list)
- Update the in-memory review list with the new review (prepend or based on pagination logic)
- Recalculate average and count from the updated list
- Update header rating display (average, stars, count)
- Update Reviews tab (title count, review list)
- **Do not reload the page**
- Maintain pagination at 5 reviews per page

**State management approach:**
- Use existing state management (verify React state, Redux, or other state library in `app`)
- Trigger state update on successful review submission callback
- Re-render header and Reviews tab with updated state

#### 5. Restricted Shopper Handling (AC7)

No changes to existing message display or form clearing logic. The product page already shows a message when a shopper is not allowed to review or has already reviewed the product. The form is not cleared in these cases. This story does not modify that behavior.

### Files to Modify

Verify exact paths in `app` codebase:

- Product page component file (see LLD)
- Header rating display component file (see LLD)
- Reviews tab component file (see LLD)
- Review submission handler/callback (see LLD)

### Edge Cases and Clarifications

- **Single review:** Count displays "1 review" (singular), not "1 reviews"
- **Zero reviews:** No stars, no "(0 ratings)", just "No reviews yet" in both header and tab
- **Anonymous review:** A review with both `firstName` and `lastName` null/empty displays "Anonymous"; a review with either field populated is considered named
- **Pagination:** After submission, the review list remains paginated at 5 per page; the new review appears in the first page (verify pagination library behavior in `app`)
- **Decimal formatting:** Average is always formatted to one decimal place (e.g., `3.0`, not `3`)

### Testing

Verify each AC by:

1. **AC1:** Load product page with reviews; confirm header shows stars, average with one decimal, and count with correct singular/plural
2. **AC2:** Load product page with zero reviews; confirm header and tab show "No reviews yet", no stars, no "(0 ratings)"
3. **AC3:** Load product page; confirm Reviews tab title count matches header count
4. **AC4:** View a review with no first name and no last name; confirm "Anonymous" and stars display
5. **AC5:** View a review with a name; confirm name, stars, text, and timestamp display
6. **AC6:** Submit a review successfully; confirm header average, header count, and Reviews tab all update without reload; confirm reviews stay paginated at 5
7. **AC7:** Attempt to submit as a shopper who already reviewed or is not allowed; confirm existing message is shown and form is not cleared

---

## Notes

- **Data source:** Average and review list already loaded by the product page today
- **Services unchanged:** Rating service, product service, order service remain as they are
- **No API changes:** No new endpoint, query parameter, response field, or setting
- **Pagination:** Reviews remain in pages of 5 after submission
- **Review form state:** Form is not cleared when shopper cannot submit (already reviewed or not allowed)

---

## References

- **Epic:** (none, standalone story)
- **Requirements:** requirements.md
- **Plan:** PER-325-plan.md
