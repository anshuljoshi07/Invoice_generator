# Invoice Generator — Implementation Plan

---

## 1. Product Overview

A web-based dashboard that lets users fill in invoice details (client info, line items, tax, branding), preview the invoice live, generate a professional PDF, and instantly deliver it via **email** and/or **WhatsApp/SMS** to the customer — all from a single page.

---

## 2. Core Features

| Feature | Description |
|---|---|
| **Invoice Form** | Business info, client info, line items, taxes, discounts, notes |
| **Live Preview** | Real-time PDF-style preview as the user types |
| **Template Selection** | 3-5 built-in invoice templates (minimal, professional, branded) |
| **Logo Upload** | Business logo embedded in the generated PDF |
| **PDF Generation** | Server-side, pixel-perfect PDF with all invoice data |
| **Email Delivery** | Send PDF as attachment to customer email |
| **WhatsApp/SMS Delivery** | Send PDF link or message to customer phone number |
| **Invoice History** | List of all generated invoices with status tracking |
| **Download** | Always downloadable directly from the browser |

---

## 3. Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| **Framework** | Next.js 14 (App Router) | SSR + API routes in one project |
| **UI** | Tailwind CSS + shadcn/ui | Fast, clean dashboard components |
| **PDF Generation** | Puppeteer (server-side) | Pixel-perfect HTML→PDF conversion |
| **PDF Preview** | React component mirrors PDF template | Live preview without generating PDF |
| **Email** | Nodemailer + Gmail SMTP (or Resend API) | Simple, reliable, free tier |
| **WhatsApp** | Twilio WhatsApp API | Send PDF link to customer phone |
| **SMS Fallback** | Twilio SMS | If WhatsApp not available |
| **Database** | SQLite via Prisma (dev) / PostgreSQL (prod) | Store invoice records + history |
| **File Storage** | Local filesystem (dev) / Cloudflare R2 (prod) | Store generated PDFs |
| **Auth** | NextAuth.js (email magic link or Google OAuth) | Simple single-user or team auth |
| **Hosting** | Vercel (frontend + API) + R2 for storage | Free tier sufficient for MVP |

---

## 4. Repository Structure

