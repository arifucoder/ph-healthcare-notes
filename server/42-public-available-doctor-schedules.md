# 42 - Public Route: Doctor-এর খালি Slot (Available Schedules)

> Patient এখন কোনো doctor-এর খালি slot দেখতে পারে না (আগের সব doctor-schedule route DOCTOR বা ADMIN-এর জন্য)। Consultation page-এ booking-এর জন্য একটা **public** route লাগবে।
> আগের note: 26 (doctor schedule), 27 (appointment), 28 (stripe payment)। Frontend অংশ: client note 25।

```
GET /api/v1/doctor-schedules/doctor/:doctorId/available?startDate=2026-11-09&endDate=2026-11-22
  → Zod (তারিখ + সর্বোচ্চ ৩১ দিন) → doctor আছে? → ভবিষ্যতের + booked না এমন slot → শুধু { id, startDateTime, endDateTime }[]
```

---

## ধাপ ১: `doctorSchedule.validation.ts` — object-এর আগে যোগ করো, তারপর export-এ দাও

```ts
const dateString = z.iso.date("Date must be in YYYY-MM-DD format");

// ---------- Public: doctor-এর খালি slot (date range) ----------
const MAX_RANGE_DAYS = 31;

const availableScheduleQueryZodSchema = z
	.object({
		startDate: dateString,
		endDate: dateString,
	})
	.refine((data) => data.endDate >= data.startDate, {
		message: "End date must be the same as or after start date",
		path: ["endDate"],
	})
	// সীমা না দিলে কেউ ১০ বছরের range চেয়ে DB-কে চাপ দিতে পারে
	.refine(
		(data) => (Date.parse(data.endDate) - Date.parse(data.startDate)) / 86_400_000 < MAX_RANGE_DAYS,
		{ message: `Date range can be at most ${MAX_RANGE_DAYS} days`, path: ["endDate"] },
	);

export const DoctorScheduleValidation = {
	createDoctorScheduleZodSchema,
	updateDoctorScheduleZodSchema,
	availableScheduleQueryZodSchema,
};
```

- `z.iso.date()` — Zod v4: শুধু `YYYY-MM-DD` format (সময় ছাড়া) মানে।
- `data.endDate >= data.startDate` — `YYYY-MM-DD` string সরাসরি তুলনা করা যায় (অক্ষরের ক্রম = তারিখের ক্রম)।
- `86_400_000` = ১ দিনের millisecond (`_` শুধু পড়ার সুবিধার জন্য)। `< 31` মানে start আর end মিলিয়ে সর্বোচ্চ ৩১ দিন।
- `path: ["endDate"]` → error message `endDate` field-এর নামে আসে।
- এটা `req.query`-র জন্য, তাই route-এ `validateRequest` না (ওটা `req.body` দেখে) — service-এর ভেতরে `.parse()`।

---

## ধাপ ২: `doctorSchedule.service.ts` — import আর নতুন function

```ts
// উপরে import
import { addDaysToDateString, buildDateTime } from "@/app/module/schedule/schedule.utils";
import { DoctorScheduleValidation } from "./doctorSchedule.validation";

// ---------- public ----------

// রোগী doctor-এর খালি slot দেখে: ভবিষ্যতের, booked না, ঢাকার দিন অনুযায়ী
const getAvailableDoctorSchedules = async (doctorId: string, query: IQueryParams) => {
	// invalid হলে ZodError → globalErrorHandler → 400
	const { startDate, endDate } = DoctorScheduleValidation.availableScheduleQueryZodSchema.parse(query);

	const doctor = await prisma.doctor.findUnique({
		where: { id: doctorId, isDeleted: false },
		select: { id: true },
	});

	if (!doctor) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	}

	const now = new Date();
	const rangeStart = buildDateTime(startDate, "00:00");
	const rangeEnd = buildDateTime(addDaysToDateString(endDate, 1), "00:00");

	const rows = await prisma.doctorSchedules.findMany({
		where: {
			doctorId,
			isBooked: false,
			// আজকের যে slot শুরু হয়ে গেছে সেটা বাদ (book-এর সময় backend এমনিতেও আটকায়)
			schedule: { startDateTime: { gte: rangeStart > now ? rangeStart : now, lt: rangeEnd } },
		},
		select: { schedule: { select: { id: true, startDateTime: true, endDateTime: true } } },
		orderBy: { schedule: { startDateTime: "asc" } },
	});

	// রোগীর শুধু সময়টুকু লাগে
	return rows.map((row) => row.schedule);
};
```

তারপর `DoctorScheduleService` object-এ `getAvailableDoctorSchedules` যোগ করো।

### ভেঙে বোঝা

**১. Validation service-এ** — `.parse()` ভুল হলে `ZodError` throw করে → Express 5 নিজেই error handler-এ পাঠায় → `handleZodError` → 400।

**২. Doctor check** — ভুল/মুছে ফেলা doctor-এ খালি list (`[]`) না দিয়ে 404, যাতে frontend বুঝতে পারে "slot নেই" আর "doctor নেই" আলাদা। `findUnique`-এ `id`-এর সাথে `isDeleted`-ও দেওয়া যায় (Prisma 5+)।

