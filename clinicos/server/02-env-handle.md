# ClinicOS নোট: Environment config

## ১. Environment variable validation

**File:** `src/app/config/env.ts` (একটাই ফাইল)

```ts
import * as z from "zod";

const envSchema = z.object({
	NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
	PORT: z.coerce.number().int().min(1).max(65535).default(5000),
	DATABASE_URL: z.url({ protocol: /^postgres(ql)?$/ }),
});

const rawEnv = Object.fromEntries(Object.entries(process.env).filter(([, value]) => value !== ""));

const parsed = envSchema.safeParse(rawEnv);

if (!parsed.success) {
	console.error(`Invalid environment variables:\n${z.prettifyError(parsed.error)}`);
	process.exit(1);
}

export const envVars = parsed.data;
export const isProduction = envVars.NODE_ENV === "production";
```