# 18 - Change Password

> Module: 39-05

```
POST /auth/change-password → checkAuth → Zod → cookie থেকে session token → "Bearer <token>"
→ Better Auth changePassword (সব পুরনো session বাতিল, নতুন session) → নতুন ৩টা cookie
```

---

## ধাপ ১: `auth.validation.ts`

```ts
import * as z from "zod";

const changePasswordZodSchema = z
	.object({
		currentPassword: z.string("Current password is required").min(1, "Current password is required"),
		newPassword: z
			.string("New password is required")
			.min(8, "Password must be at least 8 characters")
			.max(128, "Password must be at most 128 characters"),
	})
	.refine((data) => data.currentPassword !== data.newPassword, {
		message: "New password must be different from current password",
		path: ["newPassword"],
	});

export const AuthValidation = {
	changePasswordZodSchema,
};
```

- `.refine` → নতুন আর পুরনো password একই হলে error, `path: ["newPassword"]` দিলে error-টা `newPassword` field-এ দেখায়।

> ✏️ নোটে ছিল না — service-এর `IChangePasswordPayload` type, schema থেকেই বানাও:
> ```ts
> // auth.interface.ts
> import type * as z from "zod";
> import type { AuthValidation } from "./auth.validation";
>
> export type IChangePasswordPayload = z.infer<typeof AuthValidation.changePasswordZodSchema>;
> ```

## ধাপ ২: `auth.ts`-এ `bearer()` plugin

```ts
import { bearer } from "better-auth/plugins";

export const auth = betterAuth({
	// ...আগের সব কিছু
	plugins: [bearer()],
});
```

এটা ছাড়া ধাপ ৩-এর `getSession` সবসময় `null` দেবে। কেন, নিচের "Bearer কী" অংশে।

## ধাপ ৩: `auth.service.ts` — `changePassword`

```ts
const changePassword = async (payload: IChangePasswordPayload, sessionToken: string) => {
	const headers = new Headers({
		Authorization: `Bearer ${sessionToken}`,
	});

	const session = await auth.api.getSession({ headers });

	if (!session) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Invalid session token");
	}

	const { currentPassword, newPassword } = payload;

	// ভুল currentPassword হলে Better Auth APIError throw করবে
	// revokeOtherSessions: true → সব পুরনো session মুছে নতুন session, result.token-এ নতুন token
	const result = await auth.api.changePassword({
		body: {
			currentPassword,
			newPassword,
			revokeOtherSessions: true,
		},
		headers,
	});

	if (session.user.needPasswordChange) {
		await prisma.user.update({
			where: { id: session.user.id },
			data: { needPasswordChange: false },
		});
	}

	const { accessToken, refreshToken } = generateAuthTokens(session.user);

	return {
		...result,
		accessToken,
		refreshToken,
	};
};
```

> ✏️ নোটে ছিল না: `import type { IChangePasswordPayload } from "./auth.interface";` আর `AuthService` export object-এ `changePassword`।

- `revokeOtherSessions: true` → password বদলালে অন্য সব device থেকে logout (কেউ password চুরি করে থাকলে সে-ও বের হয়ে যায়)।
- `needPasswordChange` → admin/doctor-কে বানানোর সময় `true` ছিল (note 10, 14), প্রথমবার password বদলালে `false`।

## ধাপ ৪: `auth.controller.ts`

```ts
const changePassword = catchAsync(async (req: Request, res: Response) => {
	const sessionToken = CookieUtils.getCookie(req, "better-auth.session_token");

	if (!sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session token is missing");
	}

	const result = await AuthService.changePassword(req.body, sessionToken);

	const { accessToken, refreshToken, token } = result;

	tokenUtils.setAccessTokenCookie(res, accessToken);
	tokenUtils.setRefreshTokenCookie(res, refreshToken);

	// revokeOtherSessions: true হলে নতুন session token আসে
	if (token) {
		tokenUtils.setBetterAuthSessionCookie(res, token);
	}

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Password changed successfully",
		data: result,
	});
});
```

> ✏️ `AuthController` export object-এ `changePassword` যোগ করো।

- পুরনো session মুছে গেছে, তাই **নতুন token cookie-তে না বসালে** user পরের request-এই 401 পাবে।

