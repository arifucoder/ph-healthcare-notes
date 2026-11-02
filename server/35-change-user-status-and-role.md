# 35 - User Status ও Role Change (Admin Module)

Admin module-এ দুইটা নতুন route: user-কে **block/unblock** করা (`change-user-status`) আর **ADMIN ↔ SUPER_ADMIN** role বদলানো (`change-user-role`)। কে কার উপর কী করতে পারবে, সেটা service-এ check হয়। বদলানোর পরে user-এর session মুছে দেওয়া হয়, যাতে আবার login করতে হয়।

```
PATCH /admins/change-user-status → checkAuth(SUPER_ADMIN, ADMIN) → validateRequest → changeUserStatus
PATCH /admins/change-user-role   → checkAuth(SUPER_ADMIN)        → validateRequest → changeUserRole
      → নিজের উপর না → target user আছে? → permission check → transaction { user.update → session.deleteMany }
```

> Admin CRUD-এর আগের অংশ আছে [নোট ১৪](./14-hw-admin-crud.md)-এ।

### কে কী করতে পারবে

| কাজ | SUPER_ADMIN | ADMIN |
| --- | --- | --- |
| DOCTOR / PATIENT-এর status বদলানো | ✅ | ✅ |
| ADMIN / SUPER_ADMIN-এর status বদলানো | ✅ | ❌ (`403`) |
| Role বদলানো (ADMIN ↔ SUPER_ADMIN) | ✅ | ❌ (route-এই আটকায়) |
| নিজের status / role বদলানো | ❌ (`400`) | ❌ (`400`) |
| DOCTOR / PATIENT-এর role বদলানো | ❌ (আলাদা profile table আছে) | ❌ |
| Status `DELETED` করা | ❌ (এর জন্য আলাদা delete API) | ❌ |

---

## ধাপ ১: Validation — `admin.validation.ts` (পুরো file)

`src/app/module/admin/admin.validation.ts`

```ts
import { Role, UserStatus } from "@/generated/prisma/enums";
import * as z from "zod";

const updateAdminValidationSchema = z.object({
	admin: z
		.object(
			{
				name: z
					.string("Name must be a string")
					.trim()
					.min(3, "Name must be at least 3 characters")
					.max(30, "Name must be at most 30 characters")
					.optional(),

				profilePhoto: z.url({ protocol: /^https?$/, error: "Profile photo must be a valid http/https URL" }).optional(),

				contactNumber: z
					.string("Contact number must be a string")
					.trim()
					.regex(/^(\+?880|0)1[3-9]\d{8}$/, "Invalid Bangladeshi phone number")
					.optional(),
			},
			"Admin data is required",
		)
		.refine((data) => Object.keys(data).length > 0, "At least one field is required to update"),
});

// DELETED এখানে দেওয়া যাবে না — delete-এর জন্য role-specific delete API আছে
const changeUserStatusValidationSchema = z.object({
	userId: z.string("User id is required").trim().min(1, "User id is required"),
	userStatus: z.enum([UserStatus.ACTIVE, UserStatus.BLOCKED], "Status must be ACTIVE or BLOCKED"),
});

// DOCTOR/PATIENT-এ বদলানো যাবে না — ওদের আলাদা profile table লাগে
const changeUserRoleValidationSchema = z.object({
	userId: z.string("User id is required").trim().min(1, "User id is required"),
	role: z.enum([Role.ADMIN, Role.SUPER_ADMIN], "Role must be ADMIN or SUPER_ADMIN"),
});

export const AdminValidation = {
	updateAdminValidationSchema,
	changeUserStatusValidationSchema,
	changeUserRoleValidationSchema,
};
```

> `z.enum([...])`-এ পুরো enum না দিয়ে শুধু allowed value দেওয়া হয়েছে — তাই `DELETED`, `DOCTOR`, `PATIENT` পাঠালে validation-এই `400` আসবে, service পর্যন্ত যাবে না।

---

## ধাপ ২: Interface — `admin.interface.ts` (পুরো file)

`src/app/module/admin/admin.interface.ts`

```ts
import type * as z from "zod";
import type { AdminValidation } from "./admin.validation";

export type IUpdateAdminPayload = z.infer<typeof AdminValidation.updateAdminValidationSchema>;
export type IChangeUserStatusPayload = z.infer<typeof AdminValidation.changeUserStatusValidationSchema>;
export type IChangeUserRolePayload = z.infer<typeof AdminValidation.changeUserRoleValidationSchema>;
```

> Interface হাতে না লিখে `z.infer` দিয়ে Zod schema থেকে type বানানো হচ্ছে — schema বদলালে type নিজেই বদলে যাবে।

---

## ধাপ ৩: Service — `admin.service.ts`-এ যোগ করা

`src/app/module/admin/admin.service.ts` — `deleteAdmin`-এর নিচে এই অংশটা বসাও।

