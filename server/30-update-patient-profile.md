# 30 - Patient Update Profile (info + health data + photo + medical reports)

```
PATCH /patients/update-my-profile (form-data: data + profilePhoto + medicalReports)
→ checkAuth(PATIENT) → multer.fields (file → Cloudinary) → validateRequest (data JSON → Zod)
→ attachPatientUploads (file URL body-তে) → service: transaction (patient, user, healthData, reports)
→ commit-এর পরে পুরনো file Cloudinary থেকে মোছা → নতুন profile ফেরত
```

---

## ধাপ ১: `patient` module-এর file গুলো

```
src/app/module/patient/
├── patient.interface.ts
├── patient.validation.ts
├── patient.middlewares.ts
├── patient.service.ts
├── patient.controller.ts
└── patient.route.ts
```

> `patient.utils.ts` লাগে না — date-এর কাজ validation-এর `dateOfBirth` `.transform()`-এই হয় (Recommendation-এর ✅ দেখো)।

---

## ধাপ ২: `patient.validation.ts`

```ts
import { BloodGroup, Gender, MaritalStatus } from "@/generated/prisma/enums";
import * as z from "zod";

const patientInfoZodSchema = z.object({
	name: z
		.string("Name must be a string")
		.trim()
		.min(1, "Name cannot be empty")
		.max(100, "Name must be less than 100 characters")
		.optional(),
	contactNumber: z
		.string("Contact number must be a string")
		.trim()
		.min(1, "Contact number cannot be empty")
		.max(20, "Contact number must be less than 20 characters")
		.optional(),
	address: z
		.string("Address must be a string")
		.trim()
		.min(1, "Address cannot be empty")
		.max(200, "Address must be less than 200 characters")
		.optional(),
	// profilePhoto এখানে নেই — শুধু upload করা file থেকে আসবে (অচেনা key Zod বাদ দিয়ে দেয়)
});

const patientHealthDataZodSchema = z.object({
	gender: z.enum(Gender, "Gender must be MALE, FEMALE or OTHER").optional(),
	dateOfBirth: z.iso
		.date("Date of birth must be in YYYY-MM-DD format")
		// UTC মাঝরাত → @db.Date-এ ঠিক তারিখটাই save হবে
		.transform((date) => new Date(`${date}T00:00:00.000Z`))
		.refine((date) => date <= new Date(), "Date of birth cannot be in the future")
		.optional(),
	bloodGroup: z.enum(BloodGroup, "Invalid blood group").optional(),
	height: z
		.number("Height must be a number (cm)")
		.positive("Height must be positive")
		.max(300, "Height looks invalid")
		.optional(),
	weight: z
		.number("Weight must be a number (kg)")
		.positive("Weight must be positive")
		.max(500, "Weight looks invalid")
		.optional(),
	hasAllergies: z.boolean("hasAllergies must be true/false").optional(),
	hasDiabetes: z.boolean("hasDiabetes must be true/false").optional(),
	smokingStatus: z.boolean("smokingStatus must be true/false").optional(),
	pregnancyStatus: z.boolean("pregnancyStatus must be true/false").optional(),
	hasPastSurgeries: z.boolean("hasPastSurgeries must be true/false").optional(),
	recentAnxiety: z.boolean("recentAnxiety must be true/false").optional(),
	recentDepression: z.boolean("recentDepression must be true/false").optional(),
	dietaryPreferences: z.string().trim().max(200).optional(),
	mentalHealthHistory: z.string().trim().max(2000).optional(),
	immunizationStatus: z.string().trim().max(200).optional(),
	maritalStatus: z.enum(MaritalStatus, "Invalid marital status").optional(),
});

// client JSON থেকে শুধু delete করা যায়; নতুন report আসে file upload থেকে
const medicalReportDeleteZodSchema = z.object({
	reportId: z.uuid("Invalid report id"),
	shouldDelete: z.literal(true),
});

const updatePatientProfileZodSchema = z.object({
	patientInfo: patientInfoZodSchema.optional(),
	patientHealthData: patientHealthDataZodSchema.optional(),
	medicalReports: z.array(medicalReportDeleteZodSchema).max(20, "Too many reports at once").optional(),
});

export const PatientValidation = {
	updatePatientProfileZodSchema,
};
```

