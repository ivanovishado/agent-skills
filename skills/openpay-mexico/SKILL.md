---
name: openpay-mexico
description: OpenPay payment integration for Mexican market with card, SPEI, and OXXO support. Use when integrating Mexican payment processing, adding OpenPay to Next.js/React apps, implementing SPEI/OXXO/card payments, handling payment webhooks, implementing Mexican market pricing strategies, or working with peso-based transactions. Covers SDK setup, webhook verification, pricing calculations in cents, payment UI components, and security patterns.
---

# OpenPay Mexico Integration

Integrate OpenPay payment processing for Mexican market with card, SPEI, and OXXO support.

## Integration Workflow

Choose the appropriate path based on your task:

**Setting up OpenPay from scratch?** → Follow "Initial Setup" below
**Adding payment features to existing integration?** → Jump to relevant section (Database, API Routes, Components, Webhooks)
**Implementing Mexican pricing strategy?** → See "Pricing Strategy" and references/pricing-strategy.md
**Debugging webhooks?** → See "Webhook Handler" and references/webhook-setup.md

## Initial Setup

### 1. Install OpenPay SDK

```bash
npm install openpay@^1.0.5 @types/openpay@^1.0.4
```

**Important:** Use version 1.0.5, not 2.0.0 (which doesn't exist). The SDK uses callback-based API that needs promisification.

### 2. Environment Variables

Add to `.env.local`:

```bash
OPENPAY_MERCHANT_ID=your_merchant_id
OPENPAY_PRIVATE_KEY=your_private_key
OPENPAY_PUBLIC_KEY=your_public_key
OPENPAY_WEBHOOK_SECRET=your_webhook_secret
OPENPAY_SANDBOX=true  # false for production
```

### 3. Database Schema

Add payment fields to bookings table. OpenPay uses these payment methods:
- **Card**: Immediate confirmation, 2.9% + $2.50 MXN fee
- **SPEI**: Bank transfer, instant confirmation, $8 MXN flat fee
- **OXXO**: Cash payment at stores, 24-72h confirmation, 2.9% + $2.50 MXN fee

**Critical:** Always store money in cents (BIGINT) not decimals (NUMERIC) to avoid floating-point precision errors.

```sql
-- Add payment tracking fields
ALTER TABLE bookings ADD COLUMN payment_id TEXT;
ALTER TABLE bookings ADD COLUMN payment_method TEXT CHECK (payment_method IN ('card', 'spei', 'oxxo'));

-- Store all money in cents to avoid floating-point issues
ALTER TABLE bookings ADD COLUMN guest_total_cents BIGINT;
ALTER TABLE bookings ADD COLUMN platform_fee_cents BIGINT;
ALTER TABLE bookings RENAME COLUMN total_mxn TO owner_price_cents;
ALTER TABLE bookings ALTER COLUMN owner_price_cents TYPE BIGINT;

-- SPEI-specific fields
ALTER TABLE bookings ADD COLUMN spei_clabe TEXT;
ALTER TABLE bookings ADD COLUMN spei_reference TEXT;

-- OXXO-specific fields
ALTER TABLE bookings ADD COLUMN oxxo_barcode_url TEXT;
ALTER TABLE bookings ADD COLUMN oxxo_reference TEXT;
ALTER TABLE bookings ADD COLUMN oxxo_expires_at TIMESTAMP WITH TIME ZONE;
```

Run migration with `npx supabase db push` or similar command for your stack.

## SDK Wrapper

Create `src/lib/openpay.ts` to promisify the callback-based SDK:

```typescript
import Openpay from "openpay";
import { promisify } from "util";

// Initialize OpenPay
const openpay = new Openpay(
  process.env.OPENPAY_MERCHANT_ID!,
  process.env.OPENPAY_PRIVATE_KEY!,
  process.env.OPENPAY_SANDBOX === "true"
);

// Promisify charges.create
const createChargeAsync = promisify(openpay.charges.create.bind(openpay.charges));

// Card charge (immediate confirmation)
export const createCardCharge = async (
  tokenId: string,
  amountCents: number,
  description: string,
  orderId: string,
  deviceSessionId: string,
): Promise<OpenPayCharge> => {
  const chargeRequest = {
    method: "card",
    source_id: tokenId,
    amount: amountCents / 100, // Convert cents to pesos for OpenPay API
    description,
    order_id: orderId,
    device_session_id: deviceSessionId,
    currency: "MXN",
    capture: true,
  };
  return createChargeAsync(chargeRequest);
};

// SPEI charge (async confirmation via webhook)
export const createSpeiCharge = async (
  amountCents: number,
  description: string,
  orderId: string,
): Promise<OpenPayCharge> => {
  const chargeRequest = {
    method: "bank_account",
    amount: amountCents / 100,
    description,
    order_id: orderId,
    currency: "MXN",
  };
  return createChargeAsync(chargeRequest);
};

// OXXO charge (async confirmation via webhook)
export const createOxxoCharge = async (
  amountCents: number,
  description: string,
  orderId: string,
  expirationDate: Date,
): Promise<OpenPayCharge> => {
  const chargeRequest = {
    method: "store",
    amount: amountCents / 100,
    description,
    order_id: orderId,
    currency: "MXN",
    due_date: expirationDate.toISOString().split("T")[0],
  };
  return createChargeAsync(chargeRequest);
};
```

## Pricing Strategy

Mexican market psychology requires careful pricing presentation. See references/pricing-strategy.md for detailed rationale.

**Key principle:** Never show upcharges. Present card price as base, SPEI as discount.

Create `src/lib/pricing.ts`:

```typescript
// All amounts in CENTS to avoid floating-point errors
export const PLATFORM_FEE_PERCENT = 0.1; // 10%
export const OPENPAY_CARD_PERCENT = 0.029; // 2.9%
export const OPENPAY_CARD_FIXED_CENTS = 250; // $2.50 MXN
export const OPENPAY_SPEI_FIXED_CENTS = 800; // $8.00 MXN (absorbed)

export const calculatePaymentOptions = (ownerPriceCents: number) => {
  // Platform fee (10%)
  const platformFeeCents = Math.ceil(ownerPriceCents * PLATFORM_FEE_PERCENT);
  const subtotalCents = ownerPriceCents + platformFeeCents;

  // Card fee baked into base price
  const cardFeeCents = Math.ceil(subtotalCents * OPENPAY_CARD_PERCENT + OPENPAY_CARD_FIXED_CENTS);
  const basePriceCents = subtotalCents + cardFeeCents;

  // SPEI is discounted (platform absorbs $8 fee)
  const speiPriceCents = subtotalCents;
  const speiSavingsCents = cardFeeCents;

  return {
    spei: {
      priceCents: speiPriceCents,
      label: "Transferencia SPEI",
      savingsCents: speiSavingsCents,
      savingsLabel: `¡Ahorra ${formatMXN(speiSavingsCents)}!`,
    },
    card: {
      priceCents: basePriceCents,
      label: "Tarjeta de crédito/débito",
    },
    oxxo: {
      priceCents: basePriceCents,
      label: "OXXO",
    },
    platformFeeCents,
    ownerReceivesCents: ownerPriceCents,
  };
};

// Conversion utilities
export const centsToPesos = (cents: number): number => cents / 100;
export const pesosToCents = (pesos: number): number => Math.round(pesos * 100);

// Format for display
export const formatMXN = (cents: number): string => {
  const pesos = centsToPesos(cents);
  return `$${pesos.toLocaleString("es-MX", {
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
  })}`;
};

// OXXO expires 48h from now
export const getOxxoExpirationDate = (): Date => {
  const now = new Date();
  now.setHours(now.getHours() + 48);
  return now;
};
```

## Payment API Route

Create `src/app/api/payments/create-charge/route.ts` as a Route Handler (not Server Action) because:
1. Needs raw response data for client components
2. Requires fine-grained error handling for payment failures
3. Returns different response shapes based on payment method

```typescript
import { NextRequest, NextResponse } from "next/server";
import { auth } from "@clerk/nextjs/server"; // or your auth solution
import { createServerSupabaseClient } from "@/lib/supabase-server";
import {
  createCardCharge,
  createSpeiCharge,
  createOxxoCharge,
} from "@/lib/openpay";
import { calculatePaymentOptions, getOxxoExpirationDate } from "@/lib/pricing";

export async function POST(request: NextRequest) {
  // 1. Authenticate
  const { userId } = await auth();
  if (!userId) {
    return NextResponse.json({ success: false, error: "No autorizado" }, { status: 401 });
  }

  // 2. Parse request
  const { bookingId, paymentMethod, tokenId, deviceSessionId } = await request.json();

  // 3. Validate card-specific requirements
  if (paymentMethod === "card" && (!tokenId || !deviceSessionId)) {
    return NextResponse.json(
      { success: false, error: "Token de tarjeta y session ID requeridos" },
      { status: 400 }
    );
  }

  // 4. Fetch booking
  const supabase = await createServerSupabaseClient();
  const { data: booking } = await supabase
    .from("bookings")
    .select("id, owner_price_cents, terrace_id, guest_id, status, terraces(name)")
    .eq("id", bookingId)
    .single();

  if (!booking) {
    return NextResponse.json({ success: false, error: "Reserva no encontrada" }, { status: 404 });
  }

  // 5. Verify ownership and status
  if (booking.status !== "approved") {
    return NextResponse.json(
      { success: false, error: "La reserva debe estar aprobada" },
      { status: 400 }
    );
  }

  // 6. Calculate amount based on payment method
  const paymentOptions = calculatePaymentOptions(booking.owner_price_cents);
  let amountCents: number;
  switch (paymentMethod) {
    case "card":
      amountCents = paymentOptions.card.priceCents;
      break;
    case "spei":
      amountCents = paymentOptions.spei.priceCents;
      break;
    case "oxxo":
      amountCents = paymentOptions.oxxo.priceCents;
      break;
  }

  // 7. Create OpenPay charge
  const description = `Depósito Terraza ${booking.terraces?.name || ""}`;
  let charge;

  try {
    switch (paymentMethod) {
      case "card":
        charge = await createCardCharge(tokenId!, amountCents, description, bookingId, deviceSessionId!);
        break;
      case "spei":
        charge = await createSpeiCharge(amountCents, description, bookingId);
        break;
      case "oxxo":
        const expirationDate = getOxxoExpirationDate();
        charge = await createOxxoCharge(amountCents, description, bookingId, expirationDate);
        break;
    }
  } catch (openpayError: any) {
    return NextResponse.json(
      { success: false, error: openpayError?.description || "Error al procesar el pago" },
      { status: 402 }
    );
  }

  // 8. Update booking with payment info
  const updateData: any = {
    payment_id: charge.id,
    payment_method: paymentMethod,
    guest_total_cents: amountCents,
    platform_fee_cents: paymentOptions.platformFeeCents,
  };

  // Add method-specific fields
  if (paymentMethod === "spei" && charge.payment_method) {
    updateData.spei_clabe = charge.payment_method.clabe;
    updateData.spei_reference = charge.payment_method.reference;
  } else if (paymentMethod === "oxxo" && charge.payment_method) {
    updateData.oxxo_barcode_url = charge.payment_method.barcode_url;
    updateData.oxxo_reference = charge.payment_method.reference;
    updateData.oxxo_expires_at = charge.due_date;
  }

  // Card payments confirm immediately
  if (paymentMethod === "card" && charge.status === "completed") {
    updateData.status = "confirmed";
  }

  await supabase.from("bookings").update(updateData).eq("id", bookingId);

  // 9. Return success
  return NextResponse.json({ success: true, charge }, { status: 200 });
}
```

## Webhook Handler

Create `src/app/api/webhooks/openpay/route.ts` for async payment confirmations (SPEI, OXXO).

**Critical security:** Always verify webhook signature with HMAC SHA256.

```typescript
import { NextRequest, NextResponse } from "next/server";
import { createServerSupabaseClient } from "@/lib/supabase-server";
import { headers } from "next/headers";
import crypto from "crypto";

// Verify signature
const verifyWebhookSignature = (payload: string, signature: string | null): boolean => {
  if (!signature) return false;

  const webhookSecret = process.env.OPENPAY_WEBHOOK_SECRET;
  if (!webhookSecret) return false;

  const hmac = crypto.createHmac("sha256", webhookSecret);
  hmac.update(payload);
  const expectedSignature = hmac.digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
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
  if (!bookingId) return NextResponse.json({ error: "Missing order_id" }, { status: 400 });

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

See references/webhook-setup.md for testing webhooks with ngrok/localtunnel.

## UI Components

Payment form should highlight SPEI as recommended option per Mexican market psychology.

Use Motion.dev for animations (not Framer Motion):
- Spring physics: `stiffness: 300, damping: 20`
- GPU-only animations: transform, opacity
- Staggered entrance: `delay: index * 0.1`

Example payment method selector:

```tsx
import { motion } from "motion/react";
import { RadioGroup, RadioGroupItem } from "@/components/ui/radio-group";
import { Building2, CreditCard, Store } from "lucide-react";

const PaymentForm = ({ ownerPriceCents }) => {
  const [selectedMethod, setSelectedMethod] = useState("spei");
  const options = calculatePaymentOptions(ownerPriceCents);

  return (
    <RadioGroup value={selectedMethod} onValueChange={setSelectedMethod}>
      {/* SPEI - Highlighted */}
      <motion.div
        whileHover={{ scale: 1.01 }}
        whileTap={{ scale: 0.99 }}
        transition={{ type: "spring", stiffness: 300, damping: 20 }}
      >
        <label className={selectedMethod === "spei" ? "border-emerald-500 bg-emerald-50/50" : ""}>
          <RadioGroupItem value="spei" id="spei" />
          <Building2 className="text-emerald-600" />
          <span>{options.spei.label}</span>
          <span className="bg-emerald-100 text-emerald-700 rounded-full px-2 py-0.5">
            Recomendado
          </span>
          <span className="text-2xl font-bold text-emerald-600">
            {formatMXN(options.spei.priceCents)}
          </span>
          <span className="text-emerald-600">{options.spei.savingsLabel}</span>
        </label>
      </motion.div>

      {/* Card and OXXO options follow similar pattern */}
    </RadioGroup>
  );
};
```

For OXXO vouchers, include print optimization:

```tsx
<style jsx global>{`
  @media print {
    body * { visibility: hidden; }
    .print\\:shadow-none * { visibility: visible; }
  }
`}</style>
```

## Security Checklist

Before going to production, verify:

1. ✅ Webhook signature verification enabled
2. ✅ All money stored in cents (BIGINT)
3. ✅ Payment verification checks booking status
4. ✅ Payment verification checks user ownership
5. ✅ Environment variables secured (never in client code)
6. ✅ HTTPS enabled for webhooks
7. ✅ Card tokenization on client side (never send raw card data to server)
8. ✅ Device session ID included for card payments (fraud detection)

See references/security-checklist.md for comprehensive security audit.

## Testing Flow

1. **Card payment**: Immediate confirmation, updates booking to "confirmed"
2. **SPEI payment**: Shows CLABE/reference, webhook confirms when paid (seconds)
3. **OXXO payment**: Shows barcode, webhook confirms 24-72h after payment

Test cards (sandbox):
- Success: 4111 1111 1111 1111
- Insufficient funds: 4000 0000 0000 0002
- See OpenPay docs for full test card list

## Resources

### references/

- **pricing-strategy.md** - Mexican market psychology, why SPEI discount works, upcharge psychology
- **webhook-setup.md** - ngrok/localtunnel setup, webhook testing, signature debugging
- **security-checklist.md** - Comprehensive payment security audit

### scripts/

Scripts are minimal for this skill as most operations are SDK-based. For migration generation or environment verification, create project-specific scripts as needed.
