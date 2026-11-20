# Story 02-app: Frontend Review Submission Form and Display

**Repo:** `app`

---

## Context

Add the UI for customers to submit reviews (star rating selector, text input for content, submit button) on the product detail page, and display the submitted reviews list with updated aggregates (average rating, review count) in the product header and Reviews section. This story connects the frontend to the backend API created in Story 1, implements client-side validation (disable submit until fields valid, prevent double-clicks), and handles success/error responses (toast notifications, error messages).

---

## Acceptance Criteria

1. The newly submitted review must appear in the product's Reviews section with the correct rating and review content. (AC 6)
2. The rating and review count displayed in the product header must match the Reviews section. (AC 9)
3. If submission fails, an appropriate error message must be displayed without incorrectly updating the rating or review count. (AC 10 — frontend displays error toast, does not update local state)
4. Repeated clicks on the Submit button must not create duplicate reviews while a request is processing. (AC 11 — frontend disables submit button while request in-flight)
5. Upon successful submission, the review must be saved and a confirmation message must be displayed. (AC 5 — frontend shows success toast)

---

## File Plan

### New Files

1. **`storefront/modules/product/services/ReviewService.ts`** — API client wrapper for review endpoints
   - Function `createReview(productId, rating, content)` calls `POST /api/rating/storefront/products/{productId}/reviews`
   - Function `getReviews(productId)` calls `GET /api/rating/storefront/products/{productId}/reviews`
   - Uses `apiClientService` pattern from `backoffice/common/services/ApiClientService.ts`

2. **`storefront/modules/product/models/Review.ts`** — TypeScript type definitions
   - `ReviewVm` type (id, productId, customerId, rating, content, createdAt)
   - `ReviewListVm` type (reviews array, averageRating, totalReviews)
   - `ReviewFormData` type (rating, content)

3. **`storefront/modules/product/components/ReviewForm.tsx`** — Review submission form component
   - Star rating selector (1–5) with click handler
   - Textarea for review content (max 1000 characters)
   - Submit button (disabled when isSubmitting or rating === 0)
   - Validation using `react-hook-form`: rating required, content non-empty/non-whitespace
   - Success: show toast, clear form, call `onReviewSubmitted()` callback
   - Error: show toast with error message from API response

4. **`storefront/modules/product/components/ReviewList.tsx`** — Review list and aggregates display
   - Fetches reviews via `ReviewService.getReviews(productId)` on mount and when `refreshTrigger` changes
   - Displays aggregates: `{averageRating.toFixed(1)} ★ ({totalReviews} reviews)`
   - Displays list of reviews: rating stars, content, formatted timestamp
   - Empty state: "No reviews yet. Be the first to review this product!"
   - Loading state while fetching

5. **`storefront/pages/products/[slug]/reviews.tsx`** — Standalone reviews page (optional)
   - Integrates `ReviewForm` and `ReviewList` components
   - Passes `productId` prop to both

### Modified Files

1. **`storefront/pages/products/[slug].tsx`** — Product detail page
   - Add state: `const [refreshTrigger, setRefreshTrigger] = useState<number>(0);`
   - Add callback: `const handleReviewSubmitted = () => { setRefreshTrigger((prev) => prev + 1); };`
   - Integrate components in reviews section:
     ```tsx
     <div className="reviews-section mt-5">
       <h2>Customer Reviews</h2>
       <ReviewList productId={productId} refreshTrigger={refreshTrigger} />
       <div className="review-form-section mt-4">
         <h3>Write a Review</h3>
         <ReviewForm productId={productId} onReviewSubmitted={handleReviewSubmitted} />
       </div>
     </div>
     ```
   - **Note:** The repo tree does not show `storefront/pages/products/[slug].tsx` (tree stops at 1000 files and does not include any `storefront/pages/` directory). If this file does not exist, create it following Next.js pages pattern with dynamic `[slug]` route and integrate the `ReviewForm` and `ReviewList` components inline.

