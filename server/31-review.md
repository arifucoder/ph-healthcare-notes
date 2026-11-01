# 31 - Review (Rating & Comment)

Appointment শেষ হলে আর payment হয়ে গেলে patient সেই doctor-কে review (rating + comment) দিতে পারবে। প্রতিবার review add/update/delete হলে doctor-এর `averageRating` নতুন করে হিসাব হবে।

```
Patient → POST /reviews → validateRequest → checkAuth(PATIENT) → giveReview
        → (appointment COMPLETED + PAID?) → transaction { review.create → recalculateDoctorRating }
```

| Rule | কোথায় check হয় |
| --- | --- |
| নিজের appointment-ই review করা যাবে | service (`patientId` মিলানো) |
| Appointment `COMPLETED` হতে হবে | service |
| Payment `PAID` হতে হবে | service |
| এক appointment-এ একটাই review | DB (`appointmentId @unique`) → `P2002` → 409 |
| Update/Delete শুধু নিজের review | `getOwnReview` helper |

---

## ধাপ ১: Prisma schema যোগ করা

সবার আগে schema file add করতে হবে — `prisma/schema/review.prisma`

```prisma
//review.prisma
model Review {
  id        String   @id @default(uuid(7))
  rating    Int
  comment   String?  @db.Text
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  appointmentId String      @unique
  appointment   Appointment @relation(fields: [appointmentId], references: [id], onDelete: Cascade)

  patientId String
  patient   Patient @relation(fields: [patientId], references: [id], onDelete: Cascade)

  doctorId String
  doctor   Doctor @relation(fields: [doctorId], references: [id], onDelete: Cascade)

  @@index([patientId], name: "idx_review_patientId")
  @@index([doctorId, createdAt], name: "idx_review_doctorId_createdAt")
  @@map("reviews")
}
```

- `appointmentId @unique` → একটা appointment-এ একটাই review (1-to-1)।
- `@@index([doctorId, createdAt])` → এক doctor-এর review নতুন থেকে পুরনো sort করে আনতে দ্রুত।

অন্য model-গুলোতে উল্টো দিকের relation লাগবে:

```prisma
// doctor.prisma
averageRating Float    @default(0.0)
reviews       Review[]

// patient.prisma
reviews Review[]

// appointment.prisma
review Review?
```

> ✏️ উল্টো দিকের relation field গুলো নোটে ছিল না — এগুলো ছাড়া Prisma validation error দেয়। (project-এ আগে থেকেই আছে)

তারপর:

```bash
pnpm migrate
pnpm generate
```

---

## ধাপ ২: `review.interface.ts`

```ts
export interface ICreateReviewPayload {
	appointmentId: string;
	rating: number;
	comment?: string;
}

export interface IUpdateReviewPayload {
	rating?: number;
	comment?: string;
}
```

---

## ধাপ ৩: `review.validation.ts`

```ts
import * as z from "zod";

const ratingSchema = z
	.int("Rating must be a whole number between 1 and 5")
	.min(1, "Rating must be at least 1")
	.max(5, "Rating cannot be more than 5");

const commentSchema = z
	.string("Comment must be a string")
	.trim()
	.min(1, "Comment cannot be empty")
	.max(1000, "Comment cannot be more than 1000 characters");

const createReviewValidationSchema = z.object({
	appointmentId: z.uuid("A valid appointmentId is required"),
	rating: ratingSchema,
	comment: commentSchema.optional(),
});

const updateReviewValidationSchema = z
	.object({
		rating: ratingSchema.optional(),
		comment: commentSchema.optional(),
	})
	.refine((data) => data.rating !== undefined || data.comment !== undefined, {
		message: "Provide at least rating or comment to update",
	});

export const ReviewValidation = {
	createReviewValidationSchema,
	updateReviewValidationSchema,
};
```

- `ratingSchema` আর `commentSchema` একবার বানিয়ে create/update দুই জায়গায় reuse।
- `z.int()` → Zod v4-এ সরাসরি integer (4.5 দিলে error)।
- `.refine(...)` → update-এ অন্তত একটা field (rating বা comment) দিতেই হবে, খালি `{}` চলবে না।
- `validateRequest` parse করা data দিয়ে `req.body` replace করে, তাই body-তে বাড়তি field (যেমন `doctorId`) পাঠালেও সেটা বাদ পড়ে যায় — `data: payload` নিরাপদ।

---

## ধাপ ৪: `review.constant.ts`

```ts
import type { Prisma } from "@/generated/prisma/client";

export const reviewSearchableFields = ["comment"] as const;

export const reviewFilterableFields = ["doctorId", "rating"] as const;

export const reviewSortableFields = ["createdAt", "rating"] as const;

// public route — শুধু দেখানোর মতো field, email/phone/videoCallingId কখনো না
export const reviewPublicInclude = {
	patient: { select: { name: true, profilePhoto: true } },
	doctor: { select: { id: true, name: true, profilePhoto: true, designation: true } },
} satisfies Prisma.ReviewInclude;
```

