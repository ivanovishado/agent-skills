# OpenPay Security Checklist

## Pre-Production Security Audit

Complete this checklist before launching OpenPay integration to production.

## Payment Processing Security

### 1. Card Data Handling

- [ ] **Never** store raw card numbers in database
- [ ] **Never** send raw card data to your server
- [ ] Use OpenPay.js for client-side card tokenization
- [ ] Only send tokenId to server, not card details
- [ ] Implement device session ID for fraud detection
- [ ] Use HTTPS for all payment pages

**Implementation:**
```typescript
// ❌ WRONG - Sending raw card data to server
const response = await fetch("/api/payments/create-charge", {
  body: JSON.stringify({
    cardNumber: "4111111111111111",
    cvv: "123",
    expiry: "12/25",
  }),
});

// ✅ RIGHT - Tokenize on client, send token only
const token = await OpenPay.token.create({
  card_number: cardNumber,
  holder_name: holderName,
  expiration_year: expiryYear,
  expiration_month: expiryMonth,
  cvv2: cvv,
});

const response = await fetch("/api/payments/create-charge", {
  body: JSON.stringify({
    tokenId: token.data.id,
    deviceSessionId: OpenPay.deviceData.setup(),
  }),
});
```

### 2. Webhook Security

- [ ] **Always** verify webhook signature with HMAC SHA256
- [ ] Use timing-safe comparison for signature verification
- [ ] Store webhook secret in environment variables
- [ ] Never commit webhook secret to version control
- [ ] Require HTTPS for webhook URLs
- [ ] Implement rate limiting on webhook endpoint
- [ ] Log invalid signature attempts for monitoring

**Implementation:**
```typescript
// ✅ Timing-safe comparison
return crypto.timingSafeEqual(
  Buffer.from(signature),
  Buffer.from(expectedSignature)
);

// ❌ WRONG - Vulnerable to timing attacks
return signature === expectedSignature;
```

### 3. Environment Variables

- [ ] All secrets stored in `.env.local` (never `.env`)
- [ ] `.env.local` in `.gitignore`
- [ ] Different credentials for sandbox vs production
- [ ] Webhook secret differs from API keys
- [ ] Secrets never exposed to client-side code
- [ ] Secrets never logged to console

**File structure:**
```bash
# ✅ RIGHT
.env.local          # Local secrets (gitignored)
.env.example        # Template without secrets (committed)

# ❌ WRONG
.env                # Might be committed by accident
config.ts           # Hardcoded secrets
```

### 4. Money Handling

- [ ] All amounts stored in cents (BIGINT)
- [ ] No floating-point arithmetic for money
- [ ] Currency explicitly set to "MXN"
- [ ] Math.ceil() used for fee calculations
- [ ] Conversion to pesos only for display/API calls
- [ ] Tests verify precision (no floating-point errors)

**Implementation:**
```typescript
// ✅ RIGHT - Cents precision
const ownerPriceCents = 500000; // $5,000.00
const feeCents = Math.ceil(ownerPriceCents * 0.029 + 250);
const totalCents = ownerPriceCents + feeCents;

// ❌ WRONG - Floating-point errors
const ownerPrice = 5000.00;
const fee = ownerPrice * 0.029 + 2.50; // 147.50000000000003
const total = ownerPrice + fee; // Precision loss
```

## API Route Security

### 5. Authentication & Authorization

- [ ] All payment routes require authentication
- [ ] User ownership verified before payment
- [ ] Booking status checked before payment
- [ ] Payment amount verified against booking
- [ ] No direct SQL queries (use ORM/query builder)
- [ ] Input validation on all parameters

**Implementation:**
```typescript
// ✅ Complete security checks
export async function POST(request: NextRequest) {
  // 1. Authentication
  const { userId } = await auth();
  if (!userId) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  // 2. Fetch booking
  const { data: booking } = await supabase
    .from("bookings")
    .select("guest_id, status, owner_price_cents")
    .eq("id", bookingId)
    .single();

  // 3. Verify ownership
  const { data: profile } = await supabase
    .from("profiles")
    .select("id")
    .eq("clerk_id", userId)
    .single();

  if (booking.guest_id !== profile.id) {
    return NextResponse.json({ error: "Forbidden" }, { status: 403 });
  }

  // 4. Verify status
  if (booking.status !== "approved") {
    return NextResponse.json({ error: "Invalid booking status" }, { status: 400 });
  }

  // 5. Calculate & verify amount
  const paymentOptions = calculatePaymentOptions(booking.owner_price_cents);
  const amountCents = paymentOptions[paymentMethod].priceCents;

  // 6. Create charge
  const charge = await createCardCharge(tokenId, amountCents, description, bookingId, deviceSessionId);
}
```

