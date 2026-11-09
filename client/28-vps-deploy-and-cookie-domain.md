# 28 - নিজের VPS-এ Deploy (Next.js + pm2 + nginx) ও Google OAuth Cookie সমস্যার সমাধান

> Frontend (`phdoc-client`) একই VPS-এ deploy, যেখানে backend (`phdocapi`) আগে থেকেই চলছে।
> Deploy-এর পর দুটো সমস্যা: (১) Google login-এর পর frontend user-কে logged-in দেখায় না, (২) সেটা ঠিক করার পর logout কাজ করে না। দুটোরই সমাধান **cookie-তে domain বসানো** (backend + frontend দুই জায়গায়)।

```
Browser ─► Cloudflare ─► nginx (443) ─┬─ phdoc.arifuddincoder.site    → localhost:3000 (Next.js, pm2: phdoc-client)
                                      └─ phdocapi.arifuddincoder.site → localhost:5000 (Express, pm2)
Cookie domain = .arifuddincoder.site  → phdoc আর phdocapi দুজনেই একই cookie দেখে
```

---

# পর্ব ১: Deploy

## ধাপ ১: DNS

Domain provider-এ (এখানে Cloudflare) একটা **A record**:

```
phdoc  →  <VPS এর IP>   (backend এর phdocapi যে IP তে, সেই একই IP)
```

- একই VPS, তাই একই IP; কোন subdomain কোন app-এ যাবে সেটা nginx ঠিক করে (`server_name` দেখে)।

## ধাপ ২: Code আনা

VPS-এ ঢুকে:

```shell
cd ~/apps
git clone <phdoc-client repo URL> phdoc-client
cd phdoc-client
```

## ধাপ ৩: `.env` তৈরি

```dotenv
NEXT_PUBLIC_API_BASE_URL=https://phdocapi.arifuddincoder.site/api/v1
ACCESS_TOKEN_SECRET=<backend এর মানটাই হুবহু>
COOKIE_DOMAIN=.arifuddincoder.site
```

> ✏️ `COOKIE_DOMAIN` লাইনটা নোটের শেষে (পর্ব ২) ছিল — এখানে একসাথে রাখা হলো, যাতে প্রথম build-এর আগেই থাকে।

⚠️ **কোন variable কখন পড়া হয়:**

| Variable | কখন পড়া হয় | বদলালে কী করতে হবে |
|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | **build-এর সময়** code-এর ভেতরে বসে যায় | `pnpm build` আবার + restart |
| `ACCESS_TOKEN_SECRET` | server-এ চলার সময় (proxy-তে JWT verify) | শুধু restart |
| `COOKIE_DOMAIN` | server-এ চলার সময় (proxy, server action) | শুধু restart |

- তাই `.env` আগে ঠিক করো, তারপর build।
- `ACCESS_TOKEN_SECRET` backend-এর সাথে না মিললে proxy প্রতিটা token-কে ভুয়া ভাববে → সবাই logout।

## ধাপ ৪: Install আর build

```shell
node -v          # Next 16 এর জন্য Node 20.9 বা তার বেশি লাগে
pnpm install
pnpm build
```

- RAM কম হলে (১-২ GB) build-এর মাঝে **"Killed"** দেখিয়ে বন্ধ হয়ে যেতে পারে (memory শেষ)। তখন একটা swap file যোগ করে নাও (নিচে Recommendation-এ command)।

## ধাপ ৫: pm2 দিয়ে চালানো

Backend 5000 port-এ চলছে, তাই frontend চলবে 3000-এ:

```shell
pm2 start node_modules/next/dist/bin/next --name phdoc-client -- start -p 3000

# check
pm2 status
pm2 logs phdoc-client --lines 20
```

- `node_modules/next/dist/bin/next` = `next` CLI-এর আসল file; `--` এর পরের অংশ (`start -p 3000`) next-কে দেওয়া হয়।
- `--name phdoc-client` → পরে `pm2 restart phdoc-client` এভাবে নাম দিয়ে চালানো যায়।
- pm2 → crash হলে নিজে আবার চালু করে, terminal বন্ধ করলেও চলতে থাকে।

## ধাপ ৬: nginx — `/etc/nginx/sites-available/phdoc-client`

```nginx
server {
    listen 443 ssl;
    server_name phdoc.arifuddincoder.site;
    include /etc/nginx/snippets/cloudflare-arifuddincoder.site.conf;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}

server {
    listen 80;
    server_name phdoc.arifuddincoder.site;
    return 301 https://$host$request_uri;
}
```

