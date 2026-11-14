# ClinicOS — Email OTP Verification

## ১. SMTP environment variable

Test-এর জন্য আমরা SMTP হিসেবে Mailtrap use করলাম।

### ১.১ `.env` file

**File:** `.env`

```env
# ── Email (patient invite, verification) ─────
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
EMAIL_FROM="ClinicOS <no-reply@yourdomain.com>"
```

### ১.২ `env.ts`-এ add করা

এবার `env.ts` file-এ add করতে হবে:

**File:** `env.ts`

```ts
const envSchema = z
	.object({
//...

		SMTP_HOST: z.string().trim().min(1),
		SMTP_PORT: z.coerce.number().int().min(1).max(65535),
		SMTP_USER: z.string().min(1),
		SMTP_PASS: z.string().min(1),
		EMAIL_FROM: z
			.string()
			.trim()
			.regex(/^.+ <[^\s@<>]+@[^\s@<>]+\.[^\s@<>]+>$/, 'Must look like "ClinicOS <no-reply@yourdomain.com>"'),
	})
```

## ২. Package install

`nodemailer` আর `ejs` install করো:

```bash
pnpm add nodemailer
pnpm add -D @types/nodemailer

pnpm add ejs
pnpm add -D @types/ejs
```

## ৩. Email template structure

```text
apps/api/
├── templates/
│   └── emails/
│       ├── layout.ejs
│       ├── verificationCode.ejs
│       ├── invoice.ejs
│       └── prescription.ejs
└── src/...
```

> **নোট:** এটা প্রয়োজন অনুযায়ী বানিয়ে নিবো, এজন্য এখানে দিলাম না।

## ৪. কী কী বাকি আছে

```text
#	কী	অবস্থা
1	env.ts-এ CLINIC_ADDRESS, CLINIC_PHONE	নেই, layout-এর footer-এ লাগে
2	src/app/lib/mailer.ts	নেই
3	src/app/shared/renderEmail.ts	নেই
4	src/app/shared/sendEmail.ts	নেই
5	Local-এ email ধরার জন্য Mailpit	নেই
6	OTP বানানো, save করা, /auth/verify-email, /auth/resend-otp	নেই, এটা আলাদা feature
```

## ৫. Email পাঠানোর setup

### ৫.১ `env.ts`-এ দুটো লাইন

`CLINIC_NAME`-এর নিচে বসাও:

**File:** `env.ts`

```ts
CLINIC_ADDRESS: z.string().trim().min(1).default("123 Health Avenue, Dhaka"),
CLINIC_PHONE: z.string().trim().min(1).default("(555) 010-0100"),
```

### ৫.২ Mailer

**File:** `src/app/lib/mailer.ts`

```ts
import { envVars } from "@/app/config/env";
import { createTransport } from "nodemailer";

export const mailer = createTransport({
	host: envVars.SMTP_HOST,
	port: envVars.SMTP_PORT,
	secure: envVars.SMTP_PORT === 465,
	auth: {
		user: envVars.SMTP_USER,
		pass: envVars.SMTP_PASS,
	},
});
```

### ৫.৩ Email render করা

**File:** `src/app/shared/renderEmail.ts`

```ts
import { envVars, isProduction } from "@/app/config/env";
import ejs from "ejs";
import path from "node:path";

const TEMPLATE_DIR = path.join(process.cwd(), "templates", "emails");

export type TEmailTemplates = {
	verificationCode: {
		name: string;
		code: string;
		expiresInMinutes: number;
		expiresAtLabel: string;
	};
};

type TLayoutData = {
	title: string;
	previewText: string;
};

const templatePath = (name: string) => path.join(TEMPLATE_DIR, `${name}.ejs`);

export const renderEmail = async <T extends keyof TEmailTemplates>(
	template: T,
	data: TEmailTemplates[T],
	layout: TLayoutData,
): Promise<string> => {
	const common = {
		clinicName: envVars.CLINIC_NAME,
		clinicAddress: envVars.CLINIC_ADDRESS,
		clinicPhone: envVars.CLINIC_PHONE,
		year: new Date().getFullYear(),
	};
	const options = { cache: isProduction };

	const bodyHtml = await ejs.renderFile(templatePath(template), { ...common, ...data }, options);

	return ejs.renderFile(templatePath("layout"), { ...common, ...layout, bodyHtml }, options);
};
```

### ৫.৪ Email পাঠানো

**File:** `src/app/shared/sendEmail.ts`

