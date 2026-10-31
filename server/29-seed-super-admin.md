# 29 - Seed Super Admin

```
pnpm dev → server.ts → startServer() → seedSuperAdmin()
→ SUPER_ADMIN আগে থেকে আছে? → হ্যাঁ: কিছু না | না: Better Auth signUp → role = SUPER_ADMIN + Admin row
→ তারপর app.listen + cron
```

**কেন লাগে?** Admin বানাতে পারে শুধু SUPER_ADMIN, কিন্তু প্রথম SUPER_ADMIN-কে বানাবে কে? Register route সবাইকে `PATIENT` বানায়। তাই server চালু হওয়ার সময় `.env`-এর email/password দিয়ে নিজে থেকে একজন SUPER_ADMIN তৈরি করা হয় — একেই **seed** বলে। একবার তৈরি হলে পরের বার আর কিছু করে না।

---

## ধাপ ১: `.env`

```dotenv
SUPER_ADMIN_EMAIL=superadmin@phdoc.com
SUPER_ADMIN_PASSWORD=strongPassword123
```

- Password অন্তত ৮ অক্ষর (Better Auth-এর default নিয়ম), নাহলে seed fail করে server চালু হবে না।

---

## ধাপ ২: `src/app/config/env.ts`

```ts
interface EnvConfig {
	// ...অন্যগুলো
	SUPER_ADMIN_EMAIL: string;
	SUPER_ADMIN_PASSWORD: string;
}

// required env-এর list-এ
"SUPER_ADMIN_EMAIL",
"SUPER_ADMIN_PASSWORD",

// return object-এ
return {
	// ...অন্যগুলো
	SUPER_ADMIN_EMAIL: process.env.SUPER_ADMIN_EMAIL as string,
	SUPER_ADMIN_PASSWORD: process.env.SUPER_ADMIN_PASSWORD as string,
};
```

---

## ধাপ ৩: `src/app/utils/seed.ts`

```ts
import { envVars } from "@/app/config/env";
import { auth } from "@/app/lib/auth";
import { prisma } from "@/app/lib/prisma";
import { Role } from "@/generated/prisma/enums";

const SUPER_ADMIN_NAME = "Super Admin";
// Admin model-এ contactNumber required; পরে super admin নিজে profile update করে নেবে
const SUPER_ADMIN_CONTACT_PLACEHOLDER = "N/A";

export const seedSuperAdmin = async () => {
	const { SUPER_ADMIN_EMAIL, SUPER_ADMIN_PASSWORD } = envVars;

	const superAdminExists = await prisma.user.findFirst({
		where: { role: Role.SUPER_ADMIN },
		select: { id: true },
	});

	if (superAdminExists) {
		return;
	}

	// এই email দিয়ে অন্য কেউ আগে থেকে account খুলে থাকলে তাকে ছোঁয়া যাবে না
	const emailTaken = await prisma.user.findUnique({
		where: { email: SUPER_ADMIN_EMAIL },
		select: { role: true },
	});

	if (emailTaken) {
		throw new Error(`Cannot seed super admin: ${SUPER_ADMIN_EMAIL} is already used by a ${emailTaken.role} account`);
	}

	// Password hash + account তৈরি Better Auth করবে
	const { user } = await auth.api.signUpEmail({
		body: {
			name: SUPER_ADMIN_NAME,
			email: SUPER_ADMIN_EMAIL,
			password: SUPER_ADMIN_PASSWORD,
		},
	});

	try {
		await prisma.$transaction([
			// role এখানে set করা হচ্ছে, যাতে auth.ts-এ role: { input: false } দিলেও seed কাজ করে
			prisma.user.update({
				where: { id: user.id },
				data: {
					role: Role.SUPER_ADMIN,
					emailVerified: true,
					needPasswordChange: false,
				},
			}),
			prisma.admin.create({
				data: {
					userId: user.id,
					name: SUPER_ADMIN_NAME,
					email: SUPER_ADMIN_EMAIL,
					contactNumber: SUPER_ADMIN_CONTACT_PLACEHOLDER,
				},
			}),
			// signUp কোনো session বানিয়ে থাকলে মুছে ফেলা (createAdmin-এর মতো)
			prisma.session.deleteMany({
				where: { userId: user.id },
			}),
		]);
	} catch (error) {
		// শুধু এইমাত্র বানানো user-টাই মোছা হচ্ছে, id দিয়ে
		await prisma.user.delete({ where: { id: user.id } });
		throw error;
	}

	console.warn(`Super admin created: ${SUPER_ADMIN_EMAIL}`);
};
```

### Seed-এর ধাপগুলো

