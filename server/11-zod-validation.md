# 11 - Zod Validation

> Module: 38-05, 38-07

```
Request → route → validateRequest(zodSchema) → ✅ controller → service → DB
                                             → ❌ next(error) → globalErrorHandler (DB-তে hit-ই হয় না)
```

Docs: https://zod.dev/

---

## Zod validation কেন দরকার?

**১. ভুল data DB পর্যন্ত যায় না**
Zod না থাকলে request সরাসরি DB-তে hit করে, তারপর DB-র দিক থেকে error আসে। Zod থাকলে request আগে Zod-এর কাছে যায়, data-তে সমস্যা থাকলে DB-তে hit না করেই সাথে সাথে error ফেরত দেয়।

**২. Data sanitize করে**
হাত sanitize করলে যেমন জীবাণু মরে যায়, Zod-ও তেমনি অপ্রয়োজনীয় data বাদ দেয়। যেমন patient-এর `experience` লাগে না, কেউ request-এ পাঠালেও schema-তে না থাকলে Zod সেটা ফেলে দেয়। শুধু যে field-গুলো আমরা চাই সেগুলোই সামনে যায়, বাকিগুলো DB পর্যন্ত যেতেই পারে না।

| | Zod ছাড়া | Zod সহ |
|---|---|---|
| ভুল data | DB-তে hit করে, তারপর error | DB-তে যাওয়ার আগেই error |
| বাড়তি field | controller/service পর্যন্ত চলে যায় | বাদ পড়ে যায় (strip) |
| Error message | Prisma/DB-র অস্পষ্ট message | নিজের লেখা পরিষ্কার message |

---

## ধাপ ১: Zod install (38-05)

```shell
pnpm add zod
```

- Postman বা frontend থেকে যে data আসে, Zod দিয়ে সেটা validate করব।
- Request প্রথমে **route** file-এ আসে, তারপর controller-এ যায়। তাই controller-এর **আগে** Zod middleware বসাব।
- Zod-এ সব field **default-ভাবে required**। Optional চাইলে `.optional()` বলে দিতে হবে।

## ধাপ ২: `validateRequest` middleware (38-07)

Try/catch-এর জন্য যেমন `catchAsync` higher-order function বানিয়েছিলাম, Zod validation-এর জন্যও তেমন একটা বানাব।

`src/app/middleware/validateRequest.ts`

```ts
import type { NextFunction, Request, Response } from "express";
import type { ZodType } from "zod";

export const validateRequest = (zodSchema: ZodType) => {
	return async (req: Request, res: Response, next: NextFunction) => {
		const parsedResult = await zodSchema.safeParseAsync(req.body);

		if (!parsedResult.success) {
			return next(parsedResult.error);
		}

		// data sanitize করা হচ্ছে (schema-র বাইরের field বাদ, trim/lowercase করা data)
		req.body = parsedResult.data;
		next();
	};
};
```

- `safeParseAsync` error throw করে না, `{ success, data }` বা `{ success, error }` ফেরত দেয়।
- Fail হলে `next(error)` → error সরাসরি `globalErrorHandler`-এ যায়।
- Success হলে `req.body`-কে parsed data দিয়ে replace করি, তাই controller পরিষ্কার data পায়।

## ধাপ ৩: Create doctor-এর Zod schema

`src/app/module/user/user.validation.ts`

