# 07 - Login (Zod + Auth Types + Cookie + Token + Server Action)

> Module: 9-2 (Creating Zod Validation Schema, Auth Types For Login)

> ⚠️ এই নোট **শুধু login**। Login-এর পর JWT দিয়ে কোন route কে দেখতে পারবে (authorization / `proxy.ts`), সেটা পরের নোটে।
> আমাদের code teacher-এর code-এর **ছোট করা (minimized) version** — তাই প্রতিটা file কেন আছে আর teacher থেকে কী আলাদা, সেটা নিচে ধীরে ধীরে বোঝানো হয়েছে।

## 🧭 পুরো Login flow এক নজরে

```
LoginForm (browser)
  → form validate (Zod)  → loginAction(value, redirect)   ← server action (server-এ চলে)
       → আবার Zod validate
       → httpClient.post("/auth/login")  → backend { accessToken, refreshToken, token, user }
       → setAuthCookies()  → ৩টা httpOnly cookie browser-এ বসে
       → কোথায় যাবে ঠিক করা (needPasswordChange? redirect? role-এর dashboard?)
       → redirect(targetPath)
  ← error হলে { success: false, message } ফেরত → form-এ Alert দেখায়
```

### কোন file কী কাজ করে

| File | কাজ | কোথায় চলে |
|---|---|---|
| `types/auth.types.ts` | backend-এর login response দেখতে কেমন | — (শুধু type) |
| `zod/auth.validation.ts` | email/password-এর নিয়ম | client + server দুই জায়গাতেই |
| `lib/cookieConfig.ts` | cookie-র নাম, option, session-এর মেয়াদ (constant) | server + `proxy.ts` |
| `lib/jwtUtils.ts` | token-এর মেয়াদ কতক্ষণ বাকি (`jwt.decode`) | server + `proxy.ts` |
| `lib/cookieUtils.ts` | cookie **get / set / delete** — শুধু এটুকুই | শুধু server (`server-only`) |
| `lib/tokenUtils.ts` | token expire/expiring check + ৩টা cookie একসাথে বসানো | শুধু server (`server-only`) |
| `lib/authUtils.ts` | কোন role কোন route-এ যেতে পারে, login-এর পর কোথায় যাবে | server (পরে `proxy.ts`-ও ব্যবহার করবে) |
| `actions/auth.actions.ts` | **login-এর আসল কাজ** — সব জোড়া লাগায় | server (`"use server"`) |
| `components/shared/form/AppSubmitButton.tsx` | loading spinner সহ submit button | client |
| `components/modules/Auth/LoginForm.tsx` | form UI | client (`"use client"`) |
| `app/.../login/page.tsx` | page — শুধু form বসায় | server |

---

## ধাপ ১: `"use server"` না, `import "server-only"` — কেন?

```shell
pnpm add server-only
```

এটা এই নোটের **সবচেয়ে জরুরি concept**।

| | `"use server"` | `import "server-only"` |
|---|---|---|
| মানে | এই file-এর **প্রতিটা export একটা public endpoint** (server action) | এই file কখনো client bundle-এ ঢুকবে না |
| Browser থেকে call করা যায়? | ✅ হ্যাঁ — যে কেউ call করতে পারে | ❌ না |
| Client component থেকে import করলে? | চলে (server action হিসেবে) | **build error** দেয় |
| কখন ব্যবহার | form submit-এর মতো কাজ যেটা browser থেকে call হবে (`loginAction`) | helper/utility যেটা শুধু server-এর ভেতরে লাগবে (cookie, token) |

**বিপদটা কী?** Teacher-এর `cookieUtils.ts`-এর উপরে `"use server"` আছে। তাহলে `getCookie` একটা public endpoint — যে কেউ browser থেকে `getCookie("accessToken")` call করে **httpOnly cookie পড়ে ফেলতে পারবে**। অথচ httpOnly রাখার উদ্দেশ্যই ছিল JavaScript যেন token পড়তে না পারে। তাই utility file-এ `server-only`।

> 💡 আরেকটা ছোট পার্থক্য: `"use server"` file-এর সব export **async হতেই হবে**। তাই teacher-এর `isTokenExpired` async। আমাদের `server-only` file-এ এই নিয়ম নেই, তাই সাধারণ (sync) function রাখা গেছে।

---

## ধাপ ২: Auth Types — `src/types/auth.types.ts`

Backend login সফল হলে যা পাঠায়, সেটার TypeScript shape:

```ts
export type UserRole = "SUPER_ADMIN" | "ADMIN" | "DOCTOR" | "PATIENT";
export type UserStatus = "ACTIVE" | "BLOCKED" | "DELETED";

export interface IAuthUser {
	id: string;
	name: string;
	email: string;
	emailVerified: boolean;
	image: string | null;
	role: UserRole;
	status: UserStatus;
	needPasswordChange: boolean;
	isDeleted: boolean;
	deletedAt: string | null;
	createdAt: string;
	updatedAt: string;
}

export interface ILoginResponse {
	token: string;
	accessToken: string;
	refreshToken: string;
	redirect: boolean;
	url?: string;
	user: IAuthUser;
}
```

| Field | মানে |
|---|---|
| `accessToken` | আমাদের নিজের JWT — ছোট মেয়াদ, প্রতিটা request-এ "আমি কে" প্রমাণ করে |
| `refreshToken` | আমাদের নিজের JWT — বড় মেয়াদ, access token শেষ হলে নতুন আনতে লাগে |
| `token` | Better Auth-এর **session token** → cookie নাম `better-auth.session_token` |
| `user.role` | login-এর পর কোন dashboard-এ যাবে ঠিক করতে লাগে |
| `user.needPasswordChange` | `true` হলে আগে password বদলাতে পাঠাব |

- `role: string` না লিখে `UserRole` (union) — তাই ভুল role লিখলে TypeScript ধরবে, আর teacher-এর মতো `role as UserRole` cast লাগে না।
- Date গুলো `string` কারণ JSON-এ Date আসে string হয়ে।

---