| # | কী করে | কেন |
|---|---|---|
| ১ | SUPER_ADMIN আগে থেকে আছে? → `return` | প্রতিবার server চালু হলে seed চলে — duplicate হবে না |
| ২ | এই email অন্য কারো? → `throw` | কোনো patient-এর account ভুল করে super admin বানিয়ে দেওয়া যাবে না |
| ৩ | `auth.api.signUpEmail` | password hash আর `account` row Better Auth বানায় — নিজে hash করতে হয় না |
| ৪ | transaction: role = SUPER_ADMIN, `emailVerified: true`, `Admin` row, session মোছা | signUp সবাইকে PATIENT বানায়, তাই পরে role বদলানো; email verify ছাড়া login আটকে যেত |
| ৫ | fail হলে user মুছে `throw` | অর্ধেক তৈরি user (role PATIENT, Admin row নেই) পড়ে থাকবে না — পরের বার আবার চেষ্টা করা যাবে |

- `$transaction([...])` (array) → তিনটা query হয় সব সফল, না হলে কোনোটাই না।
- Signup আর transaction-এর মাঝে fail হলে নিজে user মোছা লাগে, কারণ `signUpEmail` transaction-এর বাইরে ([নোট ০৭](./07-patient-registration-with-transaction.md)-এর patient registration-এর মতো একই কৌশল)।

---

## ধাপ ৪: `server.ts` (পুরোটা)

Seed server চালু হওয়ার **আগে** চলবে, আর fail করলে server বন্ধ হবে। সাথে graceful shutdown আর cron — তাই পুরো file:

```ts
import type { Server } from "http";
import { schedule, shutdown as shutdownCron } from "node-cron";
import app from "./app";
import { envVars } from "./app/config/env";
import { prisma } from "./app/lib/prisma";
import { AppointmentService } from "./app/module/appointment/appointment.service";
import { seedSuperAdmin } from "./app/utils/seed";

let server: Server | undefined;
let isShuttingDown = false;
const PORT = envVars.PORT;

const startCronJobs = () => {
	// প্রতি ৫ মিনিটে: ৩০ মিনিটের পুরনো unpaid appointment cancel করে slot ফাঁকা করা
	schedule(
		"*/5 * * * *",
		async () => {
			try {
				const { canceledCount } = await AppointmentService.cancelUnpaidAppointments();

				if (canceledCount > 0) {
					console.warn(`[cron] Canceled ${canceledCount} unpaid appointment(s)`);
				}
			} catch (error) {
				console.error("[cron] Failed to cancel unpaid appointments:", error instanceof Error ? error.message : error);
			}
		},
		{
			name: "cancel-unpaid-appointments",
			noOverlap: true, // আগের run শেষ না হলে নতুন run শুরু হবে না
		},
	);
};

const gracefulShutdown = async (reason: string, exitCode: number) => {
	// একাধিক signal একসাথে এলে দুইবার shutdown হবে না
	if (isShuttingDown) return;
	isShuttingDown = true;

	console.warn(`${reason}. Shutting down server...`);

	// কোনো কারণে আটকে গেলে ১০ সেকেন্ড পর জোর করে বন্ধ
	setTimeout(() => process.exit(exitCode), 10_000).unref();

	await shutdownCron();

	const runningServer = server;
	if (runningServer) {
		// নতুন request নেওয়া বন্ধ, চলমান request শেষ হওয়া পর্যন্ত অপেক্ষা
		await new Promise<void>((resolve) => runningServer.close(() => resolve()));
	}

	await prisma.$disconnect();

	process.exit(exitCode);
};

const startServer = async () => {
	try {
		await seedSuperAdmin();
	} catch (error) {
		console.error("Failed to seed super admin:", error);
		await gracefulShutdown("Seeding failed", 1);
		return;
	}

	server = app.listen(PORT, () => {
		console.log(`Server is running on port http://localhost:${PORT}`);
		startCronJobs();
	});

	// যেমন port আগে থেকেই ব্যবহার হচ্ছে (EADDRINUSE)
	server.on("error", (error) => {
		console.error("Failed to start server:", error);
		void gracefulShutdown("Server error", 1);
	});
};

process.on("SIGTERM", () => void gracefulShutdown("SIGTERM received", 0));
process.on("SIGINT", () => void gracefulShutdown("SIGINT received", 0));

process.on("uncaughtException", (error) => {
	console.error("Uncaught Exception:", error);
	void gracefulShutdown("Uncaught exception", 1);
});

process.on("unhandledRejection", (reason) => {
	console.error("Unhandled Rejection:", reason);
	void gracefulShutdown("Unhandled rejection", 1);
});

