# Story 02-app LLD: Frontend review form component with star selector and content input

**Repo:** `app`

---

## Overview

This LLD details the implementation of the frontend review submission form for story 02-app. It covers the React components, API client service, TypeScript type definitions, and integration with the product detail page. The implementation enables customers to submit product reviews (1-5 star rating + written content) with client-side validation, submit button disable-during-request logic, error message display, and success confirmation toast. It consumes the backend endpoint from Story 1 (`POST /api/rating/storefront/reviews`).

---

## Acceptance Criteria (from requirements/breakdown)

1. Customers must be able to select a star rating between 1 and 5. (AC-1 — star selector UI enforces 1-5 selection)
2. Review content must be mandatory and must not contain only whitespace. (AC-2 — client-side validation prevents submit if content empty/whitespace-only)
3. The system must prevent submission if the rating or review content is missing. (AC-3 — client-side validation displays error if either missing)
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (AC-5 — success toast "Thank you! Your review has been submitted." displayed after `201 Created` response)
10. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (AC-10 — error message displayed on `400/409/500` response; no UI update on error)
11. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (AC-11 — submit button disabled while `isSubmitting = true`)

---

## File Plan

### New Files

1. **`storefront/modules/review/components/ReviewForm.tsx`**
   - Form component: star selector (1-5), review content textarea, submit button, client-side validation, error/success messages
   - Depends on: `ReviewService`, `Review` types, existing toast library (verify pattern in repo)
   - State: `rating: number | null`, `content: string`, `isSubmitting: boolean`, `error: string | null`, `success: boolean`

2. **`storefront/modules/review/services/ReviewService.ts`**
   - API client for review submission and retrieval
   - Methods: `postReview(productId, rating, content)`, `getReviews(productId, pageNo, pageSize)`
   - Depends on: existing API client pattern in storefront (verify `fetch` or axios usage in `backoffice/common/services/ApiClientService.ts` for guidance), authentication mechanism (auth token injection)

3. **`storefront/modules/review/models/Review.ts`**
   - TypeScript type definitions: `Review`, `ReviewPostRequest`, `ReviewListResponse`
   - No dependencies

4. **`storefront/modules/review/components/ReviewForm.test.tsx`**
   - Component tests for form validation, button disable, error display
   - 7 test cases (see Tests section)
   - Depends on: React Testing Library, `ReviewForm` component, mock `ReviewService`

### Modified Files

1. **Product detail page** (exact path TBD — see note below)
   - Add state: `const [reviewsKey, setReviewsKey] = useState(0);` (force reviews list refresh)
   - Add callback: `const handleReviewSubmitted = () => { setReviewsKey(prev => prev + 1); /* also refetch product data to update header */ }`
   - Render `<ReviewForm productId={product.id} onSuccess={handleReviewSubmitted} />` (place above or within reviews section)
   - **Note:** The `repo-tree/app` shows `backoffice` and `backoffice-bff` directories but no `storefront` or `storefront-ui` directory. The storefront appears to be a separate Next.js application not included in the `app` repo provided in the context. This story's implementation belongs in the **storefront repository**, which must be provided separately. The file path will be determined once the storefront repo is available (likely `pages/products/[id].tsx` or `app/products/[id]/page.tsx` depending on Next.js version).

## Interfaces and Data Contracts

### TypeScript Types (Review.ts)

```typescript
// storefront/modules/review/models/Review.ts

export interface Review {
  id: number;
  customerId: string;
  rating: number;
  content: string;
  createdOn: string; // ISO 8601 timestamp (e.g. "2025-05-23T14:35:00Z")
}

export interface ReviewPostRequest {
  productId: number;
  rating: number;
  content: string;
}

export interface ReviewListResponse {
  reviews: Review[];
  totalPages: number;
  totalElements: number;
  pageNo: number;
  pageSize: number;
}
```

---

### API Client Interface (ReviewService.ts)

```typescript
// storefront/modules/review/services/ReviewService.ts

export async function postReview(
  productId: number,
  rating: number,
  content: string
): Promise<Review>;

export async function getReviews(
  productId: number,
  pageNo: number,
  pageSize: number
): Promise<ReviewListResponse>;
```

**Method Details:**

