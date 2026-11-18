এই code যা করে (সাধারণ ভাষায়)

১. রোগী সময় বাছাই করে
রোগী website-এ doctor-এর খালি slot দেখে, যেমন ২৫ নভেম্বর সকাল ১০টা। যে slot আগেই book হয়ে গেছে, সেটা তালিকায় দেখায় না।

২. Booking করে, আর বেছে নেয় কীভাবে টাকা দেবে

“Clinic-এ এসে দেব” বাছলে appointment সাথে সাথে confirmed হয়, আর email আসে।
"এখনই online-এ দেব" বাছলে slot-টা ৩০ মিনিটের জন্য তার নামে আটকে রাখা হয়, আর তাকে Stripe-এর payment page-এ পাঠানো হয়।

দুজন একই সময়ে একই slot নিতে চাইলে একজন পায়, অন্যজন "এই সময়টা এইমাত্র নেওয়া হয়ে গেছে" message পায়।

৩. Online payment

৩০ মিনিটের মধ্যে pay করলে Stripe আমাদের server-কে জানায় (webhook)। তখন appointment confirmed হয় আর email যায়।
Pay না করলে প্রতি মিনিটে চলা cron job দেখে hold শেষ হয়ে গেছে। তখন appointment expired হয়, আর slot আবার অন্যদের জন্য খুলে যায়।

৪. Front desk-ও book করতে পারে

রোগী সামনে থাকলে desk book করে cash/card নিয়ে নেয়।
রোগী ফোনে থাকলে desk book করে রোগীকে payment link email করে।

৫. Cancel আর refund

রোগী নিজে appointment-এর ২ ঘণ্টা আগে পর্যন্ত cancel করতে পারে। Online-এ pay করে থাকলে Stripe-এ টাকা ফেরত যায়।
Doctor অসুস্থ হলে admin slot block করে। তখন ওই সময়ের সব appointment cancel হয়, টাকা ফেরত যায়, আর রোগীরা email পায়।

এখানে গল্প শেষ: রোগীর হাতে একটা confirmed appointment (APT-2026-000123), আর টাকা দেওয়া হয়েছে অথবা desk-এ দেবে।


# ClinicOS — Appointment + Payment (পুরো code)

> সব file তোমার `apps/api` folder ধরে। নতুন file হলে পুরোটা বসাও; পুরনো file-এ শুধু যেটুকু বদলেছে সেটা দেখানো আছে (`slot.service.ts` পুরোটা)।
>
> এই code আমি তোমার repo-র একটা copy-তে বসিয়ে `pnpm typecheck`, `pnpm lint`, `pnpm test` চালিয়েছি (সব pass), আর একটা আসল Postgres-এ booking-এর পুরো flow চালিয়ে দেখেছি (নিচে ১২ নম্বরে)।

## ১. Package install

```bash
pnpm add stripe node-cron
```

`package.json`-এর `scripts`-এ যোগ করো:

```json
"stripe:webhook": "stripe listen --forward-to localhost:5000/api/v1/payments/stripe/webhook"
```

## ২. Prisma schema

**`prisma/schema/enums.prisma`** — শেষে যোগ করো:

```prisma
enum AppointmentStatus {
  PENDING_PAYMENT
  CONFIRMED
  CHECKED_IN
  COMPLETED
  CANCELLED
  NO_SHOW
  EXPIRED
}

enum AppointmentSource {
  ONLINE
  STAFF
}

enum AppointmentPaymentMode {
  AT_DESK
  ONLINE
}

enum PaymentProvider {
  STRIPE
  DESK
}

enum PaymentStatus {
  PENDING
  SUCCEEDED
  FAILED
  REFUNDED
}

enum PaymentMethod {
  CASH
  CARD
  MOBILE
}
```


**`prisma/schema/scheduling.prisma`** — পুরোটা (DoctorSlot-এ `appointments` যোগ + নতুন `Appointment`)

```prisma
model DoctorSlot {
  id          String     @id @default(uuid())
  doctorId    String
  startAt     DateTime
  endAt       DateTime
  status      SlotStatus @default(OPEN)
  blockReason String?
  blockedAt   DateTime?
  blockedById String?
  createdById String
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  doctor       DoctorProfile @relation(fields: [doctorId], references: [id])
  appointments Appointment[]

  @@unique([doctorId, startAt])
  @@index([status, startAt])
  @@map("doctor_slots")
}

model Appointment {
  id                String                 @id @default(uuid())
  appointmentNumber String                 @unique
  patientId         String
  doctorId          String
  slotId            String
  startAt           DateTime
  endAt             DateTime
  status            AppointmentStatus
  source            AppointmentSource
  paymentMode       AppointmentPaymentMode
  feeCents          Int
  holdExpiresAt     DateTime?
  reason            String?
  bookedById        String
  cancelledAt       DateTime?
  cancelReason      String?
  cancelledById     String?
  createdAt         DateTime               @default(now())
  updatedAt         DateTime               @updatedAt

  patient  Patient       @relation(fields: [patientId], references: [id])
  doctor   DoctorProfile @relation(fields: [doctorId], references: [id])
  slot     DoctorSlot    @relation(fields: [slotId], references: [id])
  payments Payment[]

  @@index([slotId])
  @@index([patientId, startAt])
  @@index([doctorId, startAt])
  @@index([status, holdExpiresAt])
  @@map("appointments")
}
```


**`prisma/schema/billing.prisma`** — নতুন

```prisma
model Payment {
  id                    String          @id @default(uuid())
  appointmentId         String
  provider              PaymentProvider
  status                PaymentStatus   @default(PENDING)
  amountCents           Int
  currency              String          @default("usd")
  deskMethod            PaymentMethod?
  receivedById          String?
  stripeSessionId       String?         @unique
  stripePaymentIntentId String?         @unique
  stripeEventId         String?         @unique
  paidAt                DateTime?
  refundedAt            DateTime?
  createdAt             DateTime        @default(now())
  updatedAt             DateTime        @updatedAt

  appointment Appointment @relation(fields: [appointmentId], references: [id])

  @@index([appointmentId, status])
  @@map("payments")
}
```


**`prisma/schema/system.prisma`** — নতুন

```prisma
model Counter {
  key   String @id
  value Int

  @@map("counters")
}
```

**`prisma/schema/clinic.prisma`** — দুই জায়গায় relation:

```prisma
model Patient {
  // ... আগের সব field
  appointments Appointment[]
  // ...
}

model DoctorProfile {
  // ... আগের সব field
  appointments      Appointment[]
  slots             DoctorSlot[]
  // ...
}
```

### Migration (partial unique index হাতে যোগ করতে হয়)

```bash
pnpm format
pnpm prisma migrate dev --create-only --name add_appointment_payment
```

তৈরি হওয়া `prisma/migrations/<timestamp>_add_appointment_payment/migration.sql`-এর **একদম শেষে** এটা যোগ করো, তারপর `pnpm migrate`:

```sql
-- ===== হাতে যোগ করা (Prisma schema-তে partial index লেখা যায় না) =====
CREATE UNIQUE INDEX "appointments_one_active_per_slot"
ON "appointments" ("doctorId", "startAt")
WHERE status IN ('PENDING_PAYMENT', 'CONFIRMED', 'CHECKED_IN');

CREATE UNIQUE INDEX "payments_one_succeeded_per_appointment"
ON "payments" ("appointmentId")
WHERE status = 'SUCCEEDED';
```

## ৩. Env

**`src/app/config/env.ts`** — `EMAIL_FROM`-এর পরে, `})`-এর আগে:

```ts
		STRIPE_SECRET_KEY: z.string().startsWith("sk_", 'Must start with "sk_" (Stripe → Developers → API keys)'),
		STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_", 'Must start with "whsec_" (shown by "stripe listen")'),
```

**`.env` আর `.env.example`:**

```dotenv
# ── Stripe (sandbox) ─────────────────────────
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

## ৪. Config আর utils


**`src/app/config/appointment.config.ts`**

```ts
// Booking আর payment-এর নিয়ম (SDLC FR-11, FR-12)
export const APPOINTMENT_CONFIG = {
	// Online payment-এর জন্য slot কতক্ষণ আটকে থাকবে
	holdMinutes: 30,
	// Stripe-এর expires_at কমপক্ষে ৩০ মিনিট পরের হতে হয় (Stripe-এর নিজের সময় ধরে),
	// network-এর দেরির জন্য ১ মিনিট বেশি। Hold শেষে cron session নিজেই বন্ধ করে দেয়।
	stripeSessionMinutes: 31,
	// রোগী নিজে কত মিনিট আগে পর্যন্ত cancel করতে পারবে
	patientCancelBeforeMinutes: 120,
	currency: "usd",
	// Stripe-এ USD-র সর্বনিম্ন charge 50 cents
	minOnlineAmountCents: 50,
};
```


**`src/app/config/stripe.config.ts`**

```ts
import { Stripe } from "stripe";
import { envVars } from "./env";

// apiVersion না দিলে SDK যে version-এর জন্য বানানো সেটাই ব্যবহার হয় (types-এর সাথে মেলে)
export const stripe = new Stripe(envVars.STRIPE_SECRET_KEY);
```


**`src/app/utils/counter.ts`**

```ts
import type { Prisma } from "@/generated/prisma/client";

// Atomic counter: upsert + increment একটাই SQL (INSERT ... ON CONFLICT DO UPDATE)।
// দুটো transaction একসাথে এলে Postgres row lock একটার পর একটা চালায় → duplicate নম্বর হয় না।
export const nextCounter = async (tx: Prisma.TransactionClient, key: string): Promise<number> => {
	const counter = await tx.counter.upsert({
		where: { key },
		create: { key, value: 1 },
		update: { value: { increment: 1 } },
		select: { value: true },
	});
	return counter.value;
};
```


**`src/app/utils/money.ts`**

```ts
// 5000 (cents) → "$50.00"
export const formatMoney = (cents: number, currency: string): string =>
	new Intl.NumberFormat("en-US", { style: "currency", currency: currency.toUpperCase() }).format(cents / 100);
```

## ৫. Email

**`src/app/shared/renderEmail.ts`** — `TEmailTemplates`-এ `temporaryPasswordReset`-এর পরে যোগ করো:

```ts
	appointmentConfirmed: {
		name: string;
		appointmentNumber: string;
		doctorName: string;
		whenLabel: string;
		paymentLabel: string;
		portalUrl: string;
	};
	appointmentCancelled: {
		name: string;
		appointmentNumber: string;
		doctorName: string;
		whenLabel: string;
		reason: string;
		refundNote: string | null;
		bookUrl: string;
	};
	paymentLink: {
		name: string;
		appointmentNumber: string;
		doctorName: string;
		whenLabel: string;
		amountLabel: string;
		payUrl: string;
		expiresLabel: string;
	};
```


**`templates/emails/appointmentConfirmed.ejs`**

```html
<h1 style="margin: 0 0 12px 0; font-family: Arial, Helvetica, sans-serif; font-size: 26px; line-height: 32px; font-weight: bold; color: #0f172a">
	You're booked
