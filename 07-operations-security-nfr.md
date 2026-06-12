# Operations, Security & Non-Functional Requirements

## 1. Environments

Recommended environments:

```text
local
staging
production
```

Recommended domains:

```text
local:
  API: http://localhost:5222
  Landing/Admin: http://localhost:3000
  Tenant FE: http://localhost:3001 or localhost with NEXT_PUBLIC_DEV_TENANT_SLUG

staging:
  API: https://staging-api.scannow.site
  Landing/Admin: https://staging.scannow.site
  Tenant FE: https://*.staging.scannow.site

production:
  API: https://api.scannow.site
  Landing/Admin: https://scannow.site
  Tenant FE: https://*.scannow.site
```

## 2. Backend Configuration

Required production env:

```text
ASPNETCORE_ENVIRONMENT=Production
ConnectionStrings__ScanNowDB=...
Jwt__Issuer=...
Jwt__Audience=...
Jwt__SecretKey=...
App__ClientUrl=https://scannow.site
App__FrontendBaseUrl=https://scannow.site
App__AllowedOrigins=https://scannow.site;https://www.scannow.site
App__ProductionDomain=scannow.site
App__QrTablePath=/tables
Cloudinary__CloudName=...
Cloudinary__ApiKey=...
Cloudinary__ApiSecret=...
Email__Host=...
Email__Port=...
Email__Username=...
Email__Password=...
PayOS__ClientId=...
PayOS__ApiKey=...
PayOS__ChecksumKey=...
```

Note:

- Branch-level PayOS credentials are supported and should be used for tenant/branch payments.
- Global PayOS config may still be used by infrastructure registration/defaults.
- Do not commit production secrets to git.

## 3. Frontend Configuration

Landing/Admin FE:

```text
NEXT_PUBLIC_API_URL=https://api.scannow.site
RESEND_API_KEY=...
TO_EMAIL_ADDRESS=...
SENDER_EMAIL_ADDRESS=...
```

Tenant FE:

```text
NEXT_PUBLIC_API_URL=https://api.scannow.site
NEXT_PUBLIC_DEV_TENANT_SLUG=
```

Source config changes:

- `scan-now-nextjs/src/constants/site.ts`: `baseUrl` -> `https://scannow.site`.
- `scan-now-customer/src/constants/site.ts`: currently `https://scannow.site`; use tenant-aware/generic metadata later if needed.

## 4. CORS Requirements

Backend CORS policy should allow:

- `https://scannow.site`
- `https://www.scannow.site`
- any `https://*.scannow.site`

Current backend supports wildcard subdomain with:

```text
App:ProductionDomain=scannow.site
```

Acceptance tests:

- Browser request from `https://pho24.scannow.site` succeeds.
- Browser request from unknown external domain fails.
- Credentials/cookies work from tenant FE to API.
- Preflight for `Authorization` and `X-Tenant-Slug` succeeds.

## 5. Authentication and Session Security

Current:

- Access token stored in localStorage.
- Refresh token flow uses cookie and `withCredentials`.
- JWT role claim used for authorization.

Risks:

- localStorage token is exposed to XSS.
- Admin FE route guard is client-side.
- Refresh cookie same-site/domain settings must be validated for sibling domains.

Requirements:

- All production traffic over HTTPS.
- Refresh cookie should be `HttpOnly`, `Secure`.
- SameSite policy must support `tenant.scannow.site` -> `api.scannow.site`.
- Add Content Security Policy before production if possible.
- Add explicit admin role guard in FE and rely on backend authorization as source of truth.
- Rotate refresh tokens on refresh.
- Logout should clear refresh cookie and local access token.

## 6. Tenant Security

Threats:

- Tenant A attempts to read Tenant B branch/order/menu/session by GUID/token.
- User changes `X-Tenant-Slug` manually.
- Public user uses QR token from another tenant.
- SignalR client joins arbitrary branch group.

Controls already present:

- Tenant context from slug.
- Branch global query filter.
- Service-level branch ownership checks in many flows.
- Role-based controller authorization.

Required hardening:

- Integration tests for cross-tenant data access.
- Hub authorization for `JoinBranch`.
- Public QR/session lookups should fail when tenant header does not match table/session restaurant.
- Consider adding tenant filter to all branch-scoped entities or repository helper enforcing tenant branch access.

## 7. Secrets Management

Current repo state shows sensitive-looking values in config files. They must be treated as compromised.

Requirements:

- Remove production secrets from tracked files.
- Rotate database password and any exposed API keys.
- Use environment variables or secret manager.
- Keep `.env` files out of git.
- Add `.env.example` with placeholders only.
- Review deployment logs for accidental secret printing.

Do not document actual secret values in product docs.

## 8. Payment Security

PayOS:

- Branch-level credentials can be configured.
- Response masks credential presence and client ID preview.
- Payment links expire after 10 minutes in service code.

Requirements:

- Store PayOS credentials encrypted at rest if possible.
- Never return full credentials from API.
- Validate gateway status server-side before marking payment success.
- Add webhook signature verification if PayOS webhook is added.
- Avoid trusting only frontend return URL.
- Payment amount should be immutable once payment is pending, or pending payment must be failed before order changes. Current order append flow fails pending payments.

Known issue:

