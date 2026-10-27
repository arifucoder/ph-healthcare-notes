# 10 - Create Doctor (Transaction) ও Doctor Module

> Module: 38-03

```
doctor.prisma + Gender enum + DoctorSpecialty → migrate → user module (create-doctor) → doctor module (get all)
```

**Flow:** `POST /api/v1/user/create-doctor` → specialty check → email check → `auth.api.signUpEmail` (role DOCTOR) → `$transaction` (doctor create + doctorSpecialty createMany + doctor find) → fail হলে user delete (rollback)

---

## ধাপ ১: `prisma/schema/doctor.prisma` বানানো

```prisma
model Doctor {
  id String @id @default(uuid(7))

  name          String
  email         String    @unique
  profilePhoto  String?
  contactNumber String?
  address       String?
  isDeleted     Boolean   @default(false)
  deletedAt     DateTime?

  registrationNumber  String @unique
  experience          Int    @default(0)
  gender              Gender
  appointmentFee      Float
  qualification       String
  currentWorkingPlace String
  designation         String
  averageRating       Float  @default(0.0)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  //relations

  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)

  specialties DoctorSpecialty[]

  @@index([email], name: "idx_doctor_email")
  @@index([isDeleted], name: "idx_doctor_isDeleted")
  @@map("doctor")
}
```

- `userId @unique` → একজন user-এর একটাই doctor profile (one-to-one)।
- `onDelete: Cascade` → user delete হলে doctor-ও delete হবে।

## ধাপ ২: `enums.prisma`-তে `Gender` enum

```prisma
enum Gender {
    MALE
    FEMALE
}
```

## ধাপ ৩: `specialty.prisma`-তে `DoctorSpecialty` model (join table)

Doctor ↔ Specialty হলো many-to-many, তাই মাঝে একটা join table।

```prisma
model DoctorSpecialty {
  id          String @id @default(uuid(7))
  doctorId    String
  specialtyId String

  doctor    Doctor    @relation(fields: [doctorId], references: [id], onDelete: Cascade, onUpdate: Cascade)
  specialty Specialty @relation(fields: [specialtyId], references: [id], onDelete: Cascade, onUpdate: Cascade)

  @@unique([doctorId, specialtyId])
  @@index([doctorId], name: "idx_doctor_specialty_doctorId")
  @@index([specialtyId], name: "idx_doctor_specialty_specialtyId")
  @@map("doctor_specialties")
}
```

- `@@unique([doctorId, specialtyId])` → একই doctor-এর সাথে একই specialty দুইবার যোগ হবে না।

> ✏️ Prisma-তে relation দুই দিকেই লাগে, এটা নোটে ছিল না। `User` model-এ (`auth.prisma`) আর `Specialty` model-এ back-relation field না থাকলে migrate error দেবে:
> ```prisma
> // auth.prisma → model User { ... }
> doctor Doctor?
>
> // specialty.prisma → model Specialty { ... }
> doctorSpecialties DoctorSpecialty[]
> ```
> (field-এর নাম যা খুশি দেওয়া যায়, শুধু টাইপটা ঠিক থাকতে হবে। আপনার project-এ আগে থেকে থাকলে কিছু করতে হবে না।)

## ধাপ ৪: Migrate ও Generate

```shell
pnpm migrate && pnpm generate
```

---

## ধাপ ৫: `src/app/module/user/` module বানানো

ফাইলগুলো:
- `user.route.ts`
- `user.controller.ts`
- `user.service.ts`
- `user.interface.ts`

### `user.interface.ts`

```ts
import type { Gender } from "@/generated/prisma/enums";

export interface ICreateDoctorPayload {
	password: string;
	doctor: {
		name: string;
		email: string;
		profilePhoto?: string;
		contactNumber?: string;
		address?: string;
		registrationNumber: string;
		experience?: number;
		gender: Gender;
		appointmentFee: number;
		qualification: string;
		currentWorkingPlace: string;
		designation: string;
	};
	specialties: string[];
}
```

### `user.service.ts`

