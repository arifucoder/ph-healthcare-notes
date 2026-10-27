# 16 - getMe (`/me`) Route

> Module: 39-02

```
GET /api/v1/auth/me → checkAuth (cookie → req.user) → controller (req.user) → service (DB থেকে user + role-এর profile) → response
```

**কাজ:** যে user এখন logged in, তার নিজের সব তথ্য (user + patient / doctor / admin profile) ফেরত দেওয়া। Frontend login-এর পর বা page reload-এ এটা call করে বোঝে "আমি কে"।

**`req.user` কোথা থেকে আসে?**
`checkAuth` (note 13) cookie verify করে `req.user`-এ `{ userId, name, email, role }` রেখে দেয়। এর type আগেই বানানো:
- `src/app/interfaces/express.d.ts` → Express-এর `Request`-এ `user?` যোগ (note 13, ধাপ ১১)
- `src/app/interfaces/requestUser.interface.ts` → `IRequestUser` (note 14, ধাপ ১৩)

তাই `/me`-তে client-কে id পাঠাতে হয় না — কে request করছে সেটা cookie থেকেই জানা যায়।

---

## ধাপ ১: `auth.route.ts`

```ts
router.get("/me", checkAuth(Role.ADMIN, Role.DOCTOR, Role.PATIENT, Role.SUPER_ADMIN), AuthController.getMe)
```

> ✏️ import নোটে ছিল না:
> ```ts
> import { checkAuth } from "@/app/middleware/checkAuth";
> import { Role } from "@/generated/prisma/enums";
> ```

- ৪টা role-ই দেওয়া → যেকোনো logged-in user নিজের তথ্য দেখতে পারবে।

## ধাপ ২: `auth.controller.ts`

```ts
const getMe = catchAsync(async (req: Request, res: Response) => {
    const user = req.user;
    if (!user) {
       throw new AppError(httpStatus.UNAUTHORIZED, "Unauthorized access! Please log in.");
    }
    const result = await AuthService.getMe(user);
    sendResponse(res, {
       httpStatusCode: httpStatus.OK,
       success: true,
       message: "User profile fetched successfully",
       data: result,
    });
});
```

> ✏️ নোটে ছিল না:
> - import: `import AppError from "@/app/errorHelpers/AppError";` (`httpStatus` আগেই আছে)
> - export object-এ যোগ: `export const AuthController = { registerPatient, loginUser, getMe };`

- `if (!user)` → type-এ `user?` optional, তাই check (আর checkAuth ভুলে বাদ পড়লেও নিরাপদ)। যে variable service-এ পাঠানো হচ্ছে (`user`), সেটাকেই check করতে হবে, তবেই TypeScript বোঝে এরপর `user` আর `undefined` না।

> ✅ (Done) আগে ছিল `if (!req.user)` আর `console.log({ user })`। `req.user` check করলে আলাদা variable `user` narrow হয় না, তাই `pnpm typecheck`-এ type error আসত; আর `console.log` oxlint-এর `no-console`-এ error দিত। দুটোই ঠিক করা হয়েছে।

## ধাপ ৩: `auth.service.ts`

```ts
const getMe = async (user: IRequestUser) => {
    const isUserExists = await prisma.user.findUnique({
       where: {
          id: user.userId,
       },
       include: {
          patient: {
             include: {
                appointments: {
                   take: 5,
                   orderBy: { createdAt: "desc" },
                },
                reviews: true,
                prescriptions: true,
                medicalReports: true,
                patientHealthData: true,
             },
          },
          doctor: {
             include: {
                specialties: true,
                appointments: {
                   take: 5,
                   orderBy: { createdAt: "desc" },
                },
                reviews: true,
                prescriptions: true,
             },
          },
          admin: true,
       },
    });

    if (!isUserExists) {
       throw new AppError(httpStatus.NOT_FOUND, "User not found");
    }

    return isUserExists;
};
```

