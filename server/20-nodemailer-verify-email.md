# 20 - Nodemailer + Email Verification (OTP)

```
register → Better Auth emailOTP → sendEmail (nodemailer + ejs) → user-এর email-এ OTP
→ POST /auth/verify-email { email, otp } → emailVerified: true → auto login (৩টা cookie)
```

Reset password আর email verify করার জন্য user-এর email-এ code (OTP) পাঠাতে হবে। এজন্য email setup লাগবে।

---

## Part 1: Nodemailer setup

### ধাপ ১: Package install

| Package | কাজ | Type |
|---|---|---|
| [`nodemailer`](https://www.npmjs.com/package/nodemailer) | SMTP দিয়ে email পাঠানো | নিজেই TypeScript support করে, আলাদা type লাগে না |
| [`ejs`](https://www.npmjs.com/package/ejs) | email-এর HTML template | [`@types/ejs`](https://www.npmjs.com/package/@types/ejs) লাগে |

```shell
pnpm add nodemailer
pnpm add ejs
pnpm add -D @types/ejs
```

### ধাপ ২: Gmail App Password নেওয়া

SMTP service হিসেবে Google (Gmail) ব্যবহার করব।

1. Gmail-এ login করো। Account-এ **2-Step Verification চালু** থাকতে হবে।
2. **Manage your Google Account** → উপরের search-এ লেখো **App passwords**।
3. App-এর একটা নাম দাও (যেমন `phdoc`) → একটা ১৬ অক্ষরের password পাবে।
4. এই password-টা `.env`-এর `SMTP_PASS`-এ বসাবে।

### ধাপ ৩: `src/app/utils/email.ts`

```ts
import path from "node:path";
import ejs from "ejs";
import httpStatus from "http-status";
import nodemailer, { type Attachment } from "nodemailer";
import { envVars } from "@/app/config/env";
import AppError from "@/app/errorHelpers/AppError";

const smtpPort = Number(envVars.EMAIL_SENDER.SMTP_PORT);

const transporter = nodemailer.createTransport({
	host: envVars.EMAIL_SENDER.SMTP_HOST,
	port: smtpPort,
	secure: smtpPort === 465, // 465 → SSL, 587 → STARTTLS
	auth: {
		user: envVars.EMAIL_SENDER.SMTP_USER,
		pass: envVars.EMAIL_SENDER.SMTP_PASS,
	},
});

// dev-এ src থেকে, production build-এ dist থেকে template পড়বে
const templateDir =
	envVars.NODE_ENV === "production"
		? path.join(process.cwd(), "dist/templates")
		: path.join(process.cwd(), "src/app/templates");

// নতুন .ejs template বানালে তার নাম এখানে যোগ করবে
type EmailTemplate = "otp";

interface SendEmailOptions {
	to: string;
	subject: string;
	templateName: EmailTemplate;
	templateData: Record<string, unknown>;
	attachments?: Attachment[];
}

export const sendEmail = async ({
	to,
	subject,
	templateName,
	templateData,
	attachments,
}: SendEmailOptions): Promise<void> => {
	try {
		const templatePath = path.join(templateDir, `${templateName}.ejs`);
		const html = await ejs.renderFile(templatePath, templateData);

		await transporter.sendMail({
			from: envVars.EMAIL_SENDER.SMTP_FROM,
			to,
			subject,
			html,
			attachments,
		});
	} catch (error) {
		console.error("Email sending error:", error);
		throw new AppError(httpStatus.INTERNAL_SERVER_ERROR, "Failed to send email");
	}
};
```

- `type EmailTemplate = "otp"`: ভুল template নাম দিলে TypeScript error দেবে।
- `process.cwd()`: server project root থেকে চালাতে হবে (`pnpm dev` / `pnpm start` সেভাবেই চালায়)।

### ধাপ ৪: `src/app/config/env.ts`

```ts
interface EnvConfig {
	// ...আগের সব
	FRONTEND_URL: string;
	EMAIL_SENDER: {
		SMTP_HOST: string;
		SMTP_PORT: string;
		SMTP_USER: string;
		SMTP_PASS: string;
		SMTP_FROM: string;
	};
}
```

Required env-এর list-এ যোগ করো:

```ts
"SMTP_HOST",
"SMTP_PORT",
"SMTP_USER",
"SMTP_PASS",
"SMTP_FROM",
```

Return object-এ:

```ts
EMAIL_SENDER: {
	SMTP_HOST: process.env.SMTP_HOST as string,
	SMTP_PORT: process.env.SMTP_PORT as string,
	SMTP_USER: process.env.SMTP_USER as string,
	SMTP_PASS: process.env.SMTP_PASS as string,
	SMTP_FROM: process.env.SMTP_FROM as string,
},
```

### ধাপ ৫: `.env`

```dotenv
## smtp
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=yourmail@gmail.com
SMTP_PASS=xxxxxxxxxxxxxxxx
SMTP_FROM="PH Doc <yourmail@gmail.com>"
```

> `SMTP_PASS` = ধাপ ২-এর App Password (space ছাড়া), Gmail-এর আসল password না।

### ধাপ ৬: `tsdown.config.ts`

`tsdown` শুধু `.ts` bundle করে, `.ejs` file নেয় না। তাই `copy` দিয়ে template গুলো `dist/templates`-এ কপি করতে হবে।

```ts
import { defineConfig } from "tsdown";

export default defineConfig({
	entry: ["src/server.ts"],
	format: "esm",
	platform: "node",
	target: "node24",
	outDir: "dist",
	clean: true,
	sourcemap: true,
	fixedExtension: false, // output: dist/index.js (index.mjs না)
	copy: [{ from: "src/app/templates", to: "dist/templates" }],
});
```

### ধাপ ৭: `src/app/templates/otp.ejs`

```ejs
<h2>Hello <%= name %></h2>
<p>Your OTP is <b><%= otp %></b>. It will expire in <%= expiresInMinutes %> minutes.</p>
```

- `<%= ... %>` value বসায় আর HTML escape করে (নিরাপদ)।

### ধাপ ৮: Test করার উপায়

যেকোনো জায়গা থেকে একবার call করে দেখো:

```ts
await sendEmail({
	to: "test@example.com",
	subject: "Your OTP",
	templateName: "otp",
	templateData: { name: "Arif", otp: "123456", expiresInMinutes: 2 },
});
```

---

## Part 2: Email verification (OTP দিয়ে)

### ধাপ ১: `src/app/lib/auth.ts`

```ts
// constant ঘোষণা
const OTP_EXPIRES_IN = 2 * 60; // seconds

emailAndPassword: {
	enabled: true,
	requireEmailVerification: true,
},
emailVerification: {
	sendOnSignUp: true,
	sendOnSignIn: true,
	autoSignInAfterVerification: true,
},
plugins: [
	bearer(),
	emailOTP({
		overrideDefaultEmailVerification: true,
		async sendVerificationOTP({ email, otp, type }) {
			// এই প্রজেক্টে শুধু এই দুই type ব্যবহার হচ্ছে
			if (type !== "email-verification" && type !== "forget-password") return;

			const subjects = {
				"email-verification": "Verify your email",
				"forget-password": "Password Reset OTP",
			} as const;

			const user = await prisma.user.findUnique({ where: { email } });

			if (!user) return;
			if (type === "email-verification" && user.emailVerified) return;

			try {
				await sendEmail({
					to: email,
					subject: subjects[type],
					templateName: "otp",
					templateData: {
						name: user.name,
						otp,
						expiresInMinutes: OTP_EXPIRES_IN / 60,
					},
				});
			} catch {
				// sendEmail নিজেই error log করে। এখানে throw করলে signup অর্ধেক হয়ে থাকবে,
				// তাই আটকানো হলো; user login করলে নতুন OTP যাবে
			}
		},
		expiresIn: OTP_EXPIRES_IN,
		otpLength: 6,
	}),
],
```

> ✏️ `bearer()` plugin list-এ আগে থেকেই ছিল; `emailOTP` তার পরে যোগ হয়েছে (নোটে শুধু emailOTP ছিল)।
> ✏️ Import লাগবে — এটা নোটে ছিল না:

```ts
import { bearer, emailOTP } from "better-auth/plugins";
import { sendEmail } from "../utils/email";
import { prisma } from "./prisma";
```

| Option | মানে |
|---|---|
| `requireEmailVerification: true` | email verify না হলে login (`signInEmail`) করতে দেবে না |
| `sendOnSignUp: true` | register হলেই OTP পাঠাবে |
| `sendOnSignIn: true` | unverified user login করতে চাইলে নতুন OTP পাঠাবে |
| `autoSignInAfterVerification: true` | OTP ঠিক হলে সাথে সাথে session (token) দেবে |
| `overrideDefaultEmailVerification: true` | Better Auth-এর default link-verification-এর বদলে OTP ব্যবহার |
| `expiresIn` / `otpLength` | OTP ২ মিনিট valid, ৬ digit |

### ধাপ ২: `auth.interface.ts`

```ts
export interface IVerifyEmailPayload {
	email: string;
	otp: string;
}
```

### ধাপ ৩: `auth.validation.ts`

```ts
const verifyEmailZodSchema = z.object({
	email: z.email("Valid email is required"),
	otp: z.string("OTP is required").regex(/^\d{6}$/, "OTP must be 6 digits"),
});

export const AuthValidation = {
	// ...আগেরগুলো (changePasswordZodSchema)
	verifyEmailZodSchema,
};
```

### ধাপ ৪: `auth.route.ts`

```ts
router.post("/verify-email", validateRequest(AuthValidation.verifyEmailZodSchema), AuthController.verifyEmail);
```

- এখানে `checkAuth` নেই — user তখনো login করা না।

### ধাপ ৫: `auth.controller.ts`

```ts
const verifyEmail = catchAsync(async (req: Request, res: Response) => {
	const result = await AuthService.verifyEmail(req.body);

	// autoSignInAfterVerification: true → verify হওয়ার সাথে সাথে login
	if (result.token !== null) {
		tokenUtils.setAccessTokenCookie(res, result.accessToken);
		tokenUtils.setRefreshTokenCookie(res, result.refreshToken);
		tokenUtils.setBetterAuthSessionCookie(res, result.token);
	}

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Email verified successfully",
		data: result,
	});
});
```

> ✏️ `AuthController` object-এ `verifyEmail` যোগ করতে হবে — এটা নোটে ছিল না।

### ধাপ ৬: `auth.service.ts`

```ts
// JWT বানাতে শুধু এই field-গুলো লাগে, তাই Prisma user-ও পাঠানো যাবে
type TTokenUser = Pick<TAuthUser, "id" | "role" | "name" | "email" | "status" | "isDeleted" | "emailVerified">;

const verifyEmail = async (payload: IVerifyEmailPayload) => {
	const { email, otp } = payload;

	// ভুল/expired OTP হলে Better Auth নিজেই APIError (400) throw করে
	// সঠিক হলে Better Auth নিজেই DB-তে emailVerified: true করে দেয়
	const result = await auth.api.verifyEmailOTP({
		body: { email, otp },
	});

	// autoSignInAfterVerification বন্ধ থাকলে token আসে না
	if (result.token === null) {
		return { token: null, user: result.user };
	}

	// JWT-তে role/status লাগে, তাই DB থেকে পুরো user নেওয়া
	const user = await prisma.user.findUnique({ where: { email: result.user.email } });

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	const { accessToken, refreshToken } = generateAuthTokens(user);

	return {
		token: result.token,
		accessToken,
		refreshToken,
		user: result.user,
	};
};
```

> ✏️ `IVerifyEmailPayload` import আর `AuthService` object-এ `verifyEmail` export লাগবে — এটা নোটে ছিল না।

- Service দুই ধরনের object return করে। Controller-এ `result.token !== null` check করলে TypeScript নিজেই বুঝে যায় `accessToken`/`refreshToken` আছে (type narrowing)।

---

## Test (Postman)

**১. Register**

`POST http://localhost:5000/api/v1/auth/register`

```json
{
	"name": "Arif",
	"email": "তোমার-আসল-email@gmail.com",
	"password": "12345678"
}
```

→ Inbox-এ "Verify your email" mail আসবে (না পেলে Spam দেখো)। OTP ২ মিনিট valid।

**২. Verify email**

`POST http://localhost:5000/api/v1/auth/verify-email`

```json
{
	"email": "তোমার-আসল-email@gmail.com",
	"otp": "email-এ পাওয়া OTP"
}
```

→ `200`, Cookies tab-এ ৩টা cookie। Prisma Studio-তে `user.emailVerified = true`।

**৩. চেক করার মতো আরও case**

| Case | ফল |
|---|---|
| ভুল OTP | `400` (Better Auth) |
| ২ মিনিট পরে OTP | `400` (expired) |
| `otp: "12ab"` | `400` (Zod) |
| Verify না করে login | Better Auth `403` + নতুন OTP mail (`sendOnSignIn`) |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **`generateAuthTokens`-এর parameter type এখনো `TAuthUser`।** `TTokenUser` বানানো হয়েছে, কিন্তু function-এ ব্যবহার হয়নি। তাই `generateAuthTokens(user)`-এ Prisma user পাঠালে `pnpm typecheck`-এ error আসতে পারে (Better Auth আর Prisma-র user type পুরো এক না)। `tsx` type-check করে না, তাই dev-এ চোখে পড়বে না (untested)। আর unused type হিসেবেও থাকবে।

	```ts
	const generateAuthTokens = (user: TTokenUser) => {
		// ...একই
	};
	```

2. **Register এখন unverified user-কে token দেয়।** `requireEmailVerification: true` হওয়ায় `signUpEmail`-এর `token` এখন `null`, কিন্তু `registerPatient` তবুও `accessToken` + `refreshToken` বানিয়ে cookie set করে। `checkAuth` session চায় বলে ঢুকতে পারবে না, তবু verify-এর আগে token দেওয়ার মানে নেই। Register-এ token/cookie বাদ দাও, শুধু "OTP পাঠানো হয়েছে" জানাও।

	```ts
	// auth.service.ts → registerPatient-এর শেষে
	return { user: data.user, patient };

	// auth.controller.ts → registerPatient-এ cookie set করার লাইনগুলো বাদ
	message: "Registration successful. Please check your email for the OTP",
	```

### 🟡 ভালো practice

3. **Resend OTP route নেই।** OTP মাত্র ২ মিনিট valid। Mail দেরিতে এলে বা মিস হলে এখন user-কে login চেষ্টা করে নতুন OTP আনতে হয় — অদ্ভুত UX। আলাদা route বানাও:

	```ts
	// service
	const resendVerificationOtp = async (email: string) => {
		return auth.api.sendVerificationOTP({
			body: { email, type: "email-verification" },
		});
	};

	// route
	router.post("/resend-otp", validateRequest(AuthValidation.resendOtpZodSchema), AuthController.resendVerificationOtp);
	```

4. **Transporter শুরুতেই যাচাই।** ভুল App Password হলে প্রথম mail পাঠানোর সময় বোঝা যায়। Server start-এ একবার `verify()` করলে আগেই জানা যাবে:

	```ts
	transporter.verify().catch((error) => console.error("SMTP connection failed:", error));
	```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **Email পাঠাতে fail হলে user জানতে পারে না।** `sendVerificationOTP`-এর `catch` error গিলে ফেলে, তাই register `200` দেয় কিন্তু mail যায় না। Resend route (৩ নম্বর) থাকলে user নিজেই আবার চাইতে পারবে।
6. **Gmail-এর limit।** Gmail SMTP দিনে কয়েকশো mail পর্যন্ত চলে — শেখার জন্য ঠিক আছে। Production-এ Resend / SendGrid / AWS SES-এর মতো service ভালো; শুধু `.env`-এর SMTP মান বদলালেই চলবে।
7. **OTP-তে ভুল চেষ্টার limit।** Better Auth `emailOTP`-এর `allowedAttempts` (default ৩) — বেশি ভুল দিলে OTP বাতিল হয়, নতুন OTP লাগবে।

## 🔜 পরে যা যোগ হবে

- [ ] Forget password / reset password (`forget-password` type আগেই handle করা আছে)
- [ ] Resend OTP route
- [ ] সুন্দর HTML email template (logo, button)
