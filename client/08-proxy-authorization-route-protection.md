# 08 - Proxy: Route Protection ও Role-based Authorization

> Login-এর পর (note 07) browser-এ ৩টা httpOnly cookie বসে। এই নোটে **`src/proxy.ts`** দিয়ে প্রতিটা page request-এর আগে সেই cookie দেখে ঠিক করব: user ঢুকতে পারবে, নাকি login/dashboard-এ পাঠাতে হবে। দরকার হলে token নিজেই refresh করব।

👉 https://nextjs.org/docs/app/getting-started/proxy

## 🧭 পুরো flow এক নজরে

```
Browser → /admin/dashboard চাইল
  → proxy.ts (page render-এর আগেই চলে)
      → cookie পড়া (accessToken, refreshToken, session)
      → accessToken verify (ACCESS_TOKEN_SECRET দিয়ে)
      → মেয়াদ শেষ / ৫ মিনিটের কম বাকি? → backend /auth/refresh-token → নতুন ৩টা cookie
      → blocked/deleted? → বের করে দাও
      → সিদ্ধান্ত: next() ✅  |  /login-এ ↩️  |  /verify-email | /change-password | নিজের dashboard-এ ↩️
  → page render (Server Component নতুন token-ই দেখে)
```

---

## ধাপ ০: Proxy কী, আর কেন লাগে

**Proxy** = Next.js-এর একটা file, যেটা **প্রতিটা request route-এ পৌঁছানোর আগে** চলে। দরজার দারোয়ানের মতো: ভিতরে ঢোকার আগেই ID card (token) দেখে।

| বিষয় | মানে |
|---|---|
| File-এর জায়গা | `src/proxy.ts` (`src/` folder থাকলে `app/`-এর পাশে, root-এ না) |
| Export | `export async function proxy(request)` + `export const config = { matcher }` |
| আগের নাম | Next.js 15 পর্যন্ত `middleware.ts` / `middleware()`। Next.js 16-এ নাম বদলে `proxy` (middleware এখন deprecated) |
| Runtime | **Node.js** (Edge না)। তাই `jsonwebtoken` এখানে কাজ করে, `jose` লাগে না |
| একটা project-এ | একটাই proxy file |

**Proxy কেন, page-এ check করলে হয় না?**
- প্রতিটা protected page-এ আলাদা check লিখতে হত — একটা ভুলে গেলেই ফাঁক।
- Proxy page render-এর **আগে** redirect করে, তাই protected page-এর এক ঝলকও দেখা যায় না।
- Token refresh একটা জায়গাতেই হয়।

> ⚠️ Proxy শুধু **UX আর প্রথম দরজা**। আসল নিরাপত্তা backend-এর `checkAuth` — frontend-এর check কেউ এড়িয়ে সরাসরি API call করলেও backend আটকাবে।

### কোন route কার (note 07-এর `authUtils.ts` থেকে)

| ধরন | Route | কে ঢুকতে পারবে |
|---|---|---|
| Public | `/`, `/consultation`, `/medicine` … (কোনো list-এ নেই) | সবাই |
| Auth page | `/login`, `/register`, `/forgot-password`, `/reset-password`, `/verify-email` | শুধু logged-out (logged-in হলে dashboard-এ) |
| `DOCTOR` | `/doctor/dashboard/...` | DOCTOR |
| `ADMIN` | `/admin/dashboard/...` | ADMIN, SUPER_ADMIN |
| `PATIENT` | `/dashboard/...`, `/payment/success` | PATIENT |
| `COMMON` | `/my-profile`, `/change-password` | যেকোনো logged-in user |

Proxy এখানে `authUtils.ts`-এর ৪টা function ব্যবহার করে:

| Function | কাজ |
|---|---|
| `getRouteOwner(pathname)` | route কার? (`"ADMIN"`, `"COMMON"` … বা public হলে `null`) |
| `isAuthRoute(pathname)` | login/register-এর মতো page কিনা |
| `canAccessRoute(pathname, role)` | এই role এই route-এ যেতে পারবে কিনা |
| `getDefaultDashboardRoute(role)` | role-এর নিজের dashboard |

---

## ধাপ ১: `.env`-এ `ACCESS_TOKEN_SECRET`

`phdoc-client/.env`

```dotenv
# backend-এর ACCESS_TOKEN_SECRET-এর হুবহু একই মান। NEXT_PUBLIC_ দেবে না!
ACCESS_TOKEN_SECRET=your_backend_access_secret
```

- Proxy access token **verify** করবে, তাই backend যে secret দিয়ে sign করেছে, সেই একই secret লাগবে।
- `NEXT_PUBLIC_` দিলে secret browser-এর JavaScript bundle-এ চলে যাবে → যে কেউ দেখে নিজেই admin token বানাতে পারবে। তাই **কখনো না**।
- `NEXT_PUBLIC_API_BASE_URL` আগে থেকেই আছে (note 04), proxy backend-এ refresh call করতে এটা ব্যবহার করে।
- `.env` বদলালে `pnpm dev` আবার চালাতে হয়।

> ✏️ `.env.example`-এও একই লাইন (আসল secret ছাড়া) রাখো, যাতে পরের project-এ মনে থাকে।

