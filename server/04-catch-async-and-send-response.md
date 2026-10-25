# 04 - catchAsync & sendResponse

> লক্ষ্য: controller এর **repeat হওয়া code** কমানো (DRY — Don't Repeat Yourself)

---

## ১. সমস্যা কী?

প্রতিটা controller এ একই জিনিস বারবার লিখছি:

```ts
try {
	// ✅ শুধু এই অংশটা আলাদা (dynamic)
} catch (error) {
	// ❌ এই অংশটা সব জায়গায় একই (repeat)
}
```

**সমাধান:** এমন একটা function বানাবো যেটা নিজে try/catch চালাবে, আর try এর ভেতরের অংশটা আমরা **parameter হিসেবে function** পাঠাবো।
(Function এর ভেতরে function পাঠানো = **Higher Order Function**)

```
src/app/shared/
├── catchAsync.ts
└── sendResponse.ts
```

---

## ২. catchAsync — `src/app/shared/catchAsync.ts`

```ts
import type { NextFunction, Request, RequestHandler, Response } from "express";

export const catchAsync = (fn: RequestHandler) => {
	return async (req: Request, res: Response, next: NextFunction) => {
		try {
			await fn(req, res, next);
		} catch (error) {
			console.error(error);
			res.status(500).json({
				success: false,
				message: "Something went wrong",
				error: error instanceof Error ? error.message : "unknown error",
			});
		}
	};
};
```

কীভাবে কাজ করে:
1. `catchAsync(fn)` → আমাদের controller function (`fn`) নেয়
2. একটা **নতুন function return** করে — Express এটাকেই route এ চালায়
3. সেই নতুন function এর ভেতরে `try` এ `fn` চলে, error হলে `catch` handle করে

**যা ঠিক করা হলো:**
- `import { ... }` → **`import type { ... }`** — এগুলো সব type, আর আমাদের tsconfig এ `verbatimModuleSyntax: true`, তাই `import type` না দিলে TypeScript error দেবে।
- `/* eslint-disable ... */` সরালাম — আমরা oxlint use করি, আর `any` আর লাগছে না।
- `catch (error: any)` + `error.message` → `catch (error)` + `instanceof Error` check (type-safe)
- `'Failed to fetch'` → `"Something went wrong"` — এটা সব route (create/delete) এ use হবে, তাই সাধারণ message।
- `console.log` → `console.error` (oxlint `no-console` rule)

> 💡 পরে **global error handler** বানালে `catch` এর ভেতরে `res.status(500)...` না লিখে শুধু `next(error)` দেবো — তখন সব error এক জায়গা থেকে handle হবে।

---

## ৩. sendResponse — `src/app/shared/sendResponse.ts`

Success response ও প্রতিবার repeat হচ্ছে:

```ts
res.status(201).json({ success: true, message: "...", data: result });
```

তাই:

```ts
import type { Response } from "express";

interface IResponseData<T> {
	httpStatusCode: number;
	success: boolean;
	message: string;
	data?: T;
}

export const sendResponse = <T>(res: Response, responseData: IResponseData<T>): void => {
	const { httpStatusCode, success, message, data } = responseData;

	res.status(httpStatusCode).json({
		success,
		message,
		data,
	});
};
```

- `<T>` → **Generic**: `data` যেকোনো type এর হতে পারে (Specialty, Specialty[], ...)
- `data?` → optional, সবসময় data পাঠাতে হবে না
- ✏️ `import { Response }` → `import type { Response }`

> এতে সব API এর response এর **shape একই** থাকে — frontend এর জন্য সুবিধা।

---

## ৪. Controller — আগে vs পরে

**আগে (প্রতিটায় ~20 লাইন):**

```ts
const createSpecialty = async (req: Request, res: Response) => {
	try {
		const result = await SpecialtyService.createSpecialty(req.body);
		res.status(201).json({ success: true, message: "...", data: result });
	} catch (error) {
		console.error(error);
		res.status(500).json({ success: false, message: "...", error: "..." });
	}
};
```

**পরে — `specialty.controller.ts` (পুরোটা):**

```ts
import type { Request, Response } from "express";
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import { SpecialtyService } from "./specialty.service";

const createSpecialty = catchAsync(async (req: Request, res: Response) => {
	const payload = req.body;
	const result = await SpecialtyService.createSpecialty(payload);

	sendResponse(res, {
		httpStatusCode: 201,
		success: true,
		message: "Specialty created successfully",
		data: result,
	});
});

const getAllSpecialties = catchAsync(async (_req: Request, res: Response) => {
	const result = await SpecialtyService.getAllSpecialties();

	sendResponse(res, {
		httpStatusCode: 200,
		success: true,
		message: "Specialties fetched successfully",
		data: result,
	});
});

const deleteSpecialty = catchAsync(async (req: Request, res: Response) => {
	const id = req.params.id as string;
	const result = await SpecialtyService.deleteSpecialty(id);

	sendResponse(res, {
		httpStatusCode: 200,
		success: true,
		message: "Specialty deleted successfully",
		data: result,
	});
});

export const SpecialtyController = {
	createSpecialty,
	getAllSpecialties,
	deleteSpecialty,
};
```

**খেয়াল করো:**
- ✏️ `specialtyService` → **`SpecialtyService`** (service file এ capital S দিয়ে export করেছি)
- `deleteSpecialty` এ আগের নোটের `Request<{ id: string }>` এখানে দিলে error আসবে, কারণ `catchAsync` সাধারণ `RequestHandler` নেয়। তাই এখানে `req.params.id as string` লিখেছি।
- এখন আর কোনো controller এ `try/catch` নেই, আর response সব একই format এ যায় ✅

Route file (`specialty.route.ts`) এ কিছু বদলাতে হবে না।
