প্রথমে file structure:
```text
clinicos/
├── apps/
│   ├── api/        # Express + Prisma (backend)
│   └── web/        # Next.js 16 (frontend)
├── docs/
│   └── sdlc.md
├── docker-compose.yml
├── docker-compose.prod.yml
└── README.md
```

clinicos/.gitignore

```gitignore
# ── OS / Editor ─────────────────────────
.DS_Store
Thumbs.db
.idea/
.vscode/*
!.vscode/settings.json
!.vscode/extensions.json

# ── Secrets (never commit) ──────────────
.env
.env.*
!.env.example

# ── Dependencies (safety net) ───────────
node_modules/

# ── Logs ────────────────────────────────
*.log
npm-debug.log*
pnpm-debug.log*

# ── Database backups (patient data!) ────
backups/
*.dump
*.sql.gz
```

apps/api-এর .gitignore
```gitignore
# ── Dependencies ────────────────────────
node_modules/

# ── Build output ────────────────────────
dist/
*.tsbuildinfo

# ── Prisma generated client ─────────────
src/generated/

# ── Tests ───────────────────────────────
coverage/

# ── Uploads / generated PDFs (local dev) ─
uploads/
tmp/

# ── vscode ───────────────────────────────
.vscode
/src/generated/prisma
```

## clinicos-এর folder open করে
```bash
git init
git add .
git commit -m "chore: initialize repository with root .gitignore"
git branch -M main
```

## এবার api folder open করে

```bash
pnpm init
pnpm add -D typescript @types/node tsx tsdown

# eta deyar age package json e "type": "module" dite hobe
pnpm exec tsc --init
```

tsconfig.json হবে
```jsonc
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
	"include": ["src/**/*", "tsdown.config.ts", "prisma.config.ts"],
	"exclude": ["node_modules", "dist"]
}
```

## tsdown.config.ts
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

পরে email template-এর জন্য copy যোগ হয়েছে — update করা config দেখো নোট ২০-এর ধাপ ৬-এ।

## Lint — oxlint
```jsonc
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
		"no-console": ["warn", { "allow": ["info", "warn", "error"] }],
		"typescript/no-floating-promises": "error",
		"typescript/consistent-type-imports": "error",
		"import/no-unassigned-import": ["warn", { "allow": ["dotenv/config"] }]
	},
	"ignorePatterns": ["dist/**", "coverage/**", "src/generated/**"]
}
```

## ৭. package.json scripts

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


## Express, CORS, dotenv
```bash
pnpm add express cors dotenv
pnpm add -D @types/express @types/cors
```

.env (project root-এ, git-এ যাবে না):

```env
PORT=5000
```

server.ts

```ts
import type { Server } from "node:http";
import app from "./app";

const PORT = Number(process.env.PORT) || 5000;
const SHUTDOWN_TIMEOUT_MS = 10_000;

let server: Server | undefined;

const startServer = () => {
	server = app.listen(PORT, (error) => {
		if (error) {
			console.error("Failed to start server:", error);
			process.exit(1);
		}
		console.info(`ClinicOS API running on http://localhost:${PORT}`);
	});
};

const shutdown = (signal: NodeJS.Signals) => {
	console.info(`${signal} received, shutting down gracefully...`);

	if (!server) {
		process.exit(0);
	}

	server.close((error) => {
		if (error) {
			console.error("Error during shutdown:", error);
			process.exit(1);
		}
		console.info("Server closed");
		process.exit(0);
	});

	setTimeout(() => {
		console.error("Forced shutdown after timeout");
		process.exit(1);
	}, SHUTDOWN_TIMEOUT_MS).unref();
};

process.on("SIGTERM", shutdown);
process.on("SIGINT", shutdown);

process.on("unhandledRejection", (reason) => {
	console.error("Unhandled rejection:", reason);
	process.exit(1);
});

process.on("uncaughtException", (error) => {
	console.error("Uncaught exception:", error);
	process.exit(1);
});

startServer();
```

server.ts
```ts
import type { Request, Response } from "express";
import express from "express";

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.get("/", (_req: Request, res: Response) => {
	res.json({ message: "ClinicOS API is running" });
});

app.get("/api/v1/health", (_req: Request, res: Response) => {
	res.json({
		status: "ok",
		uptime: process.uptime(),
		timestamp: new Date().toISOString(),
	});
});

export default app;
```