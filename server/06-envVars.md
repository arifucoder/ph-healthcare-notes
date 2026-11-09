# 06 - Configuring Environment Variables (envVars)

> Module: 38-01 — Configuring Environment Variables And Create Patient Model And Completing Patient Registration With Transaction

**কেন লাগে?**
সব জায়গায় `process.env.XYZ` লিখলে:
- কোন variable missing, সেটা app চালানোর **অনেক পরে** (যখন ওই code চলে) ধরা পড়ে
- Type safety নেই — সবকিছু `string | undefined`

👉 তাই সব env variable **এক জায়গায় load + check** করে `envVars` object হিসেবে export করবো।

---

## ধাপ ১: Folder ও File

```
src/app/config/
└── env.ts
```

---

## ধাপ ২: env.ts (এখনকার পুরো file)

> 🔄 শুরুতে এখানে শুধু ৫টা variable ছিল, আর check করা হতো আলাদা `requireEnvVariable` array দিয়ে। Project বড় হওয়ার সাথে নতুন variable যোগ হয়েছে, আর check-এর পদ্ধতিও সহজ করা হয়েছে (`getEnv` helper)।

`src/app/config/env.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import dotenv from "dotenv";
import httpStatus from "http-status";

dotenv.config();

interface EnvConfig {
	NODE_ENV: string;
	PORT: string;
	DATABASE_URL: string;
	BETTER_AUTH_SECRET: string;
	BETTER_AUTH_URL: string;
	ACCESS_TOKEN_SECRET: string;
	REFRESH_TOKEN_SECRET: string;
	ACCESS_TOKEN_EXPIRES_IN: string;
	REFRESH_TOKEN_EXPIRES_IN: string;
	BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN: string;
	BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE: string;

	FRONTEND_URL: string;
	COOKIE_DOMAIN?: string; // শুধু production এ; local এ দরকার নেই

	EMAIL_SENDER: {
		SMTP_HOST: string;
		SMTP_PORT: string;
		SMTP_USER: string;
		SMTP_PASS: string;
		SMTP_FROM: string;
	};

	GOOGLE_CLIENT_ID: string;
	GOOGLE_CLIENT_SECRET: string;
	GOOGLE_CALLBACK_URL: string;

	CLOUDINARY: {
		CLOUDINARY_CLOUD_NAME: string;
		CLOUDINARY_API_KEY: string;
		CLOUDINARY_API_SECRET: string;
	};

	STRIPE: {
		STRIPE_SECRET_KEY: string;
		STRIPE_WEBHOOK_SECRET: string;
	};

	SUPER_ADMIN_EMAIL: string;
	SUPER_ADMIN_PASSWORD: string;
}

// value না থাকলে সাথে সাথে throw; থাকলে TypeScript নিজেই জানে এটা string
const getEnv = (key: string): string => {
	const value = process.env[key];

	if (!value) {
		throw new AppError(
			httpStatus.INTERNAL_SERVER_ERROR,
			`Environment variable ${key} is required but not set in .env file.`,
		);
	}

	return value;
};

const loadEnvVariables = (): EnvConfig => ({
	NODE_ENV: getEnv("NODE_ENV"),
	PORT: getEnv("PORT"),
	DATABASE_URL: getEnv("DATABASE_URL"),
	BETTER_AUTH_SECRET: getEnv("BETTER_AUTH_SECRET"),
	BETTER_AUTH_URL: getEnv("BETTER_AUTH_URL"),
	ACCESS_TOKEN_SECRET: getEnv("ACCESS_TOKEN_SECRET"),
	REFRESH_TOKEN_SECRET: getEnv("REFRESH_TOKEN_SECRET"),
	ACCESS_TOKEN_EXPIRES_IN: getEnv("ACCESS_TOKEN_EXPIRES_IN"),
	REFRESH_TOKEN_EXPIRES_IN: getEnv("REFRESH_TOKEN_EXPIRES_IN"),
	BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN: getEnv("BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN"),
	BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE: getEnv("BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE"),
	FRONTEND_URL: getEnv("FRONTEND_URL"),
	COOKIE_DOMAIN: process.env.COOKIE_DOMAIN,

	EMAIL_SENDER: {
		SMTP_HOST: getEnv("SMTP_HOST"),
		SMTP_PORT: getEnv("SMTP_PORT"),
		SMTP_USER: getEnv("SMTP_USER"),
		SMTP_PASS: getEnv("SMTP_PASS"),
		SMTP_FROM: getEnv("SMTP_FROM"),
	},

	GOOGLE_CLIENT_ID: getEnv("GOOGLE_CLIENT_ID"),
	GOOGLE_CLIENT_SECRET: getEnv("GOOGLE_CLIENT_SECRET"),
	GOOGLE_CALLBACK_URL: getEnv("GOOGLE_CALLBACK_URL"),

	CLOUDINARY: {
		CLOUDINARY_CLOUD_NAME: getEnv("CLOUDINARY_CLOUD_NAME"),
		CLOUDINARY_API_KEY: getEnv("CLOUDINARY_API_KEY"),
		CLOUDINARY_API_SECRET: getEnv("CLOUDINARY_API_SECRET"),
	},

	STRIPE: {
		STRIPE_SECRET_KEY: getEnv("STRIPE_SECRET_KEY"),
		STRIPE_WEBHOOK_SECRET: getEnv("STRIPE_WEBHOOK_SECRET"),
	},

	SUPER_ADMIN_EMAIL: getEnv("SUPER_ADMIN_EMAIL"),
	SUPER_ADMIN_PASSWORD: getEnv("SUPER_ADMIN_PASSWORD"),
});

export const envVars = loadEnvVariables();
```