- `satisfies Prisma.ReviewInclude` → type check হয়, কিন্তু object-এর exact type থেকে যায়, তাই পরে `Prisma.ReviewGetPayload<{ include: typeof reviewPublicInclude }>` দিয়ে সঠিক return type পাওয়া যায়।
- এখানে `import type` কারণ শুধু type হিসেবে লাগছে।

---

## ধাপ ৫: `review.service.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IQueryParams, IQueryResult } from "@/app/interfaces/query.interface";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import { QueryBuilder } from "@/app/utils/QueryBuilder";
import { Prisma } from "@/generated/prisma/client"; // ✅ value import, কারণ instanceof-এ লাগবে
import { AppointmentStatus, PaymentStatus, Role } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import {
	reviewFilterableFields,
	reviewPublicInclude,
	reviewSearchableFields,
	reviewSortableFields,
} from "./review.constant";
import type { ICreateReviewPayload, IUpdateReviewPayload } from "./review.interface";

type ReviewWithRelations = Prisma.ReviewGetPayload<{ include: typeof reviewPublicInclude }>;

// ---------------- Helpers ----------------

// review add/update/delete হলে doctor-এর averageRating নতুন করে হিসাব
const recalculateDoctorRating = async (tx: Prisma.TransactionClient, doctorId: string) => {
	const { _avg } = await tx.review.aggregate({
		where: { doctorId },
		_avg: { rating: true },
	});

	// শেষ review মুছে গেলে _avg.rating = null → 0
	const average = _avg.rating ?? 0;

	await tx.doctor.update({
		where: { id: doctorId },
		data: { averageRating: Math.round(average * 10) / 10 }, // 4.666 → 4.7
	});
};

// review খুঁজে বের করো + এটা এই patient-এর কিনা check
const getOwnReview = async (reviewId: string, user: IRequestUser) => {
	const review = await prisma.review.findUnique({
		where: { id: reviewId },
		include: { patient: { select: { email: true } } },
	});

	if (!review) {
		throw new AppError(httpStatus.NOT_FOUND, "Review not found");
	}

	if (review.patient.email !== user.email) {
		throw new AppError(httpStatus.FORBIDDEN, "This is not your review");
	}

	return review;
};

// ---------------- Services ----------------

const giveReview = async (user: IRequestUser, payload: ICreateReviewPayload) => {
	const patientData = await prisma.patient.findUnique({
		where: { email: user.email },
	});

	if (!patientData || patientData.isDeleted) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient profile not found");
	}

	const appointmentData = await prisma.appointment.findUnique({
		where: { id: payload.appointmentId },
	});

	if (!appointmentData) {
		throw new AppError(httpStatus.NOT_FOUND, "Appointment not found");
	}

	if (appointmentData.patientId !== patientData.id) {
		throw new AppError(httpStatus.FORBIDDEN, "You can only review your own appointments");
	}

	if (appointmentData.status !== AppointmentStatus.COMPLETED) {
		throw new AppError(httpStatus.BAD_REQUEST, "You can only review a completed appointment");
	}

	if (appointmentData.paymentStatus !== PaymentStatus.PAID) {
		throw new AppError(httpStatus.BAD_REQUEST, "You can only review after payment is done");
	}

	try {
		return await prisma.$transaction(async (tx) => {
			const review = await tx.review.create({
				data: {
					appointmentId: appointmentData.id,
					rating: payload.rating,
					comment: payload.comment,
					patientId: patientData.id,
					doctorId: appointmentData.doctorId,
				},
			});

			await recalculateDoctorRating(tx, review.doctorId);

			return review;
		});
	} catch (error) {
		// appointmentId @unique — দুইবার review (বা একসাথে দুইটা request) হলে DB নিজেই আটকায়
		if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002") {
			throw new AppError(
				httpStatus.CONFLICT,
				"You have already reviewed this appointment. You can update your review instead.",
			);
		}
		throw error;
	}
};

// Public — ?doctorId=...&rating=5&sortBy=rating&page=1&limit=10
const getAllReviews = async (query: IQueryParams): Promise<IQueryResult<ReviewWithRelations>> => {
	return new QueryBuilder<ReviewWithRelations, Prisma.ReviewWhereInput, Prisma.ReviewInclude>(prisma.review, query, {
		searchableFields: reviewSearchableFields,
		filterableFields: reviewFilterableFields,
		sortableFields: reviewSortableFields,
	})
		.where({ doctor: { isDeleted: false } })
		.search()
		.filter()
		.sort()
		.paginate()
		.include(reviewPublicInclude)
		.execute();
};

