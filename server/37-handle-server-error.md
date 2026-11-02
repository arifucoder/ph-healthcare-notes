# 37 - Handle Server Error (server.ts)

`server.ts`-এ server চালু করা, বন্ধ করা আর হঠাৎ আসা error সামলানোর code-টা আরও শক্ত করা হয়েছে। Port আগে থেকে ব্যবহার হলে, বন্ধ করার সময় কিছু fail করলে, বা কোনো error ধরা না পড়লে server যেন নিরাপদে বন্ধ হয়, সেটাই উদ্দেশ্য।

```
pnpm dev → startServer() → seedSuperAdmin() → app.listen
        ├─ listen fail (EADDRINUSE) → gracefulShutdown(1)
        └─ সফল → startCronJobs()

Ctrl+C / SIGTERM / uncaughtException / unhandledRejection
        → gracefulShutdown → cron বন্ধ → server.close → prisma.$disconnect → process.exit
```

> আগের version আর seed-এর অংশ আছে [নোট ২৯](./29-seed-super-admin.md)-এ। Cron-এর কাজ [নোট ২৮](./28-stripe-payment.md)-এ।

### নোট ২৯-এর তুলনায় কী বদলেছে

| অংশ | আগে | এখন | কেন |
| --- | --- | --- | --- |
| `PORT` | `envVars.PORT` (string) | `Number(envVars.PORT)` | `listen`-এ number দেওয়াই ঠিক |
| Timeout | `10_000` সরাসরি লেখা | `SHUTDOWN_TIMEOUT_MS` const | এক জায়গায় নাম সহ, বদলানো সহজ |
| জোর করে বন্ধ | চুপচাপ `process.exit` | আগে `console.error` log | কেন জোর করে বন্ধ হলো বোঝা যায় |
| Shutdown-এর ভেতরে error | ধরা হতো না | `try / catch / finally` | কিছু fail করলেও শেষে `process.exit` হবেই, আর exit code `1` |
| `server.close` | error উপেক্ষা | error হলে `reject` | close fail-ও `catch`-এ ধরা পড়ে |
| Server চালু কিনা | `if (runningServer)` | `if (runningServer?.listening)` | listen fail করলে server object থাকে কিন্তু চালু থাকে না, তখন `close` করলে error দেয় |
| Listen error | `server.on("error", ...)` | `app.listen(PORT, (error) => ...)` | Express 5-এ listen-এর error callback-এই আসে |
| শেষে log | নেই | `"Server closed gracefully."` | ঠিকঠাক বন্ধ হলো কিনা বোঝা যায় |

---

## ধাপ ১: পুরো `src/server.ts`

```ts
import type { Server } from "http";
import { schedule, shutdown as shutdownCron } from "node-cron";
import app from "./app";
import { envVars } from "./app/config/env";
import { prisma } from "./app/lib/prisma";
import { AppointmentService } from "./app/module/appointment/appointment.service";
import { seedSuperAdmin } from "./app/utils/seed";

const PORT = Number(envVars.PORT);
const SHUTDOWN_TIMEOUT_MS = 10_000;

let server: Server | undefined;
let isShuttingDown = false;

const startCronJobs = () => {
	// প্রতি ৫ মিনিটে: ৩০ মিনিটের পুরনো unpaid appointment cancel করে slot ফাঁকা করা
	schedule(
		"*/5 * * * *",
		async () => {
			try {
				const { canceledCount } = await AppointmentService.cancelUnpaidAppointments();

				if (canceledCount > 0) {
					console.warn(`[cron] Canceled ${canceledCount} unpaid appointment(s)`);
				}
			} catch (error) {
				console.error("[cron] Failed to cancel unpaid appointments:", error instanceof Error ? error.message : error);
			}
		},
		{
			name: "cancel-unpaid-appointments",
			noOverlap: true, // আগের run শেষ না হলে নতুন run শুরু হবে না
		},
	);
};

const gracefulShutdown = async (reason: string, exitCode: number) => {
	// একাধিক signal একসাথে এলে দুইবার shutdown হবে না
	if (isShuttingDown) return;
	isShuttingDown = true;

	console.warn(`${reason}. Shutting down server...`);

	// কোনো কারণে আটকে গেলে জোর করে বন্ধ
	setTimeout(() => {
		console.error("Graceful shutdown timed out. Forcing exit.");
		process.exit(exitCode);
	}, SHUTDOWN_TIMEOUT_MS).unref();

	let finalExitCode = exitCode;

	try {
		await shutdownCron();

		const runningServer = server;
		// listen fail করলে server চালুই হয়নি, close করার কিছু নেই
		if (runningServer?.listening) {
			// নতুন request নেওয়া বন্ধ, চলমান request শেষ হওয়া পর্যন্ত অপেক্ষা
			await new Promise<void>((resolve, reject) => {
				runningServer.close((error) => (error ? reject(error) : resolve()));
			});
		}

		await prisma.$disconnect();
		console.log("Server closed gracefully.");
	} catch (error) {
		console.error("Error during shutdown:", error);
		finalExitCode = 1;
	} finally {
		process.exit(finalExitCode);
	}
};

const startServer = async () => {
	try {
		await seedSuperAdmin();
	} catch (error) {
		console.error("Failed to seed super admin:", error);
		await gracefulShutdown("Seeding failed", 1);
		return;
	}

	// Express 5: listen fail করলে (যেমন EADDRINUSE) এই callback-টাই error সহ ডাকা হয়
	server = app.listen(PORT, (error) => {
		if (error) {
			console.error("Failed to start server:", error);
			void gracefulShutdown("Server failed to start", 1);
			return;
		}

		console.log(`Server is running on http://localhost:${PORT}`);
		startCronJobs();
	});
};