```
Invoice_generator/
├── app/                          # Next.js App Router
│   ├── layout.tsx                # Root layout (nav, auth wrapper)
│   ├── page.tsx                  # Redirect to /dashboard
│   ├── dashboard/
│   │   └── page.tsx              # Invoice history list
│   ├── invoice/
│   │   ├── new/
│   │   │   └── page.tsx          # New invoice form + live preview
│   │   └── [id]/
│   │       └── page.tsx          # View / resend existing invoice
│   ├── api/
│   │   ├── invoice/
│   │   │   ├── route.ts          # POST: create invoice + generate PDF
│   │   │   └── [id]/
│   │   │       └── route.ts      # GET: fetch invoice details
│   │   ├── pdf/
│   │   │   └── generate/
│   │   │       └── route.ts      # POST: generate PDF from invoice data
│   │   ├── send/
│   │   │   ├── email/
│   │   │   │   └── route.ts      # POST: send PDF via email
│   │   │   └── whatsapp/
│   │   │       └── route.ts      # POST: send PDF via WhatsApp/SMS
│   │   └── upload/
│   │       └── route.ts          # POST: upload business logo
│
├── components/
│   ├── invoice/
│   │   ├── InvoiceForm.tsx       # Main multi-section form
│   │   ├── LineItemsTable.tsx    # Add/remove/edit line items
│   │   ├── TaxDiscountPanel.tsx  # Tax % + discount controls
│   │   └── LivePreview.tsx       # Right-side live preview pane
│   ├── templates/
│   │   ├── TemplateMinimal.tsx   # Clean minimal template
│   │   ├── TemplateProfessional.tsx
│   │   └── TemplateBranded.tsx   # Full color with logo
│   ├── dashboard/
│   │   ├── InvoiceTable.tsx      # History table with status badges
│   │   └── StatsCards.tsx        # Total invoiced, paid, pending
│   └── ui/                       # shadcn/ui components (auto-generated)
│
├── lib/
│   ├── pdf.ts                    # Puppeteer PDF generation logic
│   ├── email.ts                  # Nodemailer / Resend client
│   ├── twilio.ts                 # Twilio WhatsApp + SMS client
│   ├── storage.ts                # File save/read (local or R2)
│   ├── calculations.ts           # Subtotal, tax, discount, total math
│   └── prisma.ts                 # Prisma client singleton
│
├── prisma/
│   ├── schema.prisma             # DB schema
│   └── migrations/               # Auto-generated migrations
│
├── templates/                    # HTML templates for Puppeteer rendering
│   ├── minimal.html
│   ├── professional.html
│   └── branded.html
│
├── public/
│   └── uploads/                  # Business logos (dev only)
│
├── types/
│   └── invoice.ts                # Shared TypeScript types
│
├── .env.local                    # Environment variables
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 5. Database Schema (Prisma)

```prisma
model Invoice {
  id              String   @id @default(cuid())
  invoiceNumber   String   @unique  // e.g. INV-2024-0001
  
  // Business (sender) info
  businessName    String
  businessEmail   String
  businessPhone   String?
  businessAddress String?
  businessLogo    String?  // URL/path to logo file

  // Customer (recipient) info
  customerName    String
  customerEmail   String
  customerPhone   String   // for WhatsApp/SMS delivery
  customerAddress String?

  // Invoice metadata
  issueDate       DateTime @default(now())
  dueDate         DateTime
  currency        String   @default("INR")
  template        String   @default("professional")
  notes           String?
  terms           String?

  // Financials (stored for history — computed from line items)
  subtotal        Float
  taxAmount       Float    @default(0)
  discountAmount  Float    @default(0)
  total           Float

  // Status
  status          String   @default("draft")  // draft | sent | paid | overdue

  // Delivery tracking
  emailSentAt     DateTime?
  whatsappSentAt  DateTime?
  pdfUrl          String?   // stored PDF path/URL

  lineItems       LineItem[]
  taxes           Tax[]

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

model LineItem {
  id          String   @id @default(cuid())
  invoiceId   String
  invoice     Invoice  @relation(fields: [invoiceId], references: [id], onDelete: Cascade)

  description String
  quantity    Float
  unitPrice   Float
  amount      Float    // quantity * unitPrice

  createdAt   DateTime @default(now())
}

model Tax {
  id        String  @id @default(cuid())
  invoiceId String
  invoice   Invoice @relation(fields: [invoiceId], references: [id], onDelete: Cascade)

  name      String   // e.g. "GST", "CGST", "SGST"
  rate      Float    // percentage, e.g. 18.0
  amount    Float    // computed tax amount
}
```

---

## 6. Invoice Form — Field Reference

### Section 1: Your Business Info
- Business Name *
- Business Email *
- Business Phone
- Business Address (multi-line)
- Business Logo (file upload, PNG/JPG, max 2MB)

### Section 2: Bill To (Customer)
- Customer Name *
- Customer Email * (for PDF delivery)
- Customer Phone * (for WhatsApp/SMS delivery, with country code)
- Customer Address

### Section 3: Invoice Details
- Invoice Number * (auto-generated: `INV-YYYY-NNNN`, user can override)
- Issue Date * (default: today)
- Due Date * (default: today + 30 days)
- Currency (INR default, dropdown: INR / USD / EUR / GBP)
- Template (dropdown: Minimal / Professional / Branded)

### Section 4: Line Items (dynamic rows)
| # | Description | Qty | Unit Price | Amount |
Each row: add / remove button. "Add Item" button below.

### Section 5: Taxes & Discounts
- Discount: flat (₹) or percentage (%)
- Taxes: add multiple (name + rate %) — e.g. CGST 9%, SGST 9%
- Totals auto-compute: Subtotal → Discount → Tax → **Total**

### Section 6: Notes & Terms
- Notes (e.g. "Thank you for your business")
- Payment Terms (e.g. "Payment due within 30 days")

---

## 7. PDF Generation Flow

```
User clicks "Generate & Send"
        │
        ▼
POST /api/invoice  (save to DB, get invoice ID)
        │
        ▼
POST /api/pdf/generate
  → Load HTML template (professional.html)
  → Inject invoice data (Handlebars or string interpolation)
  → Launch Puppeteer (headless Chromium)
  → page.goto(rendered HTML URL or data URI)
  → page.pdf({ format: 'A4', printBackground: true })
  → Save PDF to /public/uploads/{invoiceId}.pdf (dev)
           or upload to Cloudflare R2 (prod)
  → Return PDF URL
        │
        ▼
[parallel]
POST /api/send/email       POST /api/send/whatsapp
  → Nodemailer                → Twilio API
  → Attach PDF file           → Send PDF URL as message
  → Send to customerEmail     → Send to customerPhone
  → Update emailSentAt        → Update whatsappSentAt
        │                            │
        └──────────┬─────────────────┘
                   ▼
         Update invoice status → "sent"
         Show success toast + invoice ID
```

---

## 8. Email Delivery

### Provider: Resend API (recommended) or Nodemailer + Gmail

**Email content:**
- Subject: `Invoice #{invoiceNumber} from {businessName}`
- Body: Clean HTML email with invoice summary table + total
- Attachment: `Invoice_{invoiceNumber}.pdf`

**Resend setup:**
```ts
import { Resend } from 'resend';
const resend = new Resend(process.env.RESEND_API_KEY);

await resend.emails.send({
  from: 'invoices@yourdomain.com',
  to: customerEmail,
  subject: `Invoice #${invoiceNumber} from ${businessName}`,
  html: emailTemplate,
  attachments: [{ filename: `Invoice_${invoiceNumber}.pdf`, content: pdfBuffer }]
});
```

---

## 9. WhatsApp / SMS Delivery

### Provider: Twilio

**WhatsApp message format:**
```
Hi {customerName}! 👋

