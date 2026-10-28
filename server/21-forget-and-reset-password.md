# 21 - Forget Password & Reset Password (OTP দিয়ে)

> Module: 39-08

```
Forget: POST /auth/forget-password { email } → user check → requestPasswordResetEmailOTP → reset-password-otp.ejs mail
Reset:  POST /auth/reset-password { email, otp, newPassword } → user check → resetPasswordEmailOTP
        → নতুন password + সব session মুছে → needPasswordChange: false → আবার login
```

Frontend-এ user **Forget password**-এ click করলে তার email চাওয়া হবে। ওই email-এ একটা OTP যাবে। তারপর frontend-এ একটা form-এ OTP আর নতুন password নেওয়া হবে। OTP মিললে password বদলাবে, না মিললে বদলাবে না।

---

## Part 1: Forget password

### ধাপ ১: `src/app/lib/auth.ts`

`forget-password` type আগেই ([নোট ২০](./20-nodemailer-verify-email.md)) handle করা ছিল। এবার শুধু আলাদা email template ব্যবহারের জন্য `templates` map যোগ হলো।

```ts
export const auth = betterAuth({
	// ...বাকি code

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

				// type অনুযায়ী কোন .ejs template যাবে
				const templates = {
					"email-verification": "otp",
					"forget-password": "reset-password-otp",
				} as const;

				const user = await prisma.user.findUnique({ where: { email } });

				if (!user) return;
				if (type === "email-verification" && user.emailVerified) return;

				try {
					await sendEmail({
						to: email,
						subject: subjects[type],
						templateName: templates[type],
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
});
```

- `as const` থাকায় `templates[type]`-এর type হয় `"otp" | "reset-password-otp"` — তাই `sendEmail`-এর `EmailTemplate` type-এর সাথে মিলে যায়।

### ধাপ ২: `src/app/templates/reset-password-otp.ejs`

```ejs
<h2>Hello <%= name %></h2>
<p>We received a request to reset your password.</p>
<p>Your OTP is <b><%= otp %></b>. It will expire in <%= expiresInMinutes %> minutes.</p>
<p>If you didn't request this, you can ignore this email.</p>
```

### ধাপ ৩: `src/app/utils/email.ts`

নতুন template-এর নাম type-এ যোগ করা:

```ts
type EmailTemplate = "otp" | "reset-password-otp";
```

> এটা না দিলে `templateName: templates[type]`-এ type error আসবে।

### ধাপ ৪: `auth.interface.ts`

```ts
export interface IForgetPasswordPayload {
	email: string;
}
```

### ধাপ ৫: `auth.validation.ts`

```ts
const forgetPasswordZodSchema = z.object({
	email: z.string("Email is required").trim().toLowerCase().pipe(z.email("Valid email is required")),
});
```

- আগে `trim()` + `toLowerCase()` করে, তারপর `.pipe()` দিয়ে email কিনা check করে। তাই `" Arif@Gmail.com "` → `"arif@gmail.com"` হয়ে DB-তে মিলবে।

> ✏️ `AuthValidation` object-এ `forgetPasswordZodSchema` যোগ করতে হবে — এটা নোটে ছিল না।

### ধাপ ৬: `auth.service.ts`

```ts
const forgetPassword = async (email: string) => {
	const user = await prisma.user.findUnique({
		where: { email },
	});

	// ✅ শুধু active, verified, deleted না — এমন user-ই OTP পাবে
	const canReset =
		user && !user.isDeleted && user.status === UserStatus.ACTIVE && user.emailVerified;

	// ✅ account না থাকলেও error না দিয়ে চুপচাপ ফেরত যাওয়া —
	// তাহলে response দেখে কেউ বুঝতে পারবে না এই email-এর account আছে কিনা
	if (!canReset) return;

	// OTP বানিয়ে type: "forget-password" দিয়ে sendVerificationOTP-কে call করে
	await auth.api.requestPasswordResetEmailOTP({
		body: { email },
	});
};
```

> ✏️ `AuthService` object-এ `forgetPassword` export করতে হবে — এটা নোটে ছিল না।
> ✏️ আগে না থাকলে `404`, blocked হলে `403`, verify না করা হলে `400` দিত — তাতে বোঝা যেত কোন email-এর account আছে (user enumeration)। এখন সব ক্ষেত্রে একই `200`, শুধু যোগ্য user-এর কাছে OTP যায়।

