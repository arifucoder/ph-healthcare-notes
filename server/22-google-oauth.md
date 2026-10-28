# 22 - Google OAuth (Social Login)

> Module: 39-09

```
Browser: GET /api/v1/auth/login/google?redirect=/profile → googleRedirect.ejs page
→ fetch POST /api/auth/sign-in/social → Google consent screen → user email বাছাই
→ /api/auth/callback/google (Better Auth: user + session বানায়) → /api/v1/auth/google/success
→ blocked/deleted check → Patient row → ৩টা cookie → redirect FRONTEND_URL + /profile
```

Better Auth documentation-এর পদ্ধতি (frontend থেকে `authClient.signIn.social`) হুবহু follow করছি না। আমরা নিজেদের `/api/v1/auth` route system-এ custom ভাবে করছি, যাতে login শেষে নিজেদের ৩টা cookie (`accessToken`, `refreshToken`, `better-auth.session_token`) set করা যায়।

---

## ধাপ ১: Google Cloud-এ OAuth client বানানো

1. Google-এ **"google cloud"** লিখে search করো → Google Cloud-এ গিয়ে login করো।
2. **Console** → **APIs & Services** → **OAuth consent screen** → **Clients**।
3. **Create client** → Application type: **Web application** → একটা নাম দাও।
4. **Authorized JavaScript origins** (backend URL):

	```
	http://localhost:5000
	```

5. **Authorized redirect URIs**: Better Auth docs-এ `http://localhost:3000/api/auth/callback/google` দেওয়া আছে। ওদের backend 3000 port-এ চলে, আমাদের project 5000-এ, তাই:

	```
	http://localhost:5000/api/auth/callback/google
	```

6. **Create** click করলে **Client ID** আর **Client secret** পাবে — দুটোই `.env`-এ লাগবে।

> Consent screen "Testing" mode-এ থাকলে শুধু **Test users**-এ যোগ করা Gmail দিয়েই login করা যাবে।

---

## ধাপ ২: `.env`

```dotenv
## google oAuth
GOOGLE_CLIENT_ID=google_client_id
GOOGLE_CLIENT_SECRET=google_client_secret
GOOGLE_CALLBACK_URL=http://localhost:5000/api/auth/callback/google
FRONTEND_URL=http://localhost:3000
```

---

## ধাপ ৩: `src/app/config/env.ts`

> ✏️ নোটে শুধু "env.ts-এ যোগ করব" লেখা ছিল — code আগের নিয়মে লিখে দেওয়া হলো।

```ts
interface EnvConfig {
	// ...আগের সব
	FRONTEND_URL: string;
	GOOGLE_CLIENT_ID: string;
	GOOGLE_CLIENT_SECRET: string;
	GOOGLE_CALLBACK_URL: string;
}
```

Required list-এ:

```ts
"GOOGLE_CLIENT_ID",
"GOOGLE_CLIENT_SECRET",
"GOOGLE_CALLBACK_URL",
```

Return object-এ:

```ts
GOOGLE_CLIENT_ID: process.env.GOOGLE_CLIENT_ID as string,
GOOGLE_CLIENT_SECRET: process.env.GOOGLE_CLIENT_SECRET as string,
GOOGLE_CALLBACK_URL: process.env.GOOGLE_CALLBACK_URL as string,
```

---

## ধাপ ৪: `src/app/lib/auth.ts`

```ts
const isProduction = envVars.NODE_ENV === "production";

// Better Auth-এর নিজের cookie-র options (token.ts-এর baseCookieOptions-এর মতো)
const betterAuthCookieAttributes = {
	httpOnly: true,
	secure: isProduction,
	sameSite: isProduction ? "none" : "lax",
	path: "/",
} as const;

export const auth = betterAuth({
	// একদম উপরে
	baseURL: envVars.BETTER_AUTH_URL,
	secret: envVars.BETTER_AUTH_SECRET,
	trustedOrigins: [envVars.FRONTEND_URL],

	// ...database, emailAndPassword

	// emailAndPassword-এর নিচে
	socialProviders: {
		google: {
			clientId: envVars.GOOGLE_CLIENT_ID,
			clientSecret: envVars.GOOGLE_CLIENT_SECRET,
			// ✅ .env থেকে callback URL — port/domain বদলালে শুধু .env বদলালেই হবে
			redirectURI: envVars.GOOGLE_CALLBACK_URL,
			// শুধু নতুন user তৈরির সময় চলে (overrideUserInfoOnSignIn: false, default)
			mapProfileToUser: (profile) => {
				return {
					emailVerified: profile.email_verified,
					role: Role.PATIENT,
					status: UserStatus.ACTIVE,
					needPasswordChange: false,
					isDeleted: false,
					deletedAt: null,
				};
			},
		},
	},

	advanced: {
		useSecureCookies: false,
		defaultCookieAttributes: betterAuthCookieAttributes,
	},

	// ...বাকি code
});
```

