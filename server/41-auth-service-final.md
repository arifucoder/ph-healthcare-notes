# 41 - auth.service.ts (Final / Updated Version)

> Next.js frontend বানানোর সময় backend-এর `auth.service.ts` কিছুটা update করতে হয়েছে। এই নোটে **পুরো final file** একসাথে আছে — পরের project-এ এটাই follow করব।

```
Login/Register/Verify/Google/Refresh/Change-password → generateAuthTokens(user) → JWT payload (needPasswordChange সহ) → frontend token পড়ে redirect ঠিক করে
```

---

## ধাপ ১: কী কী বদলেছে

| # | কোথায় | আগে | এখন | কেন |
|---|---|---|---|---|
| ১ | `TTokenUser` + `generateAuthTokens` | JWT-তে ৭টা field | `needPasswordChange` যোগ হয়ে ৮টা | Frontend-কে জানতে হবে user-কে `/change-password`-এ পাঠাতে হবে কিনা (admin/doctor প্রথম login) |
| ২ | `getNewToken` | `jwtPayload` হাতে লেখা (note 17) | `generateAuthTokens(user)` | Payload এখন একটাই জায়গায় বানানো হয় — field যোগ করলে সব জায়গায় একসাথে বদলায় (note 17-এর recommendation ২ ✅) |
| ৩ | `changePassword` | `generateAuthTokens(session.user)` | `generateAuthTokens({ ...session.user, needPasswordChange: false })` | `session.user` DB update-এর **আগের** data, তাই ওতে এখনো `true`। নতুন token-এ `false` না দিলে frontend আবার `/change-password`-এ পাঠাত |

## ধাপ ২: কেন JWT-তে `needPasswordChange`

- Admin/doctor বানানোর সময় `needPasswordChange: true` থাকে (note 10, 14)।
- Login করার পর frontend দেখে: `true` হলে → `/change-password`, না হলে → role অনুযায়ী dashboard।
- Login response-এর `user`-এ এটা আগে থেকেই আসত, কিন্তু **token-এর ভিতরে থাকলে** পরে যেকোনো page-এ (যেমন Next.js-এর proxy/middleware-এ) cookie থেকে token পড়ে check করা যায় — আবার API call লাগে না।
- Password বদলানো বা reset হলে DB-তে `false` হয়, আর নতুন token-এও `false` যায়।

## ধাপ ৩: পুরো `src/app/module/auth/auth.service.ts`

