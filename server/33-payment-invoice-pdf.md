# 33 - Payment Invoice PDF + Shared PDF Helper

Payment সফল হলে patient-কে একটা **invoice PDF** পাঠানো হবে। Prescription-এর PDF code-টা ([নোট ৩২](./32-prescription.md)) এখন একটা **shared helper** (`utils/pdf.ts`)-এ সরানো হয়েছে, যাতে prescription আর invoice দুটোই একই helper ব্যবহার করে।

```
webhook → DB transaction (PAID) → Stripe-কে সাথে সাথে 200
                └─ background-এ: PDF → upload → invoiceUrl save → email
```

> Stripe setup, webhook আর cron আছে [নোট ২৮](./28-stripe-payment.md)-এ। এই নোটে শুধু invoice অংশ আর নতুন সাজানো `payment.service.ts`।

| File | কী আছে |
| --- | --- |
| `src/app/utils/pdf.ts` | নতুন — shared PDF helper (`createPdfBuffer`, `formatDate`, `PDF_COLORS`) |
| `prescription/prescription.utils.ts` | বদলেছে — এখন shared helper ব্যবহার করে |
| `prescription/prescription.service.ts` | শুধু import লাইন বদলেছে |
| `payment/payment.utils.ts` | নতুন — invoice PDF |
| `payment/payment.service.ts` | বদলেছে — `processInvoice` যোগ হয়েছে |
| `utils/email.ts` | `"invoice"` template নাম যোগ |
| `templates/invoice.ejs` | নতুন — invoice email |

> Payment model-এ `invoiceUrl String?` field আগে থেকেই আছে, তাই migration লাগবে না।

---

## ধাপ ১: Shared PDF helper — `src/app/utils/pdf.ts` (নতুন file)

```ts
import PDFDocument from "pdfkit";

type PdfDocument = InstanceType<typeof PDFDocument>;

// server UTC-তে চললেও তারিখ বাংলাদেশের সময়ে দেখাবে
const TIME_ZONE = "Asia/Dhaka";

export const formatDate = (date: Date): string =>
	date.toLocaleDateString("en-GB", { timeZone: TIME_ZONE, day: "2-digit", month: "short", year: "numeric" });

export const formatDateTime = (date: Date): string =>
	date.toLocaleString("en-GB", { timeZone: TIME_ZONE, dateStyle: "medium", timeStyle: "short" });

export const PDF_COLORS = {
	primary: "#0b5ed7",
	text: "#222222",
	muted: "#666666",
	border: "#cccccc",
} as const;

export interface IPdfHelpers {
	doc: PdfDocument;
	left: number;
	right: number;
	contentWidth: number;
	header: (title: string) => void;
	divider: () => void;
	sectionTitle: (title: string) => void;
	infoRow: (label: string, value: string) => void;
	footer: (lines: string[]) => void;
}

// pdfkit stream-এ একটু একটু করে data দেয় → সব chunk জোড়া লাগিয়ে একটা Buffer।
// draw-এর ভেতরে শুধু "কী আঁকবো" লিখলেই হবে।
export const createPdfBuffer = (title: string, draw: (helpers: IPdfHelpers) => void): Promise<Buffer> => {
	return new Promise((resolve, reject) => {
		const doc = new PDFDocument({ size: "A4", margin: 50, info: { Title: title, Author: "PH Doc" } });

		const chunks: Buffer[] = [];
		doc.on("data", (chunk: Buffer) => {
			chunks.push(chunk);
		});
		doc.on("end", () => resolve(Buffer.concat(chunks)));
		doc.on("error", reject);

		const left = doc.page.margins.left;
		const right = doc.page.width - doc.page.margins.right;
		const contentWidth = right - left;

		const divider = () => {
			doc.moveDown(0.5);
			doc.moveTo(left, doc.y).lineTo(right, doc.y).lineWidth(1).strokeColor(PDF_COLORS.border).stroke();
			doc.moveDown(0.8);
		};

		const header = (heading: string) => {
			doc.font("Helvetica-Bold")
				.fontSize(22)
				.fillColor(PDF_COLORS.primary)
				.text(heading, left, doc.y, { width: contentWidth, align: "center" });
			doc.font("Helvetica")
				.fontSize(10)
				.fillColor(PDF_COLORS.muted)
				.text("PH Doc - Your Health, Our Priority", left, doc.y, { width: contentWidth, align: "center" });
			divider();
		};

		const sectionTitle = (heading: string) => {
			doc.font("Helvetica-Bold").fontSize(12).fillColor(PDF_COLORS.primary).text(heading, left, doc.y);
			doc.moveDown(0.3);
		};

		// "Name: John" — label bold, value normal; প্রতিবার x = left থেকে শুরু
		const infoRow = (label: string, value: string) => {
			doc.font("Helvetica-Bold")
				.fontSize(10)
				.fillColor(PDF_COLORS.text)
				.text(`${label}: `, left, doc.y, { continued: true });
			doc.font("Helvetica").fillColor(PDF_COLORS.muted).text(value);
		};

		const footer = (lines: string[]) => {
			doc.font("Helvetica").fontSize(9).fillColor(PDF_COLORS.muted);
			for (const line of lines) {
				doc.text(line, left, doc.y, { width: contentWidth, align: "center" });
			}
		};

		draw({ doc, left, right, contentWidth, header, divider, sectionTitle, infoRow, footer });

		doc.end();
	});
};
```