| অংশ | কী রাখে | Prisma table |
|---|---|---|
| `patientInfo` | name, contactNumber, address | `patient` (name → `user`-ও) |
| `patientHealthData` | gender, DOB, blood group, height... | `patientHealthData` |
| `medicalReports` | শুধু **মোছার** id (`shouldDelete: true`) | `medical_reports` |

- সব `.optional()` → যা পাঠাবে শুধু সেটাই update (PATCH)।
- `dateOfBirth`: `"2000-05-10"` → `2000-05-10T00:00:00.000Z`। DB column `@db.Date` (শুধু তারিখ) — UTC মাঝরাত দিলে timezone-এর কারণে তারিখ এক দিন সরে যায় না।

---

## ধাপ ৩: `patient.interface.ts`

```ts
import type * as z from "zod";
import type { PatientValidation } from "./patient.validation";

// Zod থেকেই type — schema বদলালে type নিজে নিজে বদলাবে
export type TUpdatePatientProfileInput = z.infer<typeof PatientValidation.updatePatientProfileZodSchema>;
export type THealthDataInput = NonNullable<TUpdatePatientProfileInput["patientHealthData"]>;

export interface INewMedicalReport {
	reportName: string;
	reportLink: string;
	reportPublicId: string;
}

// validation-এর পরে middleware এই দুইটা যোগ করে (শুধু নিজের upload করা file)
export type TUpdatePatientProfilePayload = TUpdatePatientProfileInput & {
	profilePhoto?: string;
	newMedicalReports?: INewMedicalReport[];
};
```

| Type | কোথায় |
|---|---|
| `TUpdatePatientProfileInput` | client যা পাঠায় (Zod-পাস করা) |
| `TUpdatePatientProfilePayload` | তার সাথে server-এর যোগ করা file info → service এটা পায় |

---

## ধাপ ৪: `patient.middlewares.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import type { TUpdatePatientProfilePayload } from "./patient.interface";

type TUploadedFiles = Partial<Record<"profilePhoto" | "medicalReports", Express.Multer.File[]>>;

// validateRequest-এর পরে চলে: Zod-পাস করা body-তে শুধু আমাদের upload করা file যোগ করে
export const attachPatientUploads = (req: Request, _res: Response, next: NextFunction) => {
	const payload = req.body as TUpdatePatientProfilePayload;
	const files = req.files as TUploadedFiles | undefined;

	const photo = files?.profilePhoto?.[0];
	if (photo) {
		payload.profilePhoto = photo.path; // Cloudinary secure_url
	}

	const reports = files?.medicalReports ?? [];
	if (reports.length > 0) {
		payload.newMedicalReports = reports.map((file) => ({
			reportName: file.originalname,
			reportLink: file.path,
			reportPublicId: file.filename, // cloudinaryStorage-এ filename = public_id
		}));
	}

	next();
};
```

---

## ধাপ ৫: `patient.route.ts`

```ts
import { multerUpload } from "@/app/config/multer.config";
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { PatientController } from "./patient.controller";
import { attachPatientUploads } from "./patient.middlewares";
import { PatientValidation } from "./patient.validation";

const router = Router();

router.patch(
	"/update-my-profile",
	checkAuth(Role.PATIENT),
	multerUpload.fields([
		{ name: "profilePhoto", maxCount: 1 },
		{ name: "medicalReports", maxCount: 5 },
	]),
	validateRequest(PatientValidation.updatePatientProfileZodSchema), // আগে client data যাচাই
	attachPatientUploads, // তারপর নিজের file যোগ
	PatientController.updateMyProfile,
);

export const PatientRoutes = router;
```

- `.fields([...])` → দুই আলাদা নামের file একসাথে ([নোট ২৩](./23-multer-single-image-upload.md)-এ ছিল `.single`)। ফল: `req.files = { profilePhoto: [File], medicalReports: [File, ...] }`।

---

## ধাপ ৬: `patient.controller.ts`

```ts
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import type { TUpdatePatientProfilePayload } from "./patient.interface";
import { PatientService } from "./patient.service";

