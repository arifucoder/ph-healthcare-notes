# 34 - Dashboard Stats (Role-wise)

Dashboard-এর জন্য একটাই route: `GET /stats`। কে login করেছে (role) দেখে আলাদা আলাদা data ফেরত যায়। Admin পায় পুরো system-এর হিসাব, doctor পায় নিজের, patient পায় নিজের।

```
GET /stats → checkAuth(সব role) → getDashboardStats
          → switch(user.role) → SUPER_ADMIN / ADMIN / DOCTOR / PATIENT stats
          → Promise.all দিয়ে সব query একসাথে
```

| Role | কী কী পায় |
| --- | --- |
| `SUPER_ADMIN` | Admin-এর সব + `superAdminCount` |
| `ADMIN` | appointment, doctor, patient, admin, user, paid payment সংখ্যা, `totalRevenue`, pie chart (status), bar chart (মাস-ভিত্তিক) |
| `DOCTOR` | নিজের appointment, আলাদা patient সংখ্যা, review সংখ্যা, `averageRating`, `totalRevenue`, status distribution |
| `PATIENT` | নিজের appointment, upcoming appointment, দেওয়া review, `totalSpent`, status distribution |

> নতুন কোনো Prisma model লাগবে না — আগের model থেকেই হিসাব হয়।

---

## ধাপ ১: Service — `stats.service.ts`

`src/app/module/stats/stats.service.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import type { Prisma } from "@/generated/prisma/client";
import { AppointmentStatus, PaymentStatus, Role } from "@/generated/prisma/enums";
import httpStatus from "http-status";

// ---------------- Helpers ----------------

// সব status ফেরত দেয় — কোনো status-এর appointment না থাকলে count: 0
const getAppointmentStatusDistribution = async (where: Prisma.AppointmentWhereInput = {}) => {
	const grouped = await prisma.appointment.groupBy({
		by: ["status"],
		_count: { id: true },
		where,
	});

	return Object.values(AppointmentStatus).map((status) => ({
		status,
		count: grouped.find((item) => item.status === status)?._count.id ?? 0,
	}));
};

// শুধু PAID payment-এর যোগফল
const getTotalRevenue = async (where: Prisma.PaymentWhereInput = {}) => {
	const { _sum } = await prisma.payment.aggregate({
		_sum: { amount: true },
		where: { ...where, status: PaymentStatus.PAID },
	});

	return _sum.amount ?? 0;
};

interface IMonthlyAppointmentCount {
	month: Date;
	count: number; // SQL-এ INTEGER-এ CAST করছি, তাই bigint না
}

// শেষ ১২ মাসের মাস-ভিত্তিক appointment সংখ্যা (bar chart)
// generate_series: ১২টা মাসই আসবে; যে মাসে appointment নেই সেখানে count 0
const getMonthlyAppointmentCount = () => {
	return prisma.$queryRaw<IMonthlyAppointmentCount[]>`
		SELECT months.month,
			CAST(COUNT(a."id") AS INTEGER) AS count
		FROM generate_series(
			DATE_TRUNC('month', NOW()) - INTERVAL '11 months',
			DATE_TRUNC('month', NOW()),
			INTERVAL '1 month'
		) AS months(month)
		LEFT JOIN "appointments" a
			ON DATE_TRUNC('month', a."createdAt") = months.month
		GROUP BY months.month
		ORDER BY months.month ASC;
	`;
};

// ---------------- Role-wise stats ----------------

// ADMIN আর SUPER_ADMIN দুজনেরই কমন অংশ
const getCommonAdminStats = async () => {
	const [
		appointmentCount,
		doctorCount,
		patientCount,
		adminCount,
		userCount,
		paidPaymentCount,
		totalRevenue,
		pieChartData,
		barChartData,
	] = await Promise.all([
		prisma.appointment.count(),
		prisma.doctor.count({ where: { isDeleted: false } }),
		prisma.patient.count({ where: { isDeleted: false } }),
		prisma.admin.count({ where: { isDeleted: false, user: { role: Role.ADMIN } } }),
		prisma.user.count({ where: { isDeleted: false } }),
		prisma.payment.count({ where: { status: PaymentStatus.PAID } }),
		getTotalRevenue(),
		getAppointmentStatusDistribution(),
		getMonthlyAppointmentCount(),
	]);

	return {
		appointmentCount,
		doctorCount,
		patientCount,
		adminCount,
		userCount,
		paidPaymentCount,
		totalRevenue,
		pieChartData,
		barChartData,
	};
};

const getSuperAdminStats = async () => {
	const [commonStats, superAdminCount] = await Promise.all([
		getCommonAdminStats(),
		prisma.admin.count({ where: { isDeleted: false, user: { role: Role.SUPER_ADMIN } } }),
	]);

	return { ...commonStats, superAdminCount };
};

const getDoctorStats = async (user: IRequestUser) => {
	const doctor = await prisma.doctor.findUnique({
		where: { email: user.email },
		select: { id: true, averageRating: true },
	});

	if (!doctor) {
		throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
	}

	const [appointmentCount, uniquePatients, reviewCount, totalRevenue, appointmentStatusDistribution] =
		await Promise.all([
			prisma.appointment.count({ where: { doctorId: doctor.id } }),
			prisma.appointment.findMany({
				where: { doctorId: doctor.id },
				distinct: ["patientId"],
				select: { patientId: true },
			}),
			prisma.review.count({ where: { doctorId: doctor.id } }),
			getTotalRevenue({ appointment: { doctorId: doctor.id } }),
			getAppointmentStatusDistribution({ doctorId: doctor.id }),
		]);

	return {
		appointmentCount,
		patientCount: uniquePatients.length,
		reviewCount,
		averageRating: doctor.averageRating,
		totalRevenue,
		appointmentStatusDistribution,
	};
};

const getPatientStats = async (user: IRequestUser) => {
	const patient = await prisma.patient.findUnique({
		where: { email: user.email },
		select: { id: true },
	});

	if (!patient) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
	}

	const [appointmentCount, upcomingAppointmentCount, reviewCount, totalSpent, appointmentStatusDistribution] =
		await Promise.all([
			prisma.appointment.count({ where: { patientId: patient.id } }),
			// upcoming = SCHEDULED + সময় এখনো আসেনি (পার হয়ে যাওয়া SCHEDULED বাদ)
			prisma.appointment.count({
				where: {
					patientId: patient.id,
					status: AppointmentStatus.SCHEDULED,
					schedule: { startDateTime: { gte: new Date() } },
				},
			}),
			prisma.review.count({ where: { patientId: patient.id } }),
			getTotalRevenue({ appointment: { patientId: patient.id } }),
			getAppointmentStatusDistribution({ patientId: patient.id }),
		]);

	return {
		appointmentCount,
		upcomingAppointmentCount,
		reviewCount,
		totalSpent,
		appointmentStatusDistribution,
	};
};

// ---------------- Main ----------------

const getDashboardStats = (user: IRequestUser) => {
	switch (user.role) {
		case Role.SUPER_ADMIN:
			return getSuperAdminStats();
		case Role.ADMIN:
			return getCommonAdminStats();
		case Role.DOCTOR:
			return getDoctorStats(user);
		case Role.PATIENT:
			return getPatientStats(user);
		default:
			throw new AppError(httpStatus.BAD_REQUEST, "Invalid user role");
	}
};

export const StatsService = {
	getDashboardStats,
};
```

