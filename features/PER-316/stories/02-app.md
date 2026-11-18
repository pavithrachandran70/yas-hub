# Low-Level Design — Story 02-app

**Epic:** PER-316 (Signed-in shoppers can save a wishlist and add those items to the cart)  
**Story:** Storefront wishlist page  
**Repo:** `app`

---

## Overview

This story implements the UI for the wishlist feature, allowing signed-in shoppers to view their saved items, remove items, and see discontinued product status. The page calls the backend wishlist API (Story 1) to fetch items and handle removals. Guests are redirected to the sign-in page when attempting to access the wishlist. This story delivers the user-facing wishlist experience, enabling shoppers to manage their saved products before Story 3 adds the ability to add them to the cart.

---

## Stack

- **Language/Framework:** TypeScript, Next.js 13+ (App Router), React 18
- **Styling:** CSS Modules (matches existing backoffice pattern in `backoffice/styles/`)
- **API Client:** `fetch` with JWT authentication (matches existing pattern in `backoffice/common/services/ApiClientService.ts`)
- **State Management:** React `useState`, `useEffect` (no external state library)
- **Testing:** Jest, React Testing Library (matches existing pattern in `cart/src/test/`)
- **Existing patterns:** 
  - API service layer (`backoffice/modules/catalog/services/ProductService.ts`)
  - Toast notifications (`backoffice/common/services/ToastService.ts`)
  - Authentication utilities (inferred from backoffice JWT usage)

---

## Acceptance Criteria

### AC-1: Signed-in shopper can view and remove wishlist items

**Given** a signed-in shopper viewing their wishlist  
**When** they remove an item  
**Then** that item is no longer on their wishlist (UI reflects backend removal from Story 1)

---

### AC-4: Discontinued product display

**Given** a signed-in shopper with a discontinued product on their wishlist  
**When** they view their wishlist  
**Then** the discontinued product is displayed with a message indicating it cannot be added to the cart

---

### AC-5: Guest redirect

**Given** a guest (not signed in)  
**When** they attempt to access the wishlist page  
**Then** they are redirected to sign in (UI checks JWT and redirects to `/login?returnUrl=/wishlist`)

---

## File Touch List

### New Files

1. **Page Component:**
   - `storefront/src/app/wishlist/page.tsx`

2. **Child Component:**
   - `storefront/src/app/wishlist/WishlistItem.tsx`

3. **API Client:**
   - `storefront/src/services/wishlistService.ts`

4. **TypeScript Types:**
   - `storefront/src/types/wishlist.ts`

5. **CSS Modules:**
   - `storefront/src/app/wishlist/wishlist.module.css`
   - `storefront/src/app/wishlist/WishlistItem.module.css`

---

### Modified Files

1. **Navigation:**
   - `storefront/src/components/Header.tsx` — Add "Wishlist" link (visible only when signed in)

2. **Product Page:**
   - `storefront/src/app/product/[slug]/page.tsx` — Add "Add to Wishlist" button

3. **Authentication Service (if missing):**
   - `storefront/src/services/authService.ts` — JWT presence check and token retrieval

---

## TypeScript Interfaces

### File: `storefront/src/types/wishlist.ts`

```typescript
export interface WishlistItem {
  wishlistItemId: number;
  productId: number;
  productName: string;
  productSlug: string;
  thumbnailUrl: string | null;
  price: number;
  isAvailable: boolean;
  discontinuedMessage: string | null;
}

export interface WishlistItemListResponse {
  items: WishlistItem[];
}
```

**Rationale:** Matches Story 1 backend contract (`WishlistItemVm` from `customer/src/main/java/com/yas/customer/viewmodel/wishlist/WishlistItemListVm.java`). Field names follow camelCase TypeScript convention, mapping directly to backend JSON response.

---

## API Client

### File: `storefront/src/services/wishlistService.ts`