1. **`postReview(productId: number, rating: number, content: string): Promise<Review>`**
   - **Request:** 
     - Method: `POST`
     - URL: `/api/rating/storefront/reviews`
     - Body: `{ productId, rating, content }`
     - Headers: `Content-Type: application/json`, auth token (injected by API client)
   - **Success Response (201 Created):**
     ```json
     {
       "id": 456,
       "productId": 123,
       "customerId": "user-uuid",
       "rating": 5,
       "content": "Excellent product!",
       "createdOn": "2025-05-23T14:35:00Z"
     }
     ```
     Maps to `Review` interface and returns.
   - **Error Responses:**
     - 400 Bad Request: throw `Error` with `detail` from response body (e.g. "Rating must be between 1 and 5.")
     - 409 Conflict: throw `Error` with `detail` from response body (e.g. "You have already submitted a review for this product.")
     - 500 Internal Server Error: throw `Error` with generic message "An error occurred. Please try again later."
   - **Algorithm:**
     1. Construct request body: `{ productId, rating, content }`
     2. Call API client (use existing pattern — verify in repo, likely `fetch` or axios; backoffice uses custom `apiClientService.post`, storefront may use similar pattern)
     3. If response status is 201, parse JSON body and return as `Review`
     4. If response status is 400/409/500, parse error body (format: `{ status, title, detail }` per `common-library/src/main/java/com/yas/commonlibrary/viewmodel/error/ErrorVm.java`), throw `Error` with `detail` message
     5. For other errors (network failure, etc.), throw `Error` with generic message

2. **`getReviews(productId: number, pageNo: number, pageSize: number): Promise<ReviewListResponse>`**
   - **Request:** 
     - Method: `GET`
     - URL: `/api/rating/storefront/reviews?productId={productId}&pageNo={pageNo}&pageSize={pageSize}`
     - Headers: auth token (injected by API client)
   - **Success Response (200 OK):**
     ```json
     {
       "reviews": [ { "id": 456, "customerId": "user-uuid", "rating": 5, "content": "Excellent product!", "createdOn": "2025-05-23T14:35:00Z" } ],
       "totalPages": 1,
       "totalElements": 1,
       "pageNo": 0,
       "pageSize": 10
     }
     ```
     Maps to `ReviewListResponse` interface and returns.
   - **Algorithm:**
     1. Construct URL with query parameters
     2. Call API client with GET method
     3. If response status is 200, parse JSON body and return as `ReviewListResponse`
     4. If error, throw `Error` with appropriate message

---

## Component Specification

### ReviewForm.tsx

**File:** `storefront/modules/review/components/ReviewForm.tsx`

#### Props

```typescript
type ReviewFormProps = {
  productId: number;
  onSuccess: () => void;
};
```

#### State

```typescript
const [rating, setRating] = useState<number | null>(null);
const [content, setContent] = useState<string>('');
const [isSubmitting, setIsSubmitting] = useState<boolean>(false);
const [error, setError] = useState<string | null>(null);
const [success, setSuccess] = useState<boolean>(false);
```

#### Rendering

- **Star selector:** 
  - Render 5 clickable stars (SVG or icon library — verify existing pattern in repo; backoffice uses custom SVGs and icons, storefront may use similar)
  - Each star is a clickable button (or interactive element) with `onClick` handler that sets `rating` to star number (1-5)
  - Visual feedback: filled stars for `rating` (e.g. if `rating === 3`, first 3 stars filled, last 2 empty)
  - If `rating === null`, all stars empty
  - Accessible: use ARIA labels (e.g. `aria-label="Rate 1 out of 5"`) and keyboard navigation (arrow keys to select rating, Enter to confirm)

- **Content textarea:**
  - Controlled input: `value={content}`, `onChange={e => setContent(e.target.value)}`
  - HTML5 `required` attribute (browser-level validation)
  - Placeholder text: "Write your review here..." (or similar)
  - Rows: 5 (or adjust based on UX needs)

- **Validation error display:**
  - If `error !== null`, render error message above form (red text, `role="alert"` for accessibility)
  - Format: `<div className="error-message" role="alert">{error}</div>` (or use existing error styling in repo)

- **Success toast:**
  - If `success === true`, display toast "Thank you! Your review has been submitted."
  - Use existing toast library in repo — verify pattern (backoffice uses custom toast in `backoffice/common/services/ToastService.ts`; storefront may use similar or a library like `react-toastify`)
  - Toast auto-dismisses after a few seconds (e.g. 4 seconds per `TOAST_DURATION` in backoffice)