| Option | মানে |
|---|---|
| `baseURL` | Better Auth কোথায় চলছে (`http://localhost:5000`) — callback URL এখান থেকে বানায় |
| `trustedOrigins` | frontend (`3000`) থেকে আসা request আর redirect-কে বিশ্বাস করবে |
| `redirectURI` | Google login শেষে কোথায় ফেরত আসবে — `.env`-এর `GOOGLE_CALLBACK_URL`; Google Console-এর Authorized redirect URI-র সাথে হুবহু মিলতে হবে |
| `mapProfileToUser` | Google profile থেকে নতুন user বানানোর সময় আমাদের extra field গুলো বসায় — Google দিয়ে আসা সবাই `PATIENT` |
| `useSecureCookies: false` | cookie নামের আগে `__Secure-` বসাবে না, তাই নাম সবসময় `better-auth.session_token` থাকে (আমাদের code এই নামেই পড়ে) |
| `defaultCookieAttributes` | Better Auth-এর cookie dev-এ `lax`, production-এ `none` + `secure` |

> ✏️ `Role`, `UserStatus` import (`@/generated/prisma/enums`) লাগবে — project-এ আগেই আছে।
> ✏️ `redirectURI: envVars.GOOGLE_CALLBACK_URL` যোগ করা হয়েছে (Claude Recommendation থেকে)। আগে না দেওয়ায় Better Auth default (`BETTER_AUTH_URL + "/api/auth/callback/google"`) ব্যবহার করত আর `GOOGLE_CALLBACK_URL` env অকারণে পড়ে থাকত।

> **📌 পরে দেখব: `ph-tour-notes` থেকে `providerId`-এর মতো করে (social provider / `account` table-এর `providerId`) এখানে কীভাবে করা হয়েছিল, সেটা মিলিয়ে দেখব।**

---

## ধাপ ৫: `src/app.ts` — Better Auth handler mount

Google থেকে ফেরত আসা `/api/auth/callback/google` Better Auth নিজেই handle করে। তাই handler mount করতেই হবে, নাহলে ওই URL-এ `404` আসবে।

`const app = express();`-এর **ঠিক পরেই**, `express.json()` আর `/api/v1` route-এর আগে বসাও:

```ts
import path from "node:path";
import { toNodeHandler } from "better-auth/node";
import { auth } from "./app/lib/auth";

// ejs page render (googleRedirect) — email.ts-এর মতো একই folder
app.set("view engine", "ejs");
app.set(
	"views",
	envVars.NODE_ENV === "production"
		? path.join(process.cwd(), "dist/templates")
		: path.join(process.cwd(), "src/app/templates"),
);

app.all("/api/auth/{*any}", toNodeHandler(auth));
```

- `/api/auth/*` → Better Auth-এর নিজের route (`sign-in/social`, `callback/google`)। আমাদের route `/api/v1/auth/*` — দুটো আলাদা।
- `express.json()`-এর আগে কারণ Better Auth নিজেই body পড়ে; আগে parse হয়ে গেলে request আটকে যেতে পারে।
- `{*any}` হলো Express 5-এর wildcard syntax।

> ✏️ `path` import নোটে ছিল না।

---

## ধাপ ৬: `src/app/templates/googleRedirect.ejs`

> ✏️ এই template নোটে ছিল না — project থেকে নেওয়া।

**কেন এই page লাগে?** Better Auth-এর `sign-in/social` হলো `POST` route, আর সে Google-এর URL JSON-এ ফেরত দেয়। Browser-এ link click করলে হয় `GET`। তাই এই ছোট page browser-এ খুলে নিজেই `POST` করে, তারপর Google-এ পাঠিয়ে দেয়। Same origin (`5000`) থেকে যায় বলে Better Auth-এর state cookie-ও ঠিকমতো set হয়।

```ejs
<!DOCTYPE html>
<html lang="en">
	<head>
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
		<title>Google Login</title>
	</head>
	<body>
		<div>
			<p id="message">Redirecting To Google...</p>
		</div>

		<script>
			(async () => {
				const messageEl = document.getElementById("message");

				const showError = (detail) => {
					messageEl.textContent =
						"Error Occurred While Redirecting To Google. Please Try Again Later." + (detail ? " " + detail : "");
				};

				try {
					const response = await fetch("<%= betterAuthUrl %>/api/auth/sign-in/social", {
						method: "POST",
						headers: {
							"Content-Type": "application/json",
						},
						credentials: "include",
						body: JSON.stringify({
							provider: "google",
							callbackURL: "<%= callbackURL %>",
							errorCallbackURL: "<%= errorCallbackURL %>",
						}),
					});

					if (!response.ok) {
						showError("(" + response.status + ")");
						return;
					}

					const data = await response.json();

					if (data.url) {
						window.location.href = data.url;
					} else {
						showError();
					}
				} catch (error) {
					showError(error.message);
				}
			})();
		</script>
	</body>
</html>
```