```typescript
import { WishlistItem, WishlistItemListResponse } from '@/types/wishlist';

const BASE_URL = '/api/customer/storefront/wishlist';

function getAuthHeaders(): HeadersInit {
  const token = getToken();
  if (!token) {
    throw new Error('Authentication required');
  }
  return {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  };
}

function getToken(): string | null {
  // Implementation depends on where JWT is stored
  // Option 1: Cookie (httpOnly=false, accessed via document.cookie)
  // Option 2: localStorage (if using client-side storage)
  // Option 3: Next.js API route that forwards auth (recommended for production)
  if (typeof window === 'undefined') {
    return null;
  }
  return localStorage.getItem('access_token');
}

export async function getWishlistItems(): Promise<WishlistItemListResponse> {
  const response = await fetch(`${BASE_URL}/items`, {
    method: 'GET',
    headers: getAuthHeaders()
  });

  if (!response.ok) {
    if (response.status === 401) {
      throw new Error('UNAUTHORIZED');
    }
    throw new Error(`Failed to fetch wishlist items: ${response.statusText}`);
  }

  return response.json();
}

export async function removeWishlistItem(wishlistItemId: number): Promise<void> {
  const response = await fetch(`${BASE_URL}/items/${wishlistItemId}`, {
    method: 'DELETE',
    headers: getAuthHeaders()
  });

  if (!response.ok) {
    if (response.status === 401) {
      throw new Error('UNAUTHORIZED');
    }
    if (response.status === 404) {
      throw new Error('Wishlist item not found');
    }
    throw new Error(`Failed to remove wishlist item: ${response.statusText}`);
  }
}

export async function addProductToWishlist(productId: number): Promise<void> {
  const response = await fetch(`${BASE_URL}/items`, {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({ productId })
  });

  if (!response.ok) {
    if (response.status === 401) {
      throw new Error('UNAUTHORIZED');
    }
    if (response.status === 404) {
      throw new Error('Product not found');
    }
    throw new Error(`Failed to add product to wishlist: ${response.statusText}`);
  }
}
```

**Notes:**
- **Token Storage:** The `getToken()` function is a placeholder. The actual implementation depends on the storefront's authentication strategy:
  - **Recommended (production):** Use Next.js API route middleware to append JWT from httpOnly cookie server-side.
  - **Fallback (dev):** Read from `localStorage` or non-httpOnly cookie if the app uses client-side auth.
  - **Action Required:** Verify existing auth pattern in `storefront/` and adjust `getToken()` accordingly.
- **Error Handling:** Throws errors with specific messages for 401 (redirect to login) and 404 (item not found).
- **API Base URL:** Uses `/api/customer/storefront/wishlist` (matches Story 1 backend routes).

---

## Authentication Service (if missing)

### File: `storefront/src/services/authService.ts`

```typescript
export function isAuthenticated(): boolean {
  if (typeof window === 'undefined') {
    return false;
  }
  const token = localStorage.getItem('access_token');
  return token !== null && token !== '';
}

export function getToken(): string | null {
  if (typeof window === 'undefined') {
    return null;
  }
  return localStorage.getItem('access_token');
}

export function redirectToLogin(returnUrl: string): void {
  if (typeof window !== 'undefined') {
    window.location.href = `/login?returnUrl=${encodeURIComponent(returnUrl)}`;
  }
}
```

**Notes:**
- **Action Required:** If `storefront/src/services/authService.ts` already exists, verify it has these methods. If method signatures differ, adjust the wishlist page code to match existing patterns.
- **Alternative:** If the app uses a different auth library (e.g., NextAuth.js, Auth0 SDK), replace these functions with equivalent calls to that library's API.

---

## Wishlist Page Component

### File: `storefront/src/app/wishlist/page.tsx`