| User-এর অবস্থা | আগে | এখন |
|---|---|---|
| নেই / deleted | `404` | `200`, mail যায় না |
| blocked | `403` | `200`, mail যায় না |
| email verify করা না | `400` | `200`, mail যায় না |
| active + verified | `200` + mail | `200` + mail |

### ধাপ ৭: `auth.controller.ts`

```ts
const forgetPassword = catchAsync(async (req: Request, res: Response) => {
	const payload: IForgetPasswordPayload = req.body;

	await AuthService.forgetPassword(payload.email);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		// ✅ "sent" নিশ্চিত করে বলা হয় না, কারণ account না থাকলে mail যায় না
		message: "If an account exists with this email, an OTP has been sent",
	});
});
```

- `sendResponse`-এ `data` optional, তাই না দিলেও চলে (response-এ `data` থাকবে না)।

> ✏️ `import type { IForgetPasswordPayload } from "./auth.interface";` আর `AuthController` object-এ `forgetPassword` যোগ — এটা নোটে ছিল না।

### ধাপ ৮: `auth.route.ts`

```ts
router.post(
	"/forget-password",
	validateRequest(AuthValidation.forgetPasswordZodSchema),
	AuthController.forgetPassword,
);
```

- `checkAuth` নেই — user password ভুলে গেছে, তাই login করা থাকবে না।

---

## Part 2: Reset password

### ধাপ ১: `lib/auth.ts` → `emailAndPassword`-এ একটা লাইন যোগ

```ts
emailAndPassword: {
	enabled: true,
	requireEmailVerification: true,
	revokeSessionsOnPasswordReset: true,
},
```

- `revokeSessionsOnPasswordReset: true` → password reset হলে ওই user-এর **সব session** মুছে যায়। কেউ চুরি করে login করে থাকলে সেও বের হয়ে যাবে।

### ধাপ ২: `auth.interface.ts`

```ts
export interface IResetPasswordPayload {
	email: string;
	otp: string;
	newPassword: string;
}
```

### ধাপ ৩: `auth.validation.ts`

```ts
const resetPasswordZodSchema = z.object({
	email: z
		.string("Email is required")
		.trim()
		.toLowerCase()
		.pipe(z.email("Valid email is required")),
	otp: z.string("OTP is required").regex(/^\d{6}$/, "OTP must be 6 digits"),
	newPassword: z
		.string("New password is required")
		.min(8, "Password must be at least 8 characters")
		.max(128, "Password must be at most 128 characters"),
});
```

> ✏️ `AuthValidation` object-এ `resetPasswordZodSchema` যোগ — এটা নোটে ছিল না।

### ধাপ ৪: `auth.service.ts`

```ts
const resetPassword = async (payload: IResetPasswordPayload) => {
	const { email, otp, newPassword } = payload;

	const user = await prisma.user.findUnique({
		where: { email },
	});

	if (!user || user.isDeleted || user.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	if (user.status === UserStatus.BLOCKED) {
		throw new AppError(httpStatus.FORBIDDEN, "User is blocked");
	}

	if (!user.emailVerified) {
		throw new AppError(httpStatus.BAD_REQUEST, "Email not verified");
	}

	// ভুল বা expired OTP হলে Better Auth APIError throw করে
	// revokeSessionsOnPasswordReset: true থাকায় password update আর সব session মোছা একই transaction-এ হয়
	await auth.api.resetPasswordEmailOTP({
		body: {
			email,
			otp,
			password: newPassword,
		},
	});

	// admin-এর দেওয়া password দিয়ে বানানো account হলে, এখন আর বদলানোর দরকার নেই
	if (user.needPasswordChange) {
		await prisma.user.update({
			where: { id: user.id },
			data: { needPasswordChange: false },
		});
	}
};
```

> ✏️ `IResetPasswordPayload` import আর `AuthService` object-এ `resetPassword` export — এটা নোটে ছিল না।

### ধাপ ৫: `auth.controller.ts`

```ts
const resetPassword = catchAsync(async (req: Request, res: Response) => {
	const payload: IResetPasswordPayload = req.body;

	await AuthService.resetPassword(payload);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Password reset successfully",
	});
});
```

> ✏️ `IResetPasswordPayload` import আর `AuthController` object-এ `resetPassword` যোগ — এটা নোটে ছিল না।

### ধাপ ৬: `auth.route.ts`

```ts
router.post(
	"/reset-password",
	validateRequest(AuthValidation.resetPasswordZodSchema),
	AuthController.resetPassword,
);
```

---

## Change password বনাম Reset password

