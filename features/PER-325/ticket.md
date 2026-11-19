---
key: PER-325
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: The product page shows one rating summary for the reviews it already loads
---

Description:

On a product page, the rating in the header and the Reviews tab must describe the same reviews. Both already come from the average and the review list the page loads today. Nothing new is stored, and the rating, product, and order services stay as they are. No new endpoint, query parameter, response field, or setting.

* The header shows the average as stars plus a number with one decimal, and the count as "1 review" or "N reviews".
* When the product has no reviews, the header and the Reviews tab both say "No reviews yet". The header does not show a filled star or "(0 ratings)".
* The Reviews tab title uses that same count.
* A review with no first name and no last name still shows "Anonymous" and that review's stars.
* A review that has a name still shows the name, the stars, the text, and how long ago it was posted.
* After a shopper posts a review successfully, the header average, the header count, and the Reviews tab all update without a reload. Reviews stay in pages of 5.
* A shopper who is not allowed to review, or who has already reviewed this product, still sees the message the page shows today. The form is not cleared in those cases.
