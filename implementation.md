# Ravvarnix Invoice Generator — Implementation Plan

---

## 1. Project Overview

A dedicated web dashboard for **Ravvarnix Renewable Energy Innovations Ltd.** to:
1. **Estimate project cost** (instant calculator)
2. **Generate a professional PDF quotation** that replicates and improves on the existing format, pre-filled with the company's fixed details and the client's entered data

The dashboard follows the Ravvarnix brand identity derived from the letterhead and logo.

---

## 2. Brand Identity (from Letterhead & Logo)

| Token | Value |
|---|---|
| **Primary Green** | `#1B5E20` (dark forest green — logo text "Ravvarnix") |
| **Accent Green** | `#2E7D32` (medium green — body elements) |
| **Accent Orange** | `#F57C00` (sun/rays in logo) |
| **Light Background** | `#F9FBF9` (off-white with green tint) |
| **Table Header BG** | `#E8F5E9` (light green) |
| **Border** | `#C8E6C9` |
| **Text Primary** | `#1A1A1A` |
| **Text Muted** | `#555555` |

**Typography:** Inter (clean, modern) — replaces the PDF's mixed typefaces  
**Logo:** Ravvarnix Solar sun + wordmark (top-left of every page)

---

## 3. Fixed Company Details (Hard-coded — Never changes)

```
Company Name:   Ravvarnix Renewable Energy Innovations Ltd.
Address:        Old SBI Road, Near Post Office, Ram Nagar,
                Supela, Bhilai - 490023
GST No.:        22AAPCR5712G1ZF
E-mail:         ravvarnixsolar@gmail.com
Contact:        9691977558
```

These values are baked into the PDF template and NOT exposed on the dashboard.

---

## 4. Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  🌿 Ravvarnix Solar         [Cost Calculator]  [New Quotation]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│  │   COST CALCULATOR       │  │   RECENT QUOTATIONS         │  │
│  │   (always visible)      │  │   (table of past invoices)  │  │
│  └─────────────────────────┘  └─────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Two main features on the sidebar / tabs:**
- **Cost Calculator** — quick estimate (no client info needed)
- **Generate Quotation** — full form → PDF → email/SMS delivery

---

## 5. Feature 1: Cost Calculator

A lightweight panel on the dashboard homepage. No client details needed.

### Input Fields:
| Field | Type | Notes |
|---|---|---|
| Project Capacity (kW) | Number input | e.g. 4.9 |
| Rate per kW (₹) | Number input | e.g. 33,000 |
| GST % | Number input | default 8.9 |
| Residential under Subsidy | Checkbox | shows subsidy row |
| Subsidy Amount (₹) | Number input | visible only if checkbox checked |

### Live Output (updates on every keystroke):
```
┌──────────────────────────────────────────┐
│  ESTIMATED COST BREAKDOWN                │
├──────────────────────────────────────────┤
│  Capacity          :   4.9 kW            │
│  Base Amount       :   ₹ 1,61,700        │
│  GST (8.9%)        :   ₹  14,391         │
│  Net Payable       :   ₹ 1,76,091        │
│  Subsidy (MNRE)    : - ₹ 1,08,000        │
│  ─────────────────────────────────────── │
│  Effective Cost    :   ₹    68,091  ✦    │
├──────────────────────────────────────────┤
│  [Generate Quotation with these values]  │
└──────────────────────────────────────────┘
```

Clicking **"Generate Quotation with these values"** pre-fills the quotation form below with the calculator values and scrolls to the form.

---

## 6. Feature 2: Generate Quotation (Full Form)

A multi-section form. "Generate PDF" button is disabled until required fields are filled.

---

### Section A — Quotation Meta

| Field | Type | Default | Notes |
|---|---|---|---|
| Quotation Number | Text input | Auto: `2026-27/RRE/001` (serialized) | User can override |
| Date | Date picker | Today's date | User can change |

**Auto-serialization logic:**  
Format: `{FY}/{prefix}/{sequence}`  
e.g. FY 2026-27 → `2026-27/RRE/001`, `002`, `003`...  
Stored in DB, increments on each generated quotation.

---

### Section B — Client / Customer Details

| Field | Type | Required | Notes |
|---|---|---|---|
| Client Name | Text | Yes | |
| Client Address | Textarea | No | Multi-line |
| Client Email | Email input | **Yes** (for delivery) | Validated format |
| Client Phone | Phone input | **Yes** (for SMS/WhatsApp) | With country code |

Required fields show inline error if empty on submit attempt.  
"Generate PDF" button stays **disabled** until email + phone are filled.

