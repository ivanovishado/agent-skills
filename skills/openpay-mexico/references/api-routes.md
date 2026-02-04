# Payment API Route

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

export async function POST(request: NextRequest) {
  // 1. Authenticate
  const { userId } = await auth();
  if (!userId) {
    return NextResponse.json(
      { success: false, error: "No autorizado" },
      { status: 401 },
    );
  }

  // 2. Parse request
  const { bookingId, paymentMethod, tokenId, deviceSessionId } =
    await request.json();

  // 3. Validate card-specific requirements
  if (paymentMethod === "card" && (!tokenId || !deviceSessionId)) {
    return NextResponse.json(
      { success: false, error: "Token de tarjeta y session ID requeridos" },
      { status: 400 },
    );
  }

  // 4. Fetch booking
  const supabase = await createServerSupabaseClient();
  const { data: booking } = await supabase
    .from("bookings")
    .select(
      "id, owner_price_cents, terrace_id, guest_id, status, terraces(name)",
    )
    .eq("id", bookingId)
    .single();

  if (!booking) {
    return NextResponse.json(
      { success: false, error: "Reserva no encontrada" },
      { status: 404 },
    );
  }

  // 5. Verify status
  if (booking.status !== "approved") {
    return NextResponse.json(
      { success: false, error: "La reserva debe estar aprobada" },
      { status: 400 },
    );
  }

  // 6. Create OpenPay charge
  // Note: Calculate amountCents based on your pricing strategy (see mexico-market skill)
  const amountCents = booking.owner_price_cents; // Adjust with fees as needed
  const description = `Depósito Terraza ${booking.terraces?.name || ""}`;
  let charge;

  try {
    switch (paymentMethod) {
      case "card":
        charge = await createCardCharge(
          tokenId!,
          amountCents,
          description,
          bookingId,
          deviceSessionId!,
        );
        break;
      case "spei":
        charge = await createSpeiCharge(amountCents, description, bookingId);
        break;
      case "oxxo":
        const expirationDate = new Date();
        expirationDate.setHours(expirationDate.getHours() + 48);
        charge = await createOxxoCharge(
          amountCents,
          description,
          bookingId,
          expirationDate,
        );
        break;
    }
  } catch (openpayError: any) {
    return NextResponse.json(
      {
        success: false,
        error: openpayError?.message || "Error al procesar el pago",
      },
      { status: 402 },
    );
  }

  // 7. Update booking with payment info
  const updateData: Record<string, unknown> = {
    payment_id: charge.id,
    payment_method: paymentMethod,
    guest_total_cents: amountCents,
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

  // 8. Return success
  return NextResponse.json({ success: true, charge }, { status: 200 });
}
```

## Response Handling

The route returns different data based on payment method:

| Method | Response Data                       | Next Step                              |
| ------ | ----------------------------------- | -------------------------------------- |
| Card   | `charge.status === "completed"`     | Booking confirmed, redirect to success |
| SPEI   | `charge.payment_method.clabe`       | Show CLABE to user                     |
| OXXO   | `charge.payment_method.barcode_url` | Show voucher to user                   |
