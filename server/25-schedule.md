# 25 - Schedule (৩০ মিনিটের time slot তৈরি)

```
Admin: POST /schedules { startDate, endDate, startTime, endTime }
→ প্রতিটা দিন loop → startTime থেকে endTime পর্যন্ত ৩০ মিনিট করে slot (ঢাকার সময়, +06:00)
→ createManyAndReturn (duplicate skip) → DB-তে UTC হিসেবে save
```

Teacher-এর মতো route, controller, validation file আলাদা করে রাখা হয়েছে।

> নিচে code-এর পরে **Overview** অংশে বোঝানো আছে — schedule কেন, কীভাবে, আর date-time দিয়ে slot কীভাবে বানানো হয়।

---

## ধাপ ১: `prisma/schema/schedule.prisma`

```prisma
model Schedule {
  id              String            @id @default(uuid(7))
  startDateTime   DateTime
  endDateTime     DateTime
  createdAt       DateTime          @default(now())
  updatedAt       DateTime          @updatedAt
  doctorSchedules DoctorSchedules[]
  appointments    Appointment[]

  @@unique([startDateTime, endDateTime])
  @@map("schedules")
}
```

- `@@unique([startDateTime, endDateTime])` → একই সময়ের slot দুবার হবে না। একটা slot সব doctor share করে (`DoctorSchedules` দিয়ে)।
- `uuid(7)` → সময় অনুযায়ী সাজানো uuid।

> ✏️ নোটে ছিল না — একই file-এর `DoctorSchedules` model (project থেকে), যেটা doctor আর schedule-কে যুক্ত করে:

```prisma
model DoctorSchedules {
  doctorId String
  doctor   Doctor @relation(fields: [doctorId], references: [id], onDelete: Cascade, onUpdate: Cascade)

  scheduleId String
  schedule   Schedule @relation(fields: [scheduleId], references: [id], onDelete: Cascade, onUpdate: Cascade)

  isBooked  Boolean  @default(false)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@id([doctorId, scheduleId])
  @@index([scheduleId], name: "idx_doctor_schedules_scheduleId")
  @@map("doctor_schedules")
}
```

> ✏️ Schema বদলালে migration চালাতে হবে — এটা নোটে ছিল না:

```shell
pnpm migrate
pnpm generate
```

---

## ধাপ ২: `src/app/module/schedule/schedule.validation.ts`

> ✏️ নোটে এই code-এর উপরে file-এর নাম `src/app/routes/index.ts` লেখা ছিল — আসলে এটা `schedule.validation.ts`।

```ts
import * as z from "zod";

// ---------- বারবার ব্যবহারের field schema ----------
const dateString = z.iso.date("Date must be in YYYY-MM-DD format");
const timeString = z.iso.time({ precision: -1, error: "Time must be in HH:mm format (e.g. 09:30)" });

// ---------- Base shape ----------
const scheduleBaseSchema = z.object({
	startDate: dateString,
	endDate: dateString,
	startTime: timeString,
	endTime: timeString,
});

// ---------- Create (একটা range → অনেকগুলো slot) ----------
const createScheduleZodSchema = scheduleBaseSchema
	.refine((data) => data.endDate >= data.startDate, {
		message: "End date must be the same as or after start date",
		path: ["endDate"],
	})
	.refine((data) => data.endTime > data.startTime, {
		message: "End time must be after start time",
		path: ["endTime"],
	});

// ---------- Update (একটা slot-এর নতুন সময়) ----------
// teacher-এর service চারটা field-ই ব্যবহার করে, তাই partial না — সব required
const updateScheduleZodSchema = scheduleBaseSchema.refine(
	(data) => `${data.endDate}T${data.endTime}` > `${data.startDate}T${data.startTime}`,
	{
		message: "End date-time must be after start date-time",
		path: ["endTime"],
	},
);

// ---------- Get all (query-র date filter) ----------
const scheduleDateQueryZodSchema = z
	.object({
		startDate: dateString.optional(),
		endDate: dateString.optional(),
	})
	.refine((data) => !data.startDate || !data.endDate || data.endDate >= data.startDate, {
		message: "End date must be the same as or after start date",
		path: ["endDate"],
	});

export const ScheduleValidation = {
	createScheduleZodSchema,
	updateScheduleZodSchema,
	scheduleDateQueryZodSchema,
};
```

