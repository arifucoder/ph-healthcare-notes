# ClinicOS — Doctor Module (CRUD)

## ১. Constant

**File:** `doctor.constant.ts`

```ts
import { defineListConfig } from "@/app/shared/query/queryBuilder";
import type { Prisma } from "@/generated/prisma/client";

export const doctorListConfig = defineListConfig<
    Prisma.DoctorProfileWhereInput,
    Prisma.DoctorProfileOrderByWithRelationInput
>({
    searchableFields: ["user.name", "user.email", "user.phone", "licenseNumber"],
    filters: {
        isActive: { path: "user.isActive", type: "boolean" },
        // ?specialty=cardiology → doctorSpecialties.some.specialty.slug
        specialty: { path: "doctorSpecialties.some.specialty.slug", type: "string" },
    },
    sortableFields: ["user.name", "consultationFeeCents", "createdAt"],
    defaultSortBy: "createdAt",
    defaultSortOrder: "desc",
});
```

## ২. Validation

**File:** `doctor.validation.ts`

```ts
import { personNameSchema, phoneSchema } from "@/app/shared/commonValidation";
import * as z from "zod";

const updateDoctorSchema = z
    .object({
        name: personNameSchema.optional(),
        phone: phoneSchema.nullable().optional(),
        specialtyIds: z
            .array(z.uuid("Invalid specialty id"))
            .min(1, "Select at least one specialty")
            .max(5, "A doctor can have at most 5 specialties")
            .transform((ids) => [...new Set(ids)])
            .optional(),
        licenseNumber: z.string().trim().toUpperCase().min(3, "License number is too short").max(50).optional(),
        consultationFeeCents: z.number().int("Fee must be in whole cents").min(0).max(10_000_000).optional(),
        roomNumber: z.string().trim().max(20).nullable().optional(),
        bio: z.string().trim().max(1000).nullable().optional(),
    })
    .refine((data) => Object.keys(data).length > 0, {
        error: "Provide at least one field to update",
    });

const updateDoctorStatusSchema = z.object({
    isActive: z.boolean(),
});

export const DoctorValidation = {
    updateDoctorSchema,
    updateDoctorStatusSchema,
};
```

## ৩. Interface

**File:** `doctor.interface.ts`

```ts
import type * as z from "zod";
import type { DoctorValidation } from "./doctor.validation";

export type TUpdateDoctorPayload = z.infer<typeof DoctorValidation.updateDoctorSchema>;
export type TUpdateDoctorStatusPayload = z.infer<typeof DoctorValidation.updateDoctorStatusSchema>;
```

## ৪. Service

**File:** `doctor.service.ts`

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { buildListQuery, buildMeta } from "@/app/shared/query/queryBuilder";
import type { Prisma } from "@/generated/prisma/client";
import httpStatus from "http-status";
import { doctorListConfig } from "./doctor.constant";
import type { TUpdateDoctorPayload, TUpdateDoctorStatusPayload } from "./doctor.interface";

const doctorSelect = {
    id: true,
    licenseNumber: true,
    consultationFeeCents: true,
    roomNumber: true,
    bio: true,
    photoUrl: true,
    isPaused: true,
    createdAt: true,
    updatedAt: true,
    user: {
        select: { id: true, name: true, email: true, phone: true, isActive: true, needPasswordChange: true },
    },
    doctorSpecialties: {
        select: { specialty: { select: { id: true, name: true, slug: true } } },
    },
} satisfies Prisma.DoctorProfileSelect;

type TDoctorRow = Prisma.DoctorProfileGetPayload<{ select: typeof doctorSelect }>;

// join table-এর আকার client-এ যায় না: [{ specialty }] → [specialty]
const toDoctorResponse = ({ doctorSpecialties, ...doctor }: TDoctorRow) => ({
    ...doctor,
    specialties: doctorSpecialties.map(({ specialty }) => specialty),
});

// মুছে ফেলা doctor-এর user-এ isDeleted = true, profile-এ কিছু না
const notDeleted = { user: { isDeleted: false } } satisfies Prisma.DoctorProfileWhereInput;

const findDoctorOrThrow = async (id: string) => {
    const doctor = await prisma.doctorProfile.findFirst({
        where: { id, ...notDeleted },
        select: doctorSelect,
    });

    if (!doctor) {
        throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
    }

    return doctor;
};

