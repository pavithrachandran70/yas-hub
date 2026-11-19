# Story — PER-324

**Creating a rating accepts a star value below 1 or above 5**

## Metadata

```yaml
id: 324
type: story
epic_id: null
title: Creating a rating accepts a star value below 1 or above 5
track: quick_fix
status: planned
assignee: null
blocked_at: null
blocked_reason: null
hitl: false
covers: []
after: []
```

## Description

`RatingPostVm` currently has no bounds on the `star` field, and `RatingService.createRating` stores whatever integer is sent. A shopper who bought the product can save a rating of 0 or 99, and that value is included in the product average.

This story adds backend validation to enforce the 1–5 star range in the existing service layer.

## Technical Specification

### Scope

Modify the rating creation flow to validate the `star` field is between 1 and 5 inclusive, before database write.

### Files Changed

1. **`rating/src/main/java/com/yas/rating/viewmodel/RatingPostVm.java`**
   - Add `@Min(1)` and `@Max(5)` annotations to the `star` field
   - Existing validation annotations (e.g. `@NotNull`) remain

2. **`rating/src/main/java/com/yas/rating/service/RatingService.java`**
   - In `createRating(RatingPostVm)`, validate the view model before database write
   - Return HTTP 400 Bad Request on validation failure with standard error format
   - No change to database logic if validation passes

3. **`rating/src/test/java/com/yas/rating/service/RatingServiceTest.java`**
   - Add test cases for boundary values: 0, 1, 5, 6, -1, 99
   - Verify 400 response for invalid values
   - Verify 200 response and database record for valid values (1–5)
   - Verify invalid ratings are excluded from product average calculation

### Implementation Notes

- Use existing Spring validation framework annotations (`javax.validation.constraints.Min`, `javax.validation.constraints.Max`)
- Error response follows the existing application convention for validation errors (HTTP 400 Bad Request)
- Validation occurs in `RatingService.createRating` before calling the repository layer
- Edge case: `star: 1` and `star: 5` are valid boundary values

### Database Constraints

No schema changes required. Validation prevents invalid values from reaching the database.

### Error Response Format

Follow existing application error format:

```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "star",
      "message": "must be between 1 and 5"
    }
  ]
}
```

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

## Tests

### Unit Tests

| Test Case | Input | Expected Output | Verifies |
|-----------|-------|-----------------|----------|
| Valid boundary (min) | `star: 1` | 200 OK, database record created | AC-1 |
| Valid boundary (max) | `star: 5` | 200 OK, database record created | AC-1 |
| Valid middle values | `star: 2`, `star: 3`, `star: 4` | 200 OK, database record created | AC-1 |
| Zero star | `star: 0` | 400 Bad Request, no database record | AC-2 |
| Negative star | `star: -1` | 400 Bad Request, no database record | AC-3 |
| Above max (6) | `star: 6` | 400 Bad Request, no database record | AC-4 |
| Above max (99) | `star: 99` | 400 Bad Request, no database record | AC-4 |
| Backend enforcement | Invalid star bypassing client validation | 400 Bad Request | AC-5 |
| Average calculation | Product with valid (1,3,5) and invalid (0,99) ratings | Average includes only valid ratings | AC-6 |

### Test Files

- `rating/src/test/java/com/yas/rating/service/RatingServiceTest.java` — service-layer validation tests
- `rating/src/test/java/com/yas/rating/viewmodel/RatingPostVmTest.java` — view model annotation tests (if not already present)

## Notes

- Validation is added at the view model layer (`RatingPostVm`) using standard `javax.validation.constraints` annotations
- `RatingService.createRating` enforces validation before database write
- Error response format follows existing application convention
- Edge case: `star: 1` and `star: 5` are valid boundary values and must be explicitly tested
- No database schema changes required; validation prevents invalid values from reaching the repository layer

## References

- Ticket: PER-324
- Plan: `story-per-324-plan.md`
- Requirements: `requirements.md`