**Idea টা কী?** Buffer বানানো, margin হিসাব, header/divider/footer — এগুলো সব PDF-এ একই। তাই এগুলো `createPdfBuffer`-এর ভেতরে রাখা হয়েছে, আর প্রতিটা PDF শুধু `draw` function-এ বলে দেয় "কী কী আঁকবো"।

> `text(value, left, doc.y, ...)` — প্রতিবার x হিসেবে `left` দেওয়া হচ্ছে, যাতে আগের কোনো লেখা x সরিয়ে দিলেও পরের লেখা বাম margin থেকেই শুরু হয়।

---

## ধাপ ২: `prescription.utils.ts` — shared helper দিয়ে (পুরো file)

`src/app/module/prescription/prescription.utils.ts`

```ts
import { envVars } from "@/app/config/env";
import { createPdfBuffer, formatDate, formatDateTime, PDF_COLORS } from "@/app/utils/pdf";

export interface IPrescriptionPdfData {
	prescriptionId: string;
	doctorName: string;
	doctorEmail: string;
	specialization: string;
	patientName: string;
	patientEmail: string;
	appointmentDate: Date;
	issuedDate: Date;
	followUpDate: Date | null;
	instructions: string;
}

export const generatePrescriptionPdf = (data: IPrescriptionPdfData): Promise<Buffer> =>
	createPdfBuffer(`Prescription - ${data.patientName}`, (pdf) => {
		const { doc, left, contentWidth } = pdf;

		pdf.header("PRESCRIPTION");

		pdf.sectionTitle("Doctor Information");
		pdf.infoRow("Name", `Dr. ${data.doctorName}`);
		pdf.infoRow("Specialization", data.specialization);
		pdf.infoRow("Email", data.doctorEmail);
		doc.moveDown(0.8);

		pdf.sectionTitle("Patient Information");
		pdf.infoRow("Name", data.patientName);
		pdf.infoRow("Email", data.patientEmail);
		doc.moveDown(0.8);

		pdf.sectionTitle("Prescription Details");
		pdf.infoRow("Prescription ID", data.prescriptionId);
		pdf.infoRow("Appointment Date", formatDateTime(data.appointmentDate));
		pdf.infoRow("Issued Date", formatDate(data.issuedDate));
		if (data.followUpDate) {
			pdf.infoRow("Follow-up Date", formatDate(data.followUpDate));
		}
		pdf.divider();

		pdf.sectionTitle("Instructions & Medications");
		doc.font("Helvetica")
			.fontSize(11)
			.fillColor(PDF_COLORS.text)
			.text(data.instructions, left, doc.y, { width: contentWidth, lineGap: 4 });
		pdf.divider();

		pdf.footer([
			"This is an electronically generated prescription and does not require a signature.",
			`For more information, visit: ${envVars.FRONTEND_URL}`,
		]);
	});
```

### `prescription.service.ts`-এর import বদলানো

`formatDate` আর `formatDateTime` এখন `utils/pdf.ts`-এ চলে গেছে, তাই import লাইনটা বদলাও:

```ts
// আগে:
import { formatDate, formatDateTime, generatePrescriptionPdf, type IPrescriptionPdfData } from "./prescription.utils";

// এখন:
import { formatDate, formatDateTime } from "@/app/utils/pdf";
import { generatePrescriptionPdf, type IPrescriptionPdfData } from "./prescription.utils";
```

---

## ধাপ ৩: Invoice PDF — `payment.utils.ts` (নতুন file)

`src/app/module/payment/payment.utils.ts`

