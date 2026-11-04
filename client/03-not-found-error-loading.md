# 03 - not-found, error ও loading

```
URL পাওয়া যায়নি → not-found.tsx | হঠাৎ error → error.tsx | data আসতে দেরি → loading.tsx / <Suspense>
```

| File | কখন দেখায় | Client/Server |
|---|---|---|
| `not-found.tsx` | route নেই, বা `notFound()` call করলে | server (default) |
| `error.tsx` | render-এর সময় হঠাৎ error throw হলে | **অবশ্যই client** (`"use client"`) |
| `global-error.tsx` | root `layout.tsx`-এই error হলে | **অবশ্যই client** |
| `loading.tsx` | page তৈরি হতে সময় লাগলে | server (default) |

> 📁 Next.js-এর নিজের docs project-এর ভেতরেই আছে: `node_modules/next/dist/docs/01-app/03-api-reference/03-file-conventions/` (Next 16-এ অনেক কিছু বদলেছে, তাই এখান থেকে মিলিয়ে নেওয়া ভালো)।

---

## ধাপ ১: not-found

👉 https://nextjs.org/docs/app/api-reference/file-conventions/not-found

`not-found.tsx` রাখতে হবে `app`-এর ভেতরে (`src/app/not-found.tsx`)। একটা global root layout (`app/layout.tsx`) থাকলেই এটা কাজ করে, আর পুরো app-এর যেকোনো **unmatched URL**-এ (যেমন `/abcd`) এটাই দেখায়।

`src/app/not-found.tsx`
```tsx
import Link from "next/link";

export default function NotFound() {
	return (
		<main className="flex min-h-screen flex-col items-center justify-center gap-4 text-center">
			<h1 className="text-4xl font-bold">404</h1>
			<p className="text-muted-foreground">The page you are looking for does not exist.</p>
			<Link href="/" className="underline">
				Go back home
			</Link>
		</main>
	);
}
```

- `not-found.tsx` কোনো props নেয় না।
- Page-এর ভেতর থেকেও দেখানো যায়: যেমন doctor না পেলে `notFound()` call করলে এই page দেখাবে।

```tsx
import { notFound } from "next/navigation";
if (!doctor) notFound();
```

### ⚠️ Root layout না থাকলে → `global-not-found.tsx` (experimental)

> ✅ তোমার কথাটা সঠিক — docs-এ মিলিয়ে দেখা হয়েছে।

যদি main root layout (`app/layout.tsx`) **না থাকে**, আর আলাদা আলাদা layout থাকে (যেমন `app/(commonLayout)/layout.tsx` একটা আর `app/(protectedLayout)/layout.tsx` আরেকটা, প্রত্যেকটায় নিজের `<html>`/`<body>`) — তাহলে এদের **multiple root layouts** বলে। তখন 404 বানানোর মতো কোনো single layout থাকে না, তাই `not-found.tsx` দিয়ে global 404 ঠিকমতো কাজ করে না। তখন experimental `global-not-found.tsx` লাগে:

```ts
// next.config.ts
const nextConfig: NextConfig = {
	experimental: {
		globalNotFound: true,
	},
};
```

```tsx
// src/app/global-not-found.tsx
import "./globals.css"; // layout bypass হয়, তাই css/font নিজে import করতে হবে

export default function GlobalNotFound() {
	return (
		<html lang="en">
			<body>
				<h1>404 - Page Not Found</h1>
			</body>
		</html>
	);
}
```

> আমাদের project-এ root `app/layout.tsx` আছে, তাই শুধু `not-found.tsx`-ই যথেষ্ট — `global-not-found` লাগবে না।

---

## ধাপ ২: error

👉 https://nextjs.org/docs/app/api-reference/file-conventions/error

### error.tsx কী করে

এটা একটা **React Error Boundary**। ওই route-এর page, loading, not-found বা nested layout render হওয়ার সময় হঠাৎ error throw হলে (যেমন API down, বা কোনো bug) **পুরো site crash হয় না**। শুধু ওই অংশে fallback UI দেখায়, বাকি layout (sidebar, navbar) ঠিক থাকে।

### ৩টা নিয়ম

