# 39 - Rate Limiting (Login Brute Force থেকে সুরক্ষা)

```
POST /api/v1/auth/login → validateRequest (email trim + lowercase) → loginRateLimiter
→ (৫ বার ভুল হয়ে গেছে?) হ্যাঁ → 429 Too Many Requests
                          না  → AuthController.loginUser → Better Auth signInEmail
```

> ⚠️ এই code এখনও পুরোপুরি test করা হয়নি। নিচের "Test" অংশ চালিয়ে নিশ্চিত হয়ে নাও।

---

## Rate limiting কী, আর কেন দরকার?

**Rate limiting** মানে: নির্দিষ্ট সময়ে একটা route-এ কতবার request করা যাবে, তার একটা সীমা বেঁধে দেওয়া। সীমা পার হলে server `429 Too Many Requests` দেয়।

Rate limit না থাকলে কী হতে পারে:

| আক্রমণ | কী হয় |
| --- | --- |
| **Brute force** | Script দিয়ে একই email-এ লাখবার password আন্দাজ করা (`123456`, `password`, ...) |
| **Credential stuffing** | অন্য সাইট থেকে ফাঁস হওয়া email/password list দিয়ে একে একে চেষ্টা করা |
| **Server-এর CPU শেষ করা** | Password hashing (scrypt) ইচ্ছা করেই ধীর। হাজার হাজার login request এলে server আটকে যায় |

আমাদের `/api/v1/auth/login`-এ কোনো limit ছিল না। Better Auth-এর নিজের rate limit **শুধু তার নিজের handler** (`/api/auth/*`) দিয়ে আসা request-এ কাজ করে। আমরা `auth.api.signInEmail()` সরাসরি server থেকে call করি, তাই সেখানে কোনো সুরক্ষা ছিল না।

### IP দিয়ে না, email দিয়ে কেন গুনছি?

Frontend-এ login হয় **Next.js server action** দিয়ে, মানে request যায় Browser → Next server → আমাদের backend। ফলে backend-এর কাছে **সব user-এর IP একটাই, Next server-এর IP**।

| Key | ফলাফল |
| --- | --- |
| IP দিয়ে | কয়েকজন user ভুল password দিলেই **সবার** login বন্ধ হয়ে যাবে ❌ |
| Email দিয়ে | শুধু যে account-এ বারবার ভুল চেষ্টা হচ্ছে, সেটা কিছুক্ষণের জন্য আটকাবে ✅ |

---

## ধাপ ১: Package install

```bash
pnpm add express-rate-limit
```

👉 https://express-rate-limit.mintlify.app/

---

## ধাপ ২: Rate limiter middleware বানানো

`src/app/middleware/rateLimiter.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import { rateLimit } from "express-rate-limit";
import httpStatus from "http-status";
import AppError from "@/app/errorHelpers/AppError";

const FIFTEEN_MINUTES = 15 * 60 * 1000;

// email অনুযায়ী গোনা হয়: একটা account-এ ১৫ মিনিটে ৫ বার ভুল password দিলে আটকে যাবে
// validateRequest-এর পরে বসাতে হবে, তাহলে email আগেই trim + lowercase হয়ে থাকবে
export const loginRateLimiter = rateLimit({
	windowMs: FIFTEEN_MINUTES,
	limit: 5,
	skipSuccessfulRequests: true, // শুধু ব্যর্থ চেষ্টা (status >= 400) গোনা হয়
	standardHeaders: "draft-8",
	legacyHeaders: false,
	keyGenerator: (req: Request) => `login:${String(req.body?.email ?? "")}`,
	handler: (_req: Request, _res: Response, next: NextFunction) => {
		next(
			new AppError(
				httpStatus.TOO_MANY_REQUESTS,
				"Too many failed login attempts. Please try again after 15 minutes.",
			),
		);
	},
});
```

প্রতিটা option কী করে:

| Option | কাজ |
| --- | --- |
| `windowMs` | কত সময়ের মধ্যে গোনা হবে (millisecond-এ, এখানে ১৫ মিনিট) |
| `limit` | ওই সময়ে সর্বোচ্চ কতবার (৫) |
| `skipSuccessfulRequests` | সফল login গোনা হয় না, শুধু ভুল চেষ্টা গোনা হয় |
| `standardHeaders: "draft-8"` | Response-এ `RateLimit` header পাঠায় (কতবার বাকি আছে) |
| `legacyHeaders: false` | পুরনো `X-RateLimit-*` header বন্ধ |
| `keyGenerator` | কী দিয়ে গোনা হবে, এখানে email |
| `handler` | Limit পার হলে কী হবে। `AppError` দিয়ে `globalErrorHandler`-এ পাঠানো, যাতে response format বাকি error-এর মতোই থাকে |

---

## ধাপ ৩: Backend-এ login validation schema

আগে `/login` route-এ কোনো validation ছিল না। Limiter-এর আগে email normalize করার জন্য এটা দরকার।

`src/app/module/auth/auth.validation.ts`-এ যোগ করো:

```ts
const loginZodSchema = z.object({
	email: z.string("Email is required").trim().toLowerCase().pipe(z.email("Valid email is required")),
	password: z.string("Password is required").min(1, "Password is required").max(128, "Password is too long"),
});
```

Export object-এ যোগ করো:

```ts
export const AuthValidation = {
	loginZodSchema,
	changePasswordZodSchema,
	verifyEmailZodSchema,
	forgetPasswordZodSchema,
	resetPasswordZodSchema,
};
```