Backend-এর config থেকে শুধু দুটো জিনিস বদলেছে: **`server_name`** (দুই জায়গাতেই) আর **`proxy_pass`-এর port** (5000 → 3000)।

### ভেঙে বোঝা

| লাইন | কাজ |
|---|---|
| `listen 443 ssl` + `include ...cloudflare...conf` | HTTPS; snippet-এ Cloudflare-এর SSL certificate (একই domain-এর সব subdomain-এ চলে) |
| `proxy_pass http://localhost:3000` | বাইরের request ভেতরের Next.js-এ পাঠানো |
| `Upgrade` / `Connection 'upgrade'` | WebSocket চালু রাখা (dev-এ HMR, পরে realtime লাগলে) |
| `Host $host` | Next.js আসল domain জানে (redirect URL ঠিক হয়) |
| `X-Real-IP`, `X-Forwarded-For` | user-এর আসল IP (নইলে সব request 127.0.0.1 থেকে মনে হয়) |
| `X-Forwarded-Proto $scheme` | Next.js জানে request আসলে HTTPS ছিল |
| দ্বিতীয় `server` (port 80) | `http://` এলে `https://`-এ পাঠায় (301 = স্থায়ী) |

## ধাপ ৭: Site চালু করা

```shell
sudo ln -s /etc/nginx/sites-available/phdoc-client /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# পরে env update করলে
pm2 reload phdoc-client --update-env
```

> ✏️ নোটে ছিল `pm2 reload myapp --update-env` — এখানে app-এর নাম `phdoc-client` (ধাপ ৫-এর `--name`)।

- `ln -s` → `sites-available`-এ config, `sites-enabled`-এ শুধু link; বন্ধ করতে link মুছলেই হয়।
- `nginx -t` → config-এ ভুল আছে কিনা আগে check; ঠিক থাকলেই `reload` (`&&`)। ভুল config-এ reload দিলে সব site বন্ধ হতে পারত।
- `NEXT_PUBLIC_*` বদলালে reload যথেষ্ট না — আগে `pnpm build`।

---

# পর্ব ২: Google OAuth-এর পর cookie set না হওয়া (আর logout সমস্যা)

## সমস্যা কেন হলো

Cookie যে host set করে, `domain` না দিলে cookie **শুধু সেই host-এর** (host-only cookie)।

| Login-এর ধরন | কে cookie set করে | cookie কোন host-এর | frontend দেখে? |
|---|---|---|---|
| Email/password | Next.js server action (`phdoc`) | `phdoc.arifuddincoder.site` | ✅ |
| Google OAuth | Backend callback (`phdocapi`), তারপর frontend-এ redirect | `phdocapi.arifuddincoder.site` | ❌ |

- Local-এ সমস্যা ধরা পড়েনি: `localhost:3000` আর `localhost:5000` — cookie port মানে না, দুজনেই একই host `localhost`।
- Production-এ আলাদা subdomain → Google login-এর cookie `phdoc`-এর proxy/server component দেখতেই পায় না → user logged-out মনে হয়।

## সমাধান: cookie-তে domain বসানো

`.arifuddincoder.site` দিলে (শুরুতে dot সহ) cookie টা `phdocapi` আর `phdoc` দুটো subdomain-ই পাবে।
Local-এ `localhost`-এর জন্য domain দেওয়া যায় না, তাই মানটা `.env` থেকে নেওয়া হবে — **শুধু production-এর `.env`-এ এটা থাকবে**।

### Backend ১: `src/app/config/env.ts`

`interface EnvConfig`-এর ভেতরে, `FRONTEND_URL`-এর নিচে:

```ts
	FRONTEND_URL: string;
	COOKIE_DOMAIN?: string; // শুধু production এ; local এ দরকার নেই
```

return object-এ:

```ts
	FRONTEND_URL: getEnv("FRONTEND_URL"),
	COOKIE_DOMAIN: process.env.COOKIE_DOMAIN,
```

- `getEnv` না, সরাসরি `process.env` — কারণ `getEnv` মান না পেলে error throw করে, আর local `.env`-এ এটা থাকবে না।

### Backend ২: `src/app/utils/token.ts` — শুধু `baseCookieOptions` বদলাবে

```ts
// set আর clear দুই জায়গাতেই একই options লাগে
const baseCookieOptions: CookieOptions = {
	httpOnly: true,
	secure: isProduction,
	sameSite: isProduction ? "none" : "lax",
	path: "/",
	// ".arifuddincoder.site" দিলে phdoc আর phdocapi দুই subdomain-ই cookie পায়
	// local-এ undefined থাকে, তখন cookie শুধু localhost-এর জন্য বসে
	...(envVars.COOKIE_DOMAIN && { domain: envVars.COOKIE_DOMAIN }),
};
```

