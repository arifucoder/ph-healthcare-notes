## install packages

```shell
pnpm add jsonwebtoken cookie-parser
pnpm add -D @types/jsonwebtoken @types/cookie-parser
pnpm add cors && pnpm add -D @types/cors
```

## folder structure
```text
apps/api/src/
├─ app.ts                          ✏️ cookieParser() (+ cors credentials: true)
└─ app/
   ├─ config/env.ts                ✏️ FRONTEND_URL (cors-এর জন্য)
   ├─ utils/                       🆕
   │  ├─ jwt.ts                    🆕 jsonwebtoken-এর পাতলা wrapper
   │  ├─ token.ts                  🆕 access/refresh token বানানো
   │  └─ cookie.ts                 🆕 cookie set/clear + options
   ├─ middleware/
   │  └─ checkAuth.ts              🆕 cookie → verify → req.user → role check
   ├─ shared/
   │  └─ getAuthUser.ts            🆕 req.user নিরাপদে বের করা
   ├─ interfaces/
   │  └─ express.d.ts              🆕 Request-এ user type যোগ
   └─ module/auth/
      ├─ auth.route.ts             ✏️ /login, /refresh-token, /logout
      ├─ auth.controller.ts        ✏️ cookie set/clear এখানে
      ├─ auth.service.ts           ✏️ login, getNewTokens, logout
      ├─ auth.validation.ts        ✏️ loginSchema
      └─ auth.interface.ts         ✏️ TLoginPayload, TJwtPayload
```

app.ts-এ cookieParser() নেই। এটা ছাড়া req.cookies undefined থাকবে, refresh কাজ করবে না।
.env 
```dotenv
FRONTEND_URL=http://localhost:3000
```
In env.ts 
```text
FRONTEND_URL: z.url({ protocol: /^https?$/ }),
```
app.ts: তোমার cors({...}) block বসবে express.json()-এর আগে, আর অবশ্যই app.use("/api/v1", AppRoutes)-এর আগে। ক্রম:
```ts
app.use(
	cors({
		origin: envVars.FRONTEND_URL,
		credentials: true,
		exposedHeaders: ["X-Session-Refresh", "X-Session-Expires-At", "X-Time-Remaining"],
	}),
);
```
express.urlencoded-এর ঠিক নিচে:
```ts
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());   // ← এই line
```
এটা কী করে

Browser cookie পাঠায় একটা লম্বা string হিসেবে:

```
Cookie: accessToken=eyJhbG...; refreshToken=eyJhbG...
```

cookieParser() সেটা ভেঙে object বানিয়ে req.cookies-এ রাখে:
```ts
req.cookies.accessToken   // "eyJhbG..."
```

src/app/utils/jwt.ts
```ts
import { AppError } from "@/app/errorHelpers/AppError";
import httpStatus from "http-status";
import jwt, { type JwtPayload, type SignOptions } from "jsonwebtoken";

export type TExpiresIn = NonNullable<SignOptions["expiresIn"]>;

const createToken = (payload: JwtPayload, secret: string, expiresIn: TExpiresIn): string => {
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

// decode কোনো verify করে না, শুধু payload পড়ে, তাই throw না করে null
// auth-এর সিদ্ধান্তে (কে user, কী role) কখনো এটা না, সবসময় verifyToken
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

src/app/utils/cookie.ts

```ts
import { envVars, isProduction } from "@/app/config/env";
import type { CookieOptions, Response } from "express";

export const COOKIE_NAMES = {
	accessToken: "accessToken",
	refreshToken: "refreshToken",
	session: "better-auth.session_token",
} as const;

const UNIT_MS = {
	ms: 1,
	s: 1000,
	m: 60_000,
	h: 3_600_000,
	d: 86_400_000,
	w: 604_800_000,
	y: 31_557_600_000,
} as const;

// "15m" / "7d" → millisecond। cookie-র maxAge শুধু ms বোঝে
// format আগেই env.ts-এর DURATION_REGEX নিশ্চিত করেছে; throw-টা শুধু নিরাপত্তার জাল
const durationToMs = (duration: string): number => {
	const match = /^(\d+(?:\.\d+)?)\s?(ms|s|m|h|d|w|y)$/i.exec(duration);
	const value = Number(match?.[1]);
	const unit = match?.[2]?.toLowerCase() as keyof typeof UNIT_MS | undefined;

	if (!unit || Number.isNaN(value)) {
		throw new Error(`Invalid duration: ${duration}`);
	}

	return Math.round(value * UNIT_MS[unit]);
};

