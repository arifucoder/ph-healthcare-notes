# 13 - JWT Access/Refresh Token, Cookie ও checkAuth (Role-based Authorization)

> Module: 38-09, 38-10

```
Login/Register → Better Auth (session) + JWT (access + refresh) → ৩টা httpOnly cookie
Protected route → checkAuth(...roles) → cookie পড়া → JWT verify → DB session check → role check → controller
```

---

## Authentication বনাম Authorization

| | মানে | কে করবে |
|---|---|---|
| **Authentication** | তুমি কে? (login / register) | Better Auth |
| **Authorization** | তুমি কী কী করতে পারবে? (কোন route-এ ঢুকতে পারবে) | আমরা নিজেরা (custom) |

- `auth.service.ts`-এ `registerPatient`-এর return data-তে hover করলে দেখা যায় `redirect`, `token`, `url` ইত্যাদি আসছে। এগুলো Better Auth দিচ্ছে, কারণ registration Better Auth দিয়েই করেছি।
- DB-র `session` table দেখলে বোঝা যায়: login অবস্থায় session থাকবে, session না থাকলে user logged out।
- Session দিয়ে বুঝি user logged in কিনা। কিন্তু user **ADMIN নাকি PATIENT নাকি DOCTOR**, আর সেই role অনুযায়ী কোন route-এ ঢুকতে দেব, সেটার জন্য আরেকটা token লাগবে। একে বলে **RBAC (Role Based Access Control)**।

এর জন্য ২টা token:

| Token | কাজ |
|---|---|
| **Access token** | API access করার জন্য (প্রতিটা protected request-এ লাগে) |
| **Refresh token** | Access token expire হলে আবার login না করেই নতুন access token বানানো |

---

## ধাপ ১: `jsonwebtoken` install (38-09)

```shell
pnpm add jsonwebtoken
pnpm add -D @types/jsonwebtoken
```

Docs: https://www.npmjs.com/package/jsonwebtoken

## ধাপ ২: JWT-র ৩টা function বোঝা

### `sign` — token বানানো

টাকার নোটে যেমন বাংলাদেশ ব্যাংকের গভর্নরের sign থাকে, sign ছাড়া টাকা অচল — token-ও তেমন secret দিয়ে sign করা থাকে।

```js
var token = jwt.sign({ foo: 'bar' }, 'shhhhh');
```

- `{ foo: 'bar' }` → **payload**: ticket-টা কার, কতক্ষণ valid ইত্যাদি তথ্য।
- `'shhhhh'` → **secret**: unique sign, এটা গোপন রাখতে হবে।

### `verify` — token আসল কিনা check

```js
jwt.verify(token, 'wrong-secret', function(err, decoded) {
  // err
  // decoded undefined
});
```

যেখানে token receive করব, সেখানে একই secret দিয়ে verify করব। Secret ভুল হলে, token বদলানো হলে বা expire হলে error।

### `decode` — শুধু payload পড়া

```js
var decoded = jwt.decode(token);
```

Token-এর ভিতরের user-এর তথ্য বের করে দেয়, কিন্তু এটা দিয়ে **verify হয় না**।

> ✏️ সংশোধন: নোটে লেখা ছিল "token encrypted অবস্থায় থাকে"। আসলে JWT **encrypted না, শুধু encoded (base64url)**। যে কেউ token নিয়ে jwt.io-তে paste করলেই payload পড়তে পারবে। Sign শুধু নিশ্চিত করে যে token কেউ **বদলায়নি**। তাই payload-এ কখনো password বা গোপন তথ্য রাখবে না।

| Function | Secret লাগে? | Verify করে? | কখন |
|---|---|---|---|
| `sign` | ✅ | — | login / register-এ token বানাতে |
| `verify` | ✅ | ✅ | protected route-এ token check করতে |
| `decode` | ❌ | ❌ | শুধু payload দেখতে (বিশ্বাস করা যাবে না) |

## ধাপ ৩: `src/app/utils/jwt.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import httpStatus from "http-status";
import jwt, { type JwtPayload, type SignOptions } from "jsonwebtoken";

const createToken = (payload: JwtPayload, secret: string, { expiresIn }: SignOptions): string => {
	return jwt.sign(payload, secret, { algorithm: "HS256", expiresIn });
};

const verifyToken = (token: string, secret: string): JwtPayload => {
	try {
		const decoded = jwt.verify(token, secret, { algorithms: ["HS256"] });

		if (typeof decoded === "string") {
			throw new AppError(httpStatus.UNAUTHORIZED, "Invalid token payload");
		}

		return decoded;
	} catch (error) {
		if (error instanceof AppError) throw error;

		if (error instanceof jwt.TokenExpiredError) {
			throw new AppError(httpStatus.UNAUTHORIZED, "Token has expired");
		}

		throw new AppError(httpStatus.UNAUTHORIZED, "Invalid token");
	}
};