**কীভাবে কাজ করে:**
1. `dotenv.config()` → `.env` file থেকে সব variable `process.env`-এ load করে
2. `interface EnvConfig` → কোন কোন variable থাকবে, তার type
3. `getEnv(key)` → variable আছে কিনা check করে। না থাকলে `throw`, থাকলে `string` ফেরত দেয়
4. `loadEnvVariables()` → সব variable `getEnv` দিয়ে পড়ে একটা object বানায়
5. `export const envVars` → অন্য file-এ `envVars.PORT`, `envVars.STRIPE.STRIPE_SECRET_KEY` এভাবে ব্যবহার

### আগের পদ্ধতি বনাম `getEnv`

| | আগে (`requireEnvVariable` array) | এখন (`getEnv`) |
| --- | --- | --- |
| Variable-এর নাম কয় জায়গায় লিখতে হয় | ৩ (interface, array, return) | ২ (interface, `getEnv`) |
| Type | `process.env.X as string` (জোর করে) | `getEnv` নিজেই `string` ফেরত দেয়, `as` লাগে না |
| Array-তে নাম দিতে ভুলে গেলে | Check হয় না, `undefined` চলে আসে | এমন ভুল হওয়ার সুযোগই নেই |
| Error | `Error` | `AppError` (500) |

> `const loadEnvVariables = (): EnvConfig => ({ ... })` — arrow function-এ `{ }`-কে `( )` দিয়ে মুড়লে সরাসরি object return হয়, `return` লিখতে হয় না।

### Nested group

যেসব variable একসাথে ব্যবহার হয়, সেগুলো group করা হয়েছে:

```ts
envVars.EMAIL_SENDER.SMTP_HOST;
envVars.CLOUDINARY.CLOUDINARY_API_KEY;
envVars.STRIPE.STRIPE_WEBHOOK_SECRET;
```

`.env` file-এ কিন্তু সব flat থাকে (`SMTP_HOST=...`)। Group শুধু code-এর ভেতরে।

### কোন variable কোন নোটে যোগ হয়েছে

| Variable | কাজ | নোট |
| --- | --- | --- |
| `NODE_ENV`, `PORT`, `DATABASE_URL` | Environment, port, DB connection | ০৬ (এই নোট) |
| `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL` | Better Auth | [০৫](./05-better-auth-setup-and-auth-schemas.md) |
| `ACCESS_TOKEN_*`, `REFRESH_TOKEN_*`, `BETTER_AUTH_SESSION_TOKEN_*` | JWT আর session-এর secret ও মেয়াদ | [১৩](./13-jwt-token-cookie-and-check-auth.md) |
| `FRONTEND_URL` | CORS, email link, PDF footer | — |
| `COOKIE_DOMAIN` (optional) | production-এ cookie-র domain (`.arifuddincoder.site`); `getEnv` না, সরাসরি `process.env` কারণ local-এ নেই | [client ২৮](../client/28-vps-deploy-and-cookie-domain.md) |
| `EMAIL_SENDER.*` | Nodemailer SMTP | [২০](./20-nodemailer-verify-email.md) |
| `GOOGLE_*` | Google OAuth | [২২](./22-google-oauth.md) |
| `CLOUDINARY.*` | File upload | [২৩](./23-multer-single-image-upload.md) |
| `STRIPE.*` | Payment ও webhook | [২৮](./28-stripe-payment.md) |
| `SUPER_ADMIN_*` | Super admin seed | [২৯](./29-seed-super-admin.md) |

**ব্যবহার:**

```ts
import { envVars } from "@/app/config/env";

const PORT = Number(envVars.PORT);
```

---

## ধাপ ৩: http-status package

Package: https://www.npmjs.com/package/http-status

```shell
pnpm add http-status
```

```ts
import httpStatus from "http-status";
```

