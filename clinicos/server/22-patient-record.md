patient-record module (staff):
```text
POST  /patient-records
GET   /patient-records
GET   /patient-records/:id
PATCH /patient-records/:id
POST  /patient-records/:id/reset-password
PATCH /patient-records/:id/account-status
GET   /patient-records/:id/visits   ⏳ পরে
```

১. patient-record.constant.ts
```ts
import { defineListConfig } from "@/app/shared/query/queryBuilder";
import type { Prisma } from "@/generated/prisma/client";
import { BloodGroup, Gender } from "@/generated/prisma/enums";

export const patientRecordListConfig = defineListConfig<
	Prisma.PatientWhereInput,
	Prisma.PatientOrderByWithRelationInput
>({
	searchableFields: ["fullName", "phone", "user.email"],
	filters: {
		isActive: { path: "user.isActive", type: "boolean" },
		gender: { path: "gender", type: "enum", values: Object.values(Gender) },
		bloodGroup: { path: "bloodGroup", type: "enum", values: Object.values(BloodGroup) },
	},
	sortableFields: ["fullName", "dateOfBirth", "createdAt"],
	defaultSortBy: "createdAt",
	defaultSortOrder: "desc",
});
```

২. patient-record.validation.ts
```ts
import { emailSchema, personNameSchema, phoneSchema } from "@/app/shared/commonValidation";
import { BloodGroup, Gender } from "@/generated/prisma/enums";
import * as z from "zod";

const dateOfBirthSchema = z.iso
	.date("Date of birth must be in YYYY-MM-DD format")
	.refine((value) => new Date(value) <= new Date(), { error: "Date of birth cannot be in the future" })
	.transform((value) => new Date(value));

const addressSchema = z.string().trim().max(300);
const allergiesSchema = z.string().trim().max(500);

const createPatientSchema = z.object({
	fullName: personNameSchema,
	email: emailSchema,
	phone: phoneSchema,
	dateOfBirth: dateOfBirthSchema,
	gender: z.enum(Gender),
	address: addressSchema.optional(),
	bloodGroup: z.enum(BloodGroup).optional(),
	allergies: allergiesSchema.optional(),
	emergencyContactName: personNameSchema.optional(),
	emergencyContactPhone: phoneSchema.optional(),
});

const updatePatientSchema = z
	.object({
		fullName: personNameSchema.optional(),
		phone: phoneSchema.optional(),
		dateOfBirth: dateOfBirthSchema.optional(),
		gender: z.enum(Gender).optional(),
		address: addressSchema.nullable().optional(),
		bloodGroup: z.enum(BloodGroup).nullable().optional(),
		allergies: allergiesSchema.nullable().optional(),
		emergencyContactName: personNameSchema.nullable().optional(),
		emergencyContactPhone: phoneSchema.nullable().optional(),
	})
	.refine((data) => Object.keys(data).length > 0, {
		error: "Provide at least one field to update",
	});

const updateAccountStatusSchema = z.object({
	isActive: z.boolean(),
});

export const PatientRecordValidation = {
	createPatientSchema,
	updatePatientSchema,
	updateAccountStatusSchema,
};
```

৩. patient-record.interface.ts
```ts
import type * as z from "zod";
import type { PatientRecordValidation } from "./patient-record.validation";

export type TCreatePatientPayload = z.infer<typeof PatientRecordValidation.createPatientSchema>;
export type TUpdatePatientPayload = z.infer<typeof PatientRecordValidation.updatePatientSchema>;
export type TUpdateAccountStatusPayload = z.infer<typeof PatientRecordValidation.updateAccountStatusSchema>;
```

