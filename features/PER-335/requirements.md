# requirements.md

**Ticket:** Customer Can write and Submit Product Reviews  
**Epic:** _(none)_  
**Initiative:** _(none)_

## Summary

Customers submit reviews (rating 1–5 and content) from the product detail page. The system validates input, saves the review, updates the product's average rating and review count, prevents duplicates, and displays the review.

## Requirements

### REQ-1: Rating selection

**Given** the customer is on the product detail page  
**When** they interact with the star rating control  
**Then** they can select a rating from 1 to 5 stars

---

### REQ-2: Review content mandatory

**Given** the customer is writing a review  
**When** they attempt to submit without entering review content, or with only whitespace  
**Then** the system prevents submission and displays a validation message

---

### REQ-3: Rating required

**Given** the customer is writing a review  
**When** they attempt to submit without selecting a rating  
**Then** the system prevents submission and displays a validation message

---

### REQ-4: Rating range validation (API)

**Given** the review submission reaches the API  
**When** the rating value is below 1 or above 5  
**Then** the API rejects the request with a validation error response

---

### REQ-5: Successful submission and confirmation

**Given** the customer has entered a valid rating (1–5) and non-empty review content  
**When** they submit the review  
**Then** the system saves the review and displays a confirmation message

---

### REQ-6: Review appears in Reviews section

**Given** the customer has successfully submitted a review  
**When** they view the product's Reviews section  
**Then** the newly submitted review appears with the correct rating and review content

---

### REQ-7: Product average rating recalculated

**Given** a review has been successfully submitted  
**When** the product's average rating is computed  
**Then** it reflects the average of all valid reviews for that product

---

### REQ-8: Review count incremented

**Given** a review has been successfully submitted  
**When** the total review count is displayed  
**Then** it has increased by one

---

### REQ-9: Header displays match Reviews section

**Given** the product page is loaded  
**When** the customer views the product header and the Reviews section  
**Then** the rating and review count shown in both places match

---

### REQ-10: Submission failure shows error

**Given** the review submission fails (network error, server error, or validation failure)  
**When** the error occurs  
**Then** an appropriate error message is displayed, and the product's rating and review count remain unchanged

---

### REQ-11: Duplicate submission prevented (idempotency)

**Given** the customer has clicked the Submit button  
**When** they click Submit again while the first request is still processing  
**Then** the system does not create a duplicate review

---

### REQ-12: Reviews persist across page loads

**Given** a review has been successfully submitted  
**When** the customer refreshes or revisits the product page  
**Then** the submitted review is still visible in the Reviews section

---

## Open Questions

1. Does the system require authentication to submit a review, or can anonymous customers submit reviews?
2. What is the idempotency mechanism: a request token, a database constraint on (product_id, customer_id), or another approach?
3. Where are the product's average rating and review count stored: in the product record itself, in a separate aggregates table, or computed on read?
4. Should review content have a maximum length?
5. Are reviews moderated or immediately visible after submission?

---

## Notes

- The API endpoint for review submission is not specified; planner guidance suggests `POST /products/{id}/reviews`.
- The review entity schema (product_id, customer_id, rating, content, timestamp) is assumed from planner focus but not detailed in the ticket.
- Validation occurs at both UI and API levels; API validation is mandatory per AC 4.
- Aggregate calculation strategy (average rating, review count) must be designed in HLD.
- Idempotency approach must be designed in HLD to satisfy AC 11.