**৩. তারিখের range (ঢাকার দিন অনুযায়ী)**

| মান | উদাহরণ (`2026-11-09` → `2026-11-22`) |
|---|---|
| `rangeStart` | ঢাকার ৯ নভেম্বর রাত ১২টা = `2026-11-08T18:00Z` |
| `rangeEnd` | ঢাকার **২৩** নভেম্বর রাত ১২টা (`endDate + 1` দিন, `lt` দিয়ে) |

- `buildDateTime` সবসময় `+06:00` ধরে (note 25), তাই server UTC-তে চললেও ঢাকার দিনই হয়।
- `endDate + 1` আর `lt` → শেষ দিনের রাত ১১:৩০-এর slot-ও আসে।
- `gte: rangeStart > now ? rangeStart : now` → আজকের দিন হলে যে slot শুরু হয়ে গেছে সেগুলো বাদ।

**৪. শুধু দরকারি field** — `select`-এ শুধু schedule-এর `id`, `startDateTime`, `endDateTime`। `rows.map(row => row.schedule)` → nested object খুলে সরল array। (`id` = `scheduleId`, booking-এ এটাই পাঠাতে হয়।)

**৫. `orderBy: { schedule: { startDateTime: "asc" } }`** — relation-এর field দিয়ে sort; frontend আর সাজাতে হয় না।

---

## ধাপ ৩: `doctorSchedule.controller.ts` — নতুন controller, export-এ দাও

```ts
const getAvailableDoctorSchedules = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorScheduleService.getAvailableDoctorSchedules(getParam(req, "doctorId"), req.query);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Available schedules retrieved successfully",
		data: result,
	});
});
```

- `getParam` (note 38) → cast ছাড়াই `string`।

---

## ধাপ ৪: `doctorSchedule.route.ts` — auth ছাড়া (public)

```ts
// public: consultation page-এ রোগী (login ছাড়াও) doctor-এর খালি slot দেখে
router.get("/doctor/:doctorId/available", DoctorScheduleController.getAvailableDoctorSchedules);
```

- কোনো `checkAuth` নেই — login ছাড়াও slot দেখা যায় (book করতে login লাগে, সেটা appointment route-এ)।
- `/:doctorId/schedule/:scheduleId` route-এর সাথে গোলমাল হয় না: দ্বিতীয় অংশ `available` বনাম `schedule`, আর এটা `/doctor/` দিয়ে শুরু।

---

## ধাপ ৫: ⚠️ `appointment.service.ts`-এর Stripe `cancel_url`

`cancel_url` ছিল `/dashboard/appointments`, কিন্তু frontend nav-এ page-টার নাম `/dashboard/my-appointments`। মিলিয়ে নাও:

```ts
cancel_url: `${envVars.FRONTEND_URL}/dashboard/my-appointments?payment=cancelled`,
```

> ✏️ project-এ এখন আছে `?error=payment_cancelled` (path ঠিক আছে, শুধু query-র নাম আলাদা)। frontend-এ cancel message দেখালে এই নামটাই পড়তে হবে — দুটোর যেকোনো একটা রাখো।

---

## 🧪 Test (Postman)

| Request | ফল |
|---|---|
| `GET /api/v1/doctor-schedules/doctor/<doctorId>/available?startDate=2026-11-09&endDate=2026-11-22` | 200, `[{ id, startDateTime, endDateTime }]` সময় অনুযায়ী সাজানো |
| token ছাড়া | একই 200 (public) |
| `startDate` বাদ | 400 (Zod) |
| `startDate=09-11-2026` | 400 "Date must be in YYYY-MM-DD format" |
| `endDate` < `startDate` | 400 |
| ৪০ দিনের range | 400 "Date range can be at most 31 days" |
| ভুল / মুছে ফেলা doctorId | 404 "Doctor not found" |
| কোনো slot book হলে আবার call | ওই slot আর list-এ নেই |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **অতীতের পুরো range-এ DB call বাঁচানো** — `endDate` আজকের আগের হলে query চালানোর দরকারই নেই:
   ```ts
   if (rangeEnd <= now) return [];
   ```

2. **Public route-এ rate limit** — login ছাড়া চলে, তাই কেউ loop করে চাপ দিতে পারে। note 39-এর limiter এই route-এও আছে কিনা দেখে নিও (global হলে ঠিক আছে)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Pay-now-এ Stripe cancel করলেও slot আটকে থাকে** — appointment আগেই তৈরি হয় (UNPAID, `isBooked: true`), তাই cron cancel না করা পর্যন্ত (৩০ মিনিট) slot অন্য রোগী দেখতে পায় না। চাইলে cancel_url থেকে frontend একটা "cancel appointment" call করে সাথে সাথে ছেড়ে দিতে পারে।

4. **Doctor নিজে inactive/blocked হলে** — এখন শুধু `isDeleted` দেখা হচ্ছে। user `status: BLOCKED` হলে তার slot-ও লুকাতে চাইলে `where`-এ `user: { status: "ACTIVE" }` যোগ করা যায় (schema অনুযায়ী relation-এর নাম দেখে)।
