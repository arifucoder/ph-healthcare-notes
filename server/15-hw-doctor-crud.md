# 15 - HW: Doctor CRUD

> 🏠 Home work — Admin CRUD (note 14)-এর মতো করে doctor-এর get by id / update / soft delete।
> Doctor create আর `getAllDoctors`-এর শুরু → [note 10](./10-create-doctor-and-doctor-module.md)

```
Create → user module (POST /user/create-doctor, note 10)
বাকি  → doctor module (GET / GET :id → সবার জন্য খোলা, PATCH / DELETE → ADMIN + SUPER_ADMIN)
```

| কাজ | Method | URL | কে পারবে |
|---|---|---|---|
| সব doctor | `GET` | `/api/v1/doctors` | সবাই (login লাগে না) |
| একজন doctor | `GET` | `/api/v1/doctors/:id` | সবাই (login লাগে না) |
| Update | `PATCH` | `/api/v1/doctors/:id` | ADMIN, SUPER_ADMIN |
| Soft delete | `DELETE` | `/api/v1/doctors/:id` | ADMIN, SUPER_ADMIN |

- GET খোলা রাখা হয়েছে, কারণ patient (বা login ছাড়া যে কেউ) doctor-দের list দেখে appointment নেবে।

`src/app/module/doctor/` folder-এর ফাইলগুলো: `doctor.validation.ts`, `doctor.interface.ts`, `doctor.service.ts`, `doctor.controller.ts`, `doctor.route.ts`

---

## ধাপ ১: `doctor.validation.ts`

```ts
import { Gender } from "@/generated/prisma/enums";
import * as z from "zod";

const updateDoctorValidationSchema = z.object({
	doctor: z
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

				address: z
					.string("Address must be a string")
					.trim()
					.min(10, "Address must be at least 10 characters")
					.max(100, "Address must be at most 100 characters")
					.optional(),

				experience: z.int("Experience must be an integer").nonnegative("Experience cannot be negative").optional(),

				gender: z.enum(Gender, "Gender must be either MALE or FEMALE").optional(),

				appointmentFee: z
					.number("Appointment fee must be a number")
					.nonnegative("Appointment fee cannot be negative")
					.optional(),

				qualification: z
					.string("Qualification must be a string")
					.trim()
					.min(2, "Qualification must be at least 2 characters")
					.max(50, "Qualification must be at most 50 characters")
					.optional(),

				currentWorkingPlace: z
					.string("Current working place must be a string")
					.trim()
					.min(2, "Current working place must be at least 2 characters")
					.max(50, "Current working place must be at most 50 characters")
					.optional(),

				designation: z
					.string("Designation must be a string")
					.trim()
					.min(2, "Designation must be at least 2 characters")
					.max(50, "Designation must be at most 50 characters")
					.optional(),
			},
			"Doctor data is required",
		)
		.refine((data) => Object.keys(data).length > 0, "At least one field is required to update"),
});

export const DoctorValidation = {
	updateDoctorValidationSchema,
};
```

- Body-র shape: `{ "doctor": { ... } }` (create-এর মতো)।
- প্রতিটা field `.optional()` → যেটা পাঠাবে শুধু সেটাই update।
- `.refine` → `{ "doctor": {} }` পাঠালে error।
- **`email` আর `registrationNumber` নেই** → এগুলো update করা যাবে না। Email দিয়ে login হয়, আর registration number (BMDC) doctor-এর স্থায়ী পরিচয়, দুটোই `@unique`।

> 📝 Note 11-এর `updateDoctorZodSchema` (flat `strictObject`) এর বদলে এখন এটা ব্যবহার হচ্ছে। তাই body এখন `{ "doctor": {...} }`, আগের মতো সরাসরি `{ "name": ... }` না।

## ধাপ ২: `doctor.interface.ts`

```ts
import type { Gender } from "@/generated/prisma/enums";

export interface IUpdateDoctorPayload {
	doctor: {
		name?: string;
		profilePhoto?: string;
		contactNumber?: string;
		address?: string;
		experience?: number;
		gender?: Gender;
		appointmentFee?: number;
		qualification?: string;
		currentWorkingPlace?: string;
		designation?: string;
	};
}
```

## ধাপ ৩: `doctor.service.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { UserStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import type { IUpdateDoctorPayload } from "./doctor.interface";

// বারবার ব্যবহারের include, যাতে প্রতিটা query একই shape-এর data দেয়
const doctorInclude = {
	user: true,
	specialties: {
		include: {
			specialty: true,
		},
	},
} as const;

const getAllDoctors = async () => {
	const doctors = await prisma.doctor.findMany({
		where: { isDeleted: false },
		include: doctorInclude,
	});
	return doctors;
};

const getDoctorById = async (id: string) => {
	const doctor = await prisma.doctor.findUnique({
		where: { id, isDeleted: false },
		include: doctorInclude,
	});

	if (!doctor) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	}

	return doctor;
};