- `trim().toLowerCase()` না থাকলে `Arif@gmail.com` আর `arif@gmail.com ` আলাদা key হয়ে যেত, আর attacker সহজেই limit এড়িয়ে যেত।
- `.max(128)`: কেউ বিশাল password পাঠালে hashing-এ CPU আটকে যেতে পারে।

---

## ধাপ ৪: Route-এ বসানো

`src/app/module/auth/auth.route.ts`

```ts
import { loginRateLimiter } from "@/app/middleware/rateLimiter";
```

```ts
router.post(
	"/login",
	validateRequest(AuthValidation.loginZodSchema),
	loginRateLimiter,
	AuthController.loginUser,
);
```

**ক্রম গুরুত্বপূর্ণ:** আগে `validateRequest`, তারপর `loginRateLimiter`।
- ভুল format-এর request limiter পর্যন্ত পৌঁছায় না।
- Limiter সবসময় normalized email পায়।

---

## ধাপ ৫: Better Auth-এর সরাসরি route বন্ধ করা

`app.ts`-এ `app.all("/api/auth/{*any}", toNodeHandler(auth))` mount করা আছে (Google login-এর জন্য লাগে)। তাই `POST /api/auth/sign-in/email` দিয়ে সরাসরি login করে আমাদের rate limit **পুরো এড়িয়ে যাওয়া যেত**।

`src/app/lib/auth.ts`-এ `betterAuth({ ... })`-এর ভেতরে যোগ করো:

```ts
	// এই দুই HTTP route বন্ধ। আমাদের নিজের /api/v1/auth/login আর /register ব্যবহার হবে।
	// auth.api.signInEmail() / signUpEmail() server থেকে call করলে আগের মতোই চলবে।
	disabledPaths: ["/sign-in/email", "/sign-up/email"],
```

`/sign-up/email` বন্ধ করার কারণ: `additionalFields`-এ `input: false` নেই। তাই কেউ সরাসরি `{ "role": "ADMIN" }` পাঠিয়ে নিজেকে ADMIN বানাতে পারত। (`role: { input: false }` দিলে আবার `user.service.ts`-এর doctor/admin তৈরি ভেঙে যায়, কারণ ওখানে `role` পাঠানো হয়।)

---

## Test (Postman)

**Login (ভুল password):**

```
POST http://localhost:5000/api/v1/auth/login
Content-Type: application/json

{
  "email": "patient@gmail.com",
  "password": "wrong-password"
}
```

| চেষ্টা | Response |
| --- | --- |
| ১ থেকে ৫ | `401` Invalid email or password |
| ৬ নম্বর | `429` Too many failed login attempts... |
| সঠিক password (limit-এর আগে) | `200` (এটা গোনা হয় না) |

আরও যা দেখবে:
- Response header-এ `RateLimit` আর `RateLimit-Policy` আছে কিনা।
- অন্য email দিয়ে login করলে আটকায় না (প্রতিটা email-এর আলাদা গণনা)।
- `POST /api/auth/sign-in/email` আর `POST /api/auth/sign-up/email` → `404`।
- Admin দিয়ে doctor/admin তৈরি আর patient register আগের মতোই কাজ করে কিনা (`disabledPaths` যেন `auth.api.*`-কে প্রভাবিত না করে)।
- Google login আগের মতো কাজ করে কিনা।
- শেষে `pnpm check`।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **User enumeration বন্ধ করো।** `loginUser` এখন password check করার **আগেই** `"User is blocked"` / `"User is deleted"` (404) বলে দেয়। তাই শুধু email দিয়েই জানা যায় কোন account আছে আর তার অবস্থা কী। সমাধান: check-টা `auth.ts`-এর `databaseHooks.session.create.before`-এ নিয়ে যাও। Better Auth password verify করার পরেই session বানায়, তাই সঠিক password ছাড়া এই message কেউ দেখবে না।

	```ts
	databaseHooks: {
		session: {
			create: {
				before: async (session) => {
					const user = await prisma.user.findUnique({
						where: { id: session.userId },
						select: { status: true, isDeleted: true },
					});

					if (!user || user.isDeleted || user.status === UserStatus.DELETED) {
						throw new APIError("UNAUTHORIZED", { message: "Invalid email or password" });
					}

					if (user.status === UserStatus.BLOCKED) {
						throw new APIError("FORBIDDEN", { message: "Your account has been blocked. Please contact support." });
					}
				},
			},
		},
	},
	```

	তারপর `loginUser` থেকে `findUnique` আর status check সরিয়ে দাও (`import { APIError } from "better-auth"`)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

1. **Production-এ Redis store।** Default store memory-তে থাকে। Server restart দিলে গণনা মুছে যায়, আর একাধিক server চালালে প্রত্যেকটার গণনা আলাদা। Deploy-এর সময় `rate-limit-redis` ব্যবহার করো।
2. **Lockout-এর সীমাবদ্ধতা।** কেউ ইচ্ছা করে অন্যের email দিয়ে ৫ বার ভুল দিলে সেই user ১৫ মিনিট login করতে পারবে না। Window ছোট রাখায় ক্ষতি সীমিত।
3. **IP-based limit।** পরে Next server থেকে আসল client IP (`x-forwarded-for`) পাঠিয়ে backend-এ `app.set("trust proxy", 1)` দিয়ে IP + email দুটো দিয়েই limit করা যায়। তবে backend সরাসরি public থাকলে header জাল করা যায়, তাই সাবধানে করতে হবে।

---

## 🔜 পরে যা যোগ হবে

- [ ] `forget-password`, `verify-email` (OTP পাঠানো) route-এও limiter (email spam ঠেকাতে)
- [ ] `register` route-এ limiter
- [ ] `40-packages-and-scripts.md`-এর Dependencies table-এ `express-rate-limit` যোগ করা