const getAllDoctors = async (rawQuery: unknown) => {
    const q = buildListQuery(rawQuery, doctorListConfig, notDeleted);

    const [rows, total] = await Promise.all([
        prisma.doctorProfile.findMany({ ...q.args, select: doctorSelect }),
        prisma.doctorProfile.count({ where: q.where }),
    ]);

    return { meta: buildMeta(total, q.page, q.limit), data: rows.map(toDoctorResponse) };
};

const getDoctorById = async (id: string) => {
    return toDoctorResponse(await findDoctorOrThrow(id));
};

const updateDoctor = async (id: string, payload: TUpdateDoctorPayload) => {
    const { name, phone, specialtyIds, licenseNumber, ...profileData } = payload;

    const doctor = await findDoctorOrThrow(id);

    if (specialtyIds) {
        const validSpecialtyCount = await prisma.specialty.count({
            where: { id: { in: specialtyIds }, isDeleted: false, isActive: true },
        });

        if (validSpecialtyCount !== specialtyIds.length) {
            throw new AppError(httpStatus.BAD_REQUEST, "One or more specialties are not found or inactive");
        }
    }

    if (licenseNumber && licenseNumber !== doctor.licenseNumber) {
        const licenseTaken = await prisma.doctorProfile.findUnique({
            where: { licenseNumber },
            select: { id: true },
        });

        if (licenseTaken) {
            throw new AppError(httpStatus.CONFLICT, "A doctor with this license number already exists");
        }
    }

    const updated = await prisma.$transaction(async (tx) => {
        // name/phone থাকে user-এ
        if (name !== undefined || phone !== undefined) {
            await tx.user.update({
                where: { id: doctor.user.id },
                data: { name, phone },
            });
        }

        // specialty পুরো তালিকা বদলায়: পুরনো সব মুছে নতুনগুলো বসানো
        if (specialtyIds) {
            await tx.doctorSpecialty.deleteMany({ where: { doctorProfileId: id } });
            await tx.doctorSpecialty.createMany({
                data: specialtyIds.map((specialtyId) => ({ doctorProfileId: id, specialtyId })),
            });
        }

        return tx.doctorProfile.update({
            where: { id },
            data: { ...profileData, licenseNumber },
            select: doctorSelect,
        });
    });

    return toDoctorResponse(updated);
};

const updateDoctorStatus = async (id: string, payload: TUpdateDoctorStatusPayload) => {
    const { isActive } = payload;

    const doctor = await findDoctorOrThrow(id);

    const updated = await prisma.$transaction(async (tx) => {
        await tx.user.update({
            where: { id: doctor.user.id },
            data: { isActive },
        });

        // বন্ধ করলে সাথে সাথে সব device থেকে logout
        if (!isActive) {
            await tx.session.deleteMany({ where: { userId: doctor.user.id } });
        }

        return tx.doctorProfile.findUniqueOrThrow({
            where: { id },
            select: doctorSelect,
        });
    });

    return toDoctorResponse(updated);
};

const deleteDoctor = async (id: string) => {
    const doctor = await findDoctorOrThrow(id);

    await prisma.$transaction([
        prisma.user.update({
            where: { id: doctor.user.id },
            data: { isDeleted: true, deletedAt: new Date(), isActive: false },
        }),
        prisma.session.deleteMany({ where: { userId: doctor.user.id } }),
    ]);

    // TODO: ভবিষ্যতের CONFIRMED appointment-এর কী হবে (appointment module এলে)
    // TODO: deletedById + audit log
};

export const DoctorService = {
    getAllDoctors,
    getDoctorById,
    updateDoctor,
    updateDoctorStatus,
    deleteDoctor,
};
```

## ৫. Controller

**File:** `doctor.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { idParamSchema } from "@/app/shared/commonValidation";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { DoctorService } from "./doctor.service";

const getAllDoctors = catchAsync(async (req, res) => {
    const { meta, data } = await DoctorService.getAllDoctors(req.query);

    sendResponse(res, {
        statusCode: httpStatus.OK,
        message: "Doctors retrieved successfully",
        meta,
        data,
    });
});

const getDoctorById = catchAsync(async (req, res) => {
    const { id } = idParamSchema.parse(req.params);
    const result = await DoctorService.getDoctorById(id);

    sendResponse(res, {
        statusCode: httpStatus.OK,
        message: "Doctor retrieved successfully",
        data: result,
    });
});

const updateDoctor = catchAsync(async (req, res) => {
    const { id } = idParamSchema.parse(req.params);
    const result = await DoctorService.updateDoctor(id, req.body);

    sendResponse(res, {
        statusCode: httpStatus.OK,
        message: "Doctor updated successfully",
        data: result,
    });
});