### 6. Error Handling

- [ ] Catch all OpenPay errors
- [ ] Never expose internal error details to client
- [ ] Log detailed errors server-side
- [ ] Return generic error messages to client
- [ ] Handle network timeouts gracefully
- [ ] Implement idempotency for charge creation

**Implementation:**
```typescript
try {
  const charge = await createCardCharge(tokenId, amountCents, description, bookingId, deviceSessionId);
} catch (openpayError: any) {
  // ✅ Log detailed error server-side
  console.error("OpenPay Error:", {
    code: openpayError.error_code,
    description: openpayError.description,
    category: openpayError.category,
    http_code: openpayError.http_code,
    request_id: openpayError.request_id,
  });

  // ✅ Return generic error to client
  return NextResponse.json(
    {
      success: false,
      error: "Error al procesar el pago. Por favor intenta nuevamente.",
    },
    { status: 402 }
  );

  // ❌ WRONG - Exposing internal details
  return NextResponse.json({ error: openpayError.description });
}
```

## Database Security

### 7. Data Integrity

- [ ] Payment IDs stored with unique constraints
- [ ] Booking status transitions validated
- [ ] Foreign key constraints enforced
- [ ] Row-level security (RLS) policies enabled
- [ ] Audit trail for payment status changes
- [ ] Idempotency keys for duplicate prevention

**Supabase RLS policies:**
```sql
-- Users can only read their own bookings
CREATE POLICY "Users can read own bookings"
  ON bookings FOR SELECT
  USING (auth.uid() IN (
    SELECT clerk_id FROM profiles WHERE id = bookings.guest_id
  ));

-- Only authenticated users can create bookings
CREATE POLICY "Authenticated users can create bookings"
  ON bookings FOR INSERT
  WITH CHECK (auth.uid() IS NOT NULL);

-- Users cannot modify payment fields directly
CREATE POLICY "Users cannot modify payment fields"
  ON bookings FOR UPDATE
  USING (auth.uid() IN (
    SELECT clerk_id FROM profiles WHERE id = bookings.guest_id
  ))
  WITH CHECK (
    payment_id IS NOT DISTINCT FROM OLD.payment_id AND
    payment_method IS NOT DISTINCT FROM OLD.payment_method AND
    guest_total_cents IS NOT DISTINCT FROM OLD.guest_total_cents
  );
```

### 8. Sensitive Data

- [ ] Payment IDs encrypted at rest (if required by compliance)
- [ ] No PII in payment descriptions
- [ ] SPEI CLABE displayed only to booking owner
- [ ] OXXO references displayed only to booking owner
- [ ] Payment history access restricted to owners

## Client-Side Security

### 9. Frontend Validation

- [ ] Client-side validation for UX (never trust it for security)
- [ ] Server-side validation always enforced
- [ ] Payment method selection validated
- [ ] Amount displayed matches server calculation
- [ ] HTTPS enforced in production
- [ ] CSP headers configured

**Next.js security headers:**
```typescript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          {
            key: "Content-Security-Policy",
            value: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline' https://sandbox-api.openpay.mx; connect-src 'self' https://sandbox-api.openpay.mx;",
          },
          {
            key: "X-Frame-Options",
            value: "DENY",
          },
          {
            key: "X-Content-Type-Options",
            value: "nosniff",
          },
          {
            key: "Referrer-Policy",
            value: "strict-origin-when-cross-origin",
          },
        ],
      },
    ];
  },
};
```

### 10. Rate Limiting

- [ ] Payment creation rate limited per user
- [ ] Webhook endpoint rate limited globally
- [ ] Failed payment attempts tracked
- [ ] Suspicious activity alerts configured

**Implementation with Upstash Ratelimit:**
```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, "60 s"), // 5 requests per minute
});

export async function POST(request: NextRequest) {
  const { userId } = await auth();

  // Rate limit by user
  const { success } = await ratelimit.limit(`payment_${userId}`);
  if (!success) {
    return NextResponse.json(
      { error: "Demasiados intentos. Por favor intenta en un momento." },
      { status: 429 }
    );
  }

  // Process payment...
}
```