---

## ধাপ ২: নতুন file `src/lib/cookieConfig.ts`

> 🔄 Update (note 28 — deploy): production-এ Google login-এর cookie frontend পাচ্ছিল না, তাই cookie-তে `domain` (`COOKIE_DOMAIN`) যোগ হয়েছে। `cookieScope` (path + domain) set আর delete দুই জায়গায় একই, আর মোছার জন্য নতুন `getAuthCookieDeleteOptions`। বিস্তারিত: [note 28](./28-vps-deploy-and-cookie-domain.md)।

Cookie-র নাম, option আর session-এর মেয়াদ এখন এক জায়গায়। পুরো code আর ব্যাখ্যা **note 07-এর ধাপ ৫ (ক)**-এ আছে।

**কেন আলাদা file?** `cookieUtils.ts`-এ `next/headers`-এর `cookies()` + `server-only` আছে — ওগুলো Server Action-এর জন্য। Proxy-তে cookie পড়া/বসানো হয় `request.cookies` / `response.cookies` দিয়ে। কিন্তু **নাম আর option দুই জায়গাতেই একই হতে হবে** (নাহলে proxy এক option-এ বসাবে, action আরেক option-এ — দুটো আলাদা cookie হয়ে যাবে)। তাই shared অংশ আলাদা file-এ।

```ts
export const AUTH_COOKIES = {
	accessToken: "accessToken",
	refreshToken: "refreshToken",
	sessionToken: "better-auth.session_token",
} as const;

export type AuthCookieName = (typeof AUTH_COOKIES)[keyof typeof AUTH_COOKIES];

// backend-এর BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN (604800 = 7 দিন)-এর সাথে sync রাখো
export const SESSION_TOKEN_MAX_AGE = 7 * 24 * 60 * 60;

// production-এ ".arifuddincoder.site" → phdoc আর phdocapi দুই subdomain-ই cookie পায়
// local-এ নেই (undefined) → cookie শুধু localhost-এর জন্য
// NEXT_PUBLIC_ দেবে না: এটা শুধু server (proxy, server action)-এ লাগে
const COOKIE_DOMAIN = process.env.COOKIE_DOMAIN;

// set আর delete দুই জায়গাতেই একই path + domain লাগে, নইলে browser অন্য cookie ভাবে
const cookieScope = {
	path: "/",
	...(COOKIE_DOMAIN && { domain: COOKIE_DOMAIN }),
};

export const getAuthCookieOptions = (maxAgeInSeconds: number) => ({
	httpOnly: true, // JS দিয়ে পড়া যাবে না → XSS হলেও token চুরি হবে না
	secure: process.env.NODE_ENV === "production", // Safari localhost-এ secure cookie রাখে না
	sameSite: "lax" as const, // CSRF থেকে সুরক্ষা
	maxAge: maxAgeInSeconds,
	...cookieScope,
});

// cookie মোছার সময়ও ঠিক একই domain/path দিতে হয়
export const getAuthCookieDeleteOptions = (name: AuthCookieName) => ({
	name,
	...cookieScope,
});
```

---

## ধাপ ৩: নতুন file `src/lib/jwtUtils.ts`

> ✏️ এই file-এর পুরো code তোমার নোটে ছিল না (শুধু table-এ নাম ছিল) — project থেকে নেওয়া হয়েছে।

```ts
import type { UserRole, UserStatus } from "@/types/auth.types";
import jwt, { type JwtPayload } from "jsonwebtoken";

const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET;

if (!ACCESS_TOKEN_SECRET) {
	throw new Error("ACCESS_TOKEN_SECRET is not defined in environment variables");
}

// backend-এর generateAuthTokens()-এর payload-এর সাথে মিল রাখো
export interface IAccessTokenPayload extends JwtPayload {
	userId: string;
	role: UserRole;
	name: string;
	email: string;
	status: UserStatus;
	isDeleted: boolean;
	emailVerified: boolean;
	needPasswordChange?: boolean; // পুরনো token-এ না-ও থাকতে পারে
}

const USER_ROLES: readonly UserRole[] = ["SUPER_ADMIN", "ADMIN", "DOCTOR", "PATIENT"];

// signature + মেয়াদ দুটোই check করে। ভুল হলে null (throw করে না)
export const verifyAccessToken = (token: string): IAccessTokenPayload | null => {
	try {
		// algorithms pin করা: "alg: none" বা অন্য algorithm দিয়ে জাল token বাতিল
		const decoded = jwt.verify(token, ACCESS_TOKEN_SECRET, { algorithms: ["HS256"] });

		if (typeof decoded === "string" || !USER_ROLES.includes(decoded.role)) {
			return null;
		}

		return decoded as IAccessTokenPayload;
	} catch {
		return null;
	}
};

// decode শুধু payload পড়ে, signature verify করে না। এখানে শুধু exp জানা দরকার।
export const getTokenSecondsRemaining = (token: string): number => {
	const payload = jwt.decode(token);

	if (!payload || typeof payload === "string" || typeof payload.exp !== "number") {
		return 0;
	}

	return Math.max(payload.exp - Math.floor(Date.now() / 1000), 0);
};
```

