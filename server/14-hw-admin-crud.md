# 14 - HW: Admin CRUD

> 🏠 Home work — Doctor CRUD (note 10)-এর মতো করে Admin-এর create / get / update / soft delete

```
Create → user module (POST /user/create-admin, শুধু SUPER_ADMIN)
বাকি  → admin module (GET / GET :id / PATCH :id / DELETE :id, ADMIN + SUPER_ADMIN)
```

| কাজ | Method | URL | কে পারবে |
|---|---|---|---|
| Admin create | `POST` | `/api/v1/user/create-admin` | SUPER_ADMIN |
| সব admin | `GET` | `/api/v1/admins` | ADMIN, SUPER_ADMIN |
| একজন admin | `GET` | `/api/v1/admins/:id` | ADMIN, SUPER_ADMIN |
| Update | `PATCH` | `/api/v1/admins/:id` | ADMIN, SUPER_ADMIN (SUPER_ADMIN-কে শুধু SUPER_ADMIN) |
| Soft delete | `DELETE` | `/api/v1/admins/:id` | ADMIN, SUPER_ADMIN (SUPER_ADMIN-কে শুধু SUPER_ADMIN, নিজেকে না) |

---

## Part 1: Create Admin (user module)

### ধাপ ১: `prisma/schema/admin.prisma`

```prisma
model Admin {
  id            String    @id @default(uuid(7))
  name          String
  email         String    @unique
  profilePhoto  String?
  contactNumber String
  isDeleted     Boolean   @default(false)
  deletedAt     DateTime?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  // relations
  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([isDeleted], name: "idx_admin_isDeleted")
  @@map("admin")
}
```

- `userId @unique` → একজন user-এর একটাই admin profile (one-to-one)।
- `onDelete: Cascade` → user delete হলে admin-ও delete।

> ✏️ এটা নোটে ছিল না — `User` model-এ (`auth.prisma`) back-relation লাগবে, নইলে migrate error দেবে:
> ```prisma
> // auth.prisma → model User { ... }
> admin Admin?
> ```
> তারপর migrate আর generate:
> ```shell
> pnpm migrate && pnpm generate
> ```

### ধাপ ২: `user.interface.ts`

```ts
export interface ICreateAdminPayload {
	password: string;
	admin: {
		name: string;
		email: string;
		profilePhoto?: string;
		contactNumber: string;
	};
}
```

### ধাপ ৩: `user.validation.ts`

```ts
const createAdminValidationSchema = z.object({
	password: z
		.string("Password is required")
		.min(6, "Password must be at least 6 characters")
		.max(32, "Password must be at most 32 characters"),

	admin: z.object({
		name: z
			.string("Name is required")
			.trim()
			.min(3, "Name must be at least 3 characters")
			.max(30, "Name must be at most 30 characters"),

		email: z.email("Invalid email address").toLowerCase(),

		profilePhoto: z.url({ protocol: /^https?$/, error: "Profile photo must be a valid http/https URL" }).optional(),

		contactNumber: z
			.string("Contact number is required")
			.trim()
			.regex(/^(\+?880|0)1[3-9]\d{8}$/, "Invalid Bangladeshi phone number"),
	}),
});
```

- `z.url({ protocol: /^https?$/ })` → শুধু `http` / `https` URL মানবে (`ftp://`, `javascript:` ইত্যাদি না)।

> ✏️ এটা নোটে ছিল না — route-এ `UserValidation.createAdminValidationSchema` ব্যবহার হয়েছে, তাই file-এর শেষে একটা export object লাগবে:
> ```ts
> export const UserValidation = {
> 	createDoctorZodSchema,
> 	createAdminValidationSchema,
> };
> ```
> (note 11-এ `createDoctorZodSchema` সরাসরি `export const` ছিল। Object-এ রাখলে create-doctor route-এও `UserValidation.createDoctorZodSchema` লিখতে হবে, নইলে আগের মতো আলাদা import রাখো।)

### ধাপ ৪: `user.service.ts` — `createAdmin`

> ✏️ import নোটে ছিল না: `createDoctor`-এর import-গুলোর সাথে `AppError`, `httpStatus`, আর `ICreateAdminPayload` লাগবে।
> ```ts
> import AppError from "@/app/errorHelpers/AppError";
> import httpStatus from "http-status";
> import type { ICreateAdminPayload, ICreateDoctorPayload } from "./user.interface";
> ```

