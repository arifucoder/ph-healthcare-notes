# 01 - Next.js Setup, shadcn ও দরকারি Packages

```
Next.js install → shadcn init → project structure বোঝা → দরকারি packages add
```

## ধাপ ১: Next.js install

প্রথমে Next.js-এর documentation website-এ যাব:
👉 https://nextjs.org/docs/app/getting-started/installation

এখানে terminal command পাওয়া যাবে:

```shell
pnpm create next-app@latest my-app --yes
# আগে থেকেই যদি folder বানানো থাকে, তাহলে my-app এর জায়গায় dot (.) দিব
cd my-app
pnpm dev
```

> `--yes` দিলে সব প্রশ্নের default উত্তর নিয়ে নেয় (TypeScript, Tailwind, ESLint, App Router, `src/` folder, `@/*` alias)।

## ধাপ ২: shadcn/ui add করা

👉 https://ui.shadcn.com/docs/installation/next

এখানে বেশ কিছু option আছে। আমরা **Existing Project**-এ চলে যাব এবং সেখান থেকে সরাসরি command চালাতে পারি:

```shell
pnpm dlx shadcn@latest init
```

> 💡 Teacher বলেছেন Next.js app-এর ক্ষেত্রে **bun** ব্যবহার করলে খুব দ্রুত কাজ করে, কিন্তু আপাতত আমি এখানে **pnpm** ব্যবহার করব।

init করার পর `components.json` তৈরি হয় — এখানে shadcn-এর setting থাকে (style, base color, `@/components/ui` alias ইত্যাদি)। পরে component add করতে:

```shell
pnpm dlx shadcn@latest add button
```

## ধাপ ৩: Project structure বোঝা

| File | কাজ |
|---|---|
| `src/app/layout.tsx` | পুরো app-এর layout (root layout) — `<html>`, `<body>`, font, metadata এখানে |
| `src/app/globals.css` | globally যেসব CSS লাগবে — shadcn-এর CSS (theme variables) এখানেই add হয়েছে |
| `src/app/page.tsx` | `/` route-এর content — এখান থেকেই page-এর content show হচ্ছে |
| `src/components/ui/` | shadcn component গুলো এখানে আসে (যেমন `button.tsx`) |
| `src/lib/utils.ts` | `cn()` helper (className merge করার জন্য) |

## ধাপ ৪: দরকারি packages add করা

| কাজ | Package | Docs |
|---|---|---|
| Data fetching | TanStack Query | https://tanstack.com/query/latest/docs/framework/react/installation |
| HTTP request | Axios | https://axios.rest/pages/getting-started/first-steps.html |
| Form handling | TanStack Form | https://tanstack.com/form/latest/docs/installation |
| Form validation | Zod | https://zod.dev/ |
| Token (JWT) | jsonwebtoken | https://www.npmjs.com/package/jsonwebtoken |

### Command

```shell
# TanStack Query
pnpm add @tanstack/react-query
pnpm add -D @tanstack/eslint-plugin-query

# TanStack Form
pnpm add @tanstack/react-form
pnpm add @tanstack/react-form-nextjs
# TanStack Form devtools (teacher add করেছেন) — শুধু development-এ লাগে, তাই -D
pnpm add -D @tanstack/react-devtools
pnpm add -D @tanstack/react-form-devtools

# Axios
pnpm install axios

# Zod
pnpm add zod

# JWT
pnpm add jsonwebtoken
pnpm add -D @types/jsonwebtoken
```

> `-D` = devDependency (শুধু development-এ লাগে, যেমন type বা lint plugin)।

## 🔜 পরে যা যোগ হবে

- [ ] `QueryClientProvider` দিয়ে app wrap করা (providers file)
- [ ] Axios instance (`baseURL` = server-এর `/api/v1`)
- [ ] TanStack Form + Zod দিয়ে login/register form
- [ ] JWT token দিয়ে auth handle করা

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`@tanstack/eslint-plugin-query` শুধু install করলে কাজ করে না** — `eslint.config.mjs`-এ যোগ করতে হবে (client-এ ESLint ব্যবহার হচ্ছে):

   ```js
   import pluginQuery from "@tanstack/eslint-plugin-query";

   const eslintConfig = defineConfig([
     ...nextVitals,
     ...nextTs,
     ...pluginQuery.configs["flat/recommended"],
     // ...
   ]);
   ```

2. ✅ (Done) **Devtools গুলো `-D` তে রাখা হয়েছে** — `@tanstack/react-devtools` ও `@tanstack/react-form-devtools` শুধু development-এ লাগে। উপরের command-এ `-D` যোগ করা হয়েছে।

   > আগে dependencies-এ install করা থাকলে এভাবে devDependencies-এ সরানো যায়:
   > ```shell
   > pnpm remove @tanstack/react-devtools @tanstack/react-form-devtools
   > pnpm add -D @tanstack/react-devtools @tanstack/react-form-devtools
   > ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`jsonwebtoken` শুধু server-side চলে** (Node.js API ব্যবহার করে) — Server Component, Route Handler, Server Action-এ চলবে, কিন্তু `"use client"` component-এ (browser-এ) চলবে না। Browser-এ শুধু token decode করতে চাইলে `jwt-decode` বা `jose` ব্যবহার করা যায়।

4. **`.env` এ server URL রাখো** — browser-এ লাগলে নামের শুরুতে `NEXT_PUBLIC_` দিতে হয়:

   ```env
   NEXT_PUBLIC_API_URL=http://localhost:5000/api/v1
   ```