### ভেঙে বোঝা

| অংশ | কেন |
|---|---|
| file-এর উপরে `if (!ACCESS_TOKEN_SECRET) throw` | secret ভুলে গেলে app চালু হওয়ার সময়েই error — পরে অদ্ভুত bug খুঁজতে হয় না। এর পরে TypeScript জানে `ACCESS_TOKEN_SECRET` হলো `string` |
| `IAccessTokenPayload` | backend-এর JWT payload-এর shape (server note 41-এর `generateAuthTokens`)। Backend-এ field যোগ করলে এখানেও |
| `needPasswordChange?` | backend-এ এটা পরে যোগ হয়েছে, তাই পুরনো token-এ নাও থাকতে পারে → optional |
| `jwt.verify` (decode না) | এখানে **সিদ্ধান্ত** নেওয়া হচ্ছে (কে admin), তাই signature check বাধ্যতামূলক। `decode` হলে যে কেউ jwt.io-তে নিজের token বানিয়ে `role: "ADMIN"` লিখে ঢুকে যেত |
| `algorithms: ["HS256"]` | backend HS256 দিয়ে sign করে (server note 13)। Pin না করলে attacker অন্য algorithm-এর token দিয়ে চেষ্টা করতে পারে |
| `USER_ROLES.includes(...)` | token-এ অজানা role থাকলে বাতিল (runtime check, TypeScript-এর type runtime-এ থাকে না) |
| `try/catch` → `null` | expired/ভাঙা token-এ `jwt.verify` **throw** করে। আমরা `null` চাই, যাতে proxy সহজে "user নেই" ধরে নিতে পারে |
| `getTokenSecondsRemaining` | note 07-এ `tokenUtils`-এ ছিল, এখন এখানে — কারণ proxy-ও এটা ব্যবহার করে আর `tokenUtils`-এ `server-only` আছে |

### `verify` বনাম `decode` — কোথায় কোনটা

| | `verifyAccessToken` | `getTokenSecondsRemaining` |
|---|---|---|
| ভিতরে | `jwt.verify` | `jwt.decode` |
| Secret | ✅ লাগে | ❌ লাগে না |
| কাজ | user কে, role কী — **বিশ্বাসযোগ্য** | cookie-র `maxAge` কত দেব — শুধু হিসাব |
| কোথায় | proxy | proxy + `setAuthCookies` |

---

## ধাপ ৪: আপডেট করা `cookieUtils.ts` আর `tokenUtils.ts`

এই দুটো file এখন `cookieConfig.ts` আর `jwtUtils.ts` থেকে import করে। পুরো আপডেট করা code **note 07-এর ধাপ ৫ (খ) আর ধাপ ৬ (ঘ)**-এ আছে।

### কে কী করে, এখন

| File | কী আছে | কে ব্যবহার করে |
|---|---|---|
| `cookieConfig.ts` (নতুন) | cookie-র নাম, option, session-এর মেয়াদ | proxy আর cookieUtils/tokenUtils |
| `jwtUtils.ts` (নতুন) | `verifyAccessToken`, `getTokenSecondsRemaining` | proxy আর tokenUtils |
| `cookieUtils.ts` (আগের) | `cookies()` দিয়ে set/get/clear | শুধু Server Action |
| `tokenUtils.ts` (আগের) | `setAuthCookies()` (login-এর পর) | শুধু Server Action |

```
cookieConfig.ts ──┬──► cookieUtils.ts ──► tokenUtils.ts ──► auth.actions.ts (login)
                  │                          ▲
jwtUtils.ts ──────┼──────────────────────────┘
                  └──► proxy.ts ◄── authUtils.ts
```

---

## ধাপ ৫: নতুন file `src/proxy.ts` (পুরো code)

