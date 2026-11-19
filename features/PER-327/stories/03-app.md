# Story 03-app LLD: Reviews section display updates

**Repo:** `app`

---

## Overview

This LLD details the implementation of the reviews section display for story 03-app. It covers the React components for displaying paginated review lists, updating the product detail header with aggregate statistics, and ensuring consistency between all display locations. The implementation consumes the backend endpoints from Story 1 (`GET /api/rating/storefront/reviews` and `GET /api/product/storefront/products/{productId}`) and integrates with Story 2's review submission trigger.

---

## Acceptance Criteria (from requirements/breakdown)

6. The newly submitted review must appear in the product's Reviews section with the correct rating and review content. (AC-6 — `<ReviewsList />` fetches and displays reviews, including newly submitted one after parent's `reviewsKey` changes)
9. The rating and review count displayed in the product header must match the Reviews section. (AC-9 — both read from `product.averageRating` and `product.reviewCount` fetched from product service, single source of truth)
12. Submitted reviews must remain available after refreshing or revisiting the product page. (AC-12 — on page load, `products/[id].tsx` fetches product data and reviews from backend; reviews persisted in database, displayed on every load)

---

## File Plan

### New Files

1. **`storefront/modules/review/components/ReviewsList.tsx`**
   - Component: renders paginated list of reviews
   - Each review displays: rating (star graphic), content text, timestamp (formatted)
   - Pagination controls: Previous/Next buttons, page indicator
   - Depends on: `ReviewService`, `Review` types, existing styling patterns

2. **`storefront/modules/review/components/ReviewsList.test.tsx`**
   - Component tests for review rendering, pagination, loading/error states
   - 6 test cases (see Tests section)
   - Depends on: React Testing Library, `ReviewsList` component, mock `ReviewService`

### Modified Files

1. **`storefront/pages/products/[id].tsx`** (or exact path — verify in repo)
   - Add import: `import ReviewsList from '../modules/review/components/ReviewsList';`
   - Existing state from Story 2: `const [reviewsKey, setReviewsKey] = useState(0);`
   - Add state: `const [product, setProduct] = useState<Product | null>(null);`
   - Add effect: fetch product data on mount and when `reviewsKey` changes
   - Render `<ReviewsList productId={product.id} key={reviewsKey} />` in reviews section

2. **`storefront/modules/catalog/components/ProductDetailHeader.tsx`** (or similar — verify exact component in repo)
   - Add props: `averageRating?: number`, `reviewCount?: number` (or access from `product` object)
   - Render rating display:
     - Star graphic: show `averageRating` as filled stars (rounded to nearest 0.5 or exact value)
     - Numeric value: `{averageRating.toFixed(1)}` (e.g. "4.3")
     - Review count: `({reviewCount} reviews)` or `({reviewCount} review)` if singular
   - Handle null/undefined: display "No reviews yet" or hide rating section

**Note:** The repo-tree shows no `storefront` directory under `app`, suggesting the storefront may be a separate Next.js app not included in the `app` repo. If so, this story's implementation belongs in the `storefront` repo (not visible in provided context). **Assumption:** The `storefront` directory exists under `app` repo but is not shown in the provided tree. If this assumption is incorrect, clarify the storefront repo location before implementation.

---

## Tests

### Component Tests (6 tests)

**File:** `storefront/modules/review/components/ReviewsList.test.tsx`

**Setup:**
- Use React Testing Library (`@testing-library/react`)
- Mock `ReviewService.getReviews` using Jest or Vitest
- Render `<ReviewsList productId={123} />`

1. **`renders loading spinner while fetching`**
   - **Given:** Mock `getReviews` with delay
   - **When:** Render `<ReviewsList productId={123} />`
   - **Then:** Verify loading spinner displayed, then reviews after delay
   - **Covers:** Loading state display

2. **`displays error message on fetch failure`**
   - **Given:** Mock `getReviews` returns rejected promise
   - **When:** Render `<ReviewsList productId={123} />`
   - **Then:** Verify error message "Failed to load reviews." displayed
   - **Covers:** Error handling (AC-10 indirectly — no crash on error)