## ধাপ ৩: Zod Schema — `src/zod/auth.validation.ts`

```ts
import * as z from "zod";

export const loginZodSchema = z.object({
	email: z
		.string("Email is required")
		.trim()
		.toLowerCase()
		.pipe(z.email("Invalid email address")),
	password: z.string("Password is required").min(1, "Password is required"),
});

export type ILoginPayload = z.infer<typeof loginZodSchema>;
```

| অংশ | কেন |
|---|---|
| `.trim().toLowerCase()` | `" Arif@Gmail.com "` → `"arif@gmail.com"`। আগে পরিষ্কার করো… |
| `.pipe(z.email(...))` | …তারপর check করো এটা valid email কিনা। (সরাসরি `z.email()` দিলে আগের space-এর জন্য fail করত) |
| password শুধু `min(1)` | Login-এ শুধু "ফাঁকা না" দেখলেই হয়। `min(8)` এর নিয়ম register-এ লাগে — login-এ দিলে পুরোনো ছোট password-ওয়ালা user ঢুকতেই পারবে না |
| `z.infer` | schema থেকেই type — `{ email: string; password: string }`। আলাদা interface লিখতে হয় না |

> একই schema **২ জায়গায়**: form-এ (user-কে সাথে সাথে error দেখাতে) আর server action-এ (কেউ form বাইপাস করে সরাসরি call করলেও যেন আটকায়)।

---

## ধাপ ৪: JWT + jsonwebtoken — token-এর মেয়াদ জানতে

```shell
pnpm add jsonwebtoken
pnpm add -D @types/jsonwebtoken
```
👉 https://www.npmjs.com/package/jsonwebtoken

**JWT কী?** ৩ অংশের একটা string: `header.payload.signature`। Payload-এ থাকে user info আর **`exp`** (কখন মেয়াদ শেষ, Unix seconds-এ)।

| function | কী করে | secret লাগে? |
|---|---|---|
| `jwt.decode(token)` | শুধু payload **পড়ে** | ❌ না |
| `jwt.verify(token, secret)` | পড়ে + signature check করে (token আসল কিনা) | ✅ হ্যাঁ |

আমাদের এখানে শুধু **`exp` জানা দরকার** (cookie কতক্ষণ রাখব), তাই `decode` যথেষ্ট। Token আসল কিনা, সেটা backend প্রতিটা request-এ নিজেই verify করে।

---

## ধাপ ৫: Cookie Utils — `cookieConfig.ts` + `src/lib/cookieUtils.ts`

> 🔄 Update: আগে cookie-র নাম, option আর session-এর মেয়াদ সব `cookieUtils.ts`-এর ভিতরে ছিল। এখন এগুলো আলাদা **`cookieConfig.ts`**-এ সরানো হয়েছে, আর `jwt.decode`-এর অংশ **`jwtUtils.ts`**-এ।

**কেন ভাগ করা হলো?** পরের নোটের `proxy.ts`-এরও cookie-র নাম, option আর token-এর মেয়াদ লাগবে। কিন্তু `cookieUtils.ts`-এ `next/headers`-এর `cookies()` আর `server-only` আছে — এগুলো server action-এর জন্য। Proxy-তে cookie বসানো হয় `NextResponse`-এর উপর। তাই যে অংশ দুই জায়গাতেই লাগে (নাম, option, মেয়াদ), সেটা আলাদা file-এ রাখা হয়েছে যাতে দুজনেই import করতে পারে, আর একই option দুই জায়গায় দুবার লিখতে না হয়।

| File | কী আছে | `server-only`? | কে ব্যবহার করে |
|---|---|---|---|
| `lib/cookieConfig.ts` | `AUTH_COOKIES`, `AuthCookieName`, `SESSION_TOKEN_MAX_AGE`, `getAuthCookieOptions` | ❌ (শুধু constant) | `cookieUtils`, `tokenUtils`, `proxy.ts` |
| `lib/jwtUtils.ts` | `getTokenSecondsRemaining` (পরে `verifyAccessToken`) | ❌ | `tokenUtils`, `proxy.ts` |
| `lib/cookieUtils.ts` | cookie get / set / delete (`cookies()` দিয়ে) | ✅ | server action |
| `lib/tokenUtils.ts` | token-এর মেয়াদ দেখা + ৩টা cookie বসানো | ✅ | server action |

### (ক) `src/lib/cookieConfig.ts`

> ✏️ এই file-টা নোটে ছিল না — project থেকে নেওয়া হয়েছে।

```ts
export const AUTH_COOKIES = {
	accessToken: "accessToken",
	refreshToken: "refreshToken",
	sessionToken: "better-auth.session_token",
} as const;

export type AuthCookieName = (typeof AUTH_COOKIES)[keyof typeof AUTH_COOKIES];

// backend-এর BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN (604800 = 7 দিন)-এর সাথে sync রাখো
export const SESSION_TOKEN_MAX_AGE = 7 * 24 * 60 * 60;

export const getAuthCookieOptions = (maxAgeInSeconds: number) => ({
	httpOnly: true, // JS দিয়ে পড়া যাবে না → XSS হলেও token চুরি হবে না
	secure: process.env.NODE_ENV === "production", // Safari localhost-এ secure cookie রাখে না
	sameSite: "lax" as const, // CSRF থেকে সুরক্ষা
	path: "/",
	maxAge: maxAgeInSeconds,
});
```

- `getAuthCookieOptions` একটা **function**, কারণ `maxAge` প্রতিটা cookie-র জন্য আলাদা (token-এর নিজের মেয়াদ)।
- `sameSite: "lax" as const` → `as const` না দিলে TypeScript এটাকে সাধারণ `string` ধরে, তখন `cookieStore.set()`-এ type error আসে (ওটা চায় `"lax" | "strict" | "none"`)।

### (খ) `src/lib/cookieUtils.ts`

এর কাজ **শুধু cookie get / set / delete**। Token নিয়ে কিছু জানে না।

