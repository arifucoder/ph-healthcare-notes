# 05 - Setting Up Better Auth & Creating Auth Schemas

> Module: 37-11
> Doc: https://better-auth.com/docs/installation (installation page ধাপে ধাপে follow করবো)

**পুরো flow এক নজরে:**

```
Install → .env → auth.ts → CLI দিয়ে schema generate → migrate
→ custom field (enum + User) → auth.ts এ additionalFields
→ auth module (route/controller/service) → routes/index.ts এ যোগ
```

---

## ধাপ ১: Install

```shell
pnpm add better-auth
```

---

## ধাপ ২: Environment Variables (.env)

**Secret:** Better Auth এর secret generate করে `.env` এ রাখো (doc page এ generate button আছে, অথবা `openssl rand -base64 32`)।

```dotenv
BETTER_AUTH_SECRET=better_auth_secreat
```

**Base URL:** আমাদের backend/server এর URL।

```dotenv
BETTER_AUTH_URL=http://localhost:5000 # Base URL of your app
```

---

## ধাপ ৩: Auth instance বানানো — `src/app/lib/auth.ts`

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "./prisma";

export const auth = betterAuth({
    database: prismaAdapter(prisma, {
        provider: "postgresql", // or "mysql", "sqlite", ...etc
    }),

    // amader k email password true korte hobe email password registration er jonno
    emailAndPassword: {
        enabled: true,
    },
});
```

- `prismaAdapter` → Better Auth আমাদের Prisma দিয়েই DB তে data রাখবে
- `emailAndPassword.enabled` → email + password দিয়ে registration/login চালু

---

## ধাপ ৪: CLI দিয়ে Auth Schema Generate

Doc: https://better-auth.com/docs/concepts/cli#generate

Better Auth নিজে **৪টা model** generate করে দেয় (`User`, `Session`, `Account`, `Verification`)।

দুইটা গুরুত্বপূর্ণ option:

| Option | কেন লাগছে |
|---|---|
| `--output` | Default এ model গুলো `schema.prisma` তে লেখে। আমরা **multi-file schema** use করি, তাই বলে দিচ্ছি `prisma/schema/auth.prisma` file এ লিখতে। |
| `--config` | Better Auth default এ root / `lib` folder এ `auth.ts` খোঁজে। আমাদের file আছে `src/app/lib/auth.ts` এ, তাই path বলে দিতে হয়। |

আগে `prisma/schema/auth.prisma` নামে একটা খালি file বানিয়ে নাও, তারপর:

```shell
pnpm dlx auth@latest generate \
  --output ./prisma/schema/auth.prisma \
  --config ./src/app/lib/auth.ts \
  --adapter prisma
```

---

## ধাপ ৫: Migrate + Generate

```shell
pnpm migrate
pnpm generate
```

> 💡 এগুলো `package.json` এর scripts এ আগেই set করা আছে ([02-prisma-setup](./02-prisma-setup.md) এ বানিয়েছিলাম):
>
> ```json
> "migrate": "prisma migrate dev",
> "generate": "prisma generate",
> ```
>
> অর্থাৎ `pnpm migrate` = `prisma migrate dev`, `pnpm generate` = `prisma generate`

> ⚠️ **`pnpm dlx auth@latest migrate` চালাবো না!**
> ওটা শুধু **Kysely adapter** এর জন্য। আমরা Prisma use করি, তাই আমাদের নিজের `pnpm migrate` ই চালাতে হবে।

---

## ধাপ ৬: Mount Handler (কেন use করছি না)

Doc: https://better-auth.com/docs/installation#mount-handler

- Mount handler দিয়ে করলে Better Auth এর **default route pattern** follow করতে হয় (Express এর জন্যও setting আছে)।
- এভাবে করলে কিছু **catch / limitation** আছে — নিজের মতো route, validation, response format control করা কঠিন।

👉 তাই আমরা Better Auth এর **built-in route না**, বরং **built-in function** (`auth.api.signUpEmail`, `auth.api.signInEmail`) use করে **নিজের route** বানাবো (ধাপ ১০)।

---

## ধাপ ৭: Enums — `prisma/schema/enums.prisma`

User এর role আর status এর জন্য enum লাগবে (অন্য schema তেও লাগতে পারে, তাই আলাদা file)।

```prisma
enum Role {
    SUPER_ADMIN
    ADMIN
    DOCTOR
    PATIENT
}

enum UserStatus {
    BLOCKED
    DELETED
    ACTIVE
}
```

---

## ধাপ ৮: User model এ Custom Field — `auth.prisma`

Generate হওয়া `auth.prisma` তে শুধু predefined field আছে। আমাদের আরও কিছু field লাগবে (`// updated` লেখা গুলো নতুন):