```ts
import { envVars } from "@/app/config/env";
import { createPdfBuffer, formatDateTime, PDF_COLORS } from "@/app/utils/pdf";

export interface IInvoicePdfData {
	invoiceId: string;
	transactionId: string;
	paymentDate: Date;
	patientName: string;
	patientEmail: string;
	doctorName: string;
	appointmentDate: Date;
	amount: number;
	currency: string;
}

// 1500, "bdt" → "1500.00 BDT"
export const formatAmount = (amount: number, currency: string): string =>
	`${amount.toFixed(2)} ${currency.toUpperCase()}`;

export const generateInvoicePdf = (data: IInvoicePdfData): Promise<Buffer> =>
	createPdfBuffer(`Invoice - ${data.invoiceId}`, (pdf) => {
		const { doc, left, right } = pdf;
		const amountText = formatAmount(data.amount, data.currency);

		pdf.header("INVOICE");

		pdf.sectionTitle("Invoice Information");
		pdf.infoRow("Invoice ID", data.invoiceId);
		pdf.infoRow("Transaction ID", data.transactionId);
		pdf.infoRow("Payment Date", formatDateTime(data.paymentDate));
		pdf.infoRow("Status", "PAID");
		doc.moveDown(0.8);

		pdf.sectionTitle("Billed To");
		pdf.infoRow("Name", data.patientName);
		pdf.infoRow("Email", data.patientEmail);
		doc.moveDown(0.8);

		pdf.sectionTitle("Appointment Details");
		pdf.infoRow("Doctor", `Dr. ${data.doctorName}`);
		pdf.infoRow("Appointment Date", formatDateTime(data.appointmentDate));
		pdf.divider();

		// ---------- Payment Summary table ----------
		pdf.sectionTitle("Payment Summary");

		const amountColWidth = 150;
		const amountX = right - amountColWidth;

		const tableRow = (label: string, value: string, bold = false) => {
			const y = doc.y;
			doc.font(bold ? "Helvetica-Bold" : "Helvetica")
				.fontSize(bold ? 11 : 10)
				.fillColor(PDF_COLORS.text);
			doc.text(label, left, y);
			doc.text(value, amountX, y, { width: amountColWidth, align: "right" });
			doc.x = left; // ⚠️ না হলে পরের সব লেখা amountX থেকে শুরু হবে
			doc.moveDown(0.4);
		};

		const tableLine = () => {
			doc.moveTo(left, doc.y).lineTo(right, doc.y).lineWidth(0.5).strokeColor(PDF_COLORS.border).stroke();
			doc.moveDown(0.4);
		};

		tableRow("Description", "Amount", true);
		tableLine();
		tableRow(`Consultation Fee (Dr. ${data.doctorName})`, amountText);
		tableLine();
		tableRow("Total Paid", amountText, true);
		pdf.divider();

		pdf.footer([
			"Thank you for choosing PH Doc. This is an electronically generated invoice.",
			"Payment processed securely through Stripe.",
			`For more information, visit: ${envVars.FRONTEND_URL}`,
		]);
	});
```

**দুই column-এর table কীভাবে?** একই `y`-তে দুইবার `text()` — বাম দিকে label (`left`), ডান দিকে amount (`amountX`, `align: "right"`)। তারপর `doc.x = left` দিয়ে cursor আবার বামে ফেরত।

---

## ধাপ ৪: `payment.service.ts` (পুরো file)

`src/app/module/payment/payment.service.ts`