---

### Section C — Project Details

| Field | Type | Default | Notes |
|---|---|---|---|
| Project Capacity (kW) | Number input | — | e.g. 4.9 |
| Residential under Subsidy | Checkbox | unchecked | Controls Row 3 in Estimate + project label |
| Rate per kW (₹) | Number input | — | e.g. 33,000 |
| GST % | Number input | 8.9 | Pre-filled, editable |
| Subsidy Amount (₹) | Number input | — | **Visible only if checkbox checked** |

---

### Section D — Live Estimate Preview (read-only, auto-computed)

Mirrors the Estimate table that will appear in the PDF. Updates in real-time.

| Sr. No. | Particulars | Capacity (kW) | Rate per kW | Amount (₹) |
|---|---|---|---|---|
| 1 | Design, Engineering, Supply and Installation of grid connected rooftop solar project | `← project capacity` | `← rate per kW` | `capacity × rate` |
| 2 | Add: GST @ `{gst}%` (subject to change as per government norms) | — | — | `amount × gst/100` |
| — | **Net amount payable by customer** | | | **subtotal + GST** |
| 3 | Subsidy amount to be claimed directly from MNRE by the consumer (Refer Note 7) | — | — | `← subsidy amount` |
| — | ***Effective cost of the system*** | | | ***net − subsidy*** |

**Row 3 and "Effective cost" row are hidden if "Residential under Subsidy" is unchecked.**

---

### Section E — Action Buttons

```
[  Save Draft  ]   [  Preview PDF  ]   [  Generate & Send  ▶  ]
```

- **Save Draft** — saves to DB without sending, no required-field enforcement
- **Preview PDF** — opens a modal with rendered PDF preview
- **Generate & Send** — triggers the confirmation alert (see below)

---

## 7. Confirmation Alert (Before Final PDF Generation)

Before generating the PDF and sending, show a modal:

```
┌─────────────────────────────────────────────────────┐
│  ⚠️  Confirm & Send Quotation                       │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Quotation: 2026-27/RRE/001                         │
│  Client:    Ajay Verma                              │
│  Amount:    ₹ 2,61,360                              │
│                                                     │
│  This will be sent to:                              │
│  📧 Email:    ajay@example.com                      │
│  📱 WhatsApp: +91 94079 80242                       │
│                                                     │
│  [ ✉️  Send via Email     ]  ☑ (default on)         │
│  [ 💬  Send via WhatsApp  ]  ☑ (default on)         │
│                                                     │
│  [  Cancel  ]          [  Confirm & Generate  ]     │
└─────────────────────────────────────────────────────┘
```

User can toggle email/WhatsApp independently. At least one must be selected to proceed.

---

## 8. PDF Template Structure

The generated PDF exactly replicates the Ravvarnix quotation format but with a cleaner visual design. Pages:

---

### Page 1 — Letterhead Header + Client + Estimate + Specs

**Header Block (fixed — letterhead)**
```
┌──────────────────────────────────────────────────────────────────┐
│  [LOGO]   Ravvarnix Renewable Energy Innovations Ltd.            │
│           Old SBI Road, Near Post Office, Ram Nagar,             │
│           Supela, Bhilai - 490023                                │
│           GST: 22AAPCR5712G1ZF  |  ravvarnixsolar@gmail.com     │
│           Contact: 9691977558                                    │
└──────────────────────────────────────────────────────────────────┘
```
Clean two-column layout: Logo left | Company details right with a green accent divider line.

**Quotation Info + Client Block (flexible)**
```
┌───────────────────────────┬──────────────────────────────────────┐
│ Quotation No: 2026-27/... │ Client Name: Ajay Verma              │
│ Date: 15.05.2026          │ Address: Ramdev Medical Store...     │
│                           │ Project Capacity: 4.9kW              │
│                           │ (Residential under Subsidy)          │
│                           │ Contact: 9407980242                  │
└───────────────────────────┴──────────────────────────────────────┘
```

**ESTIMATE Table (semi-flexible — rows fixed, values flexible)**
Green header band. Bold total rows. Subsidy row conditional.

**PRICE INCLUSIONS AND TECHNICAL SPECIFICATIONS Table (fully fixed)**
All spec rows as in original PDF, slightly wider column spacing, consistent font.

---

### Page 2 — Terms (fully fixed)

Exact content from the PDF:
- Documents Required
- Commercial Terms (points 1–9)
- Bank Account Details (HDFC Bank, IFSC: HDFC0000734, Acc: 50200120103614, Current Account)