```ts
import { Gender } from "@/generated/prisma/enums";
import * as z from "zod";

export const createDoctorZodSchema = z.object({
	password: z
		.string("Password is required")
		.min(6, "Password must be at least 6 characters")
		.max(32, "Password must be at most 32 characters"),

	doctor: z.object({
		name: z
			.string("Name is required and must be a string")
			.trim()
			.min(3, "Name must be at least 3 characters")
			.max(30, "Name must be at most 30 characters"),

		email: z.email("Invalid email address").toLowerCase(),

		contactNumber: z
			.string("Contact number is required")
			.trim()
			.regex(/^(\+?880|0)1[3-9]\d{8}$/, "Invalid Bangladeshi phone number"),

		address: z
			.string("Address must be a string")
			.trim()
			.min(10, "Address must be at least 10 characters")
			.max(100, "Address must be at most 100 characters")
			.optional(),

		registrationNumber: z
			.string("Registration number is required")
			.trim()
			.min(1, "Registration number cannot be empty"),

		experience: z.int("Experience must be an integer").nonnegative("Experience cannot be negative").optional(),

		gender: z.enum(Gender, "Gender must be either MALE or FEMALE"),

		appointmentFee: z.number("Appointment fee must be a number").nonnegative("Appointment fee cannot be negative"),

		qualification: z
			.string("Qualification is required")
			.trim()
			.min(2, "Qualification must be at least 2 characters")
			.max(50, "Qualification must be at most 50 characters"),

		currentWorkingPlace: z
			.string("Current working place is required")
			.trim()
			.min(2, "Current working place must be at least 2 characters")
			.max(50, "Current working place must be at most 50 characters"),

		designation: z
			.string("Designation is required")
			.trim()
			.min(2, "Designation must be at least 2 characters")
			.max(50, "Designation must be at most 50 characters"),
	}),

	specialties: z
		.array(z.uuid("Each specialty must be a valid UUID"), "Specialties must be an array")
		.min(1, "At least one specialty is required")
		.refine((ids) => new Set(ids).size === ids.length, "Duplicate specialties are not allowed"),
});
```

কয়েকটা জিনিস বুঝে রাখি (Zod v4):

| Code | কী করে |
|---|---|
| `z.string("msg")` | string না হলে / না পাঠালে এই message |
| `.trim()` / `.toLowerCase()` | validate-এর সাথে data-ও বদলে দেয় (sanitize) |
| `z.email()`, `z.uuid()`, `z.int()` | v4-এর top-level format (`z.string().email()` এর নতুন রূপ) |
| `z.enum(Gender)` | Prisma enum সরাসরি দেওয়া যায় |
| `.regex(...)` | বাংলাদেশি নম্বর: `+880` / `880` / `0` দিয়ে শুরু, তারপর `1[3-9]` + ৮ digit |
| `.refine(...)` | নিজের custom rule, এখানে duplicate specialty id আটকানো |

## ধাপ ৪: Route-এ middleware বসানো

`user.route.ts`

```ts
import { validateRequest } from "@/app/middleware/validateRequest";
import { createDoctorZodSchema } from "./user.validation";

router.post("/create-doctor", validateRequest(createDoctorZodSchema), UserController.createDoctor);
```

### Test

| Method | URL |
|---|---|
| `POST` | `http://localhost:5000/api/v1/user/create-doctor` |

Note 10-এর body দিয়ে পাঠালে আগের মতোই doctor তৈরি হবে। এবার ভুল data পাঠিয়ে দেখো, যেমন:

```json
{
  "password": "123",
  "doctor": { "name": "Dr", "email": "not-an-email", "gender": "OTHER" },
  "specialties": []
}
```

→ DB-তে hit না করেই validation error আসবে।

## ধাপ ৫: Update-এর schema → `.partial()`

`src/app/module/doctor/doctor.validation.ts`

```ts
import * as z from "zod";

export const updateDoctorZodSchema = z.object({}).partial()
```

Update-এর সময় সব field লাগে না, তাই `.partial()` দিই। এতে সব field optional হয়ে যায়।

> এখানে `z.object({})` এখনো খালি (placeholder)। পরে update-এর field যোগ হবে।

---

## Zod কোথায় কোথায় লাগে?

- সাধারণত **create** আর **update**-এ, যখন `POST` / `PUT` / `PATCH` request-এর **body** দিয়ে data পাঠাই।
- File upload থাকলে file-ও Zod দিয়ে validate করা যায়।
- শুধু `req.body` না, আরও validate করা যায়:

| কী | উদাহরণ |
|---|---|
| `req.params` | id দিয়ে single patient get / delete |
| `req.query` | search, filter, page, limit |
| `req.cookies` | token ইত্যাদি |
| env variables | `.env`-এর সব value ঠিক আছে কিনা (server start-এর সময়) |

