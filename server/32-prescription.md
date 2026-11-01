# 32 - Prescription (PDF + Cloudinary + Email)

Appointment-এর পরে doctor patient-কে prescription দেবে। Prescription-এর একটা **PDF** বানানো হবে (`pdfkit`), সেটা **Cloudinary**-তে upload হবে, DB-তে `pdfUrl` সহ save হবে, আর patient-কে PDF attach করে **email** যাবে।

```
Doctor → POST /prescriptions → checkAuth(DOCTOR) → validateRequest → givePrescription
       → validate → uuidv7() দিয়ে ID → PDF বানাও → Cloudinary upload
       → DB-তে একবারে save (pdfUrl সহ) → fail হলে uploaded file মুছে দাও
       → email background-এ (fail হলেও prescription থেকে যাবে)
```

| Rule | কোথায় check হয় |
| --- | --- |
| শুধু নিজের appointment-এ prescription | service (`appointment.doctorId === doctor.id`) |
| Canceled appointment-এ দেওয়া যাবে না | service |
| এক appointment-এ একটাই prescription | service (আগেই check) + DB (`appointmentId @unique`) → `P2002` → 409 |
| Update/Delete শুধু নিজের prescription | `getOwnPrescription` helper |
| দেখতে পারবে: ওই doctor, ওই patient, admin | `getSinglePrescription` |

---

## ধাপ ১: Prisma schema

`prisma/schema/prescription.prisma`

> ✏️ নোটে path ছিল `prisma/prescription.prisma` — আমাদের multi-file schema `prisma/schema/`-এর ভেতরে থাকে।

```prisma
model Prescription {
  id           String    @id @default(uuid(7))
  instructions String    @db.Text
  followUpDate DateTime?
  pdfUrl       String?
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt

  appointmentId String      @unique
  appointment   Appointment @relation(fields: [appointmentId], references: [id], onDelete: Restrict, onUpdate: Cascade)

  patientId String
  patient   Patient @relation(fields: [patientId], references: [id], onDelete: Restrict, onUpdate: Cascade)

  doctorId String
  doctor   Doctor @relation(fields: [doctorId], references: [id], onDelete: Restrict, onUpdate: Cascade)

  @@index([patientId, createdAt], name: "idx_prescription_patientId_createdAt")
  @@index([doctorId, createdAt], name: "idx_prescription_doctorId_createdAt")
  @@map("prescriptions")
}
```

অন্য model-গুলোতে উল্টো দিকের relation:

```prisma
// appointment.prisma
prescription Prescription?

// doctor.prisma
prescriptions Prescription[]

// patient.prisma
prescriptions Prescription[]
```

```shell
pnpm migrate
pnpm generate
```

> ✏️ উল্টো দিকের relation আর migrate/generate command নোটে ছিল না (তোমার schema-তে আছে)।

---

## ধাপ ২: Package install

PDF বানানোর জন্য `pdfkit` লাগবে:

```shell
pnpm add pdfkit
pnpm add -D @types/pdfkit
```

> ID আগে থেকে বানাতে `uuid` package লাগে (`v7`) — project-এ আগেই install করা আছে।

---

## ধাপ ৩: Interface — `prescription.interface.ts`

`src/app/module/prescription/prescription.interface.ts`

```ts
export interface ICreatePrescriptionPayload {
	appointmentId: string;
	instructions: string;
	followUpDate?: Date;
}

export interface IUpdatePrescriptionPayload {
	instructions?: string;
	followUpDate?: Date | null; // null পাঠালে follow-up মুছে যাবে
}
```

---

## ধাপ ৪: Validation — `prescription.validation.ts`

```ts
import * as z from "zod";

const instructionsSchema = z
	.string("Instructions is required")
	.trim()
	.min(1, "Instructions cannot be empty")
	.max(5000, "Instructions cannot be more than 5000 characters");

// "2026-11-20" বা ISO string → Date object (validateRequest req.body-তে Date বসিয়ে দেবে)
const followUpDateSchema = z.coerce
	.date("Follow-up date must be a valid date")
	.refine((date) => date > new Date(), "Follow-up date must be in the future");

const createPrescriptionValidationSchema = z.object({
	appointmentId: z.uuid("A valid appointmentId is required"),
	instructions: instructionsSchema,
	followUpDate: followUpDateSchema.optional(),
});

const updatePrescriptionValidationSchema = z
	.object({
		instructions: instructionsSchema.optional(),
		followUpDate: followUpDateSchema.nullable().optional(),
	})
	.refine((data) => data.instructions !== undefined || data.followUpDate !== undefined, {
		message: "Provide at least instructions or followUpDate to update",
	});

export const PrescriptionValidation = {
	createPrescriptionValidationSchema,
	updatePrescriptionValidationSchema,
};
```