// decode কোনো verify করে না, শুধু payload পড়ে, তাই এখানে null return করাই ঠিক
const decodeToken = (token: string): JwtPayload | null => {
	const decoded = jwt.decode(token);
	return decoded && typeof decoded !== "string" ? decoded : null;
};

export const jwtUtils = {
	createToken,
	verifyToken,
	decodeToken,
};
```

- `algorithm: "HS256"` sign-এ আর `algorithms: ["HS256"]` verify-তে → শুধু এই algorithm-এর token মানা হবে (নিরাপদ)।
- `verifyToken` যেকোনো সমস্যায় `AppError(401)` throw করে, তাই global error handler সুন্দর response দেবে।

## ধাপ ৪: `src/app/utils/token.ts` — access/refresh token ও cookie

> ✏️ এই file `./cookie` থেকে `CookieUtils` import করে, যেটা ধাপ ৬-এ বানানো হয়েছে। তাই দুটো file না বানানো পর্যন্ত import error দেখাবে।

```ts
import type { CookieOptions, Response } from "express";
import type { JwtPayload, SignOptions } from "jsonwebtoken";
import { envVars } from "../config/env";
import { CookieUtils } from "./cookie";
import { jwtUtils } from "./jwt";

// এই হিসাবটা লাগে কারণ cookie-র maxAge "দিন" বোঝে না, শুধু millisecond (ms) বোঝে
const ONE_DAY = 1000 * 60 * 60 * 24; // ms

// ⚠️ এগুলো .env-এর EXPIRES_IN-এর সাথে মিলিয়ে রাখো
const ACCESS_TOKEN_MAX_AGE = ONE_DAY; // ACCESS_TOKEN_EXPIRES_IN=1d
const REFRESH_TOKEN_MAX_AGE = ONE_DAY * 7; // REFRESH_TOKEN_EXPIRES_IN=7d
// Better Auth-এর expiresIn second-এ, cookie-র maxAge ms-এ → × 1000
const SESSION_MAX_AGE = Number(envVars.BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN) * 1000;

const isProduction = envVars.NODE_ENV === "production";

// set আর clear দুই জায়গাতেই একই options লাগে
const baseCookieOptions: CookieOptions = {
    httpOnly: true, // JS (document.cookie) দিয়ে পড়া যাবে না
    secure: isProduction, // production-এ শুধু HTTPS-এ cookie যাবে
    // ⚠️ "none" দিলে secure: true লাগবেই, নইলে browser cookie ফেলে দেয় (নিচে বিস্তারিত)
    sameSite: isProduction ? "none" : "lax",
    path: "/",
};

// ---------- Token তৈরি ----------
const getAccessToken = (payload: JwtPayload): string => {
    return jwtUtils.createToken(payload, envVars.ACCESS_TOKEN_SECRET, {
        expiresIn: envVars.ACCESS_TOKEN_EXPIRES_IN as SignOptions["expiresIn"],
    });
};

const getRefreshToken = (payload: JwtPayload): string => {
    return jwtUtils.createToken(payload, envVars.REFRESH_TOKEN_SECRET, {
        expiresIn: envVars.REFRESH_TOKEN_EXPIRES_IN as SignOptions["expiresIn"],
    });
};

// ---------- Cookie set ----------
const setAccessTokenCookie = (res: Response, token: string) => {
    CookieUtils.setCookie(res, "accessToken", token, {
        ...baseCookieOptions,
        maxAge: ACCESS_TOKEN_MAX_AGE,
    });
};

const setRefreshTokenCookie = (res: Response, token: string) => {
    CookieUtils.setCookie(res, "refreshToken", token, {
        ...baseCookieOptions,
        maxAge: REFRESH_TOKEN_MAX_AGE,
    });
};

const setBetterAuthSessionCookie = (res: Response, token: string) => {
    CookieUtils.setCookie(res, "better-auth.session_token", token, {
        ...baseCookieOptions,
        maxAge: SESSION_MAX_AGE,
    });
};

// ---------- Logout-এর জন্য ----------
const clearAuthCookies = (res: Response) => {
    CookieUtils.clearCookie(res, "accessToken", baseCookieOptions);
    CookieUtils.clearCookie(res, "refreshToken", baseCookieOptions);
    CookieUtils.clearCookie(res, "better-auth.session_token", baseCookieOptions);
};