| Schema | কী যাচাই করে |
|---|---|
| `dateString` | `z.iso.date` → শুধু `YYYY-MM-DD` (যেমন `2026-11-01`) |
| `timeString` | `z.iso.time({ precision: -1 })` → শুধু `HH:mm` (seconds ছাড়া, যেমন `09:30`; `9:30` চলবে না) |
| Create | `endDate ≥ startDate`, `endTime > startTime` (প্রতিদিন একই সময়) |
| Update | পুরো `date + time` মিলিয়ে end > start (একটা slot রাতের পর দিন পার হতে পারে) |
| Date query | `?startDate` / `?endDate` optional, দুটো থাকলে end ≥ start |

- `"2026-11-01" < "2026-11-05"` — string compare-ই কাজ করে, কারণ `YYYY-MM-DD` / `HH:mm` format-এ বাম থেকে বড় একক আগে আসে (বছর → মাস → দিন)। Zod format নিশ্চিত করে, তাই নিরাপদ।

---

## ধাপ ৩: `schedule.interface.ts`

```ts
import type * as z from "zod";
import type { ScheduleValidation } from "./schedule.validation";

export type ICreateSchedulePayload = z.infer<typeof ScheduleValidation.createScheduleZodSchema>;
export type IUpdateSchedulePayload = z.infer<typeof ScheduleValidation.updateScheduleZodSchema>;
```

- Interface হাতে না লিখে Zod schema থেকে `z.infer` — schema বদলালে type নিজেই বদলায়।

---

## ধাপ ৪: `schedule.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { ScheduleController } from "./schedule.controller";
import { ScheduleValidation } from "./schedule.validation";

const router = Router();

router.post(
	"/",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(ScheduleValidation.createScheduleZodSchema),
	ScheduleController.createSchedule,
);

router.get("/", checkAuth(Role.ADMIN, Role.SUPER_ADMIN, Role.DOCTOR), ScheduleController.getAllSchedules);

router.get("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN, Role.DOCTOR), ScheduleController.getScheduleById);

router.patch(
	"/:id",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	validateRequest(ScheduleValidation.updateScheduleZodSchema),
	ScheduleController.updateSchedule,
);

router.delete("/:id", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), ScheduleController.deleteSchedule);

export const ScheduleRoutes = router;
```

| Method | URL | কে |
|---|---|---|
| `POST` | `/schedules` | ADMIN, SUPER_ADMIN |
| `GET` | `/schedules` | ADMIN, SUPER_ADMIN, DOCTOR |
| `GET` | `/schedules/:id` | ADMIN, SUPER_ADMIN, DOCTOR |
| `PATCH` | `/schedules/:id` | ADMIN, SUPER_ADMIN |
| `DELETE` | `/schedules/:id` | ADMIN, SUPER_ADMIN |

- Doctor শুধু দেখতে পারে (পরে নিজের জন্য slot বেছে নেবে)। Slot বানানো/বদলানো শুধু admin।

---

## ধাপ ৫: `src/app/routes/index.ts`-এ mount

> ✏️ এই ধাপ নোটে ছিল না।

```ts
import { ScheduleRoutes } from "../module/schedule/schedule.route";

router.use("/schedules", ScheduleRoutes);
```

> ⚠️ নাম হুবহু `ScheduleRoutes` (বড় `S`) — route file এই নামেই export করে। (Recommendation ১ দেখো।)

---

## ধাপ ৬: `schedule.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { ScheduleService } from "./schedule.service";

const createSchedule = catchAsync(async (req: Request, res: Response) => {
	const result = await ScheduleService.createSchedule(req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.CREATED,
		success: true,
		message: "Schedule created successfully",
		data: result,
	});
});

const getAllSchedules = catchAsync(async (req: Request, res: Response) => {
	const result = await ScheduleService.getAllSchedules(req.query);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Schedules fetched successfully",
		meta: result.meta,
		data: result.data,
	});
});

const getScheduleById = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	const result = await ScheduleService.getScheduleById(id as string);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Schedule fetched successfully",
		data: result,
	});
});

const updateSchedule = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	const result = await ScheduleService.updateSchedule(id as string, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Schedule updated successfully",
		data: result,
	});
});

const deleteSchedule = catchAsync(async (req: Request, res: Response) => {
	const { id } = req.params;

	await ScheduleService.deleteSchedule(id as string);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Schedule deleted successfully",
	});
});