2. **`storefront/modules/product/components/ProductHeader.tsx`** — Product header component
   - Add state: `const [averageRating, setAverageRating] = useState<number>(0);`
   - Add state: `const [totalReviews, setTotalReviews] = useState<number>(0);`
   - Add useEffect hook to fetch aggregates on mount:
     ```tsx
     useEffect(() => {
       const fetchAggregates = async () => {
         try {
           const data = await getReviews(productId);
           setAverageRating(data.averageRating);
           setTotalReviews(data.totalReviews);
         } catch (error) {
           console.error('Failed to fetch review aggregates:', error);
         }
       };
       fetchAggregates();
     }, [productId]);
     ```
   - Add display in product header:
     ```tsx
     <div className="product-rating">
       <span className="rating-stars">{averageRating.toFixed(1)} ★</span>
       <span className="review-count">({totalReviews} {totalReviews === 1 ? 'review' : 'reviews'})</span>
     </div>
     ```
   - **Note:** If `ProductHeader.tsx` does not exist, integrate aggregates display inline in product detail page

### Test Files

3. **`storefront/modules/product/components/ReviewForm.test.tsx`** — Unit tests for `ReviewForm`
4. **`storefront/modules/product/components/ReviewList.test.tsx`** — Unit tests for `ReviewList`
5. **`storefront/e2e/reviews.spec.ts`** — End-to-end test for review submission flow

### Test Files

3. **`storefront/modules/product/components/ReviewForm.test.tsx`** — Unit tests for `ReviewForm`
4. **`storefront/modules/product/components/ReviewList.test.tsx`** — Unit tests for `ReviewList`
5. **`storefront/e2e/reviews.spec.ts`** — End-to-end test for review submission flow

---

## Interfaces

### API Client Service

**File:** `storefront/modules/product/services/ReviewService.ts`

```typescript
import apiClientService from '../../common/services/ApiClientService';
import { ReviewVm, ReviewListVm } from '../models/Review';

const baseUrl = '/api/rating/storefront/products';

export async function createReview(
  productId: number,
  rating: number,
  content: string
): Promise<ReviewVm> {
  const url = `${baseUrl}/${productId}/reviews`;
  const body = JSON.stringify({ rating, content });
  const response = await apiClientService.post(url, body);
  return response;
}

export async function getReviews(productId: number): Promise<ReviewListVm> {
  const url = `${baseUrl}/${productId}/reviews`;
  const response = await apiClientService.get(url);
  return response;
}
```

---

### Type Definitions

**File:** `storefront/modules/product/models/Review.ts`

```typescript
export type ReviewVm = {
  id: number;
  productId: number;
  customerId: string;
  rating: number;
  content: string;
  createdAt: string;
};

export type ReviewListVm = {
  reviews: ReviewVm[];
  averageRating: number;
  totalReviews: number;
};

export type ReviewFormData = {
  rating: number;
  content: string;
};
```

---

### React Components

#### ReviewForm Component

**File:** `storefront/modules/product/components/ReviewForm.tsx`

```tsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { toast } from 'react-toastify';
import { createReview } from '../services/ReviewService';
import { ReviewFormData } from '../models/Review';

type ReviewFormProps = {
  productId: number;
  onReviewSubmitted: () => void;
};

export default function ReviewForm({ productId, onReviewSubmitted }: ReviewFormProps) {
  const { register, handleSubmit, reset, formState: { errors, isSubmitting } } = useForm<ReviewFormData>();
  const [rating, setRating] = useState<number>(0);

  const onSubmit = async (data: ReviewFormData) => {
    try {
      await createReview(productId, data.rating, data.content);
      toast.success('Review submitted successfully!');
      reset();
      setRating(0);
      onReviewSubmitted();
    } catch (error: any) {
      let errorMessage = 'Failed to submit review';
      
      if (error?.detail) {
        errorMessage = error.detail;
      } else if (error?.fieldErrors && error.fieldErrors.length > 0) {
        errorMessage = error.fieldErrors[0].message;
      } else if (error?.message) {
        errorMessage = error.message;
      }
      
      toast.error(errorMessage);
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Star rating selector */}
      <div className="mb-3">
        <label className="form-label">Rating *</label>
        <div className="star-rating">
          {[1, 2, 3, 4, 5].map((star) => (
            <span
              key={star}
              className={`star ${rating >= star ? 'filled' : ''}`}
              onClick={() => setRating(star)}
            >
              ★
            </span>
          ))}
        </div>
        <input
          type="hidden"
          {...register('rating', {
            required: 'Rating is required',
            validate: (value) => value > 0 || 'Please select a rating',
          })}
          value={rating}
        />
        {errors.rating && <p className="error-field">{errors.rating.message}</p>}
      </div>

      {/* Content textarea */}
      <div className="mb-3">
        <label className="form-label" htmlFor="content">Review Content *</label>
        <textarea
          id="content"
          className={`form-control ${errors.content ? 'border-danger' : ''}`}
          rows={5}
          maxLength={1000}
          {...register('content', {
            required: 'Review content is required',
            validate: {
              notEmpty: (value) => value.trim().length > 0 || 'Review content cannot be empty',
            },
            maxLength: {
              value: 1000,
              message: 'Review content must not exceed 1000 characters',
            },
          })}
        />
        {errors.content && <p className="error-field">{errors.content.message}</p>}
      </div>

      {/* Submit button */}
      <button
        type="submit"
        className="btn btn-primary"
        disabled={isSubmitting || rating === 0}
      >
        {isSubmitting ? 'Submitting...' : 'Submit Review'}
      </button>
    </form>
  );
}
```

