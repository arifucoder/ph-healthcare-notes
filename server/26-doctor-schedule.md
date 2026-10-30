# 26 - Doctor Schedule (doctor নিজের slot বেছে নেয়)

```
Admin   → Schedule (৩০ মিনিটের ফাঁকা slot, নোট ২৫)
Doctor  → POST /doctor-schedules/create-my-doctor-schedule { scheduleIds }
        → slot আছে + ভবিষ্যতের কিনা যাচাই → DoctorSchedules (doctorId + scheduleId) তৈরি
Patient → (পরে) এই slot book করবে → isBooked: true
```

`DoctorSchedules` হলো Doctor আর Schedule-এর মাঝের join table ([নোট ২৫](./25-schedule.md)-এ model আছে)। এর নিজের `id` নেই — `@@id([doctorId, scheduleId])` (composite key)। এই কারণেই QueryBuilder-এ একটা ছোট পরিবর্তন লাগে (ধাপ ১–২)।

---

## ধাপ ১: `src/app/interfaces/query.interface.ts` — `IQueryConfig`-এ একটা field যোগ

```ts
export interface IQueryConfig {
	searchableFields?: readonly string[];
	filterableFields?: readonly string[];
	sortableFields?: readonly string[];
	selectableFields?: readonly string[];
	enumFields?: readonly string[]; // value auto UPPERCASE
	caseInsensitiveFields?: readonly string[]; // exact match, কিন্তু case ignore
	tieBreakerFields?: readonly string[]; // default ["id"] — composite key model-এ বদলাও
	defaultSortBy?: string;
	defaultLimit?: number;
	maxLimit?: number;
}
```

---

## ধাপ ২: `src/app/utils/QueryBuilder.ts` — শুধু `sort()` method বদলাও

```ts
// ?sortBy=appointmentFee&sortOrder=asc
sort(): this {
	const { sortableFields = [], defaultSortBy = "createdAt", tieBreakerFields = ["id"] } = this.config;
	const requested = this.getString("sortBy");

	const sortBy = requested && sortableFields.includes(requested) ? requested : defaultSortBy;
	const sortOrder = this.getString("sortOrder")?.toLowerCase() === "asc" ? "asc" : "desc";

	// tie-breaker: একই value-ওয়ালা row page বদলালে duplicate বা miss হবে না
	this.orderBy = [toNested(sortBy, sortOrder, false)];
	for (const field of tieBreakerFields) {
		if (field !== sortBy) this.orderBy.push({ [field]: sortOrder });
	}

	return this;
}
```

**কেন?** আগের `sort()` সবসময় `{ id: sortOrder }` যোগ করত। কিন্তু `DoctorSchedules`-এ `id` column নেই → Prisma error। এখন tie-breaker config থেকে আসে:

| Model | `tieBreakerFields` | `orderBy` |
|---|---|---|
| Doctor, Schedule (আগের মতো) | দেওয়া নেই → `["id"]` | `[{ sortBy }, { id }]` |
| DoctorSchedules | `["doctorId", "scheduleId"]` | `[{ schedule: { startDateTime } }, { doctorId }, { scheduleId }]` |

- আগের module-গুলোতে কিছু বদলাতে হয় না — default `["id"]`।

---

## ধাপ ৩: `doctorSchedule.validation.ts`

```ts
import * as z from "zod";

const scheduleId = z.uuid("Each schedule id must be a valid UUID");

// ---------- Create ----------
const createDoctorScheduleZodSchema = z.object({
	scheduleIds: z
		.array(scheduleId, "scheduleIds must be an array of schedule ids")
		.min(1, "At least one schedule id is required")
		// একই id দুইবার এলে একবারই রাখা হয়
		.transform((ids) => [...new Set(ids)]),
});

// ---------- Update (কিছু যোগ, কিছু বাদ) ----------
const updateDoctorScheduleZodSchema = z
	.object({
		scheduleIds: z
			.array(
				z.object({
					id: scheduleId,
					shouldDelete: z.boolean("shouldDelete must be true or false"),
				}),
				"scheduleIds must be an array",
			)
			.min(1, "At least one schedule is required"),
	})
	.refine((data) => new Set(data.scheduleIds.map((s) => s.id)).size === data.scheduleIds.length, {
		message: "Duplicate schedule id found",
		path: ["scheduleIds"],
	});

export const DoctorScheduleValidation = {
	createDoctorScheduleZodSchema,
	updateDoctorScheduleZodSchema,
};
```