export const ScheduleController = {
	createSchedule,
	getAllSchedules,
	getScheduleById,
	updateSchedule,
	deleteSchedule,
};
```

- `id as string` → `noUncheckedIndexedAccess` চালু থাকায় `req.params.id`-এর type `string | undefined`, তাই cast।

---

## ধাপ ৭: `schedule.constant.ts`

```ts
import type { Prisma } from "@/generated/prisma/client";

// startDate / endDate filter service-এ আলাদা করে handle হয় (ঢাকার সময় অনুযায়ী)
// ?doctorSchedules.doctor.id=<uuid> → ওই doctor-এর নেওয়া slot-গুলো
export const scheduleFilterableFields = ["doctorSchedules.doctor.id"] as const;

export const scheduleSortableFields = ["startDateTime", "endDateTime", "createdAt"] as const;

export const scheduleSelectableFields = ["id", "startDateTime", "endDateTime", "createdAt"] as const;

// appointments ইচ্ছা করে রাখা হয়নি — রোগীর data appointment module থেকে আসবে
export const scheduleIncludeConfig = {
	doctorSchedules: { include: { doctor: true } },
} satisfies Prisma.ScheduleInclude;

export const SCHEDULE_INTERVAL_MINUTES = 30;

// Asia/Dhaka — Bangladesh-এ DST নেই, তাই fixed offset নিরাপদ
export const CLINIC_UTC_OFFSET = "+06:00";
```

- `doctorSchedules.doctor.id` → ৩ ভাগের path, তাই QueryBuilder `some` দিয়ে wrap করে ([নোট ২৪](./24-query-builder.md)-এর Path rule)।
- `startDate` / `endDate` filterable-এ নেই — কারণ DB-তে ওই নামে column নেই, service নিজে handle করে।

---

## ধাপ ৮: `schedule.utils.ts` (date-time helper)

```ts
import { CLINIC_UTC_OFFSET } from "./schedule.constant";

// "2026-11-01" + "09:00" → ঢাকার সকাল 9টা (যেকোনো server-এ একই ফল)
export const buildDateTime = (date: string, time: string): Date => {
	return new Date(`${date}T${time}:00${CLINIC_UTC_OFFSET}`);
};

export const addMinutes = (date: Date, minutes: number): Date => {
	return new Date(date.getTime() + minutes * 60_000);
};