---

### Page 3 — Signature (fully fixed)

- Right-aligned: "Authorized Signatory"
- "Ravvarnix, Renewable Energy Innovations (OPC) Pvt. Ltd."
- Space for physical stamp/signature

---

## 9. Calculation Logic

```ts
const baseAmount     = projectCapacityKW * ratePerKW
const gstAmount      = baseAmount * (gstPercent / 100)
const netPayable     = baseAmount + gstAmount
const subsidyAmount  = isSubsidy ? subsidyInput : 0
const effectiveCost  = netPayable - subsidyAmount
```

All values formatted in Indian number system: `₹2,61,360`

---

## 10. Email Delivery

**Subject:** `Quotation #{quotationNo} from Ravvarnix Renewable Energy Innovations Ltd.`

**Body:**
```
Dear {clientName},

Please find attached the quotation for your {capacityKW}kW rooftop
solar installation.

Quotation No: {quotationNo}
Date: {date}
Net Amount Payable: ₹{netPayable}
{if subsidy: Effective Cost after Subsidy: ₹{effectiveCost}}

For any questions, please contact us at ravvarnixsolar@gmail.com
or call 9691977558.

Warm regards,
Ravvarnix Renewable Energy Innovations Ltd.
```
**Attachment:** `Quotation_{quotationNo}_{clientName}.pdf`

---

## 11. WhatsApp / SMS Delivery

```
Hi {clientName}! 🌞

Your solar project quotation from *Ravvarnix Renewable Energy* is ready.

📋 Quotation No: {quotationNo}
⚡ Capacity: {capacityKW}kW
💰 Net Payable: ₹{netPayable}
{if subsidy: 🏛️ Post-Subsidy Cost: ₹{effectiveCost}}

Download your quotation here:
{pdfLink}

For queries: ravvarnixsolar@gmail.com | 9691977558
```

---

## 12. Tech Stack

| Layer | Technology |
|---|---|
| **Framework** | Next.js 14 (App Router, TypeScript) |
| **UI Components** | shadcn/ui + Tailwind CSS |
| **Color Theme** | Custom Ravvarnix green/orange tokens in Tailwind config |
| **PDF Generation** | `@react-pdf/renderer` — React components → PDF (no headless browser needed) |
| **PDF Preview** | Same React-PDF component rendered in modal |
| **Email** | Resend API (or Nodemailer + Gmail) |
| **WhatsApp/SMS** | Twilio API |
| **Database** | SQLite + Prisma (dev) → PostgreSQL/Neon (prod) |
| **PDF Storage** | Local `/public/generated/` (dev) → Cloudflare R2 (prod) |
| **Hosting** | Vercel |

---

## 13. Database Schema

```prisma
model Quotation {
  id              String   @id @default(cuid())
  quotationNumber String   @unique   // e.g. "2026-27/RRE/001"
  sequence        Int      @default(autoincrement())

  // Client (flexible)
  clientName      String
  clientEmail     String
  clientPhone     String
  clientAddress   String?

  // Project (flexible)
  date            DateTime @default(now())
  capacityKW      Float
  ratePerKW       Float
  gstPercent      Float    @default(8.9)
  isSubsidy       Boolean  @default(false)
  subsidyAmount   Float    @default(0)

  // Computed (stored for history)
  baseAmount      Float
  gstAmount       Float
  netPayable      Float
  effectiveCost   Float

  // Delivery
  pdfUrl          String?
  emailSentAt     DateTime?
  whatsappSentAt  DateTime?
  status          String   @default("draft")  // draft | sent | viewed

  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

model QuotationSequence {
  id              Int    @id @default(1)
  currentFY       String   // "2026-27"
  lastSequence    Int      @default(0)
}
```

---

## 14. Repository Structure