You have a new invoice from *{businessName}*.

🧾 Invoice #{invoiceNumber}
📅 Due: {dueDate}
💰 Total: ₹{total}

Download your invoice here:
{pdfUrl}

For queries, contact: {businessEmail}
```

**Twilio setup:**
```ts
import twilio from 'twilio';
const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);

await client.messages.create({
  from: 'whatsapp:+14155238886',  // Twilio sandbox number
  to: `whatsapp:${customerPhone}`,
  body: messageText,
  mediaUrl: [pdfUrl]  // optional: attach PDF directly
});
```

**SMS fallback:** Same `client.messages.create` without `whatsapp:` prefix.

---

## 10. Live Preview Architecture

The right panel mirrors the selected template in React.
- No PDF is generated during preview — it's pure CSS/React rendering.
- The same data shape that feeds the preview also feeds Puppeteer.
- Template switching instantly re-renders the preview.

```tsx
// InvoiceForm.tsx  (split-screen layout)
<div className="grid grid-cols-2 gap-6">
  <InvoiceForm data={invoiceData} onChange={setInvoiceData} />
  <LivePreview data={invoiceData} template={selectedTemplate} />
</div>
```

---

## 11. Invoice Templates

### Template 1: Minimal
- White background, black text, thin borders
- Logo top-left, invoice number top-right
- Clean line items table, totals right-aligned

### Template 2: Professional
- Light gray header band with business name
- Color accent (blue) for table headers and totals
- Two-column layout: business info left, customer info right

### Template 3: Branded
- Full-color header (customizable primary color)
- Large logo placement
- Bold total section with colored background

---

## 12. Dashboard (Invoice History)

**Stats Row:**
- Total Invoiced (sum of all invoice totals)
- Paid Invoices (count + amount)
- Pending Invoices (count + amount)
- Overdue Invoices (count + amount, highlighted red)

**Invoice Table columns:**
| Invoice # | Customer | Date | Due Date | Amount | Status | Actions |
Actions: View · Resend Email · Resend WhatsApp · Download PDF

**Status badges:**
- `draft` → gray
- `sent` → blue
- `paid` → green
- `overdue` → red

---

## 13. Environment Variables

```env
# Database
DATABASE_URL="file:./dev.db"                   # SQLite for dev
# DATABASE_URL="postgresql://..."              # PostgreSQL for prod

# Email (choose one)
RESEND_API_KEY=re_...                          # Resend (recommended)
# SMTP_HOST=smtp.gmail.com                    # or Nodemailer + Gmail
# SMTP_PORT=587
# SMTP_USER=you@gmail.com
# SMTP_PASS=your_app_password

# Twilio (WhatsApp + SMS)
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886  # Sandbox
TWILIO_SMS_NUMBER=+1...