---

#### ReviewList Component

**File:** `storefront/modules/product/components/ReviewList.tsx`

```tsx
import { useEffect, useState } from 'react';
import moment from 'moment';
import { getReviews } from '../services/ReviewService';
import { ReviewListVm, ReviewVm } from '../models/Review';

type ReviewListProps = {
  productId: number;
  refreshTrigger?: number;
};

export default function ReviewList({ productId, refreshTrigger = 0 }: ReviewListProps) {
  const [reviewList, setReviewList] = useState<ReviewVm[]>([]);
  const [averageRating, setAverageRating] = useState<number>(0);
  const [totalReviews, setTotalReviews] = useState<number>(0);
  const [isLoading, setIsLoading] = useState<boolean>(false);

  useEffect(() => {
    const fetchReviews = async () => {
      setIsLoading(true);
      try {
        const data: ReviewListVm = await getReviews(productId);
        setReviewList(data.reviews);
        setAverageRating(data.averageRating);
        setTotalReviews(data.totalReviews);
      } catch (error) {
        console.error('Failed to fetch reviews:', error);
      } finally {
        setIsLoading(false);
      }
    };

    fetchReviews();
  }, [productId, refreshTrigger]);

  if (isLoading) return <p>Loading reviews...</p>;

  return (
    <div className="review-list">
      {/* Aggregates */}
      <div className="review-aggregates mb-3">
        <h4>
          {averageRating.toFixed(1)} ★ ({totalReviews} {totalReviews === 1 ? 'review' : 'reviews'})
        </h4>
      </div>

      {/* Reviews */}
      {reviewList.length === 0 ? (
        <p>No reviews yet. Be the first to review this product!</p>
      ) : (
        <div className="reviews">
          {reviewList.map((review) => (
            <div key={review.id} className="review-item mb-4">
              <div className="review-rating">
                {'★'.repeat(review.rating)}{'☆'.repeat(5 - review.rating)}
              </div>
              <p className="review-content">{review.content}</p>
              <p className="review-date text-muted">
                {moment(review.createdAt).format('MMMM Do YYYY, h:mm a')}
              </p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## Tests

### Unit Tests (ReviewForm)

**Test File:** `storefront/modules/product/components/ReviewForm.test.tsx`

**Coverage:** 6 tests

1. **renders form with star rating selector and content textarea** — Verify form elements are present
2. **submit button is disabled until rating selected and content non-empty** — Verify validation state
3. **clicking submit with valid data calls ReviewService.createReview and shows success toast** — Verify happy path
4. **clicking submit with empty content does not call API** — Verify validation prevents submission
5. **displays error toast when API call fails** — Verify error handling
6. **submit button is disabled while request is in-flight** — Verify idempotency at UI level

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { toast } from 'react-toastify';
import ReviewForm from './ReviewForm';
import * as ReviewService from '../services/ReviewService';

jest.mock('../services/ReviewService');
jest.mock('react-toastify');

describe('ReviewForm', () => {
  const mockOnReviewSubmitted = jest.fn();
  const productId = 1;

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders form with star rating selector and content textarea', () => {
    render(<ReviewForm productId={productId} onReviewSubmitted={mockOnReviewSubmitted} />);
    
    expect(screen.getByLabelText(/Rating/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/Review Content/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /Submit Review/i })).toBeInTheDocument();
  });

  test('submit button is disabled until rating selected and content non-empty', () => {
    render(<ReviewForm productId={productId} onReviewSubmitted={mockOnReviewSubmitted} />);
    
    const submitButton = screen.getByRole('button', { name: /Submit Review/i });
    
    expect(submitButton).toBeDisabled();
    
    const star3 = screen.getAllByText('★')[2];
    fireEvent.click(star3);
    
    expect(submitButton).toBeDisabled();
    
    const contentTextarea = screen.getByLabelText(/Review Content/i);
    fireEvent.change(contentTextarea, { target: { value: 'Great product!' } });
    
    expect(submitButton).not.toBeDisabled();
  });

  test('clicking submit with valid data calls ReviewService.createReview and shows success toast', async () => {
    (ReviewService.createReview as jest.Mock).mockResolvedValue({
      id: 123,
      productId: productId,
      customerId: 'customer-uuid',
      rating: 5,
      content: 'Great product!',
      createdAt: '2025-01-15T10:30:00Z',
    });
    
    render(<ReviewForm productId={productId} onReviewSubmitted={mockOnReviewSubmitted} />);
    
    const star5 = screen.getAllByText('★')[4];
    fireEvent.click(star5);
    
    const contentTextarea = screen.getByLabelText(/Review Content/i);
    fireEvent.change(contentTextarea, { target: { value: 'Great product!' } });
    
    const submitButton = screen.getByRole('button', { name: /Submit Review/i });
    fireEvent.click(submitButton);
    
    await waitFor(() => {
      expect(ReviewService.createReview).toHaveBeenCalledWith(productId, 5, 'Great product!');
      expect(toast.success).toHaveBeenCalledWith('Review submitted successfully!');
      expect(mockOnReviewSubmitted).toHaveBeenCalled();
    });
    
    expect(contentTextarea).toHaveValue('');
  });

  test('clicking submit with empty content does not call API', async () => {
    render(<ReviewForm productId={productId} onReviewSubmitted={mockOnReviewSubmitted} />);
    
    const star5 = screen.getAllByText('★')[4];
    fireEvent.click(star5);
    
    const submitButton = screen.getByRole('button', { name: /Submit Review/i });
    fireEvent.click(submitButton);
    
    await waitFor(() => {
      expect(ReviewService.createReview).not.toHaveBeenCalled();
    });
    
    expect(screen.getByText(/Review content is required/i)).toBeInTheDocument();
  });

  test('displays error toast when API call fails', async () => {
    (ReviewService.createReview as jest.Mock).mockRejectedValue({
      detail: 'You have already reviewed this product',
    });
    
    render(<ReviewForm productId={productId} onReviewSubmitted={mockOnReviewSubmitted} />);
    
    const star5 = screen.getAllByText('★')[4];
    fireEvent.click(star5);
    
    const contentTextarea = screen.getByLabelText(/Review Content/i);
    fireEvent.change(contentTextarea, { target: { value: 'Great product!' } });
    
    const submitButton = screen.getByRole('button', { name: /Submit Review/i });
    fireEvent.click(submitButton);
    
    await waitFor(() => {
      expect(ReviewService.createReview).toHaveBeenCalledWith(productId, 5, 'Great product!');
      expect(toast.error).toHaveBeenCalledWith('You have already reviewed this product');
    });
    
    expect(contentTextarea).toHaveValue('Great product!');
  });

  test('submit button is disabled while request is in-flight', async () => {
    (ReviewService.createReview as jest.Mock).mockImplementation(() => {
      return new Promise((resolve) => setTimeout(resolve, 1000));
    });
    
    render(<ReviewForm productId={productId} onReviewSubmitted={mockOnReviewSubmitted} />);
    
    const star5 = screen.getAllByText('★')[4];
    fireEvent.click(star5);
    const contentTextarea = screen.getByLabelText(/Review Content/i);
    fireEvent.change(contentTextarea, { target: { value: 'Great product!' } });
    
    const submitButton = screen.getByRole('button', { name: /Submit Review/i });
    fireEvent.click(submitButton);
    
    await waitFor(() => {
      expect(submitButton).toBeDisabled();
      expect(submitButton).toHaveTextContent('Submitting...');
    });
  });
});
```