৪. patient-record.service.ts
```ts
import { envVars } from "@/app/config/env";
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { buildListQuery, buildMeta } from "@/app/shared/query/queryBuilder";
import { sendEmail } from "@/app/shared/sendEmail";
import type { Prisma } from "@/generated/prisma/client";
import { Role } from "@/generated/prisma/enums";
import { hashPassword } from "better-auth/crypto";
import httpStatus from "http-status";
import { randomBytes } from "node:crypto";
import { patientRecordListConfig } from "./patient-record.constant";
import type {
	TCreatePatientPayload,
	TUpdateAccountStatusPayload,
	TUpdatePatientPayload,
} from "./patient-record.interface";

const patientSelect = {
	id: true,
	fullName: true,
	phone: true,
	dateOfBirth: true,
	gender: true,
	address: true,
	bloodGroup: true,
	allergies: true,
	emergencyContactName: true,
	emergencyContactPhone: true,
	registeredById: true,
	createdAt: true,
	updatedAt: true,
	user: {
		select: { id: true, email: true, isActive: true, needPasswordChange: true },
	},
} satisfies Prisma.PatientSelect;

const notDeleted = { user: { isDeleted: false } } satisfies Prisma.PatientWhereInput;

const generateTemporaryPassword = () => randomBytes(9).toString("base64url");

const findPatientOrThrow = async (id: string) => {
	const patient = await prisma.patient.findFirst({
		where: { id, ...notDeleted },
		select: patientSelect,
	});

	if (!patient) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient not found");
	}

	return patient;
};

const createPatient = async (registeredById: string, payload: TCreatePatientPayload) => {
	const { fullName, email, ...profile } = payload;

	const existingUser = await prisma.user.findUnique({
		where: { email },
		select: { patientProfile: { select: { id: true } } },
	});

	if (existingUser) {
		throw new AppError(
			httpStatus.CONFLICT,
			existingUser.patientProfile ? "A patient with this email already exists" : "Email already in use",
		);
	}

	const temporaryPassword = generateTemporaryPassword();
	const hashedPassword = await hashPassword(temporaryPassword);
	const userId = crypto.randomUUID();

	const patient = await prisma.$transaction(async (tx) => {
		await tx.user.create({
			data: {
				id: userId,
				name: fullName,
				email,
				role: Role.PATIENT,
				emailVerified: true,
				needPasswordChange: true,
			},
		});

		await tx.account.create({
			data: {
				id: crypto.randomUUID(),
				userId,
				accountId: userId,
				providerId: "credential",
				password: hashedPassword,
			},
		});

		// TODO: audit log (audit_logs table এলে, এই tx-এর ভেতরেই)
		return tx.patient.create({
			data: { userId, fullName, registeredById, ...profile },
			select: patientSelect,
		});
	});

	let emailSent = true;

	try {
		await sendEmail({
			to: email,
			subject: `Your ${envVars.CLINIC_NAME} patient account`,
			previewText: "Your account is ready. Sign in with the temporary password inside.",
			template: "staffWelcome",
			data: {
				name: fullName,
				roleLabel: "Patient",
				email,
				temporaryPassword,
				loginUrl: `${envVars.FRONTEND_URL}/login`,
			},
		});
	} catch (error) {
		console.error(`Failed to send welcome email to patient ${patient.id}`, error);
		emailSent = false;
	}

	return { patient, temporaryPassword, emailSent };
};

const getAllPatients = async (rawQuery: unknown) => {
	const q = buildListQuery(rawQuery, patientRecordListConfig, notDeleted);

	const [data, total] = await Promise.all([
		prisma.patient.findMany({ ...q.args, select: patientSelect }),
		prisma.patient.count({ where: q.where }),
	]);

	return { meta: buildMeta(total, q.page, q.limit), data };
};

const getPatientById = async (id: string) => {
	return findPatientOrThrow(id);
};

const updatePatient = async (id: string, payload: TUpdatePatientPayload) => {
	const { fullName, ...profile } = payload;

	const patient = await findPatientOrThrow(id);

	return prisma.$transaction(async (tx) => {
		if (fullName !== undefined) {
			await tx.user.update({
				where: { id: patient.user.id },
				data: { name: fullName },
			});
		}

		// TODO: audit log
		return tx.patient.update({
			where: { id },
			data: { fullName, ...profile },
			select: patientSelect,
		});
	});
};

const resetPassword = async (id: string) => {
	const patient = await findPatientOrThrow(id);
	const userId = patient.user.id;

	const temporaryPassword = generateTemporaryPassword();
	const hashedPassword = await hashPassword(temporaryPassword);

	await prisma.$transaction(async (tx) => {
		const { count } = await tx.account.updateMany({
			where: { userId, providerId: "credential" },
			data: { password: hashedPassword },
		});

		if (count === 0) {
			throw new AppError(httpStatus.BAD_REQUEST, "Password login is not set up for this account");
		}

		await tx.user.update({
			where: { id: userId },
			data: { needPasswordChange: true },
		});

		await tx.session.deleteMany({ where: { userId } });
	});

	let emailSent = true;

	try {
		await sendEmail({
			to: patient.user.email,
			subject: `Your ${envVars.CLINIC_NAME} password has been reset`,
			previewText: "Sign in with the new temporary password inside.",
			template: "temporaryPasswordReset",
			data: {
				name: patient.fullName,
				email: patient.user.email,
				temporaryPassword,
				loginUrl: `${envVars.FRONTEND_URL}/login`,
			},
		});
	} catch (error) {
		console.error(`Failed to send password reset email to patient ${patient.id}`, error);
		emailSent = false;
	}

	return { temporaryPassword, emailSent };
};

const updateAccountStatus = async (id: string, payload: TUpdateAccountStatusPayload) => {
	const { isActive } = payload;

	const patient = await findPatientOrThrow(id);

	return prisma.$transaction(async (tx) => {
		await tx.user.update({
			where: { id: patient.user.id },
			data: { isActive },
		});

		if (!isActive) {
			await tx.session.deleteMany({ where: { userId: patient.user.id } });
		}

		return tx.patient.findUniqueOrThrow({
			where: { id },
			select: patientSelect,
		});
	});
};

export const PatientRecordService = {
	createPatient,
	getAllPatients,
	getPatientById,
	updatePatient,
	resetPassword,
	updateAccountStatus,
};
```