- `...(condition && { domain })` → মান থাকলে `domain` যোগ হয়, না থাকলে `...false` = কিছুই না (spread-এ `false` চুপচাপ বাদ যায়)।
- একই object set আর clear দুই জায়গায় → logout-এও একই domain দিয়ে মোছে।

### Backend ৩: server-এর production `.env`

```dotenv
NODE_ENV=production
FRONTEND_URL=https://phdoc.arifuddincoder.site
COOKIE_DOMAIN=.arifuddincoder.site
```

---

## এরপর নতুন সমস্যা: logout কাজ করে না

এখন backend cookie বসায় `.arifuddincoder.site` domain-এ, কিন্তু frontend-এর `clearAuthCookies` (logout) আর proxy মুছত **domain ছাড়া** (host-only `phdoc`)। Browser-এর চোখে এরা **আলাদা cookie** — তাই domain-ওয়ালা cookie থেকেই যায়, user logout হয় না।

> নিয়ম: cookie মুছতে হলে set করার সময়ের **একই `name` + `path` + `domain`** দিতে হয়।

## সমাধান: frontend-এর cookie option-এও একই domain

### Frontend ১: `src/lib/cookieConfig.ts` (পুরো ফাইল)

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

- **`cookieScope`** = set আর delete দুজনের common অংশ (`path` + `domain`) এক জায়গায় → দুটো কখনো আলাদা হবে না।
- **`getAuthCookieDeleteOptions`** নতুন — Next.js-এর `cookies().delete()` আর `response.cookies.delete()` দুটোই `{ name, path, domain }` object নেয়।
- `NEXT_PUBLIC_` নেই → browser bundle-এ যায় না; build-এ বসে না, চলার সময় পড়া হয় (restart-এই বদলায়)।
- এখানে `sameSite: "lax"` — `phdoc` আর `phdocapi` একই site (`arifuddincoder.site`), তাই lax-এ কোনো সমস্যা নেই।

### Frontend ২: `src/lib/cookieUtils.ts`

```ts
// src/lib/cookieUtils.ts
import { cookies } from "next/headers";
import "server-only";
import { AUTH_COOKIES, type AuthCookieName, getAuthCookieDeleteOptions, getAuthCookieOptions } from "./cookieConfig";

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
		cookieStore.delete(getAuthCookieDeleteOptions(name));
	}
};
```

- বদল শুধু `clearAuthCookies`-এ: আগে `cookieStore.delete(name)`, এখন `delete({ name, path, domain })`।

### Frontend ৩: `src/proxy.ts`

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

**note 08 থেকে বদল শুধু দুই জায়গায়:**

| জায়গা | আগে | এখন |
|---|---|---|
| import | `AUTH_COOKIES, getAuthCookieOptions, SESSION_TOKEN_MAX_AGE` | + `getAuthCookieDeleteOptions` |
| `finalize`-এ cookie মোছা | `response.cookies.delete(name)` | `response.cookies.delete(getAuthCookieDeleteOptions(name))` |

- `next()`-এর ভেতরের `request.cookies.delete(name)` বদলায়নি — ওটা শুধু এই request-এর server component-এর জন্য (browser-এ যায় না), তাই domain লাগে না।
- নতুন token set (`getAuthCookieOptions`) এমনিতেই `cookieScope` পায় → refresh হওয়া cookie-ও domain-ওয়ালা।

### Frontend ৪: production `.env`

```dotenv
COOKIE_DOMAIN=.arifuddincoder.site
```

(ধাপ ৩-এ বাকি variable-গুলোর সাথে একসাথে।) বদলানোর পর: `pm2 reload phdoc-client --update-env`।

---

## 📊 এখন cookie কোথায় কী

| | Backend (`token.ts`) | Frontend (`cookieConfig.ts`) |
|---|---|---|
| domain (production) | `.arifuddincoder.site` | `.arifuddincoder.site` |
| domain (local) | নেই (host-only `localhost`) | নেই |
| `sameSite` (production) | `none` | `lax` |
| `secure` (production) | `true` | `true` |
| মোছার সময় | একই `baseCookieOptions` | `getAuthCookieDeleteOptions` |

---

## 🧪 Test (production-এ)

