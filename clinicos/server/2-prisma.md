# Prisma setup

## ১. Prisma install

```bash
pnpm add -D prisma@7 @types/pg
pnpm add @prisma/client@7 @prisma/adapter-pg pg
```

## ২. Prisma init

```bash
pnpm prisma init --datasource-provider postgresql --output ../src/generated/prisma --no-skills
```

## ৩. tsconfig.json-এ prisma.config.ts যোগ

এখন prisma.config.ts file-এ datasource error দেবে। সেটা solve করার জন্য tsconfig.json file-এর include-এ add করতে হবে:

```jsonc
"include": ["prisma.config.ts"],
```

> **নোট:** আমরা prisma7.config.ts থেকে prisma.config.ts-এ convert করে নেব।

## ৪. .env

```env
NODE_ENV=development
PORT=5000
DATABASE_URL="postgresql://postgres:আপনার_password@localhost:5432/phdoc?schema=public"
```

## ৫. Multi-file schema (schema folder)

1. prisma/-এর ভেতরে schema/ folder বানাও।
2. prisma/schema.prisma সরিয়ে prisma/schema/schema.prisma-তে রাখো।

**File:** `prisma/schema/schema.prisma` (শুধু output path বদলেছে, কারণ file এক level ভেতরে গেছে):

```prisma
generator client {
  provider = "prisma-client"
  output   = "../../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

প্রতিটা model আলাদা file-এ থাকবে। এখন structure:

```text
prisma/
└── schema/
    ├── schema.prisma      # generator + datasource
    ├── specialty.prisma   # এখন বানাচ্ছি
└── migrations/        # migrate চালালে তৈরি হবে   
```

## ৬. prisma.config.ts

**File:** `prisma.config.ts`

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

## ৭. Specialty model — prisma/schema/clinic.prisma

```prisma
model Specialty {
  id          String   @id @default(uuid())
  name        String   @unique
  description String?
  slug        String   @unique
  icon        String?
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  isDeleted Boolean   @default(false)
  deletedAt DateTime?

  @@map("specialties")
}
```

## ৮. Migrate আর generate

```bash
pnpm prisma migrate dev --name add_specialty_schema
pnpm prisma generate

```

## ৯. .gitignore

.gitignore-এ যোগ করো (generated code git-এ রাখার দরকার নেই):

```gitignore
src/generated
```

## ১০. Prisma client — src/app/lib/prisma.ts

```ts
import "dotenv/config";
import { PrismaClient } from "@/generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });

export const prisma = new PrismaClient({ adapter });
```

## ১১. package.json scripts

```json
		"dev": "tsx watch --env-file=.env src/server.ts",
		"typecheck": "tsc",
		"lint": "oxlint --type-aware",
		"lint:fix": "oxlint --type-aware --fix",
		"check": "pnpm validate && pnpm typecheck && pnpm lint",
		"build": "tsdown",
		"start": "node --enable-source-maps dist/server.js",
		"generate": "prisma generate",
		"migrate": "prisma migrate dev",
		"migrate:reset": "prisma migrate reset",
		"validate": "prisma validate",
		"format": "prisma format",
		"seed": "tsx prisma/schema/seed.ts",
		"studio": "prisma studio"
```