**`followUpDate` তিন অবস্থা (update-এ):**

| পাঠালে | ফল |
| --- | --- |
| field নেই (`undefined`) | আগের follow-up date-ই থাকবে |
| `null` | follow-up date মুছে যাবে |
| `"2026-11-20"` | নতুন date বসবে (ভবিষ্যতের হতে হবে) |

---

## ধাপ ৫: Constant — `prescription.constant.ts`

```ts
import type { Prisma } from "@/generated/prisma/client";

export const prescriptionSearchableFields = ["instructions", "doctor.name", "patient.name"] as const;

export const prescriptionFilterableFields = ["doctorId", "patientId", "appointmentId"] as const;

export const prescriptionSortableFields = ["createdAt", "followUpDate"] as const;

// response-এ শুধু দরকারি field — videoCallingId, contactNumber, address কখনো না
export const prescriptionInclude = {
	doctor: { select: { id: true, name: true, email: true, profilePhoto: true, designation: true } },
	patient: { select: { id: true, name: true, email: true, profilePhoto: true } },
	appointment: {
		select: {
			id: true,
			status: true,
			schedule: { select: { startDateTime: true, endDateTime: true } },
		},
	},
} satisfies Prisma.PrescriptionInclude;

// update-এর সময় নতুন PDF বানাতে যা যা লাগে
export const prescriptionPdfInclude = {
	doctor: {
		select: {
			name: true,
			email: true,
			specialties: { select: { specialty: { select: { title: true } } } },
		},
	},
	patient: { select: { name: true, email: true } },
	appointment: { select: { schedule: { select: { startDateTime: true } } } },
} satisfies Prisma.PrescriptionInclude;
```

> `satisfies` দিলে type check হয়, কিন্তু object-এর exact type হারায় না — তাই `Prisma.PrescriptionGetPayload<{ include: typeof prescriptionInclude }>` দিয়ে সঠিক return type পাওয়া যায়।

---

## ধাপ ৬: PDF বানানো — `prescription.utils.ts`

> 🔄 পরে এই PDF code shared helper `utils/pdf.ts`-এ সরানো হয়েছে — নতুন version দেখো [নোট ৩৩](./33-payment-invoice-pdf.md)-এ।