1. **`"use client"` লাগবেই** — error boundary শুধু client component হতে পারে।
2. **একই folder-এর `layout.tsx`-কে wrap করে না** — তাই `app/error.tsx` root layout-এর error ধরে না। সেটার জন্য আলাদা `app/global-error.tsx` লাগে।
3. **Expected error-এর জন্য না** — doctor না পেলে `notFound()`, form validation-এর error UI-তেই দেখাবে। `error.tsx` শুধু **অপ্রত্যাশিত** error-এর জন্য।

```
<Layout>                ← error.tsx এটাকে ধরে না
	<ErrorBoundary>       ← error.tsx
		<Suspense>          ← loading.tsx
			<Page />
```

### `reset()` vs `retry()`

| | কী করে | কখন |
|---|---|---|
| `reset()` | শুধু error state মুছে আবার render করে, server থেকে data **আবার fetch করে না** — তাই বেশিরভাগ সময় আবার একই error দেখায় | বিশেষ দরকারে |
| `retry()` ✅ | data **আবার fetch** করে তারপর render করে — backend কিছুক্ষণ down থাকলে "Try again" চাপলে আসলেই ঠিক হয়ে যায় | বেশিরভাগ ক্ষেত্রে (docs-এর suggestion) |

> `retry` Next.js 16.3 থেকে stable (16.2-তে নাম ছিল `unstable_retry`)। আমাদের version 16.4, তাই `retry` সরাসরি চলবে।

`src/app/error.tsx`
```tsx
"use client"; // Error boundary অবশ্যই client component হতে হবে

import { useEffect } from "react";

export default function ErrorPage({ error, retry }: { error: unknown; retry: () => void }) {
	useEffect(() => {
		console.error(error);
	}, [error]);

	const digest = error instanceof Error && "digest" in error ? String(error.digest) : undefined;

	return (
		<main className="flex min-h-[60vh] flex-col items-center justify-center gap-4 text-center">
			<h2 className="text-2xl font-semibold">Something went wrong!</h2>
			{digest && <p className="text-sm text-muted-foreground">Error ID: {digest}</p>}
			<button onClick={() => retry()} className="rounded-md border px-4 py-2">
				Try again
			</button>
		</main>
	);
}
```

> `digest` = server-এ হওয়া error-এর একটা ID। Production-এ আসল error message browser-এ লুকানো থাকে, শুধু এই ID দেখায় — server log-এ এই ID দিয়ে খুঁজে পাওয়া যায়।

### global-error.tsx (optional — root layout-এর error ধরার জন্য)

এটা active হলে root layout-এর **জায়গা নিয়ে নেয়**, তাই নিজের `<html>`, `<body>` আর `globals.css` লাগবে। Client component বলে `metadata` export চলে না — তাই `<title>` সরাসরি লেখা হয়েছে।

`src/app/global-error.tsx`
```tsx
"use client";

import "./globals.css";

export default function GlobalError({ retry }: { error: unknown; retry: () => void }) {
	return (
		<html lang="en">
			<body className="flex min-h-screen flex-col items-center justify-center gap-4">
				<title>Something went wrong</title>
				<h2 className="text-2xl font-semibold">Something went wrong!</h2>
				<button onClick={() => retry()} className="rounded-md border px-4 py-2">
					Try again
				</button>
			</body>
		</html>
	);
}
```

---

## ধাপ ৩: loading

### ১. loading.tsx কী

একটা special file। কোনো route-এর content server থেকে আসতে সময় লাগলে **সাথে সাথে** এই file-এর UI দেখায়। Content তৈরি হয়ে গেলে loading UI সরে যায় আর আসল page নিজে থেকেই বসে যায়।

```tsx
export default function Loading() {
	return <p>Loading...</p>; // বা Skeleton component
}
```

- কোনো props নেয় না।
- Default-ভাবে server component। দরকার হলে (যেমন animation) `"use client"` দেওয়া যায়।
- UI হালকা রাখতে হয়: spinner, skeleton, বা page-এর title-এর মতো ছোট অংশ।

### ২. কীভাবে ব্যবহার করব

1. যে page-এর জন্য loading চাও, তার folder-এ `loading.tsx` রাখো:
   ```
   admin/dashboard/
   ├── layout.tsx
   ├── loading.tsx   ← এটা
   └── page.tsx
   ```