// "2026-11-01" → "2026-11-02" (UTC-তে হিসাব, তাই timezone-এর প্রভাব নেই)
export const addDaysToDateString = (date: string, days: number): string => {
	const d = new Date(`${date}T00:00:00Z`);
	d.setUTCDate(d.getUTCDate() + days);
	return d.toISOString().slice(0, 10);
};
```

| Function | Input | Output |
|---|---|---|
| `buildDateTime` | `"2026-11-01"`, `"09:00"` | `2026-11-01T03:00:00.000Z` (= ঢাকা 9:00) |
| `addMinutes` | `03:00Z`, `30` | `03:30Z` |
| `addDaysToDateString` | `"2026-11-30"`, `1` | `"2026-12-01"` |
| | `"2026-12-31"`, `1` | `"2027-01-01"` |
| | `"2028-02-28"`, `1` | `"2028-02-29"` (leap year) |

> উপরের সব output node দিয়ে চালিয়ে মিলিয়ে দেখা।

Detail ব্যাখ্যা নিচে **Overview**-এ।

---

## ধাপ ৯: `schedule.service.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IQueryParams } from "@/app/interfaces/query.interface";
import { prisma } from "@/app/lib/prisma";
import { QueryBuilder } from "@/app/utils/QueryBuilder";
import type { Prisma, Schedule } from "@/generated/prisma/client";
import httpStatus from "http-status";
import {
	SCHEDULE_INTERVAL_MINUTES,
	scheduleFilterableFields,
	scheduleIncludeConfig,
	scheduleSelectableFields,
	scheduleSortableFields,
} from "./schedule.constant";
import type { ICreateSchedulePayload, IUpdateSchedulePayload } from "./schedule.interface";
import { addDaysToDateString, addMinutes, buildDateTime } from "./schedule.utils";
import { ScheduleValidation } from "./schedule.validation";
```

### ক) `createSchedule` — range থেকে slot বানানো

```ts
const createSchedule = async (payload: ICreateSchedulePayload) => {
	const { startDate, endDate, startTime, endTime } = payload;

	const slots: Prisma.ScheduleCreateManyInput[] = [];

	// "YYYY-MM-DD" string compare নিরাপদ (Zod format নিশ্চিত করেছে)
	for (let date = startDate; date <= endDate; date = addDaysToDateString(date, 1)) {
		const dayEnd = buildDateTime(date, endTime);

		let slotStart = buildDateTime(date, startTime);
		let slotEnd = addMinutes(slotStart, SCHEDULE_INTERVAL_MINUTES);

		while (slotEnd <= dayEnd) {
			slots.push({ startDateTime: slotStart, endDateTime: slotEnd });
			slotStart = slotEnd;
			slotEnd = addMinutes(slotStart, SCHEDULE_INTERVAL_MINUTES);
		}
	}

	if (slots.length === 0) {
		throw new AppError(httpStatus.BAD_REQUEST, `Time range must be at least ${SCHEDULE_INTERVAL_MINUTES} minutes`);
	}

	// ১টা query: duplicate হলে skip, শুধু নতুন তৈরি হওয়াগুলো return
	return prisma.schedule.createManyAndReturn({
		data: slots,
		skipDuplicates: true,
	});
};
```

- `createManyAndReturn` → সব slot **একটা query**-তে insert আর তৈরি হওয়া row ফেরত।
- `skipDuplicates: true` → `@@unique` ভাঙলে error না দিয়ে ওই row বাদ দেয়। তাই একই range দুবার পাঠালে দ্বিতীয়বার `data: []`।

### খ) `getAllSchedules` — ঢাকার দিন অনুযায়ী filter + QueryBuilder

```ts
const getAllSchedules = async (query: IQueryParams) => {
	// invalid হলে ZodError → globalErrorHandler → 400
	const { startDate, endDate } = ScheduleValidation.scheduleDateQueryZodSchema.parse(query);

	const dateRange: Prisma.ScheduleWhereInput = {};

	if (startDate) {
		// ঢাকার ওই দিনের রাত 12টা থেকে
		dateRange.startDateTime = { gte: buildDateTime(startDate, "00:00") };
	}

	if (endDate) {
		// endDate দিনটাও ধরা হবে → পরের দিনের রাত 12টা পর্যন্ত
		dateRange.endDateTime = { lte: buildDateTime(addDaysToDateString(endDate, 1), "00:00") };
	}

	const result = await new QueryBuilder<Schedule, Prisma.ScheduleWhereInput, Prisma.ScheduleInclude>(
		prisma.schedule,
		query,
		{
			filterableFields: scheduleFilterableFields,
			sortableFields: scheduleSortableFields,
			selectableFields: scheduleSelectableFields,
			defaultSortBy: "startDateTime",
		},
	)
		.where(dateRange)
		.filter()
		.sort()
		.paginate()
		.fields()
		.dynamicInclude(scheduleIncludeConfig)
		.execute();

	return result;
};
```

- Route-এ `validateRequest` নেই (ওটা `req.body` দেখে) — তাই service-এ `.parse(query)` দিয়ে query যাচাই। `parse` fail করলে `ZodError` throw → `catchAsync` → `globalErrorHandler` → `400`।
- `search()` নেই — schedule-এ খোঁজার মতো text field নেই।

### গ) `getScheduleById`

```ts
const getScheduleById = async (id: string) => {
	const schedule = await prisma.schedule.findUnique({
		where: { id },
		include: scheduleIncludeConfig,
	});

	if (!schedule) {
		throw new AppError(httpStatus.NOT_FOUND, "Schedule not found");
	}

	return schedule;
};
```

### ঘ) `updateSchedule`

```ts
const updateSchedule = async (id: string, payload: IUpdateSchedulePayload) => {
	const { startDate, endDate, startTime, endTime } = payload;

	const schedule = await prisma.schedule.findUnique({
		where: { id },
		include: { _count: { select: { appointments: true } } },
	});

	if (!schedule) {
		throw new AppError(httpStatus.NOT_FOUND, "Schedule not found");
	}

	// booked slot-এর সময় বদলালে রোগীর appointment-ও চুপচাপ বদলে যেত
	if (schedule._count.appointments > 0) {
		throw new AppError(httpStatus.CONFLICT, "Cannot update a schedule that already has appointments");
	}

	// date-fns-এর বদলে নিজের util — সবসময় ঢাকার সময়
	const startDateTime = buildDateTime(startDate, startTime);
	const endDateTime = buildDateTime(endDate, endTime);

	// @@unique([startDateTime, endDateTime]) ভাঙলে Prisma P2002 → 500, তাই আগেই check
	const duplicate = await prisma.schedule.findFirst({
		where: { startDateTime, endDateTime, NOT: { id } },
	});

	if (duplicate) {
		throw new AppError(httpStatus.CONFLICT, "A schedule already exists at this time");
	}

	return prisma.schedule.update({
		where: { id },
		data: { startDateTime, endDateTime },
	});
};
```

- `_count` → পুরো appointment list না এনে শুধু সংখ্যা আনে (হালকা query)।

### ঙ) `deleteSchedule`

```ts
const deleteSchedule = async (id: string) => {
	const schedule = await prisma.schedule.findUnique({
		where: { id },
		include: { _count: { select: { appointments: true } } },
	});

	if (!schedule) {
		throw new AppError(httpStatus.NOT_FOUND, "Schedule not found");
	}

	// Appointment-এ onDelete: Restrict — DB আটকাত, কিন্তু P2003 → 500 হতো
	if (schedule._count.appointments > 0) {
		throw new AppError(httpStatus.CONFLICT, "Cannot delete a schedule that already has appointments");
	}

	// DoctorSchedules নিজে থেকেই cascade হয়ে মুছবে
	await prisma.schedule.delete({ where: { id } });
};