```typescript
'use client';

import { useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';
import { getWishlistItems, removeWishlistItem } from '@/services/wishlistService';
import { isAuthenticated, redirectToLogin } from '@/services/authService';
import { WishlistItem as WishlistItemType } from '@/types/wishlist';
import WishlistItem from './WishlistItem';
import styles from './wishlist.module.css';

export default function WishlistPage() {
  const router = useRouter();
  const [items, setItems] = useState<WishlistItemType[]>([]);
  const [loading, setLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // AC-5: Guest redirect
    if (!isAuthenticated()) {
      redirectToLogin('/wishlist');
      return;
    }

    fetchWishlistItems();
  }, []);

  async function fetchWishlistItems() {
    setLoading(true);
    setError(null);

    try {
      const response = await getWishlistItems();
      setItems(response.items);
    } catch (err: any) {
      if (err.message === 'UNAUTHORIZED') {
        redirectToLogin('/wishlist');
        return;
      }
      setError(err.message || 'Failed to load wishlist items');
    } finally {
      setLoading(false);
    }
  }

  async function handleRemoveItem(wishlistItemId: number) {
    try {
      await removeWishlistItem(wishlistItemId);
      // AC-1: Remove item from UI after successful backend deletion
      setItems(prevItems => prevItems.filter(item => item.wishlistItemId !== wishlistItemId));
      showToast('Item removed from wishlist', 'success');
    } catch (err: any) {
      if (err.message === 'UNAUTHORIZED') {
        redirectToLogin('/wishlist');
        return;
      }
      showToast(err.message || 'Failed to remove item', 'error');
    }
  }

  function showToast(message: string, type: 'success' | 'error') {
    // Placeholder: Replace with actual toast implementation
    // See backoffice/common/services/ToastService.ts for reference
    console.log(`[${type}] ${message}`);
    alert(message);
  }

  if (loading) {
    return (
      <div className={styles.container}>
        <h1 className={styles.title}>My Wishlist</h1>
        <div className={styles.loading}>Loading...</div>
      </div>
    );
  }

  if (error) {
    return (
      <div className={styles.container}>
        <h1 className={styles.title}>My Wishlist</h1>
        <div className={styles.error}>
          <p>{error}</p>
          <button onClick={fetchWishlistItems} className={styles.retryButton}>
            Retry
          </button>
        </div>
      </div>
    );
  }

  if (items.length === 0) {
    return (
      <div className={styles.container}>
        <h1 className={styles.title}>My Wishlist</h1>
        <div className={styles.emptyState}>
          <p>Your wishlist is empty</p>
          <a href="/" className={styles.continueShoppingLink}>
            Continue Shopping
          </a>
        </div>
      </div>
    );
  }

  return (
    <div className={styles.container}>
      <h1 className={styles.title}>My Wishlist</h1>
      <div className={styles.itemsGrid}>
        {items.map(item => (
          <WishlistItem
            key={item.wishlistItemId}
            item={item}
            onRemove={handleRemoveItem}
          />
        ))}
      </div>
    </div>
  );
}
```

**Algorithm Notes:**
1. **Mount Behavior:**
   - On mount, check `isAuthenticated()`. If false, immediately redirect to `/login?returnUrl=/wishlist` (AC-5).
   - If authenticated, call `getWishlistItems()` to fetch items from Story 1 backend.
2. **Loading State:**
   - Show "Loading..." spinner/skeleton while `loading === true`.
3. **Error Handling:**
   - If `getWishlistItems()` throws `UNAUTHORIZED`, redirect to login.
   - If other error, show error message with "Retry" button.
4. **Empty State:**
   - If `items.length === 0`, show "Your wishlist is empty" with link to continue shopping.
5. **Remove Item:**
   - Call `removeWishlistItem(wishlistItemId)` on backend.
   - On success, remove item from local `items` state (optimistic UI update, AC-1).
   - Show success toast.
   - On error, show error toast without removing item from UI.

---

### CSS Module: `storefront/src/app/wishlist/wishlist.module.css`

```css
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.title {
  font-size: 2rem;
  margin-bottom: 1.5rem;
}

.loading {
  text-align: center;
  padding: 2rem;
  color: #666;
}

.error {
  text-align: center;
  padding: 2rem;
  color: #d32f2f;
}

.retryButton {
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background-color: #1976d2;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.retryButton:hover {
  background-color: #1565c0;
}

.emptyState {
  text-align: center;
  padding: 4rem 2rem;
}

.continueShoppingLink {
  display: inline-block;
  margin-top: 1rem;
  color: #1976d2;
  text-decoration: none;
}

.continueShoppingLink:hover {
  text-decoration: underline;
}

.itemsGrid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 1.5rem;
}
```