const updateDoctorStatus = catchAsync(async (req, res) => {
    const { id } = idParamSchema.parse(req.params);
    const result = await DoctorService.updateDoctorStatus(id, req.body);

    sendResponse(res, {
        statusCode: httpStatus.OK,
        message: result.user.isActive ? "Doctor activated" : "Doctor deactivated and signed out",
        data: result,
    });
});

const deleteDoctor = catchAsync(async (req, res) => {
    const { id } = idParamSchema.parse(req.params);
    await DoctorService.deleteDoctor(id);

    sendResponse(res, {
        statusCode: httpStatus.OK,
        message: "Doctor deleted successfully",
        data: null,
    });
});

export const DoctorController = {
    getAllDoctors,
    getDoctorById,
    updateDoctor,
    updateDoctorStatus,
    deleteDoctor,
};
```

## ৬. Route

**File:** `doctor.route.ts`

```ts
import { ADMINS, checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Router } from "express";
import { DoctorController } from "./doctor.controller";
import { DoctorValidation } from "./doctor.validation";

const router = Router();

// doctor manage করে SUPER_ADMIN আর ADMIN দুজনেই
router.use(checkAuth(...ADMINS));

router.get("/", DoctorController.getAllDoctors);
router.get("/:id", DoctorController.getDoctorById);

router.patch("/:id", validateRequest(DoctorValidation.updateDoctorSchema), DoctorController.updateDoctor);
router.patch(
    "/:id/status",
    validateRequest(DoctorValidation.updateDoctorStatusSchema),
    DoctorController.updateDoctorStatus,
);

router.delete("/:id", DoctorController.deleteDoctor);

export const DoctorRoutes = router;
```

## ৭. Route register করা

**File:** `routes/index.ts`

```ts
import { DoctorRoutes } from "@/app/module/doctor/doctor.route";

// moduleRoutes-এর ভেতরে
{ path: "/doctors", route: DoctorRoutes },
```

## ৮. Doctor নিজের profile update করা

### ৮.১ Validation

**File:** `doctor.validation.ts`

```ts
// doctor নিজে শুধু এই তিনটা বদলাতে পারে; অন্য field body-তে এলে z.object ফেলে দেয়
const updateMyProfileSchema = z
	.object({
		name: personNameSchema.optional(),
		phone: phoneSchema.nullable().optional(),
		bio: z.string().trim().max(1000).nullable().optional(),
	})
	.refine((data) => Object.keys(data).length > 0, {
		error: "Provide at least one field to update",
	});
```

### ৮.২ Interface

**File:** `doctor.interface.ts`

```ts
export type TUpdateMyProfilePayload = z.infer<typeof DoctorValidation.updateMyProfileSchema>;
```

### ৮.৩ Service

**File:** `doctor.service.ts`

```ts
// id client থেকে না, session-এর userId থেকে
const updateMyProfile = async (userId: string, payload: TUpdateMyProfilePayload) => {
	const { name, phone, bio } = payload;

	const doctor = await prisma.doctorProfile.findFirst({
		where: { userId, ...notDeleted },
		select: { id: true },
	});

	if (!doctor) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
	}

	const updated = await prisma.$transaction(async (tx) => {
		// name/phone থাকে user-এ
		if (name !== undefined || phone !== undefined) {
			await tx.user.update({
				where: { id: userId },
				data: { name, phone },
			});
		}

		// bio থাকে profile-এ; না পাঠালে undefined, Prisma এড়িয়ে যায়
		return tx.doctorProfile.update({
			where: { id: doctor.id },
			data: { bio },
			select: doctorSelect,
		});
	});

	return toDoctorResponse(updated);
};
```

### ৮.৪ Controller

**File:** `doctor.controller.ts`

```ts
const updateMyProfile = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const result = await DoctorService.updateMyProfile(userId, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Profile updated successfully",
		data: result,
	});
});
```

### ৮.৫ Route

**File:** `doctor.route.ts`

```ts
import { ADMINS, checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { DoctorController } from "./doctor.controller";
import { DoctorValidation } from "./doctor.validation";

const router = Router();

// ---------- doctor নিজে (router.use-এর আগে, নাহলে ADMINS guard আটকে দেবে) ----------
router.patch(
    "/me",
    checkAuth(Role.DOCTOR),
    validateRequest(DoctorValidation.updateMyProfileSchema),
    DoctorController.updateMyProfile,
);

// ---------- এর নিচে সব admin-only ----------
router.use(checkAuth(...ADMINS));

router.get("/", DoctorController.getAllDoctors);
// ... বাকি সব যেমন আছে
```