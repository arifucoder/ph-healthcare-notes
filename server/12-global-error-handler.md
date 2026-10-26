# 12 - Global Error Handler

> আগের simple version → [09-basic-global-error-handler](./09-basic-global-error-handler.md)

```
error (Zod / AppError / Error / অন্য কিছু) → next(err) → globalErrorHandler → type দেখে statusCode + message + errorSources → JSON
```

**লক্ষ্য:** সব error-এর response একই shape-এ যাবে, আর development-এ বেশি তথ্য, production-এ কম তথ্য।

```
src/app/
├── interfaces/
│   └── error.interface.ts
├── errorHelpers/
│   ├── AppError.ts
│   └── handleZodError.ts
└── middleware/
    └── globalErrorHandler.ts
```

---

## ধাপ ১: Error-এর type — `src/app/interfaces/error.interface.ts`

```ts
// error sources-এর জন্য
export interface TErrorSources {
    path: string;
    message: string;
}

// handleZodError-এর মতো helper এটা return করবে
export interface TGenericErrorResponse {
    statusCode: number;
    message: string;
    errorSources: TErrorSources[];
}

// client-এর কাছে যা যাবে
export interface TErrorResponse {
    success: false;
    message: string;
    errorSources: TErrorSources[];
    error?: unknown;
    stack?: string;
}
```

| Interface | কোথায় লাগে |
|---|---|
| `TErrorSources` | কোন field-এ (`path`) কী সমস্যা (`message`) |
| `TGenericErrorResponse` | helper (যেমন `handleZodError`) যা return করে |
| `TErrorResponse` | শেষে client যা পায় |

## ধাপ ২: `src/app/errorHelpers/` folder

### `AppError.ts` — নিজের status code সহ error

```ts
class AppError extends Error {
    public readonly statusCode: number;

    constructor(statusCode: number, message: string, stack = "") {
       super(message);
       this.name = "AppError";
       this.statusCode = statusCode;

       if (stack) {
          this.stack = stack;
       } else {
          Error.captureStackTrace(this, this.constructor);
       }
    }
}

export default AppError;
```

- সাধারণ `Error`-এ শুধু message থাকে, `AppError`-এ সাথে `statusCode`-ও থাকে (404, 409 ইত্যাদি)।
- `Error.captureStackTrace` → stack trace-এ constructor-এর নিজের লাইন বাদ দিয়ে, যেখান থেকে error throw হয়েছে সেখান থেকে দেখায়।

ব্যবহার:

```ts
import httpStatus from "http-status";
import AppError from "@/app/errorHelpers/AppError";

throw new AppError(httpStatus.NOT_FOUND, "Specialty not found");
```

### `handleZodError.ts` — ZodError-কে সুন্দর shape-এ আনা

```ts
import httpStatus from "http-status";
import type { ZodError } from "zod";
import type { TErrorSources, TGenericErrorResponse } from "../interfaces/error.interface";

export const handleZodError = (err: ZodError): TGenericErrorResponse => {
    const errorSources: TErrorSources[] = err.issues.map((issue) => ({
        path: issue.path.map(String).join("."),
        message: issue.message,
    }));

    return {
        statusCode: httpStatus.BAD_REQUEST,
        message: "Validation Error",
        errorSources,
    };
};
```

- `err.issues` → প্রতিটা ভুল field-এর একটা করে issue।
- `issue.path` → যেমন `["doctor", "email"]` → `.join(".")` করে `"doctor.email"`।
- এখানে শুধু type লাগছে, তাই `import type { ZodError }`।

## ধাপ ৩: `src/app/middleware/globalErrorHandler.ts` update

```ts
/* eslint-disable @typescript-eslint/no-unused-vars */
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status";
import { ZodError } from "zod"; // ✅ value import, কারণ instanceof-এ লাগবে
import { envVars } from "../config/env";
import AppError from "../errorHelpers/AppError";
import { handleZodError } from "../errorHelpers/handleZodError";
import type { TErrorResponse, TErrorSources } from "../interfaces/error.interface";

export const globalErrorHandler = (err: unknown, req: Request, res: Response, next: NextFunction) => {
    const isDev = envVars.NODE_ENV === "development";

    if (isDev) {
       console.log("Error from Global Error Handler", err);
    }

    let statusCode: number = httpStatus.INTERNAL_SERVER_ERROR;
    let message = "Internal Server Error";
    let errorSources: TErrorSources[] = [{ path: "", message }];
    let stack: string | undefined;

    if (err instanceof ZodError) {
       const simplifiedError = handleZodError(err);
       statusCode = simplifiedError.statusCode;
       message = simplifiedError.message;
       errorSources = simplifiedError.errorSources;
       stack = err.stack;
    } else if (err instanceof AppError) {
       statusCode = err.statusCode;
       message = err.message;
       errorSources = [{ path: "", message: err.message }];
       stack = err.stack;
    } else if (err instanceof Error) {
       // production-এ internal message লুকানো হচ্ছে
       message = isDev ? err.message : "Something went wrong";
       errorSources = [{ path: "", message }];
       stack = err.stack;
    }

    const errorResponse: TErrorResponse = {
       success: false,
       message,
       errorSources,
       error: isDev ? err : undefined,
       stack: isDev ? stack : undefined,
    };

    res.status(statusCode).json(errorResponse);
};
```

