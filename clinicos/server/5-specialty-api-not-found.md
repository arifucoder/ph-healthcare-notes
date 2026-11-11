# Specialty module (create)

## ১. Folder structure

```text
src/app/
├─ shared/slugify.ts              (নতুন)
├─ middleware/notFound.ts         (নতুন)
├─ routes/index.ts                (নতুন)
├─ module/specialty/
│  ├─ specialty.validation.ts
│  ├─ specialty.interface.ts
│  ├─ specialty.service.ts
│  ├─ specialty.controller.ts
│  └─ specialty.route.ts
src/app.ts                        (update)
```

## ২. slugify helper

**File:** `src/app/shared/slugify.ts`

```ts
export const slugify = (value: string): string =>
	value
		.normalize("NFKD")
		.replace(/[̀-ͯ]/g, "")
		.toLowerCase()
		.replace(/['’]/g, "")
		.replace(/[^a-z0-9]+/g, "-")
		.replace(/^-+|-+$/g, "");
```

## ৩. Specialty module

### specialty.validation.ts

**File:** `src/app/module/specialty/specialty.validation.ts`

```ts
import * as z from "zod";

const createSpecialtySchema = z.object({
	name: z
		.string()
		.trim()
		.min(2, "Name must be at least 2 characters")
		.max(100)
		.regex(/[a-z0-9]/i, "Name must contain letters or numbers"),
	description: z.string().trim().max(500).optional(),
	icon: z.string().trim().max(255).optional(),
});

export const SpecialtyValidation = { createSpecialtySchema };
```

### specialty.interface.ts

**File:** `src/app/module/specialty/specialty.interface.ts`

```ts
import type * as z from "zod";
import type { SpecialtyValidation } from "./specialty.validation";

export type TCreateSpecialtyPayload = z.infer<typeof SpecialtyValidation.createSpecialtySchema>;
```

### specialty.service.ts

**File:** `src/app/module/specialty/specialty.service.ts`

```ts
import { prisma } from "@/app/lib/prisma";
import { slugify } from "@/app/shared/slugify";
import type { TCreateSpecialtyPayload } from "./specialty.interface";

const createSpecialty = async (payload: TCreateSpecialtyPayload) => {
	return prisma.specialty.create({
		data: { ...payload, slug: slugify(payload.name) },
		omit: { isDeleted: true, deletedAt: true },
	});
};

export const SpecialtyService = { createSpecialty };
```

### specialty.controller.ts

**File:** `src/app/module/specialty/specialty.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { SpecialtyService } from "./specialty.service";

const createSpecialty = catchAsync(async (req, res) => {
	const result = await SpecialtyService.createSpecialty(req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: "Specialty created successfully",
		data: result,
	});
});

export const SpecialtyController = { createSpecialty };
```

### specialty.route.ts

**File:** `src/app/module/specialty/specialty.route.ts`

```ts
import { validateRequest } from "@/app/middleware/validateRequest";
import { Router } from "express";
import { SpecialtyController } from "./specialty.controller";
import { SpecialtyValidation } from "./specialty.validation";

const router = Router();

// TODO: auth(Role.ADMIN, Role.SUPER_ADMIN) — Better Auth আসার পর
router.post(
	"/",
	validateRequest(SpecialtyValidation.createSpecialtySchema),
	SpecialtyController.createSpecialty,
);

export const SpecialtyRoutes = router;
```

## ৪. Routes index

**File:** `src/app/routes/index.ts`

```ts
import { SpecialtyRoutes } from "@/app/module/specialty/specialty.route";
import { Router } from "express";

const router = Router();

const moduleRoutes = [{ path: "/specialties", route: SpecialtyRoutes }];

for (const { path, route } of moduleRoutes) {
	router.use(path, route);
}

export const AppRoutes = router;
```

## ৫. notFound middleware

**File:** `src/app/middleware/notFound.ts`

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import type { RequestHandler } from "express";
import httpStatus from "http-status";

export const notFound: RequestHandler = (req, _res, next) => {
	next(new AppError(httpStatus.NOT_FOUND, `Route ${req.method} ${req.originalUrl} not found`));
};
```

## ৬. app.ts update

**File:** `src/app.ts` (শুধু যা যোগ হবে)

```ts
import { globalErrorHandler } from "@/app/middleware/globalErrorHandler";
import { notFound } from "@/app/middleware/notFound";
import { AppRoutes } from "@/app/routes";

// ... health route-এর পরে, export-এর আগে:
app.use("/api/v1", AppRoutes);

app.use(notFound); // কোনো route match না করলে
app.use(globalErrorHandler); // সবার শেষে

```