```prisma
model User {
  id                 String     @id
  name               String
  email              String
  emailVerified      Boolean    @default(false)
  role               Role       @default(PATIENT) // updated
  status             UserStatus @default(ACTIVE)// updated
  needPasswordChange Boolean    @default(false)// updated
  isDeleted          Boolean    @default(false)// updated
  deletedAt          DateTime?// updated
  image              String?
  createdAt          DateTime   @default(now())
  updatedAt          DateTime   @updatedAt
  sessions           Session[]
  accounts           Account[]

  @@unique([email])
  @@map("user")
}
```

তারপর আবার:

```shell
pnpm migrate && pnpm generate
```

---

## ধাপ ৯: auth.ts এ `additionalFields` যোগ

**কেন লাগে?** নতুন field গুলো শুধু Prisma জানে, **Better Auth জানে না**। তাই Better Auth কে বলে দিতে হয় — না হলে sign up / login এর response এ এই field গুলো আসবে না।

`emailAndPassword` এর নিচে `user.additionalFields` যোগ করো। পুরো file:

```ts
import { Role, UserStatus } from "@/generated/prisma/enums";
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "./prisma";

export const auth = betterAuth({
	database: prismaAdapter(prisma, {
		provider: "postgresql", // or "mysql", "sqlite", ...etc
	}),
	emailAndPassword: {
		enabled: true,
	},
	user: {
		additionalFields: {
			role: {
				type: "string",
				required: true,
				defaultValue: Role.PATIENT,
			},

			status: {
				type: "string",
				required: true,
				defaultValue: UserStatus.ACTIVE,
			},

			needPasswordChange: {
				type: "boolean",
				required: true,
				defaultValue: false,
			},

			isDeleted: {
				type: "boolean",
				required: true,
				defaultValue: false,
			},

			deletedAt: {
				type: "date",
				required: false,
				defaultValue: null,
			},
		},
	},
});

```

---

## ধাপ ১০: Auth Module — `src/app/module/auth/`

```
src/app/module/auth/
├── auth.route.ts
├── auth.controller.ts
└── auth.service.ts
```

### 10.1 auth.route.ts

```ts
import { Router } from "express";
import { AuthController } from "./auth.controller";

const router = Router()

router.post("/register", AuthController.registerPatient)
router.post("/login", AuthController.loginUser)

export const AuthRoutes = router;
```

### 10.2 auth.controller.ts

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import { AuthService } from "./auth.service";

const registerPatient = catchAsync(
    async (req: Request, res: Response) => {
        const payload = req.body;

        console.log(payload);

        const result = await AuthService.registerPatient(payload);

        sendResponse(res, {
            httpStatusCode: 201,
            success: true,
            message: "Patient registered successfully",
            data: result,
        })
    }
)

const loginUser = catchAsync(
    async (req: Request, res: Response) => {
        const payload = req.body;
        const result = await AuthService.loginUser(payload);
        sendResponse(res, {
            httpStatusCode: 200,
            success: true,
            message: "User logged in successfully",
            data: result,
        })
    }
)

export const AuthController = {
    registerPatient,
    loginUser,
};
```

### 10.3 auth.service.ts

- **Register:** শুধু `name`, `email`, `password` দিলেই patient register হবে (role default `PATIENT`)
- **Login:** যেকোনো role এর user login করতে পারবে, কিন্তু `BLOCKED` / `DELETED` হলে error

```ts
import { auth } from "@/app/lib/auth";
import { UserStatus } from "@/generated/prisma/enums";

interface IRegisterPatientPayload {
    name: string;
    email: string;
    password: string;
}

const registerPatient = async (payload: IRegisterPatientPayload) => {
    const { name, email, password } = payload;
    // amader name, email and password hoilei patient register korte parbe. 
    const data = await auth.api.signUpEmail({
        body: {
            name,
            email,
            password,
            //default values
            // needsPasswordChange: false,
            // role: Role.PATIENT
        }
    })

    if (!data.user) {
        throw new Error("Failed to register patient");
    }

    //TODO : Create Patient Profile In Transaction After Sign Up Of Patient In USer Model
    // amader patient jokhon registration korbe tokhon patient er j field gulo thakbe, data gulo thakbe like 
    // tratment history, address ei jinishgular jonno amader patient er nijosso ekta table lagbe seitateu data 
    // insert korte hobe, amara seita korbo nicher dike. 
    // const patient = await prisma.$transaction( async (tx) => {
    //     await tx.pa
    // })

    return data


}

interface ILoginUserPayload {
    email: string;
    password: string;
}

// j kono role er user login korte parbe 
const loginUser = async (payload: ILoginUserPayload) => {
    const { email, password } = payload;

    const data = await auth.api.signInEmail({
        body: {
            email,
            password,
        }
    })

    if (data.user.status === UserStatus.BLOCKED) {
        throw new Error("User is blocked");
    }

    if (data.user.isDeleted || data.user.status === UserStatus.DELETED) {
        throw new Error("User is deleted");
    }

    return data;

}