| | Change password ([নোট ১৮](./18-change-password.md)) | Reset password |
|---|---|---|
| কখন | login করা অবস্থায় | password ভুলে গেলে |
| `checkAuth` | আছে | নেই |
| প্রমাণ | `currentPassword` | email-এ পাওয়া OTP |
| Session | পুরনোগুলো বাতিল, নতুন session + cookie | সব session বাতিল, cookie দেয় না → আবার login |

---

## Test (Postman)

**ধাপ ১: Forget password (OTP পাঠানো)**

| | |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5000/api/v1/auth/forget-password` |
| Body | Body tab → `raw` → ডানের dropdown থেকে `JSON` |

```json
{
	"email": "tomar-verified-email@gmail.com"
}
```

→ Inbox-এ "Password Reset OTP" mail আসবে (২ মিনিট valid)।

**ধাপ ২: Reset password**

| | |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5000/api/v1/auth/reset-password` |
| Body | raw JSON |

```json
{
	"email": "tomar-verified-email@gmail.com",
	"otp": "123456",
	"newPassword": "newPass1234"
}
```

চেক করো:
1. Prisma Studio-তে ওই user-এর `session` row সব মুছে গেছে।
2. পুরনো password দিয়ে login → fail, নতুন password দিয়ে → success।
3. আগের cookie দিয়ে `/auth/me` → `401`।

| আরও case | ফল |
|---|---|
| Forget-এ ভুল / blocked / unverified email | `200` একই message, কিন্তু mail আসে না |
| Reset-এ ভুল email | `404 User not found` |
| Reset-এ verify না করা email | `400 Email not verified` |
| ভুল / expired OTP | Better Auth `400` |
| `newPassword` ৮ অক্ষরের কম | `400` (Zod) |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ (Done)

1. ✅ **Forget password-এ user enumeration বন্ধ** — সবসময় একই `200` message (Part 1 ধাপ ৬ ও ৭)। `resetPassword`-এ check গুলো রাখা হয়েছে, কারণ সেখানে OTP ছাড়া কিছু করা যায় না।

### 🟡 ভালো practice

2. **`verifyEmailZodSchema`-তেও একই email নিয়ম।** এখানে `trim().toLowerCase()` আছে, কিন্তু [নোট ২০](./20-nodemailer-verify-email.md)-এর `verifyEmailZodSchema`-তে নেই। Better Auth email lowercase করে রাখে, তাই `"Arif@Gmail.com"` দিয়ে verify fail হতে পারে।

	```ts
	email: z.string("Email is required").trim().toLowerCase().pipe(z.email("Valid email is required")),
	```

3. **একই user check তিন জায়গায়।** `loginUser` আর `resetPassword`-এ deleted/blocked check প্রায় একই। একটা helper বানালে কোথাও ভুলে বাদ পড়বে না:

	```ts
	import type { User } from "@/generated/prisma/client";

	const assertUserCanAuth = (user: User | null) => {
		if (!user || user.isDeleted || user.status === UserStatus.DELETED) {
			throw new AppError(httpStatus.NOT_FOUND, "User not found");
		}
		if (user.status === UserStatus.BLOCKED) {
			throw new AppError(httpStatus.FORBIDDEN, "User is blocked");
		}
		return user;
	};
	```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **Mail না গেলেও "sent successfully"।** `sendVerificationOTP`-এর `catch` error গিলে ফেলে, তাই SMTP fail হলেও forget-password `200` দেয়। `catch`-এর comment-টাও এখন শুধু signup-এর কথা বলে — forget-password-এর জন্যও এটা প্রযোজ্য, comment আপডেট করে রাখতে পারো।
5. **Rate limit।** `/forget-password` বারবার hit করে কারো inbox-এ mail-এর বন্যা বা Gmail limit শেষ করা যায়। `auth.api` সরাসরি call করলে Better Auth-এর নিজের rate limit কাজ করে না। পরে `express-rate-limit` দিয়ে auth route-গুলোতে limit দাও।
6. **`needPasswordChange` update আলাদা query।** Password reset হয়ে যাওয়ার পর এই update fail করলে flag `true` থেকে যাবে — ক্ষতি কম (user শুধু আবার change করতে বলবে), তাই ঠিক আছে।

## 🔜 পরে যা যোগ হবে

- [ ] Frontend: forget password page → OTP + নতুন password form
- [ ] Auth route-এ rate limit
- [ ] Resend OTP ([নোট ২০](./20-nodemailer-verify-email.md)-এর recommendation)