```ts
import { envVars } from "@/app/config/env";
import PDFDocument from "pdfkit";

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

// server UTC-তে চললেও তারিখ বাংলাদেশের সময়ে দেখাবে
const TIME_ZONE = "Asia/Dhaka";

export const formatDate = (date: Date): string =>
	date.toLocaleDateString("en-GB", { timeZone: TIME_ZONE, day: "2-digit", month: "short", year: "numeric" });

export const formatDateTime = (date: Date): string =>
	date.toLocaleString("en-GB", { timeZone: TIME_ZONE, dateStyle: "medium", timeStyle: "short" });

const COLORS = {
	primary: "#0b5ed7",
	text: "#222222",
	muted: "#666666",
	border: "#cccccc",
};

// pdfkit stream-এ একটু একটু করে data দেয় → সব chunk জোড়া লাগিয়ে একটা Buffer
export const generatePrescriptionPdf = (data: IPrescriptionPdfData): Promise<Buffer> => {
	return new Promise((resolve, reject) => {
		const doc = new PDFDocument({
			size: "A4",
			margin: 50,
			info: { Title: `Prescription - ${data.patientName}`, Author: `Dr. ${data.doctorName}` },
		});

		const chunks: Buffer[] = [];
		doc.on("data", (chunk: Buffer) => {
			chunks.push(chunk);
		});
		doc.on("end", () => resolve(Buffer.concat(chunks)));
		doc.on("error", reject);

		const left = doc.page.margins.left;
		const right = doc.page.width - doc.page.margins.right;
		const contentWidth = right - left;

		// ---------- ছোট helper ----------
		const drawDivider = () => {
			doc.moveDown(0.5);
			doc.moveTo(left, doc.y).lineTo(right, doc.y).lineWidth(1).strokeColor(COLORS.border).stroke();
			doc.moveDown(0.8);
		};

		const sectionTitle = (title: string) => {
			doc.font("Helvetica-Bold").fontSize(12).fillColor(COLORS.primary).text(title);
			doc.moveDown(0.3);
		};

		// "Name: John" — label bold, value normal (continued: true মানে একই লাইনে চলবে)
		const infoRow = (label: string, value: string) => {
			doc.font("Helvetica-Bold").fontSize(10).fillColor(COLORS.text).text(`${label}: `, { continued: true });
			doc.font("Helvetica").fillColor(COLORS.muted).text(value);
		};

		// ---------- Header ----------
		doc.font("Helvetica-Bold").fontSize(22).fillColor(COLORS.primary).text("PRESCRIPTION", { align: "center" });
		doc.font("Helvetica").fontSize(10).fillColor(COLORS.muted).text("PH Doc - Your Health, Our Priority", {
			align: "center",
		});
		drawDivider();

		// ---------- Doctor ----------
		sectionTitle("Doctor Information");
		infoRow("Name", `Dr. ${data.doctorName}`);
		infoRow("Specialization", data.specialization);
		infoRow("Email", data.doctorEmail);
		doc.moveDown(0.8);

		// ---------- Patient ----------
		sectionTitle("Patient Information");
		infoRow("Name", data.patientName);
		infoRow("Email", data.patientEmail);
		doc.moveDown(0.8);

		// ---------- Details ----------
		sectionTitle("Prescription Details");
		infoRow("Prescription ID", data.prescriptionId);
		infoRow("Appointment Date", formatDateTime(data.appointmentDate));
		infoRow("Issued Date", formatDate(data.issuedDate));
		if (data.followUpDate) {
			infoRow("Follow-up Date", formatDate(data.followUpDate));
		}
		drawDivider();

		// ---------- Instructions ----------
		sectionTitle("Instructions & Medications");
		doc.font("Helvetica").fontSize(11).fillColor(COLORS.text).text(data.instructions, {
			width: contentWidth,
			align: "left",
			lineGap: 4,
		});
		drawDivider();

		// ---------- Footer ----------
		doc.font("Helvetica")
			.fontSize(9)
			.fillColor(COLORS.muted)
			.text("This is an electronically generated prescription and does not require a signature.", {
				align: "center",
			})
			.text(`For more information, visit: ${envVars.FRONTEND_URL}`, { align: "center" });

		doc.end();
	});
};
```

> Prescription সবসময় English-এ লেখা হবে — তাই built-in `Helvetica` font-ই যথেষ্ট, আলাদা font লাগবে না।

> PDF disk-এ save হয় না — পুরোটা memory-তে `Buffer` হিসেবে থাকে, তারপর সরাসরি Cloudinary-তে আর email attachment-এ যায়।

---

## ধাপ ৭: Service — `prescription.service.ts`

