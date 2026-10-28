# 17 - Refresh Token দিয়ে নতুন Access Token (+ Session-এর মেয়াদ বাড়ানো)

> Module: 39-03

```
POST /auth/refresh-token → cookie (refreshToken + session_token) → refresh token verify → DB session check
→ user check → নতুন access + refresh token → session expiresAt বাড়ানো → ৩টা cookie আবার set
```

**কেন দরকার?**
Access token দিয়ে যেকোনো protected route-এ ঢোকা যায়, কিন্তু এর একটা মেয়াদ (expiration time) আছে। মেয়াদ শেষ হলে user-কে আবার login না করিয়ে **refresh token** দিয়ে নতুন access token বানিয়ে দিই।

| Token | মেয়াদ (আমাদের `.env`) | কাজ |
|---|---|---|
| `accessToken` | `1d` | প্রতিটা protected request-এ (checkAuth) |
| `refreshToken` | `7d` | শুধু `/refresh-token`-এ, নতুন access token আনতে |
| `better-auth.session_token` | `86400` sec (১ দিন) | DB-তে session আছে কিনা (logout / block হলে বাতিল) |

---

## ধাপ ১: `auth.service.ts` — `getNewToken`

> ✏️ import নোটে ছিল না (আগের import-গুলোর সাথে):
> ```ts
> import { envVars } from "@/app/config/env";
> import { jwtUtils } from "@/app/utils/jwt";
> import type { JwtPayload } from "jsonwebtoken";
> ```

```ts
const getNewToken = async (refreshToken: string, sessionToken: string) => {
	// ধাপ ১: refresh token verify (ভুল/expired হলে jwtUtils নিজেই AppError throw করে)
	const decoded = jwtUtils.verifyToken(refreshToken, envVars.REFRESH_TOKEN_SECRET);

	// ধাপ ২: DB-তে session খোঁজা
	const session = await prisma.session.findUnique({
		where: { token: sessionToken },
		include: { user: true },
	});

	if (!session || session.expiresAt <= new Date()) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session expired or invalid, please login again");
	}

	// ধাপ ৩: refresh token আর session একই user-এর কিনা
	if (decoded.userId !== session.userId) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Invalid refresh token");
	}

	// ধাপ ৪: user-এর বর্তমান অবস্থা DB থেকে check
	const { user } = session;

	if (user.isDeleted || user.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.UNAUTHORIZED, "User has been deleted");
	}

	if (user.status === UserStatus.BLOCKED) {
		throw new AppError(httpStatus.FORBIDDEN, "User is blocked");
	}

	// ধাপ ৫: fresh data দিয়ে নতুন token
	const jwtPayload: JwtPayload = {
		userId: user.id,
		role: user.role,
		name: user.name,
		email: user.email,
		status: user.status,
		isDeleted: user.isDeleted,
		emailVerified: user.emailVerified,
	};

	const newAccessToken = tokenUtils.getAccessToken(jwtPayload);
	const newRefreshToken = tokenUtils.getRefreshToken(jwtPayload);

	// ধাপ ৬: session-এর মেয়াদ বাড়ানো (.env-এ seconds → ms)
	const sessionExpiresInMs = Number(envVars.BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN) * 1000;

	const { token } = await prisma.session.update({
		where: { token: sessionToken },
		data: {
			expiresAt: new Date(Date.now() + sessionExpiresInMs),
		},
	});

	return {
		accessToken: newAccessToken,
		refreshToken: newRefreshToken,
		sessionToken: token,
	};
};
```

> ✏️ export object-এ যোগ: `export const AuthService = { registerPatient, loginUser, getMe, getNewToken };`

**৬টা ধাপ এক নজরে:**

| # | কী | fail হলে |
|---|---|---|
| ১ | Refresh token verify (`REFRESH_TOKEN_SECRET` দিয়ে) | 401 (invalid / expired) |
| ২ | DB-তে session আছে আর মেয়াদ আছে? | 401 |
| ৩ | Refresh token-এর `userId` = session-এর `userId`? | 401 |
| ৪ | User deleted / blocked? (DB থেকে) | 401 / 403 |
| ৫ | DB-র fresh data দিয়ে নতুন access + refresh token | — |
| ৬ | Session-এর `expiresAt` = এখন + ১ দিন | — |