### যে Prisma tools ব্যবহার হলো

| Tool | কাজ | এখানে কোথায় |
| --- | --- | --- |
| `count()` | কয়টা row আছে | প্রায় সব সংখ্যা |
| `aggregate({ _sum })` | একটা column-এর যোগফল | `getTotalRevenue` |
| `groupBy({ by, _count })` | group করে প্রতিটার সংখ্যা | status distribution (pie chart) |
| `findMany({ distinct })` | একই value একবারই | doctor-এর আলাদা patient |
| `$queryRaw` | সরাসরি SQL | মাস-ভিত্তিক count (bar chart) |
| `Promise.all` | সব query একসাথে চালানো | প্রতিটা role-এ |

**কয়েকটা জিনিস খেয়াল করো:**
- **`groupBy` শুধু যেসব status-এর data আছে সেগুলোই দেয়।** তাই `Object.values(AppointmentStatus).map(...)` দিয়ে সব status বানানো হয়েছে, না থাকলে `0`। এতে frontend-এ chart সবসময় ৪টা status পায়।
- **`_sum.amount` কোনো row না থাকলে `null` দেয়**, তাই `?? 0`।
- **Relation দিয়ে filter:** `getTotalRevenue({ appointment: { doctorId } })` মানে "যেসব payment-এর appointment এই doctor-এর"। Payment table-এ `doctorId` না থাকলেও চলে।
- **`$queryRaw`-এ `COUNT(*)` PostgreSQL-এ `bigint` দেয়**, আর `bigint` JSON-এ পাঠানো যায় না (error দেয়)। তাই `CAST(... AS INTEGER)`।
- **`$queryRaw`-এ table/column-এর নাম DB-র নাম:** `"appointments"` (`@@map`) আর `"createdAt"`। Quote না দিলে PostgreSQL ছোট হাতের অক্ষরে খুঁজবে।
- **Upcoming appointment:** শুধু `status: SCHEDULED` দেখলে সময় পার হয়ে যাওয়া (কিন্তু status বদলানো হয়নি এমন) appointment-ও গোনা হতো। তাই relation দিয়ে `schedule.startDateTime >= এখন` check যোগ করা হয়েছে।
- **`getDashboardStats` `async` না** — শুধু promise ফেরত দেয়। Controller-এ `await` করা হচ্ছে, আর `default`-এর `throw`-ও `catchAsync` ধরে ফেলে।

---

## ধাপ ২: Controller — `stats.controller.ts`