---

### Unit Tests (ReviewList)

**Test File:** `storefront/modules/product/components/ReviewList.test.tsx`

**Coverage:** 4 tests

7. **fetches reviews on mount and displays list with correct rating and content** — Verify data fetching and rendering
8. **displays empty state when no reviews exist** — Verify empty state
9. **displays loading state while fetching** — Verify loading state
10. **refetches reviews when refreshTrigger changes** — Verify refetch mechanism

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import ReviewList from './ReviewList';
import * as ReviewService from '../services/ReviewService';

jest.mock('../services/ReviewService');

describe('ReviewList', () => {
  const productId = 1;

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('fetches reviews on mount and displays list with correct rating and content', async () => {
    (ReviewService.getReviews as jest.Mock).mockResolvedValue({
      reviews: [
        {
          id: 1,
          productId: productId,
          customerId: 'customer-1',
          rating: 5,
          content: 'Excellent product!',
          createdAt: '2025-01-15T10:30:00Z',
        },
        {
          id: 2,
          productId: productId,
          customerId: 'customer-2',
          rating: 4,
          content: 'Good product.',
          createdAt: '2025-01-14T09:00:00Z',
        },
      ],
      averageRating: 4.5,
      totalReviews: 2,
    });
    
    render(<ReviewList productId={productId} />);
    
    await waitFor(() => {
      expect(ReviewService.getReviews).toHaveBeenCalledWith(productId);
    });
    
    expect(screen.getByText(/4.5 ★/i)).toBeInTheDocument();
    expect(screen.getByText(/\(2 reviews\)/i)).toBeInTheDocument();
    
    expect(screen.getByText('Excellent product!')).toBeInTheDocument();
    expect(screen.getByText('Good product.')).toBeInTheDocument();
  });

  test('displays empty state when no reviews exist', async () => {
    (ReviewService.getReviews as jest.Mock).mockResolvedValue({
      reviews: [],
      averageRating: 0,
      totalReviews: 0,
    });
    
    render(<ReviewList productId={productId} />);
    
    await waitFor(() => {
      expect(ReviewService.getReviews).toHaveBeenCalledWith(productId);
    });
    
    expect(screen.getByText(/No reviews yet/i)).toBeInTheDocument();
    expect(screen.getByText(/Be the first to review this product!/i)).toBeInTheDocument();
  });

  test('displays loading state while fetching', () => {
    (ReviewService.getReviews as jest.Mock).mockImplementation(() => {
      return new Promise((resolve) => setTimeout(resolve, 1000));
    });
    
    render(<ReviewList productId={productId} />);
    
    expect(screen.getByText(/Loading reviews.../i)).toBeInTheDocument();
  });

  test('refetches reviews when refreshTrigger changes', async () => {
    (ReviewService.getReviews as jest.Mock).mockResolvedValue({
      reviews: [],
      averageRating: 0,
      totalReviews: 0,
    });
    
    const { rerender } = render(<ReviewList productId={productId} refreshTrigger={0} />);
    
    await waitFor(() => {
      expect(ReviewService.getReviews).toHaveBeenCalledTimes(1);
    });
    
    rerender(<ReviewList productId={productId} refreshTrigger={1} />);
    
    await waitFor(() => {
      expect(ReviewService.getReviews).toHaveBeenCalledTimes(2);
    });
  });
});
```

---

### End-to-End Test

**Test File:** `storefront/e2e/reviews.spec.ts`

**Coverage:** 3 E2E tests

11. **submit review and verify it appears in Reviews section and aggregates are updated** — Full submission flow
12. **displays error message when submitting empty review** — Validation error flow
13. **prevents duplicate review submission while request is in-flight** — Idempotency flow

```typescript
import { test, expect } from '@playwright/test';