export const ScheduleService = {
	createSchedule,
	getAllSchedules,
	getScheduleById,
	updateSchedule,
	deleteSchedule,
};
```

| Check | না থাকলে কী হতো |
|---|---|
| Appointment আছে কিনা (update) | রোগীর appointment-এর সময় চুপচাপ বদলে যেত |
| Duplicate সময় (update) | Prisma `P2002` → `500` |
| Appointment আছে কিনা (delete) | Prisma `P2003` (Restrict) → `500` |

---

# 📘 Overview — Schedule কেন, কীভাবে, আর date-time দিয়ে কীভাবে

## ১. কেন schedule বানাচ্ছি?

Healthcare app-এ রোগী doctor-এর কাছে **নির্দিষ্ট সময়ে** appointment নেবে। তার জন্য আগে থেকে সময়গুলো ভাগ করা থাকতে হবে।

```
Admin  → Schedule (সময়ের ঘর, যেমন ১ নভেম্বর ৯:০০–৯:৩০)
Doctor → DoctorSchedules (এই ঘরগুলো থেকে নিজের সুবিধামতো বেছে নেয়)
Patient → Appointment (doctor-এর নেওয়া কোনো একটা ঘর book করে → isBooked: true)
```

- **Schedule** হলো শুধু একটা ফাঁকা time slot — কোনো doctor-এর না। তাই `@@unique([startDateTime, endDateTime])`: ৯:০০–৯:৩০ slot একটাই থাকবে, অনেক doctor সেটা share করবে।
- এভাবে admin একবারে সারা সপ্তাহ/মাসের grid বানিয়ে দেয়, প্রতিটা doctor-কে আলাদা করে সময় লিখতে হয় না।

## ২. কীভাবে slot বানানো হয় (দুইটা loop)

Input:

```json
{ "startDate": "2026-11-01", "endDate": "2026-11-02", "startTime": "09:00", "endTime": "10:00" }
```

```
বাইরের loop: দিন    → 2026-11-01, 2026-11-02            (addDaysToDateString দিয়ে +1 দিন)
ভেতরের loop: সময়   → 09:00–09:30, 09:30–10:00          (addMinutes দিয়ে +30 মিনিট)
শর্ত: slotEnd <= dayEnd (10:00) — শেষ slot 10:00 ছাড়ালে আর বানাবে না
```

Output (node দিয়ে চালানো আসল ফল):

| দিন | ঢাকার সময় | DB-তে (UTC) |
|---|---|---|
| ১ নভেম্বর | 09:00 – 09:30 | `2026-11-01T03:00:00.000Z` – `03:30:00.000Z` |
| ১ নভেম্বর | 09:30 – 10:00 | `2026-11-01T03:30:00.000Z` – `04:00:00.000Z` |
| ২ নভেম্বর | 09:00 – 09:30 | `2026-11-02T03:00:00.000Z` – `03:30:00.000Z` |
| ২ নভেম্বর | 09:30 – 10:00 | `2026-11-02T03:30:00.000Z` – `04:00:00.000Z` |

মোট slot = দিন × (মিনিট ÷ ৩০) = ২ × (৬০ ÷ ৩০) = **৪টা**।

> `09:00`–`10:15` দিলে 10:00–10:30 slot 10:15 ছাড়িয়ে যায়, তাই বাদ — শুধু ২টা slot। `09:00`–`09:20` দিলে একটাও না → `400`।

## ৩. Date-time-এর আসল সমস্যা: timezone

**DB সবসময় UTC-তে সময় রাখে।** আর JavaScript-এর `new Date()` offset না দিলে **server যে দেশে চলছে** তার timezone ধরে নেয়।

```ts
new Date("2026-11-01T09:00:00");        // ❌ offset নেই
// তোমার Mac (ঢাকা)-এ   → ঢাকা 9:00 = 03:00Z ✅
// Render/Vercel (UTC)-এ → UTC 9:00 = ঢাকা বিকাল 3:00 ❌