---

## Wishlist Item Component

### File: `storefront/src/app/wishlist/WishlistItem.tsx`

```typescript
'use client';

import { WishlistItem as WishlistItemType } from '@/types/wishlist';
import styles from './WishlistItem.module.css';

interface WishlistItemProps {
  item: WishlistItemType;
  onRemove: (wishlistItemId: number) => void;
}

export default function WishlistItem({ item, onRemove }: WishlistItemProps) {
  function handleRemoveClick() {
    onRemove(item.wishlistItemId);
  }

  return (
    <div className={styles.card}>
      {item.thumbnailUrl ? (
        <img
          src={item.thumbnailUrl}
          alt={item.productName}
          className={styles.thumbnail}
        />
      ) : (
        <div className={styles.thumbnailPlaceholder}>No Image</div>
      )}

      <div className={styles.content}>
        <a href={`/product/${item.productSlug}`} className={styles.productName}>
          {item.productName}
        </a>

        <p className={styles.price}>${item.price.toFixed(2)}</p>

        {/* AC-4: Discontinued product display */}
        {!item.isAvailable && item.discontinuedMessage && (
          <p className={styles.discontinuedMessage}>
            {item.discontinuedMessage}
          </p>
        )}

        <button
          onClick={handleRemoveClick}
          className={styles.removeButton}
        >
          Remove
        </button>
      </div>
    </div>
  );
}
```

**Algorithm Notes:**
1. **Thumbnail Display:**
   - If `thumbnailUrl` exists, show image.
   - If null, show placeholder div with "No Image".
2. **Product Link:**
   - Product name links to `/product/{productSlug}`.
3. **Price Display:**
   - Format price with 2 decimal places (e.g., `$29.99`).
4. **Discontinued Message (AC-4):**
   - If `isAvailable === false` AND `discontinuedMessage` is not null, show message in red text.
   - Story 3 will add "Add to Cart" button here; for now, only show remove button.
5. **Remove Button:**
   - Calls `onRemove(item.wishlistItemId)` callback.

---

### CSS Module: `storefront/src/app/wishlist/WishlistItem.module.css`

```css
.card {
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.thumbnail {
  width: 100%;
  height: 200px;
  object-fit: cover;
  border-radius: 4px;
}

.thumbnailPlaceholder {
  width: 100%;
  height: 200px;
  background-color: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #999;
  border-radius: 4px;
}

.content {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.productName {
  font-size: 1rem;
  font-weight: 500;
  color: #333;
  text-decoration: none;
}

.productName:hover {
  text-decoration: underline;
}

.price {
  font-size: 1.25rem;
  font-weight: bold;
  color: #1976d2;
  margin: 0;
}

.discontinuedMessage {
  color: #d32f2f;
  font-size: 0.875rem;
  margin: 0;
}

.removeButton {
  margin-top: 0.5rem;
  padding: 0.5rem 1rem;
  background-color: #f44336;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.875rem;
}

.removeButton:hover {
  background-color: #d32f2f;
}
```

---

## Header Modification

### File: `storefront/src/components/Header.tsx`

**Modification:** Add wishlist link next to cart link, visible only when signed in.

```typescript
'use client';

import { useEffect, useState } from 'react';
import { isAuthenticated } from '@/services/authService';

export default function Header() {
  const [signedIn, setSignedIn] = useState<boolean>(false);

  useEffect(() => {
    setSignedIn(isAuthenticated());
  }, []);

  return (
    <header>
      {/* Existing navigation */}
      <nav>
        <a href="/">Home</a>
        <a href="/products">Products</a>

        {signedIn && (
          <>
            <a href="/wishlist">Wishlist</a>
            <a href="/cart">Cart</a>
            <a href="/account">My Account</a>
          </>
        )}

        {!signedIn && (
          <a href="/login">Sign In</a>
        )}
      </nav>
    </header>
  );
}
```

