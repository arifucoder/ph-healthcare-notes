# Error handling আর shared utility

## ১. Install

```bash
pnpm add zod http-status
```

## ২. Error handling

`src/app/errorHelpers/` folder-এর structure:

```text
src/app/errorHelpers/
├─ AppError.ts            (আছে)
├─ error.interface.ts     (নতুন: shared type)
├─ handleZodError.ts      (নতুন)
└─ handlePrismaError.ts   (নতুন)
```

### AppError.ts

**File:** `src/app/errorHelpers/AppError.ts`

```ts
export class AppError extends Error {
	public readonly statusCode: number;

	constructor(statusCode: number, message: string, options?: ErrorOptions) {
		super(message, options);
		this.name = "AppError";
		this.statusCode = statusCode;
	}
}
```

### error.interface.ts

**File:** `src/app/errorHelpers/error.interface.ts`

```ts
export type TErrorSource = {
	path: string;
	message: string;
};

export type TGenericErrorResponse = {
	statusCode: number;
	message: string;
	errorSources: TErrorSource[];
};

```

### handleZodError.ts

**File:** `src/app/errorHelpers/handleZodError.ts`

```ts
import httpStatus from "http-status";
import type { ZodError } from "zod";
import type { TGenericErrorResponse } from "./error.interface";

export const handleZodError = (err: ZodError): TGenericErrorResponse => ({
	statusCode: httpStatus.BAD_REQUEST,
	message: "Validation failed",
	errorSources: err.issues.map((issue) => ({
		path: issue.path.map(String).join(".") || "root",
		message: issue.message,
	})),
});
```

### handlePrismaError.ts

**File:** `src/app/errorHelpers/handlePrismaError.ts`

```ts
import type { Prisma } from "@/generated/prisma/client";
import httpStatus from "http-status";
import type { TGenericErrorResponse } from "./error.interface";

const isRecord = (value: unknown): value is Record<string, unknown> => typeof value === "object" && value !== null;

const toStringArray = (value: unknown): string[] =>
	Array.isArray(value) ? value.filter((item): item is string => typeof item === "string") : [];

const getUniqueFields = (meta: Record<string, unknown> | undefined): string[] => {
	// Prisma 7 + driver adapter (pg): meta.driverAdapterError.cause.constraint.fields
	const adapterError = meta?.driverAdapterError;
	if (isRecord(adapterError) && isRecord(adapterError.cause) && isRecord(adapterError.cause.constraint)) {
		const fields = toStringArray(adapterError.cause.constraint.fields);
		if (fields.length) return fields;
	}
	return toStringArray(meta?.target);
};

export const handlePrismaKnownError = (err: Prisma.PrismaClientKnownRequestError): TGenericErrorResponse => {
	switch (err.code) {
		case "P2002": {
			const fields = getUniqueFields(err.meta);
			const label = fields.length ? fields.join(", ") : "value";
			return {
				statusCode: httpStatus.CONFLICT,
				message: `Duplicate ${label}`,
				errorSources: fields.map((field) => ({ path: field, message: `${field} already exists` })),
			};
		}
		case "P2003":
			return {
				statusCode: httpStatus.BAD_REQUEST,
				message: "Related record does not exist or is still in use",
				errorSources: [],
			};
		case "P2025":
			return {
				statusCode: httpStatus.NOT_FOUND,
				message: "Record not found",
				errorSources: [],
			};
		case "P2000":
			return {
				statusCode: httpStatus.BAD_REQUEST,
				message: "Value is too long for the field",
				errorSources: [],
			};
		default:
			return {
				statusCode: httpStatus.INTERNAL_SERVER_ERROR,
				message: "Database error",
				errorSources: [],
			};
	}
};

export const handlePrismaValidationError = (): TGenericErrorResponse => ({
	statusCode: httpStatus.BAD_REQUEST,
	message: "Invalid data sent to database",
	errorSources: [],
});
```

## ৩. Global error handler

**File:** `src/app/middleware/globalErrorHandler.ts`

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import type { TGenericErrorResponse } from "@/app/errorHelpers/error.interface";
import { handlePrismaKnownError, handlePrismaValidationError } from "@/app/errorHelpers/handlePrismaError";
import { handleZodError } from "@/app/errorHelpers/handleZodError";
import { Prisma } from "@/generated/prisma/client";
import type { ErrorRequestHandler } from "express";
import httpStatus from "http-status";
import { ZodError } from "zod";

const isDev = process.env.NODE_ENV === "development";

export const globalErrorHandler: ErrorRequestHandler = (err, _req, res, _next) => {
	let errorResponse: TGenericErrorResponse = {
		statusCode: httpStatus.INTERNAL_SERVER_ERROR,
		message: "Something went wrong",
		errorSources: [],
	};

	if (err instanceof ZodError) {
		errorResponse = handleZodError(err);
	} else if (err instanceof Prisma.PrismaClientKnownRequestError) {
		errorResponse = handlePrismaKnownError(err);
	} else if (err instanceof Prisma.PrismaClientValidationError) {
		errorResponse = handlePrismaValidationError();
	} else if (err instanceof AppError) {
		errorResponse = { statusCode: err.statusCode, message: err.message, errorSources: [] };
	} else if (err instanceof Error && isDev) {
		errorResponse.message = err.message;
	}

	if (errorResponse.statusCode >= 500) console.error(err);

	res.status(errorResponse.statusCode).json({
		success: false,
		message: errorResponse.message,
		errorSources: errorResponse.errorSources,
		...(isDev && err instanceof Error && { stack: err.stack }),
	});
};
```

## ৪. Shared utility

`src/app/shared/` folder-এর structure:

```text
src/app/shared/
├── catchAsync.ts
└── sendResponse.ts
```

### catchAsync

**File:** `src/app/shared/catchAsync.ts`

```ts
import type { NextFunction, Request, RequestHandler, Response } from "express";

export const catchAsync = (fn: RequestHandler): RequestHandler => {
	return async (req: Request, res: Response, next: NextFunction) => {
		try {
			await fn(req, res, next);
		} catch (error) {
			next(error);
		}
	};
};
```

### sendResonse.ts

```ts
import type { Response } from "express";

export type TMeta = {
	page: number;
	limit: number;
	total: number;
	totalPages: number;
};

type TSendResponse<T> = {
	statusCode: number;
	message: string;
	data?: T;
	meta?: TMeta;
};

export const sendResponse = <T>(
	res: Response,
	{ statusCode, message, data, meta }: TSendResponse<T>,
) => {
	res.status(statusCode).json({
		success: true,
		message,
		...(meta && { meta }),
		data,
	});
};
```