# ClinicOS — JWT Cookie দিয়ে Login ও `checkAuth`

## ১. Package install

```bash
pnpm add jsonwebtoken cookie-parser
pnpm add -D @types/jsonwebtoken @types/cookie-parser
pnpm add cors && pnpm add -D @types/cors
```

## ২. Folder structure

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

## ৩. CORS আর `cookieParser` setup

> **নোট:** `app.ts`-এ `cookieParser()` নেই। এটা ছাড়া `req.cookies` `undefined` থাকবে, refresh কাজ করবে না।

### ৩.১ `.env` file

**File:** `.env`

```env
FRONTEND_URL=http://localhost:3000
```

### ৩.২ `env.ts`-এ add করা

**File:** `env.ts`

```ts
FRONTEND_URL: z.url({ protocol: /^https?$/ }),
```

### ৩.৩ `app.ts`-এ `cors`

তোমার `cors({...})` block বসবে `express.json()`-এর আগে, আর অবশ্যই `app.use("/api/v1", AppRoutes)`-এর আগে। ক্রম:

**File:** `app.ts`

```ts
app.use(
    cors({
        origin: envVars.FRONTEND_URL,
        credentials: true,
        exposedHeaders: ["X-Session-Refresh", "X-Session-Expires-At", "X-Time-Remaining"],
    }),
);
```

### ৩.৪ `app.ts`-এ `cookieParser`

`express.urlencoded`-এর ঠিক নিচে:

**File:** `app.ts`

```ts
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());   // ← এই line
```

### ৩.৫ `cookieParser()` কী করে

1. Browser cookie পাঠায় একটা লম্বা string হিসেবে:

```text
Cookie: accessToken=eyJhbG...; refreshToken=eyJhbG...
```

2. `cookieParser()` সেটা ভেঙে object বানিয়ে `req.cookies`-এ রাখে:

```ts
req.cookies.accessToken   // "eyJhbG..."
```

## ৪. JWT utility

**File:** `src/app/utils/jwt.ts`

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

## ৫. Token utility

**File:** `src/app/utils/token.ts`

```ts
import { envVars } from "@/app/config/env";
import { AppError } from "@/app/errorHelpers/AppError";
import { Role } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import * as z from "zod";
import { jwtUtils, type TExpiresIn } from "./jwt";

const authPayloadSchema = z.object({
    userId: z.uuid(),
    role: z.enum(Role),
});

export type TAuthPayload = z.infer<typeof authPayloadSchema>;

const ACCESS_TOKEN_EXPIRES_IN = envVars.ACCESS_TOKEN_EXPIRES_IN as TExpiresIn;
const REFRESH_TOKEN_EXPIRES_IN = envVars.REFRESH_TOKEN_EXPIRES_IN as TExpiresIn;

const getAccessToken = ({ userId, role }: TAuthPayload): string => {
    return jwtUtils.createToken({ userId, role }, envVars.ACCESS_TOKEN_SECRET, ACCESS_TOKEN_EXPIRES_IN);
};

const getRefreshToken = ({ userId, role }: TAuthPayload): string => {
    return jwtUtils.createToken({ userId, role }, envVars.REFRESH_TOKEN_SECRET, REFRESH_TOKEN_EXPIRES_IN);
};

const parseAuthPayload = (decoded: unknown): TAuthPayload => {
    const parsed = authPayloadSchema.safeParse(decoded);

    if (!parsed.success) {
        throw new AppError(httpStatus.UNAUTHORIZED, "Invalid token payload");
    }

    return parsed.data;
};

const verifyAccessToken = (token: string): TAuthPayload => {
    return parseAuthPayload(jwtUtils.verifyToken(token, envVars.ACCESS_TOKEN_SECRET));
};

const verifyRefreshToken = (token: string): TAuthPayload => {
    return parseAuthPayload(jwtUtils.verifyToken(token, envVars.REFRESH_TOKEN_SECRET));
};

export const tokenUtils = {
    getAccessToken,
    getRefreshToken,
    verifyAccessToken,
    verifyRefreshToken,
};
```

## ৬. Cookie utility

**File:** `src/app/utils/cookie.ts`

```ts
import { envVars, isProduction } from "@/app/config/env";
import type { CookieOptions, Request, Response } from "express";

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

const durationToMs = (duration: string): number => {
    const match = /^(\d+(?:\.\d+)?)\s?(ms|s|m|h|d|w|y)$/i.exec(duration);
    const value = Number(match?.[1]);
    const unit = match?.[2]?.toLowerCase() as keyof typeof UNIT_MS | undefined;

    if (!unit || Number.isNaN(value)) {
        throw new Error(`Invalid duration: ${duration}`);
    }

    return Math.round(value * UNIT_MS[unit]);
};

const ACCESS_TOKEN_MAX_AGE = durationToMs(envVars.ACCESS_TOKEN_EXPIRES_IN);
const REFRESH_TOKEN_MAX_AGE = durationToMs(envVars.REFRESH_TOKEN_EXPIRES_IN);

const baseCookieOptions: CookieOptions = {
    httpOnly: true,
    secure: isProduction,
    sameSite: "lax",
    path: "/",
    ...(envVars.COOKIE_DOMAIN && { domain: envVars.COOKIE_DOMAIN }),
};

const setAccessTokenCookie = (res: Response, token: string) => {
    res.cookie(COOKIE_NAMES.accessToken, token, { ...baseCookieOptions, maxAge: ACCESS_TOKEN_MAX_AGE });
};

const setRefreshTokenCookie = (res: Response, token: string) => {
    res.cookie(COOKIE_NAMES.refreshToken, token, { ...baseCookieOptions, maxAge: REFRESH_TOKEN_MAX_AGE });
};

const setSessionCookie = (res: Response, token: string) => {
    res.cookie(COOKIE_NAMES.session, token, { ...baseCookieOptions, maxAge: REFRESH_TOKEN_MAX_AGE });
};

const clearAuthCookies = (res: Response) => {
    res.clearCookie(COOKIE_NAMES.accessToken, baseCookieOptions);
    res.clearCookie(COOKIE_NAMES.refreshToken, baseCookieOptions);
    res.clearCookie(COOKIE_NAMES.session, baseCookieOptions);
};

const getCookie = (req: Request, name: string): string | undefined => {
    const value: unknown = req.cookies?.[name];
    return typeof value === "string" && value.length > 0 ? value : undefined;
};

export const cookieUtils = {
    setAccessTokenCookie,
    setRefreshTokenCookie,
    setSessionCookie,
    clearAuthCookies,
    getCookie,
};
```