const updateDoctor = async (id: string, payload: IUpdateDoctorPayload) => {
	const isDoctorExist = await prisma.doctor.findUnique({
		where: { id, isDeleted: false },
	});

	if (!isDoctorExist) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	}

	const { doctor } = payload;

	const updatedDoctor = await prisma.$transaction(async (tx) => {
		// User table-ও মিলিয়ে রাখা (login / JWT-তে এটাই ব্যবহার হয়)
		if (doctor.name !== undefined || doctor.profilePhoto !== undefined) {
			await tx.user.update({
				where: { id: isDoctorExist.userId },
				data: {
					name: doctor.name,
					image: doctor.profilePhoto,
				},
			});
		}

		return tx.doctor.update({
			where: { id },
			data: doctor,
			include: doctorInclude,
		});
	});

	return updatedDoctor;
};

// soft delete: isDeleted = true, সাথে user-এর session আর account মুছে ফেলা
const deleteDoctor = async (id: string) => {
	const isDoctorExist = await prisma.doctor.findUnique({
		where: { id, isDeleted: false },
	});

	if (!isDoctorExist) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	}

	const deletedAt = new Date();

	const result = await prisma.$transaction(async (tx) => {
		await tx.doctor.update({
			where: { id },
			data: { isDeleted: true, deletedAt },
		});

		await tx.user.update({
			where: { id: isDoctorExist.userId },
			data: {
				isDeleted: true,
				deletedAt,
				status: UserStatus.DELETED,
			},
		});

		await tx.session.deleteMany({
			where: { userId: isDoctorExist.userId },
		});

		await tx.account.deleteMany({
			where: { userId: isDoctorExist.userId },
		});

		return tx.doctor.findUnique({
			where: { id },
			include: doctorInclude,
		});
	});

	return result;
};

export const DoctorService = {
	getAllDoctors,
	getDoctorById,
	updateDoctor,
	deleteDoctor,
};
```

**`doctorInclude` কেন আলাদা?**
৪টা query-তেই একই `include` লাগছে। একবার লিখে রাখলে সব response একই shape-এর হয়, আর বদলাতে হলে এক জায়গায় বদলালেই হয়। `as const` দিলে Prisma বুঝতে পারে `user: true` মানে ঠিক `true` (সাধারণ `boolean` না), তাই return type-এ `user` আর `specialties` ঠিকমতো আসে।

**Note 10 থেকে যা বদলেছে:**
- `getAllDoctors`-এ এখন `where: { isDeleted: false }` → deleted doctor list-এ আসবে না (note 10-এর recommendation ✅)।

**Update-এ কেন transaction?**
`name` আর `profilePhoto` দুই table-এ আছে: `doctor` (`name`, `profilePhoto`) আর `user` (`name`, `image`)। দুটো একসাথে `$transaction`-এ, যাতে একটা বদলে আরেকটা fail না করে। Prisma `undefined` field ignore করে, তাই যেটা পাঠানো হয়নি সেটা বদলায় না।

**Soft delete-এ কী কী হয় (সব এক transaction-এ):**

| Table | কী হয় | কেন |
|---|---|---|
| `doctor` | `isDeleted: true`, `deletedAt` | list / get থেকে লুকানো |
| `user` | `isDeleted: true`, `deletedAt`, `status: DELETED` | login আটকানো (note 13) |
| `session` | সব মুছে ফেলা | এখনই সব device থেকে logout |
| `account` | সব মুছে ফেলা | password মুছে যায়, আর কখনো login সম্ভব না |
| `doctor_specialties` | কিছু হয় না | doctor row থেকে যায়, তাই link-ও থাকে |

## ধাপ ৪: `doctor.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { DoctorService } from "./doctor.service";

const getAllDoctors = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorService.getAllDoctors();

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Doctors fetched successfully",
		data: result,
	});
});

const getDoctorById = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	const result = await DoctorService.getDoctorById(id as string);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Doctor fetched successfully",
		data: result,
	});
});

const updateDoctor = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	const result = await DoctorService.updateDoctor(id as string, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Doctor updated successfully",
		data: result,
	});
});

const deleteDoctor = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	const result = await DoctorService.deleteDoctor(id as string);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Doctor deleted successfully",
		data: result,
	});
});

export const DoctorController = {
	getAllDoctors,
	getDoctorById,
	updateDoctor,
	deleteDoctor,
};
```

- Admin-এর মতো এখানে `req.user` লাগছে না, কারণ ADMIN / SUPER_ADMIN যে কোনো doctor-কে update/delete করতে পারে, target-এর role check করার দরকার নেই।

## ধাপ ৫: `doctor.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { DoctorController } from "./doctor.controller";
import { DoctorValidation } from "./doctor.validation";

const router = Router();

router.get("/", DoctorController.getAllDoctors);

router.get("/:id", DoctorController.getDoctorById);

