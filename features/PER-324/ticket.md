---
key: PER-324
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Creating a rating accepts a star value below 1 or above 5
---

### Description:

`RatingPostVm` has no bounds on `star`, and `RatingService.createRating` stores whatever integer is sent. A shopper who bought the product can save a rating of 0 or 99, and that value is included in the product average.

Acceptance Criteria

1. The system must accept product ratings only between **1 and 5 stars**, inclusive.
2. The system must reject a rating with `star: 0`.
3. The system must reject negative star values, such as `star: -1`.
4. The system must reject star values greater than 5, such as `star: 6` or `star: 9`.
5. Invalid ratings must return an appropriate validation error (HTTP 400 Bad Request).
6. Invalid ratings must not be saved in the database.
7. Valid ratings with `star: 1` and `star: 5` must be accepted and saved successfully.
8. The validation must be enforced on the backend, even if the frontend submits an invalid value.
9. Only valid ratings between 1 and 5 must be included when calculating the product's average rating.
