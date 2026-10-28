# 19 - Logout (+ Logout from all devices)

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

## Logout from all devices

```
POST /auth/logout-all → checkAuth() → cookie থেকে session token → "Bearer <token>"
→ Better Auth revokeSessions (এই user-এর সব session DB থেকে মুছে) → এই browser-এর ৩টা cookie clear
```

### ধাপ ১: `auth.service.ts`

```ts
const logoutFromAllDevices = async (sessionToken: string) => {
	if (!sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session token is missing");
	}

	const headers = new Headers({
		Authorization: `Bearer ${sessionToken}`,
	});

	// এই user-এর সব session DB থেকে মুছে দেয়, return করে { status: true }
	const result = await auth.api.revokeSessions({ headers });

	return result;
};
```

> ✏️ `AuthService` object-এ `logoutFromAllDevices` export করতে হবে — এটা নোটে ছিল না।

### ধাপ ২: `auth.controller.ts`

```ts
const logoutFromAllDevices = catchAsync(async (req: Request, res: Response) => {
	const sessionToken = CookieUtils.getCookie(req, "better-auth.session_token");

	if (!sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session token is missing");
	}

	const result = await AuthService.logoutFromAllDevices(sessionToken);

	// শুধু এই browser-এর cookie মোছা হয়; অন্য device-গুলোকে checkAuth নিজেই আটকে দেবে
	tokenUtils.clearAuthCookies(res);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Logged out from all devices successfully",
		data: result,
	});
});
```

> ✏️ `AuthController` object-এ `logoutFromAllDevices` যোগ করতে হবে — এটা নোটে ছিল না।

### ধাপ ৩: `auth.route.ts`

```ts
router.post("/logout-all", checkAuth(), AuthController.logoutFromAllDevices);
```

- `checkAuth()`-এ কোনো role দেওয়া হয়নি। `authRoles.length === 0` হলে role check skip হয়, তাই logged in যেকোনো user এটা ব্যবহার করতে পারবে।

### অন্য device-গুলো কীভাবে বের হয়ে যায়?

ওদের browser-এ cookie এখনো থাকে, কিন্তু DB-তে session নেই:

| অন্য device যা করে | ফল |
|---|---|
| কোনো protected route hit | `checkAuth` session খুঁজে পায় না → `401` |
| `refreshToken` দিয়ে নতুন token চায় | `getNewToken`-ও DB-তে session খোঁজে → `401` |

### `logout` বনাম `logout-all`

| | `/logout` | `/logout-all` |
|---|---|---|
| Better Auth API | `signOut` | `revokeSessions` |
| DB থেকে মোছে | শুধু এই session | এই user-এর সব session |
| Response `data` | `{ success: true }` | `{ status: true }` |
| Cookie clear | এই browser | এই browser (বাকিগুলো `checkAuth`-এ আটকায়) |

### Test (Postman)

| | |
|---|---|
| Method | `POST` |
| URL | `http://localhost:5000/api/v1/auth/logout-all` |
| Body | নেই |
| Cookie | login-এর ৩টা cookie |

চেক করো:
1. একই user দিয়ে ২ বার login করো (যেমন Postman আর browser থেকে)। `session` table-এ ২টা row থাকবে।
2. একটা থেকে `/logout-all` hit করো → ওই user-এর সব row মুছে যাবে।
3. অন্যটা থেকে `/auth/me` hit করো → `401`।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Access token expire হলে logout আটকে যায়।** (এটা শুধু `/logout`-এর জন্য। `/logout-all`-এ `checkAuth` রাখাই ঠিক — সব session মুছে ফেলার মতো বড় কাজের আগে user যাচাই হওয়া দরকার। token expire হলে আগে refresh করে নেবে।) `checkAuth` আগে JWT verify করে। `accessToken` (1d) expire কিন্তু session (7d) বেঁচে আছে → logout `401` দেয়, cookie clear হয় না, DB-তে session থেকে যায়। আর cookie `httpOnly` হওয়ায় frontend নিজে মুছতেও পারে না। সমাধান: logout route থেকে `checkAuth` সরানো, আর token না থাকলেও cookie clear করা।

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

2. **একই check দুইবার।** (`logoutFromAllDevices`-এও একই কথা।) `!sessionToken` check controller আর service দুই জায়গাতেই আছে। ক্ষতি নেই, কিন্তু একটা রাখলেই যথেষ্ট (controller-এ রাখা ভালো, কারণ cookie পড়া controller-এর কাজ)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. ✅ (Done) **`sameSite: "none"` + dev-এ `secure: false`.** `token.ts`-এ এখন `sameSite: isProduction ? "none" : "lax"` — browser (Google login, Next.js) থেকেও cookie ঠিকমতো set/clear হবে। ([নোট ২২](./22-google-oauth.md)-এ Better Auth-এর cookie-তেও একই নিয়ম।)

4. **Logout-এর পর access token।** JWT নিজে stateless, তাই logout-এর পরও expire হওয়া পর্যন্ত valid। তবে আমাদের `checkAuth` DB-তে session খোঁজে, session মুছে গেলে শুধু access token দিয়ে ঢোকা যাবে না — তাই এটা নিরাপদ।

## 🔜 পরে যা যোগ হবে

- [ ] Frontend (Next.js) থেকে logout button → `credentials: "include"` দিয়ে call
- [x] সব device থেকে logout (`auth.api.revokeSessions`) ✅ (Done)