```ts
import { deleteFileFromCloudinary, uploadFileToCloudinary } from "@/app/config/cloudinary.config";
import AppError from "@/app/errorHelpers/AppError";
import type { IQueryParams, IQueryResult } from "@/app/interfaces/query.interface";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import { sendEmail } from "@/app/utils/email";
import { QueryBuilder } from "@/app/utils/QueryBuilder";
import { Prisma } from "@/generated/prisma/client"; // ✅ value import, কারণ instanceof-এ লাগবে
import { AppointmentStatus, Role } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import { v7 as uuidv7 } from "uuid";
import {
	prescriptionFilterableFields,
	prescriptionInclude,
	prescriptionPdfInclude,
	prescriptionSearchableFields,
	prescriptionSortableFields,
} from "./prescription.constant";
import type { ICreatePrescriptionPayload, IUpdatePrescriptionPayload } from "./prescription.interface";
import { formatDate, formatDateTime, generatePrescriptionPdf, type IPrescriptionPdfData } from "./prescription.utils";

type PrescriptionWithRelations = Prisma.PrescriptionGetPayload<{ include: typeof prescriptionInclude }>;

interface IUploadedPdf {
	pdfBuffer: Buffer;
	fileName: string;
	pdfUrl: string;
}

// ---------------- Helpers ----------------

const getSpecializationText = (specialties: { specialty: { title: string } }[]): string =>
	specialties.map((item) => item.specialty.title).join(", ") || "General Physician";

const getDoctorByEmail = async (email: string) => {
	const doctor = await prisma.doctor.findUnique({
		where: { email },
		include: { specialties: { select: { specialty: { select: { title: true } } } } },
	});

	if (!doctor || doctor.isDeleted) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
	}

	return doctor;
};

// prescription খুঁজে বের করো + এটা এই doctor-এর কিনা check
const getOwnPrescription = async (prescriptionId: string, user: IRequestUser) => {
	const prescription = await prisma.prescription.findUnique({
		where: { id: prescriptionId },
		include: prescriptionPdfInclude,
	});

	if (!prescription) {
		throw new AppError(httpStatus.NOT_FOUND, "Prescription not found");
	}

	if (prescription.doctor.email !== user.email) {
		throw new AppError(httpStatus.FORBIDDEN, "This is not your prescription");
	}

	return prescription;
};

// PDF বানাও → Cloudinary-তে upload
const createAndUploadPdf = async (data: IPrescriptionPdfData): Promise<IUploadedPdf> => {
	const pdfBuffer = await generatePrescriptionPdf(data);
	const fileName = `prescription-${data.prescriptionId}.pdf`;
	const uploaded = await uploadFileToCloudinary(pdfBuffer, fileName);

	return { pdfBuffer, fileName, pdfUrl: uploaded.secure_url };
};

// file মুছতে না পারলে শুধু log — মূল কাজ যেন fail না করে
const safeDeleteFile = async (url: string | null) => {
	if (!url) return;

	try {
		await deleteFileFromCloudinary(url);
	} catch (error) {
		console.error("Failed to delete file from Cloudinary:", error);
	}
};

// await করছি না — email-এর জন্য response আটকে থাকবে না, fail হলেও prescription থাকবে
const sendPrescriptionEmail = (subject: string, data: IPrescriptionPdfData, pdf: IUploadedPdf) => {
	sendEmail({
		to: data.patientEmail,
		subject,
		templateName: "prescription",
		templateData: {
			patientName: data.patientName,
			doctorName: data.doctorName,
			specialization: data.specialization,
			prescriptionId: data.prescriptionId,
			appointmentDate: formatDateTime(data.appointmentDate),
			issuedDate: formatDate(data.issuedDate),
			followUpDate: data.followUpDate ? formatDate(data.followUpDate) : null,
			instructions: data.instructions,
			pdfUrl: pdf.pdfUrl,
		},
		attachments: [{ filename: pdf.fileName, content: pdf.pdfBuffer, contentType: "application/pdf" }],
	}).catch((error: unknown) => {
		console.error("Failed to send prescription email:", error);
	});
};

// ---------------- Services ----------------

const givePrescription = async (user: IRequestUser, payload: ICreatePrescriptionPayload) => {
	const doctor = await getDoctorByEmail(user.email);

	const appointment = await prisma.appointment.findUnique({
		where: { id: payload.appointmentId },
		include: {
			patient: { select: { name: true, email: true } },
			schedule: { select: { startDateTime: true } },
			prescription: { select: { id: true } },
		},
	});

	if (!appointment) {
		throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");
	}

	if (appointment.doctorId !== doctor.id) {
		throw new AppError(httpStatus.FORBIDDEN, "You can only give prescription for your own appointments");
	}

	if (appointment.status === AppointmentStatus.CANCELED) {
		throw new AppError(httpStatus.BAD_REQUEST, "You cannot give prescription for a canceled appointment");
	}

	// PDF বানানোর আগেই check — না হলে অকারণে upload হতো
	if (appointment.prescription) {
		throw new AppError(
			httpStatus.CONFLICT,
			"You have already given prescription for this appointment. You can update the prescription instead.",
		);
	}

	// PDF-এ ID লাগবে, তাই DB-তে save করার আগেই ID বানিয়ে নিচ্ছি
	const pdfData: IPrescriptionPdfData = {
		prescriptionId: uuidv7(),
		doctorName: doctor.name,
		doctorEmail: doctor.email,
		specialization: getSpecializationText(doctor.specialties),
		patientName: appointment.patient.name,
		patientEmail: appointment.patient.email,
		appointmentDate: appointment.schedule.startDateTime,
		issuedDate: new Date(),
		followUpDate: payload.followUpDate ?? null,
		instructions: payload.instructions,
	};

	// ১. PDF বানিয়ে upload (DB transaction-এর বাইরে)
	const pdf = await createAndUploadPdf(pdfData);

	// ২. DB-তে একবারেই save — fail হলে uploaded file মুছে দাও
	const prescription = await prisma.prescription
		.create({
			data: {
				id: pdfData.prescriptionId,
				appointmentId: appointment.id,
				doctorId: appointment.doctorId, // client থেকে না, appointment থেকে
				patientId: appointment.patientId,
				instructions: pdfData.instructions,
				followUpDate: pdfData.followUpDate,
				pdfUrl: pdf.pdfUrl,
				createdAt: pdfData.issuedDate,
			},
			include: prescriptionInclude,
		})
		.catch(async (error: unknown) => {
			await safeDeleteFile(pdf.pdfUrl);

			// একসাথে দুইটা request এলে appointmentId @unique DB-তেই আটকাবে
			if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002") {
				throw new AppError(httpStatus.CONFLICT, "Prescription already exists for this appointment");
			}
			throw error;
		});

	// ৩. email (background)
	sendPrescriptionEmail(`You have received a new prescription from Dr. ${doctor.name}`, pdfData, pdf);

	return prescription;
};

// route-এ শুধু PATIENT আর DOCTOR allowed
const getMyPrescriptions = async (user: IRequestUser) => {
	const where: Prisma.PrescriptionWhereInput =
		user.role === Role.DOCTOR ? { doctor: { email: user.email } } : { patient: { email: user.email } };

	return prisma.prescription.findMany({
		where,
		include: prescriptionInclude,
		orderBy: { createdAt: "desc" },
	});
};

// Admin — ?searchTerm=fever&doctorId=...&sortBy=createdAt&page=1&limit=10
const getAllPrescriptions = async (query: IQueryParams): Promise<IQueryResult<PrescriptionWithRelations>> => {
	return new QueryBuilder<PrescriptionWithRelations, Prisma.PrescriptionWhereInput, Prisma.PrescriptionInclude>(
		prisma.prescription,
		query,
		{
			searchableFields: prescriptionSearchableFields,
			filterableFields: prescriptionFilterableFields,
			sortableFields: prescriptionSortableFields,
		},
	)
		.search()
		.filter()
		.sort()
		.paginate()
		.include(prescriptionInclude)
		.execute();
};

// ওই doctor, ওই patient, অথবা admin দেখতে পারবে
const getSinglePrescription = async (user: IRequestUser, prescriptionId: string) => {
	const prescription = await prisma.prescription.findUnique({
		where: { id: prescriptionId },
		include: prescriptionInclude,
	});

	if (!prescription) {
		throw new AppError(httpStatus.NOT_FOUND, "Prescription not found");
	}

	const isAdmin = user.role === Role.ADMIN || user.role === Role.SUPER_ADMIN;
	const isOwner = prescription.doctor.email === user.email || prescription.patient.email === user.email;

	if (!isAdmin && !isOwner) {
		throw new AppError(httpStatus.FORBIDDEN, "You are not allowed to view this prescription");
	}

	return prescription;
};

const updatePrescription = async (user: IRequestUser, prescriptionId: string, payload: IUpdatePrescriptionPayload) => {
	const oldPrescription = await getOwnPrescription(prescriptionId, user);

	const pdfData: IPrescriptionPdfData = {
		prescriptionId: oldPrescription.id,
		doctorName: oldPrescription.doctor.name,
		doctorEmail: oldPrescription.doctor.email,
		specialization: getSpecializationText(oldPrescription.doctor.specialties),
		patientName: oldPrescription.patient.name,
		patientEmail: oldPrescription.patient.email,
		appointmentDate: oldPrescription.appointment.schedule.startDateTime,
		issuedDate: oldPrescription.createdAt,
		// undefined → আগেরটাই থাকবে, null → follow-up মুছে যাবে
		followUpDate: payload.followUpDate === undefined ? oldPrescription.followUpDate : payload.followUpDate,
		instructions: payload.instructions ?? oldPrescription.instructions,
	};

	// ১. নতুন PDF upload
	const pdf = await createAndUploadPdf(pdfData);

	// ২. DB update — fail হলে নতুন file মুছে দাও, পুরনোটা অক্ষত থাকবে
	const updatedPrescription = await prisma.prescription
		.update({
			where: { id: oldPrescription.id },
			data: {
				instructions: pdfData.instructions,
				followUpDate: pdfData.followUpDate,
				pdfUrl: pdf.pdfUrl,
			},
			include: prescriptionInclude,
		})
		.catch(async (error: unknown) => {
			await safeDeleteFile(pdf.pdfUrl);
			throw error;
		});

	// ৩. DB ঠিকমতো update হওয়ার পরেই পুরনো PDF মুছবো
	await safeDeleteFile(oldPrescription.pdfUrl);

	// ৪. email (background)
	sendPrescriptionEmail(`Your prescription has been updated by Dr. ${oldPrescription.doctor.name}`, pdfData, pdf);

	return updatedPrescription;
};

const deletePrescription = async (user: IRequestUser, prescriptionId: string) => {
	const prescription = await getOwnPrescription(prescriptionId, user);

	const deletedPrescription = await prisma.prescription.delete({
		where: { id: prescription.id },
	});

	// DB থেকে মোছার পরে file — যাতে DB-তে কখনো ভাঙা link না থাকে
	await safeDeleteFile(deletedPrescription.pdfUrl);

	return deletedPrescription;
};

export const PrescriptionService = {
	givePrescription,
	getMyPrescriptions,
	getAllPrescriptions,
	getSinglePrescription,
	updatePrescription,
	deletePrescription,
};
```