**কোন error-এ কী হয়:**

| Error | statusCode | message | errorSources |
|---|---|---|---|
| `ZodError` | 400 | `"Validation Error"` | প্রতিটা ভুল field আলাদা করে |
| `AppError` | নিজের দেওয়া (404, 409...) | নিজের message | ১টা |
| সাধারণ `Error` | 500 | dev: আসল message, production: `"Something went wrong"` | ১টা |
| অন্য কিছু (string ইত্যাদি) | 500 | `"Internal Server Error"` | ১টা |

**Dev বনাম Production:**

| | development | production |
|---|---|---|
| console log | ✅ | ❌ |
| `error` (পুরো error object) | ✅ | ❌ |
| `stack` | ✅ | ❌ |
| সাধারণ `Error`-এর আসল message | ✅ | ❌ লুকানো |

- `ZodError` আগে check হয়, তারপর `AppError`, তারপর সাধারণ `Error`। `AppError` নিজেও `Error`, তাই এই order না মানলে `AppError` সাধারণ `Error` হিসেবে ধরা পড়ত।
- `ZodError` normal import (`import type` না), কারণ `instanceof`-এ runtime-এ class-টা লাগে। `verbatimModuleSyntax` থাকায় `import type` দিলে runtime-এ মুছে যেত।

---

## ধাপ ৪: Test

### Zod error

`POST http://localhost:5000/api/v1/user/create-doctor` — ভুল body দিয়ে (note 11-এর test body)

Response (৪০০) প্রায় এরকম:

```json
{
  "success": false,
  "message": "Validation Error",
  "errorSources": [
    { "path": "password", "message": "Password must be at least 6 characters" },
    { "path": "doctor.email", "message": "Invalid email address" }
  ]
}
```

> development-এ এর সাথে `error` আর `stack`-ও আসবে।

### AppError

Service-এ `throw new AppError(httpStatus.NOT_FOUND, "Specialty not found")` দিলে:

```json
{
  "success": false,
  "message": "Specialty not found",
  "errorSources": [{ "path": "", "message": "Specialty not found" }]
}
```

---

## 🔜 পরে যা যোগ হবে

- [ ] Prisma error handle (`P2002` duplicate → 409, `P2025` not found → 404) → `handlePrismaError.ts`
- [ ] Service-গুলোতে `throw new Error(...)` → `throw new AppError(...)`
- [ ] `notFound` থেকে `next(new AppError(404, ...))`

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Production-এ service-এর error message হারিয়ে যাবে** — এখন সাধারণ `Error` production-এ `"Something went wrong"` (500) দেখায়। আমাদের user/doctor service-এ এখনো `throw new Error("User with this email already exists")` আছে, তাই production-এ user বুঝবে না কী ভুল হলো। এগুলো `AppError`-এ বদলাও:
   ```ts
   throw new AppError(httpStatus.NOT_FOUND, `Specialty with id ${specialtyId} not found`);
   throw new AppError(httpStatus.CONFLICT, "User with this email already exists");
   ```

2. **Production-এ কোনো log নেই** — `console.log` শুধু `isDev`-এ চলে, তাই production-এ error হলে কোথাও কিছু থাকবে না, bug খোঁজা কঠিন হবে। সবসময় log করো, আর oxlint-এর `no-console`-এর জন্য `console.error` দাও:
   ```ts
   console.error("Error from Global Error Handler", isDev ? err : (err instanceof Error ? err.message : err));
   ```

### 🟡 ভালো practice

3. **`eslint-disable` comment কাজ করে না, unused param-এ `_` দাও** — আমরা oxlint ব্যবহার করি, তাই comment-টা বেকার (note 09-এ এটা মুছেও ফেলা হয়েছিল)। `req`, `next` ব্যবহার হয়নি:
   ```ts
   export const globalErrorHandler = (err: unknown, _req: Request, res: Response, _next: NextFunction) => {
   ```
   (parameter ৪টাই থাকছে, তাই Express এখনো error handler হিসেবেই চিনবে)

4. **`notFound` আর `globalErrorHandler`-এর order** — note 09-এ `globalErrorHandler` আগে আর `notFound` পরে বসানো। পরে `notFound` থেকে `next(new AppError(...))` পাঠাতে চাইলে `globalErrorHandler` সবার শেষে থাকতে হবে:
   ```ts
   app.use(notFound);
   app.use(globalErrorHandler);
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

5. **dev-এ `error` অনেক সময় `{}` দেখাবে (সম্ভবত)** — `Error`-এর `message`/`stack` enumerable না, তাই `res.json()` সাধারণ `Error`-কে `{}` বানিয়ে দেয়। (`ZodError`-এর `issues` ঠিকই দেখাবে।) চাইলে:
   ```ts
   error: isDev ? (err instanceof Error ? { name: err.name, message: err.message } : err) : undefined,
   ```

6. **Naming** — module-এর interface-গুলো `I` দিয়ে শুরু (`ICreateDoctorPayload`), এখানে `T` দিয়ে। কাজে সমস্যা নেই, শুধু একটা নিয়ম রাখলে খুঁজতে সহজ।
