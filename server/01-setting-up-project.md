# 01 - Setting Up Project (phdoc-server)

Stack: **Node 24 + TypeScript + Express + pnpm**
Dev-এ `tsx`, build-এ `tsdown`, lint-এ `oxlint`।

---

## ১. Folder, .gitignore, Git

```shell
mkdir phdoc-server && cd phdoc-server
git init
```

`.gitignore` (stack অনুযায়ী):

```gitignore
node_modules
dist
coverage
.env
*.log
.DS_Store
```

তারপর GitHub-এ repo বানিয়ে push করো।

---

## ২. package.json + TypeScript

```shell
pnpm init
pnpm add -D typescript @types/node tsx tsdown
pnpm tsc --init
```

> ⚠️ `tsc --init` চালানোর আগে `typescript` install করতে হবে। আর `"types": ["node"]` এর জন্য `@types/node` লাগবে।

`package.json`-এ **অবশ্যই** `"type": "module"` দাও (ESM ব্যবহার করছি):

```json
"type": "module"
```

---

## ৩. module কেন গুরুত্বপূর্ণ (ছোট করে)

| `module` | কী করে | সমস্যা |
|---|---|---|
| `commonjs` | output-এ `require()` / `module.exports` | `"type": "module"` এর সাথে conflict |
| `esnext` / `es2022` | output-এ `import/export` | Node-এর নিয়ম check করে না, runtime error হতে পারে |
| `nodenext` | Node-এর আসল নিয়ম মানে | import-এ `.js` extension লিখতে হয় |

```ts
import { prisma } from "./lib/prisma.js"; // ✅ nodenext এ লাগবে
import { prisma } from "./lib/prisma";    // ❌ nodenext এ error
```

`.js` লিখতে চাই না → তাই **tsc দিয়ে build না করে `tsdown` (bundler) দিয়ে build** করবো, আর `tsc` শুধু type check করবে।

---

## ৪. tsconfig.json (Bundler mode — final)

```json
{
	"compilerOptions": {
		// Bundler mode: import-এ extension লাগবে না
		"module": "preserve",
		"moduleResolution": "bundler",
		"target": "es2024",
		"lib": ["es2024"],
		"types": ["node"],
		"noEmit": true, // tsc শুধু check করবে, build করবে tsdown

		// Path alias
		"paths": {
			"@/*": ["./src/*"]
		},

		// Strictness
		"strict": true,
		"noUncheckedIndexedAccess": true,
		"noImplicitReturns": true,
		"noImplicitOverride": true,
		"noFallthroughCasesInSwitch": true,

		// Module behavior
		"verbatimModuleSyntax": true,
		"isolatedModules": true,
		"moduleDetection": "force",
		"resolveJsonModule": true,
		"esModuleInterop": true,
		"forceConsistentCasingInFileNames": true,
		"skipLibCheck": true
	},
	"include": ["src/**/*", "tsdown.config.ts"],
	"exclude": ["node_modules", "dist"]
}
```

`verbatimModuleSyntax: true` → type import করলে `import type` লিখতে হবে:

```ts
import type { Request, Response } from "express";
```

---

## ৫. tsdown.config.ts

```ts
import { defineConfig } from "tsdown";

export default defineConfig({
	entry: ["src/server.ts"], // ✅ entry হবে server.ts (index.ts না)
	format: "esm",
	platform: "node",
	target: "node24",
	outDir: "dist",
	clean: true,
	sourcemap: true,
	fixedExtension: false, // output: dist/server.js (server.mjs না)
});
```

> পরে email template ইত্যাদি add করলে এই config update লাগতে পারে।

এখন import এভাবে লেখা যাবে:

```ts
import { prisma } from "./lib/prisma"; // ✅ extension ছাড়া
import { auth } from "@/lib/auth";     // ✅ alias, ../../ লাগবে না
```

---

## ৬. Lint — oxlint

(Teacher `eslint` use করছে, আমি `oxlint` use করছি — দ্রুত।)

```shell
pnpm add -D oxlint oxlint-tsgolint
```