| Field | কাজ |
|---|---|
| `callbackURL` | Google login সফল হলে Better Auth এখানে পাঠাবে → আমাদের `/google/success` |
| `errorCallbackURL` | fail হলে (user cancel করলে ইত্যাদি) → আমাদের `/oauth/error` |

---

## ধাপ ৭: `auth.service.ts`

> ✏️ `TAuthSession` type নোটে ছিল না — এটা লাগবে:

```ts
// getSession-এর return type, null বাদ দিয়ে
type TAuthSession = NonNullable<Awaited<ReturnType<typeof auth.api.getSession>>>;
```

```ts
const googleLoginSuccess = async (session: TAuthSession) => {
	const { user } = session;

	// শুধু patient-এর জন্য Patient row (doctor/admin Google দিয়ে login করলে না)
	if (user.role === Role.PATIENT) {
		await prisma.patient.upsert({
			where: { userId: user.id },
			update: {},
			create: {
				userId: user.id,
				name: user.name,
				email: user.email,
			},
		});
	}

	return generateAuthTokens(user);
};
```

- `upsert` + `update: {}` → প্রথমবার login-এ Patient row বানায়, পরের বার কিছু করে না।

> ✏️ `Role` import আর `AuthService` object-এ `googleLoginSuccess` export — এটা নোটে ছিল না।

---

## ধাপ ৮: `auth.controller.ts`

```ts
const DEFAULT_REDIRECT = "/dashboard";

// শুধু "/profile" ধরনের path চলবে — "//evil.com" বা array না
const getSafeRedirectPath = (value: unknown) => {
	if (typeof value !== "string") return DEFAULT_REDIRECT;

	const isValid = value.startsWith("/") && !value.startsWith("//");
	return isValid ? value : DEFAULT_REDIRECT;
};

// GET /api/v1/auth/login/google?redirect=/profile
const googleLogin = catchAsync((req: Request, res: Response) => {
	const redirectPath = getSafeRedirectPath(req.query.redirect);

	const callbackURL = `${envVars.BETTER_AUTH_URL}/api/v1/auth/google/success?redirect=${encodeURIComponent(redirectPath)}`;
	const errorCallbackURL = `${envVars.BETTER_AUTH_URL}/api/v1/auth/oauth/error`;

	res.render("googleRedirect", {
		betterAuthUrl: envVars.BETTER_AUTH_URL,
		callbackURL,
		errorCallbackURL,
	});
});

const googleLoginSuccess = catchAsync(async (req: Request, res: Response) => {
	const redirectPath = getSafeRedirectPath(req.query.redirect);
	const loginUrl = `${envVars.FRONTEND_URL}/login`;

	// browser-এর পাঠানো cookie হুবহু Better Auth-কে দেওয়া (signed cookie সহ)
	const session = await auth.api.getSession({
		headers: fromNodeHeaders(req.headers),
	});

	if (!session) {
		return res.redirect(`${loginUrl}?error=no_session_found`);
	}

	const { user } = session;

	// blocked/deleted হলে Better Auth-এর বানানো session মুছে login page-এ ফেরত পাঠানো
	if (user.isDeleted || user.status === UserStatus.DELETED || user.status === UserStatus.BLOCKED) {
		await auth.api.revokeSession({
			body: { token: session.session.token },
			headers: fromNodeHeaders(req.headers),
		});
		tokenUtils.clearAuthCookies(res);

		const error = user.status === UserStatus.BLOCKED ? "account_blocked" : "account_deleted";
		return res.redirect(`${loginUrl}?error=${error}`);
	}

	const { accessToken, refreshToken } = await AuthService.googleLoginSuccess(session);

	tokenUtils.setAccessTokenCookie(res, accessToken);
	tokenUtils.setRefreshTokenCookie(res, refreshToken);
	// Better Auth-এর signed cookie ("token.signature")-এর বদলে raw token, যাতে checkAuth DB-তে খুঁজে পায়
	tokenUtils.setBetterAuthSessionCookie(res, session.session.token);

	res.redirect(`${envVars.FRONTEND_URL}${redirectPath}`);
});

const handleOAuthError = catchAsync((req: Request, res: Response) => {
	const error = typeof req.query.error === "string" ? req.query.error : "oauth_failed";

	res.redirect(`${envVars.FRONTEND_URL}/login?error=${encodeURIComponent(error)}`);
});
```