new Date("2026-11-01T09:00:00+06:00");  // ✅ buildDateTime যা করে
// যেকোনো server-এ → 03:00Z = ঢাকা 9:00
```

তাই `buildDateTime` সবসময় `+06:00` (`CLINIC_UTC_OFFSET`) জুড়ে দেয়। Bangladesh-এ daylight saving (DST) নেই, তাই সারা বছর একই offset — fixed string নিরাপদ।

Response-এ `"startDateTime": "2026-11-01T03:00:00.000Z"` দেখে ঘাবড়ে যেও না — `Z` মানে UTC, +৬ ঘণ্টা যোগ করলে ঢাকা 9:00। Frontend এটাকে browser-এর timezone-এ দেখাবে।

## ৪. দিন যোগ করা কেন UTC-তে (`addDaysToDateString`)

```ts
const d = new Date(`${date}T00:00:00Z`); // UTC মধ্যরাত
d.setUTCDate(d.getUTCDate() + days);     // UTC-তে +1 দিন
return d.toISOString().slice(0, 10);     // "YYYY-MM-DD" অংশ
```

- `Z` দিয়ে বানিয়ে `setUTCDate` করলে server-এর timezone কোনো প্রভাব ফেলে না — `"2026-11-01"` থেকে সবসময় `"2026-11-02"`।
- মাস/বছর পার হওয়া, leap year — সব `Date` নিজেই সামলায় (`11-30` → `12-01`, `2028-02-28` → `02-29`)।
- Loop-এ `Date` না রেখে `"YYYY-MM-DD"` string রাখা হয়েছে, যাতে `date <= endDate` string compare দিয়েই চলে।

## ৫. `?startDate` / `?endDate` filter-এর সীমা

`?startDate=2026-11-01&endDate=2026-11-01` মানে "ঢাকার ১ নভেম্বরের সব slot"।

| শর্ত | ঢাকার সময় | UTC (DB-তে compare) |
|---|---|---|
| `startDateTime >=` | ১ নভেম্বর 00:00 | `2026-10-31T18:00:00.000Z` |
| `endDateTime <=` | ২ নভেম্বর 00:00 | `2026-11-01T18:00:00.000Z` |

- UTC-তে ঢাকার দিন শুরু হয় আগের দিন সন্ধ্যা ৬টায়। শুধু `"2026-11-01"` (UTC) ধরলে ঢাকার সকাল ৬টার আগের slot বাদ পড়ত।
- `endDate`-এর পরের দিনের মধ্যরাত পর্যন্ত নেওয়া হয়, যাতে `endDate`-এর পুরো দিনটা ধরা পড়ে।

## ৬. সংক্ষেপে

| প্রশ্ন | উত্তর |
|---|---|
| কেন? | Doctor বেছে নেবে, রোগী book করবে — তার জন্য আগে থেকে ৩০ মিনিটের ঘর |
| কে বানায়? | Admin (একবারে range দিয়ে) |
| কীভাবে? | দিনের loop × ৩০ মিনিটের loop → `createManyAndReturn` |
| সময় ঠিক রাখা? | সবসময় `+06:00` দিয়ে `Date` → DB-তে UTC |
| duplicate? | `@@unique` + `skipDuplicates` (create), আগে `findFirst` (update) |

---

## Test (Postman)

Base: `http://localhost:5000/api/v1/schedules`

