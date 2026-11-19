# Requirements — PER-327

**Customers can write and submit product reviews**

## Overview

Customers visiting a product detail page can share feedback by submitting a review consisting of a star rating (1–5) and written content. The system validates input, prevents duplicate submissions during processing, persists the review, recalculates the product's aggregate rating, and displays the new review and updated statistics immediately.

## Scope

This requirement covers the review submission flow from the product detail page: the star rating selector, review content input, validation rules, submission handling, duplicate prevention, aggregate rating recalculation, and display updates. It does not cover review editing, deletion, moderation, or reporting.

## Acceptance Criteria

### AC-1: Star rating selection
**Given** a customer is viewing the product detail page  
**When** they interact with the star rating selector  
**Then** they can select a rating between 1 and 5 (inclusive)

### AC-2: Review content is mandatory
**Given** a customer is submitting a review  
**When** the review content field is empty or contains only whitespace  
**Then** the system prevents submission and displays a validation message

### AC-3: Missing input prevention
**Given** a customer attempts to submit a review  
**When** either the star rating or review content is missing  
**Then** the system prevents submission and displays a validation message

### AC-4: Invalid rating rejection via API
**Given** a review submission request is sent to the API  
**When** the rating value is below 1 or above 5  
**Then** the API rejects the request with a validation error response

### AC-5: Successful submission confirmation
**Given** a customer submits a valid review (rating 1–5, non-empty content)  
**When** the submission completes successfully  
**Then** the system saves the review and displays a confirmation message to the customer

### AC-6: Review appears in Reviews section
**Given** a customer has successfully submitted a review  
**When** the product detail page refreshes or the Reviews section updates  
**Then** the newly submitted review appears with the correct rating and review content

### AC-7: Average rating recalculation
**Given** a review has been successfully submitted  
**When** the product's aggregate statistics are recalculated  
**Then** the product's average rating reflects all valid reviews including the new one

### AC-8: Review count increment
**Given** a review has been successfully submitted  
**When** the product's aggregate statistics are updated  
**Then** the total review count increases by one

### AC-9: Rating and count consistency
**Given** the product detail page displays rating statistics  
**When** the page renders the product header and Reviews section  
**Then** the average rating and review count match in both locations

### AC-10: Error handling without incorrect updates
**Given** a review submission fails (network error, server error, validation error)  
**When** the error response is received  
**Then** an appropriate error message is displayed and the product's rating and review count remain unchanged

### AC-11: Duplicate prevention during processing
**Given** a customer clicks the Submit button  
**When** the review submission request is in progress  
**Then** repeated clicks on the Submit button do not create duplicate review records

### AC-12: Review persistence
**Given** a customer has submitted a review  
**When** they refresh the product page or navigate away and return  
**Then** the submitted review remains visible in the Reviews section with correct data

## Open Questions

- Is there a maximum length for review content?
- Should customers be allowed to submit multiple reviews for the same product, or is one review per customer per product enforced?
- What is the idempotency mechanism: request token, (customerId, productId) uniqueness constraint, or both?
- Are there any constraints on how quickly aggregate statistics must update (eventual consistency acceptable, or must be synchronous)?

## Notes

**Validation rules:**
- Rating: integer, range [1, 5]
- Review content: non-empty after trimming whitespace

**Duplicate prevention (AC-11):** The mechanism is unspecified; the requirement mandates prevention but does not prescribe client-side button disabling, server-side idempotency tokens, or database uniqueness constraints. The architect will choose the approach.

**Rating recalculation (AC-7):** The requirement mandates that the average rating reflect all valid reviews after submission but does not specify the formula (mean, weighted mean, etc.) or precision. The architect will define the calculation strategy.

**Persistence (AC-12):** Implies the review is stored durably and retrieved on subsequent page loads; the storage mechanism is an implementation detail.
