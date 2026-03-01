# API Reference (Synced to `server/src/routes`)

Base URL: `/api`

## Conventions

### Auth
- `None`: public endpoint.
- `Bearer`: requires `Authorization: Bearer <accessToken>`.

### Roles
- `USER`, `ADMIN`, `SELLER`.
- If route-level role middleware is absent, controller-level checks may apply.

### Pagination Contract
Paginated list endpoints return:
```json
{
  "success": true,
  "items": [],
  "total": 0,
  "page": 1,
  "pageSize": 20,
  "totalPages": 1
}
```
Query defaults:
- `page` default `1`
- `pageSize` default `20`
- `pageSize` max `100`

### Error Contract
```json
{
  "success": false,
  "code": "ERROR_CODE",
  "message": "Human readable message",
  "details": []
}
```

## Auth Routes (`/auth`)

### `POST /api/auth/register`
- Auth: None
- Role: Public
- Body:
```json
{
  "name": "string (2-120)",
  "email": "email",
  "phone": "string (10-20)",
  "password": "8-64 with upper/lower/number/special"
}
```
- Response:
```json
{
  "success": true,
  "data": {
    "id": "string",
    "name": "string",
    "email": "email",
    "phone": "string",
    "role": "USER",
    "accountStatus": "ACTIVE|BLOCKED|PENDING_DELETION|DELETED"
  }
}
```

