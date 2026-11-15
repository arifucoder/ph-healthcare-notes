## install packages

```shell
pnpm add jsonwebtoken cookie-parser
pnpm add -D @types/jsonwebtoken @types/cookie-parser
pnpm add cors && pnpm add -D @types/cors
```

## folder structure
```text
apps/api/src/
├─ app.ts                          ✏️ cookieParser() (+ cors credentials: true)
└─ app/
   ├─ config/env.ts                ✏️ FRONTEND_URL (cors-এর জন্য)
   ├─ utils/                       🆕
   │  ├─ jwt.ts                    🆕 jsonwebtoken-এর পাতলা wrapper
   │  ├─ token.ts                  🆕 access/refresh token বানানো
   │  └─ cookie.ts                 🆕 cookie set/clear + options
   ├─ middleware/
   │  └─ checkAuth.ts              🆕 cookie → verify → req.user → role check
   ├─ shared/
   │  └─ getAuthUser.ts            🆕 req.user নিরাপদে বের করা
   ├─ interfaces/
   │  └─ express.d.ts              🆕 Request-এ user type যোগ
   └─ module/auth/
      ├─ auth.route.ts             ✏️ /login, /refresh-token, /logout
      ├─ auth.controller.ts        ✏️ cookie set/clear এখানে
      ├─ auth.service.ts           ✏️ login, getNewTokens, logout
      ├─ auth.validation.ts        ✏️ loginSchema
      └─ auth.interface.ts         ✏️ TLoginPayload, TJwtPayload
```

app.ts-এ cookieParser() নেই। এটা ছাড়া req.cookies undefined থাকবে, refresh কাজ করবে না।
.env 
```dotenv
FRONTEND_URL=http://localhost:3000
```
In env.ts 
```text
FRONTEND_URL: z.url({ protocol: /^https?$/ }),
```
app.ts: তোমার cors({...}) block বসবে express.json()-এর আগে, আর অবশ্যই app.use("/api/v1", AppRoutes)-এর আগে। ক্রম:
```ts
app.use(
	cors({
		origin: envVars.FRONTEND_URL,
		credentials: true,
		exposedHeaders: ["X-Session-Refresh", "X-Session-Expires-At", "X-Time-Remaining"],
	}),
);
```
