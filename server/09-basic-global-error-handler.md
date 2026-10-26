# 09 - Basic Global Error Handler

> 📝 এটা **simple version**। পূর্ণ version (AppError, Zod error, dev/production আলাদা response) → [12-global-error-handler](./12-global-error-handler.md)

**কাজ:** পুরো app এর সব error **এক জায়গায়** handle করা। প্রতিটা controller এ আলাদা করে error response লিখতে হবে না।

```
Controller এ error → catchAsync → next(error) → globalErrorHandler → error JSON
```

---

## ধাপ ১: Simple global error handler (app.ts এ)

```ts
import httpStatus from "http-status";

app.use((err: unknown, _req: Request, res: Response, _next: NextFunction) => {
  console.error(err);

  res.status(httpStatus.INTERNAL_SERVER_ERROR).json({
    success: false,
    message: "Internal Server Error",
    error: err instanceof Error ? err.message : "unknown error",
  });
});
```

এটাই সবচেয়ে ছোট আর simple global error handler middleware।

### কেন ৪টা parameter লাগে (`next` use না করলেও)?

Express **parameter এর সংখ্যা গুনে** বোঝে এটা কোন ধরনের middleware:

| Parameter | Express যা ভাবে |
|---|---|
| ৩টা `(req, res, next)` | সাধারণ middleware |
| ৪টা `(err, req, res, next)` | **Error handler** ✅ |

এখানে আমরা `next` call করছি না, তবুও লিখতে হবে। ৩টা দিলে Express এটাকে সাধারণ middleware ভাববে — তখন `err` কে `req`, `req` কে `res`, `res` কে `next` মনে করবে, আর error handler হিসেবে কাজই করবে না।

---

## ধাপ ২: আলাদা file এ নেওয়া — `src/app/middleware/globalErrorHandler.ts`

এটা একটা middleware, তাই `middleware` folder এ রাখবো:

```
src/app/middleware/
├── globalErrorHandler.ts
└── notFound.ts
```

```ts
import type { NextFunction, Request, Response } from "express";
import httpStatus from "http-status";

export const globalErrorHandler = (
    err: unknown,
    _req: Request,
    res: Response,
    _next: NextFunction,
) => {
    console.error(err);

    res.status(httpStatus.INTERNAL_SERVER_ERROR).json({
       success: false,
       message: "Internal Server Error",
       error: err instanceof Error ? err.message : "unknown error",
    });
};
```

- `err: unknown` → error যেকোনো কিছু হতে পারে, তাই `any` না দিয়ে `unknown`। `instanceof Error` দিয়ে check করে তারপর `.message` নিচ্ছি (type-safe)
- `_req`, `_next` → use হচ্ছে না, তাই `_` দিয়ে শুরু — oxlint warning দেবে না। নাম বদলালেও parameter **৪টাই** আছে, তাই Express এখনো error handler হিসেবেই চেনে
- `console.error` → আমাদের `no-console` rule এ শুধু `warn` / `error` allowed
- `httpStatus.INTERNAL_SERVER_ERROR` → `500` এর বদলে নাম দিয়ে লেখা (http-status package), পড়তে সহজ

---

## ধাপ ৩: app.ts এ যোগ

সব route এর পরে, `export default app;` এর আগে:

```ts
import { globalErrorHandler } from "@/app/middleware/globalErrorHandler";
import { notFound } from "@/app/middleware/notFound";

// ... cors, express.json() ...
app.use("/api/v1", IndexRoutes);

app.use(globalErrorHandler);
app.use(notFound);

export default app;
```

---

## ধাপ ৪: catchAsync update — error পাঠাবো globalErrorHandler এ

আগে `catchAsync` নিজেই `res.status(500).json(...)` পাঠাতো। এখন global error handler আছে, তাই শুধু `next(error)` দিলেই error সেখানে চলে যাবে:

```ts
import type { NextFunction, Request, RequestHandler, Response } from "express";

export const catchAsync = (fn: RequestHandler) => {
    return async (req: Request, res: Response, next: NextFunction) => {
       try {
          await fn(req, res, next);
       } catch (error) {
          next(error);
       }
    };
};
```

> `error: any` লাগে না — `next()` যেকোনো type নেয়।

**`next(error)` কী করে?**
`next()` এ কিছু পাঠালে Express বোঝে **error হয়েছে** — তখন মাঝের সব সাধারণ middleware skip করে সরাসরি **error handler** (৪ parameter ওয়ালা) এ চলে যায়।

এখন সব error response এক জায়গা থেকে (`globalErrorHandler`) যায় ✅

---

## 🔜 পরে যা যোগ হবে

- [x] Custom `AppError` class (নিজের status code সহ error) → note 12
- [ ] Prisma error handle (যেমন `P2002` duplicate → 409, `P2025` not found → 404)
- [x] Zod validation error handle → note 12
- [x] Development এ `stack` দেখানো, production এ লুকানো → note 12

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ (Done) — code এ apply করা হয়েছে

1. Unused parameter এ `_` (`_req`, `_next`)
2. `eslint-disable` comment মুছে ফেলা (আমরা oxlint use করি)
3. `console.log` → `console.error`
4. `catchAsync` এ `error: any` → `error`
5. globalErrorHandler এ `any` → `unknown` + `instanceof Error` check
6. `500` → `httpStatus.INTERNAL_SERVER_ERROR`

### 🔵 জেনে রাখো

**৬. notFound আর globalErrorHandler এর order**
এখন দুইভাবেই কাজ করবে (বিস্তারিত [08-not-found-route](./08-not-found-route.md) নোটে)। তবে পরে `notFound` থেকে `next(new AppError(404, ...))` পাঠাতে চাইলে order হতে হবে:

```ts
app.use(notFound);
app.use(globalErrorHandler); // একদম শেষে
```

**৭. Express 5 এ async error নিজে থেকেই `next` এ যায়**
আমরা Express 5 use করছি — এখানে async function এ error হলে Express নিজেই `next(error)` call করে। তাই `catchAsync` না দিলেও error globalErrorHandler এ পৌঁছাবে। তবুও `catchAsync` রাখা ক্ষতি নেই (ক্লাসের মতো রাখো, Express 4 এ এটা লাগতোই)।

**৮. Production এ `err.message` সাবধানে**
কিছু error message এ DB/internal info থাকে। পরে `NODE_ENV === "production"` হলে সাধারণ message দেখানো ভালো।