**কেন?** `201`, `404`, `500` এর মতো number মনে রাখার বদলে নাম দিয়ে লেখা যায় — code পড়তে সহজ:

| Number | http-status |
|---|---|
| `200` | `httpStatus.OK` |
| `201` | `httpStatus.CREATED` |
| `400` | `httpStatus.BAD_REQUEST` |
| `401` | `httpStatus.UNAUTHORIZED` |
| `403` | `httpStatus.FORBIDDEN` |
| `404` | `httpStatus.NOT_FOUND` |
| `500` | `httpStatus.INTERNAL_SERVER_ERROR` |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`.env.example` update করো** — তোমার `.env.example`-এ এখন শুধু `PORT` আর `DATABASE_URL` আছে, কিন্তু app চালাতে ৩০টার মতো variable লাগে। নতুন machine-এ clone করলে কোনগুলো লাগবে বোঝা যাবে না। Value ছাড়া সব নাম রাখো:
   ```dotenv
   NODE_ENV=development
   PORT=5000
   DATABASE_URL=
   BETTER_AUTH_SECRET=
   BETTER_AUTH_URL=
   ACCESS_TOKEN_SECRET=
   REFRESH_TOKEN_SECRET=
   ACCESS_TOKEN_EXPIRES_IN=
   REFRESH_TOKEN_EXPIRES_IN=
   BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN=
   BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE=
   FRONTEND_URL=
   SMTP_HOST=
   SMTP_PORT=
   SMTP_USER=
   SMTP_PASS=
   SMTP_FROM=
   GOOGLE_CLIENT_ID=
   GOOGLE_CLIENT_SECRET=
   GOOGLE_CALLBACK_URL=
   CLOUDINARY_CLOUD_NAME=
   CLOUDINARY_API_KEY=
   CLOUDINARY_API_SECRET=
   STRIPE_SECRET_KEY=
   STRIPE_WEBHOOK_SECRET=
   SUPER_ADMIN_EMAIL=
   SUPER_ADMIN_PASSWORD=
   ```

2. **`NODE_ENV`-এর type আরও নির্দিষ্ট করো** — এখন যেকোনো string চলে, তাই `"develop"` লিখলে `isDev` চুপচাপ `false` হয়ে যাবে:
   ```ts
   NODE_ENV: "development" | "production";
   ```
   তবে তখন `getEnv`-এর `string`-কে check করে এই type-এ আনতে হবে।

3. **`prisma.ts`-এ `import "dotenv/config"` আর লাগে না** — `envVars` import করলেই `env.ts`-এর `dotenv.config()` চলে যায়। দুইবার load করলে ক্ষতি নেই, শুধু অপ্রয়োজনীয়। (`prisma7.config.ts` app-এর বাইরে চলে, ওখানে রেখে দাও।)

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **Missing env-এ `AppError` না দিলেও চলে** — `AppError` আসলে HTTP response-এর জন্য, কিন্তু এই error হয় server চালু হওয়ার সময়, তখন কোনো request নেই। Status code `500`-এর কোনো মানে নেই, শুধু message-টা কাজে লাগে। সাধারণ `throw new Error(...)` দিলেও একই কাজ হবে। এতে `env.ts`-এর সাথে `AppError`-এর কোনো সম্পর্কও থাকবে না।

5. **Fail fast** — কোনো variable না থাকলে server **চালুই হবে না**। ভুল config নিয়ে server চলার চেয়ে শুরুতেই পরিষ্কার message দিয়ে থেমে যাওয়া ভালো।

6. **`zod` দিয়ে env validation** — এখনও variable-এর নাম দুই জায়গায় লিখতে হয় (interface আর `getEnv`)। `zod` দিয়ে একবার লিখলেই check আর type দুটোই হয়, আর `PORT` auto number হয়ে যায়:
   ```ts
   const envSchema = z.object({
   	NODE_ENV: z.enum(["development", "production"]),
   	PORT: z.coerce.number(),
   	DATABASE_URL: z.url(),
   });
   export const envVars = envSchema.parse(process.env);
   ```

### ✅ (Done)

- **`interface` আর `return` না মেলা** — এখন সব variable দুই জায়গাতেই আছে, comment করা কিছু নেই।
- **`process.env.X as string`** — `getEnv` দিয়ে বদলানো হয়েছে, `as` আর লাগে না।
- **সব জায়গায় `envVars`** — `server.ts` আর `prisma.ts` এখন `envVars` ব্যবহার করে।
- **`PORT` number** — `server.ts`-এ `Number(envVars.PORT)` ([নোট ৩৭](./37-handle-server-error.md))।
- **http-status ব্যবহার** — সব controller/service-এ `httpStatus.CREATED` ইত্যাদি।