export const tokenUtils = {
    getAccessToken,
    getRefreshToken,
    setAccessTokenCookie,
    setRefreshTokenCookie,
    setBetterAuthSessionCookie,
    clearAuthCookies,
};
```

> ✏️ সংশোধন ১ — `sameSite`: নোটে ছিল সবসময় `sameSite: "none"`, আর `isProduction ? "none" : "lax"` লাইনটা comment করা ছিল। Comment করা লাইনটাই ঠিক। কেন, সেটা নিচের "🍪 `sameSite` আর `secure`" অংশে বিস্তারিত।
>
> ✏️ সংশোধন ২ — `SESSION_MAX_AGE`: নোটে ছিল `ONE_DAY * 7` আর comment-এ `=604800`, কিন্তু ধাপ ৭-এ `.env`-এ `BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN=86400` (১ দিন)। Cookie ৭ দিন আর session ১ দিন — মিল ছিল না। এখন সরাসরি env থেকে হিসাব হয়, তাই হাতে মিলাতে হবে না।

### 🍪 `sameSite` আর `secure` — কেন এটা এত জরুরি

> **এক লাইনে:** `sameSite: "none"` দিলে `secure: true` লাগবেই। নইলে browser cookie-টা চুপচাপ ফেলে দেয় — login "সফল" হয়, কিন্তু cookie save হয় না, আর পরের সব request-এ 401।

**১. দুটো option কী করে?**

| Option | কী বলে দেয় |
|---|---|
| `secure: true` | cookie শুধু **HTTPS** দিয়ে যাবে |
| `sameSite` | **অন্য site** থেকে request গেলে cookie যাবে কিনা |

**২. `sameSite`-এর ৩টা মান**

| মান | অন্য site থেকে request-এ cookie যায়? | কখন |
|---|---|---|
| `"strict"` | ❌ কখনো না | খুব কড়া, সাধারণত লাগে না |
| `"lax"` | ❌ (শুধু link-এ click করে আসলে ✅) | একই site-এ frontend + backend |
| `"none"` | ✅ সবসময় | frontend আর backend **আলাদা domain**-এ |

**৩. "Same site" মানে কী?**
Port আর subdomain দেখা হয় না, শুধু মূল domain দেখা হয়।

| Frontend | Backend | Same site? |
|---|---|---|
| `localhost:3000` | `localhost:5000` | ✅ হ্যাঁ (port আলাদা হলেও) |
| `app.mysite.com` | `api.mysite.com` | ✅ হ্যাঁ |
| `myapp.vercel.app` | `myapi.onrender.com` | ❌ না → `"none"` লাগবে |

**৪. Browser-এর নিয়ম:** `SameSite=None` আছে কিন্তু `Secure` নেই → **cookie reject** (save-ই হবে না)।

**৫. আগের code-এ কী হতো?**

```
Development: secure: false + sameSite: "none"
→ browser: "None আছে কিন্তু Secure নেই" → cookie ফেলে দিল ❌
→ login response 200 ঠিকই আসে, কিন্তু browser-এ কোনো cookie নেই
→ পরের protected request-এ checkAuth cookie পায় না → 401 😵
```

**৬. এখনকার code-এ:**

| Environment | `secure` | `sameSite` | কেন কাজ করে |
|---|---|---|---|
| Development | `false` | `"lax"` | `localhost:3000` → `localhost:5000` একই site, তাই `lax`-এই cookie যায়; `lax`-এর জন্য `secure` লাগে না |
| Production | `true` | `"none"` | frontend আর backend আলাদা domain, তাই `none` লাগে; HTTPS আছে, তাই `secure: true` দেওয়া যায় |

**৭. Postman-এ কেন ধরা পড়েনি?**
Postman browser না, `SameSite` / `Secure`-এর নিয়ম মানে না, সব cookie রেখে দেয়। তাই Postman-এ সব ঠিক দেখায়, কিন্তু frontend (browser) লাগালেই সমস্যা হয়।

**৮. Browser-এ কীভাবে check করবে?**
DevTools → **Application** → **Cookies** → backend-এর URL। Cookie না থাকলে **Network** tab-এ login request → **Response Headers**-এ `Set-Cookie`-এর পাশে ⚠️ চিহ্ন থাকবে, hover করলে কারণ দেখাবে।

> মনে রাখো: `sameSite: "none"` ⟹ `secure: true` — সবসময় জোড়ায় জোড়ায়।

**`maxAge`-এর একক মনে রাখো:**

| জায়গা | একক |
|---|---|
| Cookie `maxAge` (Express) | **millisecond** |
| JWT `expiresIn` | string (`"1d"`, `"7d"`) বা second |
| Better Auth `session.expiresIn` / `updateAge` | **second** |

**এই function-গুলো একবার লিখছি, ব্যবহার হবে অনেক জায়গায়:**

```
Register        → token বানাও + cookie set করো
Login           → token বানাও + cookie set করো
Refresh token   → নতুন access token বানাও + cookie set করো
Change password → নতুন token বানাও + cookie set করো
Logout          → সব cookie clear করো
```

### তিনটা cookie কি আসলেই লাগবে?

Teacher একটা **hybrid system** বানাচ্ছেন। প্রতিটা cookie-র আলাদা কাজ:

| Cookie | কোথায় থাকে | কী কাজে লাগে |
|---|---|---|
| `accessToken` (JWT) | শুধু token-এর ভেতরে, DB লাগে না | দ্রুত verify করা যায়, role দেখতে DB query লাগে না |
| `refreshToken` (JWT) | token-এর ভেতরে | Access token expire হলে আবার login না করে নতুন একটা আনা |
| `better-auth.session_token` | DB-র `session` table-এ | Session control: logout করলে DB থেকে মুছে দেওয়া, "সব device থেকে logout", user block হলে session বাতিল |

- **JWT-র দুর্বলতা:** একবার issue হলে expire না হওয়া পর্যন্ত valid, মাঝপথে বাতিল করা যায় না।
- **DB session-এর দুর্বলতা:** প্রতিটা request-এ DB query লাগে।
- দুটো একসাথে রাখলে দুই দিকের সুবিধা।

সত্যি বলতে এটা বাধ্যতামূলক না। অনেক real project শুধু একটা পদ্ধতি ব্যবহার করে: শুধু Better Auth session (সবচেয়ে সহজ), অথবা শুধু JWT (access + refresh)।

---

## ধাপ ৫: `cookie-parser` install (38-10)

Cookie setup-এর আগে অবশ্যই `cookie-parser` লাগবে। এটা ছাড়া `req.cookies` হবে `undefined`, তখন cookie পড়তে গেলে crash: `Cannot read properties of undefined`।

```shell
pnpm add cookie-parser
pnpm add -D @types/cookie-parser
```

> ✏️ সংশোধন: নোটে ছিল `pnpm add i -D @types/cookie-parser`। এখানে `i` থাকলে pnpm **`i` নামের একটা আলাদা npm package**-ও install করে ফেলবে। `package.json`-এ `"i"` চলে এসে থাকলে `pnpm remove i` দাও।

```ts
// app.ts
import cookieParser from "cookie-parser";