**File আর DB-র ক্রম কেন এমন?** (Cloudinary আর DB একসাথে transaction-এ রাখা যায় না)

| কাজ | ক্রম | কিছু fail করলে |
| --- | --- | --- |
| Create | upload → DB create | DB fail → নতুন file মুছে দাও |
| Update | নতুন upload → DB update → পুরনো file মোছো | DB fail → নতুন file মোছো, পুরনোটা ঠিক থাকে |
| Delete | DB delete → file মোছো | file মুছতে fail → শুধু log (DB-তে ভাঙা link থাকে না) |

---

## ধাপ ৮: Controller — `prescription.controller.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { PrescriptionService } from "./prescription.service";

const getAuthUser = (req: Request): IRequestUser => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}
	return req.user;
};

const givePrescription = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const result = await PrescriptionService.givePrescription(user, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.CREATED,
		success: true,
		message: "Prescription created successfully",
		data: result,
	});
});

const getMyPrescriptions = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const result = await PrescriptionService.getMyPrescriptions(user);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Prescriptions retrieved successfully",
		data: result,
	});
});

const getAllPrescriptions = catchAsync(async (req: Request, res: Response) => {
	const result = await PrescriptionService.getAllPrescriptions(req.query);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Prescriptions retrieved successfully",
		meta: result.meta,
		data: result.data,
	});
});