// route-এ শুধু PATIENT আর DOCTOR allowed
const getMyReviews = async (user: IRequestUser) => {
	if (user.role === Role.PATIENT) {
		return prisma.review.findMany({
			where: { patient: { email: user.email } },
			include: { doctor: { select: { id: true, name: true, profilePhoto: true, designation: true } } },
			orderBy: { createdAt: "desc" },
		});
	}

	return prisma.review.findMany({
		where: { doctor: { email: user.email } },
		include: { patient: { select: { name: true, profilePhoto: true } } },
		orderBy: { createdAt: "desc" },
	});
};

const updateReview = async (user: IRequestUser, reviewId: string, payload: IUpdateReviewPayload) => {
	const review = await getOwnReview(reviewId, user);

	return prisma.$transaction(async (tx) => {
		const updatedReview = await tx.review.update({
			where: { id: review.id },
			data: payload,
		});

		// শুধু comment বদলালে average হিসাবের দরকার নেই
		if (payload.rating !== undefined) {
			await recalculateDoctorRating(tx, updatedReview.doctorId);
		}

		return updatedReview;
	});
};

const deleteReview = async (user: IRequestUser, reviewId: string) => {
	const review = await getOwnReview(reviewId, user);

	return prisma.$transaction(async (tx) => {
		const deletedReview = await tx.review.delete({
			where: { id: review.id },
		});

		await recalculateDoctorRating(tx, deletedReview.doctorId);

		return deletedReview;
	});
};

export const ReviewService = {
	giveReview,
	getAllReviews,
	getMyReviews,
	updateReview,
	deleteReview,
};
```

মূল পয়েন্ট:

- **`Prisma` value import** — `instanceof Prisma.PrismaClientKnownRequestError` runtime-এ চলে, তাই `import type` দিলে চলবে না (`verbatimModuleSyntax` সেটা মুছে দেবে)।
- **Transaction** — review লেখা আর `averageRating` update একসাথে হয়; একটা fail করলে দুটোই rollback।
- **`P2002`** — unique constraint ভাঙলে Prisma এই code দেয়। আগে `findUnique` দিয়ে check না করে DB-কে আটকাতে দেওয়া হয়েছে, তাই একসাথে দুইটা request এলেও duplicate হবে না → 409 CONFLICT।
- **`aggregate` + `_avg`** — সব review-এর গড় DB-তেই হিসাব হয়; `Math.round(x * 10) / 10` দিয়ে এক দশমিক।
- **`getOwnReview`** — update আর delete দুইটাতেই "review আছে কিনা + নিজের কিনা" check একই, তাই helper।
- **`getMyReviews`** — patient হলে নিজের দেওয়া review (সাথে doctor info), doctor হলে নিজের পাওয়া review (সাথে patient info)।

---

## ধাপ ৬: `review.controller.ts`

```ts
import type { Request, Response } from "express";
import httpStatus from "http-status";
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import { ReviewService } from "./review.service";

const getAuthUser = (req: Request): IRequestUser => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}
	return req.user;
};

const giveReview = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const result = await ReviewService.giveReview(user, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.CREATED,
		success: true,
		message: "Review created successfully",
		data: result,
	});
});

const getAllReviews = catchAsync(async (req: Request, res: Response) => {
	const result = await ReviewService.getAllReviews(req.query);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Reviews retrieved successfully",
		meta: result.meta,
		data: result.data,
	});
});

const getMyReviews = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const result = await ReviewService.getMyReviews(user);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Reviews retrieved successfully",
		data: result,
	});
});

const updateReview = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const reviewId = req.params.id as string;
	const result = await ReviewService.updateReview(user, reviewId, req.body);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Review updated successfully",
		data: result,
	});
});

const deleteReview = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const reviewId = req.params.id as string;
	const result = await ReviewService.deleteReview(user, reviewId);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Review deleted successfully",
		data: result,
	});
});

export const ReviewController = {
	giveReview,
	getAllReviews,
	getMyReviews,
	updateReview,
	deleteReview,
};
```

- `getAuthUser` → `req.user` না থাকলে 401; থাকলে TypeScript-এর কাছে type নিশ্চিত (`IRequestUser`)।

---

## ধাপ ৭: `review.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { ReviewController } from "./review.controller";
import { ReviewValidation } from "./review.validation";

const router = Router();

router.get("/", ReviewController.getAllReviews);
router.get("/my-reviews", checkAuth(Role.PATIENT, Role.DOCTOR), ReviewController.getMyReviews);
router.post(
	"/",
	checkAuth(Role.PATIENT),
	validateRequest(ReviewValidation.createReviewValidationSchema),
	ReviewController.giveReview,
);
router.patch(
	"/:id",
	checkAuth(Role.PATIENT),
	validateRequest(ReviewValidation.updateReviewValidationSchema),
	ReviewController.updateReview,
);
router.delete("/:id", checkAuth(Role.PATIENT), ReviewController.deleteReview);