app.use(cookieParser()); // routes-এর আগে
```

## ধাপ ৬: `src/app/utils/cookie.ts`

```ts
import type { CookieOptions, Request, Response } from "express";

const setCookie = (res: Response, key: string, value: string, options: CookieOptions) => {
	res.cookie(key, value, options);
};

const getCookie = (req: Request, key: string): string | undefined => {
	return req.cookies?.[key];
};

const clearCookie = (res: Response, key: string, options: CookieOptions) => {
	res.clearCookie(key, options);
};

export const CookieUtils = {
	setCookie,
	getCookie,
	clearCookie,
};
```

- `clearCookie`-এ set করার সময়ের মতো একই `path`/`sameSite`/`secure` না দিলে browser cookie মুছবে না, তাই `baseCookieOptions` দুই জায়গাতেই ব্যবহার হচ্ছে।

## ধাপ ৭: Better Auth session-এর মেয়াদ

`.env`-এ:

```dotenv
BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN=86400   # 1 দিন (second)
BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE=43200   # 12 ঘণ্টা (second)
```

| Option | মানে | Better Auth default |
|---|---|---|
| `expiresIn` | session কতক্ষণ valid | ৭ দিন |
| `updateAge` | কতক্ষণ পর পর session-এর মেয়াদ নতুন করে বাড়ানো হবে | ১ দিন |

**❌ দুটো সমান হলে সমস্যা কেন?**
`expiresIn = 1d` আর `updateAge = 1d` হলে, session বাড়ানোর সময় আসবে ঠিক তখন, যখন session already expire হয়ে গেছে। মানে extend কখনো হবেই না। তাই `updateAge` সবসময় `expiresIn`-এর চেয়ে ছোট রাখতে হবে।

`src/app/lib/auth.ts`:

```ts
export const auth = betterAuth({
	// ...
    session: {
        expiresIn: Number(envVars.BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN),
        updateAge: Number(envVars.BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE),
        cookieCache: {
            enabled: true,
            maxAge: 60 * 5, // 5 minutes (seconds) = 300
        },
    },
});
```

- `.env`-এর সব value string, তাই `Number(...)`।

## ধাপ ৮: env variables যোগ

> ✏️ এই ধাপ নোটে ছিল না। Note 06-এর `env.ts`-এ token-এর variable-গুলো comment করা ছিল, আর `FRONTEND_URL` (CORS-এর জন্য, ধাপ ১৩) একদমই ছিল না। এগুলো না থাকলে `envVars.ACCESS_TOKEN_SECRET` ইত্যাদি `undefined` হবে আর `jwt.sign` error দেবে।

`.env`:

```dotenv
ACCESS_TOKEN_SECRET=long-random-secret-1
REFRESH_TOKEN_SECRET=long-random-secret-2
ACCESS_TOKEN_EXPIRES_IN=1d
REFRESH_TOKEN_EXPIRES_IN=7d
BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN=86400
BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE=43200
FRONTEND_URL=http://localhost:3000
```

> দুটো secret **আলাদা** রাখো। Random secret বানাতে: `openssl rand -base64 32`

`src/app/config/env.ts`-এ: `EnvConfig` interface, `requireEnvVariable` array আর return object — তিন জায়গাতেই comment তুলে দাও, আর `FRONTEND_URL` যোগ করো:

```ts
interface EnvConfig {
    // ...
    FRONTEND_URL: string;
}