**Notes:**
- **Action Required:** This is a minimal example. If `Header.tsx` already exists, locate the cart link and add the wishlist link immediately before or after it.
- **Conditional Rendering:** Wishlist link only shows when `isAuthenticated()` returns true (same logic as "My Account" link).
- **Styling:** Match existing header link styles.

---

## Product Page Modification

### File: `storefront/src/app/product/[slug]/page.tsx`

**Modification:** Add "Add to Wishlist" button next to "Add to Cart" button.

```typescript
'use client';

import { useState } from 'react';
import { addProductToWishlist } from '@/services/wishlistService';
import { isAuthenticated, redirectToLogin } from '@/services/authService';

export default function ProductPage({ params }: { params: { slug: string } }) {
  const [loading, setLoading] = useState<boolean>(false);

  // Existing code: fetch product details, etc.

  async function handleAddToWishlist(productId: number) {
    // AC-5: Guest redirect
    if (!isAuthenticated()) {
      redirectToLogin(`/product/${params.slug}`);
      return;
    }

    setLoading(true);
    try {
      await addProductToWishlist(productId);
      showToast('Product added to wishlist', 'success');
    } catch (err: any) {
      if (err.message === 'UNAUTHORIZED') {
        redirectToLogin(`/product/${params.slug}`);
        return;
      }
      showToast(err.message || 'Failed to add to wishlist', 'error');
    } finally {
      setLoading(false);
    }
  }

  function showToast(message: string, type: 'success' | 'error') {
    // Placeholder: Replace with actual toast implementation
    console.log(`[${type}] ${message}`);
    alert(message);
  }

  return (
    <div>
      {/* Existing product detail JSX */}

      <div>
        <button onClick={() => handleAddToCart(productId)}>
          Add to Cart
        </button>

        <button
          onClick={() => handleAddToWishlist(productId)}
          disabled={loading}
        >
          {loading ? 'Adding...' : 'Add to Wishlist'}
        </button>
      </div>
    </div>
  );
}
```

**Notes:**
- **Action Required:** Locate the "Add to Cart" button in the existing product page component and add the "Add to Wishlist" button alongside it.
- **Guest Handling:** If user is not signed in, redirect to `/login?returnUrl=/product/{slug}` before attempting to add to wishlist.
- **Loading State:** Disable button and show "Adding..." text while request is in flight.
- **Toast Notifications:** Replace placeholder `showToast()` with actual toast implementation (see `backoffice/common/services/ToastService.ts` for reference).

---

## Tests

### File: `storefront/__tests__/wishlist/page.test.tsx`

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import WishlistPage from '@/app/wishlist/page';
import * as wishlistService from '@/services/wishlistService';
import * as authService from '@/services/authService';

jest.mock('@/services/wishlistService');
jest.mock('@/services/authService');
jest.mock('next/navigation', () => ({
  useRouter: () => ({ push: jest.fn() })
}));