2. ভেতরে যা দেখাতে চাও সেটা লেখো (উপরের code)।
3. এটুকুই। এরপর Next.js নিজেই:
   - page তৈরি হওয়ার সময় Loading দেখাবে,
   - page তৈরি হয়ে গেলে Loading সরিয়ে আসল page বসাবে,
   - পুরো সময় sidebar/layout যেমন আছে তেমন থাকবে, শুধু content অংশে loading দেখাবে।

### ৩. সমস্যা: loading.tsx layout-কে ঢাকে না

`loading.tsx` সবসময় layout-এর **ভেতরে** বসে:

```tsx
<Layout>              ← loading এটাকে ঢাকতে পারে না
	<Suspense fallback={<Loading />}>
		<Page />
	</Suspense>
</Layout>
```

তাই layout নিজে যদি `cookies()` পড়ে বা data fetch করে সময় নেয়, তখন `loading.tsx`-এর কিছু করার থাকে না। Layout শেষ না হওয়া পর্যন্ত screen আটকে থাকে, click করার পরও মনে হয় কিছুই হচ্ছে না।

**সমাধান ১:** ধীর অংশটা নিজে `<Suspense>` দিয়ে wrap করো:

```tsx
// (dashboardLayout)/layout.tsx
import { Suspense } from "react";

export default function DashboardLayout({ children }: LayoutProps<"/">) {
	return (
		<div className="flex">
			<Suspense fallback={<p>Loading sidebar...</p>}>
				<Sidebar /> {/* এর ভেতরে cookies() পড়া হয় */}
			</Suspense>
			<main>{children}</main>
		</div>
	);
}
```

এতে sidebar load হওয়ার সময় শুধু ওই জায়গায় "Loading sidebar..." দেখায়, বাকি page আটকে থাকে না।

**সমাধান ২:** `cookies()` বা fetch-এর কাজটা layout থেকে সরিয়ে `page.tsx`-এ নিয়ে যাও। তখন `loading.tsx` নিজেই সেটা ঢেকে ফেলবে।

> 📌 এক লাইনে: `loading.tsx` শুধু page-এর ধীর কাজ ঢাকে, নিজের folder-এর layout-এর না। Layout-এ ধীর কাজ থাকলে ওই অংশটা নিজে `<Suspense>`-এ wrap করো।

### ৪. Global loading — `src/app/loading.tsx`

| ✅ কখন কাজ করে | ❌ কখন কাজ করে না |
|---|---|
| যেকোনো page তৈরি হতে সময় নিলে (যেমন server component-এ `await fetch(...)`) — login, consultation, admin/dashboard/... সব page-এ | Root `app/layout.tsx` নিজে ধীর হলে (নিজের folder-এর layout ঢাকে না) |
| ওই route-এর কাছাকাছি নিজের `loading.tsx` না থাকলে — Next.js সবসময় **সবচেয়ে কাছের** `loading.tsx` নেয়। `doctors-management/loading.tsx` থাকলে সেটাই দেখাবে; `ngos`-এর মতো যার নিজের নেই, সেখানে global-টা | Client-side data fetching হলে (যেমন `"use client"` component-এ TanStack Query-এর `useQuery`) — page আগেই render হয়ে যায়, data পরে আসে। তখন নিজে দেখাতে হবে: `if (isLoading) return <p>Loading...</p>;` |
| নিচের nested layout ধীর হলে (যেমন `(dashboardLayout)/layout.tsx`-এ `cookies()`) — এই layout root loading-এর ভেতরে পড়ে, তাই global loading এটাকে ঢাকতে পারে | একই page-এ শুধু query param বদলালে (`?page=1` → `?page=2`) — নতুন data না আসা পর্যন্ত পুরোনো UI-ই দেখায় |
| | Page খুব দ্রুত তৈরি হলে (static page, কোনো `await` নেই) — loading দেখানোর সময়ই পায় না |
| | Error হলে — তখন `error.tsx` দেখায় |

> 📌 এক লাইনে: Global loading সব server-side ধীর page-এর **backup**। কিন্তু কাছাকাছি নিজের `loading.tsx` থাকলে, client-side fetching হলে, বা root layout নিজে ধীর হলে এটা দেখাবে না।

`src/app/loading.tsx`
```tsx
import { LoaderCircle } from "lucide-react";

export default function Loading() {
	return (
		<div className="flex min-h-[60vh] flex-col items-center justify-center gap-3">
			<LoaderCircle className="size-8 animate-spin text-primary" />
			<p className="text-sm text-muted-foreground">Loading...</p>
		</div>
	);
}
```

