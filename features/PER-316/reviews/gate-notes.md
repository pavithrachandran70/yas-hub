# Critic notes for PER-316

Not blocking. For whoever reviews the specs at the gate.

## Notes (3)

- **C4** · `specs/3.md` › ## Database Schema › ### Cart Service — Verification and Migration: The spec asks Dev to verify and add a unique constraint on `cart_item`, but the entity already uses a composite primary key `@IdClass(CartItemId.class)` on `(customerId, productId)`, while the schema has single PK `id`. _(new in a re-check, in text no fix touched)_
- **C5** · `specs/3.md` › ## Database Schema › ### Cart Service — Verification and Migration › **Decision:** Add migration `changelog-0005.sql` to enforce uniqueness.: The story says to add a UNIQUE constraint on `(cart_id, product_id)` in a new migration `changelog-0005.sql`. _(the critic marked it a note)_
- **C6** · `stories/01-app.md` › ## Open Questions › 1. Product Service Batch Endpoint: The open question asks whether a batch endpoint exists and suggests parallel calls as a fallback, but the cart service's `ProductService.getProducts()` shows the product service already accepts multiple product IDs in … _(new in a re-check, in text no fix touched)_

## Outdated: about text that has changed since (1)

- **C1** · `specs/2.md` › (entire file): The file `specs/2.md` contains the backend API spec (database schema, Java entities, REST endpoints) but the breakdown states Story 2 is "Storefront wishlist page" (UI components, Next.js pages, TypeScript API client). _(its quote is in no spec file)_