```ts
import { deleteFileFromCloudinary, uploadFileToCloudinary } from "@/app/config/cloudinary.config";
import { stripe } from "@/app/config/stripe.config";
import { prisma } from "@/app/lib/prisma";
import { sendEmail } from "@/app/utils/email";
import { formatDateTime } from "@/app/utils/pdf";
import { AppointmentStatus, PaymentStatus } from "@/generated/prisma/enums";
import type { Stripe } from "stripe";
import { formatAmount, generateInvoicePdf, type IInvoicePdfData } from "./payment.utils";

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

// PDF → upload → invoiceUrl save → email
// payment ততক্ষণে PAID হয়ে গেছে, তাই এখানে কিছু fail করলেও payment-এর ক্ষতি নেই
const processInvoice = async (data: IInvoicePdfData) => {
	const pdfBuffer = await generateInvoicePdf(data);
	const fileName = `invoice-${data.invoiceId}.pdf`;
	const { secure_url: invoiceUrl } = await uploadFileToCloudinary(pdfBuffer, fileName);

	try {
		await prisma.payment.update({
			where: { id: data.invoiceId },
			data: { invoiceUrl },
		});
	} catch (error) {
		// DB-তে save না হলে file রেখে লাভ নেই
		await deleteFileFromCloudinary(invoiceUrl).catch((deleteError: unknown) => {
			console.error("Failed to delete orphan invoice:", deleteError);
		});
		throw error;
	}

	await sendEmail({
		to: data.patientEmail,
		subject: `Payment Confirmation & Invoice - Appointment with Dr. ${data.doctorName}`,
		templateName: "invoice",
		templateData: {
			patientName: data.patientName,
			doctorName: data.doctorName,
			invoiceId: data.invoiceId,
			transactionId: data.transactionId,
			paymentDate: formatDateTime(data.paymentDate),
			appointmentDate: formatDateTime(data.appointmentDate),
			amount: formatAmount(data.amount, data.currency),
			invoiceUrl,
		},
		attachments: [{ filename: fileName, content: pdfBuffer, contentType: "application/pdf" }],
	});
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
		include: {
			payment: true,
			patient: { select: { name: true, email: true } },
			doctor: { select: { name: true } },
			schedule: { select: { startDateTime: true } },
		},
	});

	if (!appointment) {
		console.error(`Appointment ${appointmentId} not found for session ${session.id}`);
		return { message: `Appointment ${appointmentId} not found` };
	}

	// আলাদা const-এ রাখলাম, যাতে নিচের check-এর পরে TypeScript বোঝে payment আর null না
	const payment = appointment.payment;

	// Cancel হয়ে গেছে (cron payment delete করে দিয়েছে), অথবা আগেই paid (দুই tab-এ দুইবার pay)
	const isInvalidPayment =
		appointment.status === AppointmentStatus.CANCELED ||
		!payment ||
		payment.id !== paymentId ||
		payment.status === PaymentStatus.PAID;

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

	// টাকা সত্যিই কাটা গেলে invoice। await করছি না, যাতে Stripe সাথে সাথে 200 পায়।
	// stripeEventId আগেই save হয়েছে → Stripe retry করলেও invoice দুইবার হবে না।
	if (paymentStatus === PaymentStatus.PAID) {
		processInvoice({
			invoiceId: payment.id,
			transactionId: payment.transactionId,
			paymentDate: new Date(),
			patientName: appointment.patient.name,
			patientEmail: appointment.patient.email,
			doctorName: appointment.doctor.name,
			appointmentDate: appointment.schedule.startDateTime,
			amount: payment.amount,
			currency: session.currency ?? "bdt",
		}).catch((error: unknown) => {
			console.error(`Failed to process invoice for payment ${payment.id}:`, error);
		});
	}

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

**Webhook-এ কোন কাজ কখন হয়:**

| কাজ | কখন | কেন |
| --- | --- | --- |
| Appointment + Payment = `PAID` | `await` করে, transaction-এ | এটাই আসল কাজ — দুটো একসাথে হবে বা কোনোটাই না |
| Stripe-কে `200` | সাথে সাথে | দেরি হলে Stripe timeout ধরে retry করে |
| PDF → upload → `invoiceUrl` → email | background-এ (`await` ছাড়া, `.catch` সহ) | fail হলেও payment ঠিক থাকে |
| Invalid payment (canceled / দুইবার pay) | refund | `idempotencyKey` থাকায় দুইবার refund হয় না |

---

## ধাপ ৫: `src/app/utils/email.ts` — template-এর নাম যোগ করা

```ts
type EmailTemplate = "otp" | "reset-password-otp" | "prescription" | "invoice";
```

---

## ধাপ ৬: Email template — `src/app/templates/invoice.ejs` (নতুন file)

```ejs
<!doctype html>
<html>
	<head>
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
	</head>
	<body style="margin: 0; padding: 20px; background: #f5f5f5; font-family: Arial, sans-serif; color: #222">
		<div style="max-width: 600px; margin: 0 auto; background: #fff; padding: 32px; border-radius: 8px">
			<h1 style="margin: 0 0 4px; color: #0b5ed7; text-align: center">✅ Payment Successful</h1>
			<p style="margin: 0 0 24px; color: #666; text-align: center; font-size: 14px">
				PH Doc - Your Health, Our Priority
			</p>

			<p>Dear <strong><%= patientName %></strong>,</p>
			<p>We have received your payment. Your appointment with Dr. <%= doctorName %> is confirmed.</p>

			<table style="width: 100%; margin: 20px 0; font-size: 14px; border-collapse: collapse">
				<tr><td style="padding: 6px 0; color: #666">Invoice ID</td><td><%= invoiceId %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Transaction ID</td><td><%= transactionId %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Payment Date</td><td><%= paymentDate %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Doctor</td><td>Dr. <%= doctorName %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Appointment</td><td><%= appointmentDate %></td></tr>
			</table>

			<div style="background: #e8f4f8; border-left: 4px solid #0b5ed7; padding: 16px; border-radius: 4px; text-align: center">
				<p style="margin: 0; color: #666; font-size: 13px">Total Paid</p>
				<p style="margin: 4px 0 0; font-size: 24px; font-weight: bold; color: #0b5ed7"><%= amount %></p>
			</div>

			<p style="text-align: center; margin: 28px 0 8px">
				<a
					href="<%= invoiceUrl %>"
					style="background: #0b5ed7; color: #fff; padding: 12px 28px; text-decoration: none; border-radius: 6px; font-weight: bold"
					>Download Invoice</a
				>
			</p>
			<p style="text-align: center; font-size: 12px; color: #888">The invoice PDF is also attached to this email.</p>

			<hr style="border: none; border-top: 1px solid #ddd; margin: 24px 0" />
			<p style="font-size: 12px; color: #888; text-align: center">
				This is an automated email from PH Doc. Please do not reply.
			</p>
		</div>
	</body>