const updateMyProfile = catchAsync(async (req: Request, res: Response) => {
	const user = req.user as IRequestUser;
	const payload = req.body as TUpdatePatientProfilePayload;

	const result = await PatientService.updateMyProfile(user, payload);

	sendResponse(res, {
		success: true,
		httpStatusCode: httpStatus.OK,
		message: "Profile updated successfully",
		data: result,
	});
});

export const PatientController = {
	updateMyProfile,
};
```

---

## ধাপ ৭: `patient.service.ts`

```ts
import { deleteFileFromCloudinary } from "@/app/config/cloudinary.config";
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { prisma } from "@/app/lib/prisma";
import httpStatus from "http-status";
import type { THealthDataInput, TUpdatePatientProfilePayload } from "./patient.interface";

// প্রথমবার create করতে এগুলো লাগবেই (Prisma schema-তে required)
const REQUIRED_HEALTH_FIELDS = ["gender", "dateOfBirth", "bloodGroup", "height", "weight"] as const;
type TRequiredHealthField = (typeof REQUIRED_HEALTH_FIELDS)[number];

// type guard: true হলে TS জানবে এই field-গুলো আর undefined না
const hasRequiredHealthFields = (
	data: THealthDataInput,
): data is THealthDataInput & Required<Pick<THealthDataInput, TRequiredHealthField>> =>
	REQUIRED_HEALTH_FIELDS.every((field) => data[field] !== undefined);

const updateMyProfile = async (user: IRequestUser, payload: TUpdatePatientProfilePayload) => {
	const { patientInfo, patientHealthData, medicalReports, profilePhoto, newMedicalReports } = payload;

	const hasAnyUpdate =
		patientInfo || patientHealthData || profilePhoto || medicalReports?.length || newMedicalReports?.length;
	if (!hasAnyUpdate) {
		throw new AppError(httpStatus.BAD_REQUEST, "Nothing to update");
	}

	const patient = await prisma.patient.findUnique({
		where: { userId: user.userId },
		include: { patientHealthData: { select: { id: true } } },
	});
	if (!patient || patient.isDeleted) {
		throw new AppError(httpStatus.NOT_FOUND, "Patient not found");
	}

	// শুধু নিজের report-ই delete করা যাবে (IDOR fix)
	const reportIdsToDelete = [...new Set(medicalReports?.map((r) => r.reportId) ?? [])];
	const reportsToDelete =
		reportIdsToDelete.length > 0
			? await prisma.medicalReport.findMany({
					where: { id: { in: reportIdsToDelete }, patientId: patient.id },
					select: { id: true, reportLink: true },
				})
			: [];
	if (reportsToDelete.length !== reportIdsToDelete.length) {
		throw new AppError(httpStatus.NOT_FOUND, "One or more medical reports not found");
	}

	await prisma.$transaction(async (tx) => {
		if (patientInfo || profilePhoto) {
			await tx.patient.update({
				where: { id: patient.id },
				data: { ...patientInfo, profilePhoto }, // undefined হলে Prisma field-টা skip করে
			});
		}

		if (patientInfo?.name || profilePhoto) {
			await tx.user.update({
				where: { id: patient.userId },
				data: { name: patientInfo?.name, image: profilePhoto },
			});
		}

		if (patientHealthData) {
			if (patient.patientHealthData) {
				await tx.patientHealthData.update({
					where: { patientId: patient.id },
					data: patientHealthData,
				});
			} else {
				if (!hasRequiredHealthFields(patientHealthData)) {
					throw new AppError(
						httpStatus.BAD_REQUEST,
						`First time health data requires: ${REQUIRED_HEALTH_FIELDS.join(", ")}`,
					);
				}
				await tx.patientHealthData.create({
					data: { ...patientHealthData, patientId: patient.id },
				});
			}
		}

		if (reportsToDelete.length > 0) {
			await tx.medicalReport.deleteMany({
				where: { id: { in: reportsToDelete.map((r) => r.id) }, patientId: patient.id },
			});
		}

		if (newMedicalReports?.length) {
			await tx.medicalReport.createMany({
				data: newMedicalReports.map((report) => ({ ...report, patientId: patient.id })),
			});
		}
	});

	// DB commit সফল হওয়ার পরেই পুরনো file মুছি
	const oldFileUrls = reportsToDelete.map((r) => r.reportLink);
	if (profilePhoto && patient.profilePhoto) {
		oldFileUrls.push(patient.profilePhoto);
	}
	// একটা file মুছতে fail করলেও request fail হবে না (DB তো আপডেট হয়েই গেছে)
	await Promise.allSettled(oldFileUrls.map((url) => deleteFileFromCloudinary(url)));

	return prisma.patient.findUnique({
		where: { id: patient.id },
		include: {
			user: { select: { id: true, name: true, email: true, image: true, role: true } },
			patientHealthData: true,
			medicalReports: { orderBy: { createdAt: "desc" } },
		},
	});
};

