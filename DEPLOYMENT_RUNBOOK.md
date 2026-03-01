# Production Deployment Runbook

## 1. Topology

- Frontend: Angular app (`client`) served as static assets (Nginx/CDN/Vercel/Netlify).
- Backend: Node.js Express API (`server`) deployed as containerized service.
- Database: MongoDB Atlas replica set (required for transactions).
- Optional integrations:
  - SMTP provider for emails (auth alerts, support acknowledgements, delivery OTP email fallback).
  - SMS webhook provider for OTP delivery.
  - Payment provider webhooks routed to `/api/payments/webhook`.

## 2. Environment Matrix

Set all values per environment (`dev`, `staging`, `prod`). Never commit secrets.

| Variable | Required | Example | Notes |
|---|---|---|---|
| `NODE_ENV` | yes | `production` | Enables prod logger behavior. |
| `PORT` | yes | `5000` | Backend listen port. |
| `MONGO_URI` | yes | `mongodb+srv://...` | Atlas SRV URI to replica set cluster. |
| `MONGO_POOL_MAX` | yes | `50` | Connection pool upper bound. |
| `MONGO_POOL_MIN` | yes | `5` | Connection pool lower bound. |
| `JWT_ACCESS_SECRET` | yes | `<long-random>` | Access token signing key. |
| `JWT_REFRESH_SECRET` | yes | `<long-random>` | Refresh token signing key. |
| `JWT_ACCESS_TTL` | yes | `15m` | Access token TTL. |
| `JWT_REFRESH_TTL` | yes | `7d` | Refresh token TTL. |
| `BCRYPT_SALT_ROUNDS` | yes | `12` | Minimum 12. |
| `CORS_WHITELIST` | yes | `https://app.example.com` | Comma-separated list. |
| `DEFAULT_DELIVERY_FEE` | yes | `40` | Used server-side only. |
| `FREE_DELIVERY_THRESHOLD` | yes | `499` | Used server-side only. |
| `TAX_PERCENT` | yes | `5` | Used server-side only. |
| `LOW_STOCK_DEFAULT_THRESHOLD` | yes | `10` | Inventory alerts baseline. |
| `APP_NAME` | yes | `grocery-marketplace` | Logger metadata. |
| `LOG_LEVEL` | yes | `info` | Logger level. |
| `FRONTEND_BASE_URL` | yes | `https://app.example.com` | Used in links/callbacks. |
| `PAYMENT_PROVIDER` | yes | `MOCK` or provider name | Payment abstraction provider. |
| `SMTP_HOST` | optional | `smtp.example.com` | Required for live email delivery. |
| `SMTP_PORT` | optional | `587` | SMTP port. |
| `SMTP_USER` | optional | `apikey` | SMTP username. |
| `SMTP_PASS` | optional | `<secret>` | SMTP password. |
| `SMTP_FROM` | optional | `no-reply@example.com` | Sender address. |
| `SMS_WEBHOOK_URL` | optional | `https://sms-gw/...` | For delivery OTP SMS hook. |
| `SERVICEABLE_PINCODES` | optional | `560001,560002` | Overrides default pincode list. |

Frontend (`client/src/environments/*`):

| Variable | Required | Example |
|---|---|---|
| `apiBaseUrl` | yes | `https://api.example.com/api` |

## 3. MongoDB Atlas Setup

1. Create an Atlas project and cluster (replica set / M10+ recommended).
2. Create database user with least privilege:
   - App user: readWrite on application database.
3. Network access:
   - Restrict by deployment egress CIDRs.
4. Capture `MONGO_URI`.
5. Disable `autoIndex` in production (already set in app config).
6. Run index sync job after deploy:
   - `npm run db:indexes`

## 4. Docker Build and Runtime

### Backend Dockerfile (recommended)

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 5000
CMD ["npm", "run", "start:server"]
```

### Build and run

```bash
docker build -t grocery-api:latest .
docker run -d --name grocery-api \
  --env-file .env.production \
  -p 5000:5000 \
  grocery-api:latest
```

### Health check

- Endpoint: `GET /health`
- Expect: `200 { success: true, status: "ok" }`

## 5. Deployment Sequence (Production)

1. Deploy backend container with production env vars.
2. Verify `/health`.
3. Run DB index sync:
   - `npm run db:indexes`
4. Run deterministic seed only if initial bootstrap or controlled refresh:
   - `npm run seed:marketplace`
5. Deploy frontend static bundle pointed to production `apiBaseUrl`.
6. Configure payment provider webhook target:
   - `POST https://api.example.com/api/payments/webhook`
7. Configure SMTP and SMS provider credentials.
8. Smoke-test critical flows:
   - Auth/login/refresh/logout
   - Cart -> order -> payment confirm
   - Order status transitions
   - Group cancel/refund lifecycle
   - Support ticket + acknowledgement email

## 6. Seed and Index Flow Safety

- Always run `db:indexes` before heavy production traffic.
- Seed script uses upserts and deterministic generation, but still treat as controlled operation.
- Recommended seed order:
  1. `npm run db:indexes`
  2. `npm run seed:marketplace`
  3. `npm run db:indexes` (optional second pass to ensure any new collections are indexed)

## 7. Rollback Plan

1. Roll back application image to previous known-good tag.
2. Keep DB intact (no destructive DB rollback unless explicit migration required).
3. Re-run `npm run db:indexes` for compatibility.
4. Disable problematic webhooks temporarily if needed.

## 8. Operational Checks

- Logs: verify no repeating `AUTH_REQUIRED`, `INVALID_WEBHOOK_SIGNATURE`, `OTP_DELIVERY_FAILED`.
- Scheduler jobs:
  - Pending payment cancellation
  - Low stock cache refresh
  - Account anonymization
- Alerts to configure:
  - 5xx rate
  - DB connection errors
  - webhook failure rate
  - zero-stock variant spikes

## 9. Minimum Go-Live Checklist

- [ ] All required env vars present and validated.
- [ ] Atlas replica set reachable from runtime.
- [ ] CORS whitelist set to production frontend domains.
- [ ] JWT secrets rotated and unique per environment.
- [ ] SMTP tested.
- [ ] SMS webhook tested.
- [ ] Payment webhook verified end-to-end.
- [ ] `db:indexes` executed successfully.
- [ ] Seed executed (if bootstrap) and verified.
- [ ] Critical user/admin/seller smoke tests passed.

## 10. Admin/Seller Portal Isolation Strategy

Angular lazy loading and `canMatch` guards prevent unauthorized route activation, but static JS chunks can still be fetched if publicly hosted. For enterprise isolation, use one of these deployment patterns:

### Option A: Separate Admin Subdomain Build
- Deploy storefront at `app.example.com`.
- Deploy admin/seller portals on separate origin(s), for example:
  - `admin.example.com`
  - `seller.example.com`
- Configure backend CORS and auth cookie/token policies per origin.
- Benefit: strongest separation of static assets and access perimeter.

### Option B: Reverse Proxy Path-Level Access Control
- Keep single frontend deployment but enforce gateway/proxy ACLs for `/admin/*` and `/seller/*` assets/routes.
- Apply identity-aware access controls (SSO group/role checks) before serving protected portal paths/chunks.
- Benefit: centralized enforcement at edge without application refactor.

### Option C: Separate CI/CD Builds Per Portal
- Produce independent artifact pipelines:
  - user-only bundle
  - admin-only bundle
  - seller-only bundle
- Publish each artifact to isolated buckets/CDN paths with distinct access policies.
- Benefit: minimal runtime overlap and explicit least-privilege distribution.
