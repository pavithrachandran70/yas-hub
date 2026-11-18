---
key: PER-316
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Signed-in shoppers can save a wishlist and add those items to the cart
---

Description:

A signed-in shopper can save a product to a wishlist, see that list later on another visit, remove an item, and add a saved item to the cart. The list is still there after they sign out and sign back in. A guest has no wishlist. Adding an item that is already on the list does not create a second row. Adding a saved item to the cart leaves it on the wishlist. If the product is no longer for sale, the wishlist still shows it and explains that it cannot be added to the cart.

This is several stories: storing the list for the customer, the storefront wishlist page, and adding a saved item to the cart. The cart, product, and customer areas all have to agree on what a saved item is.

Acceptance:

* A signed-in shopper can add a product, see it on a later visit, and remove it.
* The same product cannot appear twice on one shopper’s list.
* Adding a saved item to the cart puts it in the cart and leaves it on the wishlist.
* A product that is no longer for sale stays on the list and cannot be added to the cart.
* A guest is asked to sign in before anything is saved.
