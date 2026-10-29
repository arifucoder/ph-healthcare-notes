# 23 - Multer Single Image Upload (Cloudinary)

```
POST /specialties (form-data: file + data) → checkAuth → multer.single("file")
→ cloudinaryStorage._handleFile → file stream সরাসরি Cloudinary-তে → req.file.path = secure_url
→ validateRequest (data JSON string → object → Zod) → controller (icon: req.file?.path) → service → DB
(কোথাও error হলে → globalErrorHandler → req.file থাকলে Cloudinary থেকে মুছে দেয়)
```

| Package | কাজ |
|---|---|
| [`multer`](https://www.npmjs.com/package/multer) | form-data-তে আসা file parse করে — middleware হিসেবে কাজ করে |
| [`cloudinary`](https://www.npmjs.com/package/cloudinary) | file গুলো Cloudinary storage-এ upload করা |

> Teacher `multer-storage-cloudinary` ব্যবহার করেছেন, কিন্তু আমি করব না। তার বদলে নিজের একটা ছোট storage engine বানাব (ধাপ ৪)।

```shell
pnpm add multer
pnpm add -D @types/multer

pnpm add cloudinary
```

---

## ধাপ ১: `.env`-এ যোগ করো

```dotenv
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret
```

> Cloudinary dashboard → **API Keys** থেকে তিনটা মান পাবে।

---

## ধাপ ২: `src/app/config/env.ts`-এ তিন জায়গায় যোগ করো

```ts
// 1) interface EnvConfig-এর ভেতরে
CLOUDINARY: {
	CLOUDINARY_CLOUD_NAME: string;
	CLOUDINARY_API_KEY: string;
	CLOUDINARY_API_SECRET: string;
};

// 2) requireEnvVariable array-তে
"CLOUDINARY_CLOUD_NAME",
"CLOUDINARY_API_KEY",
"CLOUDINARY_API_SECRET",

// 3) return object-এ
CLOUDINARY: {
	CLOUDINARY_CLOUD_NAME: process.env.CLOUDINARY_CLOUD_NAME as string,
	CLOUDINARY_API_KEY: process.env.CLOUDINARY_API_KEY as string,
	CLOUDINARY_API_SECRET: process.env.CLOUDINARY_API_SECRET as string,
},
```

---

## ধাপ ৩: `src/app/config/cloudinary.config.ts` (পুরো file)

```ts
import { v2 as cloudinary, type UploadApiResponse } from "cloudinary";
import httpStatus from "http-status";
import { randomUUID } from "node:crypto";
import AppError from "../errorHelpers/AppError";
import { envVars } from "./env";

cloudinary.config({
	cloud_name: envVars.CLOUDINARY.CLOUDINARY_CLOUD_NAME,
	api_key: envVars.CLOUDINARY.CLOUDINARY_API_KEY,
	api_secret: envVars.CLOUDINARY.CLOUDINARY_API_SECRET,
	secure: true,
});

export const ROOT_FOLDER = "phdoc";

// "My Photo.JPG" → { folder: "images", uniqueName: "<uuid>-my-photo" }
export const buildFileInfo = (fileName: string) => {
	const lastDot = fileName.lastIndexOf(".");
	const hasExtension = lastDot > 0;

	const extension = hasExtension ? fileName.slice(lastDot + 1).toLowerCase() : "";
	const baseName = (hasExtension ? fileName.slice(0, lastDot) : fileName)
		.toLowerCase()
		.replace(/\s+/g, "-")
		.replace(/[^a-z0-9-]/g, "")
		.slice(0, 50);

	const uniqueName = baseName ? `${randomUUID()}-${baseName}` : randomUUID();
	const folder = extension === "pdf" ? "pdfs" : "images";

	return { uniqueName, folder };
};

export const uploadFileToCloudinary = async (buffer: Buffer, fileName: string): Promise<UploadApiResponse> => {
	if (!buffer?.length || !fileName) {
		throw new AppError(httpStatus.BAD_REQUEST, "File buffer and file name are required for upload");
	}

	const { uniqueName, folder } = buildFileInfo(fileName);

	return new Promise((resolve, reject) => {
		const uploadStream = cloudinary.uploader.upload_stream(
			{
				folder: `${ROOT_FOLDER}/${folder}`, // folder শুধু এখানে
				public_id: uniqueName, // এখানে শুধু নাম, path না
				resource_type: "auto",
			},
			(error, result) => {
				if (error || !result) {
					console.error("Cloudinary upload error:", error);
					return reject(new AppError(httpStatus.INTERNAL_SERVER_ERROR, "Failed to upload file to Cloudinary"));
				}
				resolve(result);
			},
		);

		uploadStream.end(buffer);
	});
};

// https://res.cloudinary.com/demo/image/upload/v1712/ph-healthcare/images/abc-photo.jpg
// → "ph-healthcare/images/abc-photo"
const getPublicIdFromUrl = (url: string): string | null => {
	const match = url.match(/\/upload\/(?:v\d+\/)?(.+)\.[a-z0-9]+$/i);
	return match?.[1] ?? null;
};

export const deleteFileFromCloudinary = async (url: string): Promise<void> => {
	const publicId = getPublicIdFromUrl(url);
	if (!publicId) return;

	try {
		// PDF-ও Cloudinary-তে "image" resource_type হিসেবেই save হয়
		await cloudinary.uploader.destroy(publicId, { resource_type: "image", invalidate: true });
	} catch (error) {
		console.error("Error deleting file from Cloudinary:", error);
		throw new AppError(httpStatus.INTERNAL_SERVER_ERROR, "Failed to delete file from Cloudinary");
	}
};

export { cloudinary };
```

| Function | কাজ |
|---|---|
| `buildFileInfo` | file-এর নাম পরিষ্কার করে (ছোট হাতের অক্ষর, space → `-`, ৫০ অক্ষর পর্যন্ত) + সামনে `uuid` বসায়, যাতে একই নামের দুটো file-এর মধ্যে গোলমাল না হয়; PDF হলে `pdfs` folder, না হলে `images` |
| `uploadFileToCloudinary` | RAM-এ থাকা `Buffer` upload করে (পরে memoryStorage বা অন্য কাজে লাগবে) |
| `getPublicIdFromUrl` | URL থেকে `public_id` বের করে — delete করতে এটাই লাগে |
| `deleteFileFromCloudinary` | URL দিয়ে Cloudinary থেকে file মোছে |

---

## ধাপ ৪: `src/app/config/cloudinaryStorage.ts` (নিজের storage engine)

`multer-storage-cloudinary` ব্যবহার না করে Claude-এর সাহায্যে এই file বানানো হয়েছে। এটা `memoryStorage`-এর মতো file RAM-এ রাখে না — সরাসরি Cloudinary-তে stream করে পাঠায়।

```ts
import type { Request } from "express";
import httpStatus from "http-status";
import type { StorageEngine } from "multer";
import AppError from "../errorHelpers/AppError";
import { buildFileInfo, cloudinary, ROOT_FOLDER } from "./cloudinary.config";

/**
 * নিজের বানানো multer storage engine
 * multer-storage-cloudinary-এর কাজই করে, কিন্তু cloudinary v2 দিয়ে
 * upload শেষে: req.file.path = Cloudinary URL, req.file.filename = public_id
 */
export const cloudinaryStorage: StorageEngine = {
	_handleFile(_req: Request, file, cb) {
		const { uniqueName, folder } = buildFileInfo(file.originalname);

		const uploadStream = cloudinary.uploader.upload_stream(
			{
				folder: `${ROOT_FOLDER}/${folder}`,
				public_id: uniqueName,
				resource_type: "auto",
			},
			(error, result) => {
				if (error || !result) {
					console.error("Cloudinary upload error:", error);
					return cb(new AppError(httpStatus.INTERNAL_SERVER_ERROR, "Failed to upload file to Cloudinary"));
				}

				cb(null, {
					path: result.secure_url,
					filename: result.public_id,
					size: result.bytes,
				});
			},
		);

		// RAM-এ না রেখে সরাসরি Cloudinary-তে pipe
		file.stream.pipe(uploadStream);
	},

	// multer-এর ভেতরে পরে কোনো error হলে, upload হওয়া file মুছে দেয়
	_removeFile(_req: Request, file, cb) {
		cloudinary.uploader
			.destroy(file.filename, { resource_type: "image" })
			.then(() => cb(null))
			.catch((error) => cb(error as Error));
	},
};
```

| Method | multer কখন call করে | কী করে |
|---|---|---|
| `_handleFile` | প্রতিটা file আসার সময় | stream → Cloudinary, তারপর `cb(null, { path, filename, size })` — এগুলোই `req.file`-এ বসে |
| `_removeFile` | multer-এর ভেতরে পরে error হলে | আগে upload হওয়া file Cloudinary থেকে মুছে দেয় |

`req.file` এর পর দেখতে এমন:

```ts
req.file.path; // "https://res.cloudinary.com/.../phdoc/images/<uuid>-my-photo.jpg"
req.file.filename; // "phdoc/images/<uuid>-my-photo" (public_id)
req.file.size; // bytes
```

---

## ধাপ ৫: `src/app/config/multer.config.ts` (নতুন file, পুরোটা)

```ts
import httpStatus from "http-status";
import multer from "multer";
import AppError from "../errorHelpers/AppError";
import { cloudinaryStorage } from "./cloudinaryStorage";

const ALLOWED_MIME_TYPES = ["image/jpeg", "image/png", "image/webp", "application/pdf"];
// ✅ limit একটাই জায়গায় — globalErrorHandler-এর message-ও এটা ব্যবহার করে
export const MAX_FILE_SIZE_MB = 2;
const MAX_FILE_SIZE = MAX_FILE_SIZE_MB * 1024 * 1024; // bytes

export const multerUpload = multer({
	storage: cloudinaryStorage,
	limits: { fileSize: MAX_FILE_SIZE },
	fileFilter: (_req, file, cb) => {
		if (!ALLOWED_MIME_TYPES.includes(file.mimetype)) {
			return cb(new AppError(httpStatus.BAD_REQUEST, "Only JPG, PNG, WEBP images and PDF files are allowed"));
		}
		cb(null, true);
	},
});
```

- `fileFilter` upload-এর **আগে** চলে — ভুল type হলে Cloudinary-তে কিছুই যায় না।
- `limits.fileSize` ছাড়ালে multer `MulterError` (`LIMIT_FILE_SIZE`) দেয় → ধাপ ৬-এ handle।
- `MAX_FILE_SIZE_MB` export করা হয়েছে, যাতে error message-এ আলাদা করে সংখ্যা লিখতে না হয়। Limit বদলালে শুধু এখানে বদলালেই হবে।

> ✏️ আগে `MAX_FILE_SIZE = 2 * 1024 * 1024` ছিল, কিন্তু error message-এ "5MB" লেখা ছিল — দুই জায়গায় দুই সংখ্যা। এখন একটা constant।

---

## ধাপ ৬: `globalErrorHandler.ts` — `MulterError` branch + upload হওয়া file মোছা

### ক) `MulterError` branch

```ts
import multer from "multer"; // উপরে import-এ
import { MAX_FILE_SIZE_MB } from "../config/multer.config";

	// `else if (err instanceof Error)`-এর ঠিক আগে বসাও
	} else if (err instanceof multer.MulterError) {
		statusCode = httpStatus.BAD_REQUEST;
		message = err.code === "LIMIT_FILE_SIZE" ? `File size must be less than ${MAX_FILE_SIZE_MB}MB` : err.message;
		errorSources = [{ path: err.field ?? "", message }];
		stack = err.stack;
	}
```

- `MulterError`-ও একটা `Error`, তাই `instanceof Error`-এর **আগে** বসাতে হবে — নাহলে সাধারণ `Error` branch আগে ধরে ফেলবে।

> ✏️ Message-এ এখন `MAX_FILE_SIZE_MB` (আগে ভুল করে "5MB" লেখা ছিল)।

### খ) Error হলে upload হওয়া file Cloudinary থেকে মোছা

Route-এ `multer` আগে চলে, তাই file **আগেই** Cloudinary-তে চলে যায়। এরপর Zod, DB বা অন্য যেকোনো জায়গায় error হলে ছবিটা অকারণে পড়ে থাকত। সব error শেষে `globalErrorHandler`-এ আসে, তাই এখানে একবার লিখলেই সব route-এর জন্য কাজ করবে।

```ts
import { deleteFileFromCloudinary } from "../config/cloudinary.config";

// ✅ async — file মোছা শেষ হওয়া পর্যন্ত অপেক্ষা করে
export const globalErrorHandler = async (err: unknown, req: Request, res: Response, _next: NextFunction) => {
	// error হলে এই request-এ upload হওয়া file মুছে দাও
	// .catch → মুছতে fail করলেও আসল error-টাই user দেখবে
	if (req.file?.path) {
		await deleteFileFromCloudinary(req.file.path).catch(() => {});
	}

	// ...বাকি code একই (isDev, statusCode, ZodError, AppError, APIError, MulterError, Error...)
};
```

| কোথায় error | আগে | এখন |
|---|---|---|
| Zod (যেমন `title` নেই) | ছবি Cloudinary-তে থেকে যেত | মুছে যায় |
| DB (যেমন duplicate title) | service-এর `try/catch` মুছত | `globalErrorHandler` মোছে |
| Multer নিজে (size, type) | `_removeFile` / upload-ই হয় না | একই |

- Express 5 async error handler-ও ঠিকমতো চালায়।
- `req.file` থাকে শুধু `.single()`-এ। পরে `.array()` / `.fields()` ব্যবহার করলে `req.files`-ও মুছতে হবে।

> ✏️ এই অংশ নোটে ছিল না — Claude Recommendation থেকে যোগ হয়েছে (untested)।

---

## ধাপ ৭: `app.ts` — `express.urlencoded`

```ts
app.use(express.urlencoded({ extended: true }));
```

`express.urlencoded()` request body পড়ার একটা parser। এটা শুধু `application/x-www-form-urlencoded` format-এর body পড়ে `req.body`-তে object বানিয়ে দেয়।

| Body format | কে পড়ে |
|---|---|
| `application/json` | `express.json()` |
| `application/x-www-form-urlencoded` | `express.urlencoded()` |
| `multipart/form-data` (file সহ) | `multer` — file → `req.file`, text field → `req.body` |

---

## ধাপ ৮: Specialty create-এ file upload — `specialty.route.ts`

```ts
router.post(
	"/",
	checkAuth(Role.ADMIN, Role.SUPER_ADMIN),
	multerUpload.single("file"),
	validateRequest(SpecialtyValidation.createSpecialtyZodSchema),
	SpecialtyController.createSpecialty,
);
```

- আগে authentication check (`checkAuth`), তারপর file upload — login না করা কেউ Cloudinary-তে file পাঠাতে পারবে না।

> **⚠️ `checkAuth` কখনো comment করে রেখো না।** Test-এর জন্য comment করলে, test শেষে অবশ্যই uncomment করবে — নাহলে login ছাড়াই যে কেউ তোমার Cloudinary-তে file ভরে ফেলতে পারবে। আর `checkAuth` অবশ্যই `multerUpload`-এর **আগে** থাকবে, পরে দিলে auth check-এর আগেই file upload হয়ে যাবে।
- `.single("file")` → form-data-র `file` নামের একটা file, ফল `req.file`-এ।

> ✏️ `import { multerUpload } from "@/app/config/multer.config";` — এটা নোটে ছিল না।

---

## ধাপ ৯: `validateRequest`-এ `data` JSON parse

File-এর সাথে বাকি data form-data-র `data` field-এ **JSON string** হিসেবে আসে (`req.body.data`)। তাই `validateRequest` middleware-এ একটা condition যোগ করতে হবে:

```ts
// form-data থেকে আসা JSON string → object
if (typeof req.body?.data === "string") {
	try {
		req.body = JSON.parse(req.body.data);
	} catch {
		return next(new AppError(httpStatus.BAD_REQUEST, "Invalid JSON in 'data' field"));
	}
}
```

পুরো middleware:

```ts
export const validateRequest = (zodSchema: ZodType) => {
	return async (req: Request, _res: Response, next: NextFunction) => {
		// form-data থেকে আসা JSON string → object
		if (typeof req.body?.data === "string") {
			try {
				req.body = JSON.parse(req.body.data);
			} catch {
				return next(new AppError(httpStatus.BAD_REQUEST, "Invalid JSON in 'data' field"));
			}
		}

		const parsedResult = await zodSchema.safeParseAsync(req.body);

		if (!parsedResult.success) {
			return next(parsedResult.error);
		}

		// sanitize করা data
		req.body = parsedResult.data;
		next();
	};
};
```

> ✏️ পুরো middleware নোটে ছিল না — project থেকে নেওয়া। `AppError` আর `httpStatus` import লাগবে। Unused `res` → `_res`।

> **গুরুত্বপূর্ণ:** এই route-এ `validateRequest` দেওয়া খুবই জরুরি। এটা না থাকলে `req.body.data` কখনো `req.body`-তে convert হবে না, আর controller-এ `{ data: "..." }` string যাবে।

---

## ধাপ ১০: `specialty.validation.ts`

```ts
import * as z from "zod";

const createSpecialtyZodSchema = z.object({
	title: z.string("Title is required").trim(),
	description: z.string("Description is required").trim().optional(),
});

// icon আসে req.file থেকে, body থেকে না — তাই Zod-এর বাইরে আলাদা যোগ
export type TCreateSpecialtyPayload = z.infer<typeof createSpecialtyZodSchema> & {
	icon?: string;
};

export const SpecialtyValidation = {
	createSpecialtyZodSchema,
};
```

---

## ধাপ ১১: `specialty.controller.ts`

```ts
const createSpecialty = catchAsync(async (req: Request, res: Response) => {
	const payload = {
		...req.body,
		icon: req.file?.path,
	};

	const result = await SpecialtyService.createSpecialty(payload);
	sendResponse(res, {
		httpStatusCode: 201,
		success: true,
		message: "Specialty created successfully",
		data: result,
	});
});
```

- `req.file?.path` → Cloudinary URL (file না দিলে `undefined`, আর `icon` field `String?` তাই চলবে)।

---

## ধাপ ১২: `specialty.service.ts`

```ts
import { prisma } from "@/app/lib/prisma";
import type { Specialty } from "@/generated/prisma/client";
import type { TCreateSpecialtyPayload } from "./specialty.validation";

const createSpecialty = async (payload: TCreateSpecialtyPayload): Promise<Specialty> => {
	// DB fail করলে (যেমন duplicate title) error → globalErrorHandler → ছবি মুছে যায় (ধাপ ৬-খ)
	return await prisma.specialty.create({ data: payload });
};
```

> ✏️ আগে এখানে `try/catch` দিয়ে `deleteFileFromCloudinary` call হতো। এখন `globalErrorHandler` সব error-এ file মুছে দেয়, তাই এটা সরানো হয়েছে — দুইবার মুছতে যাওয়ার দরকার নেই।

আগের version (শুধু জানার জন্য):

```ts
const createSpecialty = async (payload: TCreateSpecialtyPayload): Promise<Specialty> => {
	try {
		return await prisma.specialty.create({ data: payload });
	} catch (error) {
		// DB fail করলে (যেমন duplicate title) upload হওয়া ছবি মুছে দাও
		if (payload.icon) await deleteFileFromCloudinary(payload.icon);
		throw error;
	}
};
```

---

## Test (Postman)

| | |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5000/api/v1/specialties` |
| Auth | ADMIN / SUPER_ADMIN login-এর cookie |
| Body | **form-data** |

| Key | Type | Value |
|---|---|---|
| `file` | File | একটা `.png` / `.jpg` (২MB-এর কম) |
| `data` | Text | `{"title": "Cardiology", "description": "Heart related"}` |

→ `201`, response-এর `icon`-এ Cloudinary URL। Cloudinary Media Library → `phdoc/images/`-এ file দেখবে।

| Case | ফল |
|---|---|
| ২MB-এর বড় file | `400` (`MulterError` — `LIMIT_FILE_SIZE`) |
| `.gif` / `.txt` file | `400` "Only JPG, PNG, WEBP images and PDF files are allowed" |
| `data`-তে ভুল JSON | `400` "Invalid JSON in 'data' field" |
| `title` ছাড়া | `400` (Zod) + Cloudinary থেকে ছবি মুছে যায় |
| একই `title` দুবার | DB error + Cloudinary থেকে ছবি মুছে যায় (`globalErrorHandler`) |
| file ছাড়া | `201`, `icon: null` |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ (Done)

1. ✅ **File size message ভুল ছিল** (limit 2MB, message 5MB) — `MAX_FILE_SIZE_MB` constant (ধাপ ৫ ও ৬-ক)।
2. ✅ **`checkAuth` comment করা** — নোটে active রাখা হয়েছে, ধাপ ৮-এ সতর্কবার্তা। Project-এর `specialty.route.ts`-এ নিজে uncomment করে নিও।
3. ✅ **Error হলে ছবি Cloudinary-তে পড়ে থাকত** — `globalErrorHandler`-এ মোছা (ধাপ ৬-খ), service থেকে `try/catch` সরানো (ধাপ ১২)।
5. ✅ **Service-এ মুছতে গিয়ে আসল error হারিয়ে যেত** — service-এ আর মোছা হয় না, আর handler-এ `.catch(() => {})` থাকায় আসল error-ই দেখা যায়।

### 🟡 ভালো practice

4. **ফাঁকা `title` চলে যায়।** `.trim()` আছে কিন্তু `.min(1)` নেই, তাই `"   "` → `""` হয়ে valid হয়ে যায়।

	```ts
	title: z.string("Title is required").trim().min(1, "Title is required"),
	```

6. **`201`-এর বদলে `httpStatus.CREATED`।** বাকি project-এ `httpStatus` ব্যবহার হয় — একই রকম রাখো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

7. **`mimetype` client পাঠায়।** কেউ `.exe`-এর নাম বদলে `image/png` বলে পাঠাতে পারে। Cloudinary `resource_type: "auto"` নিজে file যাচাই করে, তাই বড় ঝুঁকি নেই — তবে জেনে রাখো।
8. **Project-এর `globalErrorHandler`-এ পুরনো কিছু জিনিস।** উপরে `/* eslint-disable ... */` আর `console.log` আছে, আর `req`/`next` unused। oxlint-এ `no-console` শুধু `warn`/`error` চলে — `console.log` → `console.warn`, comment সরিয়ে `_req`/`_next` করে দিও।
9. **Update/delete-এ পুরনো ছবি।** Specialty delete বা icon বদলালে পুরনো ছবি `deleteFileFromCloudinary(oldIcon)` দিয়ে মুছতে হবে, নাহলে Cloudinary ভরে যাবে।

## 🔜 পরে যা যোগ হবে

- [ ] Multiple file upload (`multerUpload.array` / `.fields`) — তখন `globalErrorHandler`-এ `req.files`-ও মুছতে হবে
- [ ] Specialty update-এ নতুন ছবি দিলে পুরনোটা মোছা
- [ ] Specialty delete-এ Cloudinary থেকে ছবি মোছা
- [ ] Doctor / patient profile photo upload