- **Submit button:**
  - Text: "Submit Review"
  - `disabled={isSubmitting}` (prevents repeated clicks while request in flight)
  - Visual feedback: show loading spinner or disabled state while `isSubmitting === true`

#### Event Handlers

1. **`handleRatingChange(newRating: number)`**
   - Called when user clicks a star
   - Set `rating = newRating`
   - Clear error if previously set: `setError(null)`

2. **`handleContentChange(event: React.ChangeEvent<HTMLTextAreaElement>)`**
   - Called on textarea input change
   - Set `content = event.target.value`
   - Clear error if previously set: `setError(null)`

3. **`handleSubmit(event: React.FormEvent<HTMLFormElement>)`**
   - Called when user clicks "Submit Review" button or presses Enter in form
   - **Algorithm:**
     1. Prevent default form submission: `event.preventDefault()`
     2. **Client-side validation:**
        - If `rating === null`: set `error = "Please select a rating."`, return (do not proceed)
        - If `content.trim() === ''`: set `error = "Review content is required."`, return (do not proceed)
     3. Clear error: `setError(null)`
     4. Set `isSubmitting = true`, `success = false`
     5. Call `ReviewService.postReview(productId, rating, content)`
        - **On success (201 response):**
          - Set `success = true`
          - Display toast: "Thank you! Your review has been submitted." (use existing toast library)
          - Call `onSuccess()` callback (passed as prop) → parent refreshes reviews list and product data
          - Reset form: `setRating(null)`, `setContent('')`, `setIsSubmitting(false)`
        - **On error:**
          - Catch error from `postReview` (thrown with error message)
          - Parse error message:
            - If message contains "Rating must be between 1 and 5": set `error = message`
            - If message contains "already submitted": set `error = "You have already submitted a review for this product."`
            - Otherwise: set `error = "An error occurred. Please try again later."`
          - Set `isSubmitting = false`
          - Log error to console (for debugging): `console.error('Review submission error:', err)`

#### Example JSX Structure

```tsx
export default function ReviewForm({ productId, onSuccess }: ReviewFormProps) {
  const [rating, setRating] = useState<number | null>(null);
  const [content, setContent] = useState<string>('');
  const [isSubmitting, setIsSubmitting] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState<boolean>(false);

  const handleRatingChange = (newRating: number) => {
    setRating(newRating);
    setError(null);
  };

  const handleContentChange = (event: React.ChangeEvent<HTMLTextAreaElement>) => {
    setContent(event.target.value);
    setError(null);
  };

  const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();

    // Client-side validation
    if (rating === null) {
      setError('Please select a rating.');
      return;
    }
    if (content.trim() === '') {
      setError('Review content is required.');
      return;
    }

    setError(null);
    setIsSubmitting(true);
    setSuccess(false);

    try {
      await postReview(productId, rating, content);
      setSuccess(true);
      // Display toast (use existing toast library)
      toast.success('Thank you! Your review has been submitted.');
      onSuccess(); // Trigger parent refresh
      // Reset form
      setRating(null);
      setContent('');
      setIsSubmitting(false);
    } catch (err: any) {
      const message = err.message || 'An error occurred. Please try again later.';
      if (message.includes('already submitted')) {
        setError('You have already submitted a review for this product.');
      } else {
        setError(message);
      }
      setIsSubmitting(false);
      console.error('Review submission error:', err);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="review-form">
      <h3>Write a Review</h3>

      {/* Error message */}
      {error && (
        <div className="error-message" role="alert">
          {error}
        </div>
      )}

      {/* Star selector */}
      <div className="star-selector">
        <label>Rating:</label>
        <div className="stars">
          {[1, 2, 3, 4, 5].map((star) => (
            <button
              key={star}
              type="button"
              onClick={() => handleRatingChange(star)}
              aria-label={`Rate ${star} out of 5`}
              className={star <= (rating || 0) ? 'star filled' : 'star empty'}
            >
              ★
            </button>
          ))}
        </div>
      </div>

      {/* Content textarea */}
      <div className="content-input">
        <label htmlFor="review-content">Review:</label>
        <textarea
          id="review-content"
          value={content}
          onChange={handleContentChange}
          placeholder="Write your review here..."
          rows={5}
          required
        />
      </div>

      {/* Submit button */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit Review'}
      </button>
    </form>
  );
}
```