## ৭. এখন কোন ধাপে আছি

```text
✅ jwt.ts → ✅ token.ts → ✅ cookie.ts → ✅ cookieParser
→ 👉 login   (cookie তৈরি হয়)
→ checkAuth  (cookie যাচাই করে)
→ getAuthUser → refresh-token → logout
```

## ৮. Login API

### ৮.১ Validation

**File:** `auth.validation.ts`

```ts
const loginSchema = z.object({
    email: emailSchema,
    password: z.string().min(1, "Password is required").max(128),
});
```

### ৮.২ Interface

**File:** `auth.interface.ts`

```ts
export type TLoginPayload = z.infer<typeof AuthValidation.loginSchema>;
```

### ৮.৩ Service

**File:** `auth.service.ts`

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

### ৮.৪ Controller

**File:** `auth.controller.ts`

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

### ৮.৫ Route

**File:** `auth.route.ts`

```ts
router.post("/login", authRateLimiter, validateRequest(AuthValidation.loginSchema), AuthController.login);
```

## ৯. `Request`-এ `user` type যোগ

**File:** `src/app/interfaces/express.d.ts`

```ts
import type { TAuthPayload } from "@/app/utils/token";

declare global {
    namespace Express {
        interface Request {
            user?: TAuthPayload;
        }
    }
}
```

## ১০. `checkAuth` middleware

**File:** `src/app/middleware/checkAuth.ts`

```ts
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { COOKIE_NAMES, cookieUtils } from "@/app/utils/cookie";
import { tokenUtils } from "@/app/utils/token";
import { Role } from "@/generated/prisma/enums";
import type { RequestHandler } from "express";
import httpStatus from "http-status";

export const ADMINS = [Role.SUPER_ADMIN, Role.ADMIN] as const;
export const STAFF = [...ADMINS, Role.DOCTOR] as const;

// checkAuth() = শুধু login থাকলেই চলবে; checkAuth(...ADMINS) = role-ও মিলতে হবে
export const checkAuth = (...allowedRoles: Role[]): RequestHandler => {
	return async (req, _res, next) => {
		const sessionToken = cookieUtils.getCookie(req, COOKIE_NAMES.session);
		const accessToken = cookieUtils.getCookie(req, COOKIE_NAMES.accessToken);

		if (!sessionToken || !accessToken) {
			throw new AppError(httpStatus.UNAUTHORIZED, "You are not logged in");
		}

		// আগে সস্তা কাজ: DB ছাড়াই signature + expiry যাচাই
		const payload = tokenUtils.verifyAccessToken(accessToken);

		const session = await prisma.session.findUnique({
			where: { token: sessionToken },
			select: {
				userId: true,
				expiresAt: true,
				user: { select: { role: true, isActive: true, isDeleted: true } },
			},
		});

		if (!session || session.expiresAt <= new Date() || session.userId !== payload.userId) {
			throw new AppError(httpStatus.UNAUTHORIZED, "Session expired. Please log in again");
		}

		if (!session.user.isActive || session.user.isDeleted) {
			throw new AppError(httpStatus.FORBIDDEN, "Your account is disabled. Contact the clinic");
		}

		// role নিচ্ছি DB থেকে, token থেকে না: admin role বদলালে সাথে সাথে কার্যকর হয়
		const { role } = session.user;

		if (allowedRoles.length > 0 && !allowedRoles.includes(role)) {
			throw new AppError(httpStatus.FORBIDDEN, "You do not have permission to do this");
		}

		req.user = { userId: session.userId, role };
		next();
	};
};
```

## ১১. `getAuthUser` helper

**File:** `src/app/shared/getAuthUser.ts`

```ts
import type { TAuthPayload } from "@/app/utils/token";
import type { Request } from "express";

// checkAuth পার হওয়া route-এ req.user সবসময় থাকে।
// না থাকলে route-এ checkAuth বসাতে ভুল হয়েছে: এটা আমাদের bug, user-এর দোষ না, তাই 500
export const getAuthUser = (req: Request): TAuthPayload => {
	if (!req.user) {
		throw new Error("getAuthUser called on a route without checkAuth");
	}

	return req.user;
};
```

### ১১.১ এটা কেন লাগে

তোমার `express.d.ts`-এ `user?: TAuthPayload` লেখা, মানে optional। তাই প্রতিটা controller-এ `req.user.userId` লিখলে TypeScript বলবে "`req.user` `undefined` হতে পারে"। এর তিনটা সমাধান:

| উপায় | সমস্যা |
|---|---|
| `req.user!.userId` | TypeScript-কে চুপ করানো; ভুল হলে runtime-এ crash |
| প্রতিটা controller-এ `if (!req.user)` | একই code বারবার |
| `getAuthUser(req)` | check এক জায়গায়, ফেরত আসে নিশ্চিত `TAuthPayload` |