- Public `CASH` checkout completes order but leaves payment pending. For production, public cash should probably mean "pay at cashier", not completed/paid.

## 9. Data Integrity

Requirements:

- Restaurant slug unique globally.
- Branch slug unique per restaurant.
- Table number unique per branch.
- QR code token unique globally.
- Session token unique among active sessions.
- Managed user exactly one branch.
- Voucher code unique per branch.
- Voucher quantity cannot be lower than used count.
- Menu item used in order must belong to same branch as session/table.

Potential improvements:

- Use database transactions around order/payment/voucher usage.
- Use optimistic concurrency for voucher `UsedCount`.
- Consider decimal rounding policy and currency precision.
- Apply `ServiceChargeFixed` consistently or remove field.

## 10. Availability and Scalability

Current limitations:

- Cart hub uses memory cache.
- SignalR without backplane will not work across multiple backend instances for groups/cache.
- Auto migration runs on startup; risky in multi-instance production.

Requirements:

- For single-instance MVP, memory cache is acceptable.
- For multi-instance production:
  - Use Redis/distributed cache for cart.
  - Use SignalR backplane/Azure SignalR.
  - Move migrations to CI/CD step.
- Health checks should be added for API/db.
- Background cleanup should remove expired sessions/pending payments if needed.

## 11. Performance Requirements

Targets:

- API p95 for list/read operations: < 500ms under normal load.
- Public menu load p95: < 800ms.
- Place order p95: < 1000ms excluding db contention.
- SignalR order update delivery: < 2s.
- Landing LCP: < 2.5s on typical mobile connection.

Data performance considerations:

- Current services often load lists into memory and filter/sort/paginate in application code.
- This is acceptable for MVP/small branch data but should move filtering/paging to database for scale.
- Reports may need optimized queries/materialization as data grows.

## 12. Observability

Minimum requirements:

- Structured request logs.
- Error logs with correlation ID.
- Payment operation logs without secrets.
- Tenant slug/restaurantId in logs where safe.
- Order lifecycle event logs.
- Admin critical actions logs.

Entities exist for audit logs but full audit workflow is not clearly exposed.

Recommended:

- Add request correlation middleware.
- Add audit logging for admin owner/restaurant ban/unban, user creation, payment processing.
- Monitor SignalR connection counts and failures.

## 13. Backup and Recovery

Database:

- Daily automated backup.
- Point-in-time recovery if provider supports.
- Restore drill before production launch.

Assets:

- Cloudinary assets should be managed by provider.
- Menu image deletion policy must be defined.

Config:

- Secret backup in secret manager.
- Do not rely on repo files for production secrets.

## 14. Compliance and Privacy

Data collected:

- User email/phone/name.
- Customer name/phone/order notes.
- Payment metadata.
- Restaurant/branch operational data.

Requirements:

- Minimize customer PII in order flow.
- Do not store card/bank credentials.
- Mask PayOS credentials.
- Define retention for customer phone/order data.
- Add privacy policy to landing before public launch.

## 15. Test Strategy

### 15.1 Backend tests

**QUY TAC BAT BUOC**:
- Voi moi doan code hoac tinh nang moi (API, Service, Handler, etc.) them vao backend, **luon luon phai tao testcase** (Unit Test / Integration Test) tuong ung.
- Code chi duoc tich hop hoac coi la hoan thanh khi toan bo cac testcase deu chay thanh cong (pass het) va bao phu day du cac truong hop, dam bao logic dung nhu thiet ke.

P0 integration tests:

- Auth login/refresh/logout.
- Admin create owner/restaurant.
- Tenant resolution by header.
- Cross-tenant branch access denied.
- Cross-tenant QR token/session/order access denied.
- Owner cannot access another restaurant branch.
- Managed user exactly one branch.
- Place order validates menu branch.
- Pending payment fails when order gets more items.
- Cashier cash checkout computes change.
- Voucher quantity/date/min/max constraints.
- PayOS not configured returns business error.

### 15.2 Frontend tests

P0:

- Landing route and admin route.
- Admin login and owner/restaurant list.
- Tenant slug extraction for `pho24.scannow.site`.
- Axios includes `X-Tenant-Slug`.
- Owner branch/user CRUD form validation.
- Cashier route after implemented.
- Public QR order flow after implemented.

### 15.3 End-to-end tests

Critical E2E:

1. Admin creates owner + restaurant slug.
2. Owner logs in via tenant subdomain.
3. Owner creates branch/table/menu.
4. Staff opens table.
5. Customer scans QR and places order.
6. Kitchen marks ready.
7. Waiter marks served.
8. Cashier pays cash.
9. Customer/order pages update realtime.

## 16. Production Launch Checklist

- Secrets removed and rotated.
- `scannow.site`, `*.scannow.site`, `api.scannow.site` DNS live.
- TLS valid for apex/wildcard/API.
- Backend `App:ProductionDomain=scannow.site`.
- FE API URLs point to `api.scannow.site`.
- FE base URLs use `scannow.site`.
- Public QR pages implemented.
- Payment pages implemented.
- Cashier UI implemented.
- Tenant QR URL dynamic by restaurant slug.
- Payment redirect dynamic by tenant slug.
- Cross-tenant tests pass.
- Payment test transactions pass.
- Backup/restore verified.