`src/app/module/stats/stats.controller.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { StatsService } from "./stats.service";

const getAuthUser = (req: Request): IRequestUser => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}
	return req.user;
};

const getDashboardStats = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const result = await StatsService.getDashboardStats(user);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Stats data retrieved successfully",
		data: result,
	});
});

export const StatsController = {
	getDashboardStats,
};
```

---

## ধাপ ৩: Route — `stats.route.ts`

`src/app/module/stats/stats.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { StatsController } from "./stats.controller";

const router = Router();

router.get(
	"/",
	checkAuth(Role.SUPER_ADMIN, Role.ADMIN, Role.DOCTOR, Role.PATIENT),
	StatsController.getDashboardStats,
);

export const StatsRoutes = router;
```

---

## ধাপ ৪: `routes/index.ts`-এ mount করা

`src/app/routes/index.ts`

```ts
import { StatsRoutes } from "../module/stats/stats.route";

router.use("/stats", StatsRoutes);
```

---

## 🧪 Test (Postman)

| Method | URL | Role | Body |
| --- | --- | --- | --- |
| `GET` | `/api/v1/stats` | যেকোনো role (login করা) | — |

চারটা role দিয়ে আলাদা আলাদা login করে একই URL hit করো — প্রতিবার আলাদা data আসবে।

Admin-এর response এরকম দেখাবে:

```json
{
	"success": true,
	"message": "Stats data retrieved successfully",
	"data": {
		"appointmentCount": 12,
		"doctorCount": 4,
		"patientCount": 9,
		"adminCount": 1,
		"userCount": 15,
		"paidPaymentCount": 7,
		"totalRevenue": 10500,
		"pieChartData": [
			{ "status": "SCHEDULED", "count": 3 },
			{ "status": "INPROGRESS", "count": 0 },
			{ "status": "COMPLETED", "count": 6 },
			{ "status": "CANCELED", "count": 3 }
		],
		"barChartData": [
			{ "month": "2026-10-01T00:00:00.000Z", "count": 5 },
			{ "month": "2026-11-01T00:00:00.000Z", "count": 7 }
		]
	}
}
```

> সংখ্যাগুলো উদাহরণ — তোমার DB অনুযায়ী আলাদা হবে।

**যা check করবে:**
- Login ছাড়া → `401`
- যে status-এর কোনো appointment নেই, সেটাও `count: 0` সহ আসছে
- `barChartData`-তে `count` number (string বা bigint error না)

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Bar chart-এ ফাঁকা মাস বাদ পড়ে** — যে মাসে কোনো appointment নেই, SQL সেই মাস ফেরতই দেয় না। ফলে chart-এ ১২টার জায়গায় ২-৩টা bar আসে, আর মাঝের ফাঁকা মাসগুলো উধাও হয়ে যায়। `generate_series` দিয়ে ১২ মাস বানিয়ে `LEFT JOIN` করো:
   ```sql
   SELECT m.month, CAST(COUNT(a.id) AS INTEGER) AS count
   FROM generate_series(DATE_TRUNC('month', NOW()) - INTERVAL '11 months', DATE_TRUNC('month', NOW()), INTERVAL '1 month') AS m(month)
   LEFT JOIN "appointments" a ON DATE_TRUNC('month', a."createdAt") = m.month
   GROUP BY m.month
   ORDER BY m.month ASC;
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

2. **মাসের হিসাব UTC-তে** — `DATE_TRUNC` UTC অনুযায়ী মাস কাটে। বাংলাদেশের সময়ে মাসের ১ তারিখ রাত ১২টা থেকে সকাল ৬টার মধ্যে বানানো appointment আগের মাসে গোনা হবে। খুব দরকার হলে: `DATE_TRUNC('month', "createdAt" AT TIME ZONE 'Asia/Dhaka')`।

3. **আলাদা patient গোনার সময় সব row আসে** — `findMany({ distinct })` সব patientId memory-তে এনে `.length` নেয়। Data কম থাকলে ঠিক আছে। অনেক data হলে `groupBy({ by: ["patientId"] })` বা raw SQL-এ `COUNT(DISTINCT "patientId")` দ্রুত হবে।

4. **Doctor-এর `isDeleted` check নেই** — deleted doctor login করতে পারলে সে stats পাবে। সাধারণত login-এর সময়েই আটকানো হয়, তাই সমস্যা নাও হতে পারে। Prescription module-এর মতো `!doctor || doctor.isDeleted` দিতে পারো।

### ✅ (Done)

- **"Upcoming" appointment-এ পুরনো তারিখ ঢুকে যেত** — `schedule: { startDateTime: { gte: new Date() } }` যোগ করা হয়েছে (ধাপ ১, `getPatientStats`)।

---

## 🔜 পরে যা যোগ হবে

- [ ] Frontend dashboard-এ card + pie chart + bar chart
- [ ] (ঐচ্ছিক) মাস-ভিত্তিক revenue chart