const requireEnvVariable = [
    // ...
    "ACCESS_TOKEN_SECRET",
    "REFRESH_TOKEN_SECRET",
    "ACCESS_TOKEN_EXPIRES_IN",
    "REFRESH_TOKEN_EXPIRES_IN",
    "BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN",
    "BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE",
    "FRONTEND_URL",
];

return {
    // ...
    ACCESS_TOKEN_SECRET: process.env.ACCESS_TOKEN_SECRET as string,
    REFRESH_TOKEN_SECRET: process.env.REFRESH_TOKEN_SECRET as string,
    ACCESS_TOKEN_EXPIRES_IN: process.env.ACCESS_TOKEN_EXPIRES_IN as string,
    REFRESH_TOKEN_EXPIRES_IN: process.env.REFRESH_TOKEN_EXPIRES_IN as string,
    BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN: process.env.BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN as string,
    BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE: process.env.BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE as string,
    FRONTEND_URL: process.env.FRONTEND_URL as string,
};
```

---

## ধাপ ৯: `auth.service.ts` — register ও login-এ token বানানো

প্রথমে service-এ token তৈরি করব, তারপর controller-এর response-এ user-কে পাঠাব।

```ts
import AppError from "@/app/errorHelpers/AppError";
import { auth } from "@/app/lib/auth";
import { prisma } from "@/app/lib/prisma";
import { tokenUtils } from "@/app/utils/token";
import { UserStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";

interface IRegisterPatientPayload {
	name: string;
	email: string;
	password: string;
}

interface ILoginUserPayload {
	email: string;
	password: string;
}

// Better Auth-এর user থেকে access + refresh token বানানো
type TAuthUser = Awaited<ReturnType<typeof auth.api.signInEmail>>["user"];

const generateAuthTokens = (user: TAuthUser) => {
	const jwtPayload = {
		userId: user.id,
		role: user.role,
		name: user.name,
		email: user.email,
		status: user.status,
		isDeleted: user.isDeleted,
		emailVerified: user.emailVerified,
	};

	return {
		accessToken: tokenUtils.getAccessToken(jwtPayload),
		refreshToken: tokenUtils.getRefreshToken(jwtPayload),
	};
};

const registerPatient = async (payload: IRegisterPatientPayload) => {
	const { name, email, password } = payload;

	const data = await auth.api.signUpEmail({
		body: { name, email, password },
	});

	// শুধু patient তৈরি fail করলে user মুছে ফেলা হবে
	let patient: Awaited<ReturnType<typeof prisma.patient.create>>;
	try {
		patient = await prisma.patient.create({
			data: {
				userId: data.user.id,
				name,
				email,
			},
		});
	} catch (error) {
		console.error("Failed to create patient:", error);
		await prisma.user.delete({ where: { id: data.user.id } });
		throw error;
	}

	const { accessToken, refreshToken } = generateAuthTokens(data.user);

	return {
		...data,
		accessToken,
		refreshToken,
		patient,
	};
};

const loginUser = async (payload: ILoginUserPayload) => {
	const { email, password } = payload;

	// sign in-এর আগে check, যাতে blocked/deleted user-এর session তৈরি না হয়
	const user = await prisma.user.findUnique({ where: { email } });

	if (!user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Invalid email or password");
	}

	if (user.status === UserStatus.BLOCKED) {
		throw new AppError(httpStatus.FORBIDDEN, "User is blocked");
	}

	if (user.isDeleted || user.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.NOT_FOUND, "User is deleted");
	}

	const data = await auth.api.signInEmail({
		body: { email, password },
	});

	const { accessToken, refreshToken } = generateAuthTokens(data.user);

	return {
		...data,
		accessToken,
		refreshToken,
	};
};