| Schema | কী করে |
|---|---|
| Create | uuid-এর array, অন্তত ১টা; duplicate id নিজে থেকেই বাদ (`Set`) |
| Update | `{ id, shouldDelete }`-এর array; একই id দুবার থাকলে `400` (একই id যোগ আর বাদ দুটো একসাথে হতে পারে না) |

- `.transform()`-এর ফল `validateRequest` `req.body`-তে বসিয়ে দেয় — তাই service duplicate-ছাড়া array পায়।

---

## ধাপ ৪: `doctorSchedule.interface.ts`

```ts
export interface ICreateDoctorSchedulePayload {
	scheduleIds: string[];
}

export interface IUpdateDoctorSchedulePayload {
	scheduleIds: {
		shouldDelete: boolean;
		id: string;
	}[];
}
```

---

## ধাপ ৫: `doctorSchedule.constant.ts`

```ts
import type { Prisma } from "@/generated/prisma/client";

// ?searchTerm=rahim → doctor-এর নাম/email দিয়ে খোঁজা (UUID-এ contains অর্থহীন)
export const doctorScheduleSearchableFields = ["doctor.name", "doctor.email"] as const;

// "id" নেই — DoctorSchedules-এর composite key (doctorId + scheduleId)
export const doctorScheduleFilterableFields = [
	"doctorId",
	"scheduleId",
	"isBooked",
	"schedule.startDateTime",
	"schedule.endDateTime",
	"createdAt",
] as const;

export const doctorScheduleSortableFields = ["createdAt", "schedule.startDateTime", "schedule.endDateTime"] as const;

// composite key দিয়ে tie-break
export const DOCTOR_SCHEDULE_TIE_BREAKERS = ["doctorId", "scheduleId"] as const;

// appointments ইচ্ছা করে রাখা হয়নি — রোগীর data appointment module থেকে আসবে
export const doctorScheduleIncludeConfig = {
	doctor: { include: { specialties: { include: { specialty: true } } } },
	schedule: true,
} satisfies Prisma.DoctorSchedulesInclude;
```

- `doctor.name`, `schedule.startDateTime` → ২ ভাগের path = **to-one** relation (একটা DoctorSchedules-এর একটাই doctor, একটাই schedule) — [নোট ২৪](./24-query-builder.md)-এর Path rule।

---

## ধাপ ৬: `doctorSchedule.service.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IQueryParams } from "@/app/interfaces/query.interface";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import { QueryBuilder } from "@/app/utils/QueryBuilder";
import type { DoctorSchedules, Prisma } from "@/generated/prisma/client";
import httpStatus from "http-status";
import {
	DOCTOR_SCHEDULE_TIE_BREAKERS,
	doctorScheduleFilterableFields,
	doctorScheduleIncludeConfig,
	doctorScheduleSearchableFields,
	doctorScheduleSortableFields,
} from "./doctorSchedule.constant";
import type { ICreateDoctorSchedulePayload, IUpdateDoctorSchedulePayload } from "./doctorSchedule.interface";
```

### ক) Helper function

```ts
// logged-in user-এর doctor profile (না থাকলে 404, 500 না)
const getDoctorByEmail = async (email: string) => {
	const doctor = await prisma.doctor.findUnique({
		where: { email },
		select: { id: true, isDeleted: true },
	});

	if (!doctor || doctor.isDeleted) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
	}

	return doctor;
};