---

## ধাপ ৪: `cacheComponents` চালু থাকলে loading test করা

আমাদের `next.config.ts`-এ এগুলো চালু আছে:

```ts
cacheComponents: true,
partialPrefetching: true,
```

### ❌ ভুল উপায় (error দেয়)

```tsx
// src/app/(commonLayout)/ngos/page.tsx
export default async function NGOsPage() {
	await new Promise((r) => setTimeout(r, 2000)); // test delay
	return <h1>NGOs</h1>;
}
```

Error: `Next.js encountered uncached data during prerendering or a navigation.`

### কেন error দেয়

এই mode-এ Next.js-এর একটা কড়া নিয়ম: **ধীর কাজ (`await`, `fetch`, delay) page-এর ভেতরেই নিজের `<Suspense>`-এর মধ্যে থাকতে হবে।**

- Page নিজে `async` আর ভেতরে সরাসরি `await` থাকলে error দেয়।
- `app/loading.tsx` বা `(commonLayout)/loading.tsx` রাখলেও এই error যায়নি (নিজে test করে দেখেছি, server restart করার পরও ছিল)।
- Terminal-এ `GET /ngos 200` দেখায়, মানে page চলে। "Error" লেখাটা development-এর check, page crash না।

### ✅ সঠিক উপায়

ধীর কাজটা আলাদা component-এ রাখো, আর সেটাকে `<Suspense>` দিয়ে wrap করো:

```tsx
// src/app/(commonLayout)/ngos/page.tsx
import { Suspense } from "react";
import LoadingSpinner from "@/components/shared/LoadingSpinner";

async function NGOsContent() {
	await new Promise((r) => setTimeout(r, 2000)); // test-এর জন্য, পরে মুছে দিও
	return <h1>NGOs</h1>;
}

export default function NGOsPage() {
	return (
		<Suspense fallback={<LoadingSpinner text="Loading NGOs..." />}>
			<NGOsContent />
		</Suspense>
	);
}
```

### Reusable spinner

```tsx
// src/components/shared/LoadingSpinner.tsx
import { LoaderCircle } from "lucide-react";

export default function LoadingSpinner({ text = "Loading..." }: { text?: string }) {
	return (
		<div className="flex min-h-[60vh] flex-col items-center justify-center gap-3">
			<LoaderCircle className="size-8 animate-spin text-primary" />
			<p className="text-sm text-muted-foreground">{text}</p>
		</div>
	);
}
```

`loading.tsx`-এও এটাই ব্যবহার করা যায়:

```tsx
// src/app/(commonLayout)/loading.tsx
import LoadingSpinner from "@/components/shared/LoadingSpinner";

export default function CommonLoading() {
	return <LoadingSpinner />;
}
```

### মনে রাখার নিয়ম

| ❌ এভাবে না | ✅ এভাবে |
|---|---|
| page নিজে `async`, ভেতরে সরাসরি `await` | page normal function, `await` থাকবে আলাদা component-এ, যেটা `<Suspense>`-এর ভেতরে |

- Error-এর কারণ ছিল **Suspense ঠিক জায়গায় না থাকা**, `loading.tsx` না থাকা না।
- Spinner না দেখালে দেখো `<Suspense fallback>`-এ কী দেওয়া আছে। Page-এর ভেতরে Suspense থাকলে `loading.tsx` না, ওই `fallback`-টাই দেখায়।
- `loading.tsx` রেখে দিলে ক্ষতি নেই, কিন্তু ধীর কাজের জন্য আসল ভরসা page-এর ভেতরের `<Suspense>`।
- পরে API থেকে data আনার সময় `setTimeout`-এর জায়গায় আসল `fetch` বসবে, বাকি গঠন একই থাকবে।

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`error` prop-এর type docs-এর মতো দিলে `digest` সহজে পড়া যায়** — `unknown` দিলেও চলে, কিন্তু Next.js সবসময় `Error` পাঠায়:

   ```tsx
   export default function ErrorPage({ error, retry }: { error: Error & { digest?: string }; retry: () => void }) {
   	// ...
   	{error.digest && <p>Error ID: {error.digest}</p>}
   }
   ```