</h1>

<p style="margin: 0 0 24px 0">Hi <%= name %>, your appointment at <%= clinicName %> is confirmed.</p>

<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td style="padding: 20px 24px; background-color: #eff4ff; border: 1px solid #c7d7fe; border-radius: 12px; font-family: Arial, Helvetica, sans-serif; font-size: 15px; line-height: 24px; color: #0f172a">
			<div style="font-size: 12px; font-weight: bold; letter-spacing: 2px; color: #2563eb">APPOINTMENT</div>
			<div style="padding-bottom: 12px; font-size: 18px; font-weight: bold"><%= appointmentNumber %></div>
			<div><strong>Doctor:</strong> Dr. <%= doctorName %></div>
			<div><strong>When:</strong> <%= whenLabel %></div>
			<div><strong>Payment:</strong> <%= paymentLabel %></div>
		</td>
	</tr>
</table>

<table role="presentation" cellpadding="0" cellspacing="0" border="0" style="margin: 24px 0">
	<tr>
		<td style="background-color: #2563eb; border-radius: 8px">
			<a href="<%= portalUrl %>" style="display: inline-block; padding: 12px 24px; font-family: Arial, Helvetica, sans-serif; font-size: 15px; font-weight: bold; color: #ffffff; text-decoration: none">View my appointments</a>
		</td>
	</tr>
</table>

<p style="margin: 0; color: #475569">Please arrive 10 minutes early and check in at the front desk.</p>
```


**`templates/emails/appointmentCancelled.ejs`**

```html
<h1 style="margin: 0 0 12px 0; font-family: Arial, Helvetica, sans-serif; font-size: 26px; line-height: 32px; font-weight: bold; color: #0f172a">
	Appointment cancelled
</h1>

<p style="margin: 0 0 24px 0">
	Hi <%= name %>, your appointment <strong style="color: #0f172a"><%= appointmentNumber %></strong> with Dr.
	<%= doctorName %> on <%= whenLabel %> has been cancelled.
</p>

<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td style="padding: 14px 16px; background-color: #fff1f2; border: 1px solid #fecdd3; border-radius: 10px; font-family: Arial, Helvetica, sans-serif; font-size: 14px; line-height: 21px; color: #0f172a">
			<strong>Reason:</strong> <%= reason %>
			<% if (refundNote) { %>
			<div style="padding-top: 8px"><%= refundNote %></div>
			<% } %>
		</td>
	</tr>
</table>

<table role="presentation" cellpadding="0" cellspacing="0" border="0" style="margin: 24px 0">
	<tr>
		<td style="background-color: #2563eb; border-radius: 8px">
			<a href="<%= bookUrl %>" style="display: inline-block; padding: 12px 24px; font-family: Arial, Helvetica, sans-serif; font-size: 15px; font-weight: bold; color: #ffffff; text-decoration: none">Book another time</a>
		</td>
	</tr>
</table>

<p style="margin: 0; color: #475569">We're sorry for the inconvenience. Call <%= clinicPhone %> if you need help.</p>
```


**`templates/emails/paymentLink.ejs`**

```html
<h1 style="margin: 0 0 12px 0; font-family: Arial, Helvetica, sans-serif; font-size: 26px; line-height: 32px; font-weight: bold; color: #0f172a">
	Complete your payment
</h1>

<p style="margin: 0 0 24px 0">
	Hi <%= name %>, the front desk booked <strong style="color: #0f172a"><%= appointmentNumber %></strong> with Dr.
	<%= doctorName %> on <%= whenLabel %>. Pay the consultation fee of <strong style="color: #0f172a"><%= amountLabel %></strong>
	online to confirm it.
</p>

<table role="presentation" cellpadding="0" cellspacing="0" border="0" style="margin: 0 0 24px 0">
	<tr>
		<td style="background-color: #2563eb; border-radius: 8px">
			<a href="<%= payUrl %>" style="display: inline-block; padding: 12px 24px; font-family: Arial, Helvetica, sans-serif; font-size: 15px; font-weight: bold; color: #ffffff; text-decoration: none">Pay <%= amountLabel %></a>
		</td>
	</tr>
</table>

<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td style="padding: 14px 16px; background-color: #fffbeb; border-radius: 10px; font-family: Arial, Helvetica, sans-serif; font-size: 14px; line-height: 21px; color: #475569">
			<strong style="color: #0f172a">This link works until <%= expiresLabel %>.</strong> After that the time is released for other
			patients. Payments are processed securely by Stripe.
		</td>
	</tr>
</table>
```

## ৬. Payment module (`src/app/module/payment/`)


**`src/app/module/payment/payment.stripe.ts`**

```ts
import { APPOINTMENT_CONFIG } from "@/app/config/appointment.config";
import { envVars } from "@/app/config/env";
import { stripe } from "@/app/config/stripe.config";
import { AppError } from "@/app/errorHelpers/AppError";
import httpStatus from "http-status";
import type { Stripe } from "stripe";

type TCheckoutInput = {
	appointmentId: string;
	paymentId: string;
	appointmentNumber: string;
	doctorName: string;
	amountCents: number;
	currency: string;
	customerEmail: string;
};

// Stripe-এর hosted checkout page। Card-এর তথ্য আমাদের server-এ আসে না।
export const createCheckoutSession = async (input: TCheckoutInput) => {
	// metadata দিয়েই webhook-এ চিনব কোন payment আর কোন appointment
	const metadata = { appointmentId: input.appointmentId, paymentId: input.paymentId };

	const session = await stripe.checkout.sessions.create(
		{
			mode: "payment",
			customer_email: input.customerEmail,
			line_items: [
				{
					quantity: 1,
					price_data: {
						currency: input.currency,
						unit_amount: input.amountCents, // আগে থেকেই integer cents → Math.round লাগে না
						product_data: {
							name: `Consultation with Dr. ${input.doctorName}`,
							description: input.appointmentNumber,
						},
					},
				},
			],
			metadata,
			payment_intent_data: { metadata },
			expires_at: Math.floor(Date.now() / 1000) + APPOINTMENT_CONFIG.stripeSessionMinutes * 60,
			success_url: `${envVars.FRONTEND_URL}/portal/appointments?payment=success&appointment=${input.appointmentId}`,
			cancel_url: `${envVars.FRONTEND_URL}/portal/appointments?payment=cancelled&appointment=${input.appointmentId}`,
		},
		// একই payment-এর জন্য retry হলে Stripe নতুন session না বানিয়ে আগেরটাই ফেরত দেয়
		{ idempotencyKey: `checkout-${input.paymentId}` },
	);

	if (!session.url) {
		throw new AppError(httpStatus.BAD_GATEWAY, "Stripe did not return a checkout URL");
	}

	return { sessionId: session.id, url: session.url };
};

// Hold শেষ বা desk-এ টাকা নেওয়া হলে পুরনো link বন্ধ। আগেই complete/expire হয়ে থাকলে Stripe error দেয় — সেটা স্বাভাবিক।
export const expireCheckoutSession = async (sessionId: string) => {
	try {
		await stripe.checkout.sessions.expire(sessionId);
	} catch (error) {
		console.warn(
			`[stripe] session ${sessionId} was not expired (already completed or expired?)`,
			error instanceof Error ? error.message : error,
		);
	}
};

// idempotencyKey: Stripe বা আমাদের retry হলেও একই payment দুবার refund হয় না
export const refundStripePayment = (paymentIntentId: string, paymentId: string) =>
	stripe.refunds.create({ payment_intent: paymentIntentId }, { idempotencyKey: `refund-${paymentId}` });

export const getPaymentIntentId = (session: Stripe.Checkout.Session): string | null => {
	if (!session.payment_intent) return null;
	return typeof session.payment_intent === "string" ? session.payment_intent : session.payment_intent.id;
};
```


**`src/app/module/payment/payment.service.ts`**

```ts
import { prisma } from "@/app/lib/prisma";
import type { Prisma } from "@/generated/prisma/client";
import { AppointmentStatus, PaymentStatus, SlotStatus } from "@/generated/prisma/enums";
import type { Stripe } from "stripe";
import { appointmentSelect } from "../appointment/appointment.constant";
import { emailAppointmentCancelled, emailAppointmentConfirmed, sendSafely } from "../appointment/appointment.email";
import { activeAppointmentWhere, expireStaleHolds } from "../appointment/appointment.helpers";
import { expireCheckoutSession, getPaymentIntentId, refundStripePayment } from "./payment.stripe";

type TWebhookResult = { message: string };

// Hold শেষে EXPIRED হওয়া appointment-এ দেরিতে টাকা এলে: slot তখনও খালি থাকলে আবার CONFIRMED করা যায়
const canReviveExpired = async (tx: Prisma.TransactionClient, slotId: string, now: Date) => {
	await tx.$queryRaw`SELECT id FROM "doctor_slots" WHERE id = ${slotId} FOR UPDATE`;

	const slot = await tx.doctorSlot.findUnique({ where: { id: slotId }, select: { status: true, startAt: true } });
	if (!slot || slot.status !== SlotStatus.OPEN || slot.startAt <= now) return false;

	await expireStaleHolds(tx, { slotId }, now);
	const taken = await tx.appointment.findFirst({
		where: { slotId, ...activeAppointmentWhere(now) },
		select: { id: true },
	});
	return !taken;
};

type TPaidOutcome =
	| { kind: "ignored" }
	| { kind: "duplicate" }
	| { kind: "confirmed"; appointmentId: string }
	| { kind: "refund"; appointmentId: string; paymentId: string; wasExpired: boolean };

