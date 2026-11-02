# 38 - Packages ও Scripts (phdoc-server)

`phdoc-server`-এ যত package আর script আছে, সব এক জায়গায় রাখা হলো। প্রতিটা package কী কাজ করে, কীভাবে install করতে হয় আর তার doc কোথায়, সব এখানে পাবে।

| Type | কাজ | কখন লাগে |
| --- | --- | --- |
| `dependencies` | App চালাতে লাগে | Production server-এও install হয় |
| `devDependencies` | শুধু develop/build করার সময় লাগে (type, linter, compiler) | Production-এ লাগে না |

```shell
pnpm add <package>       # dependency
pnpm add -D <package>    # devDependency
pnpm remove <package>    # বাদ দেওয়া
pnpm install             # package.json দেখে সব install
```

---

## ধাপ ১: সব একসাথে install (নতুন project-এ)

```shell
# dependencies
pnpm add express cors cookie-parser dotenv http-status zod \
  @prisma/client @prisma/adapter-pg pg \
  better-auth jsonwebtoken \
  multer cloudinary \
  nodemailer ejs \
  stripe node-cron pdfkit uuid

# devDependencies
pnpm add -D typescript tsx tsdown prisma oxlint oxlint-tsgolint \
  @types/node @types/express @types/cors @types/cookie-parser \
  @types/jsonwebtoken @types/multer @types/ejs @types/pdfkit @types/pg
```

> `i` package ইচ্ছা করে ইনস্টল করা হয়নি, তাই এখানে রাখিনি। নিচের recommendation দেখো।

---

## ধাপ ২: Dependencies