- **ধাপ ৫-এ fresh data কেন?** পুরনো token-এর payload না নিয়ে DB থেকে নতুন করে নেওয়া হচ্ছে, তাই মাঝে role / নাম বদলালে নতুন token-এ সেটাই আসবে।
- **ধাপ ৬:** Session-এর `token` বদলায় না, শুধু মেয়াদ বাড়ে। `.env`-এর value second-এ, `Date`-এর হিসাব ms-এ, তাই `× 1000` (note 13-এর `maxAge` table)।
- এই route-এ **access token লাগে না** — কারণ ওটা expire হয়ে গেছে বলেই তো এখানে আসা।

## ধাপ ২: `auth.controller.ts` — `getNewToken`

> ✏️ import নোটে ছিল না: `import { CookieUtils } from "@/app/utils/cookie";`

```ts
const getNewToken = catchAsync(async (req: Request, res: Response) => {
	const refreshToken = CookieUtils.getCookie(req, "refreshToken");
	const sessionToken = CookieUtils.getCookie(req, "better-auth.session_token");

	if (!refreshToken || !sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Refresh token or session token is missing");
	}

	const result = await AuthService.getNewToken(refreshToken, sessionToken);

	const { accessToken, refreshToken: newRefreshToken, sessionToken: newSessionToken } = result;

	tokenUtils.setAccessTokenCookie(res, accessToken);
	tokenUtils.setRefreshTokenCookie(res, newRefreshToken);
	tokenUtils.setBetterAuthSessionCookie(res, newSessionToken);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "New tokens generated successfully",
		data: {
			accessToken,
			refreshToken: newRefreshToken,
			sessionToken: newSessionToken,
		},
	});
});
```

> ✏️ export object-এ যোগ: `export const AuthController = { registerPatient, loginUser, getMe, getNewToken };`

- Token body থেকে না, **cookie থেকে** পড়া হচ্ছে (httpOnly cookie, frontend JS পড়তে পারে না)।
- `refreshToken: newRefreshToken` → destructure করার সময় নাম বদল, কারণ উপরে `refreshToken` নামের variable (পুরনোটা) আগেই আছে।
- Session token একই থাকলেও আবার cookie set করা হচ্ছে, যাতে cookie-র `maxAge`-ও নতুন করে শুরু হয়।

## ধাপ ৩: `auth.route.ts`

```ts
router.post("/refresh-token", AuthController.getNewToken);
```

- এখানে `checkAuth` **নেই** — checkAuth access token চায়, আর access token expire হয়েছে বলেই এই route call হচ্ছে।

---

## Frontend-এর দিক থেকে পুরো flow

```
১. যেকোনো protected request → 401 (access token expired / cookie নেই)
২. frontend → POST /auth/refresh-token (cookie নিজে থেকেই যায়)
   ✅ 200 → নতুন cookie set → আগের request আবার পাঠাও
   ❌ 401 → session-ও শেষ → login page-এ পাঠাও
```

Note 13-এর `X-Session-Refresh: true` header পেলে frontend আগে থেকেই refresh call করে দিতে পারে, মেয়াদ শেষ হওয়ার অপেক্ষা না করে।

---

## ধাপ ৪: Test (Postman)

### ১. Login

| Method | URL |
|---|---|
| `POST` | `http://localhost:5000/api/v1/auth/login` |

Body → raw → JSON:

```json
{
  "email": "patient@example.com",
  "password": "123456"
}
```

Send করার পর response-এর **Cookies** tab-এ ৩টা cookie দেখা উচিত: `accessToken`, `refreshToken`, `better-auth.session_token`। Value-গুলো কোথাও (notepad-এ) copy করে রাখো, পরে মিলিয়ে দেখবে।

### ২. Refresh token

| Method | URL | Body | Headers |
|---|---|---|---|
| `POST` | `http://localhost:5000/api/v1/auth/refresh-token` | লাগবে না | কিছু লাগবে না, Postman cookie নিজেই পাঠাবে |