```ts
import { canAccessRoute, getDefaultDashboardRoute, getRouteOwner, isAuthRoute } from "@/lib/authUtils";
import {
	AUTH_COOKIES,
	getAuthCookieDeleteOptions,
	getAuthCookieOptions,
	SESSION_TOKEN_MAX_AGE,
} from "@/lib/cookieConfig";
import { getTokenSecondsRemaining, type IAccessTokenPayload, verifyAccessToken } from "@/lib/jwtUtils";
import { type NextRequest, NextResponse } from "next/server";

const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL;

const REFRESH_BEFORE_EXPIRY_SECONDS = 5 * 60; // মেয়াদ শেষের ৫ মিনিট আগেই refresh
const REFRESH_TIMEOUT_MS = 5000; // backend আটকে গেলে পুরো site যেন আটকে না যায়

const ROUTES = {
	login: "/login",
	forgotPassword: "/forgot-password",
	resetPassword: "/reset-password",
	verifyEmail: "/verify-email",
	changePassword: "/change-password",
} as const;

interface IAuthTokens {
	accessToken: string;
	refreshToken: string;
	sessionToken: string;
}

type RefreshResult =
	| { status: "success"; tokens: IAuthTokens }
	| { status: "invalid" } // refresh token/session বাতিল → logout করাতে হবে
	| { status: "error" }; // network/backend সমস্যা → এবার কিছু মুছব না

// ---------- Backend call ----------
const refreshAuthTokens = async (refreshToken: string, sessionToken: string): Promise<RefreshResult> => {
	if (!API_BASE_URL) return { status: "error" };

	try {
		const res = await fetch(`${API_BASE_URL}/auth/refresh-token`, {
			method: "POST",
			headers: {
				// backend-এর getNewToken দুটো cookie-ই চায়
				Cookie: `${AUTH_COOKIES.refreshToken}=${refreshToken}; ${AUTH_COOKIES.sessionToken}=${sessionToken}`,
			},
			cache: "no-store",
			signal: AbortSignal.timeout(REFRESH_TIMEOUT_MS),
		});

		if (res.status === 401 || res.status === 403) return { status: "invalid" };
		if (!res.ok) return { status: "error" };

		const body = (await res.json()) as { data?: Partial<IAuthTokens> };
		const { accessToken, refreshToken: newRefreshToken, sessionToken: newSessionToken } = body.data ?? {};

		if (!accessToken || !newRefreshToken || !newSessionToken) return { status: "error" };

		return {
			status: "success",
			tokens: { accessToken, refreshToken: newRefreshToken, sessionToken: newSessionToken },
		};
	} catch (error) {
		console.error("[proxy] token refresh failed:", error instanceof Error ? error.message : error);
		return { status: "error" };
	}
};

// ---------- Helpers ----------
const getTokenCookies = (tokens: IAuthTokens) =>
	[
		[AUTH_COOKIES.accessToken, tokens.accessToken, getTokenSecondsRemaining(tokens.accessToken)],
		[AUTH_COOKIES.refreshToken, tokens.refreshToken, getTokenSecondsRemaining(tokens.refreshToken)],
		[AUTH_COOKIES.sessionToken, tokens.sessionToken, SESSION_TOKEN_MAX_AGE],
	] as const;

const redirectToLogin = (request: NextRequest) => {
	const { pathname, search } = request.nextUrl;
	const loginUrl = new URL(ROUTES.login, request.url);
	// login-এর পর getPostLoginRedirect() এটা আবার validate করে (open redirect safe)
	loginUrl.searchParams.set("redirect", `${pathname}${search}`);
	return NextResponse.redirect(loginUrl);
};

// ---------- Main logic ----------
const handleAuth = async (request: NextRequest): Promise<NextResponse> => {
	const { pathname, searchParams } = request.nextUrl;
	const routeOwner = getRouteOwner(pathname);
	const isAuthPage = isAuthRoute(pathname);
	const isPublicPage = routeOwner === null && !isAuthPage;

	const accessToken = request.cookies.get(AUTH_COOKIES.accessToken)?.value;
	const refreshToken = request.cookies.get(AUTH_COOKIES.refreshToken)?.value;
	const sessionToken = request.cookies.get(AUTH_COOKIES.sessionToken)?.value;

	// ⚡ Fast path: কোনো token নেই, public page → কোনো কাজই লাগবে না
	if (!accessToken && !refreshToken && isPublicPage) {
		return NextResponse.next();
	}

	// ১) access token একবারই verify
	let user: IAccessTokenPayload | null = accessToken ? verifyAccessToken(accessToken) : null;
	let newTokens: IAuthTokens | null = null;
	let refreshStatus: RefreshResult["status"] | null = null;

	// ২) token নেই / expired / শীঘ্রই expire → refresh (exp verify-এর payload থেকেই, আবার decode লাগে না)
	const secondsLeft = user?.exp ? user.exp - Math.floor(Date.now() / 1000) : 0;

	if (refreshToken && sessionToken && secondsLeft <= REFRESH_BEFORE_EXPIRY_SECONDS) {
		const result = await refreshAuthTokens(refreshToken, sessionToken);
		refreshStatus = result.status;

		if (result.status === "success") {
			newTokens = result.tokens;
			user = verifyAccessToken(result.tokens.accessToken);
		} else if (result.status === "invalid") {
			user = null; // session revoke হয়েছে (যেমন logout-all)
		}
		// "error" হলে: পুরনো token এখনো valid থাকলে সেটা দিয়েই চলবে
	}

	// ৩) blocked/deleted user-কে বের করে দাও
	if (user && (user.status !== "ACTIVE" || user.isDeleted)) {
		user = null;
		refreshStatus = "invalid";
	}

	// বাসি cookie মুছে দাও, যাতে প্রতিটা request-এ বারবার refresh চেষ্টা না হয়।
	// backend সাময়িক down ("error") হলে মুছি না, নইলে সবাই অকারণে logout হবে।
	const shouldClearCookies = !user && refreshStatus !== "error" && Boolean(accessToken || refreshToken);

	// response-এ cookie বসানো (browser-এর জন্য)
	const finalize = (response: NextResponse) => {
		if (newTokens) {
			for (const [name, value, maxAge] of getTokenCookies(newTokens)) {
				response.cookies.set(name, value, getAuthCookieOptions(maxAge));
			}
		} else if (shouldClearCookies) {
			for (const name of Object.values(AUTH_COOKIES)) response.cookies.delete(getAuthCookieDeleteOptions(name));
		}
		return response;
	};

	// request-এও বসানো, যাতে এই request-এর Server Component/Action নতুন token দেখে
	const next = () => {
		if (newTokens) {
			for (const [name, value] of getTokenCookies(newTokens)) request.cookies.set(name, value);
		} else if (shouldClearCookies) {
			for (const name of Object.values(AUTH_COOKIES)) request.cookies.delete(name);
		}
		return finalize(NextResponse.next({ request: { headers: request.headers } }));
	};

	const redirectTo = (path: string) => finalize(NextResponse.redirect(new URL(path, request.url)));

	// ৪) Public page → সবার জন্য খোলা (token refresh হয়ে থাকলে cookie-ও যাবে)
	if (isPublicPage) return next();

	// ৫) Logged-out user
	if (!user) {
		if (routeOwner !== null) return finalize(redirectToLogin(request));

		// forgot-password থেকে না এলে reset-password-এর মানে নেই
		if (pathname === ROUTES.resetPassword && !searchParams.get("email")) {
			return redirectTo(ROUTES.forgotPassword);
		}

		return next(); // login, register ইত্যাদি
	}

	// ---------- এখান থেকে user নিশ্চিতভাবে logged-in ----------
	const dashboard = getDefaultDashboardRoute(user.role);

	// ৬) Email verify বাধ্যতামূলক
	if (!user.emailVerified) {
		return pathname === ROUTES.verifyEmail
			? next()
			: redirectTo(`${ROUTES.verifyEmail}?email=${encodeURIComponent(user.email)}`);
	}

	// ৭) Password বদলানো বাধ্যতামূলক (তোমার loginAction-ও /change-password-এ পাঠায়)
	if (user.needPasswordChange) {
		return pathname === ROUTES.changePassword ? next() : redirectTo(ROUTES.changePassword);
	}

	// ৮) Logged-in user-এর login/register/verify-email/reset-password-এ কাজ নেই
	if (isAuthPage) return redirectTo(dashboard);

	// ৯) Role check (SUPER_ADMIN → ADMIN, COMMON সবার জন্য: canAccessRoute সামলায়)
	if (!canAccessRoute(pathname, user.role)) return redirectTo(dashboard);

	return next();
};

export async function proxy(request: NextRequest) {
	try {
		return await handleAuth(request);
	} catch (error) {
		console.error("[proxy] unexpected error:", error instanceof Error ? error.message : error);
		// Fail-closed: protected route-এ কোনো সন্দেহ হলে ঢুকতে দেবে না
		return getRouteOwner(request.nextUrl.pathname) === null ? NextResponse.next() : redirectToLogin(request);
	}
}

export const config = {
	matcher: [
		/*
		 * এগুলোতে proxy চলবে না:
		 * api, _next/static, _next/image, metadata file, .well-known,
		 * আর public/ folder-এর static file (ছবি, font ইত্যাদি)
		 */
		"/((?!api|_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt|\\.well-known|.*\\.(?:svg|png|jpg|jpeg|gif|webp|avif|ico|css|js|woff2?|ttf|txt|xml)$).*)",
	],
};
```

