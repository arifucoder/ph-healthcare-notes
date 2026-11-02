# 36 - Prisma Error Handle করা (globalErrorHandler)

**সমস্যা:** `findUniqueOrThrow` থেকে `500` error আসছিল। Doctor বা patient profile না পেলে Prisma `P2025` error throw করে। কিন্তু `globalErrorHandler`-এ Prisma error-এর আলাদা কোনো branch ছিল না, তাই client পেত `500 Internal Server Error`, অথচ পাওয়ার কথা `404`।

**সমাধান দুই রকম:**
1. Service-এ `findUnique` করে নিজে `AppError(NOT_FOUND)` throw করা (message নিজের মতো দেওয়া যায়)।
2. আর যেগুলো তবুও বের হয়ে যায় (`update`/`delete`-এ record না পেলে, duplicate email ইত্যাদি), সেগুলোর জন্য `globalErrorHandler`-এ Prisma error-এর branch।

```
service → Prisma error throw → catchAsync → globalErrorHandler
       → instanceof PrismaClientKnownRequestError? → handlePrismaKnownError (code দেখে status)
       → instanceof PrismaClientValidationError?   → handlePrismaValidationError (400)
```

> Global error handler-এর আগের অংশ আছে [নোট ১২](./12-global-error-handler.md)-এ।

### কোন Prisma code → কোন HTTP status

| Code | কখন হয় | Status |
| --- | --- | --- |
| `P2002` | Unique constraint ভাঙলে (একই email দুইবার) | `409 Conflict` |
| `P2003` | Foreign key ভাঙলে (নেই এমন `doctorId`, বা relation থাকা record delete) | `400 Bad Request` |
| `P2025` | Record পাওয়া যায়নি (`findUniqueOrThrow` / `update` / `delete`) | `404 Not Found` |
| `P2023` | ভুল format-এর data (uuid column-এ `"abc"`) | `400 Bad Request` |
| অন্য code | — | `400 Bad Request` |
| `PrismaClientValidationError` | Query-র shape ভুল (number field-এ string) | `400 Bad Request` |

### `findUniqueOrThrow` বনাম `findUnique` + `AppError`

```ts
// ❌ আগে: profile না পেলে P2025 → (handler ছাড়া) 500
const doctor = await prisma.doctor.findUniqueOrThrow({ where: { email: user.email } });

// ✅ এখন: নিজের message সহ 404
const doctor = await prisma.doctor.findUnique({ where: { email: user.email } });

if (!doctor) {
	throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
}
```

> নিচের handler যোগ করার পরে `findUniqueOrThrow` দিলেও `404` আসবে (message: `"Doctor not found"`)। তবু `findUnique` + `AppError` ভালো — message নিজের হাতে থাকে আর `isDeleted` check-ও একই জায়গায় করা যায়।

---

## ধাপ ১: নতুন file — `src/app/errorHelpers/handlePrismaError.ts`

```ts
import httpStatus from "http-status";
import type { Prisma } from "@/generated/prisma/client";
import type { TGenericErrorResponse } from "../interfaces/error.interface";

// duplicate হওয়া field বের করা
// সাধারণ Prisma → meta.target, driver adapter (adapter-pg) → meta.driverAdapterError.cause.constraint.fields
const getConflictFields = (meta: Record<string, unknown> | undefined): string => {
	if (Array.isArray(meta?.target)) {
		return meta.target.join(", ");
	}

	const adapterError = meta?.driverAdapterError as
		| { cause?: { constraint?: { fields?: string[] } } }
		| undefined;

	return adapterError?.cause?.constraint?.fields?.join(", ") ?? "";
};

// findUniqueOrThrow, update, delete ইত্যাদিতে record না পেলে, unique/foreign key ভাঙলে
export const handlePrismaKnownError = (err: Prisma.PrismaClientKnownRequestError): TGenericErrorResponse => {
	switch (err.code) {
		// Unique constraint ভঙ্গ (যেমন একই email দুবার)
		case "P2002": {
			const fields = getConflictFields(err.meta);
			const message = fields ? `${fields} already exists` : "Duplicate value already exists";
			return {
				statusCode: httpStatus.CONFLICT,
				message,
				errorSources: [{ path: fields, message }],
			};
		}

		// Foreign key ভঙ্গ (যেমন এমন doctorId দিলে যেটা নেই, বা relation থাকা record delete)
		case "P2003": {
			const message = "Related record does not exist or is still in use";
			return {
				statusCode: httpStatus.BAD_REQUEST,
				message,
				errorSources: [{ path: "", message }],
			};
		}

		// Record পাওয়া যায়নি (findUniqueOrThrow / update / delete)
		case "P2025": {
			const modelName = typeof err.meta?.modelName === "string" ? err.meta.modelName : "Record";
			const message = `${modelName} not found`;
			return {
				statusCode: httpStatus.NOT_FOUND,
				message,
				errorSources: [{ path: "", message }],
			};
		}

		// ভুল format-এর data (যেমন uuid column-এ "abc")
		case "P2023": {
			const message = "Invalid id or data format";
			return {
				statusCode: httpStatus.BAD_REQUEST,
				message,
				errorSources: [{ path: "", message }],
			};
		}

		default: {
			const message = "Database request error";
			return {
				statusCode: httpStatus.BAD_REQUEST,
				message,
				errorSources: [{ path: "", message: `${err.code}: ${message}` }],
			};
		}
	}
};

// query-র shape ভুল হলে (যেমন number field-এ string পাঠানো)
export const handlePrismaValidationError = (): TGenericErrorResponse => {
	const message = "Invalid data provided to database query";
	return {
		statusCode: httpStatus.BAD_REQUEST,
		message,
		errorSources: [{ path: "", message }],
	};
};
```