### ৩. মিলিয়ে দেখো

| Cookie | আগের সাথে তুলনা |
|---|---|
| `accessToken` | ✅ বদলেছে |
| `refreshToken` | ✅ বদলেছে |
| `better-auth.session_token` | একই আছে, কিন্তু Prisma Studio-তে `session` table-এ `expiresAt` বেড়েছে |

### ৪. Error case

| Case | আশা |
|---|---|
| Postman-এর Cookies থেকে `refreshToken` মুছে refresh | 401 (missing) |
| `refreshToken` cookie-র value একটু বদলে দিয়ে refresh | 401 (Invalid token) |
| Prisma Studio থেকে session row মুছে refresh | 401 (Session expired or invalid) |
| User-কে BLOCKED করে refresh | 403 |

---

## 🔜 পরে যা যোগ হবে

- [ ] Logout → `clearAuthCookies` + DB থেকে session মুছে ফেলা
- [ ] Change password → নতুন token

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Access token আর session-এর মেয়াদ একই (১ দিন), তাই refresh প্রায়ই কাজ করবে না** — Login-এর সময় access token-এর মেয়াদ ১ দিন, session-এরও ১ দিন, দুটো একই মুহূর্তে শুরু। তাই যখন access token expire হয়, ঠিক তখন session-ও expire → `getNewToken` ধাপ ২-এ 401 দেয়, user-কে আবার login করতেই হয়। Refresh শুধু তখনই কাজ করে যখন frontend আগে থেকে (যেমন `X-Session-Refresh` header দেখে) call করে। সাধারণ নিয়ম হলো access token ছোট রাখা:
   ```dotenv
   ACCESS_TOKEN_EXPIRES_IN=15m
   REFRESH_TOKEN_EXPIRES_IN=7d
   BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN=604800   # 7 দিন
   ```
   (তখন `token.ts`-এর `ACCESS_TOKEN_MAX_AGE`-ও `15 * 60 * 1000` করতে হবে — note 13-এর recommendation ৪ দেখো।)

2. **JWT payload দুই জায়গায় হাতে লেখা** — note 13-এর `generateAuthTokens`-এ আর এখানে একই ৭টা field। একটায় field যোগ করলে আরেকটায় ভুলে গেলে token-এর shape আলাদা হবে। Prisma-র `User` নিয়ে একটা function বানাও:
   ```ts
   import type { User } from "@/generated/prisma/client";

   const buildJwtPayload = (user: Pick<User, "id" | "role" | "name" | "email" | "status" | "isDeleted" | "emailVerified">) => ({
   	userId: user.id,
   	role: user.role,
   	name: user.name,
   	email: user.email,
   	status: user.status,
   	isDeleted: user.isDeleted,
   	emailVerified: user.emailVerified,
   });
   ```
   তারপর login / register / refresh তিন জায়গাতেই `buildJwtPayload(user)`।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`X-Session-Refresh` হিসাব একটু পিছিয়ে যাবে** — checkAuth-এ session-এর পুরো মেয়াদ ধরা হয় `expiresAt - createdAt`। Refresh-এর পর `expiresAt` বাড়ে কিন্তু `createdAt` একই থাকে, তাই হিসাবে মেয়াদ লম্বা দেখায় আর ২০%-এর header দেরিতে আসে। `session.updatedAt` (`@updatedAt` থাকলে refresh-এ নিজে বদলায়) ব্যবহার করলে ঠিক হয়:
   ```ts
   const sessionLifeTime = session.expiresAt.getTime() - session.updatedAt.getTime();
   ```

4. **পুরনো refresh token-ও valid থাকে** — নতুন refresh token দেওয়ার পরও পুরনোটা নিজের মেয়াদ পর্যন্ত কাজ করবে (JWT বাতিল করা যায় না)। তবে প্রতিবার session check হয়, তাই logout / block হলে পুরনো-নতুন দুটোই অকেজো। এখনকার জন্য যথেষ্ট।

5. **Token response body-তেও** — note 13-এর recommendation ৩-এর মতো: web frontend হলে body-তে token না পাঠালেও চলে, cookie-ই যথেষ্ট।