```ts
import { envVars } from "@/app/config/env";
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import { auth } from "@/app/lib/auth";
import { prisma } from "@/app/lib/prisma";
import { jwtUtils } from "@/app/utils/jwt";
import { tokenUtils } from "@/app/utils/token";
import { Role, UserStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import type {
	IChangePasswordPayload,
	ILoginUserPayload,
	IRegisterPatientPayload,
	IResetPasswordPayload,
	IVerifyEmailPayload,
} from "./auth.interface";

// Better Auth-এর user থেকে access + refresh token বানানো
type TAuthUser = Awaited<ReturnType<typeof auth.api.signInEmail>>["user"];

// JWT বানাতে শুধু এই field-গুলো লাগে, তাই Prisma user-ও পাঠানো যাবে
type TTokenUser = Pick<
	TAuthUser,
	"id" | "role" | "name" | "email" | "status" | "isDeleted" | "emailVerified" | "needPasswordChange"
>;
type TAuthSession = NonNullable<Awaited<ReturnType<typeof auth.api.getSession>>>;

const generateAuthTokens = (user: TTokenUser) => {
	const jwtPayload = {
		userId: user.id,
		role: user.role,
		name: user.name,
		email: user.email,
		status: user.status,
		isDeleted: user.isDeleted,
		emailVerified: user.emailVerified,
		needPasswordChange: user.needPasswordChange, // ✅ নতুন
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

const getMe = async (user: IRequestUser) => {
	const isUserExists = await prisma.user.findUnique({
		where: {
			id: user.userId,
		},
		include: {
			patient: {
				include: {
					appointments: {
						take: 5,
						orderBy: { createdAt: "desc" },
					},
					reviews: true,
					prescriptions: true,
					medicalReports: true,
					patientHealthData: true,
				},
			},
			doctor: {
				include: {
					specialties: true,
					appointments: {
						take: 5,
						orderBy: { createdAt: "desc" },
					},
					reviews: true,
					prescriptions: true,
				},
			},
			admin: true,
		},
	});

	if (!isUserExists) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	return isUserExists;
};

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

	// ধাপ ৫: fresh data দিয়ে নতুন token (payload একটাই জায়গায় বানানো হয়)
	const { accessToken: newAccessToken, refreshToken: newRefreshToken } = generateAuthTokens(user);

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

const changePassword = async (payload: IChangePasswordPayload, sessionToken: string) => {
	const headers = new Headers({
		Authorization: `Bearer ${sessionToken}`,
	});

	const session = await auth.api.getSession({ headers });

	if (!session) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Invalid session token");
	}

	const { currentPassword, newPassword } = payload;

	// ভুল currentPassword হলে Better Auth APIError throw করবে
	// revokeOtherSessions: true → সব পুরনো session মুছে নতুন session, result.token-এ নতুন token
	const result = await auth.api.changePassword({
		body: {
			currentPassword,
			newPassword,
			revokeOtherSessions: true,
		},
		headers,
	});

	if (session.user.needPasswordChange) {
		await prisma.user.update({
			where: { id: session.user.id },
			data: { needPasswordChange: false },
		});
	}

	// password বদলানো শেষ, তাই নতুন token-এ needPasswordChange: false
	const { accessToken, refreshToken } = generateAuthTokens({ ...session.user, needPasswordChange: false });

	return {
		...result,
		accessToken,
		refreshToken,
	};
};

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

const verifyEmail = async (payload: IVerifyEmailPayload) => {
	const { email, otp } = payload;

	// ভুল/expired OTP হলে Better Auth নিজেই APIError (400) throw করে
	// সঠিক হলে Better Auth নিজেই DB-তে emailVerified: true করে দেয়
	const result = await auth.api.verifyEmailOTP({
		body: { email, otp },
	});

	// autoSignInAfterVerification বন্ধ থাকলে token আসে না
	if (result.token === null) {
		return { token: null, user: result.user };
	}

	// JWT-তে role/status লাগে, তাই DB থেকে পুরো user নেওয়া
	const user = await prisma.user.findUnique({ where: { email: result.user.email } });

	if (!user) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	const { accessToken, refreshToken } = generateAuthTokens(user);

	return {
		token: result.token,
		accessToken,
		refreshToken,
		user: result.user,
	};
};

const forgetPassword = async (email: string) => {
	const user = await prisma.user.findUnique({
		where: { email },
	});

	// ১. আগে "আছে কি নেই": না থাকা বা deleted, দুটোতেই একই message
	if (!user || user.isDeleted || user.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	// ২. তারপর account-এর অবস্থা
	if (user.status === UserStatus.BLOCKED) {
		throw new AppError(httpStatus.FORBIDDEN, "User is blocked");
	}

	if (!user.emailVerified) {
		throw new AppError(httpStatus.BAD_REQUEST, "Email not verified");
	}

	// OTP তৈরি করে sendVerificationOTP-কে type: "forget-password" দিয়ে call করে
	await auth.api.requestPasswordResetEmailOTP({
		body: { email },
	});
};

const resetPassword = async (payload: IResetPasswordPayload) => {
	const { email, otp, newPassword } = payload;

	const user = await prisma.user.findUnique({
		where: { email },
	});

	if (!user || user.isDeleted || user.status === UserStatus.DELETED) {
		throw new AppError(httpStatus.NOT_FOUND, "User not found");
	}

	if (user.status === UserStatus.BLOCKED) {
		throw new AppError(httpStatus.FORBIDDEN, "User is blocked");
	}

	if (!user.emailVerified) {
		throw new AppError(httpStatus.BAD_REQUEST, "Email not verified");
	}

	// ভুল/expired OTP হলে Better Auth APIError throw করে
	// revokeSessionsOnPasswordReset: true থাকায় password update + সব session মোছা একই transaction-এ হয়
	await auth.api.resetPasswordEmailOTP({
		body: {
			email,
			otp,
			password: newPassword,
		},
	});

	if (user.needPasswordChange) {
		await prisma.user.update({
			where: { id: user.id },
			data: { needPasswordChange: false },
		});
	}
};

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

export const AuthService = {
	registerPatient,
	loginUser,
	getMe,
	getNewToken,
	changePassword,
	logoutUser,
	logoutFromAllDevices,
	verifyEmail,
	forgetPassword,
	resetPassword,
	googleLoginSuccess,
};
```