// checkout.session.completed: টাকা কেটেছে → payment SUCCEEDED + appointment CONFIRMED, নাহলে refund
const handleCheckoutPaid = async (session: Stripe.Checkout.Session, eventId: string): Promise<TWebhookResult> => {
	// Card ছাড়া ধীর পদ্ধতিতে (bank transfer) completed আসে কিন্তু টাকা পরে আসে
	if (session.payment_status !== "paid") return { message: `Session ${session.id} is not paid yet` };

	const paymentId = session.metadata?.paymentId;
	if (!paymentId) return { message: `Session ${session.id} has no paymentId in metadata` };

	const paymentIntentId = getPaymentIntentId(session);
	const now = new Date();

	const outcome = await prisma.$transaction(async (tx): Promise<TPaidOutcome> => {
		const payment = await tx.payment.findUnique({ where: { id: paymentId }, select: { appointmentId: true } });
		if (!payment) return { kind: "ignored" };

		// Appointment row lock → desk payment, cancel, cron এই মুহূর্তে একই appointment ছুঁতে পারবে না
		await tx.$queryRaw`SELECT id FROM "appointments" WHERE id = ${payment.appointmentId} FOR UPDATE`;

		const current = await tx.payment.findUniqueOrThrow({ where: { id: paymentId }, select: { status: true } });
		// Idempotency: Stripe একই event আবার পাঠালে (retry) কিছুই বদলায় না
		if (current.status === PaymentStatus.SUCCEEDED || current.status === PaymentStatus.REFUNDED) {
			return { kind: "duplicate" };
		}

		const appointment = await tx.appointment.findUniqueOrThrow({
			where: { id: payment.appointmentId },
			select: {
				id: true,
				status: true,
				slotId: true,
				payments: { where: { status: PaymentStatus.SUCCEEDED }, select: { id: true } },
			},
		});

		const alreadyPaid = appointment.payments.length > 0; // যেমন: desk-এ আগেই cash নেওয়া হয়েছে
		let confirm = false;
		if (!alreadyPaid) {
			if (
				appointment.status === AppointmentStatus.PENDING_PAYMENT ||
				appointment.status === AppointmentStatus.CONFIRMED
			) {
				confirm = true;
			} else if (appointment.status === AppointmentStatus.EXPIRED) {
				confirm = await canReviveExpired(tx, appointment.slotId, now);
			}
		}

		if (confirm) {
			await tx.payment.update({
				where: { id: paymentId },
				data: {
					status: PaymentStatus.SUCCEEDED,
					stripeEventId: eventId,
					stripePaymentIntentId: paymentIntentId,
					paidAt: now,
				},
			});
			await tx.appointment.update({
				where: { id: appointment.id },
				data: { status: AppointmentStatus.CONFIRMED },
			});
			return { kind: "confirmed", appointmentId: appointment.id };
		}

		// টাকা নেওয়া যাবে না (cancelled / অন্য কেউ slot নিয়েছে / আগেই paid) → refund-এর জন্য তথ্য রেখে দেওয়া
		await tx.payment.update({
			where: { id: paymentId },
			data: { stripeEventId: eventId, stripePaymentIntentId: paymentIntentId },
		});
		return {
			kind: "refund",
			appointmentId: appointment.id,
			paymentId,
			wasExpired: appointment.status === AppointmentStatus.EXPIRED,
		};
	});

	if (outcome.kind === "ignored") return { message: `Payment ${paymentId} not found` };
	if (outcome.kind === "duplicate") return { message: `Event for payment ${paymentId} already processed` };

	const appointment = await prisma.appointment.findUniqueOrThrow({
		where: { id: outcome.appointmentId },
		select: appointmentSelect,
	});

	if (outcome.kind === "confirmed") {
		await sendSafely(`confirm ${appointment.appointmentNumber}`, () =>
			emailAppointmentConfirmed(appointment, "Paid online"),
		);
		return { message: `Appointment ${appointment.appointmentNumber} confirmed` };
	}

	// Refund — Stripe call transaction-এর বাইরে; fail করলে error → 500 → Stripe নিজেই webhook আবার পাঠাবে
	if (!paymentIntentId) throw new Error(`Session ${session.id} has no payment intent to refund`);
	await refundStripePayment(paymentIntentId, outcome.paymentId);
	await prisma.payment.update({
		where: { id: outcome.paymentId },
		data: { status: PaymentStatus.REFUNDED, refundedAt: new Date() },
	});

	if (outcome.wasExpired) {
		await sendSafely(`late payment ${appointment.appointmentNumber}`, () =>
			emailAppointmentCancelled(
				appointment,
				"Your payment arrived after the 30-minute hold ended and the time was no longer available.",
				"Your payment has been refunded. It can take 5–10 business days to show on your card.",
			),
		);
	}

	return { message: `Payment ${outcome.paymentId} refunded` };
};

// Session expire / async payment fail: শুধু এই payment FAILED। Appointment-এর hold সামলায় cron।
const handleCheckoutFailed = async (session: Stripe.Checkout.Session): Promise<TWebhookResult> => {
	const paymentId = session.metadata?.paymentId;
	if (!paymentId) return { message: `Session ${session.id} has no paymentId in metadata` };

	await prisma.payment.updateMany({
		where: { id: paymentId, status: PaymentStatus.PENDING },
		data: { status: PaymentStatus.FAILED },
	});
	return { message: `Payment ${paymentId} marked failed` };
};

const handleStripeEvent = async (event: Stripe.Event): Promise<TWebhookResult> => {
	switch (event.type) {
		case "checkout.session.completed":
		case "checkout.session.async_payment_succeeded":
			return handleCheckoutPaid(event.data.object, event.id);

		case "checkout.session.expired":
		case "checkout.session.async_payment_failed":
			return handleCheckoutFailed(event.data.object);

		default:
			return { message: `Ignored event ${event.type}` };
	}
};

// Cron: hold শেষ হওয়া booking → EXPIRED (slot আবার খালি), খোলা Stripe link বন্ধ
const expireUnpaidHolds = async () => {
	const now = new Date();
	const { expiredCount, sessionIds } = await prisma.$transaction((tx) => expireStaleHolds(tx, {}, now));
	await Promise.all(sessionIds.map(expireCheckoutSession));
	return expiredCount;
};

export const PaymentService = {
	handleStripeEvent,
	expireUnpaidHolds,
};
```


**`src/app/module/payment/payment.controller.ts`**

```ts
import { envVars } from "@/app/config/env";
import { stripe } from "@/app/config/stripe.config";
import { AppError } from "@/app/errorHelpers/AppError";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import type { Stripe } from "stripe";
import { PaymentService } from "./payment.service";

const handleStripeWebhook = catchAsync(async (req, res) => {
	const signature = req.headers["stripe-signature"];
	if (typeof signature !== "string") {
		throw new AppError(httpStatus.BAD_REQUEST, "Missing Stripe signature");
	}

	let event: Stripe.Event;
	try {
		// req.body এখানে Buffer (express.raw) — JSON parse হয়ে গেলে signature মিলবে না
		event = stripe.webhooks.constructEvent(req.body, signature, envVars.STRIPE_WEBHOOK_SECRET);
	} catch (error) {
		throw new AppError(httpStatus.BAD_REQUEST, "Invalid Stripe signature", { cause: error });
	}

	// এখানে error হলে globalErrorHandler 500 দেয় → Stripe পরে আবার পাঠায়
	const result = await PaymentService.handleStripeEvent(event);

	sendResponse(res, { statusCode: httpStatus.OK, message: result.message, data: null });
});

export const PaymentController = {
	handleStripeWebhook,
};
```


**`src/app/module/payment/payment.job.ts`**

```ts
import { schedule, type ScheduledTask } from "node-cron";
import { PaymentService } from "./payment.service";

// প্রতি মিনিটে: hold শেষ হওয়া PENDING_PAYMENT → EXPIRED
export const startPaymentJobs = (): ScheduledTask[] => [
	schedule(
		"* * * * *",
		async () => {
			try {
				const expired = await PaymentService.expireUnpaidHolds();
				if (expired > 0) console.info(`[cron] expired ${expired} unpaid booking hold(s)`);
			} catch (error) {
				console.error("[cron] expire unpaid holds failed:", error instanceof Error ? error.message : error);
			}
		},
		{ name: "expire-unpaid-holds", noOverlap: true }, // আগের run শেষ না হলে নতুন run শুরু হয় না
	),
];
```

## ৭. Appointment module (`src/app/module/appointment/`)


**`src/app/module/appointment/appointment.constant.ts`**

```ts
import { defineListConfig } from "@/app/shared/query/queryBuilder";
import type { Prisma } from "@/generated/prisma/client";
import {
	AppointmentPaymentMode,
	AppointmentSource,
	AppointmentStatus,
	PaymentStatus,
} from "@/generated/prisma/enums";

export const appointmentSelect = {
	id: true,
	appointmentNumber: true,
	startAt: true,
	endAt: true,
	status: true,
	source: true,
	paymentMode: true,
	feeCents: true,
	holdExpiresAt: true,
	reason: true,
	cancelledAt: true,
	cancelReason: true,
	createdAt: true,
	patient: { select: { id: true, fullName: true, phone: true, user: { select: { email: true } } } },
	doctor: { select: { id: true, roomNumber: true, user: { select: { name: true } } } },
	payments: {
		select: {
			id: true,
			provider: true,
			status: true,
			amountCents: true,
			currency: true,
			deskMethod: true,
			paidAt: true,
			refundedAt: true,
			createdAt: true,
		},
		orderBy: { createdAt: "asc" },
	},
} satisfies Prisma.AppointmentSelect;

export type TAppointmentRow = Prisma.AppointmentGetPayload<{ select: typeof appointmentSelect }>;

export const toAppointmentResponse = (appointment: TAppointmentRow) => ({
	...appointment,
	isPaid: appointment.payments.some((payment) => payment.status === PaymentStatus.SUCCEEDED),
});

export const appointmentListConfig = defineListConfig<
	Prisma.AppointmentWhereInput,
	Prisma.AppointmentOrderByWithRelationInput
>({
	searchableFields: ["appointmentNumber", "patient.fullName", "patient.phone", "patient.user.email"],
	filters: {
		status: { path: "status", type: "enum", op: "in", values: Object.values(AppointmentStatus) },
		source: { path: "source", type: "enum", values: Object.values(AppointmentSource) },
		paymentMode: { path: "paymentMode", type: "enum", values: Object.values(AppointmentPaymentMode) },
		doctorId: { path: "doctorId", type: "uuid" },
		patientId: { path: "patientId", type: "uuid" },
	},
	sortableFields: ["startAt", "createdAt", "appointmentNumber"],
	defaultSortBy: "startAt",
	defaultSortOrder: "asc",
});
```


**`src/app/module/appointment/appointment.validation.ts`**

```ts
import { AppointmentPaymentMode, PaymentMethod } from "@/generated/prisma/enums";
import * as z from "zod";

const bookAppointmentSchema = z
	.object({
		slotId: z.uuid("Invalid slot id"),
		paymentMode: z.enum(AppointmentPaymentMode, "paymentMode must be AT_DESK or ONLINE"),
		reason: z.string().trim().min(2).max(500).optional(),
		// নিচের দুটো শুধু front desk (ADMINS) পাঠাতে পারে — service-এ যাচাই
		patientId: z.uuid("Invalid patient id").optional(),
		deskPaymentMethod: z.enum(PaymentMethod, "deskPaymentMethod must be CASH, CARD or MOBILE").optional(),
	})
	.refine((data) => !data.deskPaymentMethod || data.paymentMode === AppointmentPaymentMode.AT_DESK, {
		error: "deskPaymentMethod works only with paymentMode AT_DESK",
		path: ["deskPaymentMethod"],
	});

const cancelAppointmentSchema = z.object({
	reason: z.string().trim().min(3, "Reason must be at least 3 characters").max(200),
});

// রোগীর cancel-এ কারণ ঐচ্ছিক; body না পাঠালেও চলবে
const cancelMyAppointmentSchema = z
	.object({ reason: z.string().trim().min(3).max(200).optional() })
	.default({});

const deskPaymentSchema = z.object({
	method: z.enum(PaymentMethod, "method must be CASH, CARD or MOBILE"),
});

