# ClinicOS — Admin Module (CRUD)

সব file `src/app/module/admin/`-এ।

## ১. Constant

**File:** `admin.constant.ts`

```ts
import { defineListConfig } from "@/app/shared/query/queryBuilder";
import type { Prisma } from "@/generated/prisma/client";

export const adminListConfig = defineListConfig<Prisma.UserWhereInput, Prisma.UserOrderByWithRelationInput>({
	searchableFields: ["name", "email", "phone"],
	filters: {
		isActive: { path: "isActive", type: "boolean" },
	},
	sortableFields: ["name", "createdAt"],
	defaultSortBy: "createdAt",
	defaultSortOrder: "desc",
});
```

## ২. Validation

**File:** `admin.validation.ts`

```ts
import { personNameSchema, phoneSchema } from "@/app/shared/commonValidation";
import * as z from "zod";

const updateAdminSchema = z
	.object({
		name: personNameSchema.optional(),
		// null পাঠালে phone মুছে যায়
		phone: phoneSchema.nullable().optional(),
	})
	.refine((data) => Object.keys(data).length > 0, {
		error: "Provide at least one field to update",
	});

const updateAdminStatusSchema = z.object({
	isActive: z.boolean(),
});

export const AdminValidation = {
	updateAdminSchema,
	updateAdminStatusSchema,
};
```

## ৩. Interface

**File:** `admin.interface.ts`

```ts
import type * as z from "zod";
import type { AdminValidation } from "./admin.validation";

export type TUpdateAdminPayload = z.infer<typeof AdminValidation.updateAdminSchema>;
export type TUpdateAdminStatusPayload = z.infer<typeof AdminValidation.updateAdminStatusSchema>;
```

## ৪. Service

**File:** `admin.service.ts`

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { buildListQuery, buildMeta } from "@/app/shared/query/queryBuilder";
import { Role } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import { adminListConfig } from "./admin.constant";
import type { TUpdateAdminPayload, TUpdateAdminStatusPayload } from "./admin.interface";

const adminSelect = {
	id: true,
	name: true,
	email: true,
	phone: true,
	role: true,
	isActive: true,
	needPasswordChange: true,
	createdAt: true,
	updatedAt: true,
} as const;

// সব কাজের আগে: id-টা কি সত্যিই একজন (মুছে না ফেলা) ADMIN?
const findAdminOrThrow = async (id: string) => {
	const admin = await prisma.user.findFirst({
		where: { id, role: Role.ADMIN, isDeleted: false },
		select: adminSelect,
	});

	if (!admin) {
		throw new AppError(httpStatus.NOT_FOUND, "Admin not found");
	}

	return admin;
};

const getAllAdmins = async (rawQuery: unknown) => {
	const q = buildListQuery(rawQuery, adminListConfig, { role: Role.ADMIN, isDeleted: false });

	const [data, total] = await Promise.all([
		prisma.user.findMany({ ...q.args, select: adminSelect }),
		prisma.user.count({ where: q.where }),
	]);

	return { meta: buildMeta(total, q.page, q.limit), data };
};

const getAdminById = async (id: string) => {
	return findAdminOrThrow(id);
};

const updateAdmin = async (id: string, payload: TUpdateAdminPayload) => {
	await findAdminOrThrow(id);

	return prisma.user.update({
		where: { id },
		data: payload,
		select: adminSelect,
	});
};

const updateAdminStatus = async (id: string, payload: TUpdateAdminStatusPayload) => {
	const { isActive } = payload;

	await findAdminOrThrow(id);

	return prisma.$transaction(async (tx) => {
		const admin = await tx.user.update({
			where: { id },
			data: { isActive },
			select: adminSelect,
		});

		// বন্ধ করলে সাথে সাথে সব device থেকে logout
		if (!isActive) {
			await tx.session.deleteMany({ where: { userId: id } });
		}

		return admin;
	});
};

const deleteAdmin = async (id: string) => {
	await findAdminOrThrow(id);

	await prisma.$transaction([
		prisma.user.update({
			where: { id },
			data: { isDeleted: true, deletedAt: new Date(), isActive: false },
		}),
		prisma.session.deleteMany({ where: { userId: id } }),
	]);

	// TODO: deletedById + audit log (audit_logs table এলে)
};

export const AdminService = {
	getAllAdmins,
	getAdminById,
	updateAdmin,
	updateAdminStatus,
	deleteAdmin,
};
```

## ৫. Controller

**File:** `admin.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { idParamSchema } from "@/app/shared/commonValidation";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { AdminService } from "./admin.service";

const getAllAdmins = catchAsync(async (req, res) => {
	const { meta, data } = await AdminService.getAllAdmins(req.query);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Admins retrieved successfully",
		meta,
		data,
	});
});

const getAdminById = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AdminService.getAdminById(id);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Admin retrieved successfully",
		data: result,
	});
});

const updateAdmin = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AdminService.updateAdmin(id, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Admin updated successfully",
		data: result,
	});
});

const updateAdminStatus = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await AdminService.updateAdminStatus(id, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: result.isActive ? "Admin activated" : "Admin deactivated and signed out",
		data: result,
	});
});

const deleteAdmin = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	await AdminService.deleteAdmin(id);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Admin deleted successfully",
		data: null,
	});
});

export const AdminController = {
	getAllAdmins,
	getAdminById,
	updateAdmin,
	updateAdminStatus,
	deleteAdmin,
};
```

## ৬. Route

**File:** `admin.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { AdminController } from "./admin.controller";
import { AdminValidation } from "./admin.validation";

const router = Router();

// এই router-এর সব route শুধু super admin
router.use(checkAuth(Role.SUPER_ADMIN));

router.get("/", AdminController.getAllAdmins);
router.get("/:id", AdminController.getAdminById);

router.patch("/:id", validateRequest(AdminValidation.updateAdminSchema), AdminController.updateAdmin);
router.patch(
	"/:id/status",
	validateRequest(AdminValidation.updateAdminStatusSchema),
	AdminController.updateAdminStatus,
);

router.delete("/:id", AdminController.deleteAdmin);

export const AdminRoutes = router;
```

## ৭. Route register করা

**File:** `routes/index.ts`

```ts
import { AdminRoutes } from "@/app/module/admin/admin.route";

// moduleRoutes-এর ভেতরে
{ path: "/admins", route: AdminRoutes },
```