// সব id সত্যিই আছে কিনা + ভবিষ্যতের slot কিনা
const ensureSchedulesAvailable = async (scheduleIds: string[]) => {
	if (scheduleIds.length === 0) return;

	const schedules = await prisma.schedule.findMany({
		where: { id: { in: scheduleIds }, startDateTime: { gt: new Date() } },
		select: { id: true },
	});

	if (schedules.length !== scheduleIds.length) {
		const found = new Set(schedules.map((s) => s.id));
		const invalidIds = scheduleIds.filter((id) => !found.has(id));
		throw new AppError(httpStatus.BAD_REQUEST, `Invalid or past schedule id(s): ${invalidIds.join(", ")}`);
	}
};
```

| Helper | কেন |
|---|---|
| `getDoctorByEmail` | `req.user`-এ শুধু `userId`/`email` থাকে, `doctorId` না। `Doctor.email` `@unique`, তাই `findUnique` চলে। |
| `ensureSchedulesAvailable` | ভুল id দিলে FK error (`P2003` → `500`) না হয়ে `400`; আর অতীতের slot নেওয়া যাবে না। কোন id ভুল, message-এ সেটাও বলে। |

- `startDateTime: { gt: new Date() }` → `new Date()` এখনকার সময় (UTC), DB-ও UTC — তাই timezone নিয়ে চিন্তা নেই।

### খ) `createMyDoctorSchedule`

```ts
const createMyDoctorSchedule = async (user: IRequestUser, payload: ICreateDoctorSchedulePayload) => {
	const doctor = await getDoctorByEmail(user.email);
	await ensureSchedulesAvailable(payload.scheduleIds);

	const data: Prisma.DoctorSchedulesCreateManyInput[] = payload.scheduleIds.map((scheduleId) => ({
		doctorId: doctor.id,
		scheduleId,
	}));

	// আগে থেকে নেওয়া slot হলে error না — skip
	await prisma.doctorSchedules.createMany({ data, skipDuplicates: true });

	return prisma.doctorSchedules.findMany({
		where: { doctorId: doctor.id, scheduleId: { in: payload.scheduleIds } },
		include: { schedule: true },
		orderBy: { schedule: { startDateTime: "asc" } },
	});
};
```

- `doctorId` body থেকে না, **login করা user থেকে** — কেউ অন্য doctor-এর নামে slot নিতে পারবে না।
- `createMany` তৈরি হওয়া row ফেরত দেয় না, তাই পরে `findMany` দিয়ে slot-সহ ফেরত (আগে থেকে নেওয়াগুলোও আসবে)।

### গ) `getMyDoctorSchedules` — doctor নিজের slot দেখে

```ts
const getMyDoctorSchedules = async (user: IRequestUser, query: IQueryParams) => {
	const doctor = await getDoctorByEmail(user.email);

	return (
		new QueryBuilder<DoctorSchedules, Prisma.DoctorSchedulesWhereInput, Prisma.DoctorSchedulesInclude>(
			prisma.doctorSchedules,
			query,
			{
				filterableFields: doctorScheduleFilterableFields,
				sortableFields: doctorScheduleSortableFields,
				tieBreakerFields: DOCTOR_SCHEDULE_TIE_BREAKERS,
				defaultSortBy: "schedule.startDateTime",
			},
		)
			// code থেকে fixed condition — query দিয়ে override করা যাবে না
			.where({ doctorId: doctor.id })
			.filter()
			.sort()
			.paginate()
			.include({ schedule: true })
			.execute()
	);
};
```

- `?doctorId=<অন্য-doctor>` দিলেও `AND`-এ নিজের `doctorId`-ও থাকে → ফল খালি, অন্যের data দেখা যায় না।

### ঘ) `getAllDoctorSchedules` — admin সবার slot দেখে

```ts
const getAllDoctorSchedules = async (query: IQueryParams) => {
	return new QueryBuilder<DoctorSchedules, Prisma.DoctorSchedulesWhereInput, Prisma.DoctorSchedulesInclude>(
		prisma.doctorSchedules,
		query,
		{
			searchableFields: doctorScheduleSearchableFields,
			filterableFields: doctorScheduleFilterableFields,
			sortableFields: doctorScheduleSortableFields,
			tieBreakerFields: DOCTOR_SCHEDULE_TIE_BREAKERS,
			defaultSortBy: "schedule.startDateTime",
		},
	)
		.search()
		.filter()
		.sort()
		.paginate()
		.dynamicInclude(doctorScheduleIncludeConfig, ["schedule"])
		.execute();
};
```

- `dynamicInclude(..., ["schedule"])` → `schedule` সবসময় আসে, `?include=doctor` দিলে doctor-ও।

### ঙ) `getDoctorScheduleById` — composite key দিয়ে একটা

```ts
const getDoctorScheduleById = async (doctorId: string, scheduleId: string) => {
	const doctorSchedule = await prisma.doctorSchedules.findUnique({
		where: { doctorId_scheduleId: { doctorId, scheduleId } },
		include: { schedule: true, doctor: true },
	});

	if (!doctorSchedule) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor schedule not found");
	}

	return doctorSchedule;
};
```

- `doctorId_scheduleId` → `@@id([doctorId, scheduleId])` থেকে Prisma এই নাম বানায়। দুটো মিলিয়েই একটা row।

### চ) `updateMyDoctorSchedule` — একসাথে কিছু যোগ, কিছু বাদ

```ts
const updateMyDoctorSchedule = async (user: IRequestUser, payload: IUpdateDoctorSchedulePayload) => {
	const doctor = await getDoctorByEmail(user.email);

	const deleteIds = payload.scheduleIds.filter((s) => s.shouldDelete).map((s) => s.id);
	const createIds = payload.scheduleIds.filter((s) => !s.shouldDelete).map((s) => s.id);

	await ensureSchedulesAvailable(createIds);

	return prisma.$transaction(async (tx) => {
		if (deleteIds.length > 0) {
			const booked = await tx.doctorSchedules.findMany({
				where: { doctorId: doctor.id, scheduleId: { in: deleteIds }, isBooked: true },
				select: { scheduleId: true },
			});

			// throw করলে পুরো transaction rollback
			if (booked.length > 0) {
				throw new AppError(
					httpStatus.BAD_REQUEST,
					`Booked schedule(s) cannot be removed: ${booked.map((b) => b.scheduleId).join(", ")}`,
				);
			}
		}

		const deleted = await tx.doctorSchedules.deleteMany({
			where: { doctorId: doctor.id, isBooked: false, scheduleId: { in: deleteIds } },
		});

		const created = await tx.doctorSchedules.createMany({
			data: createIds.map((scheduleId) => ({ doctorId: doctor.id, scheduleId })),
			skipDuplicates: true,
		});

		return { deletedCount: deleted.count, createdCount: created.count };
	});
};
```

```
scheduleIds: [{ id: A, shouldDelete: true }, { id: B, shouldDelete: false }]
→ deleteIds = [A], createIds = [B]
→ transaction: booked check → A মোছা → B যোগ → { deletedCount: 1, createdCount: 1 }
```

- `$transaction` → booked slot থাকলে throw, তখন delete-ও হয় না, create-ও হয় না (সব বা কিছুই না)।
- `deleteMany`-তে আবারও `isBooked: false` — check-এর পরে কেউ book করলেও booked row মুছবে না।

### ছ) `deleteMyDoctorSchedule`

```ts
const deleteMyDoctorSchedule = async (scheduleId: string, user: IRequestUser) => {
	const doctor = await getDoctorByEmail(user.email);

	const doctorSchedule = await prisma.doctorSchedules.findUnique({
		where: { doctorId_scheduleId: { doctorId: doctor.id, scheduleId } },
		select: { isBooked: true },
	});

	if (!doctorSchedule) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor schedule not found");
	}

	if (doctorSchedule.isBooked) {
		throw new AppError(httpStatus.BAD_REQUEST, "Booked schedule cannot be deleted");
	}

	// check আর delete-এর মাঝে কেউ book করে ফেললে count 0 হবে
	const { count } = await prisma.doctorSchedules.deleteMany({
		where: { doctorId: doctor.id, scheduleId, isBooked: false },
	});

	if (count === 0) {
		throw new AppError(httpStatus.CONFLICT, "Schedule was just booked and cannot be deleted");
	}
};

