# ADR-005: Single price calculator and a `promotion` module for coupons

- **Status:** Accepted
- **Date:** 2026-10-25
- **Deciders:** vsnvi-food backend team

## Context

The order stores a pricing snapshot (items, packaging charge, delivery fee, discount, tax), and
the cart page must show the same numbers before checkout. v1 needs per-restaurant fees and
coupons: platform coupons created by admins and restaurant coupons created by owners, with flat,
percent-with-cap and free-delivery discounts, validity windows, minimum subtotal, per-customer and
total limits, first-order-only coupons, and the coupon use given back when an order is cancelled.

## Decision

- **One calculator:** `CartPricing` in `cart` computes every price line, the tax and the total.
  The cart page shows its result and checkout snapshots the same result. No other code does price
  or tax math.
- **Fees:** each restaurant sets a flat `deliveryFee` and `packagingCharge`. Tax is
  `pricing.tax-percent` from config, applied to the item subtotal minus the item discount plus the
  packaging charge; the delivery fee isn't taxed in v1.
- **Coupons live in a new `promotion` module.** It owns `Coupon` and `CouponRedemption` and is the
  only place that calculates a discount amount.
  - `cart` calls `PromotionApi.evaluate` for a preview that reserves nothing.
  - At checkout `order` calls `PromotionApi.redeem`, which locks the coupon row, re-checks every
    rule, records the redemption and returns the final discount. A mismatch with the preview fails
    checkout with `PRICE_CHANGED`.
  - On rejection or cancellation `order` calls `PromotionApi.release`.
  - `order` passes `isFirstOrder`, because `promotion` can't depend on `order` (ADR-003).
- At most one coupon per order. Funding and settlement of discounts are out of scope for v1.
- Details: [ARCHITECTURE.md §8](../ARCHITECTURE.md#8-pricing-and-coupons).

## Alternatives considered

### Coupons inside `order`
- **Pros:** one module fewer.
- **Cons:** `cart` can't preview the discount, because `cart` can't depend on `order`.
- **Rejected.**

### Separate calculators for cart and order
- **Rejected:** the preview and the charged amount would drift apart.

### Platform-wide fee rules (distance-based delivery fee)
- **Rejected for v1** in favour of per-restaurant flat fees.

## Consequences

- **Positive:** the cart and the order always agree; limits can't be oversold, thanks to the row
  lock; coupon logic is isolated and testable.
- **Negative:**
  - The cart preview can't confirm first-order eligibility; checkout is authoritative.
  - The checkout transaction holds a coupon row lock, so no external calls (such as Razorpay) may
    happen inside it. Payment is opened after commit (see ARCHITECTURE.md §7).
