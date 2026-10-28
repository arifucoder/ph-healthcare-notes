# 19 - Logout

```
POST /auth/logout → checkAuth → cookie থেকে session token → "Bearer <token>"
→ Better Auth signOut (DB থেকে session মুছে) → ৩টা cookie clear → response
```

---

## ধাপ ১: `auth.service.ts`

```ts
const logoutUser = async (sessionToken: string) => {
	if (!sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session token is missing");
	}

	const headers = new Headers({
		Authorization: `Bearer ${sessionToken}`,
	});

	// DB থেকে session মুছে দেয়, return করে { success: true }
	const result = await auth.api.signOut({ headers });

	return result;
};
```

- `Authorization: Bearer ...` header কাজ করে কারণ `lib/auth.ts`-এ `plugins: [bearer()]` দেওয়া আছে (change-password-এও একই pattern)।
- `signOut` DB-র `session` table থেকে ওই session row মুছে দেয়।

> ✏️ `AuthService` object-এ `logoutUser` export করতে হবে — এটা নোটে ছিল না।

```ts
export const AuthService = {
	// ...আগেরগুলো
	logoutUser,
};
```

---

## ধাপ ২: `auth.controller.ts`

```ts
const logoutUser = catchAsync(async (req: Request, res: Response) => {
	const sessionToken = CookieUtils.getCookie(req, "better-auth.session_token");

	if (!sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session token is missing");
	}

	const result = await AuthService.logoutUser(sessionToken);

	// set করার সময় যে options দিয়েছিলাম, ঠিক সেই options দিয়েই ৩টা cookie মুছে ফেলা
	tokenUtils.clearAuthCookies(res);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "User logged out successfully",
		data: result,
	});
});
```

> ✏️ `AuthController` object-এ `logoutUser` যোগ করতে হবে — এটা নোটে ছিল না।

### `tokenUtils.clearAuthCookies` (`src/app/utils/token.ts` — আগেই বানানো)

```ts
// set আর clear দুই জায়গাতেই একই options লাগে
const baseCookieOptions: CookieOptions = {
	httpOnly: true,
	secure: isProduction,
	sameSite: "none",
	path: "/",
};

const clearAuthCookies = (res: Response) => {
	CookieUtils.clearCookie(res, "accessToken", baseCookieOptions);
	CookieUtils.clearCookie(res, "refreshToken", baseCookieOptions);
	CookieUtils.clearCookie(res, "better-auth.session_token", baseCookieOptions);
};
```

| কেন একই options? | |
|---|---|
| `path` / `domain` আলাদা হলে | browser অন্য cookie ভাবে → আসল cookie মুছে না |
| `sameSite` / `secure` আলাদা হলে | cross-site এ clear header-টা browser ignore করতে পারে |

> `maxAge` clear করার সময় দেওয়ার দরকার নেই — `res.clearCookie` নিজেই expire করে দেয়।

---

## ধাপ ৩: `auth.route.ts`

```ts
router.post("/logout", checkAuth(Role.ADMIN, Role.DOCTOR, Role.PATIENT, Role.SUPER_ADMIN), AuthController.logoutUser);
```

- সব role-ই logout করতে পারবে, কিন্তু logged in থাকতে হবে (`checkAuth`)।
- Body লাগে না, তাই `validateRequest` নেই।

---

## Test (Postman)

| | |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5000/api/v1/auth/logout` |
| Body | নেই |
| Cookie | login করার পর পাওয়া `accessToken`, `refreshToken`, `better-auth.session_token` |

Response:

```json
{
	"success": true,
	"message": "User logged out successfully",
	"data": { "success": true }
}
```

চেক করো:
1. Postman-এর Cookies tab-এ ৩টা cookie আর নেই।
2. Prisma Studio-তে (`pnpm studio`) `session` table থেকে ওই row মুছে গেছে।
3. আবার `/auth/me` hit করলে → `401`।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Access token expire হলে logout আটকে যায়।** `checkAuth` আগে JWT verify করে। `accessToken` (1d) expire কিন্তু session (7d) বেঁচে আছে → logout `401` দেয়, cookie clear হয় না, DB-তে session থেকে যায়। আর cookie `httpOnly` হওয়ায় frontend নিজে মুছতেও পারে না। সমাধান: logout route থেকে `checkAuth` সরানো, আর token না থাকলেও cookie clear করা।

	```ts
	// auth.route.ts
	router.post("/logout", AuthController.logoutUser);

	// auth.controller.ts
	const logoutUser = catchAsync(async (req: Request, res: Response) => {
		const sessionToken = CookieUtils.getCookie(req, "better-auth.session_token");

		if (sessionToken) {
			await AuthService.logoutUser(sessionToken);
		}

		// token থাকুক বা না থাকুক, cookie সবসময় মুছবে
		tokenUtils.clearAuthCookies(res);

		sendResponse(res, {
			httpStatusCode: httpStatus.OK,
			success: true,
			message: "User logged out successfully",
			data: null,
		});
	});
	```

### 🟡 ভালো practice

2. **একই check দুইবার।** `!sessionToken` check controller আর service দুই জায়গাতেই আছে। ক্ষতি নেই, কিন্তু একটা রাখলেই যথেষ্ট (controller-এ রাখা ভালো, কারণ cookie পড়া controller-এর কাজ)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`sameSite: "none"` + dev-এ `secure: false`.** Chrome সাধারণত `SameSite=None` cookie-তে `Secure` না থাকলে reject করে। Postman-এ সমস্যা হয় না, কিন্তু Next.js frontend (browser) থেকে test করলে cookie set/clear নাও হতে পারে (untested)। তখন comment করা লাইনটা ফিরিয়ে আনো:

	```ts
	sameSite: isProduction ? "none" : "lax",
	```

4. **Logout-এর পর access token।** JWT নিজে stateless, তাই logout-এর পরও expire হওয়া পর্যন্ত valid। তবে আমাদের `checkAuth` DB-তে session খোঁজে, session মুছে গেলে শুধু access token দিয়ে ঢোকা যাবে না — তাই এটা নিরাপদ।

## 🔜 পরে যা যোগ হবে

- [ ] Frontend (Next.js) থেকে logout button → `credentials: "include"` দিয়ে call
- [ ] সব device থেকে logout (`auth.api.revokeSessions`)