**আগে:** `POST /api/v1/auth/login` দিয়ে **ADMIN / SUPER_ADMIN** login করো (cookie Postman-এ থাকবে)।

### ১. Create — `POST /schedules`

```json
{
	"startDate": "2026-11-01",
	"endDate": "2026-11-03",
	"startTime": "09:00",
	"endTime": "12:00"
}
```

→ `201`, `data`-তে **১৮টা** slot (৩ দিন × ৬টা)। প্রথমটার `startDateTime`: `"2026-11-01T03:00:00.000Z"`।

### ২. একই body আবার পাঠাও

→ `201`, `data: []` (সব duplicate, skip হয়েছে)।

### ৩. Create — ভুল input

| Body-তে | ফল |
|---|---|
| `"endTime": "09:00"` (startTime-এর সমান) | `400` End time must be after start time |
| `"endDate": "2026-10-31"` | `400` End date must be the same as or after start date |
| `"startDate": "01-11-2026"` | `400` Date must be in YYYY-MM-DD format |
| `"startTime": "9:00"` | `400` Time must be in HH:mm format |
| `"startTime": "09:00", "endTime": "09:20"` | `400` Time range must be at least 30 minutes |

### ৪. Get all — `GET /schedules`

```
http://localhost:5000/api/v1/schedules
http://localhost:5000/api/v1/schedules?startDate=2026-11-01&endDate=2026-11-01
http://localhost:5000/api/v1/schedules?startDate=2026-11-01&endDate=2026-11-03&sortBy=startDateTime&sortOrder=asc&page=1&limit=6
http://localhost:5000/api/v1/schedules?include=doctorSchedules
http://localhost:5000/api/v1/schedules?fields=id,startDateTime,endDateTime
http://localhost:5000/api/v1/schedules?doctorSchedules.doctor.id=<doctor-id>
```

| URL | যা দেখার কথা |
|---|---|
| `?startDate=2026-11-01&endDate=2026-11-01` | `meta.total: 6` |
| `?...&sortOrder=asc&limit=6` | ১ নভেম্বর 03:00Z থেকে শুরু, `meta.totalPages: 3` |
| `?include=doctorSchedules` | প্রতিটা slot-এ `doctorSchedules: []` (এখনো কোনো doctor নেয়নি) |
| `?startDate=2026-11-05&endDate=2026-11-01` | `400` (Zod) |
| `?startDate=2026/11/01` | `400` (Zod) |

> Default sort `startDateTime desc` — সবচেয়ে পরের slot আগে। আগে থেকে দেখতে `sortOrder=asc` দাও।

### ৫. Get by id — `GET /schedules/:id`

Create-এর response থেকে একটা `id` নাও।

```
http://localhost:5000/api/v1/schedules/<schedule-id>
```

→ `200`। ভুল id দিলে → `404 Schedule not found`।

### ৬. Update — `PATCH /schedules/:id`

```json
{
	"startDate": "2026-11-04",
	"endDate": "2026-11-04",
	"startTime": "15:00",
	"endTime": "15:30"
}
```

→ `200`, `startDateTime: "2026-11-04T09:00:00.000Z"` (ঢাকা 3:00 PM)।

| Case | ফল |
|---|---|
| অন্য slot-এর সময় দাও (যেমন `2026-11-01`, `09:30`–`10:00`) | `409` A schedule already exists at this time |
| `"endTime": "14:00"` | `400` End date-time must be after start date-time |
| Appointment আছে এমন slot | `409` Cannot update a schedule that already has appointments |

### ৭. Delete — `DELETE /schedules/:id`

→ `200` Schedule deleted successfully। একই id আবার → `404`।

### ৮. Role test

DOCTOR দিয়ে login করে:

