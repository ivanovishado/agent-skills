# Mexican Market Pricing Strategy

## Core Principle: Never Show Upcharges

Mexican consumers are highly sensitive to upcharges, particularly payment processing fees. Showing a "3% card surcharge" triggers strong negative reactions and cart abandonment.

**Wrong approach:**
```
Base price: $5,000
Card surcharge: +$162 (3%)
Total: $5,162
```

**Right approach:**
```
Card payment: $5,162
SPEI transfer: $5,000 (¡Ahorra $162!)
```

Same economics, opposite psychology.

## Why SPEI Discount Works

### Trust in Bank Transfers

SPEI (Sistema de Pagos Electrónicos Interbancarios) is Mexico's real-time bank transfer system. It's:
- Government-backed and regulated
- Instant confirmation (seconds)
- Zero cost to consumer
- Widely trusted across all demographics

Positioning SPEI as the "smart choice" with visible savings aligns with:
1. **Price consciousness**: Mexican consumers actively seek savings
2. **Bank trust**: SPEI carries institutional credibility
3. **Social proof**: "Recomendado" badge validates the choice

### Card Fee Psychology

Rather than punishing card users with visible surcharges, we:
1. Build card fees into the "base price"
2. Present SPEI as a discount from that base
3. Frame the savings as a reward for choosing SPEI

This preserves card as an option (important for conversion) while steering toward SPEI (lower cost for platform).

## Implementation Strategy

### Pricing Calculation

```typescript
// Start with owner's desired revenue
const ownerPriceCents = 500000; // $5,000

// Add platform fee (10%)
const platformFeeCents = 50000; // $500
const subtotalCents = 550000; // $5,500

// Calculate card processing fees
const cardFeeCents = Math.ceil(550000 * 0.029 + 250); // $162
const cardPriceCents = 566200; // $5,662 (base price)

// SPEI is subtotal (platform absorbs $8 fee)
const speiPriceCents = 550000; // $5,500
const speiSavingsCents = 16200; // $162 saved

// Present to user:
// Card: $5,662 (base)
// SPEI: $5,500 (¡Ahorra $162!)
// OXXO: $5,662 (same as card)
```

### Visual Hierarchy

**SPEI must be the visual focus:**
- ✅ Emerald green (trust, money, nature)
- ✅ "Recomendado" badge
- ✅ Larger font for savings amount
- ✅ Positioned first in list

**Card and OXXO are presented neutrally:**
- Blue for card (standard)
- Amber for OXXO (brand colors)
- No negative framing
- Available but not promoted

## Money Handling: Always Use Cents

### Why Cents Matter

Floating-point arithmetic with decimals causes precision errors:

```typescript
// WRONG - Floating point errors
const price = 5000.00;
const fee = price * 0.029; // 145.00000000000003
const total = price + fee + 2.50; // 5147.50000000000003

// RIGHT - Integer cents
const priceCents = 500000;
const feeCents = Math.ceil(priceCents * 0.029 + 250); // 14750
const totalCents = priceCents + feeCents; // 514750
```

### Database Schema

```sql
-- WRONG
ALTER TABLE bookings ADD COLUMN total_mxn NUMERIC(10, 2);

-- RIGHT
ALTER TABLE bookings ADD COLUMN total_cents BIGINT;
```

### Conversion Pattern

```typescript
// Store cents internally
const ownerPriceCents = 500000;

// Convert to pesos ONLY for display or API calls
const ownerPricePesos = ownerPriceCents / 100; // 5000.00

// Format for display
const formatted = `$${ownerPricePesos.toLocaleString("es-MX")}`; // "$5,000"
```

## OpenPay Fee Structure

### Card Payments
- **Rate**: 2.9% + $2.50 MXN fixed
- **When charged**: Immediate on card capture
- **Confirmation**: Synchronous
- **Refunds**: Full fee refunded if transaction reversed

### SPEI Payments
- **Rate**: $8 MXN flat fee
- **When charged**: When transfer received
- **Confirmation**: Webhook within seconds
- **Strategy**: Platform absorbs fee (only $8 vs $162 card fee)

### OXXO Payments
- **Rate**: 2.9% + $2.50 MXN fixed
- **When charged**: When customer pays at store
- **Confirmation**: Webhook 24-72h after payment
- **Expiration**: 48h default (configurable)

## Platform Economics

For a $5,000 terrace:

**Card payment:**
- Guest pays: $5,662
- Owner receives: $5,000
- Platform receives: $500 (10%)
- OpenPay fee: $162

**SPEI payment:**
- Guest pays: $5,500
- Owner receives: $5,000
- Platform receives: $492 ($500 - $8 SPEI fee)
- OpenPay fee: $8

**Net platform impact:**
- SPEI saves guest $162
- Platform gives up $8 to incentivize SPEI
- ROI: Spend $8 to save $154 in processing

## Testing Pricing Display

Verify these behaviors:

1. ✅ SPEI is visually prominent with "Recomendado"
2. ✅ SPEI savings are clearly displayed
3. ✅ Card price is never presented as "price + surcharge"
4. ✅ All calculations use cents internally
5. ✅ Display formats show clean peso amounts
6. ✅ No floating-point precision issues in totals

## Cultural Considerations

### Price Anchoring

Mexican consumers often encounter:
- Street vendors: Cash only, no fees
- Mercado Libre: 12x interest-free installments
- Physical stores: "5% descuento en efectivo"

This primes expectation that cash/transfer = better deal.

### Language Framing

**Use:**
- "¡Ahorra $162!" (Save $162!)
- "Recomendado" (Recommended)
- "Sin comisiones" (No commissions)
- "Gratis y segura" (Free and secure)

**Avoid:**
- "Cargo adicional" (Additional charge)
- "Comisión de tarjeta" (Card commission)
- "Recargo" (Surcharge)
- "Costo extra" (Extra cost)

### Trust Signals

For SPEI:
- Bank logos (when possible)
- "Confirmación instantánea"
- "Protegido por el Banco de México"

For cards:
- Visa/Mastercard logos
- "Pago seguro"
- SSL badges

For OXXO:
- OXXO brand colors (red/amber)
- "Paga en cualquier OXXO"
- "Válido por 48 horas"