---

## Tests

### Component Tests (ReviewForm.test.tsx)

**Setup:**
- Use React Testing Library (`@testing-library/react`)
- Mock `ReviewService.postReview` and `ReviewService.getReviews` using Jest or Vitest (verify test framework in repo)
- Render `<ReviewForm productId={123} onSuccess={mockOnSuccess} />`

**Test count:** 7 component tests

1. **`renders star selector and content textarea`**
   - **Given:** Render `<ReviewForm productId={123} onSuccess={mockOnSuccess} />`
   - **When:** Component mounts
   - **Then:**
     - Verify 5 star buttons rendered (query by `role="button"` and `aria-label` containing "Rate")
     - Verify textarea present with `required` attribute (query by `role="textbox"` or `id="review-content"`)
   - **Coverage:** AC-1 (star selector present), AC-2 (content field present)

2. **`star selector sets rating on click`**
   - **Given:** Render component
   - **When:** Click the 3rd star button
   - **Then:**
     - Verify 3 stars have class `.filled` (or equivalent filled state)
     - Verify remaining 2 stars have class `.empty`
     - Verify `rating` state = 3 (can check by inspecting submit behavior or hidden input if added)
   - **Coverage:** AC-1 (star selection works)

3. **`client-side validation prevents submit if rating missing`**
   - **Given:** Render component
   - **When:** 
     - Leave rating unselected (`rating === null`)
     - Fill content textarea with "Great product!"
     - Click submit button
   - **Then:**
     - Verify error message "Please select a rating." displayed (query by `role="alert"`)
     - Verify `postReview` mock **not** called (use `expect(mockPostReview).not.toHaveBeenCalled()`)
   - **Coverage:** AC-3 (prevents submission if rating missing)

4. **`client-side validation prevents submit if content empty`**
   - **Given:** Render component
   - **When:** 
     - Select rating (click star 5)
     - Leave content textarea empty (or fill with only whitespace "   ")
     - Click submit button
   - **Then:**
     - Verify error message "Review content is required." displayed
     - Verify `postReview` mock **not** called
   - **Coverage:** AC-2, AC-3 (prevents submission if content empty/whitespace-only)

5. **`submit button disabled during request`**
   - **Given:** Render component, mock `postReview` with delay (use `jest.fn(() => new Promise(resolve => setTimeout(resolve, 1000)))`)
   - **When:** 
     - Fill valid form (rating 5, content "Great!")
     - Click submit button
   - **Then:**
     - Verify submit button `disabled` attribute is `true` immediately (before API response)
     - Wait for mock API delay, verify button remains disabled during request
     - After response, verify button re-enabled (or form reset)
   - **Coverage:** AC-11 (button disabled during request prevents duplicate clicks)

6. **`success toast displayed on 201 response`**
   - **Given:** Render component, mock `postReview` returns resolved promise with `Review` object (201 success)
   - **When:** 
     - Fill valid form (rating 5, content "Great product!")
     - Click submit button
   - **Then:**
     - Verify `postReview` called with `(123, 5, "Great product!")`
     - Verify toast "Thank you! Your review has been submitted." displayed (mock toast library or query DOM for toast element)
     - Verify `onSuccess` callback called once
     - Verify form reset: rating cleared (all stars empty), content textarea empty
   - **Coverage:** AC-5 (success toast displayed)

7. **`error message displayed on 409 response`**
   - **Given:** Render component, mock `postReview` returns rejected promise with error message "You have already submitted a review for this product." (409 Conflict)
   - **When:** 
     - Fill valid form (rating 4, content "Good product")
     - Click submit button
   - **Then:**
     - Verify `postReview` called with `(123, 4, "Good product")`
     - Verify error message "You have already submitted a review for this product." displayed (query by `role="alert"`)
     - Verify form **not** reset: rating still 4, content still "Good product" (user can correct and retry)
     - Verify `onSuccess` callback **not** called
   - **Coverage:** AC-10 (error message displayed, no UI update on error)

---

## Implementation Code

### New Files

#### 1. Review.ts