এখন অংশে অংশে বুঝি 👇

---

## ধাপ ৬: Constant আর type

| জিনিস | মান | কেন |
|---|---|---|
| `REFRESH_BEFORE_EXPIRY_SECONDS` | ৫ মিনিট | মেয়াদ শেষ হওয়ার **আগেই** refresh — নাহলে page-এর মাঝখানে API call-এ token expire হতে পারে |
| `REFRESH_TIMEOUT_MS` | ৫ সেকেন্ড | backend আটকে গেলে প্রতিটা page load ঝুলে থাকবে না |
| `ROUTES` | ৫টা path | একই string বারবার না লিখে এক জায়গায় (typo কম) |

### `RefreshResult` — ৩টা অবস্থা কেন, ২টা না

| status | মানে | proxy কী করবে |
|---|---|---|
| `success` | নতুন token পাওয়া গেছে | নতুন cookie বসাবে |
| `invalid` | backend 401/403 → session শেষ / logout-all / blocked | cookie মুছে logout |
| `error` | network down, timeout, 500 | **কিছু মুছবে না** — পুরনো token valid থাকলে সেটা দিয়েই চলবে |

শুধু success/fail হলে backend ১ মিনিট down থাকলেই সব user logout হয়ে যেত। `error` আলাদা রাখায় সেটা হয় না।

---

## ধাপ ৭: `refreshAuthTokens` — backend থেকে নতুন token

- Backend-এর `POST /auth/refresh-token` (server note 17) **cookie থেকে** `refreshToken` আর `better-auth.session_token` পড়ে। Proxy server-এ চলছে, browser না — তাই নিজেই `Cookie` header বানিয়ে পাঠাতে হয়।
- `cache: "no-store"` → Next.js যেন এই fetch cache না করে (প্রতিবার নতুন token চাই)।
- `signal: AbortSignal.timeout(5000)` → ৫ সেকেন্ডে উত্তর না এলে বাতিল → `catch` → `error`।
- Backend response: `{ success, message, data: { accessToken, refreshToken, sessionToken } }` → `body.data` থেকে নেওয়া। তিনটার একটাও না থাকলে `error` (অসম্পূর্ণ token বসাব না)।
- `console.error` → eslint-এর `no-console`-এ সমস্যা নেই, আর proxy-র log terminal-এ (browser-এ না) দেখা যায়।