## ধাপ ৫: `auth.route.ts`

```ts
router.post(
	"/change-password",
	checkAuth(Role.ADMIN, Role.DOCTOR, Role.PATIENT, Role.SUPER_ADMIN),
	validateRequest(AuthValidation.changePasswordZodSchema),
	AuthController.changePassword,
);
```

> ✏️ import: `import { AuthValidation } from "./auth.validation";`

## ধাপ ৬: `globalErrorHandler.ts`-এ Better Auth-এর error

ভুল `currentPassword` দিলে Better Auth `APIError` throw করে। এটা না ধরলে 500 আসত (note 13-এর recommendation)।

```ts
import { APIError } from "better-auth/api";

// globalErrorHandler-এর ভেতরে, Error branch-এর আগে
else if (err instanceof APIError) {
	statusCode = err.statusCode;
	message = err.message;
}
```

---

## Bearer কী, আর কখন লাগে?

**Bearer আলাদা কোনো token না।** Login-এ পাওয়া session token-কে Better Auth-এর কাছে পাঠানোর একটা format মাত্র:

```
Authorization: Bearer abc123xyz
                      └─ login-এ পাওয়া session token, হুবহু সেটাই
```

**Plugin কেন লাগল?**
- Better Auth নিজে থেকে শুধু **নিজের signed cookie** (`token.signature`) চেনে।
- আমরা cookie নিজেরাই set করি (`setBetterAuthSessionCookie`), তাই cookie-তে থাকে **raw token**, signature নেই।
- `bearer()` plugin Better Auth-কে বলে: "`Authorization` header-এ raw token এলে সেটাও মেনে নাও।"

| কাজ | Bearer লাগে? | কেন |
|---|---|---|
| `signUpEmail` / `signInEmail` | ❌ | নতুন session **বানায়**, token **দেয়** |
| `checkAuth` | ❌ | Better Auth না, সরাসরি `prisma.session` |
| `getNewToken` (refresh) | ❌ | নিজের JWT + `prisma` |
| `getSession` / `changePassword` / `signOut` | ✅ | **আগের session** কার, জানতে হয় |

> **মনে রাখো:** `prisma` দিয়ে নিজে করলে Bearer লাগে না। `auth.api.*` দিয়ে এমন কাজ, যেখানে আগের session লাগে → Bearer লাগে।

---

## ধাপ ৭: Test (Postman)

আগে `POST /api/v1/auth/login`, তারপর:

`POST http://localhost:5000/api/v1/auth/change-password`

```json
{
  "currentPassword": "123456",
  "newPassword": "newPass@123"
}
```

| Case | আশা |
|---|---|
| ঠিক password | 200, ৩টা cookie-ই নতুন |
| ভুল `currentPassword` | 400 / 401 (Better Auth-এর message) |
| নতুন = পুরনো | 400 (Zod) |
| পরে পুরনো password দিয়ে login | fail |
| পরে নতুন password দিয়ে login | ✅ |

---

## 🔜 পরে যা যোগ হবে

- [ ] Logout → `auth.api.signOut` (Bearer দিয়ে) + `clearAuthCookies`

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`APIError` branch-এ `errorSources` বাদ** — `message` বদলাচ্ছে, কিন্তু `errorSources` থেকে যাচ্ছে default `"Internal Server Error"`। Response-এ দুটো আলাদা কথা বলবে:
   ```ts
   else if (err instanceof APIError) {
   	statusCode = err.statusCode;
   	message = err.message;
   	errorSources = [{ path: "", message: err.message }];
   	stack = err.stack;
   }
   ```

2. **`checkAuth` এখানে ৪টা role না লিখে `checkAuth()`** — note 16-এর মতো, যেকোনো logged-in user।

### 🔵 জেনে রাখো

3. **Password-এর সর্বনিম্ন দৈর্ঘ্য মিলছে না (সম্ভবত)** — এখানে `min(8)`, কিন্তু create-doctor/admin-এ `min(6)`। Better Auth-এর default `minPasswordLength` ৮, তাই `auth.ts`-এ না বদলালে ৬-৭ অক্ষরের password দিয়ে doctor/admin বানাতে গেলে Better Auth reject করবে। সব জায়গায় একই রাখো (৮)।