```ts
// src/lib/cookieUtils.ts
import { cookies } from "next/headers";
import "server-only";
import { AUTH_COOKIES, type AuthCookieName, getAuthCookieOptions } from "./cookieConfig";

export const setCookie = async (name: AuthCookieName, value: string, maxAgeInSeconds: number) => {
	const cookieStore = await cookies();
	cookieStore.set(name, value, getAuthCookieOptions(maxAgeInSeconds));
};

export const getCookie = async (name: AuthCookieName) => {
	const cookieStore = await cookies();
	return cookieStore.get(name)?.value;
};

export const clearAuthCookies = async () => {
	const cookieStore = await cookies();
	for (const name of Object.values(AUTH_COOKIES)) {
		cookieStore.delete(name);
	}
};
```

### `AUTH_COOKIES` + `AuthCookieName` — কী হচ্ছে?

- `as const` → value গুলো সাধারণ `string` না, হুবহু `"accessToken"` ইত্যাদি literal type।
- `AuthCookieName` = `"accessToken" | "refreshToken" | "better-auth.session_token"`।
- লাভ: `setCookie("acessToken", ...)` (typo) লিখলে **TypeScript error**। Teacher-এর code-এ `name: string`, তাই typo ধরা পড়ত না — cookie ভুল নামে বসে যেত।

### Cookie option গুলো (`getAuthCookieOptions`)

| Option | মানে |
|---|---|
| `httpOnly: true` | browser-এর JavaScript (`document.cookie`) পড়তে পারবে না → XSS হলেও token চুরি হবে না |
| `secure: process.env.NODE_ENV === "production"` | শুধু HTTPS-এ পাঠাবে। Local-এ `false` কারণ localhost HTTP, আর Safari সেখানে secure cookie রাখে না |
| `sameSite: "lax"` | অন্য সাইট থেকে আসা POST request-এ cookie যাবে না → CSRF থেকে সুরক্ষা |
| `path: "/"` | পুরো সাইটে cookie পাওয়া যাবে |
| `maxAge` | কত সেকেন্ড পর browser নিজেই cookie মুছবে |

- `cookies()` হলো Next.js-এর server function — **শুধু server action / route handler-এ `set`/`delete` করা যায়**, সাধারণ server component-এ শুধু পড়া যায়।
- `clearAuthCookies` → ৩টা cookie একসাথে মোছে (logout-এ লাগবে)। Teacher-এর `deleteCookie(name)` এক এক করে।

---

## ধাপ ৬: Token Utils — `jwtUtils.ts` + `src/lib/tokenUtils.ts`

এর কাজ: token-এর মেয়াদ দেখা, তারপর `cookieUtils`-এর `setCookie` দিয়ে cookie বসানো।

### (গ) `src/lib/jwtUtils.ts` — `getTokenSecondsRemaining`

> ✏️ এই file-টা নোটে ছিল না। Login-এর জন্য শুধু `getTokenSecondsRemaining` লাগে, তাই এখানে শুধু সেটুকু। `verifyAccessToken` (`ACCESS_TOKEN_SECRET` দিয়ে) আর `IAccessTokenPayload` এই file-এই যোগ হবে `proxy.ts`-এর নোটে।

```ts
import jwt from "jsonwebtoken";

// decode শুধু payload পড়ে, signature verify করে না। এখানে শুধু exp জানা দরকার।
export const getTokenSecondsRemaining = (token: string): number => {
	const payload = jwt.decode(token);

	if (!payload || typeof payload === "string" || typeof payload.exp !== "number") {
		return 0;
	}

	return Math.max(payload.exp - Math.floor(Date.now() / 1000), 0);
};
```

### (ঘ) `src/lib/tokenUtils.ts`

```ts
// src/lib/tokenUtils.ts
import "server-only";
import { AUTH_COOKIES, SESSION_TOKEN_MAX_AGE } from "./cookieConfig";
import { setCookie } from "./cookieUtils";
import { getTokenSecondsRemaining } from "./jwtUtils";

export { getTokenSecondsRemaining };

export const isTokenExpired = (token: string) => getTokenSecondsRemaining(token) === 0;

export const isTokenExpiringSoon = (token: string, thresholdInSeconds = 300) => {
	const remaining = getTokenSecondsRemaining(token);
	return remaining > 0 && remaining <= thresholdInSeconds;
};

interface AuthTokens {
	accessToken: string;
	refreshToken: string;
	sessionToken: string;
}

export const setAuthCookies = async ({ accessToken, refreshToken, sessionToken }: AuthTokens) => {
	const accessMaxAge = getTokenSecondsRemaining(accessToken);
	const refreshMaxAge = getTokenSecondsRemaining(refreshToken);

	if (!accessMaxAge || !refreshMaxAge) {
		throw new Error("Received an invalid or expired token from the server");
	}

	await Promise.all([
		setCookie(AUTH_COOKIES.accessToken, accessToken, accessMaxAge),
		setCookie(AUTH_COOKIES.refreshToken, refreshToken, refreshMaxAge),
		setCookie(AUTH_COOKIES.sessionToken, sessionToken, SESSION_TOKEN_MAX_AGE),
	]);
};
```

- `export { getTokenSecondsRemaining };` → আগে যারা `@/lib/tokenUtils` থেকে import করত, তাদের import বদলাতে হবে না (re-export)।
- `SESSION_TOKEN_MAX_AGE` এখন `cookieConfig.ts` থেকে আসে।

### `getTokenSecondsRemaining` — ভেঙে বোঝা

```
exp            = 1730800000   (token-এর মেয়াদ শেষ, seconds)
Date.now()/1000 = 1730799100  (এখন, seconds — Date.now() milliseconds দেয় তাই /1000)
বাকি            = 900 সেকেন্ড (১৫ মিনিট)
Math.max(..., 0) → মেয়াদ শেষ হলে negative না হয়ে 0
```

`if` check ৩টা কেন:
- `!payload` → token ভাঙা, decode হয়নি (`null`)
- `typeof payload === "string"` → `jwt.decode` কখনো string-ও ফেরত দেয়, তখন `exp` নেই
- `typeof payload.exp !== "number"` → token-এ `exp`-ই নেই