---

## 🔜 পরে যা যোগ হবে

- [ ] Zod error handler → `globalErrorHandler`-এ `ZodError` আলাদা করে সুন্দর response (38-07-এর শিরোনামে আছে, কিন্তু code নোটে ছিল না; note 09-এর checklist-এও আছে)
- [ ] `updateDoctorZodSchema`-এ আসল field যোগ করা
- [ ] `req.params` / `req.query` validation
- [ ] env variables Zod দিয়ে validate (note 06)
- [ ] File (profile photo) validation

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **খালি `z.object({})` দিয়ে validate করলে body পুরো ফাঁকা হয়ে যাবে** — Zod default-ভাবে schema-র বাইরের সব field ফেলে দেয়। `updateDoctorZodSchema` এখন যেমন আছে, সেটা route-এ বসালে `req.body` হয়ে যাবে `{}`, আর update কিছুই হবে না। Route-এ বসানোর আগে field যোগ করো:
   ```ts
   export const updateDoctorZodSchema = z
   	.object({
   		name: z.string().trim().min(3).max(30),
   		profilePhoto: z.url(),
   		contactNumber: z.string().trim().regex(/^(\+?880|0)1[3-9]\d{8}$/),
   		address: z.string().trim().min(10).max(100),
   		experience: z.int().nonnegative(),
   	})
   	.partial();
   ```

2. **Zod error handler এখনো নেই (সম্ভবত)** — note 09-এর simple handler `ZodError` পেলে সম্ভবত 500 আর একটা লম্বা JSON string message দেখাবে। 400 status আর field-wise message দিলে frontend-এর জন্য সহজ হবে:
   ```ts
   import { ZodError } from "zod";

   if (err instanceof ZodError) {
   	statusCode = httpStatus.BAD_REQUEST;
   	message = "Validation error";
   	errorSources = err.issues.map((issue) => ({
   		path: issue.path.join("."),
   		message: issue.message,
   	}));
   }
   ```

### 🟡 ভালো practice

3. **Unused `res` → `_res`** — `validateRequest`-এ `res` ব্যবহার হয়নি। oxlint unused args-এ `_` দিয়ে শুরু হলে warning দেয় না।
   ```ts
   return async (req: Request, _res: Response, next: NextFunction) => { ... };
   ```

4. **`contactNumber` Zod-এ required, কিন্তু Prisma ও `ICreateDoctorPayload`-এ optional** — যেকোনো এক নিয়ম রাখো। Required রাখতে চাইলে interface-এও `contactNumber: string` করো, নইলে Zod-এ `.optional()` দাও।

5. **Interface হাতে না লিখে Zod থেকে type বানাও** — দুই জায়গায় একই জিনিস লিখলে একসময় মিল থাকবে না (৪ নম্বরের মতো):
   ```ts
   export type ICreateDoctorPayload = z.infer<typeof createDoctorZodSchema>;
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

6. **`profilePhoto` schema-তে নেই, তাই strip হয়ে যাবে** — কেউ body-তে `profilePhoto` পাঠালেও Zod ফেলে দেবে, DB-তে যাবে না। সম্ভবত পরে file upload দিয়ে আসবে, তাই এখন ঠিক আছে।

7. **`z.uuid()` কড়া check করে** — RFC অনুযায়ী version/variant bit-ও দেখে। Prisma-র `uuid(7)` আর সাধারণ v4 id দুটোই pass করবে, কিন্তু হাতে বানানো ভুয়া id (যেমন `11111111-1111-1111-1111-111111111111`) fail করবে। ঢিলা check লাগলে `z.guid()` আছে।

8. **Unknown field-এ error দিতে চাইলে `z.strictObject()`** — default-এ বাড়তি field চুপচাপ বাদ পড়ে। বাড়তি field পাঠালে error দিতে চাইলে `z.object` এর জায়গায় `z.strictObject` ব্যবহার করো।
