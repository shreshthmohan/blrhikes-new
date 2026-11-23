# BLR Hikes — Project Plan

> **Stack:** React Router v7 (SSR) · PayloadCMS v3 · Cloudflare Workers (Runtime) · D1 (SQL) · R2 (Blob) · Razorpay

---

## 1. Architectural Strategy

The goal is to move from a fragmented setup to a **Single-Process Architecture** running on Cloudflare's global network.

- **Compute:** React Router v7 handling both the Admin UI (Payload) and the Public Storefront.
- **Data:** D1 for relational integrity (transactions for bookings); R2 for GPX and media.
- **Security:** GPX files are stored in a private R2 bucket. Access is granted via pre-signed URLs generated on-the-fly after validating user session and purchase status.
- **Consistency:** Implementation of a `webhook_events` audit table in D1 to ensure idempotency and prevent data loss during Razorpay/Resend interactions.

---

## 2. Phase Breakdown (12-Week Timeline)

> Based on 12 hours/week velocity.

### Phase 1: Core Infra & Content (Weeks 1–3)

| Week | Focus | Deliverables |
|------|-------|-------------|
| W1 | Scaffolding | Initialize RRv7 + Payload v3 on Workers. Bind D1/R2. Setup GitHub Actions for preview deploys. |
| W2 | Schema & Migration | Define Collections (Trails, Posts, Users). Write a Node.js script to map legacy JSON/Markdown data into D1. |
| W3 | Public Views | Build SSR-optimized `/trails` and `/blog`. Implement mobile-first Tailwind UI. |

### Phase 2: Payments & Protected Assets (Weeks 4–6)

| Week | Focus | Deliverables |
|------|-------|-------------|
| W4 | Razorpay Integration | Server-side create-order flow. Frontend checkout modal. |
| W5 | Webhook Hardening | Build the `/api/webhooks/razorpay` handler. Implement signature verification and idempotency checks using the D1 audit table. |
| W6 | GPX Signing | Logic to fetch R2 objects via signed GET requests (5-min TTL). Tie access to `user.purchaseStatus`. |

### Phase 3: Events & Messaging (Weeks 7–9)

| Week | Focus | Deliverables |
|------|-------|-------------|
| W7 | Events Engine | Schema for Events (linked to Trails). Logic for `capacity` vs `registeredCount`. |
| W8 | Registration Flow | Atomic transactions in D1 to handle event booking (preventing over-booking). |
| W9 | Notifications | Resend integration for transactional emails (purchase receipts, event tickets). |

### Phase 4: Reliability & Performance (Weeks 10–12)

| Week | Focus | Deliverables |
|------|-------|-------------|
| W10 | Observability | Sentry integration for Edge Runtime. Vitest for payment/signing logic. |
| W11 | Edge Caching | Implement `Cache-Control` headers. Cache purging via Payload hooks. |
| W12 | Cutover | 301 redirect mapping. DNS migration. Legacy site moved to `archive.blrhikes.com`. |

---

## 3. Technical Implementation Details

### Data Integrity & Transactions

D1 will handle all state. For event registrations, SQL transactions ensure `registeredCount` increments only if capacity is not exceeded:

```sql
UPDATE events SET registeredCount = registeredCount + 1
WHERE id = ? AND registeredCount < capacity;
```

### GPX Security

Assets are not public. The system will use `@aws-sdk/s3-request-presigner` (compatible with Workers) to generate time-limited URLs.

- **Storage:** `r2://blrhikes-assets/gpx/{trail-id}.gpx`
- **Delivery:** Authenticated User → Edge Worker → Signed R2 URL → Client

### CI/CD Pipeline

- **Lint/Typecheck:** Every PR.
- **Preview Deploys:** Cloudflare Pages/Workers preview for every branch.
- **Production:** Automated deploy on `main` merge after Vitest pass.

---

## 4. Key Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Cold Starts | Minimized by using Workers (not Lambda). Payload v3 is optimized for this. |
| D1 Limits | Denormalize view counts to stay within D1 write limits if traffic spikes. |
| SEO | Strict adherence to a redirect map (Worker-based) to maintain 100% of legacy link equity. |

---

## 5. Definition of Done (MVP)

- [ ] Users can browse all 31 migrated trails.
- [ ] Payment converts a "Free" user to a "Paid" user in D1.
- [ ] Signed GPX downloads function correctly for paid users.
- [ ] Event registrations trigger a confirmation email and update DB counts.