// .env থেকেই হিসাব, তাই token-এর মেয়াদ আর cookie-র মেয়াদ কখনো আলাদা হয় না
const ACCESS_TOKEN_MAX_AGE = durationToMs(envVars.ACCESS_TOKEN_EXPIRES_IN);
const REFRESH_TOKEN_MAX_AGE = durationToMs(envVars.REFRESH_TOKEN_EXPIRES_IN);

// set আর clear দুই জায়গাতেই একই options লাগে, নইলে browser cookie মোছে না
const baseCookieOptions: CookieOptions = {
	httpOnly: true, // JS (document.cookie) দিয়ে পড়া যাবে না
	secure: isProduction, // production-এ শুধু HTTPS
	sameSite: "lax",
	path: "/",
	// ".yourdomain.com" দিলে web আর api দুই subdomain-ই cookie পায়; local-এ থাকে না
	...(envVars.COOKIE_DOMAIN && { domain: envVars.COOKIE_DOMAIN }),
};

const setAccessTokenCookie = (res: Response, token: string) => {
	res.cookie(COOKIE_NAMES.accessToken, token, { ...baseCookieOptions, maxAge: ACCESS_TOKEN_MAX_AGE });
};

const setRefreshTokenCookie = (res: Response, token: string) => {
	res.cookie(COOKIE_NAMES.refreshToken, token, { ...baseCookieOptions, maxAge: REFRESH_TOKEN_MAX_AGE });
};

// session-এর আসল মেয়াদ DB-র sessions table-এ; cookie শুধু refresh token-এর সমান সময় browser-এ থাকে
const setSessionCookie = (res: Response, token: string) => {
	res.cookie(COOKIE_NAMES.session, token, { ...baseCookieOptions, maxAge: REFRESH_TOKEN_MAX_AGE });
};

const clearAuthCookies = (res: Response) => {
	res.clearCookie(COOKIE_NAMES.accessToken, baseCookieOptions);
	res.clearCookie(COOKIE_NAMES.refreshToken, baseCookieOptions);
	res.clearCookie(COOKIE_NAMES.session, baseCookieOptions);
};

export const cookieUtils = {
	setAccessTokenCookie,
	setRefreshTokenCookie,
	setSessionCookie,
	clearAuthCookies,
};
```

Ekhon dhap
```text
✅ jwt.ts → ✅ token.ts → ✅ cookie.ts → ✅ cookieParser
→ 👉 login   (cookie তৈরি হয়)
→ checkAuth  (cookie যাচাই করে)
→ getAuthUser → refresh-token → logout
```

auth.validation.ts
```ts
const loginSchema = z.object({
	email: emailSchema,
	password: z.string().min(1, "Password is required").max(128),
});
```

auth.interface.ts
```ts
export type TLoginPayload = z.infer<typeof AuthValidation.loginSchema>;
```

auth.service.ts
```ts
const login = async (payload: TLoginPayload) => {
	const { email, password } = payload;

	// ভুল email/password বা verify-না-হওয়া email হলে Better Auth নিজেই throw করে
	const { token: sessionToken, user: signedInUser } = await auth.api.signInEmail({
		body: { email, password },
	});

	const user = await prisma.user.findUniqueOrThrow({
		where: { id: signedInUser.id },
		select: {
			id: true,
			name: true,
			email: true,
			role: true,
			isActive: true,
			isDeleted: true,
			needPasswordChange: true,
		},
	});

	if (!user.isActive || user.isDeleted) {
		// signInEmail আগেই session বানিয়ে ফেলেছে, সেটা রেখে দেওয়া যাবে না
		await prisma.session.deleteMany({ where: { token: sessionToken } });
		throw new AppError(httpStatus.FORBIDDEN, "Your account is disabled. Contact the clinic");
	}

	const tokenPayload = { userId: user.id, role: user.role };

	return {
		accessToken: tokenUtils.getAccessToken(tokenPayload),
		refreshToken: tokenUtils.getRefreshToken(tokenPayload),
		sessionToken,
		user: {
			id: user.id,
			name: user.name,
			email: user.email,
			role: user.role,
			needPasswordChange: user.needPasswordChange,
		},
	};
};
```

auth.controller.ts
```ts
const login = catchAsync(async (req, res) => {
	const { accessToken, refreshToken, sessionToken, user } = await AuthService.login(req.body);

	cookieUtils.setAccessTokenCookie(res, accessToken);
	cookieUtils.setRefreshTokenCookie(res, refreshToken);
	cookieUtils.setSessionCookie(res, sessionToken);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Logged in successfully",
		data: user,
	});
});
```

auth.route.ts
```ts
router.post("/login", authRateLimiter, validateRequest(AuthValidation.loginSchema), AuthController.login);
```