# ClinicOS API — পুরো Flow (Interview-এর জন্য)

Interview-এ পুরো system বোঝানোর সহজ উপায় হলো তিন স্তরে বলা: **কাঠামো → একটা request কীভাবে চলে → মূল business flow-গুলো**। শেষে ইংরেজিতে বলার মতো একটা script আছে।

---

## ১. বড় ছবি (৩০ সেকেন্ডে)

ClinicOS একটা ২৪/৭ হাসপাতালের appointment আর front desk-এর system। রোগী নিজে online-এ book করে, অথবা admin front desk থেকে রোগীর হয়ে book করে। তারপর check-in, doctor-এর prescription, complete। টাকা নেওয়া হয় Stripe-এ online, অথবা desk-এ cash/card।

```text
Next.js (web)  ──REST + cookie──▶  Express API (modular monolith)  ──Prisma──▶  PostgreSQL
                                        │
                                        ├── Stripe (checkout + webhook)
                                        ├── Cloudinary (doctor photo)
                                        ├── SMTP (OTP, welcome, appointment email)
                                        └── node-cron (প্রতি মিনিটে unpaid hold expire)
```

**৪টা role:** `SUPER_ADMIN` (একজনই, seed দিয়ে বানানো), `ADMIN` (front desk-এর সব কাজ), `DOCTOR`, `PATIENT`।

---

## ২. একটা request কীভাবে চলে

```text
Request
 → helmet (security header) → cors → express.json → cookieParser
 → /api/v1/<module> route
 → checkAuth(roles)         JWT cookie যাচাই, role check
 → validateRequest(zod)     body ঠিক আছে কিনা
 → controller               req থেকে data বের করে
 → service                  business logic + Prisma (দরকারে transaction)
 → sendResponse             { success, message, meta, data }
 (কোথাও error হলে) → globalErrorHandler → Zod, Prisma, Multer আর AppError-কে সুন্দর status code-এ রূপান্তর
```

প্রতিটা module একই গঠনে লেখা: `route → controller → service` (+ `validation`, `interface`, `constant`)। এটাকে বলে **modular monolith**: একটাই deploy, কিন্তু ভেতরে ভাগ পরিষ্কার।

---

## ৩. মূল flow-গুলো (যে ক্রমে বাস্তবে ঘটে)

### Flow A: Setup (admin)

1. Super admin seed থেকে তৈরি হয়। তারপর সে **admin** বানায় (`POST /users/admins`)।
2. Admin **specialty** বানায়, তারপর **doctor** বানায় (`POST /users/doctors`)। System একটা random temporary password বানায়, শুধু hash DB-তে রাখে, আর আসল password email-এ পাঠায়। প্রথম login-এ password বদলানো বাধ্যতামূলক (`needPasswordChange`)।
3. Public sign up **সবসময় `PATIENT`** বানায়। Body-তে `role` পাঠালেও Zod সেটা ফেলে দেয়, তাই privilege escalation সম্ভব না।

### Flow B: Doctor-এর সময় খোলা (slot)

1. Doctor বা admin `POST /doctors/:id/slots` দিয়ে পাঠায়: তারিখ, `09:00–13:00`, ১৫ মিনিট, break।
2. **`buildSlots` একটা pure function** (DB ছাড়া, Vitest দিয়ে test করা)। এটা clinic-এর timezone ধরে range-কে slot-এ ভাগ করে। রাতের shift-ও সামলায় (`22:00–06:00` মানে পরের দিন পর্যন্ত)।
3. DB-তে সব সময় **UTC** থাকে। Clinic-এর "দিন" ঠিক হয় `CLINIC_TIME_ZONE` দিয়ে।
4. Overlap হলে 409। একই doctor-এর দুটো request একসাথে এলে `pg_advisory_xact_lock` দিয়ে একটা আরেকটার জন্য অপেক্ষা করে।
5. **Block:** doctor অসুস্থ হলে slot `BLOCKED` হয়। একই transaction-এ ওই slot-এর appointment cancel হয়। Commit-এর **পরে** refund আর রোগীকে email যায়।

