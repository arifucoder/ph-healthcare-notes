patient module (রোগী নিজে):
```text
GET   /patient/me
PATCH /patient/me
GET   /patient/visits            ⏳ পরে
GET   /patient/appointments      ⏳ পরে
GET   /patient/queue/today       ⏳ পরে
```

০. আগে একটা ছোট refactor: dateOfBirthSchema share করো

এই schema এখন দুইটা module-এ লাগবে। তাই এটাকে patient-record.validation.ts থেকে কেটে shared/commonValidation.ts-এ নিয়ে যাও, আর export করে দাও:

```ts
export const dateOfBirthSchema = z.iso
	.date("Date of birth must be in YYYY-MM-DD format")
	.refine((value) => new Date(value) <= new Date(), { error: "Date of birth cannot be in the future" })
	.transform((value) => new Date(value));
```

১. patient.validation.ts
```ts
import { dateOfBirthSchema, personNameSchema, phoneSchema } from "@/app/shared/commonValidation";
import { BloodGroup, Gender } from "@/generated/prisma/enums";
import * as z from "zod";

const updateMyProfileSchema = z
	.object({
		phone: phoneSchema.optional(),
		dateOfBirth: dateOfBirthSchema.optional(),
		gender: z.enum(Gender).optional(),
		address: z.string().trim().max(300).nullable().optional(),
		bloodGroup: z.enum(BloodGroup).nullable().optional(),
		emergencyContactName: personNameSchema.nullable().optional(),
		emergencyContactPhone: phoneSchema.nullable().optional(),
	})
	.refine((data) => Object.keys(data).length > 0, {
		error: "Provide at least one field to update",
	});

export const PatientValidation = {
	updateMyProfileSchema,
};
```

২. patient.interface.ts

```ts
import type * as z from "zod";
import type { PatientValidation } from "./patient.validation";

export type TUpdateMyProfilePayload = z.infer<typeof PatientValidation.updateMyProfileSchema>;
```

৩. patient.service.ts

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import type { Prisma } from "@/generated/prisma/client";
import httpStatus from "http-status";
import type { TUpdateMyProfilePayload } from "./patient.interface";

const myProfileSelect = {
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
	createdAt: true,
	updatedAt: true,
	user: { select: { email: true } },
} satisfies Prisma.PatientSelect;

type TMyProfileRow = Prisma.PatientGetPayload<{ select: typeof myProfileSelect }>;

const notDeleted = { user: { isDeleted: false } } satisfies Prisma.PatientWhereInput;

export const isProfileComplete = (patient: Pick<TMyProfileRow, "phone" | "dateOfBirth" | "gender">) =>
	Boolean(patient.phone && patient.dateOfBirth && patient.gender);

const toMyProfile = ({ user, ...patient }: TMyProfileRow) => ({
	...patient,
	email: user.email,
	profileComplete: isProfileComplete(patient),
});

// visits, appointments, queue — পরের সব endpoint এটা দিয়েই শুরু হবে
const getMyPatientOrThrow = async (userId: string) => {
	const patient = await prisma.patient.findFirst({
		where: { userId, ...notDeleted },
		select: { id: true },
	});

	if (!patient) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
	}

	return patient;
};

const getMyProfile = async (userId: string) => {
	const patient = await prisma.patient.findFirst({
		where: { userId, ...notDeleted },
		select: myProfileSelect,
	});

	if (!patient) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
	}

	return toMyProfile(patient);
};

const updateMyProfile = async (userId: string, payload: TUpdateMyProfilePayload) => {
	const { id } = await getMyPatientOrThrow(userId);

	const updated = await prisma.patient.update({
		where: { id },
		data: payload,
		select: myProfileSelect,
	});

	return toMyProfile(updated);
};

export const PatientService = {
	getMyProfile,
	updateMyProfile,
};
```

৪. patient.controller.ts
```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { getAuthUser } from "@/app/shared/getAuthUser";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { PatientService } from "./patient.service";

const getMyProfile = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const result = await PatientService.getMyProfile(userId);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Profile retrieved successfully",
		data: result,
	});
});

const updateMyProfile = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const result = await PatientService.updateMyProfile(userId, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Profile updated successfully",
		data: result,
	});
});

export const PatientController = {
	getMyProfile,
	updateMyProfile,
};
```

৫. patient.route.ts
```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { PatientController } from "./patient.controller";
import { PatientValidation } from "./patient.validation";

const router = Router();

router.use(checkAuth(Role.PATIENT));

router.get("/me", PatientController.getMyProfile);
router.patch("/me", validateRequest(PatientValidation.updateMyProfileSchema), PatientController.updateMyProfile);

export const PatientRoutes = router;
```

৬. routes/index.ts

```ts
{ path: "/patient", route: PatientRoutes },
```