export const DoctorScheduleService = {
	createMyDoctorSchedule,
	getMyDoctorSchedules,
	getAllDoctorSchedules,
	getDoctorScheduleById,
	updateMyDoctorSchedule,
	deleteMyDoctorSchedule,
};
```

- `delete` না, `deleteMany` — কারণ `deleteMany`-তে `isBooked: false` শর্ত দেওয়া যায়, আর না মিললে error না দিয়ে `count: 0` দেয়।

---

## ধাপ ৭: `doctorSchedule.controller.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { DoctorScheduleService } from "./doctorSchedule.service";

// req.user-এর type optional — একবার check করে নিশ্চিত (narrow) করা হয়
const getAuthUser = (req: Request): IRequestUser => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}
	return req.user;
};

const createMyDoctorSchedule = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorScheduleService.createMyDoctorSchedule(getAuthUser(req), req.body);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.CREATED,
		message: "Doctor schedule created successfully",
		data: result,
	});
});

const getMyDoctorSchedules = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorScheduleService.getMyDoctorSchedules(getAuthUser(req), req.query);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Doctor schedules retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});

const getAllDoctorSchedules = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorScheduleService.getAllDoctorSchedules(req.query);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "All doctor schedules retrieved successfully",
		data: result.data,
		meta: result.meta,
	});
});