export const PatientService = {
	updateMyProfile,
};
```

---

## ধাপ ৮: `routes/index.ts`

```ts
import { PatientRoutes } from "../module/patient/patient.route";
// ...
router.use("/patients", PatientRoutes);
```

---

## ধাপ ৯: Error হলে upload হওয়া file মোছা

Test করার সময় দেখা গেল — error হলে (যেমন Zod fail) আগে upload হওয়া ছবি Cloudinary-তে থেকে যাচ্ছে। আগের `globalErrorHandler` শুধু `req.file` (single) দেখত, কিন্তু এখানে `.fields()` → file থাকে `req.files`-এ। তাই নতুন helper:

### `src/app/utils/deleteUploadedFiles.ts` (নতুন file)

```ts
import type { Request } from "express";
import { deleteFileFromCloudinary } from "../config/cloudinary.config";

// req.file (single) / req.files (array) / req.files ({ field: File[] }) — তিন রকমই সামলায়
const collectUploadedFiles = (req: Request): Express.Multer.File[] => {
	if (req.file) return [req.file];
	if (Array.isArray(req.files)) return req.files;
	if (req.files) return Object.values(req.files).flat();
	return [];
};

// request fail করলে এই request-এ upload হওয়া file Cloudinary থেকে মুছে দেয়
export const deleteUploadedFiles = async (req: Request): Promise<void> => {
	const files = collectUploadedFiles(req);
	if (files.length === 0) return;

	// deleteFileFromCloudinary throw করতে পারে → allSettled, যাতে response আটকে না যায়
	await Promise.allSettled(files.map((file) => deleteFileFromCloudinary(file.path)));
};
```

### `globalErrorHandler.ts`-এ

```ts
import { deleteUploadedFiles } from "../utils/deleteUploadedFiles";