## ধাপ ৪: কোন function কোন নোটে বিস্তারিত

| Function | কাজ | নোট |
|---|---|---|
| `generateAuthTokens` | user থেকে access + refresh token | 13, 20 |
| `registerPatient` | Better Auth sign up + Patient row | 07, 13, 20 |
| `loginUser` | blocked/deleted check → sign in → token | 13 |
| `getMe` | logged in user + role অনুযায়ী data | 16 |
| `getNewToken` | refresh token দিয়ে নতুন token + session-এর মেয়াদ বাড়ানো | 17 |
| `changePassword` | password বদল + অন্য session বাতিল | 18 |
| `logoutUser` / `logoutFromAllDevices` | এক / সব device থেকে logout | 19 |
| `verifyEmail` | OTP দিয়ে email verify + auto login | 20 |
| `forgetPassword` / `resetPassword` | OTP দিয়ে password reset | 21 |
| `googleLoginSuccess` | Google login-এর পর Patient row + token | 22 |

## ধাপ ৫: Test

1. Admin/doctor বানাও (`needPasswordChange: true`) → `POST /api/v1/auth/login` → `accessToken` jwt.io-তে paste করো → payload-এ `"needPasswordChange": true` দেখা যাবে।
2. `POST /api/v1/auth/change-password` (body: `{ "currentPassword": "...", "newPassword": "..." }`) → নতুন `accessToken`-এ `"needPasswordChange": false`।
3. `POST /api/v1/auth/refresh-token` → নতুন token-এও ৮টা field একই থাকবে।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Backend নিজে `needPasswordChange` আটকায় না** — এখন শুধু frontend redirect করে। কেউ সরাসরি API call করলে (Postman দিয়ে) পুরনো password নিয়েই সব route ব্যবহার করতে পারবে। `checkAuth`-এ check দিলে backend-ও নিরাপদ (change-password আর logout-all route বাদে — `/logout`-এ checkAuth নেই):
   ```ts
   // checkAuth.ts — DB user check করার পরে
   const allowedPaths = ["/api/v1/auth/change-password", "/api/v1/auth/logout-all"];
   if (user.needPasswordChange && !allowedPaths.includes(req.originalUrl)) {
   	throw new AppError(httpStatus.FORBIDDEN, "Please change your password first");
   }
   ```

2. **`registerPatient` এখনো token বানায়** — note 20-এর ধাপ ৭-এ ঠিক করা হয়েছিল যে `requireEmailVerification: true` থাকায় register-এ token বানানো হবে না (verify-email-এর পরে বানানো হবে)। কিন্তু এই code-এ আবার `generateAuthTokens(data.user)` আছে, ফলে unverified user-ও access/refresh cookie পাচ্ছে। Session cookie না থাকায় `checkAuth` সম্ভবত আটকাবে (untested), তবুও এটা অপ্রয়োজনীয়:
   ```ts
   // registerPatient-এর শেষে
   return {
   	user: data.user,
   	patient,
   };
   ```
   তখন `auth.controller.ts`-এর `registerPatient` থেকেও cookie set করার অংশ সরাতে হবে (note 20 দেখো)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **JWT-তে field যোগ করলে frontend type-ও বদলাও** — client-এর `src/types/auth.types.ts`-এ token payload-এর জন্য আলাদা type রাখলে (`userId`, `role`, ..., `needPasswordChange`) proxy-তে token decode করার সময় সুবিধা হবে। Backend-এ field বাড়ালে এখানে মিলিয়ে নিতে হবে।