</html>
```

---

## 🧪 Test

Webhook local-এ test করতে Stripe CLI চালু রাখো ([নোট ২৮](./28-stripe-payment.md) দেখো):

```shell
stripe listen --forward-to localhost:5000/webhook
```

1. Patient হিসেবে login → appointment book → `paymentUrl` খুলে test card `4242 4242 4242 4242` দিয়ে pay।
2. Terminal-এ webhook `200` দেখাবে — invoice-এর জন্য অপেক্ষা না করেই।
3. কয়েক সেকেন্ড পরে check করো:
   - `pnpm studio` → `payments` table-এ `status = PAID` আর `invoiceUrl` আছে
   - Cloudinary-তে `invoice-<paymentId>.pdf`
   - Patient-এর inbox-এ email + PDF attachment
4. Stripe Dashboard থেকে একই event **Resend** করো → `"already processed. Skipping"` আসবে, দ্বিতীয় invoice বা email যাবে না।
5. Prescription create করে দেখো PDF আগের মতোই আসছে (refactor-এর পরে কিছু ভাঙেনি)।

> Port (`5000`) তোমার `.env`-এর `PORT` অনুযায়ী বদলে নিও।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Invoice fail হলে আবার বানানোর উপায় নেই** — background-এ PDF/upload/email fail করলে শুধু log হয়, আর `stripeEventId` save হয়ে যাওয়ায় Stripe retry করলেও skip হবে। ফলে `invoiceUrl` চিরকাল `null` থাকবে। একটা ছোট route রাখতে পারো যেটা `invoiceUrl` না থাকলে আবার বানায়:
   ```ts
   // PATIENT/ADMIN — GET /payments/:id/invoice
   if (!payment.invoiceUrl && payment.status === PaymentStatus.PAID) {
   	await processInvoice({ ...invoiceData });
   }
   ```

2. **`paymentDate: new Date()`** — invoice-এ webhook পৌঁছানোর সময় বসছে, আসল payment-এর সময় না। সাধারণত কয়েক সেকেন্ডের পার্থক্য, কিন্তু Stripe retry করলে ঘণ্টাও হতে পারে। চাইলে Stripe event-এর সময় ব্যবহার করো: `new Date(event.created * 1000)` (তাহলে `event.created` টা `handleCheckoutSessionCompleted`-এ পাঠাতে হবে)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Serverless-এ background কাজ** — Vercel-এর মতো serverless-এ response পাঠানোর পরে function বন্ধ হয়ে যেতে পারে, তখন `await` ছাড়া চলা `processInvoice` মাঝপথে থেমে যাবে। সাধারণ Node server (Render, Railway, VPS)-এ সমস্যা নেই। Production-এ এজন্য queue (BullMQ) ব্যবহার করা হয়।

4. **`payment_status === "unpaid"` case** — card-এ সাথে সাথে `"paid"` আসে। কিন্তু কিছু delayed payment method-এ `"unpaid"` আসে আর পরে `checkout.session.async_payment_succeeded` event আসে, যেটা এখন handle হয় না। শুধু card নিলে চিন্তা নেই।

5. **Cloudinary-তে PDF delivery** — free account-এ default ভাবে PDF delivery বন্ধ থাকে, তখন `invoiceUrl` খুললে `401` আসে। Cloudinary Settings → Security → "Allow delivery of PDF and ZIP files" চালু করো।

---

## 🔜 পরে যা যোগ হবে

- [ ] Patient-এর payment history-তে "Download Invoice" button
- [ ] `invoiceUrl` না থাকলে আবার invoice বানানোর route