## Compliance & Legal

### 11. PCI DSS Compliance

- [ ] No card data stored on servers
- [ ] OpenPay SAQ-A compliance documented
- [ ] SSL/TLS certificates valid and current
- [ ] Regular security audits scheduled
- [ ] Incident response plan documented

### 12. Data Privacy

- [ ] Privacy policy covers payment data
- [ ] Terms of service include payment terms
- [ ] Refund policy clearly stated
- [ ] User data retention policy defined
- [ ] GDPR/local privacy laws compliance

### 13. Logging & Monitoring

- [ ] All payment attempts logged
- [ ] Failed payments logged with error codes
- [ ] Webhook deliveries logged
- [ ] Suspicious activity flagged
- [ ] Daily payment reconciliation

**Logging best practices:**
```typescript
// ✅ Good logging
console.log({
  event: "payment_created",
  bookingId: booking.id,
  paymentMethod: paymentMethod,
  amountCents: amountCents,
  userId: userId,
  timestamp: new Date().toISOString(),
});

// ❌ BAD - Logging sensitive data
console.log({
  cardNumber: cardNumber, // NEVER log
  cvv: cvv,               // NEVER log
  tokenId: tokenId,       // OK to log
});
```

## Production Deployment

### 14. Pre-Launch Checklist

- [ ] Switch from sandbox to production credentials
- [ ] Update webhook URLs to production domain
- [ ] Test production webhooks with ngrok/test payments
- [ ] Verify SSL certificate valid
- [ ] Test all payment methods in production sandbox
- [ ] Confirm OpenPay account verified/approved
- [ ] Set up monitoring alerts
- [ ] Document rollback procedure

### 15. Post-Launch Monitoring

- [ ] Monitor payment success rate
- [ ] Monitor webhook delivery rate
- [ ] Set up alerts for failed payments
- [ ] Set up alerts for webhook failures
- [ ] Daily reconciliation OpenPay vs database
- [ ] Weekly security audit of payment logs

## Testing Security

### 16. Security Testing

- [ ] Attempt payment without authentication
- [ ] Attempt payment for other user's booking
- [ ] Attempt payment with invalid booking status
- [ ] Attempt payment with manipulated amount
- [ ] Attempt webhook replay attack
- [ ] Attempt webhook with invalid signature
- [ ] Attempt SQL injection in payment description
- [ ] Attempt XSS in payment fields

**Penetration testing checklist:**
```bash
# Test authentication bypass
curl -X POST https://api.example.com/api/payments/create-charge \
  -H "Content-Type: application/json" \
  -d '{"bookingId":"other_user_booking","paymentMethod":"card"}'

# Test webhook signature bypass
curl -X POST https://api.example.com/api/webhooks/openpay \
  -H "Content-Type: application/json" \
  -d '{"type":"charge.succeeded","transaction":{"order_id":"test","amount":1000000}}'

# Test amount manipulation
curl -X POST https://api.example.com/api/payments/create-charge \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"bookingId":"valid_booking","paymentMethod":"card","amount":1}'
```

## Incident Response

### 17. Security Incident Plan

Document procedures for:
- [ ] Suspected data breach
- [ ] Unauthorized payment access
- [ ] Webhook compromise
- [ ] API key exposure
- [ ] User account compromise

**Response checklist:**
1. Immediately rotate compromised credentials
2. Notify affected users within 72 hours (GDPR)
3. Document incident timeline
4. Investigate root cause
5. Implement preventive measures
6. Update security documentation

## Regular Audits

### 18. Quarterly Security Review

- [ ] Review payment logs for anomalies
- [ ] Audit user permissions
- [ ] Test disaster recovery procedure
- [ ] Update dependencies for security patches
- [ ] Review OpenPay dashboard for warnings
- [ ] Verify backup/restore procedures
- [ ] Test incident response plan

## Resources

- [OpenPay Security Best Practices](https://www.openpay.mx/docs/security.html)
- [PCI DSS Compliance Guide](https://www.pcisecuritystandards.org/)
- [OWASP Payment Security Cheat Sheet](https://cheatsheetseries.owasp.org/)
- [Next.js Security Headers](https://nextjs.org/docs/app/building-your-application/configuring/security-headers)