Teacher-এর version-এ `as JwtPayload` দিয়ে জোর করে type বলা হয়েছিল + try/catch। আমাদেরটায় **type check করে** নিরাপদে বের করা, তাই cast বা try/catch লাগে না (`decode` এমনিতে throw করে না, ভাঙা হলে `null` দেয়)।

### বাকি function

| function | কাজ | কখন লাগবে |
|---|---|---|
| `isTokenExpired` | বাকি 0 সেকেন্ড? | পরের নোটে (proxy) |
| `isTokenExpiringSoon` | ৫ মিনিটের (300s) মধ্যে শেষ হবে? → আগেভাগে refresh | পরের নোটে (refresh token) |
| `setAuthCookies` | ৩টা token একসাথে cookie-তে | **এখন, login-এ** |

### `setAuthCookies` — কেন এভাবে

- প্রতিটা cookie-র `maxAge` = **token-এর নিজের বাকি মেয়াদ** → token যখন শেষ, cookie-ও তখন নিজে মুছে যায়।
- Session token Better Auth-এর, এর মেয়াদ JWT থেকে পড়া যায় না → backend-এর সাথে মিলিয়ে fixed **৭ দিন**।
- মেয়াদ 0 হলে **throw** → ভাঙা token cookie-তে বসানোর চেয়ে login fail দেখানো ভালো। (Teacher-এর version-এ এমন হলে ১ দিনের fallback দিয়ে বসিয়ে দিত।)
- `Promise.all` → ৩টা একসাথে, একটার পর একটা অপেক্ষা না করে।

---

## ধাপ ৭: Auth Utils — `src/lib/authUtils.ts`

এখানে ২টা প্রশ্নের উত্তর: **(১) কোন route কার?** **(২) login-এর পর কোথায় যাব?**

```ts
import type { UserRole } from "@/types/auth.types";

// SUPER_ADMIN আর ADMIN একই dashboard ব্যবহার করে
type DashboardRole = Exclude<UserRole, "SUPER_ADMIN">;
export type RouteOwner = DashboardRole | "COMMON";

interface RouteConfig {
	exact: readonly string[];
	pattern: readonly RegExp[];
}

export const authRoutes = ["/login", "/register", "/forgot-password", "/reset-password", "/verify-email"] as const;

// ক্রম গুরুত্বপূর্ণ: উপরের config আগে মেলানো হয়
// (\/|$) দেওয়ায় "/dashboard-x" আর "/dashboard"-এর সাথে মিলবে না
const protectedRoutes: ReadonlyArray<[RouteOwner, RouteConfig]> = [
	["DOCTOR", { exact: [], pattern: [/^\/doctor\/dashboard(\/|$)/] }],
	["ADMIN", { exact: [], pattern: [/^\/admin\/dashboard(\/|$)/] }],
	["PATIENT", { exact: ["/payment/success"], pattern: [/^\/dashboard(\/|$)/] }],
	["COMMON", { exact: ["/my-profile", "/change-password"], pattern: [] }],
];

const dashboardRoutes: Record<DashboardRole, string> = {
	ADMIN: "/admin/dashboard",
	DOCTOR: "/doctor/dashboard",
	PATIENT: "/dashboard",
};

const toDashboardRole = (role: UserRole): DashboardRole => (role === "SUPER_ADMIN" ? "ADMIN" : role);

const isRouteMatch = (pathname: string, { exact, pattern }: RouteConfig) =>
	exact.includes(pathname) || pattern.some((regex) => regex.test(pathname));

export const isAuthRoute = (pathname: string) => (authRoutes as readonly string[]).includes(pathname);

export const getRouteOwner = (pathname: string): RouteOwner | null => {
	for (const [owner, config] of protectedRoutes) {
		if (isRouteMatch(pathname, config)) return owner;
	}
	return null; // public route
};

export const getDefaultDashboardRoute = (role: UserRole) => dashboardRoutes[toDashboardRole(role)];

export const canAccessRoute = (pathname: string, role: UserRole) => {
	const owner = getRouteOwner(pathname);
	return owner === null || owner === "COMMON" || owner === toDashboardRole(role);
};

const INTERNAL_ORIGIN = "http://internal.invalid";

// শুধু নিজের সাইটের path গ্রহণ করে। "https://evil.com", "//evil.com", "/\evil.com" সব বাতিল।
export const parseInternalPath = (path: unknown): URL | null => {
	if (typeof path !== "string" || !path.startsWith("/") || path.startsWith("//") || path.includes("\\")) {
		return null;
	}

	try {
		const url = new URL(path, INTERNAL_ORIGIN);
		return url.origin === INTERNAL_ORIGIN ? url : null;
	} catch {
		return null;
	}
};

export const getPostLoginRedirect = (role: UserRole, redirectPath?: unknown): string => {
	const url = parseInternalPath(redirectPath);

	if (url && !isAuthRoute(url.pathname) && canAccessRoute(url.pathname, role)) {
		return `${url.pathname}${url.search}${url.hash}`;
	}

	return getDefaultDashboardRoute(role);
};
```

### Type অংশ

| Type | মানে |
|---|---|
| `DashboardRole` | `Exclude` দিয়ে `SUPER_ADMIN` বাদ → `"ADMIN" \| "DOCTOR" \| "PATIENT"` (super admin-ও admin dashboard ব্যবহার করে) |
| `RouteOwner` | route-এর মালিক: ৩টা role অথবা `"COMMON"` (login করা যে কেউ) |
| `RouteConfig` | `exact` = হুবহু মিলতে হবে, `pattern` = regex দিয়ে মেলানো (`/dashboard/...` সব) |

### Route table — কোন route কার

| Route | মালিক | কে ঢুকতে পারবে |
|---|---|---|
| `/doctor/dashboard/...` | DOCTOR | শুধু doctor |
| `/admin/dashboard/...` | ADMIN | admin + super admin |
| `/dashboard/...`, `/payment/success` | PATIENT | শুধু patient |
| `/my-profile`, `/change-password` | COMMON | login করা সবাই |
| বাকি সব (`/`, `/consultation`...) | `null` | public — সবাই |