| Package | Version | কী কাজ করে | কোথায় ব্যবহার | Install | Doc |
| --- | --- | --- | --- | --- | --- |
| `express` | `^5.3.0` | Web server ও routing framework | `app.ts`, সব route | `pnpm add express` | [expressjs.com](https://expressjs.com/) |
| `cors` | `^2.8.6` | অন্য domain (frontend) থেকে request আসতে দেয় | `app.ts` | `pnpm add cors` | [GitHub](https://github.com/expressjs/cors) |
| `cookie-parser` | `^1.4.7` | Request-এর cookie পড়ে `req.cookies`-এ রাখে | `app.ts`, `checkAuth` | `pnpm add cookie-parser` | [GitHub](https://github.com/expressjs/cookie-parser) |
| `dotenv` | `^18.0.9` | `.env` file থেকে `process.env`-এ value আনে | `config/env.ts` | `pnpm add dotenv` | [GitHub](https://github.com/motdotla/dotenv) |
| `http-status` | `^2.1.0` | Status code নাম দিয়ে লেখা যায় (`httpStatus.NOT_FOUND`) | প্রায় সব জায়গায় | `pnpm add http-status` | [GitHub](https://github.com/adaltas/node-http-status) |
| `zod` | `^4.6.7` | Request body validation আর type বের করা | `*.validation.ts`, `validateRequest` | `pnpm add zod` | [zod.dev](https://zod.dev/) |
| `@prisma/client` | `^7.10.1` | Prisma-র query চালানোর client | `lib/prisma.ts` | `pnpm add @prisma/client` | [prisma.io/docs](https://www.prisma.io/docs) |
| `@prisma/adapter-pg` | `^7.10.1` | Prisma 7-কে `pg` driver দিয়ে PostgreSQL-এ যুক্ত করে (`PrismaPg`) | `lib/prisma.ts` | `pnpm add @prisma/adapter-pg` | [Prisma docs](https://www.prisma.io/docs/orm/overview/databases/postgresql) |
| `pg` | `^8.23.1` | PostgreSQL-এর Node driver (adapter এটা ব্যবহার করে) | `@prisma/adapter-pg`-এর ভেতরে | `pnpm add pg` | [node-postgres.com](https://node-postgres.com/) |
| `better-auth` | `^1.7.12` | Signup/login, session, Google OAuth, email OTP | `lib/auth.ts`, auth module | `pnpm add better-auth` | [better-auth.com](https://www.better-auth.com/docs) |
| `jsonwebtoken` | `^9.0.3` | Access/refresh JWT বানানো ও verify করা | `utils/jwt.ts`, `checkAuth` | `pnpm add jsonwebtoken` | [GitHub](https://github.com/auth0/node-jsonwebtoken) |
| `multer` | `^2.5.0` | `multipart/form-data` (file upload) পড়ে | upload middleware | `pnpm add multer` | [GitHub](https://github.com/expressjs/multer) |
| `cloudinary` | `^2.11.0` | Image/PDF Cloudinary-তে upload ও delete | `config/cloudinary.config.ts` | `pnpm add cloudinary` | [Cloudinary Node docs](https://cloudinary.com/documentation/node_integration) |
| `nodemailer` | `^10.2.1` | SMTP দিয়ে email পাঠানো (নিজের type সহ আসে) | `utils/email.ts` | `pnpm add nodemailer` | [nodemailer.com](https://nodemailer.com/) |
| `ejs` | `^7.0.2` | Email-এর HTML template (`.ejs`) render | `utils/email.ts`, `templates/` | `pnpm add ejs` | [ejs.co](https://ejs.co/) |
| `stripe` | `^23.0.1` | Stripe Checkout, webhook verify, refund | `config/stripe.config.ts`, payment | `pnpm add stripe` | [Stripe API (Node)](https://docs.stripe.com/api?lang=node) |
| `node-cron` | `^4.6.0` | নির্দিষ্ট সময় পরপর কাজ চালানো (unpaid appointment cancel) | `server.ts` | `pnpm add node-cron` | [GitHub](https://github.com/node-cron/node-cron) |
| `pdfkit` | `^0.20.2` | Prescription আর invoice PDF বানানো | `utils/pdf.ts` | `pnpm add pdfkit` | [pdfkit.org](https://pdfkit.org/) |
| `uuid` | `^14.0.3` | DB-তে save-এর আগেই UUID v7 বানানো (নিজের type সহ আসে) | prescription service | `pnpm add uuid` | [GitHub](https://github.com/uuidjs/uuid) |

---

## ধাপ ৩: Dev Dependencies

| Package | Version | কী কাজ করে | Install | Doc |
| --- | --- | --- | --- | --- |
| `typescript` | `^7.0.2` | TypeScript compiler, `tsc --noEmit` দিয়ে type check | `pnpm add -D typescript` | [typescriptlang.org](https://www.typescriptlang.org/docs/) |
| `tsx` | `^4.23.15` | `.ts` file সরাসরি চালায়, `watch` দিয়ে auto restart (type check করে না) | `pnpm add -D tsx` | [tsx.is](https://tsx.is/) |
| `tsdown` | `^0.23.1` | Production build: `src/server.ts` → `dist/` (`tsdown.config.ts`) | `pnpm add -D tsdown` | [tsdown.dev](https://tsdown.dev/) |
| `prisma` | `^7.10.1` | Prisma CLI: migrate, generate, studio | `pnpm add -D prisma` | [Prisma CLI](https://www.prisma.io/docs/orm/reference/prisma-cli-reference) |
| `oxlint` | `^1.89.0` | খুব দ্রুত linter (`.oxlintrc.json`), ESLint-এর বদলে | `pnpm add -D oxlint` | [oxc.rs](https://oxc.rs/docs/guide/usage/linter) |
| `oxlint-tsgolint` | `^7.0.2005` | oxlint-এর type-aware rule (যেমন `no-floating-promises`) | `pnpm add -D oxlint-tsgolint` | [GitHub](https://github.com/oxc-project/tsgolint) |
| `@types/node` | `^26.6.5` | Node-এর built-in (`http`, `process`, `Buffer`) type | `pnpm add -D @types/node` | [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) |
| `@types/express` | `^5.0.6` | Express 5-এর type (`Request`, `Response`) | `pnpm add -D @types/express` | [npm](https://www.npmjs.com/package/@types/express) |
| `@types/cors` | `^2.8.19` | `cors`-এর type | `pnpm add -D @types/cors` | [npm](https://www.npmjs.com/package/@types/cors) |
| `@types/cookie-parser` | `^1.4.10` | `cookie-parser`-এর type | `pnpm add -D @types/cookie-parser` | [npm](https://www.npmjs.com/package/@types/cookie-parser) |
| `@types/jsonwebtoken` | `^9.0.10` | `jsonwebtoken`-এর type | `pnpm add -D @types/jsonwebtoken` | [npm](https://www.npmjs.com/package/@types/jsonwebtoken) |
| `@types/multer` | `^2.3.0` | `multer`-এর type (`req.file`, `req.files`) | `pnpm add -D @types/multer` | [npm](https://www.npmjs.com/package/@types/multer) |
| `@types/ejs` | `^3.1.5` | `ejs`-এর type | `pnpm add -D @types/ejs` | [npm](https://www.npmjs.com/package/@types/ejs) |
| `@types/pdfkit` | `^0.17.7` | `pdfkit`-এর type | `pnpm add -D @types/pdfkit` | [npm](https://www.npmjs.com/package/@types/pdfkit) |
| `@types/pg` | `^8.23.1` | `pg`-এর type | `pnpm add -D @types/pg` | [npm](https://www.npmjs.com/package/@types/pg) |
| `i` | `^0.3.7` | ⚠️ ভুল করে install হয়েছে (নিচে দেখো) | — | [npm](https://www.npmjs.com/package/i) |

> **`@types/*` কেন লাগে?** কিছু package JavaScript-এ লেখা, তাদের সাথে type আসে না। তখন DefinitelyTyped থেকে `@types/<নাম>` install করতে হয়। যেসব package নিজেই type নিয়ে আসে (`zod`, `stripe`, `better-auth`, `uuid`, `nodemailer` 10, Prisma), তাদের জন্য `@types` লাগে না।

---

## ধাপ ৪: Scripts

| Command | আসলে যা চলে | কাজ |
| --- | --- | --- |
| `pnpm dev` | `tsx watch src/server.ts` | Development server, file বদলালে auto restart |
| `pnpm typecheck` | `tsc --noEmit` | শুধু type check, কোনো file বানায় না |
| `pnpm lint` | `oxlint` | Code-এ lint error খোঁজা |
| `pnpm lint:fix` | `oxlint --fix` | যেগুলো auto ঠিক করা যায়, ঠিক করে দেয় |
| `pnpm check` | `pnpm typecheck && pnpm lint` | Type + lint দুটোই। প্রথমটা fail করলে দ্বিতীয়টা চলে না |
| `pnpm build` | `pnpm check && tsdown` | আগে check, পাস করলে `dist/`-এ build |
| `pnpm start` | `node dist/server.js` | Build করা code চালানো (production) |
| `pnpm migrate` | `prisma migrate dev` | Schema বদলালে migration বানিয়ে DB-তে apply |
| `pnpm generate` | `prisma generate` | `src/generated/prisma`-এ Prisma client বানানো |
| `pnpm studio` | `prisma studio` | Browser-এ DB দেখা/edit করা |
| `pnpm stripe:webhook` | `stripe listen --all-snapshot --forward-to localhost:5000/webhook` | Stripe-এর webhook event local server-এ পাঠানো |

```json
"scripts": {
	"lint": "oxlint",
	"lint:fix": "oxlint --fix",
	"dev": "tsx watch src/server.ts",
	"check": "pnpm typecheck && pnpm lint",
	"build": "pnpm check && tsdown",
	"start": "node dist/server.js",
	"typecheck": "tsc --noEmit",
	"migrate": "prisma migrate dev",
	"generate": "prisma generate",
	"studio": "prisma studio",
	"stripe:webhook": "stripe listen --all-snapshot --forward-to localhost:5000/webhook"
}
```

**কয়েকটা জিনিস খেয়াল করো:**
- **`tsx` type check করে না।** তাই `pnpm dev` চলতে থাকলেও type error থাকতে পারে। সেটা ধরা পড়বে `pnpm typecheck` বা `pnpm build`-এ।
- **`build`-এর আগে `check`**, তাই type বা lint error থাকলে build হবেই না।
- **`stripe:webhook`-এর জন্য Stripe CLI লাগে।** এটা npm package না, আলাদা করে install করতে হয় ([Stripe CLI docs](https://docs.stripe.com/stripe-cli)):
  ```shell
  brew install stripe/stripe-cli/stripe
  stripe login
  ```
- Port `5000` তোমার `.env`-এর `PORT`-এর সাথে মিলতে হবে।

---

## ধাপ ৫: package.json-এর অন্য setting

| Field | Value | মানে |
| --- | --- | --- |
| `"type"` | `"module"` | ESM: `import`/`export` চলে, top-level `await` চলে (`server.ts`) |
| `"main"` | `"dist/server.js"` | Build-এর পরে মূল file |
| `devEngines.packageManager` | `pnpm ^11.20.0`, `onFail: "download"` | এই project শুধু pnpm দিয়ে চালাতে হবে। `npm` দিয়ে চালালে `EBADDEVENGINES` error দেয়। দরকার হলে ঠিক version-এর pnpm নিজে download করে নেয় |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — `package.json` বদলানো হয়নি।

### 🟡 ভালো practice

1. **`i` package মুছে দাও।** এটা একটা পুরনো inflection library, project-এ কোথাও import হয় না। সম্ভবত `pnpm i -D ...` লিখতে গিয়ে `pnpm add -D i ...` হয়ে গিয়েছিল:
   ```shell
   pnpm remove i
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

2. **`description`, `author` খালি।** GitHub-এ রাখলে একটু ভরে দিও, যেমন `"description": "PH Doc healthcare backend (Express + Prisma + PostgreSQL)"`।

3. **Package update দেখতে:**
   ```shell
   pnpm outdated          # কোনগুলোর নতুন version আছে
   pnpm update            # ^ range-এর ভেতরে update
   pnpm update --latest   # major version-ও update (breaking change থাকতে পারে, সাবধানে)
   ```

4. **`stripe:webhook`-এ port hardcode করা।** `.env`-এ `PORT` বদলালে এখানেও হাতে বদলাতে হবে, খেয়াল রেখো।
