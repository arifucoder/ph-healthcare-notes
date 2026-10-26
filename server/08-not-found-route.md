# 08 - Not Found Route

**কাজ:** User যদি এমন কোনো route এ hit করে যেটা আমাদের নেই, তাহলে Express এর default HTML page (`Cannot GET /xyz`) এর বদলে আমাদের নিজের format এ JSON response যাবে — "এই route টা নেই"।

```
Request → সব route check → কোনোটা match করেনি → notFound → 404 JSON
```

---

## ধাপ ১: notFound middleware — `src/app/middleware/notFound.ts`

```
src/app/middleware/
└── notFound.ts
```

```ts
import type { Request, Response } from "express";
import httpStatus from "http-status";

export const notFound = (req: Request, res: Response) => {
    res.status(httpStatus.NOT_FOUND).json({
       success: false,
       message: `Route ${req.originalUrl} Not Found`,
    });
};
```

- `httpStatus.NOT_FOUND` → `404` (http-status package থেকে)
- `req.originalUrl` → user যে URL এ hit করেছে (যেমন `/api/v1/abc`), message এ দেখাবে

---

## ধাপ ২: app.ts এ যোগ

```ts
import { notFound } from "@/app/middleware/notFound";

// ... cors, express.json(), routes ...
app.use("/api/v1", IndexRoutes);

// ... globalErrorHandler

app.use(notFound); // ✅ সবার শেষে, export এর ঠিক আগে

export default app;
```

**⚠️ জায়গাটা গুরুত্বপূর্ণ — সব route এর পরে দিতে হবে।**
Express উপর থেকে নিচে middleware চালায়। `notFound` এ কোনো path নেই, তাই এটা **সব request** ধরে ফেলে। উপরে দিলে সঠিক route এ যাওয়ার আগেই সবাই 404 পাবে।

---

## Test

```
GET http://localhost:5000/api/v1/abc
```

Response (`404`):

```json
{
	"success": false,
	"message": "Route /api/v1/abc Not Found"
}
```

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

**১. globalErrorHandler এর আগে না পরে? — দুইভাবেই কাজ করে**
নোটে বলা হয়েছে globalErrorHandler এর **পরে** দিতে হবে। আসলে এখন দুইভাবেই কাজ করবে, কারণ:
- Error handler (৪টা parameter: `err, req, res, next`) শুধু **error হলে** চলে, সাধারণ request এ Express ওটা skip করে
- তাই unknown route এর request error handler পার হয়ে `notFound` এ পৌঁছে যায়

তবে সাধারণ convention হলো:

```ts
app.use("/api/v1", IndexRoutes);
app.use(notFound);           // আগে
app.use(globalErrorHandler); // একদম শেষে
```

এর সুবিধা: পরে `notFound` এ `next(new AppError(404, ...))` দিলে সেটা globalErrorHandler ধরতে পারবে — সব error একই format এ যাবে। ক্লাসের মতো রাখতে চাইলে রাখো, শুধু জেনে রাখো কেন।

**২. Response format একই রাখো**
`sendResponse` এর মতোই `{ success, message }` shape আছে — ভালো। পরে globalErrorHandler বানালে খেয়াল রাখবে সব error response এর shape যেন একই থাকে (frontend এর জন্য সহজ)।