# Storage
PDF_STORAGE_PATH=./public/uploads             # Dev: local filesystem
# R2_ACCOUNT_ID=...                           # Prod: Cloudflare R2
# R2_ACCESS_KEY_ID=...
# R2_SECRET_ACCESS_KEY=...
# R2_BUCKET_NAME=invoices

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
INVOICE_NUMBER_PREFIX=INV
```

---

## 14. Build Order (Phased)

### Phase 1 — Project Setup (Day 1)
- [ ] Initialize Next.js 14 project with TypeScript + Tailwind + shadcn/ui
- [ ] Set up Prisma with SQLite, run first migration
- [ ] Create root layout with sidebar nav (Dashboard / New Invoice)
- [ ] Push to GitHub

### Phase 2 — Invoice Form (Day 2-3)
- [ ] Build `InvoiceForm.tsx` with all 6 sections
- [ ] Build `LineItemsTable.tsx` with dynamic add/remove rows
- [ ] Build `TaxDiscountPanel.tsx` with auto-computed totals
- [ ] Form state management (React state / React Hook Form + Zod)
- [ ] Logo upload component (client-side preview)

### Phase 3 — Live Preview + Templates (Day 4-5)
- [ ] Build `TemplateMinimal.tsx` component
- [ ] Build `TemplateProfessional.tsx` component
- [ ] Build `TemplateBranded.tsx` component
- [ ] Build `LivePreview.tsx` (renders selected template with form data)
- [ ] Split-screen layout: form left, preview right

### Phase 4 — PDF Generation (Day 6)
- [ ] Install Puppeteer, configure for Next.js API route
- [ ] Create HTML template files for each invoice style
- [ ] Build `/api/pdf/generate` route
- [ ] Test end-to-end: form data → PDF file → download link

### Phase 5 — Email & WhatsApp Delivery (Day 7-8)
- [ ] Set up Resend API, build `/api/send/email` route
- [ ] Set up Twilio sandbox, build `/api/send/whatsapp` route
- [ ] Build SMS fallback in same route
- [ ] Wire up "Generate & Send" button to full flow
- [ ] Show delivery status (sent / failed) per channel

### Phase 6 — Invoice Persistence & Dashboard (Day 9-10)
- [ ] Build `/api/invoice` POST route (save to DB)
- [ ] Build `/api/invoice/[id]` GET route
- [ ] Build Dashboard page with `InvoiceTable.tsx` + `StatsCards.tsx`
- [ ] Status management (draft → sent → paid → overdue)
- [ ] Resend actions from dashboard

### Phase 7 — Polish & Deploy (Day 11-12)
- [ ] Form validation (Zod schema, inline errors)
- [ ] Error handling (email fail, WhatsApp fail, PDF fail — graceful toasts)
- [ ] Responsive layout (works on tablet/mobile for viewing)
- [ ] Deploy to Vercel
- [ ] Switch SQLite → PostgreSQL (Neon free tier)
- [ ] Switch local storage → Cloudflare R2 for PDFs

---

## 15. Calculations Logic

```ts
// lib/calculations.ts

export function computeInvoiceTotals(lineItems, taxes, discount) {
  const subtotal = lineItems.reduce((sum, item) => sum + item.quantity * item.unitPrice, 0);
  
  const discountAmount = discount.type === 'percentage'
    ? subtotal * (discount.value / 100)
    : discount.value;

  const afterDiscount = subtotal - discountAmount;

  const taxAmount = taxes.reduce((sum, tax) => sum + afterDiscount * (tax.rate / 100), 0);

  const total = afterDiscount + taxAmount;

  return { subtotal, discountAmount, taxAmount, total };
}
```

---

## 16. Key UX Details

- **Auto invoice number**: system generates `INV-2024-0001`, increments automatically, user can override
- **Currency formatting**: all amounts formatted with correct symbol and 2 decimal places
- **Date pickers**: native `<input type="date">` or shadcn Calendar component
- **Unsaved state**: warn user if they navigate away with unsaved changes
- **Send confirmation modal**: shows summary (customer name, email, phone, total) before sending
- **Loading states**: spinner on "Generate & Send" button, disabled during processing
- **Success state**: show invoice number, green checkmarks for email ✅ and WhatsApp ✅
- **Download always available**: PDF URL stored in DB, re-downloadable from history

---

## 17. Estimated Costs (Production)

| Service | Free Tier | Cost Beyond |
|---|---|---|
| Vercel Hosting | 100GB bandwidth/month | $20/month pro |
| Neon PostgreSQL | 0.5 GB storage | $19/month |
| Resend Email | 3,000 emails/month | $20/month for 50k |
| Twilio WhatsApp | $0 sandbox (testing only) | ~$0.005/message |
| Cloudflare R2 | 10 GB storage, 1M requests | $0.015/GB after |

**MVP running cost at low volume: ~$0/month** (all free tiers sufficient for early usage)
