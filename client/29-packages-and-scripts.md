# 29 - Packages ও Scripts (phdoc-client)

`phdoc-client`-এর সব package, script আর ESLint config এক জায়গায়: প্রতিটা package কী কাজ করে, কোথায় ব্যবহার হয়, কীভাবে install করতে হয় আর doc কোথায়।
(server-এর জন্য একই রকম note: [server 40](../server/40-packages-and-scripts.md))

| Type | কাজ | কখন লাগে |
| --- | --- | --- |
| `dependencies` | App চালাতে (build আর run) লাগে | Production-এও install হয় |
| `devDependencies` | শুধু develop/build-এর সময় (type, linter, CSS tool) | Production-এ চালানোর সময় লাগে না |

```shell
pnpm add <package>       # dependency
pnpm add -D <package>    # devDependency
pnpm remove <package>    # বাদ দেওয়া
pnpm install             # package.json দেখে সব install
pnpm dlx shadcn@latest add <component>   # shadcn component (package না, file কপি হয়)
```

> Next.js-এ `next build` সব কিছু bundle করে, তাই কোনটা dependency আর কোনটা dev সেটা server-এর মতো কড়া না। তবু নিয়ম মেনে রাখলে পড়তে সহজ।

---

## ধাপ ১: সব একসাথে install (নতুন project-এ)

```shell
# project তৈরি (Next.js + TypeScript + Tailwind + ESLint + src/ + App Router)
pnpm create next-app@latest phdoc-client

# dependencies
pnpm add @tanstack/react-query @tanstack/react-query-devtools \
  @tanstack/react-form @tanstack/react-form-nextjs @tanstack/react-table \
  axios zod jsonwebtoken server-only \
  date-fns recharts sonner next-themes

# devDependencies
pnpm add -D @tanstack/eslint-plugin-query @types/jsonwebtoken \
  @tanstack/react-devtools @tanstack/react-form-devtools \
  @tailwindcss/turbopack

# shadcn (radix-ui, class-variance-authority, lucide-react, tw-animate-css, shadcn নিজে থেকেই আসে)
pnpm dlx shadcn@latest init
```

- `next`, `react`, `react-dom`, `typescript`, `tailwindcss`, `eslint`, `eslint-config-next`, `@types/*` → `create-next-app` নিজেই দেয়।
- `cn` → project-এর `src/lib/utils.ts`-এ ব্যবহার (note 01)।

---

## ধাপ ২: Dependencies

### Framework