**কয়েকটা জিনিস খেয়াল করো:**
- **`import type { Prisma }`** — এখানে `Prisma` শুধু type হিসেবে লাগছে (`err: Prisma.PrismaClientKnownRequestError`), তাই `import type`।
- **`getConflictFields` কেন দুই জায়গায় খোঁজে?** আমরা `PrismaPg` driver adapter ব্যবহার করি। Adapter থাকলে duplicate field-এর নাম `meta.target`-এ না এসে `meta.driverAdapterError.cause.constraint.fields`-এ আসে। দুই জায়গাতেই দেখা হচ্ছে যাতে যেকোনো setup-এ কাজ করে।
- **`err.meta` এর type `Record<string, unknown>`** — তাই `typeof ... === "string"` বা `Array.isArray` দিয়ে check করে তারপর ব্যবহার (`unknown`-কে সরাসরি ব্যবহার করা যায় না)।
- **প্রতিটা `case`-এ `{ }` block** — যাতে প্রতিটা case-এ আলাদা `const message` রাখা যায় (না হলে "already declared" error)।
- Return type `TGenericErrorResponse` — `handleZodError`-এর মতো একই shape, তাই handler-এ একই ভাবে ব্যবহার করা যায়।

---

## ধাপ ২: `globalErrorHandler.ts`-এ পরিবর্তন

উপরে import:

```ts
import { Prisma } from "@/generated/prisma/client"; // ✅ value import, কারণ instanceof-এ লাগবে
import { handlePrismaKnownError, handlePrismaValidationError } from "../errorHelpers/handlePrismaError";
```

> এখানে `import type` দিলে চলবে না — `instanceof`-এর জন্য runtime-এ আসল class লাগে।

`AppError` branch-এর পরে, `APIError`-এর আগে দুইটা branch:

```ts
} else if (err instanceof AppError) {
	statusCode = err.statusCode;
	message = err.message;
	errorSources = [{ path: "", message: err.message }];
	stack = err.stack;
} else if (err instanceof Prisma.PrismaClientKnownRequestError) {
	const simplifiedError = handlePrismaKnownError(err);
	statusCode = simplifiedError.statusCode;
	message = simplifiedError.message;
	errorSources = simplifiedError.errorSources;
	stack = err.stack;
} else if (err instanceof Prisma.PrismaClientValidationError) {
	const simplifiedError = handlePrismaValidationError();
	statusCode = simplifiedError.statusCode;
	message = simplifiedError.message;
	errorSources = simplifiedError.errorSources;
	stack = err.stack;
} else if (err instanceof APIError) {
	// ... আগের মতোই
```

### পুরো `globalErrorHandler.ts` (এখন যেমন)

`src/app/middleware/globalErrorHandler.ts`