```ts
import { envVars } from "@/app/config/env";
import { mailer } from "@/app/lib/mailer";
import { renderEmail, type TEmailTemplates } from "./renderEmail";

type TSendEmailOptions<T extends keyof TEmailTemplates> = {
	to: string;
	subject: string;
	previewText: string;
	template: T;
	data: TEmailTemplates[T];
};

export const sendEmail = async <T extends keyof TEmailTemplates>(options: TSendEmailOptions<T>) => {
	const { to, subject, previewText, template, data } = options;

	const html = await renderEmail(template, data, { title: subject, previewText });

	await mailer.sendMail({
		from: envVars.EMAIL_FROM,
		to,
		subject,
		html,
	});
};
```

## ৬. `auth.ts`-এ `emailOTP` plugin

**File:** `src/lib/auth.ts`

```ts
import { envVars } from "@/app/config/env";
import { prisma } from "@/app/lib/prisma";
import { sendEmail } from "@/app/shared/sendEmail";
import { Role } from "@/generated/prisma/enums";
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { emailOTP } from "better-auth/plugins";

const OTP_EXPIRES_MINUTES = 2;

const formatClinicTime = (date: Date) =>
	new Intl.DateTimeFormat("en-GB", {
		hour: "2-digit",
		minute: "2-digit",
		timeZone: envVars.CLINIC_TIME_ZONE,
	}).format(date);

export const auth = betterAuth({
	baseURL: envVars.BETTER_AUTH_URL,
	secret: envVars.BETTER_AUTH_SECRET,

	database: prismaAdapter(prisma, {
		provider: "postgresql",
	}),

	emailAndPassword: {
		enabled: true,
		requireEmailVerification: true,
	},

	user: {
		additionalFields: {
			role: { type: "string", input: false, defaultValue: Role.PATIENT },
			isActive: { type: "boolean", input: false, defaultValue: true },
			needPasswordChange: { type: "boolean", input: false, defaultValue: false },
		},
	},

	plugins: [
		emailOTP({
			otpLength: 6,
			expiresIn: OTP_EXPIRES_MINUTES * 60,
			allowedAttempts: 3,
			storeOTP: "hashed",
			overrideDefaultEmailVerification: true,

			async sendVerificationOTP({ email, otp, type }) {
				if (type !== "email-verification") return;

                const user = await prisma.user.findUnique({
                    where: { email },
                    select: { name: true, emailVerified: true },
                });
                if (user?.emailVerified) return;

				const expiresAt = new Date(Date.now() + OTP_EXPIRES_MINUTES * 60_000);

				// await নেই: response-এর সময় দেখে যেন বোঝা না যায় email-টা আছে কিনা
				void sendEmail({
					to: email,
					subject: `${otp} is your ${envVars.CLINIC_NAME} verification code`,
					previewText: `Use this code to verify your email. It expires in ${OTP_EXPIRES_MINUTES} minutes.`,
					template: "verificationCode",
					data: {
						name: user?.name ?? "there",
						code: otp,
						expiresInMinutes: OTP_EXPIRES_MINUTES,
						expiresAtLabel: formatClinicTime(expiresAt),
					},
				}).catch((error: unknown) => {
					console.error("Failed to send verification email", error);
				});
			},
		}),
	],

	advanced: {
		database: {
			generateId: () => crypto.randomUUID(),
		},
	},
});
```

## ৭. Registration-এর সময় OTP email পাঠানো

এবার registration-এর সময় mail পাঠাতে হবে।

1. এজন্য `auth.service.ts` file-এ `registerPatient` update করো:

**File:** `auth.service.ts`

```ts
const registerPatient = async (payload: TRegisterPayload) => {
	const { name, email, password } = payload;

	const existingUser = await prisma.user.findUnique({
		where: { email },
		select: { id: true },
	});

	if (existingUser) {
		throw new AppError(httpStatus.CONFLICT, "An account with this email already exists");
	}

	const hashedPassword = await hashPassword(password);
	const userId = crypto.randomUUID();

	const result = await prisma.$transaction(async (tx) => {
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

	try {
		await auth.api.sendVerificationOTP({
			body: { email, type: "email-verification" },
		});
	} catch (error) {
		console.error("Failed to send verification OTP", error);
	}

	return result;
};
```

2. আর `auth.controller.ts` file-এ message বদলাও:

**File:** `auth.controller.ts`

```ts
message: "Account created. Check your email for the verification code",
```

## ৮. OTP verification API

