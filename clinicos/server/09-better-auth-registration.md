# ClinicOS — Better Auth দিয়ে Patient Registration

## ১. Package install

```bash
pnpm add better-auth jsonwebtoken cookie-parser
```

## ২. Environment variable

### ২.১ `env.ts`-এ validation

**File:** `env.ts`

```ts
const DURATION_REGEX = /^\d+(?:\.\d+)?\s?(?:ms|s|m|h|d|w|y)$/i;

const isValidTimeZone = (timeZone: string): boolean => {
	try {
		new Intl.DateTimeFormat("en-US", { timeZone }).format();
		return true;
	} catch {
		return false;
	}
};
const secret = z.string().min(32, "Must be at least 32 characters (generate: openssl rand -base64 32)");
const duration = z.string().regex(DURATION_REGEX, 'Must be a duration like "15m", "1d" or "7d"');
const envSchema = z
	.object({
		CLINIC_TIME_ZONE: z.string().refine(isValidTimeZone, "Must be a valid IANA time zone, e.g. Asia/Dhaka"),
		CLINIC_NAME: z.string().trim().min(1).default("ClinicOS"),

		BETTER_AUTH_SECRET: secret,
		BETTER_AUTH_URL: z.url({ protocol: /^https?$/ }),
		ACCESS_TOKEN_SECRET: secret,
		ACCESS_TOKEN_EXPIRES_IN: duration,
		REFRESH_TOKEN_SECRET: secret,
		REFRESH_TOKEN_EXPIRES_IN: duration,

		COOKIE_DOMAIN: z
			.string()
			.regex(/^\.?[a-z0-9-]+(\.[a-z0-9-]+)+$/i, 'Must be a domain like ".yourdomain.com"')
			.optional(),
	})
	.superRefine((env, ctx) => {
		if (env.ACCESS_TOKEN_SECRET === env.REFRESH_TOKEN_SECRET) {
			ctx.addIssue({
				code: "custom",
				path: ["REFRESH_TOKEN_SECRET"],
				message: "Must be different from ACCESS_TOKEN_SECRET",
			});
		}
		if (env.NODE_ENV === "production" && !env.BETTER_AUTH_URL.startsWith("https://")) {
			ctx.addIssue({
				code: "custom",
				path: ["BETTER_AUTH_URL"],
				message: "Must use https in production",
			});
		}
	});
```

### ২.২ `.env` file

**File:** `.env`

```env
# ── Clinic ───────────────────────────────────
# IANA time zone that defines the clinic's "day" (all times stored in UTC)
CLINIC_TIME_ZONE=Asia/Dhaka
CLINIC_NAME=ClinicOS Demo Clinic

# ── Auth (Better Auth + JWT cookies) ─────────
# Generate each secret with: openssl rand -base64 32
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=http://localhost:5000
ACCESS_TOKEN_SECRET=
ACCESS_TOKEN_EXPIRES_IN=1d
REFRESH_TOKEN_SECRET=
REFRESH_TOKEN_EXPIRES_IN=7d

# Production only, e.g. .yourdomain.com (leave empty for localhost)
COOKIE_DOMAIN=
```

## ৩. Auth instance বানানো

**File:** `src/app/lib/auth.ts`

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "./prisma";

export const auth = betterAuth({
	database: prismaAdapter(prisma, {
		provider: "postgresql",
	}),

	emailAndPassword: {
		enabled: true,
	},
});
```

## ৪. CLI দিয়ে Auth Schema Generate

Doc: https://better-auth.com/docs/concepts/cli#generate

Better Auth নিজে **৪টা model** generate করে দেয় (`User`, `Session`, `Account`, `Verification`)।

দুইটা গুরুত্বপূর্ণ option:

| Option | কেন লাগছে |
|---|---|
| `--output` | Default-এ model গুলো `schema.prisma`-তে লেখে। আমরা **multi-file schema** use করি, তাই বলে দিচ্ছি `prisma/schema/auth.prisma` file-এ লিখতে। |
| `--config` | Better Auth default-এ root / `lib` folder-এ `auth.ts` খোঁজে। আমাদের file আছে `src/app/lib/auth.ts`-এ, তাই path বলে দিতে হয়। |

1. আগে `prisma/schema/auth.prisma` নামে একটা খালি file বানিয়ে নাও।
2. তারপর এই command চালাও:

```bash
pnpm dlx auth@latest generate \
  --output ./prisma/schema/auth.prisma \
  --config ./src/app/lib/auth.ts \
  --adapter prisma
