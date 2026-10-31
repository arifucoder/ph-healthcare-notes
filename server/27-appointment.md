# 27 - Appointment (slot book করা)

```
Patient → POST /appointments/book-appointment { doctorId, scheduleId }
→ patient/doctor/slot যাচাই → transaction: slot isBooked=true + Appointment + Payment(UNPAID)
→ (Pay Now হলে) Stripe payment link ফেরত   ← Payment অংশ নোট ২৮-এ
```

> Appointment আর Payment একে অপরের সাথে যুক্ত, তাই দুই নোটে ভাগ করা হয়েছে: এই নোটে **appointment**, [নোট ২৮](./28-stripe-payment.md)-এ **Stripe payment + webhook + cron**।

---

## ধাপ ১: Package install

| Package | কাজ |
|---|---|
| [`stripe`](https://www.npmjs.com/package/stripe) | online payment (নোট ২৮) |
| [`uuid`](https://www.npmjs.com/package/uuid) | `videoCallingId`, `transactionId` বানানো |
| [`node-cron`](https://www.npmjs.com/package/node-cron) | টাকা না দেওয়া (unpaid) appointment নির্দিষ্ট সময় পর নিজে থেকে cancel করা |

```shell
pnpm add stripe uuid node-cron
```

---

## ধাপ ২: Appointment schema

```prisma
model Appointment {
  id             String            @id @default(uuid(7))
  videoCallingId String            @unique @db.Uuid()
  status         AppointmentStatus @default(SCHEDULED)
  paymentStatus  PaymentStatus     @default(UNPAID)
  createdAt      DateTime          @default(now())
  updatedAt      DateTime          @updatedAt

  patientId String
  patient   Patient @relation(fields: [patientId], references: [id], onDelete: Restrict)

  doctorId String
  doctor   Doctor @relation(fields: [doctorId], references: [id], onDelete: Restrict)

  scheduleId String
  schedule   Schedule @relation(fields: [scheduleId], references: [id], onDelete: Restrict)

  review       Review?
  prescription Prescription?
  payment      Payment?

  @@index([doctorId], name: "idx_appointment_doctorId")
  @@index([patientId], name: "idx_appointment_patientId")
  @@index([scheduleId], name: "idx_appointment_scheduleId")
  @@index([status], name: "idx_appointment_status")
  @@map("appointments")
}
```

- `onDelete: Restrict` → appointment থাকলে patient/doctor/schedule মোছা যাবে না (রোগীর record হারাবে না)।
- `paymentStatus` appointment-এও রাখা — cron শুধু এটা দেখেই unpaid খুঁজে পায়।

---

## ধাপ ৩: Payment schema

```prisma
model Payment {
  id                 String        @id @default(uuid(7))
  amount             Float
  transactionId      String        @unique @db.Uuid()
  stripeEventId      String?       @unique
  status             PaymentStatus @default(UNPAID)
  paymentGatewayData Json?
  createdAt          DateTime      @default(now())
  updatedAt          DateTime      @updatedAt

  appointmentId String      @unique
  appointment   Appointment @relation(fields: [appointmentId], references: [id], onDelete: Restrict)

  @@index([status], name: "idx_payment_status")
  @@map("payments")
}
```

| Field | কেন |
|---|---|
| `appointmentId @unique` | একটা appointment-এর একটাই payment (one-to-one) |
| `stripeEventId @unique` | Stripe একই event দুবার পাঠালে দ্বিতীয়বার skip (নোট ২৮) |
| `paymentGatewayData Json?` | Stripe-এর session/payment info রেখে দেওয়া |

> ✏️ Enum (`enums.prisma`-তে আগেই আছে) আর migration — এটা নোটে ছিল না:

```prisma
enum AppointmentStatus {
  SCHEDULED
  INPROGRESS
  COMPLETED
  CANCELED
}

enum PaymentStatus {
  PAID
  UNPAID
}
```

```shell
pnpm migrate
pnpm generate
```

---

## ধাপ ৪: `appointment.validation.ts` (Claude-এর সাহায্যে)

```ts
// src/app/module/appointment/appointment.validation.ts
import { AppointmentStatus } from "@/generated/prisma/enums";
import * as z from "zod";

const bookAppointmentValidationSchema = z.object({
	doctorId: z.uuid("A valid doctorId is required"),
	scheduleId: z.uuid("A valid scheduleId is required"),
});

const changeAppointmentStatusValidationSchema = z.object({
	status: z.enum(AppointmentStatus, "Status must be SCHEDULED, INPROGRESS, COMPLETED or CANCELED"),
});

export const AppointmentValidation = {
	bookAppointmentValidationSchema,
	changeAppointmentStatusValidationSchema,
};
```

- `z.enum(AppointmentStatus)` → Prisma enum object সরাসরি দেওয়া যায় (Zod v4)।

---

## ধাপ ৫: `appointment.route.ts` (teacher-এর route follow করে)

```ts
// src/app/module/appointment/appointment.route.ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { AppointmentController } from "./appointment.controller";
import { AppointmentValidation } from "./appointment.validation";

const router = Router();

router.post(
	"/book-appointment",
	checkAuth(Role.PATIENT),
	validateRequest(AppointmentValidation.bookAppointmentValidationSchema),
	AppointmentController.bookAppointment,
);
// role অনুযায়ী user-এর নিজের appointment দেখানো হবে
router.get("/my-appointments", checkAuth(Role.PATIENT, Role.DOCTOR), AppointmentController.getMyAppointments);

router.patch(
	"/change-appointment-status/:id",
	checkAuth(Role.PATIENT, Role.DOCTOR, Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(AppointmentValidation.changeAppointmentStatusValidationSchema),
	AppointmentController.changeAppointmentStatus,
);
router.get(
	"/my-single-appointment/:id",
	checkAuth(Role.PATIENT, Role.DOCTOR),
	AppointmentController.getMySingleAppointment,
);
router.get("/all-appointments", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AppointmentController.getAllAppointments);
router.post(
	"/book-appointment-with-pay-later",
	checkAuth(Role.PATIENT),
	validateRequest(AppointmentValidation.bookAppointmentValidationSchema),
	AppointmentController.bookAppointmentWithPayLater,
);
router.post("/initiate-payment/:id", checkAuth(Role.PATIENT), AppointmentController.initiatePayment);

export const AppointmentRoutes = router;
```

> ✏️ নোটে file-এর নাম `appointment.routes.ts` লেখা ছিল — project-এ (আর `routes/index.ts`-এর import-এ) `appointment.route.ts`।

| Method | URL | কে | কাজ |
|---|---|---|---|
| `POST` | `/book-appointment` | PATIENT | book + সাথে সাথে payment link (Pay Now) |
| `POST` | `/book-appointment-with-pay-later` | PATIENT | শুধু book, টাকা পরে |
| `POST` | `/initiate-payment/:id` | PATIENT | পরে টাকা দেওয়ার link |
| `GET` | `/my-appointments` | PATIENT, DOCTOR | নিজের সব |
| `GET` | `/my-single-appointment/:id` | PATIENT, DOCTOR | নিজের একটা |
| `GET` | `/all-appointments` | ADMIN, SUPER_ADMIN | সবার |
| `PATCH` | `/change-appointment-status/:id` | সবাই (নিয়ম মেনে) | status বদলানো |

---

## ধাপ ৬: `appointment.controller.ts`

```ts
// src/app/module/appointment/appointment.controller.ts
import type { Request, Response } from "express";
import httpStatus from "http-status";
import AppError from "../../errorHelpers/AppError";
import type { IRequestUser } from "../../interfaces/requestUser.interface";
import { catchAsync } from "../../shared/catchAsync";
import { sendResponse } from "../../shared/sendResponse";
import { AppointmentService } from "./appointment.service";

// checkAuth সবসময় req.user set করে, কিন্তু TypeScript সেটা জানে না।
// তাই এখানে একবার check করে নিশ্চিত IRequestUser ফেরত দেওয়া হচ্ছে।
const getAuthUser = (req: Request): IRequestUser => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}
	return req.user;
};

const bookAppointment = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const appointment = await AppointmentService.bookAppointment(req.body, user);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.CREATED,
		message: "Appointment booked successfully",
		data: appointment,
	});
});

const getMyAppointments = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const appointments = await AppointmentService.getMyAppointments(user);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Appointments retrieved successfully",
		data: appointments,
	});
});

const changeAppointmentStatus = catchAsync(async (req: Request, res: Response) => {
	const appointmentId = req.params.id as string;
	const { status } = req.body; // ✅ পুরো body না, শুধু status
	const user = getAuthUser(req);

	const updatedAppointment = await AppointmentService.changeAppointmentStatus(appointmentId, status, user);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Appointment status updated successfully",
		data: updatedAppointment,
	});
});

const getMySingleAppointment = catchAsync(async (req: Request, res: Response) => {
	const appointmentId = req.params.id as string;
	const user = getAuthUser(req);

	const appointment = await AppointmentService.getMySingleAppointment(appointmentId, user);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Appointment retrieved successfully",
		data: appointment,
	});
});

const getAllAppointments = catchAsync(async (_req: Request, res: Response) => {
	const appointments = await AppointmentService.getAllAppointments();

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "All appointments retrieved successfully",
		data: appointments,
	});
});

const bookAppointmentWithPayLater = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const appointment = await AppointmentService.bookAppointmentWithPayLater(req.body, user);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.CREATED,
		message: "Appointment booked successfully with Pay Later option",
		data: appointment,
	});
});

const initiatePayment = catchAsync(async (req: Request, res: Response) => {
	const appointmentId = req.params.id as string;
	const user = getAuthUser(req);

	const paymentInfo = await AppointmentService.initiatePayment(appointmentId, user);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Payment initiated successfully",
		data: paymentInfo,
	});
});

export const AppointmentController = {
	bookAppointment,
	getMyAppointments,
	changeAppointmentStatus,
	getMySingleAppointment,
	getAllAppointments,
	bookAppointmentWithPayLater,
	initiatePayment,
};
```

> ✏️ নোটে উপরের path `src/app/controllers/...` লেখা ছিল — আসল file `src/app/module/appointment/appointment.controller.ts` (relative import `../../` এখান থেকেই মেলে)।

---

## ধাপ ৭: `routes/index.ts`-এ mount

```ts
// src/app/routes/index.ts
import { AppointmentRoutes } from "../module/appointment/appointment.route";
// ...
router.use("/appointments", AppointmentRoutes);
```

---

## ধাপ ৮: `appointment.interface.ts`

```ts
// src/app/module/appointment/appointment.interface.ts
import type { AppointmentStatus } from "@/generated/prisma/enums";

export interface IBookAppointmentPayload {
	doctorId: string;
	scheduleId: string;
}

export interface IChangeAppointmentStatusPayload {
	status: AppointmentStatus;
}
```

---

## ধাপ ৯: `appointment.service.ts`

> `stripe.config.ts` আর `createCheckoutSession`-এর Stripe অংশ [নোট ২৮](./28-stripe-payment.md)-এ বোঝানো আছে।

```ts
import { envVars } from "@/app/config/env";
import { stripe } from "@/app/config/stripe.config";
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import { AppointmentStatus, PaymentStatus, Role } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import { v7 as uuidv7 } from "uuid";
import type { IBookAppointmentPayload } from "./appointment.interface";

const UNPAID_APPOINTMENT_TIMEOUT_MINUTES = 30;

// ---------------- Helpers ----------------

// Appointment + slot booking + payment, সব একটা DB transaction-এ।
// এখানে কোনো বাইরের API call নেই।
const createAppointmentWithPayment = async (payload: IBookAppointmentPayload, user: IRequestUser) => {
	const patientData = await prisma.patient.findUnique({
		where: { email: user.email },
	});

	if (!patientData || patientData.isDeleted) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
	}

	const doctorData = await prisma.doctor.findUnique({
		where: { id: payload.doctorId, isDeleted: false },
	});

	if (!doctorData) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	}

	const doctorSchedule = await prisma.doctorSchedules.findUnique({
		where: {
			doctorId_scheduleId: {
				doctorId: payload.doctorId,
				scheduleId: payload.scheduleId,
			},
		},
		include: { schedule: true },
	});

	if (!doctorSchedule) {
		throw new AppError(httpStatus.NOT_FOUND, "This doctor is not available on the selected schedule");
	}

	if (doctorSchedule.schedule.startDateTime <= new Date()) {
		throw new AppError(httpStatus.BAD_REQUEST, "You cannot book a schedule that has already started");
	}

	const { appointment, payment } = await prisma.$transaction(async (tx) => {
		// Atomic booking: slot ফাঁকা থাকলেই শুধু isBooked = true হবে।
		// দুইজন একসাথে চেষ্টা করলে একজনের count হবে 0।
		const { count } = await tx.doctorSchedules.updateMany({
			where: {
				doctorId: payload.doctorId,
				scheduleId: payload.scheduleId,
				isBooked: false,
			},
			data: { isBooked: true },
		});

		if (count === 0) {
			throw new AppError(httpStatus.CONFLICT, "This schedule is already booked");
		}

		const newAppointment = await tx.appointment.create({
			data: {
				doctorId: payload.doctorId,
				patientId: patientData.id,
				scheduleId: payload.scheduleId,
				videoCallingId: uuidv7(),
			},
		});

		const newPayment = await tx.payment.create({
			data: {
				appointmentId: newAppointment.id,
				amount: doctorData.appointmentFee,
				transactionId: uuidv7(),
			},
		});

		return { appointment: newAppointment, payment: newPayment };
	});

	return { appointment, payment, doctor: doctorData };
};

interface ICheckoutSessionParams {
	appointmentId: string;
	paymentId: string;
	doctorName: string;
	amount: number;
}

const createCheckoutSession = async ({ appointmentId, paymentId, doctorName, amount }: ICheckoutSessionParams) => {
	const session = await stripe.checkout.sessions.create({
		allowed_payment_method_types: ["card"],
		mode: "payment",
		line_items: [
			{
				price_data: {
					currency: "bdt",
					product_data: {
						name: `Appointment with Dr. ${doctorName}`,
					},
					// Float * 100 থেকে 1998.9999-এর মতো মান আসতে পারে, Stripe চায় integer
					unit_amount: Math.round(amount * 100),
				},
				quantity: 1,
			},
		],
		metadata: { appointmentId, paymentId },
		success_url: `${envVars.FRONTEND_URL}/dashboard/payment/payment-success?appointment_id=${appointmentId}&payment_id=${paymentId}`,
		cancel_url: `${envVars.FRONTEND_URL}/dashboard/appointments?error=payment_cancelled`,
	});

	if (!session.url) {
		throw new AppError(httpStatus.INTERNAL_SERVER_ERROR, "Failed to create payment session");
	}

	return session.url;
};

type TStatusTransitions = Partial<Record<AppointmentStatus, AppointmentStatus[]>>;

// Rule 2: Doctor শুধু এই পরিবর্তনগুলো করতে পারবে
const DOCTOR_ALLOWED_TRANSITIONS: TStatusTransitions = {
	[AppointmentStatus.SCHEDULED]: [AppointmentStatus.INPROGRESS, AppointmentStatus.CANCELED],
	[AppointmentStatus.INPROGRESS]: [AppointmentStatus.COMPLETED],
};

// Rule 3: Patient শুধু scheduled appointment cancel করতে পারবে
const PATIENT_ALLOWED_TRANSITIONS: TStatusTransitions = {
	[AppointmentStatus.SCHEDULED]: [AppointmentStatus.CANCELED],
};

const assertAllowedTransition = (rules: TStatusTransitions, from: AppointmentStatus, to: AppointmentStatus) => {
	if (!rules[from]?.includes(to)) {
		throw new AppError(httpStatus.BAD_REQUEST, `You cannot change appointment status from ${from} to ${to}`);
	}
};

// ---------------- Services ----------------

// Pay Now
const bookAppointment = async (payload: IBookAppointmentPayload, user: IRequestUser) => {
	const { appointment, payment, doctor } = await createAppointmentWithPayment(payload, user);

	// Stripe call transaction-এর বাইরে।
	// এটা fail করলেও appointment UNPAID অবস্থায় থাকবে। Patient পরে initiatePayment দিয়ে
	// আবার চেষ্টা করতে পারবে, না করলে cron ৩০ মিনিট পর cancel করে দেবে।
	const paymentUrl = await createCheckoutSession({
		appointmentId: appointment.id,
		paymentId: payment.id,
		doctorName: doctor.name,
		amount: doctor.appointmentFee,
	});

	return { appointment, payment, paymentUrl };
};

// Pay Later
const bookAppointmentWithPayLater = async (payload: IBookAppointmentPayload, user: IRequestUser) => {
	const { appointment, payment } = await createAppointmentWithPayment(payload, user);
	return { appointment, payment };
};

const getMyAppointments = async (user: IRequestUser) => {
	if (user.role === Role.PATIENT) {
		return prisma.appointment.findMany({
			where: { patient: { email: user.email } },
			include: { doctor: true, schedule: true },
			orderBy: { createdAt: "desc" },
		});
	}

	if (user.role === Role.DOCTOR) {
		return prisma.appointment.findMany({
			where: { doctor: { email: user.email } },
			include: { patient: true, schedule: true },
			orderBy: { createdAt: "desc" },
		});
	}

	throw new AppError(httpStatus.FORBIDDEN, "Only patients and doctors can view their appointments");
};

const getMySingleAppointment = async (appointmentId: string, user: IRequestUser) => {
	if (user.role !== Role.PATIENT && user.role !== Role.DOCTOR) {
		throw new AppError(httpStatus.FORBIDDEN, "Only patients and doctors can view their appointments");
	}

	const appointment =
		user.role === Role.PATIENT
			? await prisma.appointment.findFirst({
					where: { id: appointmentId, patient: { email: user.email } },
					include: { doctor: true, schedule: true },
				})
			: await prisma.appointment.findFirst({
					where: { id: appointmentId, doctor: { email: user.email } },
					include: { patient: true, schedule: true },
				});

	if (!appointment) {
		throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");
	}

	return appointment;
};

// TODO: query builder যোগ করা
const getAllAppointments = async () => {
	return prisma.appointment.findMany({
		include: { doctor: true, patient: true, schedule: true },
	});
};

const changeAppointmentStatus = async (appointmentId: string, newStatus: AppointmentStatus, user: IRequestUser) => {
	const appointmentData = await prisma.appointment.findUnique({
		where: { id: appointmentId },
		include: { doctor: true, patient: true },
	});

	if (!appointmentData) {
		throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");
	}

	// Rule 1: Completed বা Canceled হলে আর কেউ বদলাতে পারবে না
	if (appointmentData.status === AppointmentStatus.COMPLETED || appointmentData.status === AppointmentStatus.CANCELED) {
		throw new AppError(httpStatus.BAD_REQUEST, `Appointment is already ${appointmentData.status}`);
	}

	if (appointmentData.status === newStatus) {
		throw new AppError(httpStatus.BAD_REQUEST, `Appointment is already ${newStatus}`);
	}

	if (user.role === Role.DOCTOR) {
		if (appointmentData.doctor.email !== user.email) {
			throw new AppError(httpStatus.FORBIDDEN, "This is not your appointment");
		}
		assertAllowedTransition(DOCTOR_ALLOWED_TRANSITIONS, appointmentData.status, newStatus);
	} else if (user.role === Role.PATIENT) {
		if (appointmentData.patient.email !== user.email) {
			throw new AppError(httpStatus.FORBIDDEN, "This is not your appointment");
		}
		assertAllowedTransition(PATIENT_ALLOWED_TRANSITIONS, appointmentData.status, newStatus);
	}
	// Rule 4: ADMIN / SUPER_ADMIN যেকোনো status দিতে পারবে

	return prisma.$transaction(async (tx) => {
		const updatedAppointment = await tx.appointment.update({
			where: { id: appointmentId },
			data: { status: newStatus },
		});

		// Cancel হলে slot আবার ফাঁকা করে দাও
		if (newStatus === AppointmentStatus.CANCELED) {
			await tx.doctorSchedules.updateMany({
				where: {
					doctorId: appointmentData.doctorId,
					scheduleId: appointmentData.scheduleId,
				},
				data: { isBooked: false },
			});
		}

		return updatedAppointment;
	});
};

const initiatePayment = async (appointmentId: string, user: IRequestUser) => {
	const appointmentData = await prisma.appointment.findFirst({
		where: { id: appointmentId, patient: { email: user.email } },
		include: { doctor: true, payment: true },
	});

	if (!appointmentData) {
		throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");
	}

	// Canceled check আগে, কারণ cron cancel করার সময় payment delete করে দেয়
	if (appointmentData.status === AppointmentStatus.CANCELED) {
		throw new AppError(httpStatus.BAD_REQUEST, "Appointment is canceled");
	}

	if (!appointmentData.payment) {
		throw new AppError(httpStatus.NOT_FOUND, "Payment data not found for this appointment");
	}

	if (appointmentData.payment.status === PaymentStatus.PAID) {
		throw new AppError(httpStatus.BAD_REQUEST, "Payment already completed for this appointment");
	}

	const paymentUrl = await createCheckoutSession({
		appointmentId: appointmentData.id,
		paymentId: appointmentData.payment.id,
		doctorName: appointmentData.doctor.name,
		amount: appointmentData.doctor.appointmentFee,
	});

	return { paymentUrl };
};

const cancelUnpaidAppointments = async () => {
	const cutoffTime = new Date(Date.now() - UNPAID_APPOINTMENT_TIMEOUT_MINUTES * 60 * 1000);

	const unpaidAppointments = await prisma.appointment.findMany({
		where: {
			status: AppointmentStatus.SCHEDULED, // ✅ Completed বা Canceled ছোঁয়া হবে না
			paymentStatus: PaymentStatus.UNPAID,
			createdAt: { lte: cutoffTime },
		},
		select: { id: true, doctorId: true, scheduleId: true },
	});

	if (unpaidAppointments.length === 0) {
		return { canceledCount: 0 };
	}

	const appointmentIds = unpaidAppointments.map((appointment) => appointment.id);

	await prisma.$transaction(async (tx) => {
		await tx.appointment.updateMany({
			where: { id: { in: appointmentIds } },
			data: { status: AppointmentStatus.CANCELED },
		});

		await tx.payment.deleteMany({
			where: { appointmentId: { in: appointmentIds } },
		});

		for (const appointment of unpaidAppointments) {
			await tx.doctorSchedules.updateMany({
				where: { doctorId: appointment.doctorId, scheduleId: appointment.scheduleId },
				data: { isBooked: false },
			});
		}
	});

	return { canceledCount: appointmentIds.length };
};

export const AppointmentService = {
	bookAppointment,
	getMyAppointments,
	changeAppointmentStatus,
	getMySingleAppointment,
	getAllAppointments,
	bookAppointmentWithPayLater,
	initiatePayment,
	cancelUnpaidAppointments,
};
```

---

# 📘 বুঝে নাও — Appointment কেন আর কীভাবে

## কেন?

Doctor আগে নিজের slot নিয়েছে ([নোট ২৬](./26-doctor-schedule.md))। এখন রোগী সেই slot থেকে একটা **book** করবে। Book মানে তিনটা কাজ একসাথে:

1. Slot আটকানো (`isBooked: true`) — অন্য কেউ যেন না নিতে পারে
2. `Appointment` row তৈরি (কে, কোন doctor, কোন সময়)
3. `Payment` row তৈরি (`UNPAID`, টাকা = doctor-এর `appointmentFee`)

## Booking-এর ধাপ (`createAppointmentWithPayment`)

```
১. patient আছে? (login করা email দিয়ে)        → না হলে 404
২. doctor আছে, deleted না?                      → না হলে 404
৩. এই doctor কি এই slot নিয়েছে?                → না হলে 404
৪. slot কি ভবিষ্যতের?                          → না হলে 400
৫. transaction শুরু:
   slot ফাঁকা থাকলে isBooked=true (updateMany)   → count 0 হলে 409 (অন্য কেউ নিয়ে ফেলেছে)
   Appointment তৈরি (videoCallingId = uuid)
   Payment তৈরি (UNPAID, transactionId = uuid)
```

- **কেন transaction?** মাঝপথে কিছু fail করলে (যেমন payment তৈরি) slot আটকানো আর appointment দুটোই বাতিল — অর্ধেক কাজ DB-তে থাকে না।
- **কেন `updateMany` + `isBooked: false` শর্ত?** দুই রোগী একই মুহূর্তে book করলে, DB একটাকেই update করতে দেয়। অন্যজনের `count: 0` → `409`। আগে `findUnique` করে `isBooked` দেখে তারপর update করলে দুজনেই "ফাঁকা" দেখে ফেলতে পারত।
- **কেন Stripe transaction-এর বাইরে?** Stripe হলো বাইরের internet call — ধীর, fail-ও করতে পারে। Transaction-এর ভেতরে রাখলে DB lock অনেকক্ষণ ধরে থাকত।

## Pay Now বনাম Pay Later

| | `/book-appointment` | `/book-appointment-with-pay-later` |
|---|---|---|
| DB-তে যা হয় | একই (appointment + UNPAID payment) | একই |
| Response | `paymentUrl` সহ | `paymentUrl` ছাড়া |
| টাকা দেবে | এখনই link-এ গিয়ে | পরে `/initiate-payment/:id` দিয়ে link নিয়ে |

## Status-এর নিয়ম (`changeAppointmentStatus`)

```
SCHEDULED ──► INPROGRESS ──► COMPLETED
    │
    └──► CANCELED
```

| কে | কী করতে পারে |
|---|---|
| DOCTOR (নিজের) | SCHEDULED → INPROGRESS / CANCELED, INPROGRESS → COMPLETED |
| PATIENT (নিজের) | শুধু SCHEDULED → CANCELED |
| ADMIN / SUPER_ADMIN | যেকোনো (তবে COMPLETED/CANCELED থেকে আর না) |

- Cancel হলে একই transaction-এ slot আবার `isBooked: false` → অন্য রোগী নিতে পারবে।
- `TStatusTransitions` object-এ নিয়ম লেখা থাকায় অনেক `if/else` লাগে না — নতুন নিয়ম শুধু object-এ যোগ।

---

## Test (Postman)

Base: `http://localhost:5000/api/v1/appointments`

### প্রস্তুতি

1. ADMIN: ভবিষ্যতের schedule বানাও ([নোট ২৫](./25-schedule.md))।
2. DOCTOR: সেই slot নাও ([নোট ২৬](./26-doctor-schedule.md)) → `doctorId` আর `scheduleId` কপি করো।
3. PATIENT login (email verify করা থাকতে হবে)।

### ১. Pay Later book — `POST /book-appointment-with-pay-later` (PATIENT)

```json
{
	"doctorId": "<doctor-id>",
	"scheduleId": "<schedule-id>"
}
```

→ `201`, `appointment.status: SCHEDULED`, `paymentStatus: UNPAID`, `payment.amount` = doctor-এর fee। Prisma Studio-তে `doctor_schedules.isBooked = true`।

| Case | ফল |
|---|---|
| একই body আবার | `409` This schedule is already booked |
| doctor এই slot নেয়নি | `404` This doctor is not available on the selected schedule |
| `"doctorId": "abc"` | `400` A valid doctorId is required |
| অতীতের slot | `400` You cannot book a schedule that has already started |
| DOCTOR দিয়ে login করে | `403` |

### ২. Pay Now book — `POST /book-appointment` (PATIENT)

অন্য একটা ফাঁকা slot দিয়ে একই body → `201`, response-এ `paymentUrl` (Stripe-এর link)। টাকা দেওয়ার test [নোট ২৮](./28-stripe-payment.md)-এ।

### ৩. নিজের appointment

```
GET http://localhost:5000/api/v1/appointments/my-appointments
GET http://localhost:5000/api/v1/appointments/my-single-appointment/<appointment-id>
```

| Login | ফল |
|---|---|
| PATIENT | নিজের appointment, সাথে `doctor` + `schedule` |
| DOCTOR | তার কাছে আসা appointment, সাথে `patient` + `schedule` |
| অন্যের appointment id | `404` |

### ৪. সবার — `GET /all-appointments` (ADMIN)

→ `200`, সবার appointment।

### ৫. Status বদলানো — `PATCH /change-appointment-status/:id`

```json
{ "status": "INPROGRESS" }
```

| Login | Body | ফল |
|---|---|---|
| DOCTOR | `INPROGRESS` | `200` |
| DOCTOR | `COMPLETED` (INPROGRESS থেকে) | `200` |
| DOCTOR | `SCHEDULED` (COMPLETED থেকে) | `400` Appointment is already COMPLETED |
| PATIENT | `CANCELED` (SCHEDULED থেকে) | `200` + slot আবার ফাঁকা |
| PATIENT | `INPROGRESS` | `400` You cannot change appointment status from SCHEDULED to INPROGRESS |
| অন্য doctor | যেকোনো | `403` This is not your appointment |
| যেকোনো | `"status": "DONE"` | `400` (Zod) |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Pay Later-ও ৩০ মিনিট পর cancel হয়ে যায়।** Cron শুধু `UNPAID` + ৩০ মিনিটের পুরনো দেখে — Pay Now আর Pay Later আলাদা করে না। তাই "Pay Later" আসলে "৩০ মিনিটের মধ্যে pay করো"। যদি সত্যিকারের পরে (যেমন appointment-এর আগ পর্যন্ত) pay করার সুযোগ দিতে চাও, appointment-এ একটা field রেখে cron-এ বাদ দাও (schema বদলাতে হবে, untested):

	```prisma
	isPayLater Boolean @default(false)
	```

	```ts
	where: { status: AppointmentStatus.SCHEDULED, paymentStatus: PaymentStatus.UNPAID, isPayLater: false, createdAt: { lte: cutoffTime } },
	```

2. **Paid appointment cancel হলে refund নেই।** Patient/doctor cancel করলে slot ফাঁকা হয়, কিন্তু `payment.status` `PAID`-ই থাকে, টাকা ফেরত যায় না। পরে cancel-এর সময় `stripe.refunds.create` (নোট ২৮-এর `refundCheckoutSession`-এর মতো) যোগ করো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`getAllAppointments`-এ pagination নেই** (code-এ TODO আছে) — appointment বাড়লে [নোট ২৪](./24-query-builder.md)-এর `QueryBuilder` লাগাও।
4. **`IChangeAppointmentStatusPayload` এখনো ব্যবহার হয়নি** — controller-এ `req.body.status` সরাসরি নেওয়া। চাইলে `const { status }: IChangeAppointmentStatusPayload = req.body;`

## 🔜 পরে যা যোগ হবে

- [ ] `getAllAppointments`-এ QueryBuilder
- [ ] Cancel-এ refund
- [ ] Review, Prescription (schema-তে relation আছে)
- [ ] `videoCallingId` দিয়ে video call