```ts
import { auth } from "@/app/lib/auth";
import { prisma } from "@/app/lib/prisma";
import { Role, type Specialty } from "@/generated/prisma/client";
import type { ICreateDoctorPayload } from "./user.interface";

const createDoctor = async (payload: ICreateDoctorPayload) => {
	const specialties: Specialty[] = [];

	for (const specialtyId of payload.specialties) {
		const specialty = await prisma.specialty.findUnique({
			where: {
				id: specialtyId,
			},
		});
		if (!specialty) {
			// throw new Error(`Specialty with id ${specialtyId} not found`);
			throw new Error(`Specialty with id ${specialtyId} not found`);
		}
		specialties.push(specialty);
	}

	const userExists = await prisma.user.findUnique({
		where: {
			email: payload.doctor.email,
		},
	});

	if (userExists) {
		// throw new Error("User with this email already exists");
		throw new Error("User with this email already exists");
	}

	const userData = await auth.api.signUpEmail({
		body: {
			email: payload.doctor.email,
			password: payload.password,
			role: Role.DOCTOR,
			name: payload.doctor.name,
			needPasswordChange: true,
		},
	});

	try {
		const result = await prisma.$transaction(async (tx) => {
			const doctorData = await tx.doctor.create({
				data: {
					userId: userData.user.id,
					...payload.doctor,
				},
			});

			const doctorSpecialtyData = specialties.map((specialty) => {
				return {
					doctorId: doctorData.id,
					specialtyId: specialty.id,
				};
			});

			await tx.doctorSpecialty.createMany({
				data: doctorSpecialtyData,
			});

			const doctor = await tx.doctor.findUnique({
				where: {
					id: doctorData.id,
				},
				select: {
					id: true,
					userId: true,
					name: true,
					email: true,
					profilePhoto: true,
					contactNumber: true,
					address: true,
					registrationNumber: true,
					experience: true,
					gender: true,
					appointmentFee: true,
					qualification: true,
					currentWorkingPlace: true,
					designation: true,
					createdAt: true,
					updatedAt: true,
					user: {
						select: {
							id: true,
							email: true,
							name: true,
							role: true,
							status: true,
							emailVerified: true,
							image: true,
							isDeleted: true,
							deletedAt: true,
							createdAt: true,
							updatedAt: true,
						},
					},
					specialties: {
						select: {
							specialty: {
								select: {
									title: true,
									id: true,
								},
							},
						},
					},
				},
			});

			return doctor;
		});

		return result;
	} catch (error) {
		console.log("Transaction error : ", error);
		await prisma.user.delete({
			where: {
				id: userData.user.id,
			},
		});
		throw error;
	}
};

export const UserService = {
	createDoctor,
};
```

**কেন `signUpEmail` transaction-এর ভিতরে নেই?**
`signUpEmail` হলো Better Auth-এর কাজ, Prisma-র `tx` এর অংশ না। তাই এটাকে `$transaction`-এর ভিতরে আনা সম্ভব না। এজন্য transaction fail হলে `catch`-এ হাতে করে user delete করে rollback করি (note 07-এর patient-এর মতো)।

**`select` নিয়ে:** Teacher এখানে প্রায় সব field select করেছেন, তাই response-এ সব data আসবে। Real project-এ যতটুকু data দরকার, ঠিক ততটুকুই select করব।

### `user.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { UserService } from "./user.service";

const createDoctor = catchAsync(async (req: Request, res: Response) => {
	const payload = req.body;

	const result = await UserService.createDoctor(payload);

	sendResponse(res, {
		httpStatusCode: httpStatus.CREATED,
		success: true,
		message: "Doctor registered successfully",
		data: result,
	});
});

export const UserController = {
	createDoctor,
};
```

### `user.route.ts`

```ts
import { Router } from "express";
import { UserController } from "./user.controller";

const router = Router();

router.post("/create-doctor", UserController.createDoctor);

export const UserRoutes = router;
```

## ধাপ ৬: `routes/index.ts`-এ mount

```ts
import { UserRoutes } from "../module/user/user.route"; // ✏️ import নোটে ছিল না

router.use("/user", UserRoutes);
```

## ধাপ ৭: Postman-এ Test

| Method | URL |
|---|---|
| `POST` | `http://localhost:5000/api/v1/user/create-doctor` |

> port আপনার `.env` অনুযায়ী।

Body (JSON):

```json
{
  "password": "Pa$$w0rd!",
  "doctor": {
    "name": "Dr. Mohammad Rahman",
    "email": "dr.rahman@example.com",
    "contactNumber": "+8801712345678",
    "address": "Dhanmondi, Dhaka, Bangladesh",
    "registrationNumber": "BMDC-A-12345",
    "experience": 8,
    "gender": "MALE",
    "appointmentFee": 1000,
    "qualification": "MBBS, FCPS (Medicine)",
    "currentWorkingPlace": "Dhaka Medical College Hospital",
    "designation": "Consultant Physician"
  },
  "specialties": [
    "fa73ac31-4e45-4d49-9167-f51c128c258a",
    "ea876ab9-2cfb-4ec3-9a52-478ac1f82e75"
  ]
}
```

> `specialties`-এ DB-তে থাকা আসল specialty `id` দিতে হবে (Prisma Studio বা `GET /api/v1/specialties` থেকে নাও)।

---

## Doctor create user module-এ, বাকি কাজ doctor module-এ — কেন?

| কাজ | কোন module | কারণ |
|---|---|---|
| Doctor **create** | `user` | নতুন **user account** (Better Auth) + doctor profile দুটোই তৈরি হয়, তাই account তৈরির কাজ user module-এ |
| Doctor **get / update / delete** | `doctor` | শুধু doctor profile-এর কাজ |

Admin আর Super Admin-এর জন্যও একই নিয়ম: create হবে `user` module-এ (`create-admin` ইত্যাদি), বাকি CRUD নিজ নিজ module-এ।

---

## ধাপ ৮: `src/app/module/doctor/` module বানানো

ফাইলগুলো:
- `doctor.controller.ts`
- `doctor.interface.ts`
- `doctor.route.ts`
- `doctor.service.ts`

