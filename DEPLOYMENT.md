# Deployment Notes

## Required Environment Variables

Backend (`server/.env` or root `.env`):

- `NODE_ENV=production`
- `PORT=5000`
- `MONGO_URI=mongodb+srv://...`
- `MONGO_POOL_MAX=50`
- `MONGO_POOL_MIN=5`
- `JWT_ACCESS_SECRET=...`
- `JWT_REFRESH_SECRET=...`
- `JWT_ACCESS_TTL=15m`
- `JWT_REFRESH_TTL=7d`
- `BCRYPT_SALT_ROUNDS=12`
- `CORS_WHITELIST=https://your-frontend-domain.com`
- `DEFAULT_DELIVERY_FEE=40`
- `FREE_DELIVERY_THRESHOLD=499`
- `TAX_PERCENT=5`
- `LOW_STOCK_DEFAULT_THRESHOLD=10`
- `SMTP_HOST=...` (optional but recommended)
- `SMTP_PORT=587`
- `SMTP_USER=...`
- `SMTP_PASS=...`
- `SMTP_FROM=...`

Frontend:
- `apiBaseUrl` in `environment.prod.ts`
- feature flags as needed.

## Build and Run Commands

From repo root:

```bash
npm install
npm run db:indexes
npm run seed:marketplace
npm run start:server
```

Angular build:

```bash
cd client
npm install
npm run build -- --configuration production
```

## MongoDB Atlas Setup

1. Create Atlas cluster (M10+ recommended).
2. Add database user with least privilege.
3. Add IP access list for deployment platform.
4. Set `MONGO_URI` to Atlas connection string.
5. Run `npm run db:indexes` against production DB.
6. Run `npm run seed:marketplace` only once for initial setup (safe to re-run due upserts).

## Hosting Recommendations

Backend:
- Railway or Render
- Start command: `npm run start:server`
- Health check: `/health`

Frontend:
- Vercel or Netlify
- Build command: `npm run build -- --configuration production`
- Output dir: `client/dist/<project-name>`

## Production Seed Safety

1. Ensure `MONGO_URI` points to the target environment.
2. Take DB snapshot before first run.
3. Execute:
   - `npm run db:indexes`
   - `npm run seed:marketplace`
4. Verify seeded users/coupons/categories.
5. Re-running seed is idempotent via upserts and deterministic naming/slugs.