```
Invoice_generator/
├── app/
│   ├── layout.tsx                    # Root layout with Ravvarnix branding
│   ├── page.tsx                      # Dashboard homepage (calculator + recent)
│   ├── quotation/
│   │   ├── new/page.tsx              # New quotation form
│   │   └── [id]/page.tsx            # View / resend existing quotation
│   └── api/
│       ├── quotation/route.ts        # POST: save quotation to DB
│       ├── pdf/generate/route.ts     # POST: generate PDF
│       ├── send/email/route.ts       # POST: send via email
│       └── send/whatsapp/route.ts    # POST: send via WhatsApp
│
├── components/
│   ├── dashboard/
│   │   ├── CostCalculator.tsx        # Feature 1: live calculator panel
│   │   └── RecentQuotations.tsx      # Table of past quotations
│   ├── quotation/
│   │   ├── QuotationForm.tsx         # Main form (Sections A–E)
│   │   ├── EstimatePreview.tsx       # Live estimate table in form
│   │   └── ConfirmSendModal.tsx      # Alert modal before PDF generation
│   └── pdf/
│       ├── QuotationPDF.tsx          # @react-pdf/renderer document
│       ├── PDFHeader.tsx             # Fixed letterhead component
│       ├── PDFEstimate.tsx           # Estimate table (dynamic rows)
│       ├── PDFSpecs.tsx              # Fixed specs table
│       ├── PDFTerms.tsx              # Fixed commercial terms
│       └── PDFSignature.tsx          # Fixed signature page
│
├── lib/
│   ├── calculations.ts               # Cost computation functions
│   ├── quotation-number.ts           # Auto-serialization logic
│   ├── email.ts                      # Resend/Nodemailer client
│   ├── twilio.ts                     # Twilio WhatsApp + SMS
│   ├── storage.ts                    # PDF file save/read
│   └── prisma.ts                     # Prisma singleton
│
├── constants/
│   └── company.ts                    # Fixed company details (name, address, GST, etc.)
│
├── prisma/schema.prisma
├── .env.local
└── package.json
```

---

## 15. Build Order

### Phase 1 — Setup + Brand (Day 1)
- [ ] Initialize Next.js 14 + TypeScript + Tailwind + shadcn/ui
- [ ] Configure Ravvarnix color tokens in `tailwind.config.ts`
- [ ] Set up Prisma + SQLite, run first migration
- [ ] Create root layout with Ravvarnix header (logo + nav)
- [ ] Add company constants file

### Phase 2 — Cost Calculator (Day 2)
- [ ] Build `CostCalculator.tsx` with all inputs + live output
- [ ] Wire up calculation logic from `lib/calculations.ts`
- [ ] Dashboard homepage with calculator + placeholder recent table

### Phase 3 — Quotation Form (Day 3–4)
- [ ] Build `QuotationForm.tsx` with Sections A–D
- [ ] Build `EstimatePreview.tsx` (live table in the form)
- [ ] Subsidy checkbox toggling Row 3 logic
- [ ] Auto-quotation number generation
- [ ] Required field validation + disabled button state

### Phase 4 — PDF Template (Day 5–6)
- [ ] Build all PDF components using `@react-pdf/renderer`
- [ ] Match layout to original PDF with cleaner styling
- [ ] Conditional subsidy rows in `PDFEstimate.tsx`
- [ ] Fixed pages: specs, terms, signature
- [ ] Build `/api/pdf/generate` route
- [ ] Preview PDF in modal on dashboard

### Phase 5 — Email + WhatsApp (Day 7)
- [ ] Set up Resend API → `/api/send/email`
- [ ] Set up Twilio → `/api/send/whatsapp`
- [ ] Build `ConfirmSendModal.tsx` with toggles
- [ ] Wire up "Generate & Send" button end-to-end

### Phase 6 — History + Polish (Day 8–9)
- [ ] Build `RecentQuotations.tsx` table (quotation no, client, amount, status, actions)
- [ ] Resend from history
- [ ] Quotation sequence persistence across FY
- [ ] Toast notifications (success / error per channel)
- [ ] Form state warnings on unsaved changes

### Phase 7 — Deploy (Day 10)
- [ ] Deploy to Vercel
- [ ] Switch SQLite → Neon PostgreSQL
- [ ] Switch local PDF storage → Cloudflare R2
- [ ] End-to-end test with real email + WhatsApp

---

## 16. Environment Variables

```env
# Database
DATABASE_URL="file:./dev.db"

# Email
RESEND_API_KEY=re_...
FROM_EMAIL=invoices@ravvarnix.com

# Twilio (WhatsApp + SMS)
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886

# Storage
PDF_STORAGE_PATH=./public/generated

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
QUOTATION_PREFIX=RRE
```

---

## 17. Key UX Rules

- Quotation number auto-increments but is **always editable** before generating
- Date defaults to today but has a **date picker** for manual override
- Rate per kW and GST% are **sticky** — remembered from last session (localStorage)
- Subsidy amount field **animates in/out** when checkbox is toggled
- "Generate & Send" button shows a **spinner** and disables during processing
- After send: show green checkmarks for Email ✅ and WhatsApp ✅ independently
- All monetary values formatted as **Indian numbering** (₹2,61,360 not ₹261360)
- PDF always downloadable from the history table even after sending