```ts
import { Prisma } from "@/generated/prisma/client"; // ✅ value import, কারণ instanceof-এ লাগবে
import { APIError } from "better-auth";
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status";
import multer from "multer";
import { ZodError } from "zod"; // ✅ value import, কারণ instanceof-এ লাগবে
import { envVars } from "../config/env";
import AppError from "../errorHelpers/AppError";
import { handlePrismaKnownError, handlePrismaValidationError } from "../errorHelpers/handlePrismaError";
import { handleZodError } from "../errorHelpers/handleZodError";
import type { TErrorResponse, TErrorSources } from "../interfaces/error.interface";
import { deleteUploadedFiles } from "../utils/deleteUploadedFiles";

export const globalErrorHandler = async (err: unknown, req: Request, res: Response, _next: NextFunction) => {
	const isDev = envVars.NODE_ENV === "development";

	if (isDev) {
		console.log("Error from Global Error Handler", err);
	}

	// error হলে এই request-এ upload হওয়া file আর রাখার দরকার নেই
	await deleteUploadedFiles(req);

	let statusCode: number = httpStatus.INTERNAL_SERVER_ERROR;
	let message = "Internal Server Error";
	let errorSources: TErrorSources[] = [{ path: "", message }];
	let stack: string | undefined;

	if (err instanceof ZodError) {
		const simplifiedError = handleZodError(err);
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources;
		stack = err.stack;
	} else if (err instanceof AppError) {
		statusCode = err.statusCode;
		message = err.message;
		errorSources = [{ path: "", message: err.message }];
		stack = err.stack;
	} else if (err instanceof Prisma.PrismaClientKnownRequestError) {
		const simplifiedError = handlePrismaKnownError(err);
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources;
		stack = err.stack;
	} else if (err instanceof Prisma.PrismaClientValidationError) {
		const simplifiedError = handlePrismaValidationError();
		statusCode = simplifiedError.statusCode;
		message = simplifiedError.message;
		errorSources = simplifiedError.errorSources;
		stack = err.stack;
	} else if (err instanceof APIError) {
		statusCode = err.statusCode;
		message = err.message;
		errorSources = [{ path: "", message: err.message }];
		stack = err.stack;
	} else if (err instanceof multer.MulterError) {
		statusCode = httpStatus.BAD_REQUEST;
		message = err.code === "LIMIT_FILE_SIZE" ? "File size must be less than 2MB" : err.message;
		errorSources = [{ path: err.field ?? "", message }];
		stack = err.stack;
	} else if (err instanceof Error) {
		// production-এ internal message লুকানো হচ্ছে
		message = isDev ? err.message : "Something went wrong";
		errorSources = [{ path: "", message }];
		stack = err.stack;
	}

	const errorResponse: TErrorResponse = {
		success: false,
		message,
		errorSources,
		error: isDev ? err : undefined,
		stack: isDev ? stack : undefined,
	};

	res.status(statusCode).json(errorResponse);
};
```

> ✏️ eslint-disable comment সরানো হয়েছে (আমরা oxlint ব্যবহার করি) — unused param-এ _ দেওয়া হয়েছে।

**Branch-এর ক্রম:** নির্দিষ্ট error class (`ZodError`, `AppError`, Prisma, `APIError`, `MulterError`) আগে, আর সাধারণ `Error` সবার শেষে। কারণ সবগুলোই আসলে `Error`-এর child — `err instanceof Error` আগে থাকলে বাকিগুলোতে কখনো পৌঁছাতো না। নির্দিষ্ট class-গুলোর নিজেদের মধ্যে ক্রম যেকোনো রকম হতে পারে।

---

## 🧪 Test

| Case | কীভাবে | Expected |
| --- | --- | --- |
| `P2002` | একই email দিয়ে দুইবার doctor create | `409` — `"email already exists"` |
| `P2025` | নেই এমন id দিয়ে `update`/`delete` (service-এ আগে check না থাকলে) | `404` — `"<Model> not found"` |
| `P2003` | নেই এমন `specialtyId` দিয়ে doctor-এ specialty যোগ | `400` |
| `P2023` | uuid column-এ `"abc"` দিয়ে খোঁজা | `400` — `"Invalid id or data format"` |
| Profile নেই | Doctor profile ছাড়া user দিয়ে doctor route | `404` (আর `500` না) |

Error response এরকম আসবে:

```json
{
	"success": false,
	"message": "email already exists",
	"errorSources": [{ "path": "email", "message": "email already exists" }]
}
```

> Development-এ `error` আর `stack`-ও আসবে। `meta`-তে আসল field কোথায় আছে দেখতে terminal-এর console log দেখো।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`console.log` → `console.error`** — আমাদের oxlint-এর `no-console` rule শুধু `warn` আর `error` allow করে, তাই `console.log`-এ lint warning/error আসবে। Error log-এর জন্য `console.error`-ই সঠিক:
   ```ts
   if (isDev) {
   	console.error("Error from Global Error Handler", err);
   }
   ```

2. **অচেনা Prisma code-এ `400` না দিয়ে `500`** — `default`-এ সব অচেনা code `400` (client-এর দোষ) হয়ে যাচ্ছে। কিন্তু কিছু code server-এর সমস্যা, যেমন `P2024` (connection pool timeout)। এগুলোতে `400` দিলে মনে হবে client ভুল করেছে, আর আসল bug চোখ এড়িয়ে যাবে:
   ```ts
   default: {
   	const message = "Database request error";
   	return {
   		statusCode: httpStatus.INTERNAL_SERVER_ERROR,
   		message,
   		errorSources: [{ path: "", message: `${err.code}: ${message}` }],
   	};
   }
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **DB বন্ধ থাকলে** — Database-এ connect করতে না পারলে `PrismaClientInitializationError` আসে, যেটা এখানে আলাদা ধরা হয়নি। তাই শেষের `Error` branch-এ গিয়ে `500` হয় — এটাই ঠিক। চাইলে আলাদা branch দিয়ে `503 Service Unavailable` দিতে পারো।

4. **Service-এ নিজে `P2002` ধরা এখনো কাজের** — Prescription আর review service-এ `P2002` ধরে নিজস্ব message দেওয়া হয়েছিল (`"Prescription already exists for this appointment"`)। Global handler থাকলেও ওগুলো রেখে দাও — global message (`"appointmentId already exists"`) user-এর জন্য অতটা পরিষ্কার না।