### Flow C: রোগী doctor খোঁজে (public)

- `GET /public/doctors`: specialty, নাম দিয়ে search, তারিখ দিয়ে filter, pagination, আর প্রতিটা doctor-এর **পরের খালি slot** (একটা `groupBy` query, তাই N+1 নেই)।
- Public response-এ email, phone বা license **যায় না**, এর জন্য আলাদা select।
- `GET /doctors/:id/slots`: ১৪ দিনের খালি slot, দিন অনুযায়ী group করা।

### Flow D: Booking (সবচেয়ে গুরুত্বপূর্ণ) ⭐

```text
POST /appointments { slotId, paymentMode }
 └─ transaction:
     1. slot row lock (SELECT ... FOR UPDATE)
     2. যাচাই: slot OPEN? BLOCKED হলে 409; ১ ঘণ্টার কম আগে হলে 400; doctor paused?
               একই রোগী, একই doctor, একই দিনে আরেকটা আছে?
     3. hold শেষ হওয়া পুরনো booking থাকলে → EXPIRED
     4. appointment তৈরি (fee-র snapshot, number APT-2026-000123)
 └─ ONLINE  → PENDING_PAYMENT + Stripe checkout URL (৩০ মিনিটের hold)
 └─ AT_DESK → CONFIRMED (admin চাইলে সাথে সাথে টাকা নেয় + check-in)
```

- **রোগী** নিজে book করলে `patientId` সবসময় session থেকে আসে। **Admin** রোগীর হয়ে book করলে body থেকে `patientId` দেয়।
- **Double booking দুই স্তরে আটকানো:** slot row lock, আর DB-তে partial unique index। দুটো request একসাথে এলে একটা 201 পায়, আরেকটা 409।

### Flow E: Payment (Stripe)

```text
রোগী Stripe-এর page-এ pay করে
 → Stripe → POST /payments/stripe/webhook (raw body, signature যাচাই)
 → idempotent: একই event আবার এলে কিছুই বদলায় না
 → checkout.session.completed: payment SUCCEEDED + appointment CONFIRMED
     (এর মধ্যে appointment cancel হয়ে গেলে, বা slot অন্য কেউ নিয়ে নিলে → refund)
```

- **Cron (প্রতি মিনিটে):** ৩০ মিনিটে pay না করলে appointment `EXPIRED` হয়, slot আবার খালি হয়, আর Stripe link বন্ধ হয়।
- টাকা সবসময় **integer cents**-এ রাখা হয়, float কখনো না।
- Stripe call **transaction-এর বাইরে** হয়, যাতে বাইরের ধীর service-এর জন্য DB lock আটকে না থাকে।

### Flow F: হাসপাতালের দিন (state machine)

```text
PENDING_PAYMENT ──pay──▶ CONFIRMED ──check-in──▶ CHECKED_IN ──doctor complete──▶ COMPLETED
      │                    │    │
   (৩০ মি.)             cancel  no-show (শুরুর ৩০ মিনিট পরে)
      ▼                    ▼    ▼
   EXPIRED             CANCELLED NO_SHOW
```

1. **Check-in** (admin): শুধু `CONFIRMED` আর fee paid হলে, আর appointment-এর ২ ঘণ্টা আগে থেকে। Update শর্তসহ হয়, তাই দুজন একসাথে click করলে একজন 409 পায়।
2. **Doctor-এর আজকের তালিকা:** `GET /appointments?date=today&status=CHECKED_IN`। Frontend প্রতি ৫ সেকেন্ডে polling করে। আলাদা queue table নেই, কারণ **slot-ই queue**।
3. **Prescription** (`PUT /appointments/:id/prescription`): শুধু ওই appointment-এর doctor লিখতে পারে, অন্য কেউ চাইলে 403। এটা upsert, অর্থাৎ পুরোটা replace হয়।
4. **Complete:** `CHECKED_IN → COMPLETED`।