process.on("SIGTERM", () => void gracefulShutdown("SIGTERM received", 0));
process.on("SIGINT", () => void gracefulShutdown("SIGINT received", 0));

process.on("uncaughtException", (error) => {
	console.error("Uncaught Exception:", error);
	void gracefulShutdown("Uncaught exception", 1);
});

process.on("unhandledRejection", (reason) => {
	console.error("Unhandled Rejection:", reason);
	void gracefulShutdown("Unhandled rejection", 1);
});

// package.json-এ "type": "module", তাই top-level await চলে
await startServer();
```

---

## ধাপ ২: অংশগুলো বুঝে নেওয়া

### `gracefulShutdown`-এর ক্রম

| ক্রম | কাজ | কেন এই ক্রমে |
| --- | --- | --- |
| ১ | `isShuttingDown` check | দুইবার `Ctrl+C` বা একসাথে দুই signal → shutdown একবারই |
| ২ | Timeout timer চালু (`unref`) | কিছু আটকে গেলে ১০ সেকেন্ড পরে জোর করে বন্ধ। `unref` মানে এই timer নিজে process-কে চালু রাখবে না |
| ৩ | `shutdownCron()` | বন্ধের সময় নতুন cron run যেন DB-তে কাজ শুরু না করে |
| ৪ | `server.close()` | নতুন request নেওয়া বন্ধ, চলমান request শেষ হওয়া পর্যন্ত অপেক্ষা |
| ৫ | `prisma.$disconnect()` | সব request শেষ হওয়ার **পরে** DB বন্ধ — না হলে চলমান request DB পাবে না |
| ৬ | `finally` → `process.exit` | কিছু fail করলেও process অবশ্যই বন্ধ হবে |

### `try / catch / finally` আর `finalExitCode`

```ts
let finalExitCode = exitCode; // ধরো Ctrl+C → 0