---

## ধাপ ৮: Helper — `getTokenCookies` আর `redirectToLogin`

**`getTokenCookies`** → ৩টা cookie-র `[নাম, মান, maxAge]` list। `as const` দিলে প্রতিটা item-এর type ঠিক থাকে, তাই loop-এ `const [name, value, maxAge]` লেখা যায়। Access/refresh-এর `maxAge` = token-এর নিজের বাকি মেয়াদ, session-এর fixed ৭ দিন (note 07-এর মতোই)।

**`redirectToLogin`** → `/login?redirect=/admin/dashboard?tab=x` বানায়। Login-এর পর `getPostLoginRedirect()` (note 07) এই `redirect` আবার check করে — নিজের সাইটের path আর এই role-এর অনুমতি আছে কিনা — তাই **open redirect safe**।

---

## ধাপ ৯: `handleAuth` — মূল সিদ্ধান্ত, ধাপে ধাপে

```
শুরু
 │
 ├─ token নেই + public page ──────────────────────────► next() ⚡ (কিছুই করে না)
 │
 ├─ ১) accessToken verify → user (বা null)
 ├─ ২) ৫ মিনিটের কম বাকি / token নেই + refresh আর session আছে → refresh
 ├─ ৩) status ACTIVE না / isDeleted → user = null
 │
 ├─ ৪) public page ──────────────────────────────────► next()
 ├─ ৫) user নেই:
 │      ├─ protected route ──────────────────────────► /login?redirect=...
 │      ├─ /reset-password কিন্তু ?email নেই ───────► /forgot-password
 │      └─ বাকি (login, register…) ──────────────────► next()
 │
 │   ── এখান থেকে user logged-in ──
 ├─ ৬) email verify হয়নি ─────────────────────────────► /verify-email?email=...
 ├─ ৭) needPasswordChange ────────────────────────────► /change-password
 ├─ ৮) auth page-এ এসেছে (login ইত্যাদি) ─────────────► নিজের dashboard
 ├─ ৯) এই role-এর route না ───────────────────────────► নিজের dashboard
 └─ সব ঠিক ───────────────────────────────────────────► next() ✅
```

### Fast path ⚡
কোনো token নেই আর public page (যেমন home) → verify, refresh কিছুই লাগে না। বেশিরভাগ visitor এই রাস্তায় যায়, তাই site দ্রুত থাকে।

### ১) Access token একবারই verify
`verifyAccessToken` signature + মেয়াদ দুটোই দেখে। ভুল/expired/নেই → `user = null`।

### ২) Refresh কখন
```ts
const secondsLeft = user?.exp ? user.exp - Math.floor(Date.now() / 1000) : 0;
```
- `user` null হলে (token নেই বা expired) `secondsLeft = 0` → refresh চেষ্টা।
- ৫ মিনিটের কম বাকি থাকলেও refresh।
- `exp` verify-এর payload থেকেই নেওয়া, তাই আবার decode করতে হয় না।
- Refresh-এর জন্য `refreshToken` **আর** `sessionToken` দুটোই লাগে (backend দুটোই চায়)।

ফল অনুযায়ী:
- `success` → নতুন access token আবার verify করে `user` বানানো।
- `invalid` → `user = null` (যেমন অন্য device থেকে logout-all করা হয়েছে)।
- `error` → কিছুই বদলায় না। পুরনো token এখনো valid থাকলে (৫ মিনিটের মধ্যে) সেটা দিয়েই চলবে।

### ৩) Blocked/deleted user
Token-এ `status` আর `isDeleted` আছে। `ACTIVE` না হলে বের করে দেওয়া হয়, আর `refreshStatus = "invalid"` দেওয়ায় cookie-ও মুছে যায়।

> মনে রাখো: token-এর তথ্য **token বানানোর সময়ের**। Admin এখন block করলে proxy সেটা জানবে পরের refresh-এর সময়। কিন্তু backend-এর `checkAuth` প্রতিটা API call-এ DB দেখে, তাই data সাথে সাথেই সুরক্ষিত।

### `shouldClearCookies` — বাসি cookie মোছা
```ts
const shouldClearCookies = !user && refreshStatus !== "error" && Boolean(accessToken || refreshToken);
```
তিনটা শর্ত একসাথে:
1. `!user` → valid user নেই
2. `refreshStatus !== "error"` → backend down হওয়ার কারণে না (তাহলে মুছলে সবাই অকারণে logout)
3. কোনো cookie আসলেই আছে → নাহলে মোছার কিছু নেই

না মুছলে প্রতিটা request-এ বাতিল refresh token দিয়ে বারবার backend call হত।

