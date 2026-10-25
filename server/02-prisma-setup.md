# 02 - Connecting Prisma With The Project & Creating Specialty Model

> Module: 37-06
> Prisma 8 এখনো stable না, তাই **Prisma 7** use করবো।

---

## ১. Install

```shell
pnpm add -D prisma@7 @types/pg
pnpm add @prisma/client@7 @prisma/adapter-pg pg
```

> ⚠️ `prisma` (CLI) আর `@prisma/client` এর version **একই** হতে হবে — তাই দুটোতেই `@7` দিলাম (`prisma@prev` না)।

- `@prisma/adapter-pg` + `pg` → Prisma 7 এ database connect হয় **driver adapter** দিয়ে
- `@types/pg` → `pg` এর TypeScript types

---

## ২. Prisma init

```shell
pnpm prisma init --datasource-provider postgresql --output ../src/generated/prisma --skills=none --no-postinstall
```

- `--output` → generated client কোথায় যাবে (`src/generated/prisma`)
- `--skills=none` → AI skill file গুলো তৈরি হবে না

এটা বানাবে: `prisma/schema.prisma`, `prisma7.config.ts`, `.env`

---

## ৩. .env

`DATABASE_URL` auto add হয়, কিন্তু নিজের মতো বদলে নাও:

```dotenv
PORT=5000
DATABASE_URL="postgresql://postgres:আপনার_password@localhost:5432/phdoc?schema=public"
```

Format: `postgresql://USER:PASSWORD@HOST:PORT/DB_NAME?schema=public`

---

## ৪. Multi-file schema (schema folder)

1. `prisma/` এর ভেতরে `schema/` folder বানাও
2. `prisma/schema.prisma` → সরিয়ে `prisma/schema/schema.prisma` এ রাখো
3. Content (শুধু `output` path বদলেছে, কারণ file এক level ভেতরে গেছে):

```prisma
generator client {
  provider = "prisma-client"
  output   = "../../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

> Prisma 7 এ `datasource` এর ভেতরে `url` লেখা হয় না — URL আসে `prisma7.config.ts` থেকে।

প্রতিটা model আলাদা file এ থাকবে। এখন structure:

```
prisma/
└── schema/
    ├── schema.prisma      # generator + datasource
    ├── specialty.prisma   # এখন বানাচ্ছি
    └── migrations/        # migrate চালালে তৈরি হবে
```

(পরে `enums.prisma`, `user.prisma`, `doctor.prisma` ইত্যাদি এখানেই যোগ হবে।)

---

## ৫. prisma7.config.ts (project root)

```ts
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema", // file না, পুরো folder
  migrations: {
    // path: "prisma/schema/migrations", // eta dile schema folder er vitore migrations folder o hobe som amra folder er baire rakhte chaichi 
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env["DATABASE_URL"],
  },
});
```

> চেক করতে: `pnpm prisma validate` → প্রথম লাইনে দেখাবে `Loaded Prisma config from prisma7.config.ts.`

`tsconfig.json` এ include করো:

```json
"include": ["src/**/*", "tsdown.config.ts", "prisma7.config.ts"]
```

---

## ৬. Specialty model — `prisma/schema/specialty.prisma`

```prisma
model Specialty {
  id          String  @id @default(uuid())
  title       String  @unique
  description String? @db.Text
  icon        String? @db.VarChar(255)

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // soft delete
  isDeleted Boolean   @default(false)
  deletedAt DateTime?

  @@index([isDeleted], name: "idx_specialty_isDeleted")
  @@map("specialties")
}
```

- `?` → optional (null হতে পারে)
- `@db.Text` / `@db.VarChar(255)` → PostgreSQL এর আসল column type
- `@updatedAt` → update হলে auto time বসে
- `isDeleted` + `deletedAt` → **soft delete** (data মুছে না ফেলে শুধু mark করা)
- `@@map("specialties")` → DB তে table এর নাম `specialties`

> ✏️ `@@index([title])` সরিয়ে দিলাম — `@unique` নিজেই index বানায়, তাই আলাদা index লাগে না।

---

## ৭. Migration আর Generate

```shell
pnpm prisma migrate dev --name add_specialty_schema
pnpm prisma generate
```

- `migrate dev` → DB তে table বানায় + migration file তৈরি করে
- `generate` → `src/generated/prisma` এ TypeScript client বানায়
- Prisma 7 এ `migrate dev` নিজে `generate` চালায় না, তাই আলাদা করে চালাতে হয়

`.gitignore` এ যোগ করো (generated code git এ রাখার দরকার নেই):

```gitignore
src/generated
```

---

## ৮. Prisma client — `src/app/lib/prisma.ts`

এটা দিয়েই সব query লিখবো।

```ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../../generated/prisma/client";

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });

export const prisma = new PrismaClient({ adapter });
```

- `PrismaPg` → `pg` driver দিয়ে connect করে
- `!` → TypeScript কে বলা "এটা undefined হবে না"

---

## ৯. package.json scripts

```json
"migrate": "prisma migrate dev",
"generate": "prisma generate",
"studio": "prisma studio"
```

ব্যবহার: `pnpm migrate --name xyz`, `pnpm generate`, `pnpm studio` (browser এ DB দেখা)

---

## ১০. Test — একটা specialty create করে দেখি

`src/app.ts` এ:

```ts
import { prisma } from "./app/lib/prisma"; // ✅ import লাগবে

app.post("/specialty", async (_req: Request, res: Response) => {
  const specialty = await prisma.specialty.create({
    data: { title: "Neurology" },
  });
  res.status(201).json({ message: "specialty created", data: specialty });
});
```

Postman / Thunder Client দিয়ে `POST http://localhost:5000/specialty` → `201` আসলে setup ঠিক আছে ✅
`pnpm studio` দিয়েও দেখতে পারো data DB তে ঢুকেছে কিনা।

> ⚠️ `title` হলো `@unique`, তাই দ্বিতীয়বার একই request দিলে error আসবে (`P2002` — Unique constraint failed)। এটা স্বাভাবিক।
> ✏️ এখানে `req` ব্যবহার হচ্ছে না, তাই `_req` লিখলাম।