test.describe('Product Reviews', () => {
  test('submit review and verify it appears in Reviews section and aggregates are updated', async ({ page }) => {
    await page.goto('/products/test-product-slug');
    
    await expect(page.locator('h1')).toContainText('Test Product');
    
    await page.locator('text=Customer Reviews').scrollIntoViewIfNeeded();
    
    await page.locator('.star-rating .star').nth(4).click();
    
    await page.locator('textarea#content').fill('This is a test review. Great product!');
    
    await page.locator('button:has-text("Submit Review")').click();
    
    await expect(page.locator('.Toastify__toast--success')).toContainText('Review submitted successfully!');
    
    await page.waitForTimeout(1000);
    
    await expect(page.locator('.review-item')).toContainText('This is a test review. Great product!');
    await expect(page.locator('.review-rating')).toContainText('★★★★★');
    
    await page.locator('.product-header').scrollIntoViewIfNeeded();
    await expect(page.locator('.product-rating .rating-stars')).toContainText('★');
    await expect(page.locator('.product-rating .review-count')).toContainText('review');
    
    const headerReviewCount = await page.locator('.product-rating .review-count').textContent();
    const reviewSectionCount = await page.locator('.review-aggregates h4').textContent();
    expect(headerReviewCount).toContain('1');
    expect(reviewSectionCount).toContain('1');
  });

  test('displays error message when submitting empty review', async ({ page }) => {
    await page.goto('/products/test-product-slug');
    
    await page.locator('text=Customer Reviews').scrollIntoViewIfNeeded();
    
    await page.locator('.star-rating .star').nth(4).click();
    
    await page.locator('button:has-text("Submit Review")').click();
    
    await expect(page.locator('.error-field')).toContainText('Review content is required');
  });

  test('prevents duplicate review submission while request is in-flight', async ({ page }) => {
    await page.goto('/products/test-product-slug');
    
    await page.locator('text=Customer Reviews').scrollIntoViewIfNeeded();
    
    await page.locator('.star-rating .star').nth(4).click();
    await page.locator('textarea#content').fill('Test review for idempotency.');
    
    const submitButton = page.locator('button:has-text("Submit Review")');
    await submitButton.click();
    
    await expect(submitButton).toBeDisabled();
    await expect(submitButton).toContainText('Submitting...');
    
    await expect(page.locator('.Toastify__toast--success')).toContainText('Review submitted successfully!');
    
    await expect(submitButton).not.toBeDisabled();
    await expect(submitButton).toContainText('Submit Review');
  });
});
```

**Total Test Coverage:** 13 tests (6 unit tests for ReviewForm, 4 unit tests for ReviewList, 3 E2E tests)

## Dependencies

- **Story 1 (Backend Review Submission API and Database Schema)** — Backend API endpoints must exist (`POST /api/rating/storefront/products/{productId}/reviews`, `GET /api/rating/storefront/products/{productId}/reviews`)

---

## Out of Scope

- Backend validation logic (covered in Story 1)
- Review editing or deletion by customers (not in ticket scope)
- Review pagination (all reviews displayed; pagination can be added later)
- Backoffice UI for admin review management (covered in Story 3)

---

## Notes

- The storefront frontend uses Next.js and TypeScript, following the existing pattern in `backoffice/pages/` and `backoffice/modules/`
- The API client follows the pattern in `backoffice/common/services/ApiClientService.ts` (using `fetch` with JSON body)
- Toast notifications use `react-toastify`, observed in `backoffice/common/services/ToastService.ts` and `backoffice/pages/reviews/index.tsx`
- Form state management uses `react-hook-form`, observed in `backoffice/common/items/Input.tsx` and `backoffice/modules/catalog/components/ProductGeneralInformation.tsx`
- Date formatting uses `moment`, observed in `backoffice/pages/reviews/index.tsx`
- The product detail page structure (`storefront/pages/products/[slug].tsx`) is assumed to exist based on the backoffice pattern (`backoffice/pages/catalog/products/[id]/edit.tsx`). If it does not exist, create it following Next.js pages structure
- The `ProductHeader` component (`storefront/modules/product/components/ProductHeader.tsx`) is assumed to exist. If it does not exist, integrate aggregates display inline in the product detail page
- Star rating selector uses Unicode characters (`★` for filled, `☆` for empty). CSS classes (`.filled`) control the visual appearance
- Idempotency is enforced at the UI level (disable submit button while request is in-flight) and at the backend level (database unique constraint on `(product_id, customer_id)`). The UI-level prevention improves UX but is not a security guarantee
- Error handling extracts messages from the API response (`detail` or `fieldErrors[].message`) and displays them in toast notifications. Generic messages are used for network errors or missing error details
- The `refreshTrigger` prop in `ReviewList` allows the parent component to trigger a refetch after a new review is submitted. This avoids duplicating the fetch logic in the parent