| কী করবে | ফল |
|---|---|
| `https://phdoc.arifuddincoder.site` | site খোলে, `http://` দিলে `https://`-এ যায় |
| `pm2 status` | `phdoc-client` online |
| Email দিয়ে login | dashboard; DevTools → Application → Cookies-এ domain `.arifuddincoder.site` |
| Google দিয়ে login | frontend-এ logged-in (avatar দেখায়) |
| Logout (দুই ধরনেই) | তিনটা cookie মুছে যায়, protected page-এ গেলে login-এ পাঠায় |
| Local-এ (`pnpm dev`) | আগের মতোই কাজ করে (domain ছাড়া) |

> পুরনো host-only cookie (domain-এর আগে বসানো) browser-এ থেকে গেলে অদ্ভুত আচরণ হতে পারে — test-এর আগে একবার site-এর সব cookie মুছে নাও।

---

## ✏️ যেসব note এই পরিবর্তনে update হয়েছে

- server [06 envVars](../server/06-envVars.md) — `COOKIE_DOMAIN`
- server [13 jwt token cookie](../server/13-jwt-token-cookie-and-check-auth.md) — `baseCookieOptions`-এ `domain`
- server [19 logout](../server/19-logout.md) — clear-এও একই domain
- server [22 google oauth](../server/22-google-oauth.md) — production-এর cookie সমস্যা
- client [07 login](./07-login.md) — `cookieConfig.ts`, `cookieUtils.ts`
- client [08 proxy](./08-proxy-authorization-route-protection.md) — `getAuthCookieDeleteOptions`
- client [11 logout](./11-logout-and-user-dropdown.md) — `clearAuthCookies`

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Project-এর backend `env.ts`-এ `getEnv("COOKIE_DOMAIN")` আছে** — নোটে ঠিকভাবে `process.env.COOKIE_DOMAIN` লেখা, কিন্তু project file-এ (`phdoc-server/src/app/config/env.ts`, line ~100) `getEnv` দেওয়া। `getEnv` মান না পেলে error throw করে, আর local `.env`-এ `COOKIE_DOMAIN` নেই → **local-এ server চালু হবে না** (production-এ চলবে, কারণ সেখানে আছে)। নোটের মতো করে দাও:
   ```ts
   COOKIE_DOMAIN: process.env.COOKIE_DOMAIN,
   ```

### 🟡 ভালো practice

2. **Reboot-এর পর pm2 নিজে চালু হওয়া** — নইলে VPS restart হলে site বন্ধ থাকবে:
   ```shell
   pm2 save          # এখনকার চলমান app-এর তালিকা মনে রাখে
   pm2 startup       # যে command দেখায় সেটা sudo দিয়ে একবার চালাও
   ```

3. **Swap file (RAM কম হলে build-এর জন্য)**:
   ```shell
   sudo fallocate -l 2G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # reboot-এর পরও থাকবে
   ```

4. **Better Auth-এর নিজের cookie-তেও domain** — `auth.ts`-এর `betterAuthCookieAttributes`-এ `domain` নেই। Google login-এর সময় Better Auth নিজেও `better-auth.session_token` বসায় (`phdocapi` host-only), তারপর `tokenUtils` domain-ওয়ালাটা বসায়। একই নামের দুটো cookie থাকলে logout-এ একটা থেকে যেতে পারে। test করে সমস্যা দেখলে একই spread যোগ করো (untested):
   ```ts
   const betterAuthCookieAttributes = {
   	httpOnly: true,
   	secure: isProduction,
   	sameSite: isProduction ? "none" : "lax",
   	path: "/",
   	...(envVars.COOKIE_DOMAIN && { domain: envVars.COOKIE_DOMAIN }),
   } as const;
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **`sameSite: "none"` এখন আর জরুরি না** — `phdoc` আর `phdocapi` একই "site" (`arifuddincoder.site`), তাই browser-এর চোখে এদের মধ্যে request same-site; backend-এও `"lax"` দিলে চলত, আর CSRF থেকে বেশি সুরক্ষিত। frontend আর backend কখনো সম্পূর্ণ আলাদা domain-এ গেলে (যেমন `vercel.app` + নিজের domain) তখনই `"none"` লাগবে।

6. **Deploy সহজ করা** — প্রতিবার একই command; পরে একটা script বা GitHub Actions:
   ```shell
   cd ~/apps/phdoc-client && git pull && pnpm install && pnpm build && pm2 reload phdoc-client
   ```

7. **Cloudflare SSL mode** — origin certificate ব্যবহার করলে Cloudflare dashboard-এ SSL/TLS mode **Full (strict)** রাখো; "Flexible" দিলে redirect loop হতে পারে।

---

## 🔜 পরে যা যোগ হবে

- [ ] GitHub Actions দিয়ে auto deploy
- [ ] `pm2 startup` + `pm2 save`
- [ ] Stripe webhook URL production domain-এ