const getDoctorScheduleById = catchAsync(async (req: Request, res: Response) => {
	const { doctorId, scheduleId } = req.params;
	const result = await DoctorScheduleService.getDoctorScheduleById(doctorId as string, scheduleId as string);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Doctor schedule retrieved successfully",
		data: result,
	});
});

const updateMyDoctorSchedule = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorScheduleService.updateMyDoctorSchedule(getAuthUser(req), req.body);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Doctor schedule updated successfully",
		data: result,
	});
});

const deleteMyDoctorSchedule = catchAsync(async (req: Request, res: Response) => {
	const { scheduleId } = req.params;
	await DoctorScheduleService.deleteMyDoctorSchedule(scheduleId as string, getAuthUser(req));

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Doctor schedule deleted successfully",
	});
});

export const DoctorScheduleController = {
	createMyDoctorSchedule,
	getMyDoctorSchedules,
	getAllDoctorSchedules,
	getDoctorScheduleById,
	updateMyDoctorSchedule,
	deleteMyDoctorSchedule,
};
```

- `getAuthUser` → `checkAuth` আগেই `req.user` বসায়, কিন্তু type-এ `user?` optional (`express.d.ts`)। প্রতিবার `req.user!` না লিখে একটা helper।

---

## ধাপ ৮: `doctorSchedule.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { DoctorScheduleController } from "./doctorSchedule.controller";
import { DoctorScheduleValidation } from "./doctorSchedule.validation";

const router = Router();

router.post(
	"/create-my-doctor-schedule",
	checkAuth(Role.DOCTOR),
	validateRequest(DoctorScheduleValidation.createDoctorScheduleZodSchema),
	DoctorScheduleController.createMyDoctorSchedule,
);

router.get("/my-doctor-schedules", checkAuth(Role.DOCTOR), DoctorScheduleController.getMyDoctorSchedules);

router.get("/", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), DoctorScheduleController.getAllDoctorSchedules);

router.get(
	"/:doctorId/schedule/:scheduleId",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	DoctorScheduleController.getDoctorScheduleById,
);

router.patch(
	"/update-my-doctor-schedule",
	checkAuth(Role.DOCTOR),
	validateRequest(DoctorScheduleValidation.updateDoctorScheduleZodSchema),
	DoctorScheduleController.updateMyDoctorSchedule,
);

router.delete(
	"/delete-my-doctor-schedule/:scheduleId",
	checkAuth(Role.DOCTOR),
	DoctorScheduleController.deleteMyDoctorSchedule,
);

export const DoctorScheduleRoutes = router;
```

| Method | URL | কে | কাজ |
|---|---|---|---|
| `POST` | `/create-my-doctor-schedule` | DOCTOR | slot নেওয়া |
| `GET` | `/my-doctor-schedules` | DOCTOR | নিজের slot |
| `GET` | `/` | ADMIN, SUPER_ADMIN | সবার slot |
| `GET` | `/:doctorId/schedule/:scheduleId` | ADMIN, SUPER_ADMIN | একটা |
| `PATCH` | `/update-my-doctor-schedule` | DOCTOR | যোগ + বাদ একসাথে |
| `DELETE` | `/delete-my-doctor-schedule/:scheduleId` | DOCTOR | একটা বাদ |

- `/my-doctor-schedules` route `/:doctorId/...`-এর **আগে** — নাহলে Express `my-doctor-schedules`-কে `doctorId` ভাবত (যদিও এখানে path-এর গঠন আলাদা, তবু নির্দিষ্ট route আগে রাখা ভালো অভ্যাস)।

---

## ধাপ ৯: `src/app/routes/index.ts` — দুটো লাইন যোগ

```ts
import { DoctorScheduleRoutes } from "../module/doctorSchedule/doctorSchedule.route";
// ...
router.use("/doctor-schedules", DoctorScheduleRoutes);
```

---

## Test (Postman)

Base: `http://localhost:5000/api/v1/doctor-schedules`