export const AuthService = {
	registerPatient,
	loginUser,
};
```

- `generateAuthTokens` → register আর login দুই জায়গায় একই payload দিয়ে token, তাই আলাদা function।
- `loginUser`-এ blocked/deleted check **`signInEmail`-এর আগে**, যাতে ওদের জন্য DB-তে session-ই তৈরি না হয়।

## ধাপ ১০: `auth.controller.ts` — cookie set ও response

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import { tokenUtils } from "@/app/utils/token";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { AuthService } from "./auth.service";

const registerPatient = catchAsync(async (req: Request, res: Response) => {
	const payload = req.body;

	const result = await AuthService.registerPatient(payload);

	const { accessToken, refreshToken, token, ...rest } = result;

	tokenUtils.setAccessTokenCookie(res, accessToken);
	tokenUtils.setRefreshTokenCookie(res, refreshToken);

	// signUpEmail-এর token null হতে পারে, তাই check করে set করা
	if (token) {
		tokenUtils.setBetterAuthSessionCookie(res, token);
	}

	sendResponse(res, {
		httpStatusCode: httpStatus.CREATED,
		success: true,
		message: "Patient registered successfully",
		data: {
			token,
			accessToken,
			refreshToken,
			...rest,
		},
	});
});

const loginUser = catchAsync(async (req: Request, res: Response) => {
	const payload = req.body;

	const result = await AuthService.loginUser(payload);

	const { accessToken, refreshToken, token, ...rest } = result;

	tokenUtils.setAccessTokenCookie(res, accessToken);
	tokenUtils.setRefreshTokenCookie(res, refreshToken);
	tokenUtils.setBetterAuthSessionCookie(res, token);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "User logged in successfully",
		data: {
			token,
			accessToken,
			refreshToken,
			...rest,
		},
	});
});

export const AuthController = {
	registerPatient,
	loginUser,
};
```

- `signUpEmail`-এর `token` null হতে পারে (যেমন email verification চালু থাকলে auto sign-in হয় না), তাই register-এ `if (token)`। Login-এ `signInEmail` সবসময় token দেয়।

---

## ধাপ ১১: `req.user`-এর type — `src/app/interfaces/express.d.ts`

Token generate আর set করা শেষ, এবার route protect করব। `checkAuth` user-এর তথ্য `req.user`-এ রাখবে, কিন্তু Express-এর `Request`-এ `user` নেই। তাই type বাড়িয়ে দিই:

```ts
import type { Role } from "@/generated/prisma/enums";

declare global {
	namespace Express {
		interface Request {
			user?: {
				userId: string;
				name: string;
				email: string;
				role: Role;
			};
		}
	}
}
```

- `declare global` → Express-এর global `Request` interface-এ নতুন field যোগ (declaration merging)।
- `user?` optional, কারণ public route-এ `req.user` থাকবে না।

## ধাপ ১২: `src/app/middleware/checkAuth.ts`

```ts
import { envVars } from "@/app/config/env";
import AppError from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { CookieUtils } from "@/app/utils/cookie";
import { jwtUtils } from "@/app/utils/jwt";
import { type Role, UserStatus } from "@/generated/prisma/enums";
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status";

// session-এর মেয়াদ এর চেয়ে কম % বাকি থাকলে frontend-কে জানানো হবে
const SESSION_REFRESH_THRESHOLD_PERCENT = 20;

export const checkAuth =
    (...authRoles: Role[]) =>
        async (req: Request, res: Response, next: NextFunction) => {
            try {
                // ধাপ ১: দুইটা cookie-ই থাকতে হবে
                const sessionToken = CookieUtils.getCookie(req, "better-auth.session_token");
                const accessToken = CookieUtils.getCookie(req, "accessToken");

                if (!sessionToken || !accessToken) {
                    throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
                }

                // ধাপ ২: JWT verify (DB লাগে না, তাই আগে)
                // ভুল বা expired হলে verifyToken নিজেই AppError throw করে
                const decoded = jwtUtils.verifyToken(accessToken, envVars.ACCESS_TOKEN_SECRET);

                // ধাপ ৩: DB-তে session খোঁজা
                const session = await prisma.session.findUnique({
                    where: { token: sessionToken },
                    include: { user: true },
                });

                // ✅ Bug ১ fix: session না পেলে বা মেয়াদ শেষ হলে ঢুকতে দেওয়া হবে না
                if (!session || session.expiresAt <= new Date()) {
                    throw new AppError(httpStatus.UNAUTHORIZED, "Session expired or invalid! Please log in again.");
                }

                const { user } = session;

                // ✅ Bug ২ fix: দুইটা token একই user-এর কিনা
                if (decoded.userId !== user.id) {
                    throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Token mismatch.");
                }

                // ধাপ ৪: user-এর অবস্থা DB থেকে (JWT-র পুরনো data-র উপর ভরসা না করে)
                if (user.isDeleted || user.status === UserStatus.DELETED) {
                    throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! User is deleted.");
                }

                if (user.status === UserStatus.BLOCKED) {
                    throw new AppError(httpStatus.FORBIDDEN, "Your account has been blocked.");
                }

                // ধাপ ৫: role check, শুধু DB-র role দিয়ে
                if (authRoles.length > 0 && !authRoles.includes(user.role)) {
                    throw new AppError(
                        httpStatus.FORBIDDEN,
                        "Forbidden access! You do not have permission to access this resource.",
                    );
                }

                // ধাপ ৬: session শেষ হওয়ার কাছাকাছি থাকলে frontend-কে header দিয়ে জানানো
                const sessionLifeTime = session.expiresAt.getTime() - session.createdAt.getTime();
                const timeRemaining = session.expiresAt.getTime() - Date.now();
                const percentRemaining = (timeRemaining / sessionLifeTime) * 100;

                if (percentRemaining < SESSION_REFRESH_THRESHOLD_PERCENT) {
                    res.setHeader("X-Session-Refresh", "true");
                    res.setHeader("X-Session-Expires-At", session.expiresAt.toISOString());
                    res.setHeader("X-Time-Remaining", timeRemaining.toString());
                }

                // ধাপ ৭: controller-এর জন্য user-এর তথ্য রেখে দেওয়া
                req.user = {
                    userId: user.id,
                    name: user.name,
                    email: user.email,
                    role: user.role,
                };

                next();
            } catch (error) {
                next(error);
            }
        };
```