উপরের import-গুলো এমন থাকতে হবে:

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import type { Prisma } from "@/generated/prisma/client";
import { Role, UserStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import type { IChangeUserRolePayload, IChangeUserStatusPayload, IUpdateAdminPayload } from "./admin.interface";
```

> ✏️ import list নোটে ছিল না (তোমার file-এ আছে)। `Prisma` type import পরে যোগ হয়েছে — `satisfies Prisma.UserSelect`-এর জন্য।

```ts
// response-এ শুধু দরকারি field
const userPublicSelect = {
	id: true,
	name: true,
	email: true,
	role: true,
	status: true,
	updatedAt: true,
} satisfies Prisma.UserSelect; // field-এর নামে typo হলে এখানেই error

// SUPER_ADMIN → নিজে বাদে যে কারো status বদলাতে পারে
// ADMIN → শুধু DOCTOR আর PATIENT-এর status বদলাতে পারে
const changeUserStatus = async (user: IRequestUser, payload: IChangeUserStatusPayload) => {
	const { userId, userStatus } = payload;

	if (userId === user.userId) {
		throw new AppError(httpStatus.BAD_REQUEST, "You cannot change your own status");
	}

	const targetUser = await prisma.user.findUnique({
		where: { id: userId },
		select: { role: true, status: true, isDeleted: true },
	});

	// deleted user-কে আবার ACTIVE করা যাবে না
	if (!targetUser || targetUser.isDeleted || targetUser.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	const isTargetAdminLevel = targetUser.role === Role.ADMIN || targetUser.role === Role.SUPER_ADMIN;

	if (user.role === Role.ADMIN && isTargetAdminLevel) {
		throw new AppError(httpStatus.FORBIDDEN, "Only super admin can change the status of an admin or super admin");
	}

	if (targetUser.status === userStatus) {
		throw new AppError(httpStatus.BAD_REQUEST, `User is already ${userStatus}`);
	}

	return prisma.$transaction(async (tx) => {
		const updatedUser = await tx.user.update({
			where: { id: userId },
			data: { status: userStatus },
			select: userPublicSelect,
		});

		// block হলে সব device থেকে logout
		if (userStatus === UserStatus.BLOCKED) {
			await tx.session.deleteMany({ where: { userId } });
		}

		return updatedUser;
	});
};

// শুধু SUPER_ADMIN, আর শুধু ADMIN ↔ SUPER_ADMIN
const changeUserRole = async (user: IRequestUser, payload: IChangeUserRolePayload) => {
	const { userId, role } = payload;

	if (userId === user.userId) {
		throw new AppError(httpStatus.BAD_REQUEST, "You cannot change your own role");
	}

	const targetUser = await prisma.user.findUnique({
		where: { id: userId },
		select: { role: true, status: true, isDeleted: true },
	});

	if (!targetUser || targetUser.isDeleted || targetUser.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	// DOCTOR/PATIENT-এর আলাদা profile table আছে, তাই role বদলানো যাবে না
	if (targetUser.role === Role.DOCTOR || targetUser.role === Role.PATIENT) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			"Role of doctor or patient cannot be changed. Delete and recreate the user with the new role",
		);
	}

	if (targetUser.role === role) {
		throw new AppError(httpStatus.BAD_REQUEST, `User is already ${role}`);
	}

	return prisma.$transaction(async (tx) => {
		const updatedUser = await tx.user.update({
			where: { id: userId },
			data: { role },
			select: userPublicSelect,
		});

		// পুরনো role-এর JWT যেন আর না চলে → আবার login করতে হবে
		await tx.session.deleteMany({ where: { userId } });

		return updatedUser;
	});
};
```

নিচের export object-টা এমন হবে:

```ts
export const AdminService = {
	getAllAdmins,
	getAdminById,
	updateAdmin,
	deleteAdmin,
	changeUserStatus,
	changeUserRole,
};
```

### Check-গুলোর ক্রম কেন এমন?

| ক্রম | Check | Error |
| --- | --- | --- |
| ১ | নিজের উপর? (DB লাগে না, তাই সবার আগে) | `400` |
| ২ | Target user আছে আর deleted না? | `404` |
| ৩ | Permission (ADMIN → admin-level না / DOCTOR-PATIENT-এর role না) | `403` / `400` |
| ৪ | আগে থেকেই একই status/role? | `400` |
| ৫ | Transaction: update + session মোছা | — |

**Session মুছলে কাজ হয় কেন?** আমাদের `checkAuth` প্রতিটা request-এ DB থেকে session খোঁজে, আর **role ও status DB থেকেই** নেয় (JWT-র পুরনো data-র উপর ভরসা করে না)। তাই session মুছে দিলে পুরনো cookie দিয়ে আর ঢোকা যায় না — আবার login করতে হয়, আর তখন নতুন role নিয়ে নতুন token বানে।

> Block করা user-এর ক্ষেত্রে session না মুছলেও `checkAuth` `status === BLOCKED` দেখে `403` দিত। Session মোছা হচ্ছে যাতে DB-তে ফালতু session পড়ে না থাকে আর unblock করার পরে আবার নতুন করে login করতে হয়।

---

## ধাপ ৪: Controller — `admin.controller.ts`-এ যোগ করা

`deleteAdmin`-এর নিচে বসাও। এই file-এর বাকি function-গুলোর মতোই `req.user` check রাখা হয়েছে:

```ts
const changeUserStatus = catchAsync(async (req: Request, res: Response) => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}

	const result = await AdminService.changeUserStatus(req.user, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "User status changed successfully",
		data: result,
	});
});