```ts
const createAdmin = async (payload: ICreateAdminPayload) => {
	const { admin, password } = payload;

	const userExists = await prisma.user.findUnique({
		where: { email: admin.email },
	});

	if (userExists) {
		throw new AppError(httpStatus.CONFLICT, "User with this email already exists");
	}

	// Better Auth-এর User table-এ শুধু তার নিজের field-গুলো পাঠানো হচ্ছে
	const userData = await auth.api.signUpEmail({
		body: {
			name: admin.name,
			email: admin.email,
			password,
			image: admin.profilePhoto,
			role: Role.ADMIN, // client থেকে না, সবসময় hardcode
			needPasswordChange: true,
		},
	});

	try {
		const adminData = await prisma.admin.create({
			data: {
				userId: userData.user.id,
				name: admin.name,
				email: admin.email,
				profilePhoto: admin.profilePhoto,
				contactNumber: admin.contactNumber,
			},
		});

		// signUpEmail auto sign-in করে session বানায়, নতুন admin-এর এটা লাগবে না
		await prisma.session.deleteMany({
			where: { userId: userData.user.id },
		});

		return adminData;
	} catch (error) {
		console.error("Error creating admin:", error);
		await prisma.user.delete({
			where: { id: userData.user.id },
		});
		throw error;
	}
};
```

**Flow:**

```
email আছে? → 409 CONFLICT
signUpEmail (role: ADMIN, needPasswordChange: true) → user + account + session
admin create → session delete → ✅
কোথাও fail → user delete (Cascade-এ admin/account/session-ও মুছে যায়) → error
```

- `role: Role.ADMIN` hardcode → client body-তে `role: "SUPER_ADMIN"` পাঠিয়ে নিজেকে বড় বানাতে পারবে না।
- `needPasswordChange: true` → super admin যে password দিয়েছে, admin প্রথম login-এ সেটা বদলাবে।
- `session.deleteMany` → super admin অন্যের account বানাচ্ছে, তাই নতুন admin-এর নামে auto তৈরি session রাখার মানে নেই।

### ধাপ ৫: `user.controller.ts`

```ts
const createAdmin = catchAsync(async (req: Request, res: Response) => {
	const result = await UserService.createAdmin(req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.CREATED,
		success: true,
		message: "Admin created successfully",
		data: result,
	});
});
```

> ✏️ `UserService` আর `UserController` export object-এ `createAdmin` যোগ করতে হবে (নোটে ছিল না):
> ```ts
> export const UserService = { createDoctor, createAdmin };
> export const UserController = { createDoctor, createAdmin };
> ```

### ধাপ ৬: `user.route.ts`

```ts
router.post(
	"/create-admin",
	checkAuth(Role.SUPER_ADMIN),
	validateRequest(UserValidation.createAdminValidationSchema),
	UserController.createAdmin,
);
```

- Order: `checkAuth` → `validateRequest` → controller (note 13)।
- শুধু **SUPER_ADMIN** নতুন admin বানাতে পারবে।

---

## Part 2: Admin module (get / update / delete)

`src/app/module/admin/` folder, ফাইলগুলো:
- `admin.validation.ts`
- `admin.interface.ts`
- `admin.service.ts`
- `admin.controller.ts`
- `admin.route.ts`

সাথে `src/app/interfaces/requestUser.interface.ts` (ধাপ ১৩)।

### ধাপ ৭: `admin.validation.ts`

```ts
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

export const AdminValidation = {
	updateAdminValidationSchema,
};
```

- Body-র shape: `{ "admin": { ... } }` (create-এর মতো)।
- `email` নেই → email update করা যাবে না (login-এর পরিচয়, বদলালে User table-ও বদলাতে হতো)।
- `.refine` → `{ "admin": {} }` পাঠালে error, অকারণে DB query হবে না।
- `"Admin data is required"` → `admin` key-টাই না পাঠালে এই message।

### ধাপ ৮: `admin.interface.ts`

```ts
import type * as z from "zod";
import type { AdminValidation } from "./admin.validation";

export type IUpdateAdminPayload = z.infer<typeof AdminValidation.updateAdminValidationSchema>;
```

- হাতে interface না লিখে Zod schema থেকে type বানানো (`z.infer`) → schema বদলালে type নিজে থেকেই বদলাবে, দুই জায়গায় মিলাতে হবে না।

### ধাপ ৯: `admin.service.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import { Role, UserStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import type { IUpdateAdminPayload } from "./admin.interface";

const getAllAdmins = async () => {
	const admins = await prisma.admin.findMany({
		where: { isDeleted: false },
		include: { user: true },
	});
	return admins;
};

