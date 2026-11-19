---
key: PER-327
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Customers can write and submit product reviews
---

## Description

Customers should be able to share their feedback about products by submitting reviews from the product detail page.

* Allow customers to submit a review for a product.
* Allow customers to select a star rating from 1 to 5.
* Require review content before submission.
* Validate that the star rating is within the allowed range.
* Display the submitted review in the Reviews section.
* Update the product's average rating and total review count after successful submission.

## Acceptance Criteria

 1. Customers must be able to select a star rating between 1 and 5.
 2. Review content must be mandatory and must not contain only whitespace.
 3. The system must prevent submission if the rating or review content is missing.
 4. The system must reject invalid ratings, including values below 1 or above 5, through API validation.
 5. Upon successful submission, the review must be saved and a confirmation message must be displayed.
 6. The newly submitted review must appear in the product's Reviews section with the correct rating and review content.
 7. The product's average rating must be recalculated based on all valid reviews.
 8. The total review count must increase by one after successful submission.
 9. The rating and review count displayed in the product header must match the Reviews section.
10. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count.
11. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing.
12. Submitted reviews must remain available after refreshing or revisiting the product page.