- **Regex `(\/|$)`**: `/^\/dashboard(\/|$)/` মানে `/dashboard` এর পরে হয় `/` আসবে, নয়তো শেষ। তাই `/dashboard` ✅ `/dashboard/my-appointments` ✅ কিন্তু `/dashboard-x` ❌। (Teacher-এর `/^\/dashboard/` এ `/dashboard-x`-ও মিলে যেত।)
- **ক্রম গুরুত্বপূর্ণ**: `for` loop উপর থেকে নিচে মেলায়, প্রথম যেটা মেলে সেটাই owner।
- Teacher ৪টা আলাদা variable + ৪টা `if` লিখেছিলেন। আমরা একটা **array + loop** — নতুন role/route যোগ করতে শুধু array-তে এক লাইন।

### Function গুলো

| function | কাজ | উদাহরণ |
|---|---|---|
| `toDashboardRole` | SUPER_ADMIN → ADMIN, বাকি যেমন আছে | `"SUPER_ADMIN"` → `"ADMIN"` |
| `isRouteMatch` | একটা path কোনো config-এর সাথে মেলে কিনা | |
| `isAuthRoute` | login/register এর মতো page কিনা | `"/login"` → `true` |
| `getRouteOwner` | path-এর মালিক কে | `"/admin/dashboard/x"` → `"ADMIN"` |
| `getDefaultDashboardRoute` | role-এর নিজের dashboard | `"DOCTOR"` → `"/doctor/dashboard"` |
| `canAccessRoute` | এই role এই path-এ যেতে পারবে? | PATIENT + `/admin/dashboard` → `false` |

### `parseInternalPath` — Open Redirect থেকে বাঁচা 🔒

**সমস্যা:** URL-এ `?redirect=` থাকে, যেমন `/login?redirect=/dashboard/my-appointments` — login-এর পর সেখানে ফেরত পাঠাই। কিন্তু হ্যাকার যদি কাউকে এই link পাঠায়:

```
https://phdoc.com/login?redirect=https://evil.com
```

User আসল সাইটে login করল → সাথে সাথে `evil.com`-এ চলে গেল (যেটা দেখতে হুবহু phdoc-এর মতো, আবার password চাইবে)। এটাকে বলে **Open Redirect**।

Teacher-এর `isValidRedirectForRole`-এ `"https://evil.com"`-এর route owner হয় `null` (public), তাই এটা **valid ধরে নিত**। আমাদের `parseInternalPath` আগে check করে:

| Input | ফলাফল | কেন |
|---|---|---|
| `"/dashboard?tab=1"` | ✅ URL | নিজের সাইটের path |
| `"https://evil.com"` | ❌ null | `/` দিয়ে শুরু না |
| `"//evil.com"` | ❌ null | `//` = browser এটাকে অন্য domain ধরে |
| `"/\evil.com"` | ❌ null | কিছু browser `\`-কে `/` ধরে → `//evil.com` |
| `123` / `undefined` | ❌ null | string না |

শেষে `new URL(path, INTERNAL_ORIGIN)` দিয়ে একটা নকল origin (`internal.invalid`)-এর সাথে জুড়ে দেখি — origin বদলে গেলে মানে path অন্য সাইটে যাচ্ছে → বাতিল। এটা ছাড়াও `url.pathname`, `url.search` আলাদা করে পাওয়া যায় (teacher `split("?")` দিয়ে করেছিলেন)।

### `getPostLoginRedirect` — login-এর পর কোথায়?

```
redirect path আছে, নিজের সাইটের, auth page না, আর এই role ঢুকতে পারে?
   ✅ → সেখানে যাও (query + hash সহ)
   ❌ → role-এর default dashboard
```

| Role | redirect | কোথায় যাবে |
|---|---|---|
| PATIENT | `/dashboard/my-appointments` | `/dashboard/my-appointments` ✅ |
| PATIENT | `/admin/dashboard` | `/dashboard` (admin-এর route, অনুমতি নেই) |
| DOCTOR | `/login` | `/doctor/dashboard` (auth page-এ ফেরত পাঠানোর মানে নেই) |
| ADMIN | `https://evil.com` | `/admin/dashboard` 🔒 |
| SUPER_ADMIN | — | `/admin/dashboard` |

---

## ধাপ ৮: Login Server Action — `src/actions/auth.actions.ts`

এখানেই সব জোড়া লাগে।

```ts
"use server";

import axios from "axios";
import { redirect } from "next/navigation";
import { getPostLoginRedirect } from "@/lib/authUtils";
import { httpClient } from "@/lib/axios/httpClient";
import { setAuthCookies } from "@/lib/tokenUtils";
import type { ApiErrorResponse } from "@/types/api.types";
import type { ILoginResponse } from "@/types/auth.types";
import { loginZodSchema } from "@/zod/auth.validation";

const GENERIC_ERROR = "Something went wrong. Please try again.";
const EMAIL_NOT_VERIFIED = "Email not verified";

const getErrorMessage = (error: unknown): string => {
	if (axios.isAxiosError<ApiErrorResponse>(error)) {
		const { response } = error;

		if (!response) {
			return error.code === "ECONNABORTED"
				? "The server took too long to respond. Please try again."
				: "Unable to reach the server. Please check your connection.";
		}

		// 5xx-এর ভেতরের details user-কে দেখানো হয় না
		if (response.status >= 500) return GENERIC_ERROR;

		return response.data?.message ?? GENERIC_ERROR;
	}

	console.error("[auth.actions]", error instanceof Error ? error.message : "Unknown error");
	return GENERIC_ERROR;
};

// "use server" file-এর প্রতিটা export একটা public endpoint।
// তাই input-কে unknown ধরে আবার validate করা হয়।
export async function loginAction(payload: unknown, redirectPath?: unknown): Promise<ApiErrorResponse> {
	const parsed = loginZodSchema.safeParse(payload);

	if (!parsed.success) {
		return { success: false, message: parsed.error.issues[0]?.message ?? "Invalid input" };
	}

	let targetPath: string;

	try {
		const { data } = await httpClient.post<ILoginResponse>("/auth/login", parsed.data);

		await setAuthCookies({
			accessToken: data.accessToken,
			refreshToken: data.refreshToken,
			sessionToken: data.token,
		});

		targetPath = data.user.needPasswordChange
			? "/change-password"
			: getPostLoginRedirect(data.user.role, redirectPath);
	} catch (error) {
		const message = getErrorMessage(error);

		if (message !== EMAIL_NOT_VERIFIED) {
			return { success: false, message };
		}

		targetPath = `/verify-email?email=${encodeURIComponent(parsed.data.email)}`;
	}

	// try/catch-এর বাইরে: NEXT_REDIRECT ধরে আবার throw করার hack লাগে না
	redirect(targetPath);
}
```