```typescript
// storefront/modules/review/models/Review.ts

export interface Review {
  id: number;
  customerId: string;
  rating: number;
  content: string;
  createdOn: string; // ISO 8601 timestamp
}

export interface ReviewPostRequest {
  productId: number;
  rating: number;
  content: string;
}

export interface ReviewListResponse {
  reviews: Review[];
  totalPages: number;
  totalElements: number;
  pageNo: number;
  pageSize: number;
}
```

---

#### 2. ReviewService.ts

```typescript
// storefront/modules/review/services/ReviewService.ts

import { Review, ReviewListResponse } from '../models/Review';

const baseUrl = '/api/rating/storefront/reviews';

// Helper function to get auth token (verify existing pattern in storefront)
// Example: const getAuthToken = () => localStorage.getItem('authToken');
// Or: const getAuthToken = () => document.cookie.match(/authToken=([^;]+)/)?.[1];
// Adjust based on storefront's authentication mechanism
const getAuthToken = (): string => {
  // Placeholder - implement based on storefront's auth pattern
  return ''; // Return actual token from session/cookie/localStorage
};

export async function postReview(
  productId: number,
  rating: number,
  content: string
): Promise<Review> {
  const response = await fetch(baseUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${getAuthToken()}`,
    },
    body: JSON.stringify({ productId, rating, content }),
  });

  if (response.status === 201) {
    const data = await response.json();
    return data as Review;
  }

  // Handle error responses
  if (response.status === 400 || response.status === 409 || response.status === 500) {
    try {
      const errorData = await response.json();
      // ErrorVm format: { statusCode, title, detail, fieldErrors }
      throw new Error(errorData.detail || 'An error occurred. Please try again later.');
    } catch (err) {
      throw new Error('An error occurred. Please try again later.');
    }
  }

  // Other errors (network, etc.)
  throw new Error('An error occurred. Please try again later.');
}