const listQuerySchema = z.object({
	date: z.iso.date("date must be in YYYY-MM-DD format").optional(),
});

export const AppointmentValidation = {
	bookAppointmentSchema,
	cancelAppointmentSchema,
	cancelMyAppointmentSchema,
	deskPaymentSchema,
	listQuerySchema,
};
```


**`src/app/module/appointment/appointment.interface.ts`**

```ts
import type * as z from "zod";
import type { AppointmentValidation } from "./appointment.validation";

export type TBookAppointmentPayload = z.infer<typeof AppointmentValidation.bookAppointmentSchema>;
export type TCancelAppointmentPayload = z.infer<typeof AppointmentValidation.cancelAppointmentSchema>;
export type TCancelMyAppointmentPayload = z.infer<typeof AppointmentValidation.cancelMyAppointmentSchema>;
export type TDeskPaymentPayload = z.infer<typeof AppointmentValidation.deskPaymentSchema>;
```


**`src/app/module/appointment/appointment.email.ts`**

```ts
import { envVars } from "@/app/config/env";
import { sendEmail } from "@/app/shared/sendEmail";
import { formatMoney } from "@/app/utils/money";
import { formatClinicTime } from "../slot/slot.utils";

const TZ = envVars.CLINIC_TIME_ZONE;

// Email-এর জন্য appointment-এর যেটুকু লাগে
export type TEmailAppointment = {
	appointmentNumber: string;
	startAt: Date;
	feeCents: number;
	patient: { fullName: string; user: { email: string } };
	doctor: { user: { name: string } };
};

// Email fail করলে booking/cancel fail হওয়া উচিত না — শুধু log
export const sendSafely = async (label: string, task: () => Promise<void>) => {
	try {
		await task();
	} catch (error) {
		console.error(`[email] ${label} failed:`, error instanceof Error ? error.message : error);
	}
};

const common = (appointment: TEmailAppointment) => ({
	name: appointment.patient.fullName,
	appointmentNumber: appointment.appointmentNumber,
	doctorName: appointment.doctor.user.name,
	whenLabel: formatClinicTime(appointment.startAt, TZ),
});

export const emailAppointmentConfirmed = (appointment: TEmailAppointment, paymentLabel: string) =>
	sendEmail({
		to: appointment.patient.user.email,
		subject: `Appointment confirmed · ${appointment.appointmentNumber}`,
		previewText: `Dr. ${appointment.doctor.user.name} · ${formatClinicTime(appointment.startAt, TZ)}`,
		template: "appointmentConfirmed",
		data: { ...common(appointment), paymentLabel, portalUrl: `${envVars.FRONTEND_URL}/portal/appointments` },
	});

export const emailAppointmentCancelled = (appointment: TEmailAppointment, reason: string, refundNote: string | null) =>
	sendEmail({
		to: appointment.patient.user.email,
		subject: `Appointment cancelled · ${appointment.appointmentNumber}`,
		previewText: reason,
		template: "appointmentCancelled",
		data: { ...common(appointment), reason, refundNote, bookUrl: `${envVars.FRONTEND_URL}/doctors` },
	});

export const emailPaymentLink = (appointment: TEmailAppointment, payUrl: string, currency: string, expiresAt: Date) =>
	sendEmail({
		to: appointment.patient.user.email,
		subject: `Pay to confirm ${appointment.appointmentNumber}`,
		previewText: `Consultation fee ${formatMoney(appointment.feeCents, currency)}`,
		template: "paymentLink",
		data: {
			...common(appointment),
			amountLabel: formatMoney(appointment.feeCents, currency),
			payUrl,
			expiresLabel: formatClinicTime(expiresAt, TZ),
		},
	});
```


**`src/app/module/appointment/appointment.helpers.ts`**

```ts
import { envVars } from "@/app/config/env";
import { prisma } from "@/app/lib/prisma";
import { nextCounter } from "@/app/utils/counter";
import type { Prisma } from "@/generated/prisma/client";
import { AppointmentStatus, PaymentProvider, PaymentStatus } from "@/generated/prisma/enums";
import { expireCheckoutSession, refundStripePayment } from "../payment/payment.stripe";
import { addDaysToDate, clinicDateOf, clinicDayStartUtc } from "../slot/slot.utils";
import { emailAppointmentCancelled, sendSafely } from "./appointment.email";

const TZ = envVars.CLINIC_TIME_ZONE;

// Slot দখলে রাখে এমন appointment: CONFIRMED/CHECKED_IN, অথবা hold এখনো চলছে এমন PENDING_PAYMENT।
// "Booked" কোনো আলাদা flag না — এই শর্তটাই (SDLC 3.8)।
export const activeAppointmentWhere = (now: Date): Prisma.AppointmentWhereInput => ({
	OR: [
		{ status: { in: [AppointmentStatus.CONFIRMED, AppointmentStatus.CHECKED_IN] } },
		{ status: AppointmentStatus.PENDING_PAYMENT, holdExpiresAt: { gt: now } },
	],
});

export const CANCELLABLE_STATUSES: AppointmentStatus[] = [AppointmentStatus.PENDING_PAYMENT, AppointmentStatus.CONFIRMED];

// Clinic-এর একটা দিন ("2026-11-20") → UTC range
export const clinicDayRange = (date: string) => ({
	gte: clinicDayStartUtc(date, TZ),
	lt: clinicDayStartUtc(addDaysToDate(date, 1), TZ),
});

// APT-2026-000123 — বছর অনুযায়ী আলাদা counter
export const nextAppointmentNumber = async (tx: Prisma.TransactionClient, now: Date) => {
	const year = clinicDateOf(now, TZ).slice(0, 4);
	const value = await nextCounter(tx, `appointment:${year}`);
	return `APT-${year}-${String(value).padStart(6, "0")}`;
};

// Hold শেষ হওয়া PENDING_PAYMENT → EXPIRED, তার PENDING payment → FAILED।
// ফেরত: কতগুলো expire হলো + কোন Stripe session-গুলো commit-এর পরে বন্ধ করতে হবে।
export const expireStaleHolds = async (tx: Prisma.TransactionClient, where: Prisma.AppointmentWhereInput, now: Date) => {
	const expired = await tx.appointment.updateManyAndReturn({
		where: { AND: [where, { status: AppointmentStatus.PENDING_PAYMENT, holdExpiresAt: { lte: now } }] },
		data: { status: AppointmentStatus.EXPIRED },
		select: { id: true },
	});
	if (expired.length === 0) return { expiredCount: 0, sessionIds: [] };

	const ids = expired.map((appointment) => appointment.id);
	const pending = await tx.payment.findMany({
		where: { appointmentId: { in: ids }, status: PaymentStatus.PENDING },
		select: { stripeSessionId: true },
	});
	await tx.payment.updateMany({
		where: { appointmentId: { in: ids }, status: PaymentStatus.PENDING },
		data: { status: PaymentStatus.FAILED },
	});

	return {
		expiredCount: ids.length,
		sessionIds: pending.flatMap((payment) => (payment.stripeSessionId ? [payment.stripeSessionId] : [])),
	};
};

const cancellationSelect = {
	id: true,
	appointmentNumber: true,
	startAt: true,
	feeCents: true,
	patient: { select: { fullName: true, user: { select: { email: true } } } },
	doctor: { select: { user: { select: { name: true } } } },
	payments: {
		where: { status: { in: [PaymentStatus.PENDING, PaymentStatus.SUCCEEDED] } },
		select: { id: true, provider: true, status: true, stripeSessionId: true, stripePaymentIntentId: true },
	},
} satisfies Prisma.AppointmentSelect;

export type TCancelledAppointment = Prisma.AppointmentGetPayload<{ select: typeof cancellationSelect }>;

type TCancelContext = { reason: string; actorId: string; now: Date };

// Transaction-এর ভেতরে: active appointment → CANCELLED, payment-এর DB অবস্থা ঠিক করা।
// Staff cancel, রোগীর cancel আর slot block — তিন জায়গা থেকেই এটা ব্যবহার হয়।
export const cancelActiveAppointments = async (
	tx: Prisma.TransactionClient,
	where: Prisma.AppointmentWhereInput,
	{ reason, actorId, now }: TCancelContext,
): Promise<TCancelledAppointment[]> => {
	// শর্ত update-এর ভেতরেই → এই মুহূর্তে অন্য কেউ status বদলালে সেটা আর ছোঁয়া হয় না
	const updated = await tx.appointment.updateManyAndReturn({
		where: { AND: [where, { status: { in: CANCELLABLE_STATUSES } }] },
		data: { status: AppointmentStatus.CANCELLED, cancelledAt: now, cancelReason: reason, cancelledById: actorId },
		select: { id: true },
	});
	if (updated.length === 0) return [];

	const ids = updated.map((appointment) => appointment.id);

	// payment-এর অবস্থা বদলানোর আগেই পড়ে রাখা (commit-এর পরে refund/expire-এর জন্য লাগবে)
	const cancelled = await tx.appointment.findMany({ where: { id: { in: ids } }, select: cancellationSelect });

	await tx.payment.updateMany({
		where: { appointmentId: { in: ids }, status: PaymentStatus.PENDING },
		data: { status: PaymentStatus.FAILED },
	});
	// Desk-এ নেওয়া টাকা desk থেকেই ফেরত দেওয়া হয়
	await tx.payment.updateMany({
		where: { appointmentId: { in: ids }, provider: PaymentProvider.DESK, status: PaymentStatus.SUCCEEDED },
		data: { status: PaymentStatus.REFUNDED, refundedAt: now },
	});

	return cancelled;
};

// Commit-এর পরে: Stripe refund / খোলা session বন্ধ → রোগীকে জানানো কী হলো
const settlePayments = async (appointment: TCancelledAppointment): Promise<string | null> => {
	let refundNote: string | null = null;

	for (const payment of appointment.payments) {
		if (payment.provider === PaymentProvider.DESK && payment.status === PaymentStatus.SUCCEEDED) {
			refundNote = "Please collect your consultation fee from the front desk.";
			continue;
		}

		if (payment.status === PaymentStatus.PENDING && payment.stripeSessionId) {
			await expireCheckoutSession(payment.stripeSessionId);
			continue;
		}

		if (payment.status === PaymentStatus.SUCCEEDED && payment.stripePaymentIntentId) {
			try {
				await refundStripePayment(payment.stripePaymentIntentId, payment.id);
				await prisma.payment.update({
					where: { id: payment.id },
					data: { status: PaymentStatus.REFUNDED, refundedAt: new Date() },
				});
				refundNote = "Your online payment has been refunded. It can take 5–10 business days to show on your card.";
			} catch (error) {
				console.error(`[refund] payment ${payment.id} failed — refund it from the Stripe dashboard`, error);
				refundNote = "Your refund is being processed. The front desk will contact you if anything else is needed.";
			}
		}
	}

	return refundNote;
};