### `POST /api/auth/login`
- Auth: None
- Role: Public
- Body:
```json
{ "email": "email", "password": "string" }
```
- Response:
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "string",
      "name": "string",
      "email": "email",
      "phone": "string",
      "role": "USER|ADMIN|SELLER",
      "accountStatus": "ACTIVE|BLOCKED|PENDING_DELETION|DELETED"
    },
    "accessToken": "jwt",
    "refreshToken": "jwt"
  }
}
```

### `POST /api/auth/refresh`
- Auth: None
- Role: Public
- Body:
```json
{ "refreshToken": "jwt" }
```
- Response:
```json
{
  "success": true,
  "data": {
    "accessToken": "jwt",
    "refreshToken": "jwt"
  }
}
```

### `POST /api/auth/logout`
- Auth: Bearer
- Role: Any authenticated role
- Body:
```json
{ "refreshToken": "jwt" }
```
- Response:
```json
{ "success": true, "message": "Logged out" }
```

### `POST /api/auth/logout-all`
- Auth: Bearer
- Role: Any authenticated role
- Body: none
- Response:
```json
{ "success": true, "message": "Logged out from all devices" }
```

## Product Routes (`/products`)

### `GET /api/products/banners/home`
- Auth: None
- Role: Public
- Body: none
- Response:
```json
{
  "success": true,
  "data": [
    {
      "_id": "ObjectId",
      "title": "string",
      "subtitle": "string",
      "imageUrl": "string",
      "ctaText": "string",
      "ctaLink": "string",
      "displayOrder": 0
    }
  ]
}
```

### `GET /api/products`
- Auth: None
- Role: Public
- Query:
  - `page`, `pageSize`
  - `categoryId`, `subCategoryId`, `sellerId` (ObjectId)
  - `brand`, `search`
  - `sortBy`: `relevance|price_asc|price_desc|popularity|newest|discount_desc`
  - `isFeatured`: `true|false`
  - `discountMin`, `minRating`, `minPrice`, `maxPrice`
  - `inStock`: `true|false`
- Response: pagination contract (`items` are product cards).

### `GET /api/products/:slug`
- Auth: None
- Role: Public
- Path params: `slug`
- Response:
```json
{
  "success": true,
  "data": {
    "_id": "ObjectId",
    "name": "string",
    "slug": "string",
    "SKU": "string",
    "brand": "string",
    "variants": [],
    "rating": 0,
    "totalReviews": 0,
    "images": [],
    "description": "string",
    "tags": []
  }
}
```

## Public Routes (`/public`)

### `GET /api/public/delivery-slots`
- Auth: None
- Role: Public
- Body: none
- Response: pagination-shaped fixed list (next 7-day available slots).

### `GET /api/public/delivery-availability/:pincode`
- Auth: None
- Role: Public
- Path params: `pincode` (6 digits)
- Response:
```json
{
  "success": true,
  "data": {
    "pincode": "string",
    "serviceable": true,
    "expectedDeliveryDate": "ISO_DATE"
  }
}
```
or
```json
{
  "success": true,
  "data": {
    "pincode": "string",
    "serviceable": false,
    "message": "Not Deliverable to this pincode"
  }
}
```

### `POST /api/public/newsletter`
- Auth: None
- Role: Public
- Body:
```json
{ "email": "email" }
```
- Response:
```json
{
  "success": true,
  "data": {
    "email": "email",
    "message": "Subscription successful | Email already subscribed"
  }
}
```

### `POST /api/public/support`
- Auth: None
- Role: Public
- Body:
```json
{
  "name": "string",
  "email": "email",
  "subject": "string",
  "category": "string",
  "message": "string"
}
```
- Response: created ticket document.

### `POST /api/public/support/authenticated`
- Auth: Bearer
- Role: Any authenticated role
- Body: same as `/support`
- Response: created ticket document.

## Catalog Meta Routes (`/meta`)

### `GET /api/meta/categories/root`
- Auth: None
- Role: Public
- Response: pagination-shaped fixed list (`items` root categories).

### `GET /api/meta/categories/by-slug/:slug`
- Auth: None
- Role: Public
- Response: single category by slug.

### `GET /api/meta/categories/:parentId/subcategories`
- Auth: None
- Role: Public
- Response: pagination-shaped fixed list (`items` subcategories).

### `GET /api/meta/brands/featured`
- Auth: None
- Role: Public
- Response: pagination-shaped fixed list (`items` featured active brands).

## Offers Routes (`/offers`)

### `GET /api/offers`
- Auth: None
- Role: Public
- Response:
```json
{
  "success": true,
  "data": {
    "coupons": [],
    "discountedProducts": [],
    "bundleOffers": [],
    "seasonalSales": []
  }
}
```

### `GET /api/offers/help-content`
- Auth: None
- Role: Public
- Response:
```json
{
  "success": true,
  "data": {
    "returnRefundPolicy": {},
    "deliveryInformation": {},
    "cancellationPolicy": {}
  }
}
```

## Cart Routes (`/cart`)

All cart endpoints:
- Auth: Bearer
- Role: `USER`

### `GET /api/cart`
- Body: none
- Response: normalized cart snapshot:
```json
{
  "success": true,
  "data": {
    "items": [],
    "summary": {
      "totalMRP": 0,
      "totalDiscount": 0,
      "couponDiscount": 0,
      "deliveryFee": 0,
      "tax": 0,
      "grandTotal": 0
    },
    "couponCode": "string"
  }
}
```

### `POST /api/cart/items`
- Body:
```json
{ "productId": "ObjectId", "variantId": "ObjectId", "quantity": 1 }
```
- Response: updated cart snapshot (`data`).

### `PATCH /api/cart/items`
- Body: same as add item
- Response: updated cart snapshot (`data`).

### `DELETE /api/cart/items`
- Body:
```json
{ "productId": "ObjectId", "variantId": "ObjectId" }
```
- Response: updated cart snapshot (`data`).

### `POST /api/cart/coupon`
- Body:
```json
{ "code": "SAVE10" }
```
- Response: updated cart snapshot (`data`).

### `DELETE /api/cart/coupon`
- Body: none
- Response: updated cart snapshot (`data`).

## Wishlist Routes (`/wishlist`)

All wishlist endpoints:
- Auth: Bearer
- Role: `USER`

### `GET /api/wishlist`
- Body: none
- Response: pagination-shaped fixed list (`items` wishlisted products).

### `POST /api/wishlist/items`
- Body:
```json
{ "productId": "ObjectId" }
```
- Response: updated wishlist payload (`data`).

### `DELETE /api/wishlist/items`
- Body:
```json
{ "productId": "ObjectId" }
```
- Response: updated wishlist payload (`data`).

### `POST /api/wishlist/notify-stock`
- Body:
```json
{ "productId": "ObjectId" }
```
- Response: subscription result (`data`).

## Order Routes (`/orders`)

All order endpoints require Bearer auth.

### `GET /api/orders/my`
- Auth: Bearer
- Role: `USER`
- Query: `type` (`active|history`), `page`, `pageSize`
- Response: pagination contract (`items` user child orders).

### `GET /api/orders/my-groups`
- Auth: Bearer
- Role: `USER`
- Query: `type` (`active|history`), `page`, `pageSize`
- Response: pagination contract (`items` parent/group orders).

### `GET /api/orders/groups/:orderGroupId`
- Auth: Bearer
- Role: owner user, group seller participant, or admin (service-level enforcement)
- Response:
```json
{
  "success": true,
  "data": {
    "parent": {},
    "children": []
  }
}
```

### `POST /api/orders/groups/:orderGroupId/cancel`
- Auth: Bearer
- Role: `USER|ADMIN`
- Body: none
- Response:
```json
{
  "success": true,
  "data": {
    "orderGroupId": "string",
    "aggregateOrderStatus": "string",
    "aggregatePaymentStatus": "string",
    "cancelledCount": 0,
    "children": []
  }
}
```

### `POST /api/orders/groups/:orderGroupId/refund/initiate`
- Auth: Bearer
- Role: `ADMIN`
- Body: none
- Response:
```json
{
  "success": true,
  "data": {
    "orderGroupId": "string",
    "aggregateOrderStatus": "string",
    "aggregatePaymentStatus": "string",
    "refunds": [],
    "children": []
  }
}
```

### `POST /api/orders/groups/:orderGroupId/refund/settle`
- Auth: Bearer
- Role: `ADMIN`
- Body:
```json
{ "refundReferenceId": "optional string" }
```
- Response:
```json
{
  "success": true,
  "data": {
    "orderGroupId": "string",
    "aggregateOrderStatus": "string",
    "aggregatePaymentStatus": "string",
    "refundedCount": 0,
    "children": []
  }
}
```

### `GET /api/orders/:orderId`
- Auth: Bearer
- Role: owner user, seller of that order, or admin
- Response: single order (`data`).

### `POST /api/orders`
- Auth: Bearer
- Role: `USER`
- Body:
```json
{
  "shippingAddress": {
    "fullName": "string",
    "phone": "string",
    "line1": "string",
    "line2": "string",
    "city": "string",
    "state": "string",
    "pincode": "string",
    "label": "string"
  },
  "deliverySlotId": "ObjectId",
  "paymentMethod": "COD|ONLINE",
  "couponCode": "optional string"
}
```
- Response:
```json
{
  "success": true,
  "data": {
    "orderGroupId": "GRP...",
    "orderId": "ORD...",
    "orderIds": ["ORD..."],
    "totalOrders": 1
  }
}
```

### `POST /api/orders/:orderId/confirm-payment`
- Auth: Bearer
- Role: `USER|ADMIN`
- Body:
```json
{
  "paymentGatewayOrderId": "optional string",
  "paymentGatewayPaymentId": "optional string"
}
```
- Response: updated child order (`data`).

### `PATCH /api/orders/:orderId/status`
- Auth: Bearer
- Role: `USER|ADMIN|SELLER` (state-machine + actor checks in service)
- Body:
```json
{
  "nextStatus": "PENDING_PAYMENT|CONFIRMED|PROCESSING|PACKED|SHIPPED|OUT_FOR_DELIVERY|DELIVERED|CANCELLED|REFUND_INITIATED|REFUNDED",
  "deliveryOtp": "optional string (required for DELIVERED)",
  "trackingId": "optional string (required in seller UI for SHIPPED)"
}
```
- Response: updated child order (`data`).

## Payment Routes (`/payments`)

### `POST /api/payments/webhook`
- Auth: None
- Role: Gateway callback
- Headers:
  - `x-payment-signature` (optional for mock provider, required by real providers)
- Body:
```json
{
  "externalOrderId": "optional string",
  "externalPaymentId": "optional string",
  "event": "optional string",
  "status": "optional string"
}
```
- Response:
```json
{ "success": true, "data": { "ok": true, "duplicate": false } }
```

### `POST /api/payments/create-order`
- Auth: Bearer
- Role: `USER`
- Body:
```json
{ "orderId": "string" }
```
- Response:
```json
{
  "success": true,
  "data": {
    "paymentTransactionId": "ObjectId",
    "orderId": "ORD...",
    "externalOrderId": "string",
    "paymentUrl": "string",
    "amount": 0,
    "currency": "INR"
  }
}
```

### `POST /api/payments/refund`
- Auth: Bearer
- Role: `ADMIN`
- Body:
```json
{ "orderId": "string" }
```
- Response:
```json
{
  "success": true,
  "data": {
    "orderId": "ORD...",
    "refundReferenceId": "string"
  }
}
```

## Account Routes (`/account`)

All account endpoints:
- Auth: Bearer
- Role: `USER|ADMIN|SELLER`

### `GET /api/account/me`
- Response: user profile (`data`).

### `PATCH /api/account/me`
- Body:
```json
{
  "name": "optional string",
  "email": "optional email",
  "phone": "optional string",
  "profileImage": "optional string"
}
```
- Response: updated profile (`data`).

### `PATCH /api/account/me/notifications`
- Body:
```json
{
  "emailOrders": "optional boolean",
  "emailPromotions": "optional boolean",
  "smsOrders": "optional boolean",
  "pushNotifications": "optional boolean"
}
```
- Response: updated notification preferences (`data`).

### `PUT /api/account/me/addresses`
- Body:
```json
{
  "addresses": [
    {
      "label": "string",
      "fullName": "string",
      "phone": "string",
      "line1": "string",
      "line2": "optional string",
      "city": "string",
      "state": "string",
      "pincode": "string",
      "isDefault": "optional boolean"
    }
  ]
}
```
- Response: updated addresses (`data`).

### `GET /api/account/me/sessions`
- Response: session list (`data`).

### `PATCH /api/account/me/two-factor`
- Body:
```json
{ "enabled": true }
```
- Response: updated 2FA state (`data`).

### `GET /api/account/me/cards`
- Response: masked saved cards (`data`).

### `POST /api/account/me/cards`
- Body:
```json
{
  "last4": "string(4)",
  "brand": "string",
  "expiryMonth": 1,
  "expiryYear": 2028,
  "gatewayToken": "string",
  "isDefault": "optional boolean"
}
```
- Response: updated card list/state (`data`).

### `DELETE /api/account/me/cards/:cardId`
- Response: updated card list/state (`data`).

### `PATCH /api/account/me/cards/:cardId/default`
- Response: updated card list/state (`data`).

### `POST /api/account/me/change-password`
- Body:
```json
{
  "currentPassword": "string",
  "newPassword": "string"
}
```
- Response:
```json
{ "success": true, "message": "Password changed successfully" }
```

### `POST /api/account/me/logout-all`
- Body: none
- Response:
```json
{ "success": true, "message": "Logged out from all devices" }
```

### `POST /api/account/me/data-download`
- Body: none
- Response:
```json
{ "success": true, "message": "Data download request submitted" }
```

### `POST /api/account/me/profile-otp/request`
- Body:
```json
{ "type": "email|phone", "value": "string" }
```
- Response:
```json
{ "success": true, "message": "OTP sent successfully" }
```

### `POST /api/account/me/profile-otp/verify`
- Body:
```json
{ "type": "email|phone", "value": "string", "otp": "string" }
```
- Response:
```json
{ "success": true, "message": "OTP verified successfully" }
```

### `POST /api/account/me/delete-request`
- Body: none
- Response:
```json
{ "success": true, "message": "Account deletion requested" }
```

### `POST /api/account/me/recently-viewed`
- Body:
```json
{ "productId": "ObjectId" }
```
- Response:
```json
{ "success": true, "message": "Recently viewed updated" }
```

### `GET /api/account/me/recently-viewed`
- Response: recently viewed products (`data`).

## Admin Routes (`/admin`)

All admin endpoints:
- Auth: Bearer
- Role: `ADMIN`

### Dashboard / Analytics
- `GET /api/admin/dashboard/metrics`
  - Response: `{ todayRevenue, todayOrders, activeUsers, lowStockCount, revenueDeltaPct }`
- `GET /api/admin/dashboard/analytics`
  - Query: `rangeDays (7|30|90)`, `fromDate`, `toDate`
  - Response: `{ metrics, revenueSeries, ordersByStatus, topSellingProducts, lowStock, recentOrders }`

### Orders / Inventory / Users
- `GET /api/admin/orders`
  - Query: `status`, `paymentStatus`, `sellerId`, `orderId`, `fromDate`, `toDate`, `page`, `pageSize`
  - Response: pagination contract.
- `GET /api/admin/inventory/low-stock`
  - Query: `force=true|false`
  - Response: `{ data: [], items: [], total, page, pageSize, totalPages, updatedAt, source }`
- `GET /api/admin/inventory/movements`
  - Query: `productId`, `variantId`, `reason`, `page`, `pageSize`
  - Response: pagination contract.
- `GET /api/admin/users`
  - Query: `search`, `role`, `status`, `page`, `pageSize`
  - Response: pagination contract.
- `PATCH /api/admin/users/:id/block`
  - Body: `{ "block": true|false, "reason": "optional string" }`
  - Response: success message.

### Bulk CSV / Import Jobs
- `GET /api/admin/products/csv-template`
  - Response: CSV download.
- `POST /api/admin/products/bulk-upload`
  - Body: `{ "csvContent": "string" }`
  - Response: `{ jobId, status }` (HTTP `202`).
- `GET /api/admin/products/bulk-upload/:jobId`
  - Response: import job status document.
- `GET /api/admin/products/bulk-upload/:jobId/failures`
  - Response: CSV failure report download.
- `GET /api/admin/inventory/stock-csv-template`
  - Response: CSV download.
- `POST /api/admin/inventory/bulk-stock-upload`
  - Body: `{ "csvContent": "string" }`
  - Response: `{ jobId, status }` (HTTP `202`).

### Seller Product Moderation
- `GET /api/admin/products/pending-approval`
  - Query: `page`, `pageSize` (pagination contract).
- `PATCH /api/admin/products/:id/review`
  - Body: `{ "action": "APPROVE|REJECT", "note": "optional string" }`
  - Response: updated product.

### Inventory Threshold
- `PATCH /api/admin/inventory/threshold`
  - Body: `{ "productId": "ObjectId", "threshold": 10 }`
  - Response: updated product threshold data.

### Catalog Upserts
- `POST /api/admin/products/upsert` (product payload)
- `DELETE /api/admin/products/:id`
- `POST /api/admin/categories/upsert` (category payload)
- `DELETE /api/admin/categories/:id`
- `POST /api/admin/coupons/upsert` (coupon payload)
- `POST /api/admin/banners/upsert` (banner payload)
- `GET /api/admin/banners` (all banners)
- `PATCH /api/admin/banners/reorder`
  - Body: `{ "bannerIds": ["ObjectId", "..."] }`
- `POST /api/admin/brands/upsert` (brand payload)
- `POST /api/admin/bundle-offers/upsert` (bundle offer payload)
- `POST /api/admin/seasonal-sales/upsert` (seasonal sale payload)
- `POST /api/admin/policies/upsert`
  - Body:
```json
{
  "key": "RETURN_REFUND_POLICY|DELIVERY_INFORMATION|CANCELLATION_POLICY|COOKIE_POLICY",
  "title": "string",
  "contentHtml": "string"
}
```
- Common response: `{ success: true, data }` for upserts and `{ success: true, message }` for deletes.

## Seller Routes (`/seller`)

All seller endpoints:
- Auth: Bearer
- Role: `SELLER`

### `GET /api/seller/dashboard`
- Response:
```json
{
  "success": true,
  "data": {
    "metrics": {
      "todayRevenue": 0,
      "todayOrders": 0,
      "lowStockCount": 0,
      "pendingOrders": 0,
      "revenueDeltaPct": 0
    },
    "revenueSeries": [],
    "topProducts": [],
    "recentOrders": []
  }
}
```

### `GET /api/seller/analytics`
- Query: `fromDate`, `toDate`
- Response:
```json
{
  "success": true,
  "data": {
    "revenueSeries": [],
    "topProducts": [],
    "fulfillmentRate": 0,
    "totalOrders": 0,
    "totalRevenue": 0
  }
}
```

### `GET /api/seller/analytics/export`
- Query: `fromDate`, `toDate`
- Response: CSV download (`seller_sales_report.csv`).

### `GET /api/seller/orders`
- Query: `status`, `page`, `pageSize`
- Response: pagination contract.

### `GET /api/seller/orders/:orderId`
- Response: seller-scoped order detail (`404` if not seller-owned).

### `GET /api/seller/products`
- Query: `page`, `pageSize`
- Response: pagination contract.

### `POST /api/seller/products/upsert`
- Body: seller product payload (same core shape as admin product upsert without `sellerId` field; seller inferred from token).
- Response: `{ success: true, data }`.

### `PATCH /api/seller/inventory/stock`
- Body:
```json
{ "productId": "ObjectId", "variantId": "ObjectId", "stock": 0 }
```
- Response: updated product snapshot (`data`).

### `GET /api/seller/inventory/movements`
- Query: `page`, `pageSize`
- Response: pagination contract.

## Reports Routes (`/reports`)

All report endpoints:
- Auth: Bearer
- Role: `ADMIN`

### Data endpoints
- `GET /api/reports/sales`
  - Query: `fromDate`, `toDate`
  - Response: sales aggregate object.
- `GET /api/reports/revenue`
  - Query: `fromDate`, `toDate`
  - Response: revenue aggregate + series.
- `GET /api/reports/product-performance`
  - Response: product performance array.
- `GET /api/reports/customer-growth`
  - Query: `fromDate`, `toDate`
  - Response: customer growth aggregate + registrations series.

### Export endpoints
- `GET /api/reports/sales/export` -> CSV
- `GET /api/reports/sales/export-pdf` -> PDF
- `GET /api/reports/revenue/export` -> CSV
- `GET /api/reports/customer-growth/export` -> CSV
- `GET /api/reports/product-performance/export` -> CSV