### প্রস্তুতি

1. **ADMIN** login → `POST /api/v1/schedules` দিয়ে ভবিষ্যতের কিছু slot বানাও ([নোট ২৫](./25-schedule.md)):

	```json
	{ "startDate": "2026-11-10", "endDate": "2026-11-10", "startTime": "09:00", "endTime": "11:00" }
	```

2. **DOCTOR** login → `GET /api/v1/schedules?startDate=2026-11-10&endDate=2026-11-10&sortOrder=asc` → ৪টা slot-এর `id` কপি করো (`A`, `B`, `C`, `D`)।

### ১. Slot নেওয়া — `POST /create-my-doctor-schedule` (DOCTOR)

```json
{
	"scheduleIds": ["<A>", "<B>", "<A>"]
}
```

→ `201`, `data`-তে ২টা row (A, B — duplicate A বাদ), প্রতিটায় `schedule` আর `isBooked: false`।

| Body | ফল |
|---|---|
| `{ "scheduleIds": [] }` | `400` At least one schedule id is required |
| `{ "scheduleIds": ["abc"] }` | `400` Each schedule id must be a valid UUID |
| অন্য কোনো table-এর uuid | `400` Invalid or past schedule id(s): ... |
| অতীতের slot-এর id | `400` Invalid or past schedule id(s): ... |
| একই body আবার | `201`, একই ২টা row (নতুন তৈরি হয়নি) |

### ২. নিজের slot — `GET /my-doctor-schedules` (DOCTOR)

```
http://localhost:5000/api/v1/doctor-schedules/my-doctor-schedules
http://localhost:5000/api/v1/doctor-schedules/my-doctor-schedules?isBooked=false
http://localhost:5000/api/v1/doctor-schedules/my-doctor-schedules?sortBy=schedule.startDateTime&sortOrder=asc&page=1&limit=1
http://localhost:5000/api/v1/doctor-schedules/my-doctor-schedules?doctorId=<অন্য-doctor-id>
```

| URL | যা দেখার কথা |
|---|---|
| প্রথমটা | `meta.total: 2`, প্রতিটায় `schedule` |
| `?limit=1` | `meta.totalPages: 2`, A আগে |
| `?doctorId=<অন্য>` | `data: []` (নিজের doctorId fixed) |

### ৩. যোগ + বাদ একসাথে — `PATCH /update-my-doctor-schedule` (DOCTOR)

```json
{
	"scheduleIds": [
		{ "id": "<A>", "shouldDelete": true },
		{ "id": "<C>", "shouldDelete": false },
		{ "id": "<D>", "shouldDelete": false }
	]
}
```

→ `200`, `{ "deletedCount": 1, "createdCount": 2 }`। এখন নিজের slot: B, C, D।

| Body | ফল |
|---|---|
| একই id দুবার (`A` true, `A` false) | `400` Duplicate schedule id found |
| `"shouldDelete": "yes"` | `400` shouldDelete must be true or false |
| Booked slot মুছতে চাওয়া | `400` Booked schedule(s) cannot be removed (কিছুই বদলায় না) |

### ৪. একটা বাদ — `DELETE /delete-my-doctor-schedule/:scheduleId` (DOCTOR)

```
http://localhost:5000/api/v1/doctor-schedules/delete-my-doctor-schedule/<B>
```

→ `200`। আবার একই → `404 Doctor schedule not found`।

### ৫. Admin — `GET /` আর `GET /:doctorId/schedule/:scheduleId`

ADMIN login করে:

```
http://localhost:5000/api/v1/doctor-schedules
http://localhost:5000/api/v1/doctor-schedules?searchTerm=<doctor-এর নামের অংশ>
http://localhost:5000/api/v1/doctor-schedules?include=doctor
http://localhost:5000/api/v1/doctor-schedules?doctorId=<doctor-id>&isBooked=false
http://localhost:5000/api/v1/doctor-schedules/<doctor-id>/schedule/<C>
```

| URL | যা দেখার কথা |
|---|---|
| `/` | সব doctor-এর slot, প্রতিটায় `schedule` |
| `?include=doctor` | `doctor` + তার `specialties` |
| `/<doctor-id>/schedule/<C>` | `200`, `schedule` + `doctor` |
| `/<doctor-id>/schedule/<B>` (মোছা) | `404` |

> `doctor-id` পাবে `GET /api/v1/doctors` বা create-এর response-এর `doctorId` থেকে।

### ৬. Role test

| Login | Request | ফল |
|---|---|---|
| ADMIN | `POST /create-my-doctor-schedule` | `403` |
| DOCTOR | `GET /` | `403` |
| PATIENT | `GET /my-doctor-schedules` | `403` |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Admin list-এ deleted doctor-এর slot-ও আসে।** `getAllDoctorSchedules`-এ `isDeleted` check নেই, তাই soft-delete করা doctor-এর slot-ও দেখা যায়:

	```ts
	.where({ doctor: { isDeleted: false } })
	.search()
	```

2. **Interface হাতে লেখা।** [নোট ২৫](./25-schedule.md)-এর schedule module-এ `z.infer` ব্যবহার হয়েছে — এখানেও একই রকম রাখলে schema বদলালে type নিজেই বদলাবে:

	```ts
	import type * as z from "zod";
	import type { DoctorScheduleValidation } from "./doctorSchedule.validation";

	export type ICreateDoctorSchedulePayload = z.infer<typeof DoctorScheduleValidation.createDoctorScheduleZodSchema>;
	export type IUpdateDoctorSchedulePayload = z.infer<typeof DoctorScheduleValidation.updateDoctorScheduleZodSchema>;
	```

3. **Booked slot মোছার status code মেলে না।** `updateMyDoctorSchedule` আর `deleteMyDoctorSchedule`-এর প্রথম check `400` দেয়, কিন্তু race-এর ক্ষেত্রে `409`। "অবস্থার কারণে করা যাচ্ছে না" — দুটোই `409 CONFLICT` হলে বেশি পরিষ্কার (schedule module-এও তাই)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **Date দিয়ে filter করতে পুরো ISO দিতে হয়।** `?schedule.startDateTime[gte]=2026-11-10` দিলে Prisma "ISO-8601 DateTime" চায় → error ([নোট ২৪](./24-query-builder.md)-এর ধাপ ৯ থাকলে `400`, না থাকলে `500`)। পুরো সময় দাও, আর `+` URL-এ space হয়ে যায় বলে `Z` ব্যবহার করো:

	```
	?schedule.startDateTime[gte]=2026-11-09T18:00:00Z   (= ঢাকা ১০ নভেম্বর 00:00)
	```

5. **নিজের list-এ অতীতের slot-ও আসে।** Doctor সাধারণত সামনের slot দেখতে চায়। Frontend থেকে উপরের মতো `[gte]` দিয়ে পাঠাও, বা service-এ default করে দাও।
6. **Project-এর `QueryBuilder.ts`-এ comment দুবার।** `sort()`-এর উপরে `// ?sortBy=appointmentFee&sortOrder=asc` লাইনটা দুবার আছে — একটা মুছে দিও।
7. **Validation আর transaction-এর মাঝে schedule মুছে গেলে।** `ensureSchedulesAvailable` transaction-এর বাইরে চলে। এর মধ্যে admin ওই schedule মুছলে `createMany` FK error (`P2003`) দেবে → `500`। খুবই বিরল।

## 🔜 পরে যা যোগ হবে

- [ ] Appointment booking → `isBooked: true`
- [ ] Patient-এর জন্য public route: কোনো doctor-এর ফাঁকা (`isBooked: false`) ভবিষ্যতের slot
