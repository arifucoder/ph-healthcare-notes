# 05 - TanStack Query Integration (Provider Setup)

> Module: 44-09

```
install → getQueryClient.ts (QueryClient বানানো) → QueryProvider.tsx ("use client") → layout.tsx-এ wrap → পরে prefetch + hydrate
```

## ধাপ ১: Install

👉 https://tanstack.com/query/latest/docs/framework/react/installation

```shell
pnpm add @tanstack/react-query
pnpm add -D @tanstack/eslint-plugin-query
```

## ধাপ ২: Next.js-এ কেন Advanced SSR guide

👉 https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr

Next.js-এ server side আছে, আর TanStack Query দিয়ে server side-এও data fetch করা যায়। তাই আমরা এই guide follow করব — TanStack Query এমনভাবে ব্যবহার করব যাতে **data fetching server side-এ হয়** (দ্রুত প্রথম load + SEO), আর client side-এ cache, refetch-এর সুবিধা থাকে। এটা ব্যবহার করতে আমাদের একটা **provider** add করতে হবে।

## ধাপ ৩: ৩টা core concept

TanStack Query মূলত ৩টা জিনিসের উপর কাজ করে:

| Concept | Hook | কখন |
|---|---|---|
| **Query** | `useQuery` | data **GET** করা |
| **Mutation** | `useMutation` | data বদলানো — **POST, PATCH, PUT, DELETE** |
| **Query Invalidation** | `queryClient.invalidateQueries` | mutation-এর পরে পুরোনো cache বাতিল করে নতুন করে আনা |

### Query + `queryKey`

Data GET করার সময় আমরা চাই data **cache** হয়ে থাকুক। তাই একটা **`queryKey`** দিতে হয় — এই key দিয়েই TanStack Query ওই data-কে track করে।

```tsx
const { data, isLoading } = useQuery({
	queryKey: ["specialties"], // এই নামে cache হবে
	queryFn: () => httpClient.get<Specialty[]>("/specialties"),
});
```

### Mutation + invalidate

`useMutation`-এ `onSuccess`-এর ভেতরে আবার ওই `queryKey` দিতে হয়। ধরো key `["specialties"]` — invalidate করলে **যত জায়গায় এই key ব্যবহার হয়েছে, তত জায়গায় query নতুন করে refresh হবে।**

```tsx
const queryClient = useQueryClient();

const { mutate, isPending } = useMutation({
	mutationFn: (payload: { title: string }) => httpClient.post("/specialties", payload),
	onSuccess: () => {
		queryClient.invalidateQueries({ queryKey: ["specialties"] }); // list নিজে নিজে update
	},
});
```

```
mutate() → POST /specialties ✅ → onSuccess → invalidate ["specialties"] → সব useQuery(["specialties"]) আবার fetch
```

> ✏️ উপরের ২টা ছোট example Claude যোগ করেছে, concept বোঝানোর জন্য — এটা নোটে ছিল না।

## ধাপ ৪: Server vs Client component — কোনটা কোথায় (44-09)

- Client component **তখনই** বানাব যখন কোনো interaction থাকবে — click করলে কিছু হবে, hover করলে কিছু হবে, toast দেখাবে ইত্যাদি।
- বাকি সব server component।

উদাহরণ:

```
LoginPage (server) → LoginCard (server) → LoginForm ("use client") ← শুধু এটাই client
```

### নিয়ম: server → client → server

| | হয়? |
|---|---|
| Server component-এর ভেতরে client component import | ✅ |
| Client component-এর **file-এ** server component **import** | ❌ (import করলে ওটাও client হয়ে যায়) |
| Server component-কে **`children`** (prop) হিসেবে client component-এর ভেতরে পাঠানো | ✅ |

```tsx
// ✅ server component children হিসেবে client-এর ভেতরে
<QueryProvider>         {/* client */}
	<ServerPage />        {/* server — children হিসেবে গেছে, তাই server-ই থাকে */}
</QueryProvider>
```

> এই কারণেই `QueryProvider` client হলেও root layout-এ সব page wrap করা যায় — page গুলো `children` হিসেবে যায়, তাই server component-ই থাকে।

## ধাপ ৫: TanStack Query project-এ add করা

### ১. `src/lib/getQueryClient.ts` (নতুন file)

```ts
import { defaultShouldDehydrateQuery, environmentManager, QueryClient } from "@tanstack/react-query";

function makeQueryClient() {
	return new QueryClient({
		defaultOptions: {
			queries: {
				// SSR-এ data আসার পরপরই client যেন আবার fetch না করে
				staleTime: 60 * 1000,
			},
			dehydrate: {
				// server-এ যে query এখনো চলছে (pending), সেটাও client-এ stream হবে
				shouldDehydrateQuery: (query) =>
					defaultShouldDehydrateQuery(query) || query.state.status === "pending",
				// error লুকানোর কাজ Next.js নিজেই করে, তাই এখানে বন্ধ
				shouldRedactErrors: () => false,
			},
		},
	});
}

let browserQueryClient: QueryClient | undefined;

export function getQueryClient() {
	if (environmentManager.isServer()) {
		// Server: প্রতিটা request-এ নতুন client, যাতে এক user-এর data অন্য user-এর কাছে না যায়
		return makeQueryClient();
	}

	// Browser: একটাই client পুরো app-এ ব্যবহার হবে
	if (!browserQueryClient) browserQueryClient = makeQueryClient();
	return browserQueryClient;
}
```