const getSinglePrescription = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const prescriptionId = req.params.id as string;
	const result = await PrescriptionService.getSinglePrescription(user, prescriptionId);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Prescription retrieved successfully",
		data: result,
	});
});

const updatePrescription = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const prescriptionId = req.params.id as string;
	const result = await PrescriptionService.updatePrescription(user, prescriptionId, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Prescription updated successfully",
		data: result,
	});
});

const deletePrescription = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const prescriptionId = req.params.id as string;
	const result = await PrescriptionService.deletePrescription(user, prescriptionId);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Prescription deleted successfully",
		data: result,
	});
});

export const PrescriptionController = {
	givePrescription,
	getMyPrescriptions,
	getAllPrescriptions,
	getSinglePrescription,
	updatePrescription,
	deletePrescription,
};
```

---

## ধাপ ৯: Route — `prescription.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { PrescriptionController } from "./prescription.controller";
import { PrescriptionValidation } from "./prescription.validation";

const router = Router();

router.get("/", checkAuth(Role.SUPER_ADMIN, Role.ADMIN), PrescriptionController.getAllPrescriptions);

// ⚠️ "/my-prescriptions" অবশ্যই "/:id"-এর আগে — না হলে "my-prescriptions"-কে id ধরে নেবে
router.get(
	"/my-prescriptions",
	checkAuth(Role.PATIENT, Role.DOCTOR),
	PrescriptionController.getMyPrescriptions,
);