### Flow G: Cancel আর refund

- **রোগী** শুরুর ২ ঘণ্টা আগে পর্যন্ত cancel করতে পারে, পরে চাইলে 409। **Staff** যেকোনো সময় পারে।
- Online-এ pay করা থাকলে Stripe refund যায়, `idempotencyKey` সহ, যাতে দুবার refund না হয়। Desk-এ pay করা থাকলে `REFUNDED` চিহ্নিত হয়, টাকা desk থেকে ফেরত দেওয়া হয়।

### Flow H: রোগীর portal

- `/patient/me`, `/patient/appointments`, `/patient/stats`। **কোনো id client থেকে নেওয়া হয় না**, সব session থেকে আসে।
- অন্য রোগীর appointment চাইলে **404** (403 না), যাতে "এমন record আছে কিনা" সেটাও জানা না যায়। এটা **IDOR protection**।

### Flow I: Doctor photo

`multer` (memory, ≤ 2 MB) → **magic bytes** দেখে আসল type যাচাই → Cloudinary-তে server বানানো নামে upload → DB update → পুরনো ছবি মোছা।

---

## ৪. Interview-এ যে প্রশ্নগুলো আসবেই

| প্রশ্ন | তোমার উত্তরের মূল কথা |
|---|---|
| Double booking কীভাবে আটকাও? | Slot row lock + partial unique index। Race condition হলে একজন 409 পায়। |
| Webhook দুবার এলে? | `stripeEventId` unique, তাই একই event দ্বিতীয়বার কিছু বদলায় না (idempotent)। |
| অন্যের data দেখা আটকাও কীভাবে? | Patient route-এ id session থেকে আসে, অন্যের record চাইলে 404 (IDOR)। |
| Timezone? | DB-তে UTC, clinic-এর দিন `CLINIC_TIME_ZONE` দিয়ে; `@date-fns/tz` ব্যবহার করি, তাই DST থাকা দেশেও একই code চলে। |
| Email বা Stripe fail করলে? | এগুলো transaction-এর বাইরে, commit-এর পরে চলে। Email fail করলে শুধু log হয়, booking fail হয় না। |
| Realtime কেন polling? | এই আকারের clinic-এ ৫ সেকেন্ডের polling যথেষ্ট। Scale বাড়লে SSE, আর একাধিক server হলে Redis pub/sub। |
| Security? | Helmet, rate limit, trust proxy, Zod validation, role check, IDOR-safe route, magic bytes, cents-এ টাকা। |

---

## ৫. ইংরেজিতে বলার script

**৩০ সেকেন্ডের pitch:**

> "ClinicOS is a booking and front-desk system for a 24/7 hospital. Patients book online and pay with Stripe, or the front desk books for them. Then come check-in, the doctor's prescription, and completion. It's an Express and Prisma modular monolith on Postgres, with a Next.js frontend."

**২ মিনিটের walkthrough:**

> "Doctors open time ranges, and a pure, unit-tested function splits them into slots in the clinic's time zone while everything is stored in UTC. When a patient books, I lock the slot row inside a transaction, and a partial unique index is a second safety net, so two concurrent requests can never both win — one gets a 409. Online bookings hold the slot for 30 minutes; a Stripe webhook, verified and idempotent, confirms the payment, and a cron job expires unpaid holds. The appointment follows a strict state machine: pending payment, confirmed, checked in, completed. Every transition is a conditional update, so double clicks are safe. Patient routes never take an ID from the client — it always comes from the session — so there's no IDOR. External calls like Stripe and email run after the transaction commits, so a slow third party never holds a database lock."

---

## অনুশীলন

Flow D (booking) আর Flow E (webhook) কাগজে নিজে আঁকো, তারপর জোরে বলো, কোনো দেখা ছাড়া। এই দুটো ঠিকমতো বলতে পারলে interview-এর অর্ধেক কাজ হয়ে যায়।