export async function getReviews(
  productId: number,
  pageNo: number,
  pageSize: number
): Promise<ReviewListResponse> {
  const url = `${baseUrl}?productId=${productId}&pageNo=${pageNo}&pageSize=${pageSize}`;
  
  const response = await fetch(url, {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${getAuthToken()}`,
    },
  });

  if (response.status === 200) {
    const data = await response.json();
    return data as ReviewListResponse;
  }

  throw new Error('Failed to load reviews.');
}
```

**Note:** The `getAuthToken()` function is a placeholder. Verify the storefront's authentication mechanism (session cookie, localStorage, context provider, etc.) and implement token retrieval accordingly. The backoffice uses a Next.js BFF pattern (`backoffice-bff`) with server-side session management; the storefront may use a similar approach or client-side token storage.

---

#### 3. ReviewForm.tsx

```tsx
// storefront/modules/review/components/ReviewForm.tsx

import { useState } from 'react';
import { postReview } from '../services/ReviewService';
import { toastSuccess, toastError } from '../../common/services/ToastService'; // Adjust import path

type ReviewFormProps = {
  productId: number;
  onSuccess: () => void;
};

export default function ReviewForm({ productId, onSuccess }: ReviewFormProps) {
  const [rating, setRating] = useState<number | null>(null);
  const [content, setContent] = useState<string>('');
  const [isSubmitting, setIsSubmitting] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState<boolean>(false);

  const handleRatingChange = (newRating: number) => {
    setRating(newRating);
    setError(null);
  };

  const handleContentChange = (event: React.ChangeEvent<HTMLTextAreaElement>) => {
    setContent(event.target.value);
    setError(null);
  };

  const handleSubmit = async (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();

    // Client-side validation
    if (rating === null) {
      setError('Please select a rating.');
      return;
    }
    if (content.trim() === '') {
      setError('Review content is required.');
      return;
    }

    setError(null);
    setIsSubmitting(true);
    setSuccess(false);

    try {
      await postReview(productId, rating, content);
      setSuccess(true);
      
      // Display success toast
      toastSuccess('Thank you! Your review has been submitted.');
      
      // Trigger parent refresh
      onSuccess();
      
      // Reset form
      setRating(null);
      setContent('');
      setIsSubmitting(false);
    } catch (err: any) {
      const message = err.message || 'An error occurred. Please try again later.';
      if (message.includes('already submitted')) {
        setError('You have already submitted a review for this product.');
      } else {
        setError(message);
      }
      setIsSubmitting(false);
      console.error('Review submission error:', err);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="review-form">
      <h3>Write a Review</h3>

      {/* Error message */}
      {error && (
        <div className="error-message" role="alert">
          {error}
        </div>
      )}

      {/* Star selector */}
      <div className="star-selector">
        <label>Rating:</label>
        <div className="stars">
          {[1, 2, 3, 4, 5].map((star) => (
            <button
              key={star}
              type="button"
              onClick={() => handleRatingChange(star)}
              aria-label={`Rate ${star} out of 5`}
              className={star <= (rating || 0) ? 'star filled' : 'star empty'}
            >
              ★
            </button>
          ))}
        </div>
      </div>

      {/* Content textarea */}
      <div className="content-input">
        <label htmlFor="review-content">Review:</label>
        <textarea
          id="review-content"
          value={content}
          onChange={handleContentChange}
          placeholder="Write your review here..."
          rows={5}
          required
        />
      </div>

      {/* Submit button */}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit Review'}
      </button>
    </form>
  );
}
```

**Note:** The toast import path assumes the storefront uses a similar pattern to the backoffice (`common/services/ToastService.ts`). If the storefront uses a different toast library (e.g. `react-toastify`), adjust the import:

```tsx
// If using react-toastify:
import { toast } from 'react-toastify';

// In handleSubmit success block:
toast.success('Thank you! Your review has been submitted.');
```

---

#### 4. ReviewForm.test.tsx

```tsx
// storefront/modules/review/components/ReviewForm.test.tsx

import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import ReviewForm from './ReviewForm';
import * as ReviewService from '../services/ReviewService';
import * as ToastService from '../../common/services/ToastService';

// Mock dependencies
jest.mock('../services/ReviewService');
jest.mock('../../common/services/ToastService');

describe('ReviewForm', () => {
  const mockOnSuccess = jest.fn();
  const mockPostReview = ReviewService.postReview as jest.MockedFunction<typeof ReviewService.postReview>;
  const mockToastSuccess = ToastService.toastSuccess as jest.MockedFunction<typeof ToastService.toastSuccess>;

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders star selector and content textarea', () => {
    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Verify 5 star buttons
    const starButtons = screen.getAllByRole('button', { name: /Rate \d out of 5/ });
    expect(starButtons).toHaveLength(5);
    
    // Verify textarea
    const textarea = screen.getByRole('textbox', { name: /Review/ });
    expect(textarea).toBeInTheDocument();
    expect(textarea).toHaveAttribute('required');
  });

  test('star selector sets rating on click', () => {
    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Click 3rd star
    const star3 = screen.getByRole('button', { name: 'Rate 3 out of 5' });
    fireEvent.click(star3);
    
    // Verify filled stars (1, 2, 3)
    const stars = screen.getAllByRole('button', { name: /Rate \d out of 5/ });
    expect(stars[0]).toHaveClass('star filled');
    expect(stars[1]).toHaveClass('star filled');
    expect(stars[2]).toHaveClass('star filled');
    expect(stars[3]).toHaveClass('star empty');
    expect(stars[4]).toHaveClass('star empty');
  });

  test('client-side validation prevents submit if rating missing', async () => {
    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Fill content only
    const textarea = screen.getByRole('textbox', { name: /Review/ });
    fireEvent.change(textarea, { target: { value: 'Great product!' } });
    
    // Click submit
    const submitButton = screen.getByRole('button', { name: /Submit Review/ });
    fireEvent.click(submitButton);
    
    // Verify error message
    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent('Please select a rating.');
    });
    
    // Verify API not called
    expect(mockPostReview).not.toHaveBeenCalled();
  });

  test('client-side validation prevents submit if content empty', async () => {
    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Select rating only
    const star5 = screen.getByRole('button', { name: 'Rate 5 out of 5' });
    fireEvent.click(star5);
    
    // Click submit (content is empty)
    const submitButton = screen.getByRole('button', { name: /Submit Review/ });
    fireEvent.click(submitButton);
    
    // Verify error message
    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent('Review content is required.');
    });
    
    // Verify API not called
    expect(mockPostReview).not.toHaveBeenCalled();
  });

  test('submit button disabled during request', async () => {
    // Mock API with delay
    mockPostReview.mockImplementation(
      () => new Promise(resolve => setTimeout(() => resolve({
        id: 456,
        customerId: 'user-uuid',
        rating: 5,
        content: 'Great!',
        createdOn: '2025-05-23T14:35:00Z'
      }), 1000))
    );

    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Fill form
    const star5 = screen.getByRole('button', { name: 'Rate 5 out of 5' });
    fireEvent.click(star5);
    const textarea = screen.getByRole('textbox', { name: /Review/ });
    fireEvent.change(textarea, { target: { value: 'Great!' } });
    
    // Click submit
    const submitButton = screen.getByRole('button', { name: /Submit Review/ });
    fireEvent.click(submitButton);
    
    // Verify button disabled immediately
    expect(submitButton).toBeDisabled();
    expect(submitButton).toHaveTextContent('Submitting...');
    
    // Wait for response
    await waitFor(() => {
      expect(mockToastSuccess).toHaveBeenCalled();
    });
  });

  test('success toast displayed on 201 response', async () => {
    mockPostReview.mockResolvedValue({
      id: 456,
      customerId: 'user-uuid',
      rating: 5,
      content: 'Great product!',
      createdOn: '2025-05-23T14:35:00Z'
    });

    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Fill form
    const star5 = screen.getByRole('button', { name: 'Rate 5 out of 5' });
    fireEvent.click(star5);
    const textarea = screen.getByRole('textbox', { name: /Review/ });
    fireEvent.change(textarea, { target: { value: 'Great product!' } });
    
    // Click submit
    const submitButton = screen.getByRole('button', { name: /Submit Review/ });
    fireEvent.click(submitButton);
    
    // Verify API called
    await waitFor(() => {
      expect(mockPostReview).toHaveBeenCalledWith(123, 5, 'Great product!');
    });
    
    // Verify toast displayed
    expect(mockToastSuccess).toHaveBeenCalledWith('Thank you! Your review has been submitted.');
    
    // Verify callback called
    expect(mockOnSuccess).toHaveBeenCalledTimes(1);
    
    // Verify form reset
    const stars = screen.getAllByRole('button', { name: /Rate \d out of 5/ });
    stars.forEach(star => expect(star).toHaveClass('star empty'));
    expect(textarea).toHaveValue('');
  });

  test('error message displayed on 409 response', async () => {
    mockPostReview.mockRejectedValue(
      new Error('You have already submitted a review for this product.')
    );

    render(<ReviewForm productId={123} onSuccess={mockOnSuccess} />);
    
    // Fill form
    const star4 = screen.getByRole('button', { name: 'Rate 4 out of 5' });
    fireEvent.click(star4);
    const textarea = screen.getByRole('textbox', { name: /Review/ });
    fireEvent.change(textarea, { target: { value: 'Good product' } });
    
    // Click submit
    const submitButton = screen.getByRole('button', { name: /Submit Review/ });
    fireEvent.click(submitButton);
    
    // Verify API called
    await waitFor(() => {
      expect(mockPostReview).toHaveBeenCalledWith(123, 4, 'Good product');
    });
    
    // Verify error message
    expect(screen.getByRole('alert')).toHaveTextContent(
      'You have already submitted a review for this product.'
    );
    
    // Verify form NOT reset
    const stars = screen.getAllByRole('button', { name: /Rate \d out of 5/ });
    expect(stars[3]).toHaveClass('star filled');
    expect(textarea).toHaveValue('Good product');
    
    // Verify callback NOT called
    expect(mockOnSuccess).not.toHaveBeenCalled();
  });
});
```

---

### Modified Files

#### products/[id].tsx (Integration)

```tsx
// storefront/pages/products/[id].tsx
// (or exact path — verify in repo)

import { useState, useEffect } from 'react';
import { useRouter } from 'next/router';
import ReviewForm from '../modules/review/components/ReviewForm';
// ... other imports

export default function ProductDetailPage() {
  const router = useRouter();
  const { id } = router.query;
  const productId = Number(id);

  const [product, setProduct] = useState<Product | null>(null);
  const [reviewsKey, setReviewsKey] = useState(0);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (productId) {
      fetchProduct();
    }
  }, [productId, reviewsKey]);

  const fetchProduct = async () => {
    setLoading(true);
    setError(null);
    try {
      // Fetch product data (verify exact service method in repo)
      const productData = await ProductService.getProduct(productId);
      setProduct(productData);
      setLoading(false);
    } catch (err: any) {
      setError('Failed to load product.');
      setLoading(false);
      console.error('Failed to load product:', err);
    }
  };

  const handleReviewSubmitted = () => {
    setReviewsKey(prev => prev + 1); // Trigger product + reviews refresh
  };

  if (loading) {
    return <div>Loading product...</div>;
  }

  if (error || !product) {
    return <div className="alert alert-danger">{error || 'Product not found'}</div>;
  }

  return (
    <div className="product-detail-page">
      {/* Product header, description, etc. */}
      
      {/* Reviews section */}
      <div className="product-reviews-section">
        <ReviewForm productId={product.id} onSuccess={handleReviewSubmitted} />
        {/* ReviewsList component (Story 3) will be rendered here */}
      </div>
    </div>
  );
}
```

**Note:** The exact path and structure depend on the storefront's codebase. Verify:
- Product detail page location (e.g. `pages/products/[id].tsx` or `app/products/[id]/page.tsx` for Next.js 13+ App Router)
- Product service method name (e.g. `ProductService.getProduct(productId)`)
- Existing state management (may already have product fetching logic)

---

## Styling

### CSS (suggested classes)

```css
/* storefront/styles/ReviewForm.module.css (or global CSS) */

.review-form {
  max-width: 600px;
  margin: 2rem 0;
  padding: 1.5rem;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fff;
}

.review-form h3 {
  margin-bottom: 1rem;
  font-size: 1.5rem;
  color: #333;
}

.error-message {
  padding: 0.75rem;
  margin-bottom: 1rem;
  background: #fee;
  border: 1px solid #fcc;
  border-radius: 4px;
  color: #c33;
}

.star-selector {
  margin-bottom: 1rem;
}

.star-selector label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

.stars {
  display: flex;
  gap: 0.25rem;
}

.star {
  background: none;
  border: none;
  font-size: 2rem;
  cursor: pointer;
  padding: 0;
  transition: color 0.2s;
}

.star.filled {
  color: #FFD700; /* Gold */
}

.star.empty {
  color: #D3D3D3; /* Light gray */
}

.star:hover {
  transform: scale(1.1);
}

.star:focus {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

.content-input {
  margin-bottom: 1rem;
}

.content-input label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

.content-input textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-family: inherit;
  font-size: 1rem;
  resize: vertical;
}

.content-input textarea:focus {
  outline: none;
  border-color: #007bff;
  box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.1);
}

button[type="submit"] {
  padding: 0.75rem 2rem;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

button[type="submit"]:hover:not(:disabled) {
  background: #0056b3;
}

button[type="submit"]:disabled {
  background: #ccc;
  cursor: not-allowed;
}
```

**Integration:** Import the CSS module in `ReviewForm.tsx`:
```tsx
import styles from '../../styles/ReviewForm.module.css';
// Update className attributes to use styles.reviewForm, styles.starSelector, etc.
```

Or use global CSS (adjust path based on storefront's CSS structure).

---

## Implementation Notes

### Critical Path Items

1. **Verify storefront location:** Confirm whether `storefront` is a separate repo or under `app` repo but not shown in tree.

2. **Verify authentication mechanism:** Determine how the storefront handles auth tokens (session cookie, localStorage, context provider) and implement `getAuthToken()` in `ReviewService.ts` accordingly.

3. **Verify toast library:** Check if storefront uses `react-toastify`, custom toast service (like backoffice), or another library. Adjust imports and usage in `ReviewForm.tsx`.

4. **Verify product detail page path:** Find the exact file path for the product detail page and existing product fetching logic before modifying.

5. **Verify test framework:** Check `package.json` for testing dependencies (Jest, Vitest, React Testing Library). If missing, install and configure before writing tests.

### Optional Enhancements (Out of Scope)

- Keyboard navigation for star selector (arrow keys to select, Enter to confirm)
- Loading spinner animation on submit button
- Character count for review content (e.g. "0/5000 characters")
- Preview mode (show how the review will look before submission)
- Star rating hover preview (show potential rating before clicking)

---

**End of LLD**