### `getErrorMessage` — user-কে কোন message দেখাব

| অবস্থা | কী দেখাবে |
|---|---|
| Server-এ পৌঁছায়নি, timeout (`ECONNABORTED`) | "The server took too long..." |
| Server-এ পৌঁছায়নি (server বন্ধ / net নেই) | "Unable to reach the server..." |
| 5xx (server-এর ভেতরের সমস্যা) | generic message — ভেতরের details লুকানো |
| 4xx (ভুল password ইত্যাদি) | backend-এর নিজের `message` (যেমন "Invalid credentials") |
| axios-এর error না (যেমন `setAuthCookies`-এর throw) | log করে generic message |

> Teacher দেখাতেন `Login failed: ${error.message}` → user দেখত `"Request failed with status code 401"` — বোঝা যায় না। আমরা backend-এর আসল message দেখাই।

### `loginAction` — ধাপে ধাপে

1. **`payload: unknown`** — `"use server"` মানে public endpoint, কেউ form ছাড়াই যা খুশি পাঠাতে পারে। তাই type-কে বিশ্বাস না করে `unknown` ধরে **Zod দিয়ে আবার validate**।
2. Validation fail → প্রথম error message ফেরত।
3. Backend-এ `POST /auth/login` → `{ data }` = login response (`httpClient` আগেই `response.data` দেয়, তাই এখানে `data` = server-এর `data` field)।
4. **`setAuthCookies`** → ৩টা token cookie-তে। `token` → `sessionToken` নামে।
5. **কোথায় যাবে** → `needPasswordChange` হলে `/change-password`, নাহলে `getPostLoginRedirect`।
6. Error হলে: "Email not verified" হলে verify page-এ পাঠাও (email সহ, `encodeURIComponent` দিয়ে যাতে `+` ইত্যাদি URL ভাঙে না), অন্য error হলে form-এ দেখাতে message ফেরত।
7. **`redirect(targetPath)` সবার শেষে**।

### `redirect()` কেন try/catch-এর বাইরে? (গুরুত্বপূর্ণ)

Next.js-এর `redirect()` আসলে একটা **বিশেষ error throw করে** (`NEXT_REDIRECT`), আর Next সেটা ধরে page বদলায়।

```
try {
  redirect("/dashboard");   ← NEXT_REDIRECT error throw হলো
} catch (error) {           ← আমাদের catch সেটা ধরে ফেলল! 😱 redirect আর হয় না
}
```

তাই teacher-কে catch-এর ভেতরে এই hack লিখতে হয়েছিল:
```ts
if (error.digest.startsWith("NEXT_REDIRECT")) throw error; // আবার ছুড়ে দাও
```

আমরা try-এর ভেতরে শুধু `targetPath` **ঠিক করি**, আর `redirect()` call করি try/catch শেষ হওয়ার **পরে** → catch কখনো ধরতে পারে না, hack লাগে না।

> `Promise<ApiErrorResponse>` — সফল হলে function কিছু ফেরত দেয় না (redirect হয়ে যায়), তাই return type শুধু error response।

---

## ধাপ ৯: Submit Button — `src/components/shared/form/AppSubmitButton.tsx`

```tsx
import type { ComponentProps, ReactNode } from "react";
import { Button } from "@/components/ui/button";
import { Spinner } from "@/components/ui/spinner";
import { cn } from "@/lib/utils";

type AppSubmitButtonProps = Omit<ComponentProps<typeof Button>, "type"> & {
	isPending: boolean;
	pendingLabel?: ReactNode;
};

export default function AppSubmitButton({
	isPending,
	pendingLabel,
	disabled,
	children,
	className,
	...props
}: AppSubmitButtonProps) {
	return (
		<Button
			{...props}
			type="submit"
			disabled={disabled || isPending}
			aria-busy={isPending}
			className={cn("w-full", className)}
		>
			{isPending && <Spinner aria-hidden="true" />}
			{isPending ? (pendingLabel ?? children) : children}
		</Button>
	);
}
```

| অংশ | কেন |
|---|---|
| `ComponentProps<typeof Button>` | shadcn `Button`-এর সব props (variant, size...) এখানেও দেওয়া যায় |
| `Omit<..., "type">` | `type` বাইরে থেকে বদলানো যাবে না — এটা সবসময় `"submit"` |
| `disabled \|\| isPending` | loading চলাকালীন আবার click করা যাবে না (double submit বন্ধ) |
| `aria-busy` | screen reader-কে জানায় "কাজ চলছে" |
| `pendingLabel ?? children` | loading-এ "Logging in..." দেখায়, না দিলে আগের লেখাই |
| `cn("w-full", className)` | default full width, বাইরে থেকে class দিলে merge হয় |

> `Spinner` shadcn component — না থাকলে `pnpm dlx shadcn@latest add spinner`।

---

## ধাপ ১০: Login Form — `src/components/modules/Auth/LoginForm.tsx`