export const ReviewRoutes = router;
```

- `/my-reviews` এখানে `GET` আর `/:id` route-গুলো `PATCH`/`DELETE`, তাই conflict নেই। তবুও static path (`/my-reviews`) dynamic path (`/:id`)-এর আগে রাখা ভালো অভ্যাস।

---

## ধাপ ৮: `routes/index.ts`-এ mount

```ts
import { ReviewRoutes } from "../module/review/review.route";
// ...
router.use("/reviews", ReviewRoutes);
```

---

## 🧪 Test / API

Base: `http://localhost:5000/api/v1/reviews` (port নিজের `.env` অনুযায়ী)

| Method | URL | Auth | Body |
| --- | --- | --- | --- |
| GET | `/reviews?doctorId=<id>&rating=5&sortBy=rating&sortOrder=desc&page=1&limit=10` | Public | — |
| GET | `/reviews?searchTerm=good` | Public | — |
| GET | `/reviews/my-reviews` | PATIENT / DOCTOR | — |
| POST | `/reviews` | PATIENT | `{ "appointmentId": "<uuid>", "rating": 5, "comment": "Very helpful doctor" }` |
| PATCH | `/reviews/:id` | PATIENT (নিজের) | `{ "rating": 4 }` বা `{ "comment": "..." }` |
| DELETE | `/reviews/:id` | PATIENT (নিজের) | — |

যা যা test করবে:

- [ ] `COMPLETED` নয় এমন appointment → 400
- [ ] `PAID` নয় → 400
- [ ] অন্যের appointment → 403
- [ ] একই appointment-এ দ্বিতীয়বার → 409
- [ ] `rating: 6` বা `4.5` → Zod error
- [ ] PATCH-এ `{}` → "Provide at least rating or comment to update"
- [ ] review add/update/delete-এর পর doctor-এর `averageRating` বদলেছে কিনা (Prisma Studio)

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

**1. একসাথে একই doctor-এর দুইটা review এলে average একটু পুরনো হতে পারে** (সম্ভাব্য, test করা হয়নি)
Postgres-এর default isolation `READ COMMITTED`। দুইটা transaction একসাথে চললে প্রত্যেকে অন্যটার uncommitted review দেখে না, তাই শেষে যে লিখবে তার `averageRating`-এ অন্যটার rating বাদ পড়তে পারে। পরের যেকোনো review-তে আবার ঠিক হয়ে যায়, তাই বড় সমস্যা না। পুরো ঠিক রাখতে চাইলে doctor row আগে lock করো:

```ts
const recalculateDoctorRating = async (tx: Prisma.TransactionClient, doctorId: string) => {
	// doctor row lock → একই doctor-এর অন্য transaction অপেক্ষা করবে
	await tx.$executeRaw`SELECT id FROM doctors WHERE id = ${doctorId} FOR UPDATE`;
	// ...বাকি আগের মতো
};
```
(table name `@@map` অনুযায়ী মিলিয়ে নিও।)

**2. Deleted patient নিজের review update/delete করতে পারছে**
`giveReview`-এ `isDeleted` check আছে, কিন্তু `getOwnReview`-এ নেই। সামঞ্জস্য রাখতে:

```ts
include: { patient: { select: { email: true, isDeleted: true } } },
// ...
if (review.patient.email !== user.email || review.patient.isDeleted) {
	throw new AppError(httpStatus.FORBIDDEN, "This is not your review");
}
```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

**3. Comment মুছে ফেলার উপায় নেই**
`commentSchema`-এ `.min(1)` থাকায় update-এ খালি comment দেওয়া যায় না। কেউ শুধু rating রেখে comment সরাতে চাইলে `nullable()` দাও:

```ts
comment: commentSchema.nullable().optional(), // null → comment মুছে যাবে
```
(তাহলে `IUpdateReviewPayload`-এ `comment?: string | null` করতে হবে, আর refine-এ `!== undefined` আগের মতোই কাজ করবে।)

**4. Doctor-এর review count দেখাতে চাইলে**
Doctor profile-এ "4.7 ⭐ (23 reviews)" দেখাতে `_count` ব্যবহার করা যায়, আলাদা column লাগবে না:

```ts
include: { _count: { select: { reviews: true } } }
```

---

## 🔜 পরে যা যোগ হবে

- [ ] Doctor list-এ `averageRating` দিয়ে sort/filter
- [ ] Admin-এর জন্য খারাপ/spam review মুছে ফেলার route
- [ ] Frontend-এ star rating UI