এবার OTP verification API বানাবো।

### ৮.১ Validation

**File:** `auth.validation.ts`

```ts
const emailSchema = z.string().trim().toLowerCase().pipe(z.email("Invalid email address"));
const verifyEmailSchema = z.object({
	email: emailSchema,
	otp: z.string().regex(/^\d{6}$/, "Code must be exactly 6 digits"),
});
```

### ৮.২ Type

**File:** `auth.service.ts`

```ts
export type TVerifyEmailPayload = z.infer<typeof AuthValidation.verifyEmailSchema>;
```

### ৮.৩ Better Auth error handler

**File:** `src/app/errorHelpers/handleBetterAuthError.ts` (নতুন)

```ts
import type { APIError } from "better-auth/api";
import httpStatus from "http-status";
import type { TGenericErrorResponse } from "./error.interface";

// Better Auth-এর error code → আমাদের নিজের status আর message
const AUTH_ERRORS: Record<string, { statusCode: number; message: string }> = {
	INVALID_OTP: {
		statusCode: httpStatus.BAD_REQUEST,
		message: "Invalid verification code",
	},
	OTP_EXPIRED: {
		statusCode: httpStatus.BAD_REQUEST,
		message: "Verification code has expired. Request a new one",
	},
	TOO_MANY_ATTEMPTS: {
		statusCode: httpStatus.TOO_MANY_REQUESTS,
		message: "Too many wrong attempts. Request a new code",
	},
};

export const handleBetterAuthError = (err: APIError): TGenericErrorResponse => {
	const code = err.body?.code;
	const known = code ? AUTH_ERRORS[code] : undefined;

	if (known) {
		return { statusCode: known.statusCode, message: known.message, errorSources: [] };
	}

	// Better Auth-এর ভেতরের সমস্যার বিবরণ client-কে দেখানো হয় না
	if (err.statusCode >= 500) {
		return {
			statusCode: httpStatus.INTERNAL_SERVER_ERROR,
			message: "Something went wrong",
			errorSources: [],
		};
	}

	return {
		statusCode: err.statusCode,
		message: err.body?.message ?? "Request failed",
		errorSources: [],
	};
};
```

### ৮.৪ Global error handler

**File:** `src/app/middleware/globalErrorHandler.ts`

```ts
else if (isAPIError(err)) {
		errorResponse = handleBetterAuthError(err);
	} else if (err instanceof Error && isDev) {
		errorResponse.message = err.message;
	}
```

### ৮.৫ Service

**File:** `auth.service.ts`

```ts
const verifyEmail = async (payload: TVerifyEmailPayload) => {
	const { email, otp } = payload;

	const { user } = await auth.api.verifyEmailOTP({
		body: { email, otp },
	});

	return { email: user.email, emailVerified: user.emailVerified };
};
```

### ৮.৬ Controller

**File:** `auth.controller.ts`

```ts
const verifyEmail = catchAsync(async (req, res) => {
	const result = await AuthService.verifyEmail(req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Email verified successfully",
		data: result,
	});
});
```

### ৮.৭ Route

**File:** `auth.route.ts`

```ts
router.post(
	"/verify-email",
	authRateLimiter,
	validateRequest(AuthValidation.verifyEmailSchema),
	AuthController.verifyEmail,
);
```

## ৯. Resend OTP

### ৯.১ Validation

**File:** `auth.validation.ts`

```ts
const resendOtpSchema = z.object({
	email: emailSchema,
});

export const AuthValidation = {
	resendOtpSchema,
};
```

### ৯.২ Interface

**File:** `auth.interface.ts`

```ts
export type TResendOtpPayload = z.infer<typeof AuthValidation.resendOtpSchema>;
```

### ৯.৩ Service

**File:** `auth.service.ts`

```ts
const resendOtp = async (payload: TResendOtpPayload) => {
    const { email } = payload;

    await auth.api.sendVerificationOTP({
        body: { email, type: "email-verification" },
    });
};
```

### ৯.৪ Controller

**File:** `auth.controller.ts`

```ts
const resendOtp = catchAsync(async (req, res) => {
	await AuthService.resendOtp(req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "If an account exists for this email, a new code has been sent",
		data: null,
	});
});
```

### ৯.৫ Route

**File:** `auth.route.ts`

```ts
router.post(
    "/resend-otp",
    authRateLimiter,
    validateRequest(AuthValidation.resendOtpSchema),
    AuthController.resendOtp,
);
```