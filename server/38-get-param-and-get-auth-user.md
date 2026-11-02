# 38 - Shared Helper: `getParam` ও `getAuthUser`

`pnpm check` (typecheck + oxlint) চালানোর পরে controller-গুলোতে বারবার একই ধরনের lint/type error আসছিল। দুইটা জায়গায়: `req.params` আর `req.user`। তাই `src/app/shared/`-এ দুইটা ছোট helper বানানো হয়েছে, যাতে **cast (`as`) ছাড়াই** নিরাপদে `string` আর `IRequestUser` পাওয়া যায়।

```
controller → getParam(req, "id")  → string না হলে 400  → string
controller → getAuthUser(req)     → user না থাকলে 401  → IRequestUser
```

| Helper | কী ঠিক করে | না থাকলে |
| --- | --- | --- |
| `getParam(req, key)` | `req.params[key]`-এর type `string \| string[] \| undefined` → `string` | `400 Bad Request` |
| `getAuthUser(req)` | `req.user`-এর type `IRequestUser \| undefined` → `IRequestUser` | `401 Unauthorized` |

---

## সমস্যাটা কী ছিল?

### ১. `req.params` সবসময় `string` না

Express 5-এর type-এ (`@types/express-serve-static-core`):

```ts
export interface ParamsDictionary {
	[key: string]: string | string[];
}
```

Express 5-এ wildcard route (`/files/*path`) থাকলে param একটা **array** হয়, তাই type-এ `string[]`-ও আছে। আবার আমাদের tsconfig-এ `noUncheckedIndexedAccess: true`, তাই `req.params.id`-এর আসল type দাঁড়ায় `string | string[] | undefined`।

Service চায় `string`, তাই আগে এভাবে লেখা হতো:

```ts
// ❌ আগে
const prescriptionId = req.params.id as string;
```

`as` দিয়ে TypeScript-কে জোর করে বলা হচ্ছে "এটা string", কিন্তু runtime-এ কোনো check হয় না। Type-aware oxlint (`typescript/no-unsafe-type-assertion`) এটাকে error হিসেবে ধরে।

### ২. `req.user` optional

`src/app/interfaces/express.d.ts`-এ:

```ts
declare global {
	namespace Express {
		interface Request {
			user?: IRequestUser; // optional, কারণ সব route-এ checkAuth থাকে না
		}
	}
}
```

তাই service-এ `req.user` পাঠালে type error আসে (`IRequestUser | undefined` দেওয়া যায় না `IRequestUser`-এ)। `req.user!` বা `req.user as IRequestUser` দিলে আবার lint ধরে। তাই প্রতিটা controller-এ আলাদা করে `if (!req.user) throw ...` লেখা হচ্ছিল, বা প্রতিটা controller file-এ একই `getAuthUser` function কপি করা হচ্ছিল।

---

## ধাপ ১: `src/app/shared/getParam.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { Request } from "express";
import httpStatus from "http-status";

// req.params-এর type string | string[]; runtime-এ check করে string নিশ্চিত করি
export const getParam = (req: Request, key: string): string => {
	const value = req.params[key];

	if (typeof value !== "string" || !value) {
		throw new AppError(httpStatus.BAD_REQUEST, `Route parameter "${key}" is required`);
	}

	return value;
};
```

- `typeof value !== "string"` → `undefined` বা `string[]` হলে বাদ।
- `!value` → খালি string (`""`) হলেও বাদ।
- এই check-এর পরে TypeScript নিজেই বোঝে `value` এখন `string` (**type narrowing**), তাই `as` লাগে না।

---

## ধাপ ২: `src/app/shared/getAuthUser.ts`

```ts
import AppError from "@/app/errorHelpers/AppError";
import type { IRequestUser } from "@/app/interfaces/requestUser.interface";
import type { Request } from "express";
import httpStatus from "http-status";

// req.user optional — check করে narrow করি, না থাকলে 401
export const getAuthUser = (req: Request): IRequestUser => {
	if (!req.user) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
	}
	return req.user;
};
```

- `checkAuth` ঠিকমতো চললে `req.user` সবসময় থাকবে, তাই এই `401` সাধারণত আসে না।
- কিন্তু কেউ route-এ `checkAuth` দিতে ভুলে গেলে crash না হয়ে পরিষ্কার `401` আসবে।

---

## ধাপ ৩: Controller-এ ব্যবহার

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { getAuthUser } from "@/app/shared/getAuthUser";
import { getParam } from "@/app/shared/getParam";
import { sendResponse } from "@/app/shared/sendResponse";
import type { Request, Response } from "express";
import httpStatus from "http-status";
import { PrescriptionService } from "./prescription.service";

const getSinglePrescription = catchAsync(async (req: Request, res: Response) => {
	const user = getAuthUser(req);
	const prescriptionId = getParam(req, "id");
	const result = await PrescriptionService.getSinglePrescription(user, prescriptionId);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Prescription retrieved successfully",
		data: result,
	});
});
```

### আগে বনাম এখন

| | আগে | এখন |
| --- | --- | --- |
| Param | `req.params.id as string` | `getParam(req, "id")` |
| User | `if (!req.user) throw ...` প্রতিটা function-এ, বা প্রতিটা file-এ নিজস্ব `getAuthUser` | `getAuthUser(req)` (shared থেকে import) |
| Runtime check | নেই (`as` শুধু TypeScript-কে চুপ করায়) | আছে — ভুল হলে `400`/`401` |
| Lint | `no-unsafe-type-assertion` error | পরিষ্কার |
| একই code কয় জায়গায় | প্রতিটা controller-এ | একটাই file |

> `throw` করা error `catchAsync` ধরে `globalErrorHandler`-এ পাঠায়, আর সেখানে `AppError` branch status code দিয়ে response দেয় ([নোট ১২](./12-global-error-handler.md))।

---

## 🧪 Test

```shell
pnpm check
```

`as string` বা `req.user!`-এর lint error আর আসবে না।

| Case | Expected |
| --- | --- |
| `GET /api/v1/prescriptions/<id>` (login করে) | `200` |
| Route-এ `checkAuth` না দিয়ে `getAuthUser` ব্যবহার | `401` "Unauthorized access! Please log in." |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **সব controller-এ shared `getAuthUser` ব্যবহার করো** — `getParam` সব জায়গায় বসানো হয়ে গেছে (আর কোথাও `as string` নেই)। কিন্তু `getAuthUser`-এর shared version এখন শুধু `patient.controller.ts` import করে। নিচের ৫টা file-এ এখনো নিজস্ব `const getAuthUser = ...` কপি আছে:
   - `appointment.controller.ts`
   - `doctorSchedule.controller.ts`
   - `prescription.controller.ts`
   - `review.controller.ts`
   - `stats.controller.ts`

   আর `admin.controller.ts`-এ প্রতিটা function-এ `if (!req.user)` লেখা আছে। এগুলোতে local function মুছে import করো:
   ```ts
   import { getAuthUser } from "@/app/shared/getAuthUser";
   ```
   তারপর অপ্রয়োজনীয় `AppError` / `IRequestUser` import মুছে দিও, না হলে unused import-এর lint আসবে।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

2. **Param-এর format check নেই** — `getParam` শুধু দেখে value আছে কিনা। `id` uuid কিনা দেখে না, তাই `"abc"` দিলে DB পর্যন্ত যায় আর `P2023` → `400` আসে ([নোট ৩৬](./36-handle-prisma-error.md))। চাইলে Zod দিয়ে params-ও validate করা যায়, তবে এখনকার handler-এই কাজ চলে যাচ্ছে।