```tsx
"use client";

import { loginAction } from "@/actions/auth.actions";
import AppField from "@/components/shared/form/AppField";
import AppSubmitButton from "@/components/shared/form/AppSubmitButton";
import { Alert, AlertDescription } from "@/components/ui/alert";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/components/ui/card";
import { type ILoginPayload, loginZodSchema } from "@/zod/auth.validation";
import { useForm } from "@tanstack/react-form";
import { Eye, EyeOff } from "lucide-react";
import Link from "next/link";
import { useState, useTransition } from "react";

const defaultValues: ILoginPayload = { email: "", password: "" };

// submit-এর মুহূর্তে URL থেকে পড়া হয়, তাই render-এর সময় searchParams লাগে না
// এখানে কোনো যাচাই নেই। নিরাপদ কিনা, server action-এর getPostLoginRedirect সেটা check করে।
const getRedirectFromUrl = () => new URLSearchParams(window.location.search).get("redirect") ?? undefined;

export default function LoginForm() {
    const [serverError, setServerError] = useState<string | null>(null);
    const [showPassword, setShowPassword] = useState(false);
    const [isPending, startTransition] = useTransition();

    const form = useForm({
        defaultValues,
        validators: { onChange: loginZodSchema },
        onSubmit: ({ value }) => {
            setServerError(null);
            const redirectPath = getRedirectFromUrl();

            // redirect হয়ে নতুন page না আসা পর্যন্ত isPending true থাকে, তাই double-submit হয় না
            startTransition(async () => {
                const result = await loginAction(value, redirectPath);
                if (result?.success === false) {
                    setServerError(result.message);
                }
            });
        },
    });

    return (
        <Card className="mx-auto w-full max-w-md">
            <CardHeader className="text-center">
                <CardTitle className="text-2xl font-bold">Welcome back</CardTitle>
                <CardDescription>Log in to your PH Doc account</CardDescription>
            </CardHeader>

            <CardContent>
                <form
                    noValidate
                    className="space-y-4"
                    onSubmit={(e) => {
                        e.preventDefault();
                        e.stopPropagation();
                        void form.handleSubmit();
                    }}
                >
                    <form.Field name="email">
                        {(field) => (
                            <AppField
                                field={field}
                                label="Email"
                                type="email"
                                placeholder="you@example.com"
                                autoComplete="email"
                                inputMode="email"
                                autoCapitalize="none"
                                spellCheck={false}
                                autoFocus
                            />
                        )}
                    </form.Field>

                    <form.Field name="password">
                        {(field) => (
                            <AppField
                                field={field}
                                label="Password"
                                type={showPassword ? "text" : "password"}
                                placeholder="Enter your password"
                                autoComplete="current-password"
                                endAdornment={
                                    <Button
                                        type="button"
                                        variant="ghost"
                                        size="icon-sm"
                                        onClick={() => setShowPassword((prev) => !prev)}
                                        aria-label={showPassword ? "Hide password" : "Show password"}
                                        aria-pressed={showPassword}
                                    >
                                        {showPassword ? <EyeOff aria-hidden="true" /> : <Eye aria-hidden="true" />}
                                    </Button>
                                }
                            />
                        )}
                    </form.Field>

                    <div className="text-right">
                        <Link href="/forgot-password" className="text-sm text-primary underline-offset-4 hover:underline">
                            Forgot password?
                        </Link>
                    </div>

                    {serverError && (
                        <Alert variant="destructive" aria-live="polite">
                            <AlertDescription>{serverError}</AlertDescription>
                        </Alert>
                    )}

                    <AppSubmitButton isPending={isPending} pendingLabel="Logging in...">
                        Log in
                    </AppSubmitButton>
                </form>
            </CardContent>

            <CardFooter className="justify-center border-t pt-4">
                <p className="text-sm text-muted-foreground">
                    Don&apos;t have an account?{" "}
                    <Link href="/register" className="font-medium text-primary underline-offset-4 hover:underline">
                        Sign up
                    </Link>
                </p>
            </CardFooter>
        </Card>
    );
}
```

> ✏️ `AppField` (`src/components/shared/form/AppField.tsx`) এই নোটে ছিল না — label + input + error message দেখানোর shared component, teacher-এর project-এও একই নামে আছে।

### গুরুত্বপূর্ণ অংশগুলো

**১. `useTransition` (teacher: `useMutation`)**

| | Teacher: `useMutation` | আমাদের: `useTransition` |
|---|---|---|
| কোথা থেকে | TanStack Query | React নিজে |
| `isPending` কখন `false` | server action **return করলেই** | redirect হয়ে **নতুন page আসা পর্যন্ত** `true` থাকে |
| ফল | redirect চলার মাঝে button আবার active → double click সম্ভব | পুরো সময় button disabled ✅ |

Login-এ cache বা refetch-এর দরকার নেই, তাই TanStack Query-র চেয়ে React-এর built-in `useTransition` যথেষ্ট।

**২. `validators: { onChange: loginZodSchema }`** — পুরো schema form-level-এ একবার। Teacher প্রতিটা field-এ আলাদা দিয়েছিলেন (`loginZodSchema.shape.email`)। কাজ একই।

**৩. `getRedirectFromUrl` (teacher: page থেকে `searchParams` prop)**
- Teacher page-এ `await searchParams` করে `redirectPath` prop হিসেবে দিতেন → page **dynamic** হয়ে যেত (প্রতি request-এ render)।
- আমরা submit-এর মুহূর্তে `window.location.search` থেকে পড়ি → page **static** থাকে (দ্রুত), আর Cache Components-এ Suspense-ঝামেলা নেই।
- এখানে কোনো যাচাই নেই — যাচাই করে server-এর `getPostLoginRedirect` (client-এর check বিশ্বাস করা যায় না)।

**৪. `onSubmit` form tag-এ**
- `e.preventDefault()` → browser-এর নিজের form submit (page reload) বন্ধ।
- `void form.handleSubmit()` → TanStack Form validate করে, ঠিক থাকলে আমাদের `onSubmit` চালায়। `void` = "promise-টা ইচ্ছা করেই await করছি না" (ESLint no-floating-promises খুশি)।
- `noValidate` → browser-এর নিজের email popup বন্ধ, শুধু Zod-এর error দেখাবে।