2. **Button-এর জায়গায় shadcn `Button`** — বাকি site-এর সাথে design মিলবে:

   ```tsx
   import { Button } from "@/components/ui/button";
   <Button variant="outline" onClick={() => retry()}>Try again</Button>
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`app/not-found.tsx`-এ Navbar/Footer থাকবে না**

   **কেন:** Unmatched URL (যেমন `/abcd`) কোনো route group-এর সাথে মেলে না, তাই Next.js শুধু root `app/layout.tsx`-এর ভেতরে `app/not-found.tsx` render করে। `(commonLayout)/layout.tsx` (যেখানে Navbar/Footer থাকবে) মাঝখানে আসেই না:

   ```
   /consultation  →  RootLayout → (commonLayout)/layout [Navbar+Footer] → page     ✅ navbar আছে
   /abcd          →  RootLayout → not-found.tsx                                    ❌ navbar নেই
   ```

   **সমাধান:** Navbar আর Footer সরাসরি `NotFound` component-এর ভেতরে বসাও:

   ```tsx
   // src/app/not-found.tsx
   import Link from "next/link";
   import Navbar from "@/components/shared/Navbar";
   import Footer from "@/components/shared/Footer";

   export default function NotFound() {
   	return (
   		<>
   			<Navbar />
   			<main className="flex min-h-[60vh] flex-col items-center justify-center gap-4 text-center">
   				<h1 className="text-4xl font-bold">404</h1>
   				<p className="text-muted-foreground">The page you are looking for does not exist.</p>
   				<Link href="/" className="underline">
   					Go back home
   				</Link>
   			</main>
   			<Footer />
   		</>
   	);
   }
   ```

   > `Navbar` আর `Footer` এখনো বানানো হয়নি — `(commonLayout)/layout.tsx`-এর জন্য যখন `src/components/shared/`-এ বানাবে, তখন এখানেও একই component import করবে। `min-h-screen` এর বদলে `min-h-[60vh]` দেওয়া হয়েছে, যাতে navbar/footer সহ page অতিরিক্ত লম্বা না হয়।

4. **Dashboard-এর জন্য আলাদা `error.tsx`**

   **কেন:** `error.tsx` নিজের folder-এর `layout.tsx`-কে wrap করে না, শুধু তার **নিচের** অংশ (page) ঢাকে। এখন শুধু `app/error.tsx` আছে, যেটা `(dashboardLayout)/layout.tsx`-এর **উপরে** বসে। তাই dashboard-এর কোনো page-এ error হলে পুরো dashboard layout (sidebar সহ) সরে গিয়ে error page দেখায়।

   `(dashboardLayout)/error.tsx` রাখলে এটা dashboard layout-এর **ভেতরে** বসে:

   ```
   শুধু app/error.tsx থাকলে:
   RootLayout → [app/error.tsx] → (dashboardLayout)/layout [Sidebar] → page 💥
                     ↑ এখানে ধরে → sidebar সহ সব চলে যায় ❌

   (dashboardLayout)/error.tsx থাকলে:
   RootLayout → (dashboardLayout)/layout [Sidebar] → [(dashboardLayout)/error.tsx] → page 💥
                                                         ↑ এখানে ধরে → sidebar থেকে যায় ✅
   ```

   ```tsx
   // src/app/(dashboardLayout)/error.tsx
   "use client";

   import { useEffect } from "react";

   export default function DashboardError({ error, retry }: { error: Error & { digest?: string }; retry: () => void }) {
   	useEffect(() => {
   		console.error(error);
   	}, [error]);

   	return (
   		<div className="flex min-h-[60vh] flex-col items-center justify-center gap-4 text-center">
   			<h2 className="text-2xl font-semibold">Failed to load this section</h2>
   			<p className="text-sm text-muted-foreground">Sidebar থেকে অন্য page-এ যেতে পারো, বা আবার চেষ্টা করো।</p>
   			<button onClick={() => retry()} className="rounded-md border px-4 py-2">
   				Try again
   			</button>
   		</div>
   	);
   }
   ```

   > User sidebar দিয়ে অন্য page-এ চলে যেতে পারে, পুরো app reload করতে হয় না। একইভাবে চাইলে `admin/dashboard/error.tsx`, `doctor/dashboard/error.tsx` আলাদা করেও রাখা যায় — Next.js সবসময় **সবচেয়ে কাছের** `error.tsx` নেয়।