// package.json-এ "type": "module", তাই top-level await চলে
await startServer();
```

### `server.ts`-এ কী বদলাল

| অংশ | কাজ |
|---|---|
| `startServer()` | আগে seed, সফল হলে তারপর `app.listen` — super admin ছাড়া server চলবে না |
| `app.listen`-এর callback-এ `startCronJobs()` | server সত্যিই চালু হলে তবেই cron ([নোট ২৮](./28-stripe-payment.md)) |
| `gracefulShutdown` | বন্ধের সময় ক্রমানুসারে: cron থামানো → নতুন request বন্ধ, চলমানগুলো শেষ → DB disconnect → exit |
| `isShuttingDown` | `Ctrl+C` দুবার চাপলে বা দুটো signal একসাথে এলে shutdown দুবার চলবে না |
| `setTimeout(...).unref()` | কিছু আটকে গেলে ১০ সেকেন্ড পর জোর করে বন্ধ; `unref` → এই timer নিজে process-কে বাঁচিয়ে রাখে না |
| `SIGTERM` / `SIGINT` | `SIGINT` = `Ctrl+C`, `SIGTERM` = hosting (Render/Docker) বন্ধ করলে → exit code `0` (স্বাভাবিক) |
| `uncaughtException` / `unhandledRejection` | ধরা না পড়া error → log করে নিরাপদে বন্ধ, exit code `1` |
| `void gracefulShutdown(...)` | ফল ব্যবহার করছি না জানাতে `void` — oxlint-এর `no-floating-promises` খুশি থাকে |
| `await startServer()` | ESM-এ top-level `await` চলে |

---

## Test

### ১. প্রথমবার চালু

```shell
pnpm dev
```

Terminal-এ:

```
Super admin created: superadmin@phdoc.com
Server is running on port http://localhost:5000
```

Prisma Studio: `user` table-এ role `SUPER_ADMIN`, `emailVerified: true`; `admins` table-এ একই email, `contactNumber: N/A`; `account` table-এ hash করা password।

### ২. আবার চালু (restart)

→ `Super admin created` আর আসবে না, শুধু `Server is running...` — duplicate হয়নি।

### ৩. Postman-এ login

`POST http://localhost:5000/api/v1/auth/login`

```json
{
	"email": "superadmin@phdoc.com",
	"password": "strongPassword123"
}
```

→ `200`, ৩টা cookie। তারপর:

| Request | ফল |
|---|---|
| `GET /api/v1/auth/me` | `200`, `role: SUPER_ADMIN`, `admin` data সহ |
| SUPER_ADMIN-only route (যেমন admin বানানো / `GET /api/v1/admins`) | `200` |

### ৪. Fail হওয়ার case

Test-এর জন্য DB-তে super admin না রেখে (Studio থেকে মুছে) চেষ্টা করো:

| `.env` | ফল |
|---|---|
| `SUPER_ADMIN_EMAIL` = আগে register করা কোনো patient-এর email | `Failed to seed super admin: Error: Cannot seed super admin: ... is already used by a PATIENT account` → `Seeding failed. Shutting down server...` |
| `SUPER_ADMIN_PASSWORD=123` (৮-এর কম) | Better Auth error → server বন্ধ |
| `SUPER_ADMIN_EMAIL` মুছে দেওয়া | `env.ts` নিজেই "missing env" error দেবে |

### ৫. Graceful shutdown

`pnpm dev` চলা অবস্থায় `Ctrl+C` → `SIGINT received. Shutting down server...`।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`console.log` → oxlint error/warning।** `app.listen`-এর ভেতরে `console.log` আছে, কিন্তু project-এর oxlint-এ `no-console` শুধু `warn`/`error` চলতে দেয় — `pnpm lint`-এ ধরা পড়বে।

	```ts
	console.warn(`Server is running on port http://localhost:${PORT}`);
	```

2. **Seed-এর সময় super admin-এর email-এ OTP mail যায়।** `sendOnSignUp: true` থাকায় `signUpEmail` "Verify your email" OTP পাঠায় ([নোট ২০](./20-nodemailer-verify-email.md)) — যদিও পরে আমরা নিজেই `emailVerified: true` করে দিই। ক্ষতি নেই (SMTP fail হলেও error গিলে ফেলা হয়), কিন্তু অকারণ mail। চাইলে `sendVerificationOTP`-এ super admin-এর email বাদ দাও:

	```ts
	if (email === envVars.SUPER_ADMIN_EMAIL) return;
	```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`needPasswordChange: false`।** `.env`-এর password অনেকে জানে (team, deploy server)। প্রথম login-এর পর বদলাতে বাধ্য করতে চাইলে `true` দাও — change-password flow ([নোট ১৮](./18-change-password.md)) আগেই আছে।
4. **Production-এ।** Hosting-এর env-এ শক্ত password দিও; super admin তৈরি হয়ে গেলে `.env`-এর password বদলালেও DB-র password বদলায় না (seed আর চলে না)।