**checkAuth-এর ভিতরের ধাপগুলো:**

| # | কী check | fail হলে |
|---|---|---|
| ১ | `session_token` + `accessToken` দুটো cookie আছে? | 401 |
| ২ | JWT verify (আগে, কারণ DB লাগে না, সস্তা) | 401 (invalid / expired) |
| ৩ | DB-তে session আছে আর মেয়াদ আছে? | 401 |
| — | JWT-র `userId` আর session-এর user একই? | 401 |
| ৪ | user deleted / blocked? (DB থেকে) | 401 / 403 |
| ৫ | role allowed? (DB-র role দিয়ে) | 403 |
| ৬ | session শেষের কাছাকাছি? → response header | — |
| ৭ | `req.user` set → `next()` | — |

- `checkAuth(...authRoles)` → যত খুশি role দেওয়া যায়। কোনো role না দিলে (`checkAuth()`) যেকোনো logged-in user ঢুকতে পারবে।
- **401 বনাম 403:** 401 = তুমি কে জানি না (login করো), 403 = তোমাকে চিনি কিন্তু অনুমতি নেই।
- Status/role JWT থেকে না নিয়ে **DB থেকে** নেওয়া হচ্ছে, কারণ JWT-র data issue হওয়ার পর পুরনো হয়ে যেতে পারে (যেমন login-এর পরে user block হলে)।

## ধাপ ১৩: `src/app.ts` — CORS ঠিক করা

```ts
app.use(
	cors({
		origin: envVars.FRONTEND_URL, // যেমন http://localhost:3000
		credentials: true,            // cookie আসা-যাওয়ার অনুমতি
		exposedHeaders: ["X-Session-Refresh", "X-Session-Expires-At", "X-Time-Remaining"],
	}),
);
```

- `credentials: true` দিলে `origin: "*"` চলে না, নির্দিষ্ট URL দিতে হয়।
- Frontend-এও `fetch(..., { credentials: "include" })` বা axios-এ `withCredentials: true` দিতে হবে, নইলে cookie যাবে না।
- `exposedHeaders` না দিলে browser-এর JS custom header (`X-Session-Refresh` ইত্যাদি) পড়তে পারে না।

## ধাপ ১৪: Route-এ ব্যবহার

```ts
// route ফাইলে
import { checkAuth } from "@/app/middleware/checkAuth"; // ✏️ import নোটে ছিল না
import { Role } from "@/generated/prisma/enums";

router.post("/create-doctor", checkAuth(Role.ADMIN, Role.SUPER_ADMIN), validateRequest(createDoctorZodSchema), UserController.createDoctor);

router.get("/me", checkAuth(), UserController.getMe); // যেকোনো logged-in user
```

- Order: **আগে `checkAuth`, তারপর `validateRequest`** → login না থাকলে body validate করারই দরকার নেই।
- `UserController.getMe` এখনো বানানো হয়নি, এটা শুধু উদাহরণ।
- `Role` এখানে value হিসেবে (`Role.ADMIN`) ব্যবহার হচ্ছে, তাই normal import (`import type` না)।

---

## ধাপ ১৫: Test (Postman)

| # | Method | URL | Body | আশা |
|---|---|---|---|---|
| ১ | `POST` | `/api/v1/auth/register` | `{ "name", "email", "password" }` | 201, ৩টা cookie set |
| ২ | `POST` | `/api/v1/auth/login` | `{ "email", "password" }` | 200, ৩টা cookie set |
| ৩ | `POST` | `/api/v1/user/create-doctor` (login ছাড়া) | doctor body | 401 |
| ৪ | `POST` | `/api/v1/user/create-doctor` (patient হিসেবে login) | doctor body | 403 |
| ৫ | `POST` | `/api/v1/user/create-doctor` (admin হিসেবে login) | doctor body | 201 |