| Package | Version | কী কাজ করে | কোথায় ব্যবহার | Doc |
| --- | --- | --- | --- | --- |
| `next` | `16.4.3` | React framework: App Router, Server Component, Server Action, proxy | পুরো app | [nextjs.org/docs](https://nextjs.org/docs) |
| `react` | `19.3.0` | UI library | সব component | [react.dev](https://react.dev/) |
| `react-dom` | `19.3.0` | React-কে browser-এর DOM-এ render করে | Next.js-এর ভেতরে | [react.dev](https://react.dev/reference/react-dom) |

> `next`, `react`, `react-dom`-এ `^` নেই (হুবহু version) — তিনটা একসাথে মিলিয়ে upgrade করতে হয়, আর `eslint-config-next`-ও একই version।

### Data fetching, form, table (TanStack)

| Package | Version | কী কাজ করে | কোথায় ব্যবহার | Doc |
| --- | --- | --- | --- | --- |
| `@tanstack/react-query` | `^5.105.1` | Server data fetch, cache, invalidate, SSR hydration | `queries/*`, `useQuery`, `useActionMutation` | [tanstack.com/query](https://tanstack.com/query/latest) |
| `@tanstack/react-query-devtools` | `^5.105.3` | Browser-এ query cache দেখার panel (শুধু dev-এ দেখায়) | `providers/QueryProvider.tsx` | [Devtools](https://tanstack.com/query/latest/docs/framework/react/devtools) |
| `@tanstack/react-form` | `^1.33.5` | Type-safe form state + Zod validation | login, create/edit doctor form | [tanstack.com/form](https://tanstack.com/form/latest) |
| `@tanstack/react-form-nextjs` | `^1.33.5` | TanStack Form-কে Next.js Server Action-এর সাথে জোড়া (server validation) | এখনো import হয়নি | [Next.js guide](https://tanstack.com/form/latest/docs/framework/react/guides/ssr) |
| `@tanstack/react-table` | `^9.3.1` | Headless table: sort, pagination, column | `DataTable`, admin doctor table (note 16–17) | [tanstack.com/table](https://tanstack.com/table/latest) |

### API, validation, auth

| Package | Version | কী কাজ করে | কোথায় ব্যবহার | Doc |
| --- | --- | --- | --- | --- |
| `axios` | `^1.21.0` | Backend-এ HTTP request (`httpClient`) | `lib/axios/httpClient.ts` (note 04) | [axios-http.com](https://axios-http.com/docs/intro) |
| `zod` | `^4.6.7` | Form আর Server Action-এর input validation | `src/zod/*`, action-এ `safeParse` | [zod.dev](https://zod.dev/) |
| `jsonwebtoken` | `^9.0.3` | Access token verify (proxy) আর মেয়াদ পড়া — **শুধু server-এ** | `lib/jwtUtils.ts` (note 08) | [GitHub](https://github.com/auth0/node-jsonwebtoken) |
| `server-only` | `^0.0.1` | এই file client component-এ import হলে build error → secret browser-এ যায় না | `cookieUtils.ts`, `tokenUtils.ts` ইত্যাদি (note 07) | [Next.js docs](https://nextjs.org/docs/app/getting-started/server-and-client-components#preventing-environment-poisoning) |

### UI (shadcn ও তার সঙ্গী)

| Package | Version | কী কাজ করে | কোথায় ব্যবহার | Doc |
| --- | --- | --- | --- | --- |
| `shadcn` | `^4.23.0` | shadcn CLI + `shadcn/tailwind.css` (theme-এর base style) | `globals.css`-এ `@import "shadcn/tailwind.css"` | [ui.shadcn.com](https://ui.shadcn.com/docs) |
| `radix-ui` | `^1.8.1` | Accessible headless component (Dialog, Dropdown, Select, Sheet…) — এক package-এ সব | `components/ui/*` | [radix-ui.com](https://www.radix-ui.com/primitives) |
| `class-variance-authority` | `^0.7.1` | Component-এর variant (`variant="outline"`, `size="sm"`) → class | `button.tsx`, `badge.tsx` ইত্যাদি | [cva.style](https://cva.style/docs) |
| `cn` | `^0.5.1` | একাধিক class জোড়া + Tailwind-এর conflict মেটানো (`clsx` + `tailwind-merge`-এর কাজ) | `lib/utils.ts` → সব জায়গায় `cn(...)` | [npm](https://www.npmjs.com/package/cn) |
| `lucide-react` | `^1.66.0` | Icon (`<Star />`, `<Menu />`…) | প্রায় সব component | [lucide.dev](https://lucide.dev/guide/packages/lucide-react) |
| `tw-animate-css` | `^1.4.0` | Tailwind v4-এ animation class (dialog/dropdown খোলা-বন্ধ) | `globals.css`-এ import | [GitHub](https://github.com/Wombosvideo/tw-animate-css) |
| `sonner` | `^2.0.8` | Toast notification | `useActionMutation`, root layout-এ `<Toaster />` | [sonner.emilkowal.ski](https://sonner.emilkowal.ski/) |
| `next-themes` | `^0.4.6` | Light/dark theme | এখন শুধু `components/ui/sonner.tsx`-এ `useTheme` | [GitHub](https://github.com/pacocoursey/next-themes) |

### অন্যান্য

| Package | Version | কী কাজ করে | কোথায় ব্যবহার | Doc |
| --- | --- | --- | --- | --- |
| `recharts` | `^3.10.1` | Chart (pie, bar) | dashboard chart (note 13–14) | [recharts.org](https://recharts.org/) |
| `date-fns` | `^4.4.0` | তারিখের হিসাব/format | `NotificationDropdown` (`formatDistanceToNow`) | [date-fns.org](https://date-fns.org/) |

> সময় দেখানোর জন্য `date-fns`-এর `format()` ব্যবহার হয় না — ওটা browser-এর timezone নেয়। ঢাকার সময় ঠিক রাখতে `lib/scheduleTime.ts`-এ `Intl.DateTimeFormat` + `timeZone: "Asia/Dhaka"` (note 21)।

---

## ধাপ ৩: Dev Dependencies

| Package | Version | কী কাজ করে | Doc |
| --- | --- | --- | --- |
| `typescript` | `^5` | Type check (`tsc --noEmit`) | [typescriptlang.org](https://www.typescriptlang.org/docs/) |
| `@types/node` | `^20` | Node-এর type (`process.env` ইত্যাদি) | [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) |
| `@types/react`, `@types/react-dom` | `^19` | React-এর type | ঐ |
| `@types/jsonwebtoken` | `^9.0.10` | `jsonwebtoken`-এর type (package নিজে type দেয় না) | ঐ |
| `tailwindcss` | `^4` | Tailwind CSS v4 (config file ছাড়া, `globals.css`-এ `@import "tailwindcss"`) | [tailwindcss.com](https://tailwindcss.com/docs) |
| `@tailwindcss/turbopack` | `^4` | Turbopack-এ Tailwind চালানোর loader — `next.config.ts`-এর `turbopack.rules`-এ `*.css` | [tailwindcss.com](https://tailwindcss.com/docs/installation) |
| `eslint` | `^9` | Code-এর ভুল/খারাপ pattern ধরে (flat config) | [eslint.org](https://eslint.org/docs/latest/) |
| `eslint-config-next` | `16.4.3` | Next.js-এর ESLint rule (core-web-vitals + typescript) | [Next.js ESLint](https://nextjs.org/docs/app/api-reference/config/eslint) |
| `@tanstack/eslint-plugin-query` | `^5.105.1` | TanStack Query-র ভুল ধরে (যেমন queryKey-তে dependency বাদ পড়া) | [ESLint plugin](https://tanstack.com/query/latest/docs/eslint/eslint-plugin-query) |
| `@tanstack/react-devtools` | `^0.10.14` | TanStack-এর সব devtools এক panel-এ | এখনো import হয়নি | [TanStack Devtools](https://tanstack.com/devtools/latest) |
| `@tanstack/react-form-devtools` | `^0.2.34` | Form state দেখার devtools panel | এখনো import হয়নি | ঐ |

> postcss config নেই — Turbopack-এ Tailwind চলে `@tailwindcss/turbopack` loader দিয়ে (`next.config.ts`)।

---

## ধাপ ৪: Scripts

```json
"scripts": {
	"dev": "next dev",
	"build": "next build",
	"start": "next start",
	"lint": "eslint",
	"check": "tsc --noEmit && eslint"
}
```

| Script | Command | কী করে | কখন |
| --- | --- | --- | --- |
| `pnpm dev` | `next dev` | Development server (`localhost:3000`), Turbopack, file save করলে নিজে reload | কাজ করার সময় |
| `pnpm build` | `next build` | Production build (`.next/`): type check + optimize + prerender | deploy-এর আগে (note 28) |
| `pnpm start` | `next start` | Build করা app চালায় (আগে `build` লাগবে) | production-এ (pm2 দিয়ে, note 28) |
| `pnpm lint` | `eslint` | পুরো project lint (`eslint.config.mjs` দেখে) | commit-এর আগে |
| `pnpm check` | `tsc --noEmit && eslint` | আগে type check, ঠিক থাকলে তবেই lint | commit/push-এর আগে, এক command-এ সব |

- `tsc --noEmit` → শুধু type error খোঁজে, কোনো JS file বানায় না (Next.js নিজে compile করে)।
- `&&` → প্রথমটা fail করলে দ্বিতীয়টা চলে না।
- `pnpm dev` type error দেখায় না (শুধু browser-এ চালায়) — তাই `check` বা `build` দিয়ে ধরা পড়ে। server-এর `pnpm check`-এর মতোই (server note 40)।

---

## ধাপ ৫: `eslint.config.mjs`

```javascript
import pluginQuery from "@tanstack/eslint-plugin-query";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";
import { defineConfig, globalIgnores } from "eslint/config";

const eslintConfig = defineConfig([
	...nextVitals,
	...nextTs,
	...pluginQuery.configs["flat/recommended"],
	{
		rules: {
			// "_" দিয়ে শুরু হওয়া নাম = ইচ্ছা করে ব্যবহার করা হয়নি
			"@typescript-eslint/no-unused-vars": [
				"warn",
				{
					varsIgnorePattern: "^_",
					argsIgnorePattern: "^_",
					destructuredArrayIgnorePattern: "^_",
					ignoreRestSiblings: true,
				},
			],
		},
	},
	// Override default ignores of eslint-config-next.
	globalIgnores([
		// Default ignores of eslint-config-next:
		".next/**",
		"out/**",
		"build/**",
		"next-env.d.ts",
	]),
]);

export default eslintConfig;
```

### ভেঙে বোঝা

**১. Flat config (ESLint 9)** — একটা array, উপর থেকে নিচে config জোড়া হয়; পরেরটা আগেরটার rule override করে। `defineConfig` → editor-এ autocomplete আর type check।

| অংশ | কী দেয় |
|---|---|
| `nextVitals` (`core-web-vitals`) | Next.js + React + React Hooks + accessibility rule; performance-এর ভুল (যেমন `<img>`-এর বদলে `next/image`) error হিসেবে |
| `nextTs` (`typescript`) | `typescript-eslint`-এর recommended rule |
| `pluginQuery.configs["flat/recommended"]` | TanStack Query: `exhaustive-deps` (queryFn-এ যা ব্যবহার, queryKey-তে থাকতে হবে), `no-rest-destructuring` ইত্যাদি |
| নিজের `rules` | unused variable-এর নিয়ম (নিচে) |
| `globalIgnores` | কোন folder lint হবে না |

**২. `no-unused-vars` — `_` দিয়ে শুরু হলে ছাড়**

| Option | উদাহরণ (warning আসবে না) |
|---|---|
| `varsIgnorePattern: "^_"` | `const _unused = 1;` |
| `argsIgnorePattern: "^_"` | `(_event, value) => ...` |
| `destructuredArrayIgnorePattern: "^_"` | `const [_first, second] = arr;` |
| `ignoreRestSiblings: true` | `const { password, ...rest } = user;` — `password` বাদ দেওয়ার জন্য নেওয়া, ব্যবহার না হলেও ঠিক |

- `"warn"` → build আটকায় না, শুধু হলুদ দাগ।
- server-এর oxlint-এও একই `_` নিয়ম (server note 40) — দুই project-এ এক অভ্যাস।

**৩. `globalIgnores`** — `.next/` (build output), `out/` (static export), `build/`, `next-env.d.ts` (Next.js নিজে বানায়)। নিজে লেখা হলে `eslint-config-next`-এর default ignore বাদ পড়ে, তাই আবার লেখা আছে।

> client-এ **ESLint**, server-এ **oxlint** — তাই client-এর code-এ `// eslint-disable-next-line ...` comment কাজ করে, server-এ করে না।

---

## ধাপ ৬: `package.json`-এর অন্য setting

| Field | মান | মানে |
| --- | --- | --- |
| `name` | `phdoc-client` | project-এর নাম |
| `version` | `0.1.0` | নিজের version (publish করা হয় না, তাই গুরুত্বপূর্ণ না) |
| `private` | `true` | ভুল করে npm-এ publish হবে না |
| `packageManager` | `pnpm@12.15.0` | এই project-এ pnpm-এর এই version; Corepack থাকলে নিজে থেকেই এটা ব্যবহার করে |

- `"type": "module"` নেই, তবু `eslint.config.mjs`-এ `import` চলে — `.mjs` extension মানেই ESM।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — package.json বদলানো হয়নি।

### 🟡 ভালো practice

1. **ব্যবহার না হওয়া package** — project-এর `src/`-এ এই তিনটা কোথাও import নেই (grep করে দেখা):
   - `@tanstack/react-form-nextjs`
   - `@tanstack/react-devtools`
   - `@tanstack/react-form-devtools`

   পরে ব্যবহারের plan না থাকলে বাদ দাও — install আর deploy (VPS-এ `pnpm install`) একটু দ্রুত হবে:
   ```shell
   pnpm remove @tanstack/react-form-nextjs @tanstack/react-devtools @tanstack/react-form-devtools
   ```

2. **`date-fns` শুধু এক জায়গায়** — `formatDistanceToNow` ("5 minutes ago")। এটা `Intl.RelativeTimeFormat` দিয়েও করা যায়, তখন package লাগবে না। তবে date-fns tree-shake হয় (শুধু ব্যবহৃত function bundle-এ যায়), তাই রাখলেও ক্ষতি কম।

3. **`@types/node` `^20`** — VPS-এ Node 20.9+ (note 28); local-এ নতুন Node ব্যবহার করলে type মিলিয়ে `@types/node`-এর version-ও সেই major-এ রাখো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **`next-themes` এখন শুধু toast-এর জন্য** — `ThemeProvider` বসিয়ে header-এ dark mode toggle দিলে পুরো app-এ কাজ করবে (note 26-এর 🔜 list-এ আছে)।

5. **Upgrade-এর সময়** — `next`, `react`, `react-dom`, `eslint-config-next` একসাথে:
   ```shell
   pnpm add next@latest react@latest react-dom@latest && pnpm add -D eslint-config-next@latest
   pnpm check && pnpm build
   ```