describe('WishlistPage', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('testWishlistPage_GuestRedirect - redirects guest to login (AC-5)', () => {
    const mockRedirectToLogin = jest.fn();
    (authService.isAuthenticated as jest.Mock).mockReturnValue(false);
    (authService.redirectToLogin as jest.Mock).mockImplementation(mockRedirectToLogin);

    render(<WishlistPage />);

    expect(mockRedirectToLogin).toHaveBeenCalledWith('/wishlist');
  });

  test('testWishlistPage_LoadsItems - renders items for signed-in user', async () => {
    (authService.isAuthenticated as jest.Mock).mockReturnValue(true);
    (wishlistService.getWishlistItems as jest.Mock).mockResolvedValue({
      items: [
        {
          wishlistItemId: 1,
          productId: 101,
          productName: 'Product A',
          productSlug: 'product-a',
          thumbnailUrl: 'https://example.com/a.jpg',
          price: 29.99,
          isAvailable: true,
          discontinuedMessage: null
        },
        {
          wishlistItemId: 2,
          productId: 102,
          productName: 'Product B',
          productSlug: 'product-b',
          thumbnailUrl: 'https://example.com/b.jpg',
          price: 49.99,
          isAvailable: true,
          discontinuedMessage: null
        }
      ]
    });

    render(<WishlistPage />);

    await waitFor(() => {
      expect(screen.getByText('Product A')).toBeInTheDocument();
      expect(screen.getByText('Product B')).toBeInTheDocument();
    });
  });

  test('testWishlistPage_EmptyState - shows empty message when no items', async () => {
    (authService.isAuthenticated as jest.Mock).mockReturnValue(true);
    (wishlistService.getWishlistItems as jest.Mock).mockResolvedValue({ items: [] });

    render(<WishlistPage />);

    await waitFor(() => {
      expect(screen.getByText('Your wishlist is empty')).toBeInTheDocument();
      expect(screen.getByText('Continue Shopping')).toBeInTheDocument();
    });
  });

  test('testWishlistPage_RemoveItem - removes item from list on delete (AC-1)', async () => {
    (authService.isAuthenticated as jest.Mock).mockReturnValue(true);
    (wishlistService.getWishlistItems as jest.Mock).mockResolvedValue({
      items: [
        {
          wishlistItemId: 1,
          productId: 101,
          productName: 'Product A',
          productSlug: 'product-a',
          thumbnailUrl: 'https://example.com/a.jpg',
          price: 29.99,
          isAvailable: true,
          discontinuedMessage: null
        }
      ]
    });
    (wishlistService.removeWishlistItem as jest.Mock).mockResolvedValue(undefined);

    render(<WishlistPage />);

    await waitFor(() => {
      expect(screen.getByText('Product A')).toBeInTheDocument();
    });

    const removeButton = screen.getByText('Remove');
    await userEvent.click(removeButton);

    await waitFor(() => {
      expect(wishlistService.removeWishlistItem).toHaveBeenCalledWith(1);
      expect(screen.queryByText('Product A')).not.toBeInTheDocument();
    });
  });

  test('testWishlistPage_ErrorState - shows error message and retry button', async () => {
    (authService.isAuthenticated as jest.Mock).mockReturnValue(true);
    (wishlistService.getWishlistItems as jest.Mock).mockRejectedValue(
      new Error('Network error')
    );

    render(<WishlistPage />);

    await waitFor(() => {
      expect(screen.getByText('Network error')).toBeInTheDocument();
      expect(screen.getByText('Retry')).toBeInTheDocument();
    });
  });
});
```

---

### File: `storefront/__tests__/wishlist/WishlistItem.test.tsx`

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import WishlistItem from '@/app/wishlist/WishlistItem';
import { WishlistItem as WishlistItemType } from '@/types/wishlist';

describe('WishlistItem', () => {
  const mockOnRemove = jest.fn();

  afterEach(() => {
    jest.clearAllMocks();
  });

  test('testWishlistItem_AvailableProduct - renders available product correctly', () => {
    const item: WishlistItemType = {
      wishlistItemId: 1,
      productId: 101,
      productName: 'Product A',
      productSlug: 'product-a',
      thumbnailUrl: 'https://example.com/a.jpg',
      price: 29.99,
      isAvailable: true,
      discontinuedMessage: null
    };

    render(<WishlistItem item={item} onRemove={mockOnRemove} />);

    expect(screen.getByText('Product A')).toBeInTheDocument();
    expect(screen.getByText('$29.99')).toBeInTheDocument();
    expect(screen.getByText('Remove')).toBeInTheDocument();
    expect(screen.getByAltText('Product A')).toHaveAttribute('src', item.thumbnailUrl);
    expect(screen.queryByText(/no longer available/i)).not.toBeInTheDocument();
  });

  test('testWishlistItem_DiscontinuedProduct - displays discontinued message (AC-4)', () => {
    const item: WishlistItemType = {
      wishlistItemId: 2,
      productId: 102,
      productName: 'Product B',
      productSlug: 'product-b',
      thumbnailUrl: 'https://example.com/b.jpg',
      price: 49.99,
      isAvailable: false,
      discontinuedMessage: 'This product is no longer available'
    };

    render(<WishlistItem item={item} onRemove={mockOnRemove} />);

    expect(screen.getByText('Product B')).toBeInTheDocument();
    expect(screen.getByText('This product is no longer available')).toBeInTheDocument();
  });

  test('testWishlistItem_RemoveButton - calls onRemove with correct ID', async () => {
    const item: WishlistItemType = {
      wishlistItemId: 1,
      productId: 101,
      productName: 'Product A',
      productSlug: 'product-a',
      thumbnailUrl: 'https://example.com/a.jpg',
      price: 29.99,
      isAvailable: true,
      discontinuedMessage: null
    };

    render(<WishlistItem item={item} onRemove={mockOnRemove} />);

    const removeButton = screen.getByText('Remove');
    await userEvent.click(removeButton);

    expect(mockOnRemove).toHaveBeenCalledWith(1);
  });
});
```