> 🔄 Update (note 28): `finalize`-এ browser-এর cookie মোছা এখন `response.cookies.delete(getAuthCookieDeleteOptions(name))` — production-এ cookie `.arifuddincoder.site` domain-এ থাকে, তাই মুছতেও একই domain লাগে। `next()`-এর `request.cookies.delete(name)` বদলায়নি (ওটা শুধু এই request-এর ভেতরে, browser-এ যায় না)।

### `finalize` বনাম `next` — cookie দুই জায়গায় কেন (গুরুত্বপূর্ণ ⭐)

| | কোথায় বসায় | কার জন্য |
|---|---|---|
| `finalize(response)` | `response.cookies.set(...)` | **browser** — পরের request থেকে নতুন cookie পাঠাবে |
| `next()` | `request.cookies.set(...)` + `NextResponse.next({ request: { headers } })` | **এই request-এরই** Server Component / Server Action |

সমস্যাটা: proxy refresh করে নতুন token পেল, কিন্তু শুধু response-এ বসালে এই page-এর Server Component এখনো browser-এর পাঠানো **পুরনো** (expired) cookie দেখবে → backend call 401। তাই request-এর cookie-ও বদলে, সেই বদলানো headers দিয়ে `NextResponse.next()` — এতে page নতুন token দেখে।

- Request cookie-তে শুধু নাম আর মান (`[name, value]`) — option লাগে না, কারণ এটা browser-এ যায় না।
- Redirect-এ শুধু `finalize`, কারণ redirect হলে এই request-এর page আর render হয় না; browser নতুন cookie নিয়ে নতুন URL-এ যাবে।

### ৪) Public page
সবার জন্য খোলা। তবে `next()` দিয়ে ফেরত, যাতে refresh হয়ে থাকলে নতুন cookie-ও চলে যায়।

### ৫) Logged-out user
- Protected route → `/login?redirect=...`। `finalize` দেওয়ায় বাসি cookie-ও মুছে যায়।
- `/reset-password` সরাসরি খুললে (`?email=` নেই) → `/forgot-password`, কারণ OTP পাঠানো email ছাড়া reset করা যায় না।
- login/register ইত্যাদি → ঢুকতে দাও।

### ৬) Email verify বাধ্যতামূলক
`emailVerified: false` হলে `/verify-email` ছাড়া কোথাও যেতে পারবে না। `encodeURIComponent` → email-এ `+` বা অন্য চিহ্ন থাকলেও URL ভাঙবে না।

### ৭) Password বদলানো বাধ্যতামূলক
Admin/doctor প্রথম login-এ (`needPasswordChange: true`) শুধু `/change-password`। `loginAction`-ও (note 07) এখানেই পাঠায় — proxy নিশ্চিত করে যে URL বদলেও পালানো যাবে না।

> Backend-এ password বদলানোর পর নতুন token-এ `needPasswordChange: false` আসে (server note 41)। Frontend-এর change-password action সেই নতুন token দিয়ে `setAuthCookies` না করলে proxy বারবার `/change-password`-এ ফেরত পাঠাবে।

### ৮) Logged-in user auth page-এ
Login করা অবস্থায় `/login` খুললে → নিজের dashboard। (Email verify না হলে ৬ নম্বরে আগেই `/verify-email`-এ চলে যায়।)

### ৯) Role check
`canAccessRoute` → public, `COMMON`, বা নিজের role-এর route হলে ✅। SUPER_ADMIN-কে ADMIN ধরা হয় (`toDashboardRole`)। না মিললে → নিজের dashboard (403 page না দেখিয়ে)।

---

## ধাপ ১০: `proxy()` export — Fail-closed

```ts
export async function proxy(request: NextRequest) {
	try {
		return await handleAuth(request);
	} catch (error) {
		// ...
		return getRouteOwner(request.nextUrl.pathname) === null ? NextResponse.next() : redirectToLogin(request);
	}
}
```

- Function-এর নাম **অবশ্যই `proxy`** (বা default export) — Next.js এই নামেই খোঁজে।
- অপ্রত্যাশিত error হলে:
  - public page → খুলতে দাও (site যেন পুরো বন্ধ না হয়)
  - protected page → login-এ (**fail-closed**: সন্দেহ হলে দরজা বন্ধ)
- `return await` → `await` না দিলে `handleAuth`-এর error এই `try/catch`-এ ধরা পড়ত না।

---

## ধাপ ১১: `config.matcher` — কোথায় proxy চলবে না

```ts
"/((?!api|_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt|\\.well-known|.*\\.(?:svg|png|jpg|jpeg|gif|webp|avif|ico|css|js|woff2?|ttf|txt|xml)$).*)"
```