**৫. Input attributes** — `autoComplete="current-password"` (password manager চিনবে), `inputMode="email"` (mobile-এ `@` সহ keyboard), `autoCapitalize="none"` (mobile প্রথম অক্ষর বড় করবে না)।

**৬. Password show/hide** — `type="button"` না দিলে button টা form submit করে ফেলত! `aria-label`/`aria-pressed` screen reader-এর জন্য।

---

## ধাপ ১১: Login Page — `src/app/(commonLayout)/(authRouteGroup)/login/page.tsx`

```tsx
import type { Metadata } from "next";
import LoginForm from "@/components/modules/Auth/LoginForm";

export const metadata: Metadata = {
	title: "Login",
};

// searchParams পড়া হয় না, তাই পুরো page static shell-এ prerender হয়
export default function LoginPage() {
	return (
		<main className="flex min-h-[70vh] items-center justify-center px-4 py-10">
			<LoginForm />
		</main>
	);
}
```

Page নিজে server component, শুধু form বসায়। `async` নেই, `searchParams` নেই → static।

---

## 🧪 Test

| করো | হওয়ার কথা |
|---|---|
| ফাঁকা form submit | "Invalid email address" / "Password is required" |
| ভুল password | backend-এর message Alert-এ (যেমন "Invalid credentials") |
| Server বন্ধ করে login | "Unable to reach the server..." |
| ঠিক PATIENT login | `/dashboard`-এ যাবে |
| `/login?redirect=/my-profile` দিয়ে login | `/my-profile`-এ যাবে |
| `/login?redirect=https://evil.com` | নিজের dashboard-এ যাবে 🔒 |
| verify না করা email | `/verify-email?email=...` |
| DevTools → Application → Cookies | `accessToken`, `refreshToken`, `better-auth.session_token` — ৩টাতেই **HttpOnly ✓** |
| Console-এ `document.cookie` | ৩টার কোনোটাই দেখাবে না (httpOnly) |

---

## 📊 Teacher বনাম আমাদের — সারাংশ

| বিষয় | Teacher | আমাদের | কেন |
|---|---|---|---|
| utils file-এর উপরে | `"use server"` | `import "server-only"` | `getCookie` public endpoint হয়ে যেত |
| Cookie name | `string` | `AuthCookieName` union | typo ধরা পড়ে |
| Cookie config | প্রতিটা file-এ আলাদা | `cookieConfig.ts`-এ একবার | `proxy.ts`-ও একই option ব্যবহার করে |
| `secure` | সবসময় `true` | production-এ শুধু `true` | Safari localhost |
| Token → cookie | `setTokenInCookies` ৩ বার, fallback ১ দিন | `setAuthCookies` একবারে, ভাঙা token হলে throw | ভুল token বসে না |
| Token helper | async (`"use server"`-এর নিয়ম) | sync | `server-only`-এ নিয়ম নেই |
| Route config | ৪টা variable + if chain | array + loop | যোগ করা সহজ |
| Regex | `/^\/dashboard/` | `/^\/dashboard(\/\|$)/` | `/dashboard-x` মিলবে না |
| Redirect check | `isValidRedirectForRole` | `parseInternalPath` + `getPostLoginRedirect` | **Open redirect বন্ধ** |
| Action input | `ILoginPayload` | `unknown` + Zod | public endpoint, বিশ্বাস করা যায় না |
| `redirect()` | try-এর ভেতরে + `NEXT_REDIRECT` hack | try/catch-এর পরে | hack লাগে না |
| Error message | `Login failed: ${error.message}` | backend-এর message / সুন্দর message | user বোঝে |
| needPasswordChange | `/reset-password?email=` | `/change-password` | user তো login করা, তাই change |
| Pending state | `useMutation` | `useTransition` | redirect পর্যন্ত disabled |
| redirect param | page-এ `searchParams` → prop | submit-এ `window.location` | page static |

---

## 🔜 পরে যা যোগ হবে
- [ ] `proxy.ts` — login না করলে protected route-এ ঢুকতে না দেওয়া, ভুল role হলে নিজের dashboard-এ পাঠানো (`getRouteOwner`, `canAccessRoute`, `isTokenExpired` এখানে লাগবে)
- [ ] Access token শেষ হওয়ার আগে refresh token দিয়ে নতুন token (`isTokenExpiringSoon`)
- [x] `httpClient`-এ cookie থেকে token পড়ে backend-এ পাঠানো — note 15
- [x] Logout (`clearAuthCookies`) — note 11
- [ ] `getUserInfo` (`/auth/me`)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice
1. **`"Email not verified"` string-এর উপর নির্ভরতা ভঙ্গুর** — backend message একটু বদলালেই (যেমন "Email is not verified") verify page-এ যাওয়া বন্ধ হয়ে যাবে, কোনো error ছাড়াই। পরে backend থেকে একটা `code` (যেমন `"EMAIL_NOT_VERIFIED"`) পাঠালে সেটা check করা নিরাপদ:
   ```ts
   if (axios.isAxiosError(error) && error.response?.data?.code === "EMAIL_NOT_VERIFIED") { ... }
   ```
2. **`SESSION_TOKEN_MAX_AGE` (`cookieConfig.ts`) backend-এর সাথে sync** — backend-এ session ৭ দিন না হলে এখানেও বদলাতে হবে, নাহলে cookie থাকবে কিন্তু session শেষ (বা উল্টো)। Comment-এ লেখা আছে, শুধু মনে রাখো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো
3. **`authUtils.ts`-এ `server-only` দেওয়া হয়নি — এটা ঠিক আছে।** এতে কোনো secret বা cookie নেই, আর পরে `proxy.ts` এবং হয়তো client component (যেমন sidebar-এ role অনুযায়ী menu) এটা ব্যবহার করবে।
4. **`jwt.decode` বিশ্বাসযোগ্য না** — শুধু cookie-র মেয়াদ বা UI-এর জন্য ঠিক আছে। কোনো "এই user admin কিনা" সিদ্ধান্ত frontend-এ `decode` দিয়ে নিলে, সেটা শুধু UX — আসল নিরাপত্তা সবসময় backend-এর `verify`।