const getAdminById = async (id: string) => {
	const admin = await prisma.admin.findUnique({
		where: { id, isDeleted: false },
		include: { user: true },
	});

	if (!admin) {
		throw new AppError(httpStatus.NOT_FOUND, "Admin not found");
	}

	return admin;
};

const updateAdmin = async (id: string, payload: IUpdateAdminPayload, user: IRequestUser) => {
	//TODO: কে update করছে check করা। শুধু super admin admin-কে update করতে পারবে, আর super admin-কে শুধু super admin; admin super admin-কে update করতে পারবে না

	const isAdminExist = await prisma.admin.findUnique({
		where: { id, isDeleted: false },
		include: { user: true },
	});

	if (!isAdminExist) {
		throw new AppError(httpStatus.NOT_FOUND, "Admin not found");
	}

	// ADMIN, ADMIN-কে update করতে পারে, কিন্তু SUPER_ADMIN-কে শুধু SUPER_ADMIN
	if (isAdminExist.user.role === Role.SUPER_ADMIN && user.role !== Role.SUPER_ADMIN) {
		throw new AppError(httpStatus.FORBIDDEN, "Only super admin can update a super admin");
	}

	const { admin } = payload;

	const updatedAdmin = await prisma.$transaction(async (tx) => {
		// User table-ও মিলিয়ে রাখা (login / JWT-তে এটাই ব্যবহার হয়)
		if (admin.name !== undefined || admin.profilePhoto !== undefined) {
			await tx.user.update({
				where: { id: isAdminExist.userId },
				data: {
					name: admin.name,
					image: admin.profilePhoto,
				},
			});
		}

		return tx.admin.update({
			where: { id },
			data: admin,
			include: { user: true },
		});
	});

	return updatedAdmin;
};

// soft delete: isDeleted = true, সাথে user-এর session আর account মুছে ফেলা
const deleteAdmin = async (id: string, user: IRequestUser) => {
	//TODO: কে delete করছে check করা। শুধু super admin admin-কে delete করতে পারবে, আর super admin-কে শুধু super admin; admin super admin-কে delete করতে পারবে না

	const isAdminExist = await prisma.admin.findUnique({
		where: { id, isDeleted: false },
		include: { user: true },
	});

	if (!isAdminExist) {
		throw new AppError(httpStatus.NOT_FOUND, "Admin not found");
	}

	if (isAdminExist.userId === user.userId) {
		throw new AppError(httpStatus.BAD_REQUEST, "You cannot delete yourself");
	}
	// ADMIN, ADMIN-কে delete করতে পারে, কিন্তু SUPER_ADMIN-কে শুধু SUPER_ADMIN
	if (isAdminExist.user.role === Role.SUPER_ADMIN && user.role !== Role.SUPER_ADMIN) {
		throw new AppError(httpStatus.FORBIDDEN, "Only super admin can delete a super admin");
	}

	const deletedAt = new Date();

	const result = await prisma.$transaction(async (tx) => {
		await tx.admin.update({
			where: { id },
			data: { isDeleted: true, deletedAt },
		});

		await tx.user.update({
			where: { id: isAdminExist.userId },
			data: {
				isDeleted: true,
				deletedAt,
				status: UserStatus.DELETED,
			},
		});

		await tx.session.deleteMany({
			where: { userId: isAdminExist.userId },
		});

		await tx.account.deleteMany({
			where: { userId: isAdminExist.userId },
		});

		return tx.admin.findUnique({
			where: { id },
			include: { user: true },
		});
	});

	return result;
};

export const AdminService = {
	getAllAdmins,
	getAdminById,
	updateAdmin,
	deleteAdmin,
};
```

**Update-এ কেন transaction?**
`name` আর `profilePhoto` দুই table-এ আছে: `admin` (`name`, `profilePhoto`) আর `user` (`name`, `image`)। একটা update হয়ে আরেকটা fail করলে data অমিল হবে, তাই দুটো একসাথে `$transaction`-এ।
- Prisma `undefined` field ignore করে, তাই শুধু `contactNumber` পাঠালে `user` table-এ কিছুই বদলাবে না (আর `if` দিয়ে user update-টাই skip হয়)।

**Soft delete-এ কী কী হয় (সব এক transaction-এ):**

| Table | কী হয় | কেন |
|---|---|---|
| `admin` | `isDeleted: true`, `deletedAt` | list / get থেকে লুকানো |
| `user` | `isDeleted: true`, `deletedAt`, `status: DELETED` | login আটকানো (note 13-এর `loginUser` check) |
| `session` | সব মুছে ফেলা | এখনই সব device থেকে logout (checkAuth DB-তে session পাবে না) |
| `account` | সব মুছে ফেলা | Better Auth-এর password এখানে থাকে, তাই আর কখনো login সম্ভব না |

**Permission নিয়ম (service-এ):**

| কে → কাকে | Update | Delete |
|---|---|---|
| SUPER_ADMIN → ADMIN | ✅ | ✅ |
| SUPER_ADMIN → SUPER_ADMIN | ✅ | ✅ (নিজেকে না) |
| ADMIN → ADMIN | ✅ | ✅ (নিজেকে না) |
| ADMIN → SUPER_ADMIN | ❌ 403 | ❌ 403 |
| যে কেউ → নিজেকে | ✅ | ❌ 400 |

### ধাপ ১০: `admin.controller.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { AdminService } from "./admin.service";