export const finalizeCancellations = async (appointments: TCancelledAppointment[], reason: string) => {
	for (const appointment of appointments) {
		const refundNote = await settlePayments(appointment);
		await sendSafely(`cancel ${appointment.appointmentNumber}`, () =>
			emailAppointmentCancelled(appointment, reason, refundNote),
		);
	}
};
```


**`src/app/module/appointment/appointment.service.ts`**

```ts
import { APPOINTMENT_CONFIG } from "@/app/config/appointment.config";
import { envVars } from "@/app/config/env";
import { SLOT_CONFIG } from "@/app/config/slot.config";
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { buildListQuery, buildMeta } from "@/app/shared/query/queryBuilder";
import type { TAuthPayload } from "@/app/utils/token";
import { Prisma } from "@/generated/prisma/client";
import {
	AppointmentPaymentMode,
	AppointmentSource,
	AppointmentStatus,
	PaymentProvider,
	PaymentStatus,
	Role,
	SlotStatus,
} from "@/generated/prisma/enums";
import httpStatus from "http-status";
import { createCheckoutSession, expireCheckoutSession } from "../payment/payment.stripe";
import { isProfileComplete } from "../patient/patient.service";
import { addDaysToDate, clinicDateOf } from "../slot/slot.utils";
import {
	appointmentListConfig,
	appointmentSelect,
	type TAppointmentRow,
	toAppointmentResponse,
} from "./appointment.constant";
import { emailAppointmentConfirmed, emailPaymentLink, sendSafely } from "./appointment.email";
import {
	activeAppointmentWhere,
	CANCELLABLE_STATUSES,
	cancelActiveAppointments,
	clinicDayRange,
	expireStaleHolds,
	finalizeCancellations,
	nextAppointmentNumber,
} from "./appointment.helpers";
import type {
	TBookAppointmentPayload,
	TCancelAppointmentPayload,
	TCancelMyAppointmentPayload,
	TDeskPaymentPayload,
} from "./appointment.interface";

const TZ = envVars.CLINIC_TIME_ZONE;
const MINUTE_MS = 60_000;
const { currency } = APPOINTMENT_CONFIG;

// ---------------- ছোট helper ----------------

const findOwnDoctorId = async (userId: string) => {
	const doctor = await prisma.doctorProfile.findUnique({ where: { userId }, select: { id: true } });
	if (!doctor) throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
	return doctor.id;
};

// IDOR-safe: রোগীর id সবসময় session থেকে
const findMyPatientId = async (userId: string) => {
	const patient = await prisma.patient.findFirst({
		where: { userId, user: { isDeleted: false } },
		select: { id: true },
	});
	if (!patient) throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
	return patient.id;
};

const findAppointmentOrThrow = async (where: Prisma.AppointmentWhereInput): Promise<TAppointmentRow> => {
	const appointment = await prisma.appointment.findFirst({ where, select: appointmentSelect });
	if (!appointment) throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");
	return appointment;
};

// Row lock: এই appointment নিয়ে একসাথে দুটো কাজ (desk payment + webhook ইত্যাদি) চলবে না
const lockAppointment = async (tx: Prisma.TransactionClient, id: string) => {
	await tx.$queryRaw`SELECT id FROM "appointments" WHERE id = ${id} FOR UPDATE`;
};

const assertCancellable = (status: AppointmentStatus) => {
	if (!CANCELLABLE_STATUSES.includes(status)) {
		throw new AppError(httpStatus.CONFLICT, `A ${status} appointment can't be cancelled`);
	}
};

// Partial unique index (doctorId, startAt)-এ ধাক্কা লাগলে Prisma P2002 → সুন্দর 409
const rethrowBookingConflict = (error: unknown): never => {
	if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002") {
		throw new AppError(httpStatus.CONFLICT, "This time was just booked — please pick another slot", { cause: error });
	}
	throw error;
};

// ---------------- Booking ----------------

// PATIENT → নিজের id (session), ADMINS → body-র patientId
const resolveBookingPatient = async (authUser: TAuthPayload, payload: TBookAppointmentPayload) => {
	if (authUser.role === Role.PATIENT) {
		if (payload.deskPaymentMethod) {
			throw new AppError(httpStatus.FORBIDDEN, "Only the front desk can record a desk payment");
		}

		const patient = await prisma.patient.findFirst({
			where: { userId: authUser.userId, user: { isDeleted: false } },
			select: { id: true, phone: true, dateOfBirth: true, gender: true },
		});
		if (!patient) throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
		if (!isProfileComplete(patient)) {
			throw new AppError(httpStatus.BAD_REQUEST, "Complete your profile first");
		}

		return { patientId: patient.id, source: AppointmentSource.ONLINE };
	}

	if (!payload.patientId) {
		throw new AppError(httpStatus.BAD_REQUEST, "patientId is required when booking for a patient");
	}

	const patient = await prisma.patient.findFirst({
		where: { id: payload.patientId, user: { isDeleted: false, isActive: true } },
		select: { id: true },
	});
	if (!patient) throw new AppError(httpStatus.NOT_FOUND, "Patient not found");

	return { patientId: patient.id, source: AppointmentSource.STAFF };
};

// Slot row lock (FOR UPDATE) → অন্য booking বা block এই slot-এ অপেক্ষা করবে
const lockSlot = async (tx: Prisma.TransactionClient, slotId: string) => {
	const locked = await tx.$queryRaw<{ id: string }[]>`SELECT id FROM "doctor_slots" WHERE id = ${slotId} FOR UPDATE`;
	if (locked.length === 0) throw new AppError(httpStatus.NOT_FOUND, "Slot not found");

	return tx.doctorSlot.findUniqueOrThrow({
		where: { id: slotId },
		select: {
			id: true,
			doctorId: true,
			startAt: true,
			endAt: true,
			status: true,
			blockReason: true,
			doctor: {
				select: {
					consultationFeeCents: true,
					isPaused: true,
					user: { select: { isActive: true, isDeleted: true } },
				},
			},
		},
	});
};

type TLockedSlot = Awaited<ReturnType<typeof lockSlot>>;

// কে book করছে তার উপর নিয়ম আলাদা (SDLC 3.10)
const assertSlotBookable = (slot: TLockedSlot, isStaff: boolean, now: Date) => {
	if (!slot.doctor.user.isActive || slot.doctor.user.isDeleted) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	}
	if (slot.status === SlotStatus.BLOCKED) {
		throw new AppError(httpStatus.CONFLICT, `This time is not available: ${slot.blockReason ?? "blocked"}`);
	}

	const today = clinicDateOf(now, TZ);

	if (isStaff) {
		// Front desk: রোগী সামনে দাঁড়িয়ে — এখন চলছে এমন slot-ও চলবে
		if (slot.endAt <= now) throw new AppError(httpStatus.BAD_REQUEST, "This slot has already ended");
	} else {
		const earliest = new Date(now.getTime() + SLOT_CONFIG.minLeadMinutes * MINUTE_MS);
		if (slot.startAt < earliest) {
			throw new AppError(
				httpStatus.BAD_REQUEST,
				`Online bookings must be at least ${SLOT_CONFIG.minLeadMinutes} minutes ahead`,
			);
		}
		const lastBookableDate = addDaysToDate(today, SLOT_CONFIG.bookingWindowDays - 1);
		if (clinicDateOf(slot.startAt, TZ) > lastBookableDate) {
			throw new AppError(httpStatus.BAD_REQUEST, `Booking is open until ${lastBookableDate}`);
		}
	}

	if (slot.doctor.isPaused && clinicDateOf(slot.startAt, TZ) === today) {
		throw new AppError(httpStatus.CONFLICT, "This doctor is not taking new bookings today");
	}
};

// Stripe session খোলা (transaction-এর বাইরে)। Fail করলে payment FAILED করে 502।
const openCheckout = async (appointment: TAppointmentRow, paymentId: string) => {
	try {
		const session = await createCheckoutSession({
			appointmentId: appointment.id,
			paymentId,
			appointmentNumber: appointment.appointmentNumber,
			doctorName: appointment.doctor.user.name,
			amountCents: appointment.feeCents,
			currency,
			customerEmail: appointment.patient.user.email,
		});
		await prisma.payment.update({ where: { id: paymentId }, data: { stripeSessionId: session.sessionId } });
		return session.url;
	} catch (error) {
		await prisma.payment.updateMany({
			where: { id: paymentId, status: PaymentStatus.PENDING },
			data: { status: PaymentStatus.FAILED },
		});
		throw new AppError(httpStatus.BAD_GATEWAY, "Online payment is unavailable right now — please try again", {
			cause: error,
		});
	}
};