router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(DoctorValidation.updateDoctorValidationSchema),
	DoctorController.updateDoctor,
);

router.delete("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), DoctorController.deleteDoctor);

export const DoctorRoutes = router;
```

- `routes/index.ts`-এ `router.use("/doctors", DoctorRoutes)` আগেই আছে (note 10), নতুন কিছু লাগবে না।

---

## ধাপ ৬: Test (Postman)

Base URL: `http://localhost:5000/api/v1`। PATCH / DELETE-এর আগে ADMIN বা SUPER_ADMIN হিসেবে `POST /auth/login`।

### Update doctor

`PATCH /doctors/:id` — `:id` হলো **doctor table-এর `id`** (`userId` না)

```json
{
  "doctor": {
    "appointmentFee": 1200,
    "designation": "Senior Consultant"
  }
}
```

### বাকি

| Method | URL | Login | আশা |
|---|---|---|---|
| `GET` | `/doctors` | লাগে না | 200, deleted ছাড়া সব doctor + specialties |
| `GET` | `/doctors/:id` | লাগে না | 200 / 404 |
| `DELETE` | `/doctors/:id` | ADMIN / SUPER_ADMIN | 200, পরে GET করলে 404 |

**যে case-গুলো test করবে:**

| Case | আশা |
|---|---|
| login ছাড়া PATCH / DELETE | 401 |
| PATIENT / DOCTOR হিসেবে PATCH / DELETE | 403 |
| `{ "doctor": {} }` | 400 (At least one field...) |
| `{ "doctor": { "gender": "OTHER" } }` | 400 |
| ভুল / deleted id | 404 |
| Deleted doctor দিয়ে login | blocked |

---

## 🔜 পরে যা যোগ হবে

- [ ] Doctor-এর specialty যোগ / বাদ দেওয়া (update-এ এখন specialties বদলানো যায় না)
- [ ] Doctor নিজের profile নিজে update (`/me`, `checkAuth(Role.DOCTOR)`)
- [ ] Search / filter (specialty, নাম, fee) আর pagination
- [ ] `req.params.id` Zod দিয়ে validate
- [ ] Profile photo file upload

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Public GET route-এ পুরো `user` object যাচ্ছে** — `GET /doctors` আর `GET /doctors/:id`-এ login লাগে না, কিন্তু `doctorInclude`-এর `user: true` user-এর সব field পাঠায়: `role`, `status`, `needPasswordChange`, `emailVerified`, `isDeleted` ইত্যাদি। (Password থাকে না, ওটা `account`-এ।) এগুলো বাইরের কারো জানার দরকার নেই। Public query-র জন্য আলাদা করে দরকারি field-গুলোই নাও:
   ```ts
   const publicDoctorInclude = {
   	specialties: {
   		select: { specialty: { select: { id: true, title: true } } },
   	},
   } as const;
   // getAllDoctors / getDoctorById-এ include: publicDoctorInclude
   ```
   (doctor-এর নিজের `name`, `qualification`, `appointmentFee` ইত্যাদি doctor table-এই আছে, তাই `user` ছাড়াও profile দেখানো যায়।)

### 🟡 ভালো practice

2. **Interface হাতে না লিখে `z.infer`** — admin-এ (note 14) `z.infer` ব্যবহার করেছ, এখানে হাতে লেখা। Schema-তে field যোগ করলে interface-এ ভুলে গেলে অমিল হবে। একই নিয়ম রাখো:
   ```ts
   import type * as z from "zod";
   import type { DoctorValidation } from "./doctor.validation";

   export type IUpdateDoctorPayload = z.infer<typeof DoctorValidation.updateDoctorValidationSchema>;
   ```

3. **Soft delete-এ `account.deleteMany`** — admin-এর মতোই (note 14, recommendation ২): password hash মুছে গেলে restore করলেও doctor login করতে পারবে না। Login এমনিতেই `isDeleted` / `status` check-এ আটকায়, session-ও মুছছে, তাই এটা বাদ দিলেও নিরাপদ।

4. **Unused `req` → `_req`** — `getAllDoctors` controller-এ (oxlint unused args)।
   ```ts
   const getAllDoctors = catchAsync(async (_req: Request, res: Response) => { ... });
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **`as const`-এর বদলে `satisfies`** — `as const` কাজ করে, কিন্তু typo করলে (যেমন `specialtys`) error দেয় না। `satisfies` দিলে Prisma-র type দিয়ে check হয়, আর type-ও ঠিক থাকে:
   ```ts
   import type { Prisma } from "@/generated/prisma/client";

   const doctorInclude = {
   	user: true,
   	specialties: { include: { specialty: true } },
   } satisfies Prisma.DoctorInclude;
   ```

6. **`isDoctorExist`-এর `userId`** — update/delete-এ শুধু `userId` লাগছে, তাই `select: { userId: true }` দিলে কম data আসবে। ছোট উন্নতি।