router.get(
	"/:id",
	checkAuth(Role.SUPER_ADMIN, Role.ADMIN, Role.DOCTOR, Role.PATIENT),
	PrescriptionController.getSinglePrescription,
);

router.post(
	"/",
	checkAuth(Role.DOCTOR),
	validateRequest(PrescriptionValidation.createPrescriptionValidationSchema),
	PrescriptionController.givePrescription,
);

router.patch(
	"/:id",
	checkAuth(Role.DOCTOR),
	validateRequest(PrescriptionValidation.updatePrescriptionValidationSchema),
	PrescriptionController.updatePrescription,
);

router.delete("/:id", checkAuth(Role.DOCTOR), PrescriptionController.deletePrescription);

export const PrescriptionRoutes = router;
```

---

## ধাপ ১০: `routes/index.ts`-এ mount করা

`src/app/routes/index.ts`

```ts
import { PrescriptionRoutes } from "../module/prescription/prescription.route";

router.use("/prescriptions", PrescriptionRoutes);
```

> ✏️ এই ধাপটা নোটে ছিল না (নোটে ৭ এর পরে সরাসরি ৯ ছিল)। এখন তোমার `routes/index.ts`-এ `PrescriptionRoutes` mount করা **নেই** — না করলে সব request `404` দেবে।

---

## ধাপ ১১: `src/app/utils/email.ts` — template-এর নাম যোগ করা

`EmailTemplate` একটা type union, যাতে template-এর নামে typo থাকলে TypeScript ধরে ফেলে। নতুন template-এর নাম এখানে না দিলে service-এ type error আসবে।

```ts
type EmailTemplate = "otp" | "reset-password-otp" | "prescription";
```

> `sendEmail`-এ `attachments?: Attachment[]` option থাকতে হবে — PDF attach করার জন্য (তোমার `email.ts`-এ আছে)।

---

## ধাপ ১২: Email template — `src/app/templates/prescription.ejs`

```ejs
<!doctype html>
<html>
	<head>
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
	</head>
	<body style="margin: 0; padding: 20px; background: #f5f5f5; font-family: Arial, sans-serif; color: #222">
		<div style="max-width: 600px; margin: 0 auto; background: #fff; padding: 32px; border-radius: 8px">
			<h1 style="margin: 0 0 4px; color: #0b5ed7; text-align: center">📋 Prescription</h1>
			<p style="margin: 0 0 24px; color: #666; text-align: center; font-size: 14px">
				PH Doc - Your Health, Our Priority
			</p>

			<p>Dear <strong><%= patientName %></strong>,</p>
			<p>Dr. <%= doctorName %> has issued a prescription for you. Details are below.</p>

			<table style="width: 100%; margin: 20px 0; font-size: 14px; border-collapse: collapse">
				<tr><td style="padding: 6px 0; color: #666">Doctor</td><td>Dr. <%= doctorName %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Specialization</td><td><%= specialization %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Prescription ID</td><td><%= prescriptionId %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Appointment</td><td><%= appointmentDate %></td></tr>
				<tr><td style="padding: 6px 0; color: #666">Issued</td><td><%= issuedDate %></td></tr>
				<% if (followUpDate) { %>
				<tr><td style="padding: 6px 0; color: #666">Follow-up</td><td><strong><%= followUpDate %></strong></td></tr>
				<% } %>
			</table>

			<div style="background: #e8f4f8; border-left: 4px solid #17a2b8; padding: 16px; border-radius: 4px">
				<p style="margin: 0 0 8px; font-weight: bold; color: #17a2b8">💊 Instructions & Medications</p>
				<div style="white-space: pre-wrap; font-size: 14px; line-height: 1.7"><%= instructions %></div>
			</div>

			<p style="text-align: center; margin: 28px 0 8px">
				<a
					href="<%= pdfUrl %>"
					style="background: #0b5ed7; color: #fff; padding: 12px 28px; text-decoration: none; border-radius: 6px; font-weight: bold"
					>Download PDF</a
				>
			</p>
			<p style="text-align: center; font-size: 12px; color: #888">The PDF is also attached to this email.</p>

			<hr style="border: none; border-top: 1px solid #ddd; margin: 24px 0" />
			<p style="font-size: 12px; color: #888; text-align: center">
				This is an automated email from PH Doc. Please do not reply.
			</p>
		</div>
	</body>