const bookAppointment = async (authUser: TAuthPayload, payload: TBookAppointmentPayload) => {
	const { patientId, source } = await resolveBookingPatient(authUser, payload);
	const isStaff = source === AppointmentSource.STAFF;
	const isOnline = payload.paymentMode === AppointmentPaymentMode.ONLINE;
	const now = new Date();

	const created = await prisma
		.$transaction(async (tx) => {
			const slot = await lockSlot(tx, payload.slotId);
			assertSlotBookable(slot, isStaff, now);

			const feeCents = slot.doctor.consultationFeeCents;
			if (isOnline && feeCents < APPOINTMENT_CONFIG.minOnlineAmountCents) {
				throw new AppError(httpStatus.BAD_REQUEST, "This fee is too small for online payment — pay at the front desk");
			}

			// এই slot-এ hold শেষ হওয়া পুরনো booking থাকলে আগে EXPIRED (নাহলে unique index-এ আটকে যেত)
			const stale = await expireStaleHolds(tx, { slotId: slot.id }, now);

			const taken = await tx.appointment.findFirst({
				where: { slotId: slot.id, ...activeAppointmentWhere(now) },
				select: { id: true },
			});
			if (taken) throw new AppError(httpStatus.CONFLICT, "This time was just booked — please pick another slot");

			// একই রোগী, একই doctor, একই দিনে একটার বেশি active appointment না (FR-11)
			const sameDay = await tx.appointment.findFirst({
				where: {
					patientId,
					doctorId: slot.doctorId,
					startAt: clinicDayRange(clinicDateOf(slot.startAt, TZ)),
					...activeAppointmentWhere(now),
				},
				select: { appointmentNumber: true },
			});
			if (sameDay) {
				throw new AppError(
					httpStatus.CONFLICT,
					`This patient already has ${sameDay.appointmentNumber} with this doctor on the same day`,
				);
			}

			const appointment = await tx.appointment.create({
				data: {
					appointmentNumber: await nextAppointmentNumber(tx, now),
					patientId,
					doctorId: slot.doctorId,
					slotId: slot.id,
					startAt: slot.startAt,
					endAt: slot.endAt,
					status: isOnline ? AppointmentStatus.PENDING_PAYMENT : AppointmentStatus.CONFIRMED,
					source,
					paymentMode: payload.paymentMode,
					feeCents, // booking-এর সময়ের fee-র snapshot
					holdExpiresAt: isOnline ? new Date(now.getTime() + APPOINTMENT_CONFIG.holdMinutes * MINUTE_MS) : null,
					reason: payload.reason ?? null,
					bookedById: authUser.userId,
				},
				select: { id: true },
			});

			let paymentId: string | null = null;

			if (isOnline) {
				const payment = await tx.payment.create({
					data: { appointmentId: appointment.id, provider: PaymentProvider.STRIPE, amountCents: feeCents, currency },
					select: { id: true },
				});
				paymentId = payment.id;
			} else if (payload.deskPaymentMethod) {
				// Front desk-এ এখনই টাকা নেওয়া হলো
				await tx.payment.create({
					data: {
						appointmentId: appointment.id,
						provider: PaymentProvider.DESK,
						status: PaymentStatus.SUCCEEDED,
						deskMethod: payload.deskPaymentMethod,
						receivedById: authUser.userId,
						amountCents: feeCents,
						currency,
						paidAt: now,
					},
				});
			}

			return { appointmentId: appointment.id, paymentId, staleSessionIds: stale.sessionIds };
		})
		.catch(rethrowBookingConflict);

	await Promise.all(created.staleSessionIds.map(expireCheckoutSession));

	const appointment = await findAppointmentOrThrow({ id: created.appointmentId });

	if (!created.paymentId) {
		const paymentLabel = payload.deskPaymentMethod
			? "Paid at the front desk"
			: "Pay at the front desk before your visit";
		await sendSafely(`confirm ${appointment.appointmentNumber}`, () =>
			emailAppointmentConfirmed(appointment, paymentLabel),
		);
		return { appointment: toAppointmentResponse(appointment), checkoutUrl: null };
	}

	let checkoutUrl: string;
	try {
		checkoutUrl = await openCheckout(appointment, created.paymentId);
	} catch (error) {
		// Stripe-এ যাওয়াই গেল না → slot ৩০ মিনিট আটকে না রেখে এখনই ছেড়ে দাও
		await prisma.appointment.updateMany({
			where: { id: appointment.id, status: AppointmentStatus.PENDING_PAYMENT },
			data: { status: AppointmentStatus.EXPIRED },
		});
		throw error;
	}

	if (isStaff && appointment.holdExpiresAt) {
		const expiresAt = appointment.holdExpiresAt;
		await sendSafely(`payment link ${appointment.appointmentNumber}`, () =>
			emailPaymentLink(appointment, checkoutUrl, currency, expiresAt),
		);
	}

	return { appointment: toAppointmentResponse(appointment), checkoutUrl };
};

// ---------------- দেখা ----------------

const getAppointments = async (authUser: TAuthPayload, rawQuery: unknown, date?: string) => {
	const conditions: Prisma.AppointmentWhereInput[] = [];
	if (authUser.role === Role.DOCTOR) conditions.push({ doctorId: await findOwnDoctorId(authUser.userId) });
	if (date) conditions.push({ startAt: clinicDayRange(date) });

	const q = buildListQuery(rawQuery, appointmentListConfig, conditions.length > 0 ? { AND: conditions } : undefined);

	const [rows, total] = await Promise.all([
		prisma.appointment.findMany({ ...q.args, select: appointmentSelect }),
		prisma.appointment.count({ where: q.where }),
	]);

	return { meta: buildMeta(total, q.page, q.limit), data: rows.map(toAppointmentResponse) };
};

const getAppointmentById = async (authUser: TAuthPayload, id: string) => {
	const scope = authUser.role === Role.DOCTOR ? { doctorId: await findOwnDoctorId(authUser.userId) } : {};
	return toAppointmentResponse(await findAppointmentOrThrow({ id, ...scope }));
};

const getMyAppointments = async (userId: string, rawQuery: unknown) => {
	const patientId = await findMyPatientId(userId);
	const q = buildListQuery(rawQuery, appointmentListConfig, { patientId });

	const [rows, total] = await Promise.all([
		prisma.appointment.findMany({ ...q.args, select: appointmentSelect }),
		prisma.appointment.count({ where: q.where }),
	]);

	return { meta: buildMeta(total, q.page, q.limit), data: rows.map(toAppointmentResponse) };
};

const getMyAppointmentById = async (userId: string, id: string) => {
	const patientId = await findMyPatientId(userId);
	// অন্য রোগীর appointment → 404 (403 না — আছে কিনা সেটাও জানানো হয় না)
	return toAppointmentResponse(await findAppointmentOrThrow({ id, patientId }));
};

// ---------------- Cancel ----------------

const cancelAppointment = async (authUser: TAuthPayload, id: string, { reason }: TCancelAppointmentPayload) => {
	const current = await findAppointmentOrThrow({ id });
	assertCancellable(current.status);

	const cancelled = await prisma.$transaction((tx) =>
		cancelActiveAppointments(tx, { id }, { reason, actorId: authUser.userId, now: new Date() }),
	);
	if (cancelled.length === 0) {
		throw new AppError(httpStatus.CONFLICT, "This appointment was just changed — refresh and try again");
	}

	await finalizeCancellations(cancelled, reason);
	return toAppointmentResponse(await findAppointmentOrThrow({ id }));
};

const cancelMyAppointment = async (userId: string, id: string, payload: TCancelMyAppointmentPayload) => {
	const patientId = await findMyPatientId(userId);
	const current = await findAppointmentOrThrow({ id, patientId });
	assertCancellable(current.status);

	const now = new Date();
	const minutesLeft = (current.startAt.getTime() - now.getTime()) / MINUTE_MS;
	if (minutesLeft < APPOINTMENT_CONFIG.patientCancelBeforeMinutes) {
		throw new AppError(
			httpStatus.CONFLICT,
			`Online cancellation closes ${APPOINTMENT_CONFIG.patientCancelBeforeMinutes / 60} hours before the appointment — please call the front desk`,
		);
	}

	const paidAtDesk = current.payments.some(
		(payment) => payment.provider === PaymentProvider.DESK && payment.status === PaymentStatus.SUCCEEDED,
	);
	if (paidAtDesk) {
		throw new AppError(httpStatus.CONFLICT, "You paid at the front desk — please contact the front desk to cancel");
	}

	const reason = payload.reason ?? "Cancelled by patient";
	const cancelled = await prisma.$transaction((tx) =>
		cancelActiveAppointments(tx, { id, patientId }, { reason, actorId: userId, now }),
	);
	if (cancelled.length === 0) {
		throw new AppError(httpStatus.CONFLICT, "This appointment was just changed — refresh and try again");
	}

	await finalizeCancellations(cancelled, reason);
	return toAppointmentResponse(await findAppointmentOrThrow({ id, patientId }));
};

// ---------------- Payment (desk + online link) ----------------

// Front desk-এ fee নেওয়া ("Pay at the front desk" বাছাই করা রোগী এলে, অথবা online link-এর বদলে cash)
const recordDeskPayment = async (authUser: TAuthPayload, id: string, { method }: TDeskPaymentPayload) => {
	const now = new Date();

	const sessionIds = await prisma.$transaction(async (tx) => {
		await lockAppointment(tx, id);

		const appointment = await tx.appointment.findUnique({
			where: { id },
			select: {
				status: true,
				holdExpiresAt: true,
				feeCents: true,
				payments: {
					where: { status: { in: [PaymentStatus.PENDING, PaymentStatus.SUCCEEDED] } },
					select: { status: true, stripeSessionId: true },
				},
			},
		});
		if (!appointment) throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");

		const holdActive =
			appointment.status === AppointmentStatus.PENDING_PAYMENT &&
			appointment.holdExpiresAt !== null &&
			appointment.holdExpiresAt > now;
		if (appointment.status !== AppointmentStatus.CONFIRMED && !holdActive) {
			throw new AppError(httpStatus.CONFLICT, `Can't take payment for a ${appointment.status} appointment`);
		}
		if (appointment.payments.some((payment) => payment.status === PaymentStatus.SUCCEEDED)) {
			throw new AppError(httpStatus.CONFLICT, "Consultation fee is already paid");
		}

		// খোলা Stripe link থাকলে সেটা বাতিল — একই fee দুবার না
		await tx.payment.updateMany({
			where: { appointmentId: id, status: PaymentStatus.PENDING },
			data: { status: PaymentStatus.FAILED },
		});
		await tx.payment.create({
			data: {
				appointmentId: id,
				provider: PaymentProvider.DESK,
				status: PaymentStatus.SUCCEEDED,
				deskMethod: method,
				receivedById: authUser.userId,
				amountCents: appointment.feeCents,
				currency,
				paidAt: now,
			},
		});
		if (holdActive) {
			await tx.appointment.update({
				where: { id },
				data: { status: AppointmentStatus.CONFIRMED, paymentMode: AppointmentPaymentMode.AT_DESK },
			});
		}

		return appointment.payments.flatMap((payment) => (payment.stripeSessionId ? [payment.stripeSessionId] : []));
	});

	await Promise.all(sessionIds.map(expireCheckoutSession));
	return toAppointmentResponse(await findAppointmentOrThrow({ id }));
};

// নতুন Stripe checkout: staff-এর "payment link" আর রোগীর "Pay now" — দুটোই এটা ব্যবহার করে
const startOnlinePayment = async (id: string, patientId: string | null) => {
	const now = new Date();

	const prepared = await prisma.$transaction(async (tx) => {
		await lockAppointment(tx, id);

		const appointment = await tx.appointment.findFirst({
			where: { id, ...(patientId ? { patientId } : {}) },
			select: {
				status: true,
				holdExpiresAt: true,
				feeCents: true,
				payments: {
					where: { status: { in: [PaymentStatus.PENDING, PaymentStatus.SUCCEEDED] } },
					select: { status: true, stripeSessionId: true },
				},
			},
		});
		if (!appointment) throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");

		if (appointment.payments.some((payment) => payment.status === PaymentStatus.SUCCEEDED)) {
			throw new AppError(httpStatus.CONFLICT, "Consultation fee is already paid");
		}

		const holdActive =
			appointment.status === AppointmentStatus.PENDING_PAYMENT &&
			appointment.holdExpiresAt !== null &&
			appointment.holdExpiresAt > now;
		if (appointment.status !== AppointmentStatus.CONFIRMED && !holdActive) {
			throw new AppError(
				httpStatus.CONFLICT,
				appointment.status === AppointmentStatus.PENDING_PAYMENT || appointment.status === AppointmentStatus.EXPIRED
					? "The booking hold has ended — please book again"
					: `Can't pay for a ${appointment.status} appointment`,
			);
		}
		if (appointment.feeCents < APPOINTMENT_CONFIG.minOnlineAmountCents) {
			throw new AppError(httpStatus.BAD_REQUEST, "This fee is too small for online payment — pay at the front desk");
		}

		await tx.payment.updateMany({
			where: { appointmentId: id, status: PaymentStatus.PENDING },
			data: { status: PaymentStatus.FAILED },
		});
		// Pending হলে hold আবার ৩০ মিনিট; CONFIRMED হলে slot আগেই নিশ্চিত, hold লাগে না
		await tx.appointment.update({
			where: { id },
			data: {
				paymentMode: AppointmentPaymentMode.ONLINE,
				...(holdActive ? { holdExpiresAt: new Date(now.getTime() + APPOINTMENT_CONFIG.holdMinutes * MINUTE_MS) } : {}),
			},
		});
		const payment = await tx.payment.create({
			data: { appointmentId: id, provider: PaymentProvider.STRIPE, amountCents: appointment.feeCents, currency },
			select: { id: true },
		});

		return {
			paymentId: payment.id,
			oldSessionIds: appointment.payments.flatMap((p) => (p.stripeSessionId ? [p.stripeSessionId] : [])),
		};
	});

	await Promise.all(prepared.oldSessionIds.map(expireCheckoutSession));

	const appointment = await findAppointmentOrThrow({ id });
	const checkoutUrl = await openCheckout(appointment, prepared.paymentId);
	return { appointment, checkoutUrl };
};