const changeUserRole = catchAsync(async (req: Request, res: Response) => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}

	const result = await AdminService.changeUserRole(req.user, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "User role changed successfully",
		data: result,
	});
});

export const AdminController = {
	getAllAdmins,
	getAdminById,
	updateAdmin,
	deleteAdmin,
	changeUserStatus,
	changeUserRole,
};
```

---

## ধাপ ৫: Route — `admin.route.ts` (পুরো file, ক্রম খেয়াল করো)

`src/app/module/admin/admin.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { AdminController } from "./admin.controller";
import { AdminValidation } from "./admin.validation";

const router = Router();

// ⚠️ নির্দিষ্ট path আগে, "/:id" পরে — না হলে "/:id" এগুলোকে গিলে ফেলবে
router.patch(
	"/change-user-status",
	checkAuth(Role.SUPER_ADMIN, Role.ADMIN),
	validateRequest(AdminValidation.changeUserStatusValidationSchema),
	AdminController.changeUserStatus,
);

router.patch(
	"/change-user-role",
	checkAuth(Role.SUPER_ADMIN),
	validateRequest(AdminValidation.changeUserRoleValidationSchema),
	AdminController.changeUserRole,
);

router.get("/", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AdminController.getAllAdmins);

router.get("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AdminController.getAdminById);

// ADMIN, ADMIN-কে update করতে পারবে; SUPER_ADMIN-কে update করা service আটকায়
router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(AdminValidation.updateAdminValidationSchema),
	AdminController.updateAdmin,
);

// ADMIN, ADMIN-কে delete করতে পারবে; SUPER_ADMIN-কে delete করা service আটকায়
router.delete("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AdminController.deleteAdmin);

export const AdminRoutes = router;
```

> `PATCH /change-user-status` আর `PATCH /:id` দুটোই `PATCH`। `/:id` আগে থাকলে Express `"change-user-status"`-কে `id` ধরে `updateAdmin`-এ পাঠিয়ে দিত। তাই নির্দিষ্ট path সবসময় উপরে।

> `routes/index.ts`-এ `AdminRoutes` আগেই `/admins`-এ mount করা আছে — নতুন কিছু লাগবে না।

---

## 🧪 Test (Postman)

| Method | URL | Role | Body |
| --- | --- | --- | --- |
| `PATCH` | `/api/v1/admins/change-user-status` | SUPER_ADMIN / ADMIN | `{ "userId": "<user-id>", "userStatus": "BLOCKED" }` |
| `PATCH` | `/api/v1/admins/change-user-role` | SUPER_ADMIN | `{ "userId": "<user-id>", "role": "SUPER_ADMIN" }` |

> `userId` হলো **User table-এর id** (Doctor/Patient/Admin table-এর id না)।

**যা check করবে:**

| Case | Expected |
| --- | --- |
| Patient-কে `BLOCKED` | `200`, ওই patient-এর পুরনো cookie দিয়ে request → `401` |
| Blocked patient আবার login | `403` "Your account has been blocked." (login-এ status check থাকলে) |
| আবার `ACTIVE` | `200`, এখন login করা যায় |
| একই status আবার পাঠানো | `400` "User is already BLOCKED" |
| ADMIN হয়ে অন্য ADMIN-কে block | `403` |
| নিজের userId পাঠানো | `400` |
| `"userStatus": "DELETED"` | `400` (validation) |
| ADMIN হয়ে `change-user-role` | `403` (checkAuth) |
| Doctor-এর role বদলানো | `400` |
| ADMIN → SUPER_ADMIN | `200`, ওই user-কে আবার login করতে হবে |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

1. **কে কখন কাকে block করল, তার record নেই** — এখন শুধু শেষ অবস্থা থাকে। Real project-এ এজন্য একটা audit log table রাখা হয় (`actorId`, `targetUserId`, `action`, `oldValue`, `newValue`, `createdAt`)। শেখার project-এ দরকার নেই।

2. **REST style** — অনেকে `userId` body-তে না দিয়ে URL-এ রাখে: `PATCH /users/:id/status`, `PATCH /users/:id/role`। তখন route-এর ক্রম নিয়েও চিন্তা থাকে না। এখনকারটাও ঠিক আছে, শুধু অন্য style জেনে রাখো।

3. **শেষ SUPER_ADMIN** — নিজের role/status বদলানো আটকানো আছে, তাই কেউ নিজেকে নামাতে পারবে না, কমপক্ষে একজন SUPER_ADMIN সবসময় থেকে যাবে। এটা ভালো design — পরে "নিজের উপর" check সরালে এই সুরক্ষাও চলে যাবে, খেয়াল রেখো।

### ✅ (Done)

- **`as const` → `satisfies Prisma.UserSelect`** — `userPublicSelect`-এ field-এর নামে typo হলে এখন object-এর জায়গাতেই TypeScript error দেবে, আর type-ও ঠিক থাকবে (ধাপ ৩)।
