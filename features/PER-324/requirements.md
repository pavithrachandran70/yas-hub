# Requirements — PER-324

**Creating a rating accepts a star value below 1 or above 5**

## Context

`RatingPostVm` currently has no bounds on the `star` field, and `RatingService.createRating` stores whatever integer is sent. A shopper who bought the product can save a rating of 0 or 99, and that value is included in the product average.

This ticket adds backend validation to enforce the 1–5 star range.

## Acceptance Criteria

### AC-1: Accept valid star values (1–5 inclusive)

**Given** a shopper submits a rating for a product they purchased  
**When** the `star` field is 1  
**Then** the rating is accepted and saved to the database

---

**Given** a shopper submits a rating for a product they purchased  
**When** the `star` field is 5  
**Then** the rating is accepted and saved to the database

---

**Given** a shopper submits a rating for a product they purchased  
**When** the `star` field is 2, 3, or 4  
**Then** the rating is accepted and saved to the database

### AC-2: Reject star value of 0

**Given** a shopper submits a rating for a product they purchased  
**When** the `star` field is 0  
**Then** the system returns HTTP 400 Bad Request with a validation error message  
**And** the rating is not saved to the database

### AC-3: Reject negative star values

**Given** a shopper submits a rating for a product they purchased  
**When** the `star` field is -1 or any negative integer  
**Then** the system returns HTTP 400 Bad Request with a validation error message  
**And** the rating is not saved to the database

### AC-4: Reject star values above 5

**Given** a shopper submits a rating for a product they purchased  
**When** the `star` field is 6, 9, 99, or any integer greater than 5  
**Then** the system returns HTTP 400 Bad Request with a validation error message  
**And** the rating is not saved to the database

### AC-5: Validation is enforced on the backend

**Given** the frontend submits an invalid star value (bypassing any client-side validation)  
**When** `RatingService.createRating` processes the request  
**Then** the backend validation rejects the invalid value  
**And** returns HTTP 400 Bad Request

### AC-6: Only valid ratings are included in product average

**Given** valid ratings with star values between 1 and 5 exist for a product  
**When** the product's average rating is calculated  
**Then** only those valid ratings are included in the average  
**And** no invalid ratings (0, negative, >5) are included

## Verification

1. Submit a rating with `star: 1` → verify 200 OK and database record created
2. Submit a rating with `star: 5` → verify 200 OK and database record created
3. Submit a rating with `star: 0` → verify 400 Bad Request and no database record
4. Submit a rating with `star: -1` → verify 400 Bad Request and no database record
5. Submit a rating with `star: 6` → verify 400 Bad Request and no database record
6. Submit a rating with `star: 99` → verify 400 Bad Request and no database record
7. Query the product average after submitting valid and invalid ratings → verify only valid ratings (1–5) are included

## Notes

- The validation must be added to `RatingPostVm` (view model validation layer) and enforced in `RatingService.createRating` before database write.
- The error response format should follow the existing application convention for validation errors (HTTP 400 Bad Request).
- Edge case: `star: 1` and `star: 5` are valid boundary values and must be explicitly tested.