const sendPaymentLink = async (id: string) => {
	const { appointment, checkoutUrl } = await startOnlinePayment(id, null);
	const expiresAt = appointment.holdExpiresAt ?? new Date(Date.now() + APPOINTMENT_CONFIG.holdMinutes * MINUTE_MS);

	await sendSafely(`payment link ${appointment.appointmentNumber}`, () =>
		emailPaymentLink(appointment, checkoutUrl, currency, expiresAt),
	);

	return { appointment: toAppointmentResponse(appointment), checkoutUrl };
};

const payMyAppointment = async (userId: string, id: string) => {
	const patientId = await findMyPatientId(userId);
	const { appointment, checkoutUrl } = await startOnlinePayment(id, patientId);
	return { appointment: toAppointmentResponse(appointment), checkoutUrl };
};

export const AppointmentService = {
	bookAppointment,
	getAppointments,
	getAppointmentById,
	getMyAppointments,
	getMyAppointmentById,
	cancelAppointment,
	cancelMyAppointment,
	recordDeskPayment,
	sendPaymentLink,
	payMyAppointment,
};
```


**`src/app/module/appointment/appointment.controller.ts`**

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { idParamSchema } from "@/app/shared/commonValidation";
import { getAuthUser } from "@/app/shared/getAuthUser";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { AppointmentService } from "./appointment.service";
import { AppointmentValidation } from "./appointment.validation";

const bookAppointment = catchAsync(async (req, res) => {
	const result = await AppointmentService.bookAppointment(getAuthUser(req), req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: result.checkoutUrl ? "Appointment held — complete the payment within 30 minutes" : "Appointment booked",
		data: result,
	});
});

const getAppointments = catchAsync(async (req, res) => {
	const { date } = AppointmentValidation.listQuerySchema.parse(req.query);
	const { meta, data } = await AppointmentService.getAppointments(getAuthUser(req), req.query, date);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Appointments retrieved successfully", meta, data });
});

const getAppointmentById = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.getAppointmentById(getAuthUser(req), id);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Appointment retrieved successfully", data: result });
});

const cancelAppointment = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.cancelAppointment(getAuthUser(req), id, req.body);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Appointment cancelled", data: result });
});

const recordDeskPayment = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.recordDeskPayment(getAuthUser(req), id, req.body);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Consultation fee received", data: result });
});

const sendPaymentLink = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.sendPaymentLink(id);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Payment link sent to the patient", data: result });
});

// ---------- রোগী নিজে (/patient/appointments) ----------

const getMyAppointments = catchAsync(async (req, res) => {
	const { meta, data } = await AppointmentService.getMyAppointments(getAuthUser(req).userId, req.query);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Appointments retrieved successfully", meta, data });
});

const getMyAppointmentById = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.getMyAppointmentById(getAuthUser(req).userId, id);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Appointment retrieved successfully", data: result });
});

const cancelMyAppointment = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.cancelMyAppointment(getAuthUser(req).userId, id, req.body);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Appointment cancelled", data: result });
});

const payMyAppointment = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AppointmentService.payMyAppointment(getAuthUser(req).userId, id);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Checkout created", data: result });
});

export const AppointmentController = {
	bookAppointment,
	getAppointments,
	getAppointmentById,
	cancelAppointment,
	recordDeskPayment,
	sendPaymentLink,
	getMyAppointments,
	getMyAppointmentById,
	cancelMyAppointment,
	payMyAppointment,
};
```


**`src/app/module/appointment/appointment.route.ts`**

```ts
import { ADMINS, checkAuth, STAFF } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { AppointmentController } from "./appointment.controller";
import { AppointmentValidation } from "./appointment.validation";

// ---------- /appointments (রোগী book করে; বাকি সব staff) ----------
const router = Router();

router.post(
	"/",
	checkAuth(Role.PATIENT, ...ADMINS),
	validateRequest(AppointmentValidation.bookAppointmentSchema),
	AppointmentController.bookAppointment,
);

router.get("/", checkAuth(...STAFF), AppointmentController.getAppointments);
router.get("/:id", checkAuth(...STAFF), AppointmentController.getAppointmentById);

router.patch(
	"/:id/cancel",
	checkAuth(...ADMINS),
	validateRequest(AppointmentValidation.cancelAppointmentSchema),
	AppointmentController.cancelAppointment,
);
router.post(
	"/:id/desk-payment",
	checkAuth(...ADMINS),
	validateRequest(AppointmentValidation.deskPaymentSchema),
	AppointmentController.recordDeskPayment,
);
router.post("/:id/payment-link", checkAuth(...ADMINS), AppointmentController.sendPaymentLink);

export const AppointmentRoutes = router;

// ---------- /patient/appointments (শুধু রোগী নিজে; patientId সবসময় session থেকে) ----------
const patientRouter = Router();

patientRouter.use(checkAuth(Role.PATIENT));

patientRouter.get("/", AppointmentController.getMyAppointments);
patientRouter.get("/:id", AppointmentController.getMyAppointmentById);
patientRouter.patch(
	"/:id/cancel",
	validateRequest(AppointmentValidation.cancelMyAppointmentSchema),
	AppointmentController.cancelMyAppointment,
);
patientRouter.post("/:id/pay", AppointmentController.payMyAppointment);

export const PatientAppointmentRoutes = patientRouter;
```

## ৮. Slot module — `slot.service.ts` (পুরোটা replace)

বদল: public list-এ booked slot লুকানো, manage-এ `OPEN/BOOKED/BLOCKED`, block করলে appointment cancel + refund + email, booking হওয়া slot delete করা যায় না।


**`src/app/module/slot/slot.service.ts`**

