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

## ধাপ ২: env.ts

```ts
import dotenv from "dotenv";
// import AppError from "../errorHelpers/AppError";

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
}

const loadEnvVariables = (): EnvConfig => {
    // ---- শুরু: Required variable check ----
    // .env file এ নিচের সব variable আছে কিনা check করা হচ্ছে।
    // কোনোটা না থাকলে app চালু হওয়ার সময়েই পরিষ্কার error message দেবে —
    // কোন variable টা missing সেটা সরাসরি বলে দেবে, পরে অজানা bug খুঁজতে হবে না।
    const requireEnvVariable = [
        "NODE_ENV",
        "PORT",
        "DATABASE_URL",
        "BETTER_AUTH_SECRET",
        "BETTER_AUTH_URL",
        // "ACCESS_TOKEN_SECRET",
        // "REFRESH_TOKEN_SECRET",
        // "ACCESS_TOKEN_EXPIRES_IN",
        // "REFRESH_TOKEN_EXPIRES_IN",
        // "BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN",
        // "BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE",
    ];

    requireEnvVariable.forEach((variable) => {
        if (!process.env[variable]) {
            throw new Error(`Environment variable ${variable} is required but not set in .env file.`);
            // throw new AppError(
            //  status.INTERNAL_SERVER_ERROR,
            //  `Environment variable ${variable} is required but not set in .env file.`,
            // );
        }
    });
    // ---- শেষ: Required variable check ----

    return {
        NODE_ENV: process.env.NODE_ENV as string,
        PORT: process.env.PORT as string,
        DATABASE_URL: process.env.DATABASE_URL as string,
        BETTER_AUTH_SECRET: process.env.BETTER_AUTH_SECRET as string,
        BETTER_AUTH_URL: process.env.BETTER_AUTH_URL as string,
        // ACCESS_TOKEN_SECRET: process.env.ACCESS_TOKEN_SECRET as string,
        // REFRESH_TOKEN_SECRET: process.env.REFRESH_TOKEN_SECRET as string,
        // ACCESS_TOKEN_EXPIRES_IN: process.env.ACCESS_TOKEN_EXPIRES_IN as string,
        // REFRESH_TOKEN_EXPIRES_IN: process.env.REFRESH_TOKEN_EXPIRES_IN as string,
        // BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN: process.env.BETTER_AUTH_SESSION_TOKEN_EXPIRES_IN as string,
        // BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE: process.env.BETTER_AUTH_SESSION_TOKEN_UPDATE_AGE as string,
    };
};

export const envVars = loadEnvVariables();

```

**কীভাবে কাজ করে:**
1. `dotenv.config()` → `.env` file থেকে সব variable `process.env` এ load করে
2. `interface EnvConfig` → কোন কোন variable থাকবে, তার type
3. `requireEnvVariable` loop → প্রতিটা required variable আছে কিনা check, না থাকলে `throw`
4. `return {...}` → সব variable একটা object এ সাজিয়ে দেয়
5. `export const envVars` → অন্য file এ `envVars.PORT`, `envVars.DATABASE_URL` এভাবে use করবো

> 📝 Comment করা variable গুলো (`ACCESS_TOKEN_*`, `REFRESH_TOKEN_*`, `BETTER_AUTH_SESSION_*`) পরে JWT / session এর কাজে লাগবে — তখন uncomment করবো।
> `AppError` এর অংশটাও পরে custom error class বানানোর পর use করবো।

**ব্যবহার:**

```ts
import { envVars } from "@/app/config/env";

const PORT = envVars.PORT;
```

---

## ধাপ ৩: http-status package

Package: https://www.npmjs.com/package/http-status

```shell
pnpm add http-status
```

**কেন?** `201`, `404`, `500` এর মতো number মনে রাখার বদলে নাম দিয়ে লেখা যায় — code পড়তে সহজ:

| Number | http-status |
|---|---|
| `200` | `status.OK` |
| `201` | `status.CREATED` |
| `400` | `status.BAD_REQUEST` |
| `401` | `status.UNAUTHORIZED` |
| `403` | `status.FORBIDDEN` |
| `404` | `status.NOT_FOUND` |
| `500` | `status.INTERNAL_SERVER_ERROR` |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

**১. `interface` আর `return` মিলছে না → TypeScript error**
`EnvConfig` interface এ `ACCESS_TOKEN_SECRET` সহ ৬টা field **required**, কিন্তু `return` এ সেগুলো comment করা। তাই `pnpm typecheck` / `pnpm build` এ error আসবে:
`Type '{...}' is missing the following properties from type 'EnvConfig': ACCESS_TOKEN_SECRET, ...`
(`pnpm dev` এ ধরা পড়ে না কারণ `tsx` type check করে না।)

Fix — interface এও ওগুলো comment করে দাও (পরে একসাথে uncomment করবে):

```ts
interface EnvConfig {
    NODE_ENV: string;
    PORT: string;
    DATABASE_URL: string;
    BETTER_AUTH_SECRET: string;
    BETTER_AUTH_URL: string;
    // ACCESS_TOKEN_SECRET: string;
    // ...
}
```

**২. "crash না করে error দেবে" — আসলে app বন্ধ হয়ে যাবে (এবং সেটাই ভালো)**
`throw new Error` দিলে server **চালু হবে না**। এটাকে বলে **fail fast** — ভুল config নিয়ে server চলার চেয়ে শুরুতেই থেমে যাওয়া ভালো। পার্থক্য হলো, এখন error message পরিষ্কার।

### 🟡 ভালো practice

**৩. সব জায়গায় `envVars` use করো**
এখন `prisma.ts`, `server.ts`, `prisma7.config.ts` এ আলাদা আলাদা `dotenv/config` + `process.env` আছে। App এর ভেতরে (`src/`) সব জায়গায় `envVars` use করলে একটাই source থাকবে:

```ts
// server.ts
const PORT = Number(envVars.PORT);

// prisma.ts
const adapter = new PrismaPg({ connectionString: envVars.DATABASE_URL });
```

(`prisma7.config.ts` app এর বাইরে চলে — ওখানে `dotenv/config` থাকুক।)

**৪. `PORT` কে number বানাও**
`.env` থেকে সব `string` আসে। `app.listen` এ number দেওয়াই ঠিক — `Number(envVars.PORT)`।

**৫. `NODE_ENV` এর type আরও নির্দিষ্ট করো**

```ts
NODE_ENV: "development" | "production";
```

এতে ভুল বানান (`"prod"`) ধরা পড়বে।

**৬. `.env.example` file রাখো**
`.env` git এ যায় না, তাই নতুন কেউ (বা নতুন machine এ তুমি) জানবে না কোন variable লাগবে। Value ছাড়া একটা `.env.example` commit করো:

```dotenv
NODE_ENV=
PORT=
DATABASE_URL=
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=
```

### 🔵 পরে কাজে লাগবে

**৭. `zod` দিয়ে env validation (advanced)**
এখনকার পদ্ধতিতে variable name তিন জায়গায় লিখতে হয় (interface, array, return)। `zod` দিয়ে একবার লিখলেই check + type দুটোই হয়ে যায়, আর `PORT` এর মতো value auto number এ convert করা যায়:

```ts
const envSchema = z.object({
    NODE_ENV: z.enum(["development", "production"]),
    PORT: z.coerce.number(),
    DATABASE_URL: z.string().url(),
});
export const envVars = envSchema.parse(process.env);
```

**৮. http-status দিয়ে status code লেখো**
Controller গুলোতে `httpStatusCode: 201` এর বদলে `httpStatusCode: status.CREATED` লিখলে পড়তে সহজ হবে।