> ✏️ নোটে ছিল না:
> - import: `import type { IRequestUser } from "@/app/interfaces/requestUser.interface";` (`prisma`, `AppError`, `httpStatus` আগেই আছে)
> - export object-এ যোগ: `export const AuthService = { registerPatient, loginUser, getMe };`
>
> ⚠️ `appointments`, `reviews`, `prescriptions`, `medicalReports`, `patientHealthData` — এই model-গুলো schema-তে না থাকলে Prisma error দেবে (`Unknown field`)। এখনো না বানানো থাকলে ওই লাইনগুলো comment করে রাখো, model বানানোর পর খুলে দিও।

**কী কী আসে:**

| Role | `patient` | `doctor` | `admin` |
|---|---|---|---|
| PATIENT | ✅ + appointments (শেষ ৫টা), reviews, prescriptions, medicalReports, patientHealthData | `null` | `null` |
| DOCTOR | `null` | ✅ + specialties, appointments (শেষ ৫টা), reviews, prescriptions | `null` |
| ADMIN / SUPER_ADMIN | `null` | `null` | ✅ |

- একটা query-তেই user আর তার role-এর profile আসছে। যার যে profile নেই, সেটা `null`।
- `id: user.userId` → id নেওয়া হচ্ছে `req.user` থেকে, body/params থেকে না, তাই কেউ অন্যের id দিয়ে অন্যের তথ্য দেখতে পারবে না।

**কেন `appointments`-এ `take: 5`?**
সব appointment একসাথে আনলে (একজন doctor-এর হাজারটাও হতে পারে) query slow হয়ে যাবে। তাই আপাতত শুধু শেষ ৫টা (`orderBy: { createdAt: "desc" }`)। পরে query (pagination) দিয়ে ঠিক করা হবে।

---

## ধাপ ৪: Test (Postman)

আগে `POST /api/v1/auth/login` (cookie Postman নিজেই রাখবে), তারপর:

| Method | URL | Body | আশা |
|---|---|---|---|
| `GET` | `http://localhost:5000/api/v1/auth/me` | নেই | 200, নিজের user + profile |
| `GET` | একই, login ছাড়া | নেই | 401 |

Patient, doctor, admin — তিন রকম user দিয়ে login করে দেখো, কোন profile ভরা আর কোনটা `null`।

---

## 🔜 পরে যা যোগ হবে

- [ ] Appointment, review, prescription, medical report, health data model
- [ ] `take: 5`-এর বদলে query দিয়ে pagination
- [ ] `/me` থেকে নিজের profile update (`PATCH /auth/me` বা role অনুযায়ী)

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`specialties: true` শুধু join table-এর row দেয়** — `DoctorSpecialty`-র `id`, `doctorId`, `specialtyId` আসবে, specialty-র নাম (`title`) আসবে না। Note 15-এর মতো ভিতরে include দাও:
   ```ts
   specialties: {
   	include: { specialty: true },
   },
   ```

2. **৪টা role না লিখে `checkAuth()`** — note 13 অনুযায়ী `checkAuth()`-এ কোনো role না দিলে যেকোনো logged-in user ঢুকতে পারে। ফল একই, কিন্তু পরে নতুন role যোগ হলে এখানে ভুলে যাওয়ার ভয় থাকে না:
   ```ts
   router.get("/me", checkAuth(), AuthController.getMe);
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Role অনুযায়ী শুধু দরকারি অংশ include** — এখন প্রতিটা request-এ patient, doctor, admin তিনটা relation-ই খোঁজা হয় (দুটো `null` আসবে)। `user.role` দেখে শুধু ওই role-এর include দিলে query হালকা হবে:
   ```ts
   const include =
   	user.role === Role.PATIENT ? { patient: { include: { /* ... */ } } }
   	: user.role === Role.DOCTOR ? { doctor: { include: { /* ... */ } } }
   	: { admin: true };
   ```

4. **Deleted / blocked check দরকার নেই** — checkAuth আগেই DB থেকে `isDeleted` আর `status` check করেছে, তাই service-এ আবার করার দরকার নেই।

### ✅ (Done)

- **`if (!req.user)` → `if (!user)`** — একই variable check, তাই `pnpm typecheck`-এ আর type error নেই (ধাপ ২)।
- **`console.log({ user })` মুছে ফেলা** — oxlint `no-console` error আর নেই।