---

## Tests Section

This section consolidates all test cases required to verify the acceptance criteria for Story 02-app.

### Test Cases Mapping

**AC-1 (View and Remove):**
- `testWishlistPage_LoadsItems` — Verifies items are displayed
- `testWishlistPage_RemoveItem` — Verifies item removal updates UI

**AC-4 (Discontinued Display):**
- `testWishlistItem_DiscontinuedProduct` — Verifies discontinued message appears

**AC-5 (Guest Redirect):**
- `testWishlistPage_GuestRedirect` — Verifies redirect to login

**Additional Coverage:**
- `testWishlistPage_EmptyState` — Verifies empty state display
- `testWishlistPage_ErrorState` — Verifies error handling
- `testWishlistItem_AvailableProduct` — Verifies available product display
- `testWishlistItem_RemoveButton` — Verifies remove callback

**Total Test Count:** 8 tests (5 in page.test.tsx + 3 in WishlistItem.test.tsx)

**Verification Count from Breakdown:** The breakdown specified 8 unit tests for Story 2. This matches exactly.

---

## Open Questions

1. **Authentication Strategy:** How does the storefront store and retrieve JWT tokens? The placeholder `getToken()` function in `wishlistService.ts` assumes `localStorage`. Verify actual implementation.
   - **Resolution path:** Inspect existing API service files in `storefront/src/services/` or `storefront/src/utils/`. Adjust `getToken()` to match.

2. **Toast Notification System:** Does the storefront have a toast notification library (e.g., `react-toastify`, `notistack`) or custom toast component? The placeholder `showToast()` function uses `alert()`.
   - **Resolution path:** Inspect existing UI for toast/snackbar implementations. If using a library, install it and replace placeholder. If custom, import and use the custom toast component.

3. **Header Component Location:** The file path `storefront/src/components/Header.tsx` is assumed. Verify actual location of the header component.
   - **Resolution path:** Search for `Header.tsx` or `header.tsx` in `storefront/src/`. Adjust file path in this LLD if located elsewhere.

4. **Product Page Component Location:** The file path `storefront/src/app/product/[slug]/page.tsx` is assumed based on Next.js App Router convention. Verify actual location.
   - **Resolution path:** Search for the product detail page component. If using Pages Router (`pages/product/[slug].tsx`), adjust modification instructions accordingly.

---

## Notes

- **Idempotency:** Add-to-wishlist from product page is idempotent (backend returns 200 if product already on wishlist). UI shows success toast in both cases.
- **Error Handling:** All API errors return descriptive messages. 401 errors trigger redirect to login with return URL.
- **Empty State:** "Your wishlist is empty" message includes link to continue shopping (homepage).
- **Loading State:** Show spinner/skeleton while fetching items to improve perceived performance.
- **Styling:** CSS Modules follow existing backoffice pattern. Adjust class names and styles to match storefront design system.
- **Testing:** Unit tests cover all acceptance criteria. Integration/E2E test for full user journey (add to wishlist → view → remove) is recommended but out of scope for this story's unit test count.
- **Accessibility:** Ensure buttons have proper ARIA labels and keyboard navigation works (not explicitly covered in AC but should be tested manually).
- **Test Files:** All test code is written once in the Tests section above. The File Touch List names the test files but does not repeat the test code.