**সহজ কথায়:**

| অংশ | কাজ |
|---|---|
| `QueryClient` | TanStack Query-এর "মগজ" — সব cache এখানে থাকে |
| `staleTime: 60 * 1000` | data ১ মিনিট পর্যন্ত "fresh" — এর মধ্যে আবার fetch করবে না। SSR-এ এটা জরুরি, নাহলে server-এ আনা data browser-এ এসেই আবার fetch হতো |
| `dehydrate` | server-এর cache-কে client-এ পাঠানোর setting (dehydrate = cache-কে JSON বানানো) |
| Server-এ প্রতিবার নতুন | server-এ একই client রাখলে user A-এর data user B দেখে ফেলতে পারে ❌ |
| Browser-এ একটাই | browser-এ একজনই user; প্রতিবার নতুন বানালে cache হারিয়ে যেত |

> `environmentManager.isServer()` — TanStack Query v5-এর নতুন API (আমাদের version 5.105-এ আছে, check করা হয়েছে)। পুরোনো docs/tutorial-এ `isServer` দেখবে, কাজ একই।

### ২. `src/providers/QueryProvider.tsx`

```tsx
"use client";

import type { ReactNode } from "react";
import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { getQueryClient } from "@/lib/getQueryClient";

export default function QueryProvider({ children }: { children: ReactNode }) {
	const queryClient = getQueryClient();

	return (
		<QueryClientProvider client={queryClient}>
			{children}
			<ReactQueryDevtools initialIsOpen={false} />
		</QueryClientProvider>
	);
}
```

- `"use client"` লাগবে, কারণ `QueryClientProvider` ভেতরে React Context ব্যবহার করে (context শুধু client-এ চলে)।
- `QueryClientProvider` = নিচের সব component-কে `queryClient` দিয়ে দেয়, তাই যেকোনো জায়গায় `useQuery` চলে।

Devtools ব্যবহার করতে চাইলে package টা install করতে হবে:

```shell
pnpm add @tanstack/react-query-devtools
```

Production build-এ devtools **automatically বাদ পড়ে যায়**, তাই রেখে দিলে কোনো সমস্যা নেই। না চাইলে `ReactQueryDevtools`-এর import আর লাইনটা মুছে দিও।

> Devtools = screen-এর কোণায় ছোট একটা button, যেখানে সব query, তাদের cache, status (fresh/stale/fetching) দেখা যায় — debugging-এ খুব কাজের।

### ৩. `src/app/layout.tsx`-এ লাগাও

```tsx
import QueryProvider from "@/providers/QueryProvider";

// ... বাকি import, font, metadata আগের মতো

export default function RootLayout({ children }: LayoutProps<"/">) {
	return (
		<html lang="en" className={`${geistSans.variable} ${geistMono.variable} h-full antialiased`}>
			<body className="min-h-full flex flex-col">
				<QueryProvider>{children}</QueryProvider>
			</body>
		</html>
	);
}
```

## ধাপ ৬: পরের ধাপ — Prefetching ও de/hydrating

👉 https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr#prefetching-and-de-hydrating-data

আমরা যেহেতু server side render করব, তাই **Prefetching and de/hydrating data** technique follow করব। মূল idea:

```
Server page:  queryClient.prefetchQuery(...)  → data server-এ আনা
              dehydrate(queryClient)          → cache-কে JSON বানানো
              <HydrationBoundary state={...}> → HTML-এর সাথে client-এ পাঠানো
Client:       useQuery(একই queryKey)          → fetch না করেই cache থেকে সাথে সাথে data ✅
```

## 🔜 পরে যা যোগ হবে

- [ ] Server page-এ `prefetchQuery` + `HydrationBoundary`
- [ ] Client component-এ `useQuery` দিয়ে সেই data পড়া
- [ ] `useMutation` + `invalidateQueries` (create/update/delete)
- [ ] `queryKey` গুলো এক জায়গায় রাখা

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **ESLint plugin এখনো config-এ যোগ হয়নি** — `@tanstack/eslint-plugin-query` install আছে, কিন্তু `eslint.config.mjs`-এ নেই, তাই কোনো কাজ করছে না (নোট 01-এর recommendation)। এটা `queryKey`-তে dependency বাদ পড়ার মতো ভুল ধরে:

   ```js
   import pluginQuery from "@tanstack/eslint-plugin-query";

   const eslintConfig = defineConfig([
   	...nextVitals,
   	...nextTs,
   	...pluginQuery.configs["flat/recommended"],
   	// ...
   ]);
   ```

2. **`queryKey`-তে variable থাকলে সেটাও key-তে দাও** — নাহলে id বদলালেও পুরোনো data দেখাবে:

   ```tsx
   useQuery({
   	queryKey: ["doctor", id], // ✅ id key-তে আছে
   	queryFn: () => httpClient.get<Doctor>(`/doctors/${id}`),
   });
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Prefetch করা server component-ও `<Suspense>`-এর ভেতরে** — `cacheComponents: true` চালু, তাই `await queryClient.prefetchQuery(...)` থাকা component সরাসরি page-এ বসালে নোট 03-এর `uncached data` error আসবে।

4. **`invalidateQueries` prefix ধরে কাজ করে** — `["doctors"]` invalidate করলে `["doctors", 1]`, `["doctors", { page: 2 }]` সবই refresh হয়। তাই key-এর প্রথম অংশে resource-এর নাম রাখো।
