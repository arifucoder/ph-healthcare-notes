## Install
```shell
pnpm add better-auth jsonwebtoken cookie-parser
```

In env.ts 
```ts
const DURATION_REGEX = /^\d+(?:\.\d+)?\s?(?:ms|s|m|h|d|w|y)$/i;

const isValidTimeZone = (timeZone: string): boolean => {
	try {
		new Intl.DateTimeFormat("en-US", { timeZone }).format();
		return true;
	} catch {
		return false;
	}
};
const secret = z.string().min(32, "Must be at least 32 characters (generate: openssl rand -base64 32)");
const duration = z.string().regex(DURATION_REGEX, 'Must be a duration like "15m", "1d" or "7d"');
const envSchema = z
	.object({
		CLINIC_TIME_ZONE: z.string().refine(isValidTimeZone, "Must be a valid IANA time zone, e.g. Asia/Dhaka"),
		CLINIC_NAME: z.string().trim().min(1).default("ClinicOS"),

		BETTER_AUTH_SECRET: secret,
		BETTER_AUTH_URL: z.url({ protocol: /^https?$/ }),
		ACCESS_TOKEN_SECRET: secret,
		ACCESS_TOKEN_EXPIRES_IN: duration,
		REFRESH_TOKEN_SECRET: secret,
		REFRESH_TOKEN_EXPIRES_IN: duration,

		COOKIE_DOMAIN: z
			.string()
			.regex(/^\.?[a-z0-9-]+(\.[a-z0-9-]+)+$/i, 'Must be a domain like ".yourdomain.com"')
			.optional(),
	})
	.superRefine((env, ctx) => {
		if (env.ACCESS_TOKEN_SECRET === env.REFRESH_TOKEN_SECRET) {
			ctx.addIssue({
				code: "custom",
				path: ["REFRESH_TOKEN_SECRET"],
				message: "Must be different from ACCESS_TOKEN_SECRET",
			});
		}
		if (env.NODE_ENV === "production" && !env.BETTER_AUTH_URL.startsWith("https://")) {
			ctx.addIssue({
				code: "custom",
				path: ["BETTER_AUTH_URL"],
				message: "Must use https in production",
			});
		}
	});
```

In .env file
```dotenv
# ── Clinic ───────────────────────────────────
# IANA time zone that defines the clinic's "day" (all times stored in UTC)
CLINIC_TIME_ZONE=Asia/Dhaka
CLINIC_NAME=ClinicOS Demo Clinic

# ── Auth (Better Auth + JWT cookies) ─────────
# Generate each secret with: openssl rand -base64 32
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=http://localhost:5000
ACCESS_TOKEN_SECRET=
ACCESS_TOKEN_EXPIRES_IN=1d
REFRESH_TOKEN_SECRET=
REFRESH_TOKEN_EXPIRES_IN=7d

# Production only, e.g. .yourdomain.com (leave empty for localhost)
COOKIE_DOMAIN=
```

ধাপ ৩: Auth instance বানানো — src/app/lib/auth.ts
```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { prisma } from "./prisma";

export const auth = betterAuth({
	database: prismaAdapter(prisma, {
		provider: "postgresql",
	}),

	emailAndPassword: {
		enabled: true,
	},
});
```

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


ধাপ ৭: Enums — prisma/schema/enums.prisma
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

ধাপ ৮: User model এ Custom Field — auth.prisma

```prisma
model User {
  id                 String    @id
  name               String
  email              String
  emailVerified      Boolean   @default(false)
  image              String?
  role               Role      @default(PATIENT)
  isActive           Boolean   @default(true)
  needPasswordChange Boolean   @default(false)
  isDeleted          Boolean   @default(false)
  deletedAt          DateTime?
  createdAt          DateTime  @default(now())
  updatedAt          DateTime  @updatedAt

  sessions Session[]
  accounts Account[]

  @@unique([email])
  @@map("user")
}
```
tarpor
```shell
pnpm migrate && pnpm generate
```