```

## ৫. Enums

**File:** `prisma/schema/enums.prisma`

```prisma
enum Role {
    SUPER_ADMIN
    ADMIN
    DOCTOR
    PATIENT
}

enum Gender {
  MALE
  FEMALE
  OTHER
}

enum BloodGroup {
  A_POSITIVE
  A_NEGATIVE
  B_POSITIVE
  B_NEGATIVE
  AB_POSITIVE
  AB_NEGATIVE
  O_POSITIVE
  O_NEGATIVE
}
```

## ৬. User model-এ custom field

1. `auth.prisma`-তে `User` model-এ custom field যোগ করো:

**File:** `prisma/schema/auth.prisma`

```prisma
model User {
  id                 String    @id
  name               String
  email              String
  emailVerified      Boolean   @default(false)
  image              String?
  role               Role      @default(PATIENT)
  isActive           Boolean   @default(true)
  needPasswordChange Boolean   @default(false)
  isDeleted          Boolean   @default(false)
  deletedAt          DateTime?
  createdAt          DateTime  @default(now())
  updatedAt          DateTime  @updatedAt

  sessions Session[]
  accounts Account[]

  @@unique([email])
  @@map("user")
}
```

2. তারপর migrate আর generate চালাও:

```bash
pnpm migrate && pnpm generate
```

## ৭. `auth.ts`-এ config update

**File:** `src/app/lib/auth.ts`

```ts
import { envVars } from "@/app/config/env";
import { prisma } from "@/app/lib/prisma";
import { Role } from "@/generated/prisma/enums";
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";

export const auth = betterAuth({
	baseURL: envVars.BETTER_AUTH_URL,
	secret: envVars.BETTER_AUTH_SECRET,

	database: prismaAdapter(prisma, {
		provider: "postgresql",
	}),

	emailAndPassword: {
		enabled: true,
	},

	user: {
		additionalFields: {
			role: { type: "string", input: false, defaultValue: Role.PATIENT },
			isActive: { type: "boolean", input: false, defaultValue: true },
			needPasswordChange: { type: "boolean", input: false, defaultValue: false },
		},
	},

	advanced: {
		database: {
			generateId: () => crypto.randomUUID(),
		},
	},
});
```

## ৮. Patient schema

আমরা যেহেতু patient registration-এর সময় transaction (`tx`) করে `patient` table-এও data পাঠাবো, তাই আমাদের `Patient`-এর schema বানাতে হবে।

1. `clinic.prisma`-তে `Patient` model লেখো:

**File:** `clinic.prisma`

```prisma
model Patient {
  id       String @id @default(uuid())
  userId   String @unique
  fullName String

  phone       String?
  dateOfBirth DateTime? @db.Date
  gender      Gender?
  address     String?

  bloodGroup            BloodGroup?
  allergies             String?
  emergencyContactName  String?
  emergencyContactPhone String?

  registeredById String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  user         User  @relation("PatientAccount", fields: [userId], references: [id], onDelete: Cascade)
  registeredBy User? @relation("PatientRegisteredBy", fields: [registeredById], references: [id])

  @@index([phone])
  @@index([fullName])
  @@map("patients")
}
```

2. তারপর terminal-এ এই command গুলো চালাও:

```bash
pnpm prisma format
pnpm prisma validate
pnpm prisma migrate dev --name add_patient
pnpm check
pnpm generate
```

## ৯. Auth module

নতুন folder: `src/app/module/auth/`

### ৯.১ Validation

**File:** `auth.validation.ts`

```ts
import * as z from "zod";

