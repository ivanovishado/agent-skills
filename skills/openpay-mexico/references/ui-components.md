# UI Components

Payment form components for OpenPay integration.

## Payment Method Selector

Use Motion.dev for animations (not Framer Motion):

- Spring physics: `stiffness: 300, damping: 20`
- GPU-only animations: transform, opacity
- Staggered entrance: `delay: index * 0.1`

```tsx
import { motion } from "motion/react";
import { RadioGroup, RadioGroupItem } from "@/components/ui/radio-group";
import { Building2, CreditCard, Store } from "lucide-react";

interface PaymentFormProps {
  options: {
    spei: { priceCents: number; label: string; savingsLabel?: string };
    card: { priceCents: number; label: string };
    oxxo: { priceCents: number; label: string };
  };
  formatMXN: (cents: number) => string;
}

const PaymentForm = ({ options, formatMXN }: PaymentFormProps) => {
  const [selectedMethod, setSelectedMethod] = useState("spei");

  return (
    <RadioGroup value={selectedMethod} onValueChange={setSelectedMethod}>
      {/* SPEI - Highlighted */}
      <motion.div
        whileHover={{ scale: 1.01 }}
        whileTap={{ scale: 0.99 }}
        transition={{ type: "spring", stiffness: 300, damping: 20 }}
      >
        <label
          className={
            selectedMethod === "spei"
              ? "border-emerald-500 bg-emerald-50/50"
              : ""
          }
        >
          <RadioGroupItem value="spei" id="spei" />
          <Building2 className="text-emerald-600" />
          <span>{options.spei.label}</span>
          <span className="bg-emerald-100 text-emerald-700 rounded-full px-2 py-0.5">
            Recomendado
          </span>
          <span className="text-2xl font-bold text-emerald-600">
            {formatMXN(options.spei.priceCents)}
          </span>
          {options.spei.savingsLabel && (
            <span className="text-emerald-600">
              {options.spei.savingsLabel}
            </span>
          )}
        </label>
      </motion.div>

      {/* Card */}
      <motion.div
        whileHover={{ scale: 1.01 }}
        whileTap={{ scale: 0.99 }}
        transition={{ type: "spring", stiffness: 300, damping: 20 }}
      >
        <label
          className={
            selectedMethod === "card" ? "border-blue-500 bg-blue-50/50" : ""
          }
        >
          <RadioGroupItem value="card" id="card" />
          <CreditCard className="text-blue-600" />
          <span>{options.card.label}</span>
          <span className="text-2xl font-bold">
            {formatMXN(options.card.priceCents)}
          </span>
        </label>
      </motion.div>

      {/* OXXO */}
      <motion.div
        whileHover={{ scale: 1.01 }}
        whileTap={{ scale: 0.99 }}
        transition={{ type: "spring", stiffness: 300, damping: 20 }}
      >
        <label
          className={
            selectedMethod === "oxxo" ? "border-amber-500 bg-amber-50/50" : ""
          }
        >
          <RadioGroupItem value="oxxo" id="oxxo" />
          <Store className="text-amber-600" />
          <span>{options.oxxo.label}</span>
          <span className="text-2xl font-bold">
            {formatMXN(options.oxxo.priceCents)}
          </span>
        </label>
      </motion.div>
    </RadioGroup>
  );
};
```

## OXXO Voucher

Display the barcode and payment instructions for OXXO payments.

```tsx
interface OxxoVoucherProps {
  barcodeUrl: string;
  reference: string;
  amount: string;
  expiresAt: string;
}

const OxxoVoucher = ({
  barcodeUrl,
  reference,
  amount,
  expiresAt,
}: OxxoVoucherProps) => {
  return (
    <div className="print:shadow-none p-6 border rounded-lg">
      <div className="flex justify-between items-center mb-4">
        <img src="/paynet-logo.svg" alt="Paynet" className="h-8" />
        <img src="/oxxo-logo.svg" alt="OXXO" className="h-8" />
      </div>

      <div className="text-center mb-4">
        <img src={barcodeUrl} alt="Código de barras" className="mx-auto" />
        <p className="text-sm text-muted-foreground mt-2">
          Referencia: {reference}
        </p>
      </div>

      <div className="border-t pt-4">
        <p className="text-2xl font-bold text-center">{amount}</p>
        <p className="text-sm text-center text-muted-foreground">
          Válido hasta: {expiresAt}
        </p>
      </div>

      <div className="flex gap-2 mt-4">
        <Button onClick={() => window.print()} variant="outline">
          Imprimir
        </Button>
        <Button onClick={shareViaWhatsApp}>Enviar por WhatsApp</Button>
      </div>
    </div>
  );
};
```

## Print Optimization

For OXXO vouchers, include print optimization:

```tsx
<style jsx global>{`
  @media print {
    body * {
      visibility: hidden;
    }
    .print\\:shadow-none,
    .print\\:shadow-none * {
      visibility: visible;
    }
    .print\\:shadow-none {
      position: absolute;
      left: 0;
      top: 0;
    }
  }
`}</style>
```