### `doctor.interface.ts`

```ts
export interface IUpdateDoctorPayload {
	name?: string;
	profilePhoto?: string;
	contactNumber?: string;
	address?: string;
	experience?: number;
}
```

### `doctor.service.ts`

```ts
import { prisma } from "@/app/lib/prisma";

const getAllDoctors = async () => {
	const doctors = await prisma.doctor.findMany({
		include: {
			user: true,
			specialties: {
				include: {
					specialty: true,
				},
			},
		},
	});
	return doctors;
};

// const getDoctorById = async (id: string) => {}

// const updateDoctor = async (id: string, payload: IUpdateDoctorPayload) => {}

// const deleteDoctor = async (id: string) => {} //soft delete

export const DoctorService = {
	getAllDoctors,
};
```

### `doctor.controller.ts`

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

export const DoctorController = {
	getAllDoctors,
};
```

### `doctor.route.ts`

```ts
import { Router } from "express";
import { DoctorController } from "./doctor.controller";

const router = Router();

router.get("/", DoctorController.getAllDoctors);
// router.get("/:id", DoctorController.getDoctorById);
// router.put("/:id", DoctorController.updateDoctor);
//router.patch("/:id", DoctorController.updateDoctor);
// router.delete("/:id", DoctorController.deleteDoctor);

export const DoctorRoutes = router;
```

## ধাপ ৯: `routes/index.ts`-এ mount

```ts
import { DoctorRoutes } from "../module/doctor/doctor.route"; // ✏️ import নোটে ছিল না

router.use("/doctors", DoctorRoutes);
```

### Test

| Method | URL |
|---|---|
| `GET` | `http://localhost:5000/api/v1/doctors` |

---

## 🔜 পরে যা যোগ হবে

- [x] `getDoctorById` — `GET /api/v1/doctors/:id` → [note 15](./15-hw-doctor-crud.md)
- [x] `updateDoctor` — `PATCH /api/v1/doctors/:id` (`IUpdateDoctorPayload`) → [note 15](./15-hw-doctor-crud.md)
- [x] `deleteDoctor` (soft delete: `isDeleted: true`, `deletedAt`) — `DELETE /api/v1/doctors/:id` → [note 15](./15-hw-doctor-crud.md)
- [x] Admin / Super Admin create (user module-এ) → [note 14](./14-hw-admin-crud.md)
- [x] Request validation (Zod) → [note 11](./11-zod-validation.md)
- [x] Auth guard — শুধু ADMIN / SUPER_ADMIN doctor create করতে পারবে → [note 13](./13-jwt-token-cookie-and-check-auth.md)

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **`console.log` → `console.error`** — oxlint-এর `no-console` rule শুধু `warn`/`error` allow করে, তাই `pnpm lint`-এ `console.log` error দেবে।
   ```ts
   console.error("Transaction error : ", error);
   ```

2. ✅ (Done → [note 15](./15-hw-doctor-crud.md)) **`getAllDoctors`-এ deleted doctor বাদ দাও** — soft delete চালু হলে deleted doctor-ও list-এ চলে আসবে।
   ```ts
   const doctors = await prisma.doctor.findMany({
   	where: { isDeleted: false },
   	include: { ... },
   });
   ```

### 🟡 ভালো practice

3. **Specialty check এক query-তে** — loop-এ প্রতিটা id-র জন্য আলাদা query হচ্ছে। `findMany` দিয়ে একবারেই করা যায়:
   ```ts
   const specialties = await prisma.specialty.findMany({
   	where: { id: { in: payload.specialties } },
   });
   if (specialties.length !== payload.specialties.length) {
   	throw new Error("One or more specialties not found");
   }
   ```

4. **Unused `req` → `_req`** — `getAllDoctors` controller-এ `req` ব্যবহার হয়নি। `_` দিলে oxlint warning দেবে না।
   ```ts
   const getAllDoctors = catchAsync(async (_req: Request, res: Response) => { ... });
   ```

5. **`include: { user: true }` না দিয়ে `select`** — সব user field client-এ পাঠানো দরকার নেই। দরকারি field-গুলোই নাও (create-এর মতো)।

6. **Error-এ status code** — এখন `throw new Error(...)` দিলে global error handler সম্ভবত 500 দেখাবে। "not found" / "already exists" এর জন্য 404 / 409 ভালো। পরে একটা custom `AppError(statusCode, message)` class বানালে সুবিধা হবে।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

7. **`signUpEmail` session তৈরি করে** — Better Auth sign up-এর সময় doctor-এর জন্য একটা session/token-ও বানায় (autoSignIn চালু থাকলে)। Admin অন্য কারো account বানাচ্ছে, তাই এটা দরকার নেই, তবে এখন কোনো সমস্যা করে না।

8. **`createMany`-তে খালি array** — `specialties: []` পাঠালে doctor তৈরি হবে কিন্তু কোনো specialty থাকবে না। কমপক্ষে ১টা specialty চাইলে Zod validation-এ `.min(1)` দিও।