export const globalErrorHandler = async (err: unknown, req: Request, res: Response, _next: NextFunction) => {
	// ...isDev log

	// error হলে এই request-এ upload হওয়া file আর রাখার দরকার নেই
	await deleteUploadedFiles(req);

	// ...বাকি একই
};
```

> ✏️ eslint-disable comment সরানো হয়েছে (আমরা oxlint ব্যবহার করি) — unused param-এ _ দেওয়া হয়েছে (`next` → `_next`)।
> পুরো file আর ব্যাখ্যা: [নোট ২৩ — ধাপ ৬-গ ও ৬-ঘ](./23-multer-single-image-upload.md)।

---

# 📘 বুঝে নাও — কেন আর কীভাবে

## কেন একটা route-এ এত কিছু?

Patient-এর profile আসলে ৪ জায়গায় ছড়ানো: `patient` (নাম, ঠিকানা, ছবি), `user` (login-এর নাম/ছবি), `patientHealthData` (স্বাস্থ্য), `medical_reports` (file)। Frontend-এ একটাই "Save" button — তাই একটা request-এ সব, আর সব একসাথে সফল বা একসাথে বাতিল (transaction)।

## Middleware-এর ক্রম কেন এমন?

```
checkAuth → multer.fields → validateRequest → attachPatientUploads → controller
```

| ক্রম | কেন এখানে |
|---|---|
| `checkAuth` প্রথমে | login ছাড়া কেউ Cloudinary-তে file পাঠাতে পারবে না |
| `multer` তারপর | form-data না পড়লে `req.body.data` পাওয়াই যায় না |
| `validateRequest` তারপর | `data` JSON → object → Zod। Zod অচেনা key (যেমন client-এর পাঠানো `profilePhoto`) বাদ দিয়ে দেয় |
| `attachPatientUploads` **Zod-এর পরে** | আগে দিলে Zod আমাদের যোগ করা `profilePhoto` মুছে ফেলত। পরে দেওয়ায়: client নিজে `profilePhoto: "http://hacker..."` লিখলেও বাদ পড়ে, শুধু সত্যিকারের upload করা file-এর URL বসে |

## Service-এর ধাপ

```
১. কিছু পাঠিয়েছে কি? → না হলে 400 "Nothing to update"
২. patient খোঁজা (login user থেকে) → না হলে 404
৩. মোছার report গুলো সত্যিই এই patient-এর? → না হলে 404     (IDOR আটকানো)
৪. transaction:
   - patient update (info + photo)
   - user update (নাম/ছবি বদলালে — login-এর জায়গাতেও মিলবে)
   - healthData: আগে থাকলে update, না থাকলে create (প্রথমবার ৫টা field লাগবেই)
   - পুরনো report মোছা, নতুন report তৈরি
৫. commit সফল → তারপর পুরনো file Cloudinary থেকে মোছা
৬. নতুন পুরো profile ফেরত
```

**গুরুত্বপূর্ণ ব্যাপারগুলো:**

| কী | কেন |
|---|---|
| **IDOR fix** (`patientId: patient.id` দিয়ে খোঁজা) | অন্য রোগীর report id জানলেও সে মুছতে পারবে না — শুধু নিজের id-গুলো মেলে |
| **update না create** (health data) | প্রথমবার row নেই → create; Prisma-তে gender, DOB, blood group, height, weight required, তাই প্রথমবার এগুলো না দিলে `400` |
| **Type guard** (`data is ...`) | `hasRequiredHealthFields` true হলে TypeScript জানে ৫টা field আর `undefined` না → `create`-এ type error হয় না |
| **পুরনো file মোছা commit-এর পরে** | আগে মুছে ফেললে, তারপর DB fail করলে → DB-তে পুরনো URL থাকবে কিন্তু file নেই (ভাঙা ছবি)। পরে মুছলে সবচেয়ে খারাপ হলে একটা অকারণ file থেকে যায় |
| `Promise.allSettled` | DB আগেই update হয়ে গেছে — file মুছতে fail করলেও user-কে error দেখানোর মানে নেই |

## File কখন মোছে — দুই দিক

| অবস্থা | কোন file মোছে | কে মোছে |
|---|---|---|
| Request **fail** (Zod, 404, transaction error) | এই request-এ **নতুন upload হওয়া** file | `globalErrorHandler` → `deleteUploadedFiles` |
| Request **সফল** | **পুরনো** ছবি + মুছতে বলা report | service-এর শেষে |

এতে Cloudinary-তে অকারণ file জমে না।

---

## Test (Postman)

URL: `PATCH http://localhost:5000/api/v1/patients/update-my-profile` — **PATIENT** login থাকতে হবে।

### ১. শুধু JSON (file ছাড়া)

Body → **raw → JSON**:

```json
{ "patientInfo": { "address": "Chattogram" }, "patientHealthData": { "height": 172 } }
```

→ প্রথমবার (health data নেই) → `400 First time health data requires: gender, dateOfBirth, bloodGroup, height, weight`। পুরো দিয়ে আবার:

```json
{
	"patientInfo": { "address": "Chattogram", "contactNumber": "01700000000" },
	"patientHealthData": {
		"gender": "MALE",
		"dateOfBirth": "2000-05-10",
		"bloodGroup": "O_POSITIVE",
		"height": 172,
		"weight": 68
	}
}
```