> ✏️ Import আর export নোটে ছিল না:

```ts
import { envVars } from "@/app/config/env";
import { auth } from "@/app/lib/auth";
import { UserStatus } from "@/generated/prisma/enums";
import { fromNodeHeaders } from "better-auth/node";

export const AuthController = {
	// ...আগেরগুলো
	googleLogin,
	googleLoginSuccess,
	handleOAuthError,
};
```

| জিনিস | কেন |
|---|---|
| `getSafeRedirectPath` | `?redirect=//evil.com` দিয়ে user-কে অন্য site-এ পাঠানো (open redirect) আটকায় |
| `fromNodeHeaders(req.headers)` | Express-এর header → Better Auth-এর `Headers`; signed session cookie সহ |
| Raw token cookie | Better Auth cookie-তে `token.signature` রাখে, কিন্তু `checkAuth` DB-তে শুধু `token` খোঁজে |
| `sendResponse` নেই | এগুলো browser redirect, JSON না |

---

## ধাপ ৯: `auth.route.ts`

```ts
router.get("/login/google", AuthController.googleLogin);
router.get("/google/success", AuthController.googleLoginSuccess);
router.get("/oauth/error", AuthController.handleOAuthError);
```

- তিনটাই `GET`, কারণ browser navigation দিয়ে আসে।

---

## Test (Browser — Postman-এ চলবে না)

Google consent screen আর redirect লাগে, তাই browser-এ test করতে হবে।

1. Browser-এ খোলো:

	```
	http://localhost:5000/api/v1/auth/login/google
	```

	(চাইলে `?redirect=/profile`)

2. "Redirecting To Google..." → Google account বাছাই করো।
3. শেষে `http://localhost:3000/dashboard`-এ যাবে (frontend না চললে page error দেখাবে, সমস্যা নেই)।
4. DevTools → **Application** → **Cookies** → `http://localhost:5000`-এ `accessToken`, `refreshToken`, `better-auth.session_token` দেখবে।
5. Prisma Studio: `user` (role `PATIENT`, `emailVerified: true`), `account` (`providerId: google`), `patient` row।

| Case | ফল |
|---|---|
| Google-এ Cancel | `/oauth/error` হয়ে `/login?error=...` (Better Auth-এর error code) |
| Blocked user | `/login?error=account_blocked`, cookie মুছে যায় |
| `?redirect=//evil.com` | `/dashboard`-এ যায় |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ (Done)

1. ✅ **`GOOGLE_CALLBACK_URL` ব্যবহার করা** — ধাপ ৪-এ `redirectURI: envVars.GOOGLE_CALLBACK_URL`। এখন port বা domain বদলালে শুধু `.env` (আর Google Console) বদলালেই হবে।

### 🟡 ভালো practice

2. **Logout-এ Better Auth-এর বাকি cookie।** `session.cookieCache` চালু থাকায় Better Auth `better-auth.session_data` cookie-ও রাখে। `clearAuthCookies` এটা মোছে না — ৫ মিনিট পর নিজেই expire হয়, ক্ষতি কম। চাইলে যোগ করো (untested):

	```ts
	CookieUtils.clearCookie(res, "better-auth.session_data", baseCookieOptions);
	```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **একই email আগে থেকে থাকলে।** Email/password দিয়ে বানানো account-এর email দিয়ে Google login করলে Better Auth নতুন user বানায় না, পুরনো user-এর সাথে Google account link করে (Google email verified হলে)। তখন `mapProfileToUser` চলে না — তাই doctor/admin Google দিয়ে ঢুকলে তার আসল role-ই থাকে।
4. **Production-এ।** Google Console-এ production-এর backend URL দিয়ে আলাদা origin আর redirect URI যোগ করতে হবে, আর consent screen "Testing" থেকে "In production"-এ publish করতে হবে।
5. **Frontend আর backend আলাদা domain হলে।** Cookie এখন `localhost:5000`-এ set হয়, আর `localhost:3000` একই host বলে frontend-ও পায়। Production-এ আলাদা domain হলে এই কারণেই `sameSite: "none"` + `secure: true` লাগে (এখন সেটাই করা আছে)।

## 🔜 পরে যা যোগ হবে

- [ ] Frontend (Next.js) login page-এ "Login with Google" button → `/api/v1/auth/login/google?redirect=...`
- [ ] Login page-এ `?error=` দেখে message দেখানো
- [ ] **`ph-tour-notes` থেকে `providerId`-এর মতো করে দেখা**
