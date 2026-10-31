# 28 - Stripe Payment + Webhook + Cron (auto cancel)

```
Patient → book-appointment / initiate-payment → Stripe Checkout link (paymentUrl)
→ browser-এ card দিয়ে pay → Stripe → POST /webhook (checkout.session.completed)
→ signature যাচাই → Appointment + Payment = PAID
Cron (প্রতি ৫ মিনিট) → ৩০ মিনিটের পুরনো UNPAID → CANCELED + slot ফাঁকা
```

> Appointment অংশ [নোট ২৭](./27-appointment.md)-এ। এখানে Stripe setup, webhook আর cron।

---

## ধাপ ১: Stripe account (sandbox)

1. [stripe.com](https://stripe.com/) → **Sign up** → Sign up with Google।
2. Country list-এ Bangladesh নেই, তাই **United States** দিয়ে next।
3. Business name: `phdoc` → **Set up manually** → **Go to sandbox**।
4. বাম দিকের নিচে **Developers** → **API keys** → **Secret key** কপি (`sk_test_...`)। Publishable key backend-এ লাগে না।

---

## ধাপ ২: Stripe CLI install + login

Webhook key-এর জন্য Google-এ "stripe webhook" খুঁজলে docs পাবে: <https://dashboard.stripe.com/webhooks>। Local-এ test করতে **Stripe CLI** লাগে: <https://docs.stripe.com/cli>

```shell
# আগে থেকে login থাকলে logout
stripe logout --all

# login (browser খুলবে, allow করো)
stripe login
```

---

## ধাপ ৩: `package.json`-এ webhook script

```json
{
	"scripts": {
		"stripe:webhook": "stripe listen --forward-to localhost:5000/webhook"
	}
}
```

- `5000` কারণ আমাদের server port 5000।
- URL-এ `/api/v1` নেই — তাই `/webhook` route আলাদা করে `app.ts`-এ বানাতে হবে (ধাপ ৪)। Stripe-এর event এই `localhost:5000/webhook`-এ আসবে।

> ✏️ তোমার project-এর `package.json`-এ script-টা `stripe listen --all-snapshot --forward-to localhost:5000/webhook` — নোটের সাথে একটু আলাদা।

---

## ধাপ ৪: `app.ts`-এ test webhook (আসল code লেখার আগে)

আগে শুধু দেখার জন্য — event আসছে কিনা:

```ts
app.post("/webhook", express.raw({ type: "application/json" }), async (req: Request, res: Response) => {
	console.warn("Webhook received:", req.body);
	res.status(200).json({ received: true });
});
```

> ✏️ `console.log` → `console.warn` (oxlint-এর `no-console` শুধু `warn`/`error` চলতে দেয়)।

---

## ধাপ ৫: Dashboard-এ webhook destination

[Workbench → Webhooks](https://dashboard.stripe.com/test/workbench/webhooks) → **Add destination** → events বাছাই:

```
checkout.session.async_payment_failed
checkout.session.async_payment_succeeded
checkout.session.expired
checkout.session.completed
payment_intent.succeeded
payment_intent.payment_failed
```

**কোনগুলো আসলে দরকার?** (আমাদের code অনুযায়ী)

| Event | লাগবে? | কেন |
|---|---|---|
| `checkout.session.completed` | ✅ **অবশ্যই** | এটা পেলেই DB-তে PAID হয় |
| `checkout.session.expired` | ✅ রাখো | শুধু message দেয় (cron পরে cancel করে) |
| `payment_intent.payment_failed` | ✅ রাখো | শুধু message দেয় |
| `checkout.session.async_payment_succeeded` / `_failed` | ❌ এখন না | শুধু card নিচ্ছি (card তাৎক্ষণিক) — bank transfer-এর মতো ধীর পদ্ধতি যোগ করলে লাগবে |
| `payment_intent.succeeded` | ❌ না | `checkout.session.completed`-এই একই খবর আসে, code-এ handle নেই |

এরপর **Continue** → **Webhook endpoint** → **Continue** → Destination name: `phdoc-webhook`, Endpoint URL: `http://localhost:5000/webhook` → **Add**।

> Endpoint URL-এ আসল URL দিতে হবে **deploy-এর পরে** (যেমন `https://api.phdoc.com/webhook`)। Local-এ event আসে **Stripe CLI** দিয়ে (ধাপ ৬), dashboard destination ছাড়াও চলে। Dashboard সাধারণত `localhost` URL নেয় না — না নিলে deploy পর্যন্ত এই ধাপ বাদ রাখো।

**দুইটা terminal:**

| Terminal ১ | Terminal ২ |
|---|---|
| `pnpm dev` | `pnpm stripe:webhook` |

Stripe payment সফল হলে Stripe নিজে আমাদের webhook-এ request পাঠায় — সেই request দেখে আমরা appointment-এর payment status update করি।

---

## ধাপ ৬: Webhook secret → `.env`

```shell
pnpm stripe:webhook
```

চালালে terminal-এ দেখাবে `Your webhook signing secret is whsec_...` → `.env`-এ রাখো, তারপর:

```shell
pnpm dev
```

> ✏️ `.env` — এটা নোটে ছিল না:

```dotenv
STRIPE_SECRET_KEY=sk_test_xxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxx
```

---

## ধাপ ৭: `env.ts`-এ Stripe

```ts
// interface EnvConfig-এ
STRIPE: {
	STRIPE_SECRET_KEY: string;
	STRIPE_WEBHOOK_SECRET: string;
};
```

> ✏️ Required list আর return object — এটা নোটে ছিল না:

```ts
// required list-এ
"STRIPE_SECRET_KEY",
"STRIPE_WEBHOOK_SECRET",

// return object-এ
STRIPE: {
	STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY as string,
	STRIPE_WEBHOOK_SECRET: process.env.STRIPE_WEBHOOK_SECRET as string,
},
```

---

## ধাপ ৮: `src/app/config/stripe.config.ts`

```ts
import { Stripe } from "stripe";
import { envVars } from "./env";

export const stripe = new Stripe(envVars.STRIPE.STRIPE_SECRET_KEY, {
	apiVersion: "2026-09-30.endive",
});
```

- `apiVersion` fixed রাখলে Stripe API বদলালেও আমাদের code ভাঙবে না।

---

## ধাপ ৯: `payment.service.ts`

```ts
// src/app/module/payment/payment.service.ts
import { stripe } from "@/app/config/stripe.config";
import { prisma } from "@/app/lib/prisma";
import { AppointmentStatus, PaymentStatus } from "@/generated/prisma/enums";
import type { Stripe } from "stripe";

const getPaymentIntentId = (session: Stripe.Checkout.Session) => {
	if (!session.payment_intent) return null;
	return typeof session.payment_intent === "string" ? session.payment_intent : session.payment_intent.id;
};

// টাকা কেটে গেছে কিন্তু appointment আর valid না → Stripe-কে refund করতে বলো।
// idempotencyKey দেওয়ায় Stripe retry করলেও দুইবার refund হবে না।
const refundCheckoutSession = async (session: Stripe.Checkout.Session, eventId: string) => {
	const paymentIntentId = getPaymentIntentId(session);

	if (session.payment_status !== "paid" || !paymentIntentId) {
		return false;
	}

	await stripe.refunds.create({ payment_intent: paymentIntentId }, { idempotencyKey: `refund-${eventId}` });

	return true;
};

const handleCheckoutSessionCompleted = async (session: Stripe.Checkout.Session, eventId: string) => {
	const appointmentId = session.metadata?.appointmentId;
	const paymentId = session.metadata?.paymentId;

	if (!appointmentId || !paymentId) {
		console.error(`Missing appointmentId or paymentId in metadata of session ${session.id}`);
		return { message: "Missing appointmentId or paymentId in session metadata" };
	}

	const appointment = await prisma.appointment.findUnique({
		where: { id: appointmentId },
		include: { payment: true },
	});

	if (!appointment) {
		console.error(`Appointment ${appointmentId} not found for session ${session.id}`);
		return { message: `Appointment ${appointmentId} not found` };
	}

	// Cancel হয়ে গেছে (cron payment delete করে দিয়েছে), অথবা আগেই paid (দুই tab-এ দুইবার pay)
	const isInvalidPayment =
		appointment.status === AppointmentStatus.CANCELED ||
		!appointment.payment ||
		appointment.payment.id !== paymentId ||
		appointment.payment.status === PaymentStatus.PAID;

	if (isInvalidPayment) {
		const refunded = await refundCheckoutSession(session, eventId);
		console.warn(`Invalid payment for appointment ${appointmentId}. Refunded: ${refunded}`);
		return { message: `Appointment is not payable. Refunded: ${refunded}` };
	}

	const paymentStatus = session.payment_status === "paid" ? PaymentStatus.PAID : PaymentStatus.UNPAID;

	await prisma.$transaction([
		// ✅ appointment-এর paymentStatus update, যাতে cron আর এটা cancel না করে
		prisma.appointment.update({
			where: { id: appointmentId },
			data: { paymentStatus },
		}),
		prisma.payment.update({
			where: { id: paymentId },
			data: {
				status: paymentStatus,
				stripeEventId: eventId,
				paymentGatewayData: {
					sessionId: session.id,
					paymentIntentId: getPaymentIntentId(session),
					amountTotal: session.amount_total,
					currency: session.currency,
					paymentStatus: session.payment_status,
					customerEmail: session.customer_details?.email ?? null,
				},
			},
		}),
	]);

	return { message: `Payment ${paymentStatus} for appointment ${appointmentId}` };
};

const handleStripeWebhookEvent = async (event: Stripe.Event) => {
	// Idempotency: Stripe একই event আবার পাঠালে skip
	const alreadyProcessed = await prisma.payment.findUnique({
		where: { stripeEventId: event.id },
		select: { id: true },
	});

	if (alreadyProcessed) {
		return { message: `Event ${event.id} already processed. Skipping` };
	}

	switch (event.type) {
		case "checkout.session.completed":
			// event.type দেখে TypeScript নিজেই বোঝে এখানে object হলো Checkout.Session
			return handleCheckoutSessionCompleted(event.data.object, event.id);

		case "checkout.session.expired":
			// কিছু করার দরকার নেই: appointment UNPAID থাকবে, cron পরে cancel করবে
			return { message: `Checkout session ${event.data.object.id} expired` };

		case "payment_intent.payment_failed":
			return { message: `Payment intent ${event.data.object.id} failed` };

		default:
			return { message: `Unhandled event type ${event.type}` };
	}
};

export const PaymentService = {
	handleStripeWebhookEvent,
};
```

---

## ধাপ ১০: `payment.controller.ts`

```ts
// src/app/module/payment/payment.controller.ts
import { envVars } from "@/app/config/env";
import { stripe } from "@/app/config/stripe.config";
import AppError from "@/app/errorHelpers/AppError";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import type Stripe from "stripe";
import { PaymentService } from "./payment.service";

const handleStripeWebhookEvent = catchAsync(async (req: Request, res: Response) => {
	const signature = req.headers["stripe-signature"];

	if (typeof signature !== "string") {
		throw new AppError(httpStatus.BAD_REQUEST, "Missing Stripe signature");
	}

	let event: Stripe.Event;

	try {
		// req.body এখানে Buffer (express.raw), না হলে verify fail করবে
		event = stripe.webhooks.constructEvent(req.body, signature, envVars.STRIPE.STRIPE_WEBHOOK_SECRET);
	} catch (error) {
		console.error("Stripe webhook signature verification failed:", error);
		throw new AppError(httpStatus.BAD_REQUEST, "Invalid Stripe webhook signature");
	}

	// এখানে error হলে globalErrorHandler 500 দেবে, আর Stripe নিজে retry করবে
	const result = await PaymentService.handleStripeWebhookEvent(event);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Stripe webhook event processed successfully",
		data: result,
	});
});

export const PaymentController = {
	handleStripeWebhookEvent,
};
```

---

## ধাপ ১১: `app.ts`-এ আসল webhook mount

ধাপ ৪-এর test route সরিয়ে এটা বসাও — `app.all("/api/auth/{*any}", ...)`-এর ঠিক নিচে, `express.urlencoded` / `express.json()`-এর **আগে**:

```ts
import { PaymentController } from "./app/module/payment/payment.controller";

// Stripe webhook: signature যাচাইয়ের জন্য raw body লাগে, তাই express.json()-এর আগে
app.post("/webhook", express.raw({ type: "application/json" }), PaymentController.handleStripeWebhookEvent);
```

---

## ধাপ ১২: `server.ts`-এ cron job

```ts
import { schedule } from "node-cron";
import { AppointmentService } from "./app/module/appointment/appointment.service";

const startCronJobs = () => {
	// প্রতি ৫ মিনিটে: ৩০ মিনিটের পুরনো unpaid appointment cancel করে slot ফাঁকা করা
	schedule(
		"*/5 * * * *",
		async () => {
			try {
				const { canceledCount } = await AppointmentService.cancelUnpaidAppointments();

				if (canceledCount > 0) {
					console.warn(`[cron] Canceled ${canceledCount} unpaid appointment(s)`);
				}
			} catch (error) {
				console.error("[cron] Failed to cancel unpaid appointments:", error instanceof Error ? error.message : error);
			}
		},
		{
			name: "cancel-unpaid-appointments",
			noOverlap: true, // আগের run শেষ না হলে নতুন run শুরু হবে না
		},
	);
};

// startServer-এর ভেতরে, app.listen-এর callback-এ call
startCronJobs();
```

> ✏️ Import দুটো নোটে ছিল না। তোমার project-এ `startCronJobs()` `app.listen`-এর callback-এ, আর shutdown-এর সময় `shutdown as shutdownCron` দিয়ে cron বন্ধ করা হয়।

| `"*/5 * * * *"` | মানে |
|---|---|
| ৫টা ঘর | মিনিট, ঘণ্টা, দিন, মাস, সপ্তাহের দিন |
| `*/5` | প্রতি ৫ মিনিটে চলবে |

- **Cron কত ঘন ঘন চলে** (`*/5`) আর **কত পুরনো হলে cancel** (`UNPAID_APPOINTMENT_TIMEOUT_MINUTES = 30`) — দুটো আলাদা জিনিস।
- `*/5` → appointment ৩০–৩৫ মিনিটের মধ্যে cancel হয়। `*/30` করলে ৩০–৬০ মিনিট লেগে যেতে পারে। তাই ছোট interval ভালো।

---

# 📘 বুঝে নাও — Stripe payment কীভাবে কাজ করে

## কেন Stripe Checkout?

Card-এর তথ্য আমাদের server-এ আসলে অনেক নিরাপত্তার দায়িত্ব। **Stripe Checkout**-এ user Stripe-এর নিজের page-এ card দেয় — আমরা শুধু একটা link বানাই আর পরে ফলাফল জানি।

## পুরো flow (৪ ধাপ)

```
① Book / initiate-payment
   আমাদের server → stripe.checkout.sessions.create(...) → paymentUrl
   metadata-তে { appointmentId, paymentId } রেখে দিই ← পরে চেনার জন্য

② User paymentUrl-এ যায় → card দেয় → Stripe টাকা কাটে
   → success_url-এ (frontend) ফেরত যায়

③ Stripe → আমাদের POST /webhook (checkout.session.completed)
   ⚠️ success_url-এ ফেরা দিয়ে PAID ধরি না — user tab বন্ধ করে দিতে পারে, বা URL নকল করতে পারে।
      Stripe-এর server থেকে আসা webhook-ই বিশ্বাসযোগ্য।

④ Webhook handler → metadata থেকে appointmentId → Appointment + Payment = PAID
```

## Webhook-এর ৩টা নিরাপত্তা

| কী | কেন | কোথায় |
|---|---|---|
| **Signature যাচাই** | যে কেউ `/webhook`-এ নকল "paid" request পাঠাতে পারে। Stripe প্রতিটা request `whsec_` secret দিয়ে sign করে — `constructEvent` মিলিয়ে দেখে। | controller |
| **Raw body** (`express.raw`) | Signature আসল bytes-এর উপর হয়। `express.json()` আগে parse করে ফেললে bytes বদলে যায় → যাচাই fail। তাই route `express.json()`-এর **আগে**। | `app.ts` |
| **Idempotency** (`stripeEventId @unique`) | Stripe একই event একাধিকবার পাঠাতে পারে (retry)। আগে process হলে skip। | service |

## উল্টাপাল্টা অবস্থায় refund

| অবস্থা | কী হয় |
|---|---|
| ৩০ মিনিট পর cron cancel করেছে, তারপর user pay করল | appointment CANCELED → **refund** |
| দুই tab খুলে দুবার pay | দ্বিতীয়বার payment আগেই PAID → **refund** |
| Stripe একই refund event আবার পাঠাল | `idempotencyKey: refund-<eventId>` → Stripe দ্বিতীয়বার refund করে না |

## Cron কেন?

Pay না করা appointment slot আটকে রাখে — অন্য রোগী নিতে পারে না। তাই প্রতি ৫ মিনিটে cron:

```
SCHEDULED + UNPAID + ৩০ মিনিটের বেশি পুরনো
→ transaction: status = CANCELED, payment row মুছে, slot isBooked = false
```

Payment row মুছে দেওয়ায় পরে কেউ পুরনো link-এ pay করলে webhook `!appointment.payment` দেখে refund করে।

## টাকার হিসাব

- Stripe টাকা নেয় **সবচেয়ে ছোট একক**-এ (BDT → পয়সা)। ৫০০ টাকা = `50000`। তাই `amount * 100`।
- `Math.round` → `19.99 * 100 = 1998.9999...` (Float-এর সমস্যা) → `1999`।

---

## Test (Postman + Browser)

**আগে:** দুই terminal-এ `pnpm dev` আর `pnpm stripe:webhook` চালু। PATIENT login, আর doctor-এর একটা ফাঁকা slot ([নোট ২৭](./27-appointment.md)-এর প্রস্তুতি)।

### ১. Pay Now — সফল payment

1. `POST http://localhost:5000/api/v1/appointments/book-appointment`

	```json
	{ "doctorId": "<doctor-id>", "scheduleId": "<schedule-id>" }
	```

	→ `201`, response-এ `paymentUrl`।

2. `paymentUrl` **browser-এ** খোলো (Postman-এ না) → test card:

	| Field | Value |
	|---|---|
	| Card | `4242 4242 4242 4242` |
	| Expiry | ভবিষ্যতের যেকোনো (যেমন `12/34`) |
	| CVC | যেকোনো ৩ সংখ্যা |
	| নাম / email | যেকোনো |

3. Pay → `success_url`-এ (frontend) যাবে — frontend না চললে page error দেখাবে, সমস্যা নেই।
4. **Terminal ২**-এ দেখবে: `checkout.session.completed ... [200]`।
5. Prisma Studio: `appointments.paymentStatus = PAID`, `payments.status = PAID`, `stripeEventId` আর `paymentGatewayData` ভরা।

### ২. Pay Later → পরে pay

1. `POST /appointments/book-appointment-with-pay-later` (অন্য slot) → `appointment.id` কপি।
2. `POST http://localhost:5000/api/v1/appointments/initiate-payment/<appointment-id>` → `paymentUrl` → browser-এ pay → PAID।

| Case | ফল |
|---|---|
| PAID appointment-এ আবার initiate | `400` Payment already completed for this appointment |
| CANCELED appointment-এ initiate | `400` Appointment is canceled |
| অন্য patient-এর appointment | `404` |

### ৩. Card declined

Card `4000 0000 0000 0002` → Stripe page-এই "card declined" দেখাবে, webhook-এ `checkout.session.completed` আসবে না → appointment UNPAID থাকবে।

### ৪. Cron auto cancel

Pay Later দিয়ে book করে কিছু না করে **৩০–৩৫ মিনিট** অপেক্ষা → server terminal-এ `[cron] Canceled 1 unpaid appointment(s)`। DB: status CANCELED, payment row নেই, slot `isBooked: false`।

> দ্রুত test করতে সাময়িক `UNPAID_APPOINTMENT_TIMEOUT_MINUTES = 1` আর cron `"* * * * *"` (প্রতি মিনিট) করো — test শেষে ফিরিয়ে দিও।

### ৫. Refund (cancel-এর পরে pay)

1. `book-appointment` → `paymentUrl` রেখে দাও, **pay করো না**।
2. Cron cancel করা পর্যন্ত অপেক্ষা (উপরের মতো)।
3. এখন পুরনো `paymentUrl`-এ pay → Terminal ২-এ `[200]`, response message: `Appointment is not payable. Refunded: true`। Stripe Dashboard → **Transactions**-এ refund দেখবে।

> Stripe Checkout link নিজেও ২৪ ঘণ্টা পর expire হয়।

### ৬. নকল webhook আটকানো

Postman: `POST http://localhost:5000/webhook`, body যেকোনো JSON →

| Header | ফল |
|---|---|
| `stripe-signature` ছাড়া | `400` Missing Stripe signature |
| `stripe-signature: abc` | `400` Invalid Stripe webhook signature |

### ৭. Stripe CLI দিয়ে event

```shell
# metadata ছাড়া একটা নকল completed event
stripe trigger checkout.session.completed
```

→ Terminal ২-এ `[200]`, message: `Missing appointmentId or paymentId in session metadata` (আমাদের metadata নেই, তাই কিছু বদলায় না)।

```shell
# আগের কোনো event আবার পাঠানো (id Terminal ২ বা Dashboard থেকে)
stripe events resend evt_xxxxxxxx
```

→ সফল payment-এর event হলে: `Event evt_... already processed. Skipping`।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Cron আর webhook একসাথে চললে paid appointment cancel হতে পারে।** `cancelUnpaidAppointments` আগে `findMany` করে UNPAID খোঁজে, তারপর আলাদা query-তে cancel করে। এই মাঝের মুহূর্তে webhook PAID করে ফেললে cron তবুও cancel করবে আর payment মুছে দেবে (বিরল, untested)। `updateMany`-তে আবার শর্ত দিলে নিরাপদ:

	```ts
	await tx.appointment.updateMany({
		where: { id: { in: appointmentIds }, status: AppointmentStatus.SCHEDULED, paymentStatus: PaymentStatus.UNPAID },
		data: { status: AppointmentStatus.CANCELED },
	});

	await tx.payment.deleteMany({
		where: { appointmentId: { in: appointmentIds }, status: PaymentStatus.UNPAID },
	});
	```

2. **Dashboard-এ শুধু দরকারি event।** ধাপ ৫-এর table অনুযায়ী `checkout.session.completed`, `checkout.session.expired`, `payment_intent.payment_failed` রাখলেই চলে — বাকিগুলো শুধু `Unhandled event type` লিখে বাড়তি request বাড়ায়।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Deploy-এর পরে আলাদা secret।** `stripe listen`-এর `whsec_` শুধু local CLI-র। Production-এ dashboard-এ আসল URL দিয়ে endpoint বানালে সেটার নিজের signing secret আসে — সেটা production `.env`-এ দিতে হবে।
4. **Refund DB-তে লেখা হয় না।** Refund হলে শুধু log হয়। পরে হিসাব রাখতে চাইলে `PaymentStatus`-এ `REFUNDED` যোগ করে রেখে দিতে পারো।
5. **Async payment।** এখন শুধু card, তাই `checkout.session.completed`-এ `payment_status` সবসময় `paid`। পরে bank transfer-এর মতো ধীর পদ্ধতি যোগ করলে `checkout.session.async_payment_succeeded` handle করতে হবে, নাহলে টাকা এলেও UNPAID থাকবে।

## 🔜 পরে যা যোগ হবে

- [ ] Frontend: `paymentUrl`-এ redirect, `payment-success` page
- [ ] Deploy-এর পর dashboard webhook endpoint + production secret
- [ ] Cancel-এ refund ([নোট ২৭](./27-appointment.md))