try {
	// ... কোনো ধাপ fail করলে
} catch (error) {
	finalExitCode = 1; // স্বাভাবিক বন্ধ হলেও fail হলে 1
} finally {
	process.exit(finalExitCode); // সবসময় চলবে
}
```

Exit code `0` মানে সব ঠিক আছে, আর `1` মানে কিছু সমস্যা হয়েছে। Docker বা Render এই code দেখে বোঝে server ঠিকমতো বন্ধ হয়েছে কিনা।

### `server.close`-কে Promise বানানো

`server.close()` callback দিয়ে কাজ করে, `await` করা যায় না। তাই `new Promise` দিয়ে মুড়ে নেওয়া হয়েছে: error এলে `reject`, না এলে `resolve`। `reject` হলে সেটা নিচের `catch`-এ ধরা পড়ে।

### Express 5-এর `app.listen` callback

Express 5-এ `app.listen`-এর callback-এ প্রথম argument হিসেবে `error` আসে (যেমন port আগে থেকে ব্যবহার হলে `EADDRINUSE`)। তাই আলাদা `server.on("error", ...)` লাগে না। তবে error এলে `server` object তৈরি হয়ে যায় কিন্তু চালু থাকে না, তাই shutdown-এ `listening` check করা হয়।

### Process event-গুলো

| Event | কখন | Exit code |
| --- | --- | --- |
| `SIGINT` | Terminal-এ `Ctrl+C` | `0` |
| `SIGTERM` | Hosting (Render/Docker) বা `kill` দিয়ে বন্ধ করলে | `0` |
| `uncaughtException` | `try/catch` ছাড়া sync error | `1` |
| `unhandledRejection` | `.catch` / `try/catch` ছাড়া reject হওয়া promise | `1` |

> `void gracefulShutdown(...)` — promise-এর ফল ব্যবহার করছি না জানাতে `void`। এতে oxlint-এর `no-floating-promises` rule error দেয় না।

---

## 🧪 Test

| Case | কীভাবে | Terminal-এ যা দেখবে |
| --- | --- | --- |
| স্বাভাবিক চালু | `pnpm dev` | `Server is running on http://localhost:5000` |
| `Ctrl+C` | চলা অবস্থায় `Ctrl+C` | `SIGINT received. Shutting down server...` → `Server closed gracefully.` |
| Port আগে থেকে ব্যবহার হচ্ছে | দুইটা terminal-এ একসাথে `pnpm dev` (অথবা `.env`-এ অন্য চালু app-এর port) | `Failed to start server: ... EADDRINUSE` → `Server failed to start. Shutting down server...` |
| Seed fail | [নোট ২৯](./29-seed-super-admin.md)-এর fail case | `Failed to seed super admin` → `Seeding failed. Shutting down server...` |
| Unhandled rejection | কোথাও test-এর জন্য `void Promise.reject(new Error("test"))` লিখে চালাও (পরে মুছে দিও) | `Unhandled Rejection: Error: test` → shutdown |

> Exit code দেখতে (macOS/Linux): server বন্ধ হওয়ার পরে `echo $?`। `Ctrl+C`-এ `0`, fail হলে `1` আসার কথা। `tsx watch` নিজে process চালায়, তাই exit code ঠিক দেখতে `pnpm build` করে `node dist/server.js` দিয়ে চালানো ভালো।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **দুইটা `console.log` → oxlint warning/error** — `"Server closed gracefully."` আর `"Server is running..."`-এ `console.log` আছে, কিন্তু আমাদের oxlint-এর `no-console` শুধু `warn`/`error` allow করে:
   ```ts
   console.warn("Server closed gracefully.");
   console.warn(`Server is running on http://localhost:${PORT}`);
   ```

2. **Timeout হলে exit code `1` দাও** — এখন timeout হলেও `exitCode` (যেমন `Ctrl+C`-এ `0`) দিয়ে বন্ধ হয়। কিন্তু timeout মানেই কিছু আটকে গিয়েছিল, তাই এটাকে fail হিসেবে ধরা উচিত:
   ```ts
   setTimeout(() => {
   	console.error("Graceful shutdown timed out. Forcing exit.");
   	process.exit(1);
   }, SHUTDOWN_TIMEOUT_MS).unref();
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Keep-alive connection-এ `server.close` দেরি করতে পারে** — browser বা Postman connection খোলা রাখলে `server.close()` সেগুলো শেষ হওয়া পর্যন্ত অপেক্ষা করে, তখন ১০ সেকেন্ডের timeout-এ গিয়ে বন্ধ হয়। Node 18.2+-এ `server.closeIdleConnections()` দিয়ে idle connection সাথে সাথে বন্ধ করা যায়:
   ```ts
   runningServer.close((error) => (error ? reject(error) : resolve()));
   runningServer.closeIdleConnections();
   ```

4. **`PORT` invalid হলে** — `.env`-এ `PORT=abc` থাকলে `Number("abc")` হয় `NaN`। এটা ভালোভাবে ধরতে হলে `env.ts`-এই port number কিনা check করা যায়।