const registerSchema = z.object({
	name: z.string().trim().min(2, "Name must be at least 2 characters").max(100),
	email: z.string().trim().toLowerCase().pipe(z.email("Invalid email address")),
	password: z.string().min(8, "Password must be at least 8 characters").max(128),
});

export const AuthValidation = {
	registerSchema,
};
```

### ৯.২ Interface

**File:** `auth.interface.ts`

```ts
import type * as z from "zod";
import type { AuthValidation } from "./auth.validation";

export type TRegisterPayload = z.infer<typeof AuthValidation.registerSchema>;
```

### ৯.৩ Service

**File:** `auth.service.ts`

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { Role } from "@/generated/prisma/enums";
import { hashPassword } from "better-auth/crypto";
import httpStatus from "http-status";
import type { TRegisterPayload } from "./auth.interface";

const registerPatient = async (payload: TRegisterPayload) => {
	const { name, email, password } = payload;

	const existingUser = await prisma.user.findUnique({
		where: { email },
		select: { id: true },
	});

	if (existingUser) {
		throw new AppError(httpStatus.CONFLICT, "An account with this email already exists");
	}

	// hash করা ধীর কাজ, তাই transaction-এর বাইরে
	const hashedPassword = await hashPassword(password);
	const userId = crypto.randomUUID();

	return prisma.$transaction(async (tx) => {
		const user = await tx.user.create({
			data: { id: userId, name, email, role: Role.PATIENT },
			select: { id: true, name: true, email: true, role: true, emailVerified: true },
		});

		await tx.account.create({
			data: {
				id: crypto.randomUUID(),
				userId,
				accountId: userId,
				providerId: "credential",
				password: hashedPassword,
			},
		});

		const patient = await tx.patient.create({
			data: { userId, fullName: name },
			select: { id: true, fullName: true },
		});

		return { user, patient };
	});
};

export const AuthService = {
	registerPatient,
};
```

### ৯.৪ Controller

**File:** `auth.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { AuthService } from "./auth.service";

const registerPatient = catchAsync(async (req, res) => {
	const result = await AuthService.registerPatient(req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: "Account created successfully",
		data: result,
	});
});

export const AuthController = {
	registerPatient,
};
```

### ৯.৫ Route

**File:** `auth.route.ts`

```ts
import { validateRequest } from "@/app/middleware/validateRequest";
import { Router } from "express";
import { AuthController } from "./auth.controller";
import { AuthValidation } from "./auth.validation";

const router = Router();

// TODO: rate limiter
router.post("/register", validateRequest(AuthValidation.registerSchema), AuthController.registerPatient);

export const AuthRoutes = router;
```

### ৯.৬ Route register করা

**File:** `routes/index.ts`

```ts
import { AuthRoutes } from "@/app/module/auth/auth.route";

const moduleRoutes = [
	{ path: "/auth", route: AuthRoutes },
	{ path: "/specialties", route: SpecialtyRoutes },
];
```

## ১০. Rate limiter

### ১০.১ Rate limiter কী

Rate limiter মানে: একটা নির্দিষ্ট সময়ে একজন client কতবার request করতে পারবে, তার সীমা। সীমা পার হলে server `429 Too Many Requests` দেয়।

**কীভাবে কাজ করে:** Server প্রতিটা client-এর (সাধারণত IP ধরে) একটা গণনা রাখে:

```text
IP 103.x.x.x → এই ১৫ মিনিটে 4টা request   ✅
IP 103.x.x.x → ৬ নম্বর request              ❌ 429
(১৫ মিনিট পর গণনা আবার শূন্য)
```

### ১০.২ Package install

```bash
pnpm add express-rate-limit
```

### ১০.৩ Middleware বানানো

**File:** `src/app/middleware/rateLimiter.ts`

```ts
import { rateLimit } from "express-rate-limit";

export const authRateLimiter = rateLimit({
	windowMs: 15 * 60 * 1000,
	limit: 10,
	standardHeaders: "draft-7",
	legacyHeaders: false,
});
```

### ১০.৪ Route-এ যোগ করা

**File:** `auth.route.ts`

```ts
router.post("/register", authRateLimiter, validateRequest(...), AuthController.registerPatient);
```