const getAllAdmins = catchAsync(async (req: Request, res: Response) => {
	const result = await AdminService.getAllAdmins();

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Admins fetched successfully",
		data: result,
	});
});

const getAdminById = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	const result = await AdminService.getAdminById(id as string);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Admin fetched successfully",
		data: result,
	});
});

const updateAdmin = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}

	const result = await AdminService.updateAdmin(id as string, req.body, req.user);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Admin updated successfully",
		data: result,
	});
});

const deleteAdmin = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}

	const result = await AdminService.deleteAdmin(id as string, req.user);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Admin deleted successfully",
		data: result,
	});
});

export const AdminController = {
	getAllAdmins,
	getAdminById,
	updateAdmin,
	deleteAdmin,
};
```

- `if (!req.user)` → checkAuth থাকলে `req.user` সবসময় থাকবে, কিন্তু type-এ `user?` optional, তাই TypeScript-কে নিশ্চিত করতে এই check (আর ভুল করে route থেকে checkAuth বাদ পড়লেও নিরাপদ)।
- `id as string` → `noUncheckedIndexedAccess`-এর জন্য `req.params.id`-এর type `string | undefined` হয়।

### ধাপ ১১: `admin.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { AdminController } from "./admin.controller";
import { AdminValidation } from "./admin.validation";

const router = Router();

router.get("/", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AdminController.getAllAdmins);

router.get("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AdminController.getAdminById);

// ADMIN, ADMIN-কে update করতে পারে; SUPER_ADMIN-কে update করা service আটকায়
router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(AdminValidation.updateAdminValidationSchema),
	AdminController.updateAdmin,
);

