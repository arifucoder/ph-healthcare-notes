# 03 - Route Management & Specialty CRUD API

> Module: 37-07, 37-08

---

## ১. Module Pattern (Route → Controller → Service)

```
Request → Route → Controller → Service → Prisma (DB)
```

| File | কাজ |
|---|---|
| `*.route.ts` | কোন URL + method এ কোন controller যাবে |
| `*.controller.ts` | `req` থেকে data নেয়, `res` পাঠায় |
| `*.service.ts` | শুধু DB logic (Prisma query) |

প্রতিটা module এর জন্য **minimum এই ৩টা file** লাগবে।

Folder: `src/app/module/specialty/`

```
src/
├── app/
│   ├── lib/
│   │   └── prisma.ts
│   ├── module/
│   │   └── specialty/
│   │       ├── specialty.route.ts
│   │       ├── specialty.controller.ts
│   │       └── specialty.service.ts
│   └── routes/
│       └── index.ts
├── app.ts
└── server.ts
```

> ✏️ নোটে `src/module` লেখা ছিল, কিন্তু `routes/index.ts` এ import `../module/...` — তাই module folder টা আসলে **`src/app/module`** এর ভেতরে।

---

## ২. specialty.service.ts

```ts
import { prisma } from "@/app/lib/prisma";
import type { Prisma, Specialty } from "@/generated/prisma/client";

const createSpecialty = async (
	payload: Prisma.SpecialtyCreateInput,
): Promise<Specialty> => {
	const specialty = await prisma.specialty.create({
		data: payload,
	});
	return specialty;
};

const getAllSpecialties = async (): Promise<Specialty[]> => {
	const specialties = await prisma.specialty.findMany();
	return specialties;
};

const deleteSpecialty = async (id: string): Promise<Specialty> => {
	const specialty = await prisma.specialty.delete({
		where: { id },
	});
	return specialty;
};

export const SpecialtyService = {
	createSpecialty,
	getAllSpecialties,
	deleteSpecialty,
};
```

> ✏️ `payload: Specialty` → `Prisma.SpecialtyCreateInput` করলাম।
> `Specialty` type এ `id`, `createdAt`, `updatedAt` সব **required** — কিন্তু create করার সময় এগুলো আমরা দেই না, Prisma নিজে বসায়। `SpecialtyCreateInput` এ শুধু create এর জন্য দরকারি field থাকে।

> 💡 Model এ `isDeleted` আছে, তাই পরে চাইলে **soft delete** করা যায়:
> `delete` এর বদলে `update({ where: { id }, data: { isDeleted: true, deletedAt: new Date() } })`
> আর `findMany({ where: { isDeleted: false } })`

---

## ৩. specialty.controller.ts

`async/await` use করলে **try/catch** দিবো।

```ts
import type { Request, Response } from "express";
import { SpecialtyService } from "./specialty.service";

const createSpecialty = async (req: Request, res: Response) => {
	try {
		const payload = req.body;
		const result = await SpecialtyService.createSpecialty(payload);

		res.status(201).json({
			success: true,
			message: "Specialty created successfully",
			data: result,
		});
	} catch (error) {
		console.error(error);
		res.status(500).json({
			success: false,
			message: "Specialty create failed",
			error: error instanceof Error ? error.message : "unknown error",
		});
	}
};

const getAllSpecialties = async (_req: Request, res: Response) => {
	try {
		const result = await SpecialtyService.getAllSpecialties();

		res.status(200).json({
			success: true,
			message: "Specialties fetched successfully",
			data: result,
		});
	} catch (error) {
		console.error(error);
		res.status(500).json({
			success: false,
			message: "Specialties fetch failed",
			error: error instanceof Error ? error.message : "unknown error",
		});
	}
};

const deleteSpecialty = async (req: Request<{ id: string }>, res: Response) => {
	try {
		const { id } = req.params;
		const result = await SpecialtyService.deleteSpecialty(id);

		res.status(200).json({
			success: true,
			message: "Specialty deleted successfully",
			data: result,
		});
	} catch (error) {
		console.error(error);
		res.status(500).json({
			success: false,
			message: "Specialty delete failed",
			error: error instanceof Error ? error.message : "unknown error",
		});
	}
};

export const SpecialtyController = {
	createSpecialty,
	getAllSpecialties,
	deleteSpecialty,
};
```