৫. patient-record.controller.ts
```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { idParamSchema } from "@/app/shared/commonValidation";
import { getAuthUser } from "@/app/shared/getAuthUser";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { PatientRecordService } from "./patient-record.service";

const createPatient = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const result = await PatientRecordService.createPatient(userId, req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: result.emailSent
			? "Patient registered. Login details have been sent to their email"
			: "Patient registered, but the welcome email could not be sent. Give them the temporary password shown here",
		data: result,
	});
});

const getAllPatients = catchAsync(async (req, res) => {
	const { meta, data } = await PatientRecordService.getAllPatients(req.query);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Patients retrieved successfully",
		meta,
		data,
	});
});

const getPatientById = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await PatientRecordService.getPatientById(id);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Patient retrieved successfully",
		data: result,
	});
});

const updatePatient = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await PatientRecordService.updatePatient(id, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Patient updated successfully",
		data: result,
	});
});

const resetPassword = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await PatientRecordService.resetPassword(id);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: result.emailSent
			? "Password reset. The new temporary password has been sent to their email"
			: "Password reset, but the email could not be sent. Give them the temporary password shown here",
		data: result,
	});
});

const updateAccountStatus = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await PatientRecordService.updateAccountStatus(id, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: result.user.isActive ? "Patient account activated" : "Patient account deactivated and signed out",
		data: result,
	});
});

export const PatientRecordController = {
	createPatient,
	getAllPatients,
	getPatientById,
	updatePatient,
	resetPassword,
	updateAccountStatus,
};
```

৬. patient-record.route.ts
```ts
import { ADMINS, STAFF, checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Router } from "express";
import { PatientRecordController } from "./patient-record.controller";
import { PatientRecordValidation } from "./patient-record.validation";

const router = Router();

router.post(
	"/",
	checkAuth(...ADMINS),
	validateRequest(PatientRecordValidation.createPatientSchema),
	PatientRecordController.createPatient,
);

router.get("/", checkAuth(...STAFF), PatientRecordController.getAllPatients);
router.get("/:id", checkAuth(...STAFF), PatientRecordController.getPatientById);

router.patch(
	"/:id",
	checkAuth(...ADMINS),
	validateRequest(PatientRecordValidation.updatePatientSchema),
	PatientRecordController.updatePatient,
);
router.post("/:id/reset-password", checkAuth(...ADMINS), PatientRecordController.resetPassword);
router.patch(
	"/:id/account-status",
	checkAuth(...ADMINS),
	validateRequest(PatientRecordValidation.updateAccountStatusSchema),
	PatientRecordController.updateAccountStatus,
);

export const PatientRecordRoutes = router;
```

৭. Module-এর বাইরে আরও তিনটা ছোট কাজ

routes/index.ts: import করে moduleRoutes-এ যোগ করো:
```ts
{ path: "/patient-records", route: PatientRecordRoutes },
```

shared/renderEmail.ts: TEmailTemplates-এ নতুন template-এর type যোগ করো:
```ts
temporaryPasswordReset: {
	name: string;
	email: string;
	temporaryPassword: string;
	loginUrl: string;
};
```