| Request | ফল |
|---|---|
| `GET /schedules` | `200` |
| `POST /schedules` | `403` |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Project-এ route-এর নাম মেলে না → server চালুই হবে না।** তোমার `src/app/routes/index.ts`-এ লেখা `import { scheduleRoutes } ...` (ছোট `s`), কিন্তু `schedule.route.ts` export করে `ScheduleRoutes` (বড় `S`)। ESM-এ এটা চালু হওয়ার সময়ই `SyntaxError: ... does not provide an export named 'scheduleRoutes'` দেবে। (code পড়ে নিশ্চিত — এখান থেকে `pnpm typecheck` চালাতে পারিনি, TypeScript-এর binary এই machine-এ চলে না।)

	```ts
	import { ScheduleRoutes } from "../module/schedule/schedule.route";

	router.use("/schedules", ScheduleRoutes);
	```

### 🟡 ভালো practice

2. **একটার ওপর আরেকটা slot (overlap)।** `@@unique` শুধু হুবহু একই start+end আটকায়। আগে `09:00` থেকে বানানো থাকলে, পরে `09:10` থেকে দিলে `09:10–09:40` তৈরি হবে — যেটা `09:00–09:30` আর `09:30–10:00` দুটোর সাথে overlap করে। মিনিট শুধু `00` বা `30` রাখলে grid সবসময় মিলে যাবে:

	```ts
	const timeString = z.iso
		.time({ precision: -1, error: "Time must be in HH:mm format (e.g. 09:30)" })
		.refine((time) => time.endsWith(":00") || time.endsWith(":30"), {
			message: "Time must be on the hour or half hour (e.g. 09:00, 09:30)",
		});
	```

	এতে update-এও শুধু grid-এর সময় দেওয়া যাবে।

3. **অনেক বড় range-এ লক্ষ লক্ষ row।** `2026-01-01` থেকে `2030-12-31`, `00:00`–`23:30` দিলে ~৮৫ হাজার slot একবারে বানাতে যাবে। একটা সীমা দাও (যেমন ৬০ দিন):

	```ts
	const MAX_SCHEDULE_DAYS = 60;

	.refine(
		(data) => {
			const days = (Date.parse(data.endDate) - Date.parse(data.startDate)) / 86_400_000;
			return days < MAX_SCHEDULE_DAYS;
		},
		{ message: `Date range must be less than ${MAX_SCHEDULE_DAYS} days`, path: ["endDate"] },
	)
	```

4. **অতীতের তারিখে slot।** এখন `2020-01-01` দিলেও slot তৈরি হয় — কেউ book করতে পারবে না, শুধু জায়গা নেবে। ঢাকার আজকের তারিখ দিয়ে আটকাও:

	```ts
	// ঢাকার আজকের তারিখ "YYYY-MM-DD"
	const todayInDhaka = () => new Date(Date.now() + 6 * 60 * 60_000).toISOString().slice(0, 10);

	.refine((data) => data.startDate >= todayInDhaka(), {
		message: "Start date cannot be in the past",
		path: ["startDate"],
	})
	```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **সব duplicate হলে `201` + `data: []`।** User বুঝবে না কেন খালি। Message-এ সংখ্যা দেখাতে পারো:

	```ts
	message: result.length > 0 ? `${result.length} schedules created` : "All schedules already exist",
	```

6. **Update-এ race condition।** `findFirst` আর `update`-এর মাঝে অন্য কেউ একই সময় বানালে `P2002` → `500`। খুবই বিরল। চাইলে `globalErrorHandler`-এ `Prisma.PrismaClientKnownRequestError` + `err.code === "P2002"` → `409` ধরলে সব module-এ কাজ করবে।
7. **Update-এ slot-এর দৈর্ঘ্য।** Update যেকোনো দৈর্ঘ্য নেয় (যেমন ৩ ঘণ্টা বা মাঝরাত পার হওয়া)। সবসময় ৩০ মিনিট চাইলে `endDate`/`endTime` না নিয়ে শুধু start নিয়ে `addMinutes(start, 30)` করা সহজ।
8. **Default sort।** QueryBuilder-এ default `desc`, তাই সবচেয়ে পরের slot আগে আসে। Schedule-এ সাধারণত আগেরটা আগে দরকার — পরে `IQueryConfig`-এ `defaultSortOrder` যোগ করা যায়।

## 🔜 পরে যা যোগ হবে

- [ ] Doctor schedule — doctor নিজের জন্য slot বেছে নেবে (`DoctorSchedules`)
- [ ] Appointment booking → `isBooked: true`
- [ ] Doctor-এর জন্য শুধু ফাঁকা (`isBooked: false`) slot দেখানো