```ts
import { envVars } from "@/app/config/env";
import { SLOT_CONFIG } from "@/app/config/slot.config";
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import type { TAuthPayload } from "@/app/utils/token";
import type { Prisma } from "@/generated/prisma/client";
import { Role, SlotStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import {
	activeAppointmentWhere,
	cancelActiveAppointments,
	finalizeCancellations,
} from "../appointment/appointment.helpers";
import type {
	TBlockSlotsPayload,
	TManageSlotQuery,
	TOpenSlotsPayload,
	TPublicSlotQuery,
	TUnblockSlotsPayload,
} from "./slot.interface";
import {
	addDaysToDate,
	buildSlots,
	clinicDateOf,
	clinicDayStartUtc,
	formatClinicTime,
	groupByClinicDate,
	hasOverlap,
} from "./slot.utils";

const TZ = envVars.CLINIC_TIME_ZONE;
const DAY_MS = 24 * 60 * 60 * 1000;

type TDateRange = { from: string; to: string };

// "me" বা id → আসল doctorProfile.id। Doctor শুধু নিজেরটা, admin যেকোনো doctor-এর
const resolveDoctorId = async (authUser: TAuthPayload, doctorIdParam: string): Promise<string> => {
	if (authUser.role === Role.DOCTOR) {
		const own = await prisma.doctorProfile.findUnique({
			where: { userId: authUser.userId },
			select: { id: true },
		});
		if (!own) throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
		if (doctorIdParam !== "me" && doctorIdParam !== own.id) {
			throw new AppError(httpStatus.FORBIDDEN, "You can only manage your own slots");
		}
		return own.id;
	}

	if (doctorIdParam === "me") {
		throw new AppError(httpStatus.BAD_REQUEST, "'me' works only for doctors — use a doctor id");
	}

	const doctor = await prisma.doctorProfile.findFirst({
		where: { id: doctorIdParam, user: { isDeleted: false } },
		select: { id: true },
	});
	if (!doctor) throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	return doctor.id;
};

// Clinic-এর তারিখ (inclusive) → UTC range, দিনের সীমা সহ
const toUtcRange = ({ from, to }: TDateRange, maxDays: number) => {
	const days = (Date.parse(to) - Date.parse(from)) / DAY_MS + 1;
	if (days > maxDays) {
		throw new AppError(httpStatus.BAD_REQUEST, `Date range can be at most ${maxDays} days`);
	}
	return { gte: clinicDayStartUtc(from, TZ), lt: clinicDayStartUtc(addDaysToDate(to, 1), TZ) };
};

const openSlots = async (authUser: TAuthPayload, doctorIdParam: string, payload: TOpenSlotsPayload) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);
	const now = new Date();
	const today = clinicDateOf(now, TZ);
	const lastDate = addDaysToDate(today, SLOT_CONFIG.openMaxDaysAhead);

	const pastDate = payload.dates.find((date) => date < today);
	if (pastDate) throw new AppError(httpStatus.BAD_REQUEST, `${pastDate} is in the past`);

	const tooFar = payload.dates.find((date) => date > lastDate);
	if (tooFar) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			`Slots can be opened at most ${SLOT_CONFIG.openMaxDaysAhead} days ahead (until ${lastDate})`,
		);
	}

	const allSlots = payload.dates.flatMap((date) =>
		buildSlots({
			date,
			startTime: payload.startTime,
			endTime: payload.endTime,
			slotMinutes: payload.slotMinutes,
			breaks: payload.breaks,
			timeZone: TZ,
		}),
	);
	const futureSlots = allSlots.filter((slot) => slot.startAt > now);

	const [firstSlot] = futureSlots;
	const lastSlot = futureSlots.at(-1);
	if (!firstSlot || !lastSlot) {
		throw new AppError(httpStatus.BAD_REQUEST, "All selected times are in the past");
	}
	if (futureSlots.length > SLOT_CONFIG.maxSlotsPerRequest) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			`This would create ${futureSlots.length} slots; the limit is ${SLOT_CONFIG.maxSlotsPerRequest} per request`,
		);
	}

	const created = await prisma.$transaction(async (tx) => {
		// একই doctor-এর দুটো open request একসাথে এলে একটা আরেকটার জন্য অপেক্ষা করে
		await tx.$executeRaw`SELECT pg_advisory_xact_lock(hashtext(${doctorId}))`;

		const existing = await tx.doctorSlot.findMany({
			where: { doctorId, startAt: { lt: lastSlot.endAt }, endAt: { gt: firstSlot.startAt } },
			select: { startAt: true, endAt: true },
		});

		const conflicts = futureSlots.filter((slot) => existing.some((other) => hasOverlap(slot, other)));
		const [firstConflict] = conflicts;
		if (firstConflict) {
			throw new AppError(
				httpStatus.CONFLICT,
				`${conflicts.length} new slot(s) overlap existing slots — first at ${formatClinicTime(firstConflict.startAt, TZ)}`,
			);
		}

		const result = await tx.doctorSlot.createMany({
			data: futureSlots.map((slot) => ({ doctorId, ...slot, createdById: authUser.userId })),
		});
		return result.count;
	});

	return { created, skippedPast: allSlots.length - futureSlots.length };
};

const getPublicSlots = async (doctorId: string, query: TPublicSlotQuery) => {
	const doctor = await prisma.doctorProfile.findFirst({
		where: { id: doctorId, user: { isDeleted: false, isActive: true } },
		select: { id: true },
	});
	if (!doctor) throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");

	const now = new Date();
	const today = clinicDateOf(now, TZ);
	// ১৪ দিনের window = আজ + পরের ১৩ দিন (আজ নিজেও ১ দিন)
	const lastBookableDate = addDaysToDate(today, SLOT_CONFIG.bookingWindowDays - 1);

	// অতীতের from দিলেও আজ থেকে ধরা হয় — অতীতের slot এমনিতেও book করা যায় না
	const from = query.from && query.from > today ? query.from : today;
	const to = query.to ?? lastBookableDate;

	if (from > lastBookableDate || to > lastBookableDate) {
		throw new AppError(httpStatus.BAD_REQUEST, `Booking is open until ${lastBookableDate}`);
	}
	if (from > to) {
		throw new AppError(httpStatus.BAD_REQUEST, "'from' must be on or before 'to'");
	}

	const range = toUtcRange({ from, to }, SLOT_CONFIG.bookingWindowDays);
	const earliest = new Date(now.getTime() + SLOT_CONFIG.minLeadMinutes * 60_000);

	const slots = await prisma.doctorSlot.findMany({
		where: {
			doctorId,
			AND: [
				{ startAt: range },
				{ startAt: { gte: earliest } },
				// booked slot (active appointment আছে) দেখানো হয় না; BLOCKED দেখায় কারণসহ
				{ OR: [{ status: SlotStatus.BLOCKED }, { appointments: { none: activeAppointmentWhere(now) } }] },
			],
		},
		select: { id: true, startAt: true, endAt: true, status: true, blockReason: true },
		orderBy: { startAt: "asc" },
	});

	const items = slots.map(({ status, blockReason, ...slot }) =>
		status === SlotStatus.BLOCKED
			? { ...slot, status: "BLOCKED" as const, blockReason }
			: { ...slot, status: "AVAILABLE" as const },
	);

	// from/to ফেরত দেওয়া হয় → frontend জানে calendar-এর কোন দিনগুলো দেখাবে
	return { from, to, days: groupByClinicDate(items, TZ) };
};

const getManageSlots = async (authUser: TAuthPayload, doctorIdParam: string, query: TManageSlotQuery) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);
	const range = toUtcRange(query, SLOT_CONFIG.manageMaxRangeDays);
	const now = new Date();

	const slots = await prisma.doctorSlot.findMany({
		where: { doctorId, startAt: range },
		select: {
			id: true,
			startAt: true,
			endAt: true,
			status: true,
			blockReason: true,
			blockedAt: true,
			appointments: {
				where: activeAppointmentWhere(now),
				select: { id: true, appointmentNumber: true, status: true, patient: { select: { fullName: true } } },
				take: 1,
			},
		},
		orderBy: { startAt: "asc" },
	});

	// OPEN / BOOKED / BLOCKED — "booked" DB-তে রাখা হয় না, active appointment থেকে বের করা হয়
	const items = slots.map(({ appointments, ...slot }) => {
		const [booking] = appointments;
		const state = slot.status === SlotStatus.BLOCKED ? "BLOCKED" : booking ? "BOOKED" : "OPEN";
		return { ...slot, state, booking: booking ?? null };
	});

	return groupByClinicDate(items, TZ);
};

// Prisma-তে undefined মানে "filter নেই"। তাই range ফাঁকা থাকলে সব slot block হয়ে যেত — এখানে আটকানো হলো
const toTargetFilter = (payload: TBlockSlotsPayload): Prisma.DoctorSlotWhereInput => {
	if (payload.slotIds) return { id: { in: payload.slotIds } };
	if (payload.from && payload.to) return { startAt: { lt: payload.to }, endAt: { gt: payload.from } };
	throw new AppError(httpStatus.BAD_REQUEST, "Send either slotIds or a from–to range");
};

const blockSlots = async (authUser: TAuthPayload, doctorIdParam: string, payload: TBlockSlotsPayload) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);
	const target = toTargetFilter(payload);
	const now = new Date();

	const result = await prisma.$transaction(async (tx) => {
		const blocked = await tx.doctorSlot.updateManyAndReturn({
			where: { doctorId, status: SlotStatus.OPEN, AND: [{ startAt: { gt: now } }, target] },
			data: {
				status: SlotStatus.BLOCKED,
				blockReason: payload.reason,
				blockedAt: now,
				blockedById: authUser.userId,
			},
			select: { id: true },
		});

		if (blocked.length === 0) {
			throw new AppError(httpStatus.BAD_REQUEST, "No open future slots matched — they may be past or already blocked");
		}

		// একই transaction-এ: এই slot-গুলোর active appointment → CANCELLED (cancelReason = block-এর কারণ)
		const cancelled = await cancelActiveAppointments(
			tx,
			{ slotId: { in: blocked.map((slot) => slot.id) } },
			{ reason: payload.reason, actorId: authUser.userId, now },
		);

		return { blocked: blocked.length, cancelled };
	});

	// Commit-এর পরে: Stripe refund + রোগীদের email (বাইরের call transaction-এ না)
	await finalizeCancellations(result.cancelled, payload.reason);

	return { blocked: result.blocked, cancelledAppointments: result.cancelled.length };
};

const unblockSlots = async (authUser: TAuthPayload, doctorIdParam: string, payload: TUnblockSlotsPayload) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);

	const result = await prisma.doctorSlot.updateMany({
		where: { doctorId, id: { in: payload.slotIds }, status: SlotStatus.BLOCKED, startAt: { gt: new Date() } },
		data: { status: SlotStatus.OPEN, blockReason: null, blockedAt: null, blockedById: null },
	});

	if (result.count === 0) {
		throw new AppError(httpStatus.BAD_REQUEST, "No blocked future slots matched");
	}
	return { unblocked: result.count };
};

const deleteSlot = async (authUser: TAuthPayload, doctorIdParam: string, slotId: string) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);

	const slot = await prisma.doctorSlot.findFirst({
		where: { id: slotId, doctorId },
		select: { id: true, startAt: true },
	});
	if (!slot) throw new AppError(httpStatus.NOT_FOUND, "Slot not found");
	if (slot.startAt <= new Date()) {
		throw new AppError(httpStatus.BAD_REQUEST, "Past slots can't be deleted");
	}

	// একবার booking হয়েছে এমন slot মোছা যায় না (appointment-এর ইতিহাস থাকে) → block করতে হবে
	const bookings = await prisma.appointment.count({ where: { slotId: slot.id } });
	if (bookings > 0) {
		throw new AppError(httpStatus.CONFLICT, "This slot has bookings — block it instead");
	}

	await prisma.doctorSlot.delete({ where: { id: slot.id } });
};

export const SlotService = {
	openSlots,
	getPublicSlots,
	getManageSlots,
	blockSlots,
	unblockSlots,
	deleteSlot,
};
```

## ৯. `src/app/routes/index.ts`

```ts
import { AppointmentRoutes, PatientAppointmentRoutes } from "../module/appointment/appointment.route";

// moduleRoutes-এ, "/patient"-এর আগে:
	{ path: "/appointments", route: AppointmentRoutes },
	// "/patient"-এর আগে: PatientRoutes-এর router.use(checkAuth) দুবার না চলে
	{ path: "/patient/appointments", route: PatientAppointmentRoutes },
	{ path: "/patient", route: PatientRoutes },
```

## ১০. `src/app.ts` — webhook, `express.json()`-এর **আগে**

```ts
import { PaymentController } from "./app/module/payment/payment.controller";

// Stripe webhook: signature যাচাইয়ের জন্য raw body (Buffer) লাগে → express.json()-এর আগে
app.post(
	"/api/v1/payments/stripe/webhook",
	express.raw({ type: "application/json" }),
	PaymentController.handleStripeWebhook,
);

app.use(express.json());
```

## ১১. `src/server.ts` — cron চালু/বন্ধ

```ts
import type { ScheduledTask } from "node-cron";
import { startPaymentJobs } from "./app/module/payment/payment.job";

let cronTasks: ScheduledTask[] = [];

// app.listen-এর callback-এ, console.info-র পরে:
		cronTasks = startPaymentJobs();

// shutdown()-এর শুরুতে:
	for (const task of cronTasks) {
		void task.stop();
	}
```

## ১২. যা চালিয়ে যাচাই করা হয়েছে (আসল Postgres, তোমার code-এর copy)

| Case | ফল |
|---|---|
| দুজন রোগী একই slot একসাথে (`Promise.all`) | একটা 201 `APT-2026-000001`, একটা 409 |
| একই রোগী, একই doctor, একই দিনে দ্বিতীয় booking | 409 |
| রোগী এখন চলছে এমন slot | 400 (১ ঘণ্টা আগের নিয়ম) |
| Admin একই slot + `deskPaymentMethod: CASH` | 201, `isPaid: true`, `DESK/SUCCEEDED/CASH` |
| রোগী `deskPaymentMethod` পাঠালে | 403 |
| Stripe fail (ভুল key) | 502, appointment `EXPIRED`, slot সাথে সাথে আবার book করা গেল |
| Webhook `completed` | `CONFIRMED` + payment `SUCCEEDED`; একই event আবার → "already processed" |
| Paid appointment-এ desk payment | 409 |
| Hold শেষ → cron | `EXPIRED`, payment `FAILED`; public list-এ slot আবার দেখায় |
| Hold শেষের পরে দেরিতে payment (slot খালি) | আবার `CONFIRMED` |
| Manage view | `BOOKED APT-...` / `OPEN` |
| দুটো slot block "Doctor is sick" | ২টা appointment `CANCELLED`, cancelReason = কারণ |
| Booking হওয়া slot delete | 409 "block it instead" |
| রোগী ২ ঘণ্টার কম আগে cancel | 409 |
| রোগী অন্যের appointment দেখতে চাইলে | 404 |
| Admin desk-paid appointment cancel | payment `REFUNDED` |