- `(?!...)` = negative lookahead → "এগুলো দিয়ে শুরু **না** হলে" মিলবে।
- বাদ: `api`, Next.js-এর build file (`_next/static`), image optimization (`_next/image`), metadata file, `.well-known`, আর `public/`-এর ছবি/font/css/js।
- কেন: প্রতিটা ছবি বা CSS-এর জন্য token verify করা অকারণ সময় নষ্ট। Proxy শুধু **page** request-এর জন্য।
- Regex string-এ `\\.` → JavaScript string-এ `\\` মানে একটা `\`, তাই regex পায় `\.` (আসল dot)।

---

## 🧪 Test

`pnpm dev` চালিয়ে browser-এ (DevTools → Application → Cookies দেখতে দেখতে):

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | Logout অবস্থায় `/dashboard` | `/login?redirect=%2Fdashboard` |
| ২ | ওই URL থেকে patient login | সরাসরি `/dashboard` |
| ৩ | Patient হয়ে `/admin/dashboard` | `/dashboard`-এ ফেরত |
| ৪ | Login অবস্থায় `/login` | নিজের dashboard |
| ৫ | Logout অবস্থায় `/reset-password` (email ছাড়া) | `/forgot-password` |
| ৬ | নতুন admin/doctor (needPasswordChange) login → `/my-profile` | `/change-password` |
| ৭ | DevTools-এ `accessToken` cookie মুছে page reload | refresh হয়ে নতুন `accessToken` cookie, page খোলে |
| ৮ | `accessToken`-এর মান একটু বদলে দাও (জাল token) | verify fail → refresh → নতুন token (refresh token valid হলে) |
| ৯ | তিনটা cookie-ই মুছে protected page | `/login` |
| ১০ | Backend বন্ধ রেখে (token valid অবস্থায়) public page | খোলে, cookie মোছে না |
| ১১ | `/` বা ছবি load | Terminal-এ proxy-র কোনো কাজ/log নেই |

> Test ৭-এর জন্য backend চালু রাখো। চাইলে proxy-তে সাময়িক `console.warn("[proxy]", pathname)` দিয়ে দেখতে পারো কোন request-এ proxy চলছে (পরে মুছে দিও)।

---

## 📊 সারাংশ

| কাজ | কোথায় |
|---|---|
| Secret | `.env`-এ `ACCESS_TOKEN_SECRET` (`NEXT_PUBLIC_` ছাড়া) |
| Cookie নাম/option | `cookieConfig.ts` (proxy + action দুজনেই) |
| Token verify | `jwtUtils.ts` → `verifyAccessToken` (HS256 pin) |
| Route কার | `authUtils.ts` |
| সিদ্ধান্ত + refresh | `proxy.ts` |
| আসল নিরাপত্তা | backend `checkAuth` |

---

## 🔜 পরে যা যোগ হবে
- [ ] Change password page + server action (নতুন token দিয়ে `setAuthCookies`, নাহলে proxy `/change-password`-এ আটকে রাখবে)
- [ ] Verify email / forgot / reset password page-এর action
- [x] Logout action (`clearAuthCookies` + backend `/auth/logout`) — note 11
- [x] `httpClient`-এ cookie থেকে token পড়ে backend-এ পাঠানো — note 15
- [ ] `getUserInfo` (`/auth/me`) দিয়ে dashboard-এ user-এর তথ্য
- [ ] Role অনুযায়ী sidebar menu (`authUtils` থেকে)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Secret না থাকলে পুরো site বন্ধ হয়ে যাবে** — `jwtUtils.ts`-এর উপরের `throw` file import হওয়ার সময়েই চলে, `proxy()`-এর `try/catch`-এর বাইরে। তাই `ACCESS_TOKEN_SECRET` ভুলে গেলে (যেমন Vercel-এ deploy করার সময় env দিতে ভুলে গেলে) public page সহ **সব page** error দেবে। Local-এ এটা ভালো (সাথে সাথে বোঝা যায়)। শুধু মনে রাখো: deploy করার সময় env variable list-এ এটা দিতে হবে।

2. **একসাথে অনেক request-এ একাধিক refresh** — একটা page খুললে Next.js `<Link>` prefetch সহ কয়েকটা request একসাথে যায়। Token expire হলে প্রতিটাই আলাদা করে refresh call করতে পারে। এখনকার backend পুরনো refresh token বাতিল করে না (server note 17-এর recommendation ৪), তাই সবগুলোই সফল হবে — সমস্যা নেই। কিন্তু পরে backend-এ "refresh token rotation" (পুরনোটা বাতিল) যোগ করলে এখানে সমস্যা হবে — তখন এটা মাথায় রেখো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Matcher-এ `api` সব `/api...` path বাদ দেয়** — `(?!api)` `/api-docs` বা `/apiary`-র মতো page-কেও বাদ দেবে। এখন এমন page নেই। কখনো বানালে `api/` লিখো:
   ```ts
   "/((?!api/|_next/static|...).*)"
   ```

4. **Access token ১ দিন, session ৭ দিন** — backend-এ `ACCESS_TOKEN_EXPIRES_IN=1d`। Proxy ৫ মিনিট আগে refresh করে, তাই user ৭ দিন পর্যন্ত logged-in থাকবে। Access token ছোট (যেমন `15m`) করলে block/role বদল proxy আরো তাড়াতাড়ি জানবে। এই proxy code কিছু না বদলেই সেটা সামলাতে পারবে।

5. **Proxy ভারী কাজের জায়গা না** — এখানে শুধু token verify + দরকার হলে একটা refresh call। DB query বা `/auth/me` call এখানে দিও না — প্রতিটা page navigation ধীর হয়ে যাবে। User-এর পুরো তথ্য page/Server Component-এ নাও।
