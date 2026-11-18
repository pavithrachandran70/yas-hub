# Requirements — PER-316

**Ticket:** Signed-in shoppers can save a wishlist and add those items to the cart

---

## Overview

A signed-in shopper can save products to a personal wishlist, view that list across sessions, remove items, and add saved items to the cart. The wishlist persists after sign-out and sign-in. Guests have no wishlist and are prompted to sign in before saving. Adding a product already on the list does not duplicate it. Adding a saved item to the cart leaves it on the wishlist. Discontinued products remain visible on the wishlist but cannot be added to the cart.

---

## Acceptance Criteria

### AC-1: Signed-in shopper can add, view, and remove wishlist items across sessions

**Given** a signed-in shopper  
**When** they add a product to their wishlist  
**Then** the product appears on their wishlist immediately

**Given** a signed-in shopper with items on their wishlist  
**When** they sign out and sign back in  
**Then** their wishlist still contains the same items

**Given** a signed-in shopper viewing their wishlist  
**When** they remove an item  
**Then** that item is no longer on their wishlist

---

### AC-2: The same product cannot appear twice on one shopper's wishlist

**Given** a signed-in shopper  
**When** they add a product already on their wishlist  
**Then** the wishlist does not create a duplicate entry  
**And** the existing entry remains unchanged

---

### AC-3: Adding a saved item to the cart leaves it on the wishlist

**Given** a signed-in shopper with a product on their wishlist  
**When** they add that product to the cart  
**Then** the product is added to the cart  
**And** the product remains on the wishlist

---

### AC-4: Discontinued products stay on the wishlist but cannot be added to the cart

**Given** a signed-in shopper with a discontinued product on their wishlist  
**When** they view their wishlist  
**Then** the discontinued product is displayed  
**And** the UI indicates the product cannot be added to the cart

**Given** a signed-in shopper viewing a discontinued product on their wishlist  
**When** they attempt to add it to the cart  
**Then** the add-to-cart action is prevented  
**And** the shopper sees an explanation that the product is no longer available

---

### AC-5: Guests are prompted to sign in before saving to a wishlist

**Given** a guest (not signed in)  
**When** they attempt to add a product to a wishlist  
**Then** they are redirected to sign in  
**And** no wishlist data is saved

---

## Scope

This ticket encompasses:

1. **Persistence layer:** Store wishlist items tied to a customer across sessions
2. **Wishlist page UI:** Display saved items, allow removal, show discontinued product state
3. **Add-to-cart integration:** Add a wishlist item to the cart while leaving it on the wishlist
4. **Guest handling:** Redirect guests to sign-in before wishlist operations
5. **Duplicate prevention:** Ensure a product appears at most once per customer's wishlist
6. **Discontinued product handling:** Query and display product availability state; prevent cart adds for discontinued products

---

## Cross-Domain Interactions

- **Customer domain:** Owns the wishlist data (which customer, which products)
- **Product domain:** Provides product details and availability status
- **Cart domain:** Receives items added from the wishlist

The persistence contract identifies a saved item by customer ID and product ID.

---

## Out of Scope

- Wishlist sharing, collaborative lists, or multi-user access
- Wishlist item quantity or variants (size, color)
- Wishlist-to-order conversion without cart
- Notifications when a discontinued product becomes available again
- Analytics or reporting on wishlist usage

---

## Notes

- The ticket explicitly states this is "several stories" — expect a breakdown into: (1) backend storage and API, (2) storefront wishlist page, (3) add-to-cart integration
- Guest vs signed-in distinction is a hard boundary: no partial saves, no anonymous wishlists
- "No longer for sale" is a product state the product domain knows; wishlist queries must join or fetch product availability