→ `200`। এরপর শুধু `{ "patientHealthData": { "height": 173 } }` দিলেও চলবে (এখন update)।

### ২. form-data দিয়ে file পাঠানো

Body tab → **form-data**:

| Key | Type | Value |
|---|---|---|
| `data` | Text | `{"patientInfo":{"name":"Arif"}}` |
| `profilePhoto` | File | `my-photo.jpg` |
| `medicalReports` | File | `blood-test.pdf` |
| `medicalReports` | File | `xray.png` |

- Key ঘরে mouse নিলে ডান পাশে **Text / File** dropdown আসে — file-এর row-এ **File** দাও, তারপর **Select Files**।
- একাধিক report: একই key `medicalReports` দিয়ে আলাদা row, প্রতি row-এ একটা file।

→ `200`, response-এ `profilePhoto` (Cloudinary URL), `user.name: "Arif"`, `user.image`, আর `medicalReports`-এ ২টা।

### ৩. ছবি বদলানো

আবার নতুন `profilePhoto` দিয়ে পাঠাও → `200`। Cloudinary Media Library-তে **পুরনো ছবি মুছে গেছে**, নতুনটা আছে।

### ৪. Report মোছা

`data`-তে (raw JSON-ও চলবে):

```json
{ "medicalReports": [{ "reportId": "<report-id>", "shouldDelete": true }] }
```

→ `200`, report DB আর Cloudinary দুই জায়গা থেকে মুছে যায়।

### ৫. Error case

| পাঠাও | ফল |
|---|---|
| `{}` | `400` Nothing to update |
| অন্য patient-এর `reportId` | `404` One or more medical reports not found |
| `"height": "172"` (string) | `400` Height must be a number (cm) |
| `"dateOfBirth": "2099-01-01"` | `400` Date of birth cannot be in the future |
| `"gender": "male"` | `400` Gender must be MALE, FEMALE or OTHER |
| `"shouldDelete": false` | `400` (Zod — শুধু `true` চলে) |
| `data`-তে ভুল JSON | `400` Invalid JSON in 'data' field |
| `profilePhoto` = ৩MB ছবি | `400` File size must be less than 2MB |
| ছবি + ভুল `height` একসাথে | `400`, আর Cloudinary-তে নতুন ছবি **থাকে না** (`deleteUploadedFiles`) |
| DOCTOR login | `403` |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ (Done)

1. ✅ **`patient.utils.ts` বাদ** — এর `convertToDateTime` কোথাও ব্যবহার হতো না, আর install-না-করা `date-fns` import করত (`pnpm typecheck` / `pnpm build`-এ error)। নোট থেকে সরানো হয়েছে; project থেকেও file-টা মুছে দিও।

### 🟡 ভালো practice

2. **`req.user as IRequestUser`।** বাকি module-এ (doctorSchedule, appointment) `getAuthUser(req)` helper আছে, যা `req.user` না থাকলে `401` দেয়। `as` শুধু TypeScript-কে চুপ করায়। একই helper একটা shared file-এ (`src/app/utils/getAuthUser.ts`) রেখে সব জায়গায় ব্যবহার করো।

3. **`reportPublicId` save হয়, কিন্তু মোছার সময় ব্যবহার হয় না।** মোছার সময় `reportLink` থেকে regex দিয়ে public_id বের করা হয়। DB-তে `reportPublicId` থাকলে সেটা দিয়ে সরাসরি `cloudinary.uploader.destroy(publicId)` বেশি নির্ভরযোগ্য।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **`"patientInfo": {}` দিলে।** খালি object-ও "update আছে" ধরে `200` দেয়, কিন্তু কিছু বদলায় না। ক্ষতি নেই।
5. **`pregnancyStatus` সবার জন্য খোলা।** `gender: MALE`-এর জন্য `pregnancyStatus: true` আটকানো নেই — পরে চাইলে Zod `.refine` দিয়ে।

## 🔜 পরে যা যোগ হবে

- [ ] Doctor / Admin update profile (একই pattern)
- [ ] Frontend profile form (form-data + `data` JSON)