- Base URL: `http://localhost:5000`
- Login-এর পর Postman-এর **Cookies** (response-এর নিচে) tab-এ `accessToken`, `refreshToken`, `better-auth.session_token` দেখা যাবে। Postman একই domain-এর পরের request-এ এগুলো নিজে থেকেই পাঠায়।
- ৫ নম্বরের জন্য DB-তে একজন ADMIN user লাগবে (এখনো seed বানানো হয়নি, Prisma Studio-তে কারো `role` বদলে test করা যায়)।

---

## 🔜 পরে যা যোগ হবে

- [x] Refresh token route → access token expire হলে নতুন access token → [note 17](./17-refresh-token-new-access-token.md)
- [ ] Logout → `clearAuthCookies` + DB থেকে session মুছে ফেলা
- [ ] Change password → নতুন token
- [x] `getMe` route → [note 16](./16-get-me-route.md)
- [ ] Admin seed (প্রথম SUPER_ADMIN বানানো)

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি (উপরে ✏️ চিহ্নিত সংশোধনগুলো ছাড়া)।

### 🔴 গুরুত্বপূর্ণ

1. **ভুল password দিলে সম্ভবত 500 আসবে** — `signInEmail` ভুল password-এ Better Auth-এর `APIError` throw করে। এটা `AppError` না, তাই global error handler (note 12) এটাকে সাধারণ `Error` ধরে **500** দেয়, আর production-এ message হয় "Something went wrong"। একই জিনিস register-এ duplicate email-এ। (test করা হয়নি, Postman-এ ভুল password দিয়ে দেখে নাও।) Fix — `globalErrorHandler`-এ `APIError` আলাদা করে ধরো:
   ```ts
   import { APIError } from "better-auth/api";

   } else if (err instanceof APIError) {
   	statusCode = err.statusCode;
   	message = err.message;
   	errorSources = [{ path: "", message: err.message }];
   	stack = err.stack;
   }
   ```
   (`AppError`-এর branch-এর পরে, সাধারণ `Error`-এর আগে বসাও।)

### 🟡 ভালো practice

2. **Session কখনো extend হবে না (সম্ভবত)** — `updateAge` দিয়ে session বাড়ানোর কাজটা Better Auth করে শুধু তখন, যখন তার নিজের `getSession` call হয়। আমাদের `checkAuth` সরাসরি `prisma.session` পড়ে, তাই `updateAge` আর `cookieCache` কোনোটাই এখন কাজে লাগছে না — session ১ দিন পরে শেষ, user-কে আবার login করতে হবে। এখন সমস্যা না, শুধু জেনে রাখো যে `X-Session-Refresh` header পেলে frontend-কে নতুন করে login/refresh করাতে হবে।

3. **Token response body-তেও যাচ্ছে** — cookie `httpOnly` দেওয়ার উদ্দেশ্য হলো JS যেন token পড়তে না পারে (XSS থেকে বাঁচতে)। কিন্তু body-তে `accessToken`/`refreshToken` পাঠালে frontend JS সেটা পেয়েই যায়। Postman/mobile app-এর জন্য রাখা যায়, শুধু web frontend হলে body থেকে বাদ দিও।

4. **`ACCESS_TOKEN_MAX_AGE` আর `.env` হাতে মিলাতে হচ্ছে** — `.env`-এ `1d` লেখা, code-এ `ONE_DAY`। `.env` বদলালে cookie বদলাবে না। সহজ উপায়: `.env`-এ second-এ রাখো (`ACCESS_TOKEN_EXPIRES_IN=86400`), তারপর:
   ```ts
   const ACCESS_TOKEN_MAX_AGE = Number(envVars.ACCESS_TOKEN_EXPIRES_IN) * 1000;
   // jwt-তে: expiresIn: Number(envVars.ACCESS_TOKEN_EXPIRES_IN)
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **`better-auth.session_token` cookie আমরা নিজে বানাচ্ছি** — Better Auth-এর নিজের cookie-তে token-এর সাথে একটা signature থাকে (`token.signature`), আর production-এ নাম হয় `__Secure-better-auth.session_token`। আমরা raw token রাখছি আর নিজেরাই DB-তে খুঁজছি, তাই এখন ঠিকঠাক কাজ করবে। কিন্তু পরে `auth.api.getSession({ headers })` ব্যবহার করলে এই cookie চিনবে না।

6. **Login-এ "User is deleted" (404)** — এতে বোঝা যায় এই email-এর account আছে। চাইলে deleted-এর জন্যও `"Invalid email or password"` (401) দিতে পারো, তাহলে কেউ email আছে কিনা খুঁজে বের করতে পারবে না।

7. **JWT payload-এ `status`, `isDeleted`** — এগুলো token-এ থাকলেও checkAuth DB থেকেই check করছে (ভালো)। তাই payload-এ এগুলো না রাখলেও চলে, token ছোট হবে। শুধু `userId`, `role`, `email` যথেষ্ট।