**যা ঠিক করা হলো:**
- `res.status(201)` শুধু **create** এর জন্য। GET আর DELETE এ **`200`** হবে।
  - `200` OK → সফল (get, update, delete)
  - `201` Created → নতুন কিছু তৈরি হয়েছে
- `catch (error: any)` → `catch (error)`। `strict` mode এ `error` এমনিতেই `unknown`, আর `instanceof Error` দিয়ে check করছি — তাই `any` লাগে না।
- `// oxlint-disable-next-line no-unused-vars` সরিয়ে দিলাম — `error` তো use হচ্ছে, তাই এই comment অপ্রয়োজনীয়।
- `console.log` → `console.error` (আমাদের oxlint `no-console` rule এ শুধু `warn`/`error` allowed)
- `getAllSpecialties` এ `req` use হয় না → `_req`
- `id as string` এর বদলে `Request<{ id: string }>` — এতে TypeScript নিজেই জানে `id` হলো `string`, জোর করে cast লাগে না।

---

## ৪. specialty.route.ts

```ts
import { Router } from "express";
import { SpecialtyController } from "./specialty.controller";

const router = Router();

router.post("/", SpecialtyController.createSpecialty);
router.get("/", SpecialtyController.getAllSpecialties);
router.delete("/:id", SpecialtyController.deleteSpecialty);

export const SpecialtyRoutes = router;
```

`/:id` → URL এর এই অংশ `req.params.id` তে আসে।

---

## ৫. Route Management — `src/app/routes/index.ts`

প্রতিটা route `app.ts` এ লিখলে, module বাড়লে `app.ts` অনেক লম্বা হয়ে যাবে:

```ts
// ❌ এভাবে না
app.use("/api/v1/specialties", SpecialtyRoutes);
app.use("/api/v1/doctors", DoctorRoutes);
app.use("/api/v1/users", UserRoutes);
// ...
```

তাই সব route এক জায়গায় রাখবো:

```ts
import { Router } from "express";
import { SpecialtyRoutes } from "@/app/module/specialty/specialty.route";

const router = Router();

router.use("/specialties", SpecialtyRoutes);
// নতুন module এলে শুধু এখানে এক লাইন যোগ করবো

export const IndexRoutes = router;
```

---

## ৬. app.ts

```ts
import cors from "cors";
import express from "express";
import type { Request, Response } from "express";
import { IndexRoutes } from "@/app/routes";

const app = express();

app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.use("/api/v1", IndexRoutes); // ✅ সব route একসাথে

app.get("/", (_req: Request, res: Response) => {
	res.json({ message: "Welcome to phdoc!" });
});

export default app;
```

> ✏️ `IndexRoutes` এর import লাইনটা নোটে ছিল না — যোগ করলাম।
> `"@/app/routes"` লিখলে folder এর `index.ts` নিজে থেকেই load হয়।
> আগের test এর জন্য বানানো `app.post("/specialty", ...)` route টা এখন মুছে দাও।

---

## ৭. API Endpoints

| Method | URL | কাজ | Status |
|---|---|---|---|
| POST | `/api/v1/specialties` | Create | 201 |
| GET | `/api/v1/specialties` | Get all | 200 |
| DELETE | `/api/v1/specialties/:id` | Delete | 200 |

Create body (JSON):

```json
{
	"title": "Cardiology",
	"description": "Heart related treatment"
}
```

> ⚠️ ভুল `id` দিয়ে delete করলে Prisma `P2025` (Record not found) error দেবে → এখন `500` আসবে। পরে global error handler বানিয়ে এটাকে `404` করা যাবে।
