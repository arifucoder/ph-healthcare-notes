# 09 - Global Error Handler

> 📝 এটা **simple version**। পরে আরও অনেক কিছু যোগ হবে (AppError, Prisma error, Zod error ...) — তখন এই নোট update করবো।

**কাজ:** পুরো app এর সব error **এক জায়গায়** handle করা। প্রতিটা controller এ আলাদা করে error response লিখতে হবে না।

```
Controller এ error → catchAsync → next(error) → globalErrorHandler → error JSON
```

---

## ধাপ ১: Simple global error handler (app.ts এ)

```ts
// eslint-disable-next-line @typescript-eslint/no-explicit-any
app.use((err: any, req: Request, res: Response, next: NextFunction) => {
  console.log(err);

  res.status(500).json({
    success: false,
    message: 'Internal Server Error',
    error: err.message,
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

// eslint-disable-next-line @typescript-eslint/no-explicit-any
export const globalErrorHandler = (err: any, req: Request, res: Response, next: NextFunction) => {
    console.log(err);

    res.status(500).json({
       success: false,
       message: "Internal Server Error",
       error: err.message,
    });
};
```

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
       } catch (error: any) {
          next(error);
       }
    };
};
```

**`next(error)` কী করে?**
`next()` এ কিছু পাঠালে Express বোঝে **error হয়েছে** — তখন মাঝের সব সাধারণ middleware skip করে সরাসরি **error handler** (৪ parameter ওয়ালা) এ চলে যায়।

এখন সব error response এক জায়গা থেকে (`globalErrorHandler`) যায় ✅

---

## 🔜 পরে যা যোগ হবে

- [ ] Custom `AppError` class (নিজের status code সহ error)
- [ ] Prisma error handle (যেমন `P2002` duplicate → 409, `P2025` not found → 404)
- [ ] Zod validation error handle
- [ ] Development এ `stack` দেখানো, production এ লুকানো

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

**১. Unused parameter এ `_` দাও**
`req` আর `next` use হচ্ছে না, তাই oxlint `no-unused-vars` warning দেবে। আমাদের `.oxlintrc.json` এ `_` দিয়ে শুরু হলে ignore হয়:

```ts
export const globalErrorHandler = (err: any, _req: Request, res: Response, _next: NextFunction) => {
```

> নাম বদলালেও parameter **৪টাই** থাকছে, তাই Express এখনো error handler হিসেবেই চিনবে।

**২. `eslint-disable` comment অপ্রয়োজনীয়**
আমরা oxlint use করি, eslint না — তাই `// eslint-disable-next-line ...` লাইনটা কোনো কাজ করছে না। মুছে দিতে পারো।

**৩. `console.log` → `console.error`**
আমাদের `no-console` rule এ শুধু `warn` আর `error` allowed।

**৪. `catchAsync` এ `error: any` লাগে না**
`next(error)` যেকোনো type নেয়, তাই শুধু `catch (error)` লিখলেই হয়।

**৫. `any` এর বদলে `unknown` (globalErrorHandler এ)**

```ts
(err: unknown, _req: Request, res: Response, _next: NextFunction) => {
    res.status(500).json({
        success: false,
        message: "Internal Server Error",
        error: err instanceof Error ? err.message : "unknown error",
    });
};
```

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