`.oxlintrc.json` (project root-এ):

```json
{
	"$schema": "./node_modules/oxlint/configuration_schema.json",
	"plugins": ["typescript", "unicorn", "oxc", "import", "node"],
	"env": {
		"node": true,
		"es2024": true
	},
	"options": {
		"typeAware": true
	},
	"categories": {
		"correctness": "error",
		"suspicious": "warn"
	},
	"rules": {
		"no-unused-vars": ["warn", { "argsIgnorePattern": "^_", "varsIgnorePattern": "^_" }],
		"no-console": ["warn", { "allow": ["warn", "error"] }],
		"typescript/no-floating-promises": "error",
		"typescript/consistent-type-imports": "error",
		"import/no-unassigned-import": ["warn", { "allow": ["dotenv/config"] }]
	},
	"ignorePatterns": ["dist/**", "coverage/**", "src/generated/**"]
}
```

> `no-console` rule এর কারণে `console.log` দিলে warning আসবে — চাইলে `"info"` allow list-এ যোগ করো।

---

## ৭. package.json scripts (সব একসাথে, final)

```json
"type": "module",
"main": "dist/server.js",
"scripts": {
	"dev": "tsx watch src/server.ts",
	"typecheck": "tsc --noEmit",
	"lint": "oxlint",
	"lint:fix": "oxlint --fix",
	"check": "pnpm typecheck && pnpm lint",
	"build": "pnpm check && tsdown",
	"start": "node dist/server.js"
}
```

> ⚠️ আগের নোটে `build` দুইবার ছিল — JSON-এ একই key দুইবার রাখা যায় না, তাই একটাই রাখলাম।

---

## ৮. Express, CORS, dotenv

```shell
pnpm add express cors dotenv
pnpm add -D @types/express @types/cors
```

`.env` (project root-এ, git-এ যাবে না):

```env
PORT=5000
```

---

## ৯. src/server.ts

```ts
import "dotenv/config"; // সবার আগে .env load
import type { Server } from "node:http";
import app from "./app";

const PORT = Number(process.env.PORT) || 5000;
let server: Server;

const startServer = async () => {
	try {
		server = app.listen(PORT, () => {
			console.info(`Server is running on port ${PORT}`);
		});
	} catch (error) {
		console.error(error);
		process.exit(1);
	}
};

void startServer(); // ✅ no-floating-promises error এড়াতে
```

**যা ঠিক করা হলো:**
- `"http"` → `"node:http"` (Node built-in module এর modern নিয়ম)
- `startServer();` → `void startServer();` — না হলে `typescript/no-floating-promises` **error** দেবে
- `PORT` এখন `.env` থেকে আসে (dotenv)
- error হলে `console.error` + `process.exit(1)`

---

## ১০. src/app.ts

```ts
import cors from "cors";
import express from "express";
import type { Request, Response } from "express";

const app = express();

app.use(cors());
app.use(express.json());                         // JSON body পড়ার জন্য
app.use(express.urlencoded({ extended: true })); // form data পড়ার জন্য

app.get("/", (_req: Request, res: Response) => {
	res.json({ message: "Welcome to phdoc!" });
});

export default app;
```

> `req` ব্যবহার হচ্ছে না, তাই `_req` লিখলাম — না হলে `no-unused-vars` warning আসবে।

---

## ১১. Folder Structure

```
phdoc-server/
├── src/
│   ├── app/          # modules, lib, middlewares... সব এখানে
│   │   ├── modules/
│   │   └── lib/
│   ├── app.ts        # express app
│   └── server.ts     # server start (entry)
├── .env
├── .gitignore
├── .oxlintrc.json
├── package.json
├── tsconfig.json
└── tsdown.config.ts
```

> নোট: `src/app.ts` file আর `src/app/` folder একই নামে — `import app from "./app"` তখন `app.ts` file কেই ধরে, তাই কাজ করবে।

---

## Run

```shell
pnpm dev     # development
pnpm build   # check + build
pnpm start   # production
```