export const AuthService = {
    registerPatient,
    loginUser,
};
```

> 📝 **TODO (পরে করবো):** Patient register হলে তার নিজস্ব data (treatment history, address ইত্যাদি) এর জন্য আলাদা **Patient table** লাগবে। Sign up এর পর `prisma.$transaction` দিয়ে সেখানেও data insert করবো।

---

## ধাপ ১১: Route যোগ — `src/app/routes/index.ts`

```ts
router.use("/auth", AuthRoutes);
```

---

## API Endpoints

| Method | URL | কাজ | Body |
|---|---|---|---|
| POST | `/api/v1/auth/register` | Patient register | `name`, `email`, `password` |
| POST | `/api/v1/auth/login` | যেকোনো user login | `email`, `password` |

---

## Quick Recap

| কী | কোথায় |
|---|---|
| Auth instance | `src/app/lib/auth.ts` |
| Auth models (User, Session, Account, Verification) | `prisma/schema/auth.prisma` |
| Enums (Role, UserStatus) | `prisma/schema/enums.prisma` |
| Auth module | `src/app/module/auth/` |
| Migrate | `pnpm migrate` ✅ (`auth migrate` ❌) |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code কাজ করছে, তাই কিছু বদলানো হয়নি। সময় করে improve করতে পারো।

### 🔴 গুরুত্বপূর্ণ (Security)

**১. `console.log(payload)` সরিয়ে দাও (auth.controller.ts)**
Register এর payload এ **password** আছে — এটা terminal/log এ plain text এ print হচ্ছে। Production এ log এ password থাকা বড় security risk।

**২. Blocked/Deleted check — login এর আগে করো**
এখন `signInEmail` আগে চলে, তারপর status check হয়। মানে blocked user এর জন্যও **session তৈরি হয়ে DB তে save হয়ে যায়**, শুধু response এ error যায়।
Better: আগে `prisma.user.findUnique({ where: { email } })` দিয়ে status check করো, তারপর `signInEmail` call করো।

**৩. `additionalFields` এ `input: false` দাও**
`role`, `status`, `isDeleted` ইত্যাদি field যেন user নিজে sign up এর সময় পাঠিয়ে set করতে না পারে (যেমন নিজেকে `SUPER_ADMIN` বানানো):

```ts
role: {
	type: "string",
	required: true,
	defaultValue: Role.PATIENT,
	input: false, // user এটা পাঠাতে পারবে না
},
```

### 🟡 ভালো practice

**৪. ✅ (Done) `import` → `import type` (auth.controller.ts)**
`import { Request, Response } from "express"` — tsconfig এ `verbatimModuleSyntax: true` আছে, তাই `pnpm typecheck` / `pnpm build` এ error দিতে পারে (`pnpm dev` এ tsx type check করে না বলে ধরা পড়ে না)।

```ts
import type { Request, Response } from "express";
```

**৫. Error এর সঠিক status code**
এখন `throw new Error("User is blocked")` → `catchAsync` সবসময় **500** পাঠায়। কিন্তু এটা server error না:
- Blocked / Deleted → `403 Forbidden`
- ভুল password → `401 Unauthorized`

পরে একটা custom `AppError(statusCode, message)` class + global error handler বানালে এটা ঠিক হবে।

**৬. Request body validation**
`req.body` সরাসরি service এ যাচ্ছে। `zod` দিয়ে validate করলে ভুল data (খালি email, ছোট password) আগেই আটকানো যাবে।

**৭. ✅ (Done) Import style একই রাখো**
কোথাও `@/generated/...` (alias), কোথাও `../../../generated/...` (relative)। সব জায়গায় `@/` alias use করলে পড়তে সহজ, file move করলেও ভাঙবে না।

**৮. Comment এ নামের ভুল**
`auth.service.ts` এর comment এ `needsPasswordChange` লেখা, কিন্তু field এর নাম `needPasswordChange` — পরে uncomment করলে কাজ করবে না।

### 🔵 পরে কাজে লাগবে

**৯. Cookie / Session token**
`auth.api.signInEmail()` server থেকে call করলে browser এ **cookie set হয় না** — token শুধু response body তে আসে। Frontend (Next.js) এ cookie লাগলে `returnHeaders: true` বা `asResponse: true` option দেখো।

**১০. Patient profile transaction**
TODO তে `prisma.$transaction` এর কথা আছে, কিন্তু `signUpEmail` Prisma transaction এর **ভেতরে চলবে না** (Better Auth নিজের মতো DB call করে)। তাই:
sign up → patient create → patient create fail হলে **user টা delete করে দাও** (rollback এর মতো)।

**১১. Frontend connect করার সময়**
`auth.ts` এ `trustedOrigins: ["http://localhost:3000"]` লাগবে, আর `cors()` এ `origin` + `credentials: true` দিতে হবে।