// ADMIN, ADMIN-কে delete করতে পারে; SUPER_ADMIN-কে delete করা service আটকায়
router.delete("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), AdminController.deleteAdmin);

export const AdminRoutes = router;
```

- Route শুধু **role** দেখে (কে ঢুকতে পারবে)। **কাকে** update/delete করছে (target-এর role) সেটা DB না দেখে বোঝা যায় না, তাই ওই check service-এ।

### ধাপ ১২: `routes/index.ts`-এ mount

```ts
import { AdminRoutes } from "../module/admin/admin.route"; // ✏️ import নোটে ছিল না

router.use("/admins", AdminRoutes);
```

### ধাপ ১৩: `src/app/interfaces/requestUser.interface.ts`

```ts
import type { Role } from "@/generated/prisma/enums";

export interface IRequestUser {
	userId: string;
	name: string;
	email: string;
	role: Role;
}
```

- `req.user`-এর shape (note 13-এর `checkAuth` যা set করে)। Service-এ `user: IRequestUser` হিসেবে পাঠানো হয়।

---

## ধাপ ১৪: Test (Postman)

Base URL: `http://localhost:5000/api/v1`। আগে SUPER_ADMIN হিসেবে `POST /auth/login` করে নাও (cookie Postman নিজেই পাঠাবে)।

### Create admin

`POST /user/create-admin`

```json
{
  "password": "Admin@123",
  "admin": {
    "name": "Rahim Uddin",
    "email": "rahim.admin@example.com",
    "contactNumber": "01712345678",
    "profilePhoto": "https://example.com/rahim.jpg"
  }
}
```

### Update admin

`PATCH /admins/:id` — `:id` হলো **admin table-এর `id`** (`userId` না)

```json
{
  "admin": {
    "contactNumber": "01812345678"
  }
}
```

### বাকি

| Method | URL | আশা |
|---|---|---|
| `GET` | `/admins` | 200, deleted ছাড়া সব admin |
| `GET` | `/admins/:id` | 200 / 404 |
| `DELETE` | `/admins/:id` | 200, পরে GET করলে 404 |

**যে case-গুলো test করবে:**

| Case | আশা |
|---|---|
| ADMIN হিসেবে `POST /user/create-admin` | 403 |
| একই email দিয়ে আবার create | 409 |
| `PATCH` body `{ "admin": {} }` | 400 (At least one field...) |
| ADMIN হিসেবে SUPER_ADMIN-কে PATCH / DELETE | 403 |
| নিজেকে DELETE | 400 |
| Deleted admin দিয়ে login | blocked |

---

## 🔜 পরে যা যোগ হবে

- [ ] Super admin seed (প্রথম SUPER_ADMIN DB-তে বানানো)
- [ ] `req.params.id` Zod দিয়ে validate (uuid)
- [ ] Search / filter / pagination (`getAllAdmins`)
- [ ] Profile photo file upload

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **TODO comment আর code-এর নিয়ম মিলছে না** — TODO বলছে "শুধু super admin admin-কে update/delete করতে পারবে", কিন্তু code-এ ADMIN অন্য ADMIN-কেও update/delete করতে পারে। আবার create-এ শুধু SUPER_ADMIN। ফলে **একজন ADMIN নতুন admin বানাতে পারে না, কিন্তু অন্য admin-কে delete করে দিতে পারে** — নিয়মটা উল্টাপাল্টা। যেকোনো একটা ঠিক করো:
   - (ক) শুধু SUPER_ADMIN update/delete করবে (TODO অনুযায়ী) → route-এ `checkAuth(Role.SUPER_ADMIN)`, আর service-এর role check লাগবেই না:
     ```ts
     router.patch("/:id", checkAuth(Role.SUPER_ADMIN), validateRequest(...), AdminController.updateAdmin);
     router.delete("/:id", checkAuth(Role.SUPER_ADMIN), AdminController.deleteAdmin);
     ```
   - (খ) এখনকার code-ই ঠিক → TODO comment দুটো মুছে দাও, যাতে পরে বিভ্রান্তি না হয়।

### 🟡 ভালো practice

2. **Soft delete-এ `account` মুছে ফেলা** — Soft delete-এর উদ্দেশ্য হলো পরে restore করা যায়। কিন্তু `account` table-এ Better Auth-এর password hash থাকে, ওটা মুছলে restore করলেও admin আর login করতে পারবে না। আর login তো এমনিতেই `isDeleted` / `status: DELETED` check-এ আটকে যায় (note 13), session-ও মুছে যাচ্ছে। তাই `account.deleteMany` বাদ দিলেও নিরাপদ, আর restore সম্ভব থাকে।

3. **`IRequestUser` আর `express.d.ts` একই জিনিস দুইবার** — দুই জায়গায় একই ৪টা field। `express.d.ts`-এ এটাই ব্যবহার করো:
   ```ts
   import type { IRequestUser } from "./requestUser.interface";

   declare global {
   	namespace Express {
   		interface Request {
   			user?: IRequestUser;
   		}
   	}
   }
   ```

4. **Unused `req` → `_req`** — `getAllAdmins` controller-এ `req` ব্যবহার হয়নি (oxlint unused args)।
   ```ts
   const getAllAdmins = catchAsync(async (_req: Request, res: Response) => { ... });
   ```

5. **`createDoctor`-এও একই উন্নতি** — `createAdmin`-এ `AppError(409)` আর auto session delete আছে, কিন্তু note 10-এর `createDoctor`-এ এখনো `throw new Error(...)` আর session রয়ে যায়। দুটো একই রকম করো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

6. **`id as string`-এর বদলে typed params** — cast না করে Request-এ type দেওয়া যায়:
   ```ts
   const getAdminById = catchAsync(async (req: Request<{ id: string }>, res: Response) => {
   	const result = await AdminService.getAdminById(req.params.id);
   ```
   (`catchAsync`-এর `RequestHandler` type-এর সাথে মিলবে কিনা test করে দেখো।)

7. **Deleted admin-এর email আর ব্যবহার করা যাবে না** — soft delete-এ user row থেকে যায়, `email @unique`, তাই ওই email দিয়ে নতুন admin বানাতে গেলে 409। দরকার হলে delete-এর সময় email বদলে দেওয়া যায় (যেমন `deleted_<timestamp>_<email>`)।

8. **`include: { user: true }`** — response-এ user-এর সব field যায় (password থাকে না, ওটা `account`-এ)। তবুও `select` দিয়ে দরকারি field-গুলোই পাঠানো ভালো।