3. **`displays "No reviews yet" if reviews array empty`**
   - **Given:** Mock `getReviews` returns empty array
   - **When:** Render `<ReviewsList productId={123} />`
   - **Then:** Verify "No reviews yet. Be the first to review!" message displayed
   - **Covers:** Empty state handling

4. **`renders reviews with correct rating and content`**
   - **Given:** Mock `getReviews` returns 2 reviews (rating 4 and 5)
   - **When:** Render `<ReviewsList productId={123} />`
   - **Then:** Verify 2 review items rendered with correct star count (4 filled + 1 empty, 5 filled + 0 empty) and content text
   - **Covers:** AC-6 (newly submitted review appears with correct rating and content)

5. **`pagination buttons update pageNo and refetch`**
   - **Given:** Mock `getReviews` returns different data for each page (3 pages total)
   - **When:** Render, wait for load, click "Next" button
   - **Then:** Verify `getReviews` called with pageNo=0 initially, then pageNo=1 after click; verify page indicator shows "Page 2 of 3"; verify "Previous" button enabled after navigation
   - **Covers:** Pagination functionality

6. **`reviews persist after component remount`**
   - **Given:** Mock `getReviews` returns review list
   - **When:** Render `<ReviewsList productId={123} key={0} />`, then render again with `key={1}` (simulates parent's reviewsKey increment)
   - **Then:** Verify `getReviews` called twice (once per mount), reviews displayed after each mount
   - **Covers:** AC-12 (submitted reviews remain available after refresh — component remount simulates page refresh behavior)

**File:** `storefront/modules/catalog/components/ProductDetailHeader.test.tsx`

**Setup:**
- Use React Testing Library
- Render `<ProductDetailHeader product={mockProduct} />` with different `mockProduct` data

**Note:** This test file may already exist. Add the following test to it (or create the file if it doesn't exist).

---

## Implementation Code

### New Files

#### 1. ReviewsList.tsx

```tsx
// storefront/modules/review/components/ReviewsList.tsx

import { useState, useEffect } from 'react';
import { getReviews } from '../services/ReviewService';
import { Review } from '../models/Review';

type ReviewsListProps = {
  productId: number;
};

const DEFAULT_PAGE_SIZE = 10;

export default function ReviewsList({ productId }: ReviewsListProps) {
  const [reviews, setReviews] = useState<Review[]>([]);
  const [pageNo, setPageNo] = useState<number>(0);
  const [totalPages, setTotalPages] = useState<number>(0);
  const [loading, setLoading] = useState<boolean>(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetchReviews();
  }, [productId, pageNo]);

  const fetchReviews = async () => {
    setLoading(true);
    setError(null);
    try {
      const response = await getReviews(productId, pageNo, DEFAULT_PAGE_SIZE);
      setReviews(response.reviews);
      setTotalPages(response.totalPages);
      setLoading(false);
    } catch (err: any) {
      setError('Failed to load reviews.');
      setLoading(false);
      console.error('Failed to load reviews:', err);
    }
  };

  const handlePreviousPage = () => {
    if (pageNo > 0) {
      setPageNo(pageNo - 1);
    }
  };

  const handleNextPage = () => {
    if (pageNo < totalPages - 1) {
      setPageNo(pageNo + 1);
    }
  };

  const formatDate = (isoString: string): string => {
    const date = new Date(isoString);
    return date.toLocaleDateString('en-US');
  };

  const renderStars = (rating: number): JSX.Element => {
    return (
      <div className="star-rating">
        {[1, 2, 3, 4, 5].map((star) => (
          <span key={star} className={star <= rating ? 'star-filled' : 'star-empty'}>
            ★
          </span>
        ))}
      </div>
    );
  };

  if (loading) {
    return (
      <div className="reviews-list-loading">
        <div className="spinner-border" role="status">
          <span className="sr-only">Loading reviews...</span>
        </div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="alert alert-danger" role="alert">
        {error}
      </div>
    );
  }

  if (reviews.length === 0) {
    return (
      <div className="no-reviews-message">
        <p>No reviews yet. Be the first to review!</p>
      </div>
    );
  }

  return (
    <div className="reviews-list">
      <h3>Customer Reviews</h3>
      <ul className="review-items">
        {reviews.map((review) => (
          <li key={review.id} className="review-item">
            <div className="review-rating">{renderStars(review.rating)}</div>
            <p className="review-content">{review.content}</p>
            <span className="review-date">{formatDate(review.createdOn)}</span>
          </li>
        ))}
      </ul>

      {totalPages > 1 && (
        <div className="pagination-controls">
          <button onClick={handlePreviousPage} disabled={pageNo === 0}>
            Previous
          </button>
          <span className="page-indicator">
            Page {pageNo + 1} of {totalPages}
          </span>
          <button onClick={handleNextPage} disabled={pageNo === totalPages - 1}>
            Next
          </button>
        </div>
      )}
    </div>
  );
}
```

---

#### 2. ReviewsList.test.tsx

```tsx
// storefront/modules/review/components/ReviewsList.test.tsx

import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import ReviewsList from './ReviewsList';
import * as ReviewService from '../services/ReviewService';

jest.mock('../services/ReviewService');

describe('ReviewsList', () => {
  const mockGetReviews = ReviewService.getReviews as jest.MockedFunction<typeof ReviewService.getReviews>;

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('renders loading spinner while fetching', async () => {
    mockGetReviews.mockImplementation(
      () => new Promise(resolve => setTimeout(() => resolve({
        reviews: [{ id: 1, customerId: 'user-1', rating: 5, content: 'Great!', createdOn: '2025-05-23T10:00:00Z' }],
        totalPages: 1,
        totalElements: 1,
        pageNo: 0,
        pageSize: 10
      }), 1000))
    );

    render(<ReviewsList productId={123} />);

    expect(screen.getByRole('status')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.queryByRole('status')).not.toBeInTheDocument();
      expect(screen.getByText('Great!')).toBeInTheDocument();
    });
  });

  test('displays error message on fetch failure', async () => {
    mockGetReviews.mockRejectedValue(new Error('Network error'));

    render(<ReviewsList productId={123} />);

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent('Failed to load reviews.');
    });

    expect(screen.queryByText('Great!')).not.toBeInTheDocument();
  });

  test('displays "No reviews yet" if reviews array empty', async () => {
    mockGetReviews.mockResolvedValue({
      reviews: [],
      totalPages: 0,
      totalElements: 0,
      pageNo: 0,
      pageSize: 10
    });

    render(<ReviewsList productId={123} />);

    await waitFor(() => {
      expect(screen.getByText('No reviews yet. Be the first to review!')).toBeInTheDocument();
    });

    expect(screen.queryByRole('list')).not.toBeInTheDocument();
  });

  test('renders reviews with correct rating and content', async () => {
    mockGetReviews.mockResolvedValue({
      reviews: [
        { id: 1, customerId: 'user-1', rating: 4, content: 'Good product', createdOn: '2025-05-23T10:00:00Z' },
        { id: 2, customerId: 'user-2', rating: 5, content: 'Excellent!', createdOn: '2025-05-22T14:30:00Z' }
      ],
      totalPages: 1,
      totalElements: 2,
      pageNo: 0,
      pageSize: 10
    });

    render(<ReviewsList productId={123} />);

    await waitFor(() => {
      expect(screen.getByText('Good product')).toBeInTheDocument();
      expect(screen.getByText('Excellent!')).toBeInTheDocument();
    });

    const reviewItems = screen.getAllByRole('listitem');
    expect(reviewItems).toHaveLength(2);

    const firstReviewStars = reviewItems[0].querySelectorAll('.star-filled');
    expect(firstReviewStars).toHaveLength(4);

    const secondReviewStars = reviewItems[1].querySelectorAll('.star-filled');
    expect(secondReviewStars).toHaveLength(5);
  });

  test('pagination buttons update pageNo and refetch', async () => {
    mockGetReviews
      .mockResolvedValueOnce({
        reviews: [{ id: 1, customerId: 'user-1', rating: 5, content: 'Review 1', createdOn: '2025-05-23T10:00:00Z' }],
        totalPages: 3,
        totalElements: 25,
        pageNo: 0,
        pageSize: 10
      })
      .mockResolvedValueOnce({
        reviews: [{ id: 2, customerId: 'user-2', rating: 4, content: 'Review 2', createdOn: '2025-05-22T10:00:00Z' }],
        totalPages: 3,
        totalElements: 25,
        pageNo: 1,
        pageSize: 10
      });

    render(<ReviewsList productId={123} />);

    await waitFor(() => {
      expect(screen.getByText('Review 1')).toBeInTheDocument();
    });

    expect(mockGetReviews).toHaveBeenCalledWith(123, 0, 10);

    const previousButton = screen.getByText('Previous');
    expect(previousButton).toBeDisabled();

    const nextButton = screen.getByText('Next');
    fireEvent.click(nextButton);

    await waitFor(() => {
      expect(mockGetReviews).toHaveBeenCalledWith(123, 1, 10);
      expect(screen.getByText('Page 2 of 3')).toBeInTheDocument();
      expect(screen.getByText('Review 2')).toBeInTheDocument();
    });

    expect(previousButton).not.toBeDisabled();
  });

  test('reviews persist after component remount', async () => {
    mockGetReviews.mockResolvedValue({
      reviews: [{ id: 1, customerId: 'user-1', rating: 5, content: 'Persisted review', createdOn: '2025-05-23T10:00:00Z' }],
      totalPages: 1,
      totalElements: 1,
      pageNo: 0,
      pageSize: 10
    });

    const { rerender } = render(<ReviewsList productId={123} key={0} />);

    await waitFor(() => {
      expect(screen.getByText('Persisted review')).toBeInTheDocument();
    });

    expect(mockGetReviews).toHaveBeenCalledTimes(1);

    rerender(<ReviewsList productId={123} key={1} />);

    await waitFor(() => {
      expect(screen.getByText('Persisted review')).toBeInTheDocument();
    });

    expect(mockGetReviews).toHaveBeenCalledTimes(2);
  });
});
```

---

### Modified Files

#### products/[id].tsx (Integration)

```tsx
// storefront/pages/products/[id].tsx
// (or exact path — verify in repo)

import { useEffect, useState } from 'react';
import { useRouter } from 'next/router';
import ReviewForm from '../modules/review/components/ReviewForm';
import ReviewsList from '../modules/review/components/ReviewsList';
import ProductDetailHeader from '../modules/catalog/components/ProductDetailHeader';
import { getProduct } from '../modules/catalog/services/ProductService';
import { Product } from '../modules/catalog/models/Product';

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
      const productData = await getProduct(productId);
      setProduct(productData);
      setLoading(false);
    } catch (err: any) {
      setError('Failed to load product.');
      setLoading(false);
      console.error('Failed to load product:', err);
    }
  };

  const handleReviewSubmitted = () => {
    setReviewsKey(prev => prev + 1);
  };

  if (loading) {
    return <div>Loading product...</div>;
  }

  if (error || !product) {
    return <div className="alert alert-danger">{error || 'Product not found'}</div>;
  }

  return (
    <div className="product-detail-page">
      <ProductDetailHeader product={product} />

      {/* ... other product sections (description, images, etc.) */}

      {/* Reviews section */}
      <div className="product-reviews-section">
        <ReviewForm productId={product.id} onSuccess={handleReviewSubmitted} />
        <ReviewsList productId={product.id} key={reviewsKey} />
      </div>
    </div>
  );
}
```

---

#### ProductDetailHeader.tsx (Rating Display)

```tsx
// storefront/modules/catalog/components/ProductDetailHeader.tsx
// (or similar — verify exact component in repo)

type ProductDetailHeaderProps = {
  product: Product;
};

export default function ProductDetailHeader({ product }: ProductDetailHeaderProps) {
  const renderStars = (averageRating: number): JSX.Element => {
    const roundedRating = Math.round(averageRating * 2) / 2;
    const fullStars = Math.floor(roundedRating);
    const hasHalfStar = (roundedRating % 1) !== 0;
    const emptyStars = 5 - fullStars - (hasHalfStar ? 1 : 0);

    return (
      <div className="star-rating">
        {Array.from({ length: fullStars }).map((_, i) => (
          <span key={`full-${i}`} className="star-filled">★</span>
        ))}
        {hasHalfStar && <span className="star-half">☆</span>}
        {Array.from({ length: emptyStars }).map((_, i) => (
          <span key={`empty-${i}`} className="star-empty">☆</span>
        ))}
      </div>
    );
  };

  return (
    <div className="product-detail-header">
      <h1>{product.name}</h1>
      <p className="product-price">${product.price}</p>

      {/* Rating and review count section (NEW) */}
      {product.averageRating !== undefined && product.reviewCount !== undefined ? (
        <div className="product-rating-section">
          {renderStars(product.averageRating)}
          <span className="numeric-rating">{product.averageRating.toFixed(1)}</span>
          <span className="review-count">
            ({product.reviewCount} {product.reviewCount === 1 ? 'review' : 'reviews'})
          </span>
        </div>
      ) : (
        <div className="no-reviews-message">No reviews yet</div>
      )}

      {/* ... other product details */}
    </div>
  );
}
```

---

## Styling (Suggested CSS)

```css
/* storefront/styles/ReviewsList.module.css (or global CSS) */

.reviews-list {
  margin: 2rem 0;
}

.reviews-list h3 {
  margin-bottom: 1rem;
  font-size: 1.5rem;
  color: #333;
}

.reviews-list-loading {
  display: flex;
  justify-content: center;
  padding: 2rem;
}

.no-reviews-message {
  padding: 2rem;
  text-align: center;
  color: #666;
  background: #f9f9f9;
  border-radius: 8px;
}

.review-items {
  list-style: none;
  padding: 0;
  margin: 0;
}

.review-item {
  padding: 1.5rem;
  margin-bottom: 1rem;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fff;
}

.review-rating {
  margin-bottom: 0.5rem;
}

.review-content {
  margin: 0.75rem 0;
  font-size: 1rem;
  line-height: 1.5;
  color: #333;
}

.review-date {
  font-size: 0.875rem;
  color: #999;
}

.pagination-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
  margin-top: 2rem;
}

.pagination-controls button {
  padding: 0.5rem 1rem;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background 0.2s;
}

.pagination-controls button:hover:not(:disabled) {
  background: #0056b3;
}

.pagination-controls button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.page-indicator {
  font-weight: 600;
  color: #333;
}

.star-rating {
  display: inline-flex;
  gap: 2px;
}

.star-filled {
  color: #FFD700;
  font-size: 1.2rem;
}

.star-empty {
  color: #D3D3D3;
  font-size: 1.2rem;
}

.star-half {
  color: #FFD700;
  font-size: 1.2rem;
}

.product-rating-section {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-top: 0.5rem;
}

.numeric-rating {
  font-weight: bold;
  font-size: 1.1rem;
}

.review-count {
  color: #666;
  font-size: 0.95rem;
}
```

---

## Implementation Notes

### Critical Path Items

1. **Verify storefront location:** Confirm whether `storefront` is a separate repo or under `app` repo but not shown in tree.

2. **Verify Product type fields:** Check if `storefront/modules/catalog/models/Product.ts` already includes `averageRating?: number` and `reviewCount?: number` fields. If not, add them.

3. **Verify ProductService.getProduct method:** Check if `storefront/modules/catalog/services/ProductService.ts` has a `getProduct(productId: number)` method. If not, implement it.

4. **Verify backend endpoint response:** Ensure `GET /api/product/storefront/products/{productId}` returns `averageRating` and `reviewCount` in the response. If not, update Story 1 backend implementation.

5. **Verify test framework:** Check `storefront/package.json` for testing dependencies (Jest, Vitest, React Testing Library). If missing, install before writing tests.

---

**End of Story 03-app LLD**
