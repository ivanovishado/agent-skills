# Webhook Handler

Create `src/app/api/webhooks/openpay/route.ts` for async payment confirmations (SPEI, OXXO).

## Dashboard Configuration

Configure these events in OpenPay Dashboard → Webhooks:

| Category | Event        | Purpose                       |
| -------- | ------------ | ----------------------------- |
| Cargos   | Completados  | Card payment success          |
| SPEI     | Recibidos    | Bank transfer received        |
| Paynet   | Cobrado      | OXXO cash payment made        |
| Paynet   | Expirado     | Release terrace if unpaid     |
| Webhooks | Verificación | Required for URL verification |

## Implementation

**Critical security:** Always verify webhook signature with HMAC SHA256.

```typescript
import { NextRequest, NextResponse } from "next/server";
import { createServerSupabaseClient } from "@/lib/supabase-server";
import { headers } from "next/headers";
import crypto from "crypto";

// Verify signature
const verifyWebhookSignature = (
  payload: string,
  signature: string | null,
): boolean => {
  if (!signature) return false;

  const webhookSecret = process.env.OPENPAY_WEBHOOK_SECRET;
  if (!webhookSecret) return false;

  const hmac = crypto.createHmac("sha256", webhookSecret);
  hmac.update(payload);
  const expectedSignature = hmac.digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature),
  );
};

export async function POST(request: NextRequest) {
  try {
    // 1. Get raw body for signature verification
    const rawBody = await request.text();
    const headersList = await headers();
    const signature = headersList.get("x-openpay-signature");

    // 2. Verify signature (critical security step)
    const isValid = verifyWebhookSignature(rawBody, signature);
    if (!isValid) {
      return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
    }

    // 3. Parse event
    const event = JSON.parse(rawBody);
    const { type, transaction } = event;

    // 4. Handle event types
    switch (type) {
      case "charge.succeeded":
        return await handleChargeSucceeded(transaction);
      case "charge.failed":
        return handleChargeFailed(transaction);
      case "charge.cancelled":
        return await handleChargeCancelled(transaction);
      case "charge.pending":
        // OXXO charges start pending, don't update yet
        return NextResponse.json({ received: true }, { status: 200 });
      default:
        return NextResponse.json({ received: true }, { status: 200 });
    }
  } catch (error) {
    // Always return 200 to prevent OpenPay retries
    return NextResponse.json({ received: true }, { status: 200 });
  }
}

async function handleChargeSucceeded(transaction: any) {
  const bookingId = transaction.order_id;
  if (!bookingId)
    return NextResponse.json({ error: "Missing order_id" }, { status: 400 });

  const supabase = await createServerSupabaseClient();

  // Update to confirmed
  await supabase
    .from("bookings")
    .update({ status: "confirmed" })
    .eq("id", bookingId)
    .eq("payment_id", transaction.id)
    .eq("status", "approved");

  return NextResponse.json({ received: true }, { status: 200 });
}

async function handleChargeFailed(transaction: any) {
  // Log failure, optionally notify user
  return NextResponse.json({ received: true }, { status: 200 });
}

async function handleChargeCancelled(transaction: any) {
  const bookingId = transaction.order_id;
  if (!bookingId) return NextResponse.json({ received: true }, { status: 200 });

  const supabase = await createServerSupabaseClient();

  await supabase
    .from("bookings")
    .update({ status: "cancelled" })
    .eq("id", bookingId)
    .eq("payment_id", transaction.id);

  return NextResponse.json({ received: true }, { status: 200 });
}
```

## Local Testing with ngrok

1. Install ngrok: `npm install -g ngrok` or download from ngrok.com
2. Start your dev server: `npm run dev`
3. Start ngrok tunnel: `ngrok http 3000`
4. Copy the HTTPS URL (e.g., `https://abc123.ngrok.io`)
5. Configure in OpenPay Dashboard: `https://abc123.ngrok.io/api/webhooks/openpay`
6. Find verification code at `http://localhost:4040` (ngrok inspector)

## Static Domain (Free)

To avoid changing URLs each time:

1. Go to [ngrok Dashboard](https://dashboard.ngrok.com/cloud-edge/domains)
2. Create a free static domain (e.g., `your-name.ngrok-free.app`)
3. Start ngrok: `ngrok http --url=your-name.ngrok-free.app 3000`
4. Configure this permanent URL in OpenPay once