</html>
```

> `<%= %>` HTML escape করে — তাই instructions-এ কেউ `<script>` লিখলেও email-এ চলবে না। `white-space: pre-wrap` দিয়ে `\n` নতুন লাইন হিসেবে দেখায়।

---

## 🧪 Test (Postman)

| Method | URL | Role | Body |
| --- | --- | --- | --- |
| `POST` | `/api/v1/prescriptions` | DOCTOR | নিচের JSON |
| `GET` | `/api/v1/prescriptions/my-prescriptions` | DOCTOR / PATIENT | — |
| `GET` | `/api/v1/prescriptions?searchTerm=napa&page=1&limit=10` | ADMIN / SUPER_ADMIN | — |
| `GET` | `/api/v1/prescriptions/:id` | ওই doctor / ওই patient / admin | — |
| `PATCH` | `/api/v1/prescriptions/:id` | DOCTOR (নিজের) | `{ "instructions": "...", "followUpDate": null }` |
| `DELETE` | `/api/v1/prescriptions/:id` | DOCTOR (নিজের) | — |

Create body (doctor হিসেবে login করে):

```json
{
	"appointmentId": "<appointment-uuid>",
	"instructions": "Napa 500mg - 1+0+1 after meal, 5 days\nDrink plenty of water",
	"followUpDate": "2026-11-20"
}
```

**যা check করবে:**
- Response-এ `pdfUrl` আছে, link খুললে PDF দেখা যায়
- Patient-এর inbox-এ email + PDF attachment এসেছে
- একই appointment-এ আবার POST → `409 Conflict`
- অন্য doctor-এর prescription PATCH/DELETE → `403 Forbidden`
- Update-এর পরে Cloudinary-তে পুরনো PDF মুছে গেছে

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **`routes/index.ts`-এ mount করো** (ধাপ ১০) — এখন mount করা নেই, তাই সব endpoint `404` দেবে।

### 🟡 ভালো practice

2. **কোন appointment status-এ prescription দেওয়া যাবে** — এখন শুধু `CANCELED` আটকানো হচ্ছে, মানে ভবিষ্যতের `SCHEDULED` appointment-এও prescription দেওয়া যায়। চাইলে শুধু `INPROGRESS`/`COMPLETED`-এ allow করো:
   ```ts
   const allowed: AppointmentStatus[] = [AppointmentStatus.INPROGRESS, AppointmentStatus.COMPLETED];
   if (!allowed.includes(appointment.status)) {
   	throw new AppError(httpStatus.BAD_REQUEST, "Prescription can be given only during or after the appointment");
   }
   ```
   (তোমার enum-এর exact নাম মিলিয়ে নিও।)

3. **`getMyPrescriptions`-এ pagination নেই** — অনেক prescription হলে সব একবারে আসবে। Admin-এর মতো `QueryBuilder` দিয়ে `where`-টা base filter হিসেবে দিতে পারো, অথবা অন্তত `take`/`skip` যোগ করো।

4. **Follow-up date-এর বাংলাদেশ সময়** — `"2026-11-20"` UTC midnight হিসেবে parse হয় (বাংলাদেশে সকাল ৬টা)। আজকের তারিখ পাঠালে কখনো pass, কখনো fail করতে পারে। দরকার হলে "কমপক্ষে আগামীকাল" rule দাও।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **Cloudinary-তে PDF delivery** — free account-এ default ভাবে PDF/ZIP delivery বন্ধ থাকে, তখন `pdfUrl` খুললে `401` আসে। Cloudinary Settings → Security → "Allow delivery of PDF and ZIP files" চালু করতে হয়।

6. **Email শুধু একটা process-এর ভেতরে background-এ** — server restart হলে পাঠানো না হওয়া email হারিয়ে যাবে। Production-এ এজন্য queue (যেমন BullMQ) ব্যবহার করা হয়।

7. **Update-এ একই সাথে দুই request** — দুটো PATCH একসাথে এলে একটার নতুন PDF Cloudinary-তে অনাথ থেকে যেতে পারে। শেখার project-এ সমস্যা না, শুধু জেনে রাখো।

---

## 🔜 পরে যা যোগ হবে

- [ ] Frontend-এ patient-এর "My Prescriptions" page + PDF download
- [ ] Doctor dashboard থেকে prescription লেখা/edit করা
