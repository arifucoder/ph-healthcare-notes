# 15 - Admin Dashboard Page: Data আনা (Server Action + TanStack Query Prefetch)

> Module: 48-4 (Understanding How Dashboard Page Will Be Developed), 48-5 (Creating Dashboard Data Interface, Server Action And Prefetching With Tanstack Query)

> Note 12–14-এ বানানো ছোট component (StatsCard, Bar, Pie) এবার backend-এর আসল data দিয়ে admin dashboard-এ বসানো হলো। সাথে একটা বড় পরিবর্তন: **`httpClient` এখন নিজে থেকেই browser-এর auth cookie backend-এ পাঠায়** — তাই protected API (`/stats`, পরে appointment, doctor management…) সব কাজ করবে।

## 🧭 এক নজরে

```
Browser → /admin/dashboard
  → proxy (note 08): token verify / refresh
  → page.tsx (Server)  → heading সাথে সাথে + <Suspense> skeleton
       → AdminDashboardSection (Server)
            → queryClient.query(adminDashboardQueryOptions())
                 → getAdminDashboardData()  ("use server")
                      → httpClient.get("/stats")  + Cookie: accessToken, session
                           → backend checkAuth → stats.service → data
            → <HydrationBoundary state={dehydrate(queryClient)}>
                 → AdminDashboardContent ("use client")
                      → useQuery(একই options) → cache-এ data আছে → আবার fetch না
                      → StatsCard × 4 + AppointmentBarChart + AppointmentPieChart
```

---

## ধাপ ০: Dashboard page বানানোর ৬টা ধাপ (48-4)

TanStack Query দিয়ে frontend-এ data দেখানোর সময় আমরা সবসময় এই ধাপগুলো মানি (note 06-এর doctors page-এও একই):

| # | ধাপ | এই নোটে কোন file |
|---|---|---|
| ১ | Dashboard Data Interface | `types/dashboard.types.ts` → `IAdminDashboardData` |
| ২ | Server Action (Axios Data Fetching) | `services/dashboard.services.ts` → `getAdminDashboardData` |
| ৩ | Server Action Call In Page File Using TanStack Query Prefetch Pattern | `queries/dashboard.queries.ts` + `admin/dashboard/page.tsx` |
| ৪ | Passing The Data As Props To Client | `HydrationBoundary` (data prop না, cache হিসেবে যায়) |
| ৫ | Calling The TanStack Query Hooks In Client Component | `AdminDashboardContent` → `useQuery` |
| ৬ | Using The Micro Components In The Client Component | `StatsCard`, `AppointmentBarChart`, `AppointmentPieChart` |

> ✏️ ধাপ ৪-এ data আসলে prop হিসেবে যায় না: server-এর QueryClient-এর cache `dehydrate` করে `HydrationBoundary` দিয়ে browser-এর QueryClient-এ পাঠানো হয়, আর client component `useQuery` দিয়ে সেখান থেকে পড়ে। এতে একই data পরে refetch/refresh করা যায়, prop হলে যেত না।

```
types → services → queries → page (prefetch) → HydrationBoundary → client component (useQuery) → micro components
```

---

## ধাপ ১: `src/lib/axios/httpClient.ts` (পুরো file) — cookie পাঠানো

```ts
import { AUTH_COOKIES } from "@/lib/cookieConfig";
import type { ApiResponse } from "@/types/api.types";
import axios, { type AxiosInstance, type Method } from "axios";
import { cookies } from "next/headers";
import { connection } from "next/server";
import "server-only";

const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL;

if (!API_BASE_URL) {
	throw new Error("NEXT_PUBLIC_API_BASE_URL is not defined in environment variables");
}

export interface ApiRequestOptions {
	params?: Record<string, unknown>;
	headers?: Record<string, string>;
}

// browser-এর cookie Next server-এ আসে, কিন্তু axios নিজে থেকে backend-এ পাঠায় না।
// শুধু checkAuth-এর দরকারি দুটো cookie পাঠাই (সব cookie না)।
// refreshToken পাঠাই না: ওটা শুধু refresh endpoint-এর জন্য, proxy সেটা সামলায়।
const getAuthCookieHeader = async (): Promise<string | undefined> => {
	const cookieStore = await cookies();

	const pairs = [AUTH_COOKIES.accessToken, AUTH_COOKIES.sessionToken]
		.map((name) => {
			const value = cookieStore.get(name)?.value;
			return value ? `${name}=${value}` : null;
		})
		.filter((pair): pair is string => pair !== null);

	return pairs.length > 0 ? pairs.join("; ") : undefined;
};

const axiosInstance = async (): Promise<AxiosInstance> => {
	// axios ভেতরে Date.now() ব্যবহার করে, যেটা Next.js চেনে না।
	// connection() Next-কে বলে: এই request আসল request-time-এ চালাও, prerender কোরো না।
	await connection();

	const cookieHeader = await getAuthCookieHeader();

	return axios.create({
		baseURL: API_BASE_URL,
		timeout: 30000,
		headers: {
			"Content-Type": "application/json",
			...(cookieHeader && { Cookie: cookieHeader }),
		},
	});
};

const request = async <TData>(
	method: Method,
	endpoint: string,
	data?: unknown,
	options?: ApiRequestOptions,
): Promise<ApiResponse<TData>> => {
	try {
		const instance = await axiosInstance();
		const response = await instance.request<ApiResponse<TData>>({
			method,
			url: endpoint,
			data,
			params: options?.params,
			headers: options?.headers, // এখানে Cookie দিলে (যেমন logoutAction) উপরেরটার বদলে এটা যায়
		});
		return response.data;
	} catch (error) {
		// পুরো error log কোরো না: error.config.data-তে request body (password) থাকে
		if (axios.isAxiosError(error)) {
			console.error(`[httpClient] ${method} ${endpoint} failed`, {
				status: error.response?.status,
				code: error.code,
			});
		} else {
			console.error(`[httpClient] ${method} ${endpoint} failed`, error instanceof Error ? error.message : error);
		}
		throw error;
	}
};

export const httpClient = {
	get: <TData>(endpoint: string, options?: ApiRequestOptions) => request<TData>("GET", endpoint, undefined, options),

	post: <TData>(endpoint: string, data: unknown, options?: ApiRequestOptions) =>
		request<TData>("POST", endpoint, data, options),

	put: <TData>(endpoint: string, data: unknown, options?: ApiRequestOptions) =>
		request<TData>("PUT", endpoint, data, options),

	patch: <TData>(endpoint: string, data: unknown, options?: ApiRequestOptions) =>
		request<TData>("PATCH", endpoint, data, options),

	delete: <TData>(endpoint: string, options?: ApiRequestOptions) =>
		request<TData>("DELETE", endpoint, undefined, options),
};
```

এই পরিবর্তনের পর থেকে সব protected API call-এ (doctor, appointment...) cookie নিজে থেকেই যাবে।

### সমস্যাটা কী ছিল

```
Browser ──(cookie সহ)──► Next.js server ──(axios, cookie ছাড়া ❌)──► Backend checkAuth → 401
```
- Browser Next.js-এর server-এ request পাঠালে cookie নিজে থেকেই যায়।
- কিন্তু Next.js server থেকে **axios** যখন backend-এ নতুন request পাঠায়, সেটা একদম নতুন request — browser-এর cookie সেখানে **নিজে থেকে যায় না**।
- Backend-এর `checkAuth` cookie থেকে `accessToken` আর `better-auth.session_token` পড়ে → না পেলে 401।

### সমাধান: `getAuthCookieHeader`

```
Browser ──► Next.js server ── cookies() দিয়ে পড়া ── Cookie: accessToken=...; better-auth.session_token=... ──► Backend ✅
```

| অংশ | কাজ |
|---|---|
| `cookies()` | Next.js-এর server function, এই request-এর browser cookie পড়ে |
| `[accessToken, sessionToken]` | শুধু এই দুটো — backend-এর `checkAuth` ঠিক এই দুটোই পড়ে |
| `.map(...)` | প্রতিটা থেকে `"name=value"`, না থাকলে `null` |
| `.filter((pair): pair is string => ...)` | `null` বাদ। `pair is string` (type guard) → TypeScript বোঝে এরপর array-তে শুধু string |
| `.join("; ")` | cookie header-এর format: `a=1; b=2` |
| কোনো cookie না থাকলে `undefined` | logged-out অবস্থায় (public page) খালি header পাঠানো হয় না |
| `...(cookieHeader && { Cookie: cookieHeader })` | cookie থাকলে তবেই `Cookie` header যোগ, নাহলে কিছু না |

### কেন সব cookie না, শুধু দুটো?
- **কম জিনিস পাঠানো = কম ঝুঁকি।** Browser-এ অন্য cookie-ও থাকতে পারে, backend-এর সেগুলো দরকার নেই।
- **`refreshToken` পাঠানো হয় না**: এটা শুধু `/auth/refresh-token` endpoint-এর জন্য, আর সেটা proxy নিজে সামলায় (note 08)।

### `import "server-only"` — নতুন
`cookies()` শুধু server-এ চলে, আর এই file-এ এখন auth cookie নিয়ে কাজ হয়। কোনো client component ভুল করে `httpClient` import করলে **build-এর সময়েই error** (note 07-এর ধাপ ১)। Client থেকে data আনতে হলে সবসময় Server Action (`services/`) দিয়ে।

### আগের নিয়ম যা থাকল
- `connection()` → prerender না করে আসল request-এর সময় চালানো (note 04)।
- Error log-এ পুরো `error` না, শুধু status আর code (password লিক হয় না)।
- `options.headers`-এ নিজে `Cookie` দিলে (যেমন `logoutAction`, note 11) সেটাই যায় — axios-এ request-এর header instance-এর header-এর উপর বসে।

> ✏️ Note 04-এর `httpClient`-এ cookie ছিল না — সেটা এখান থেকে আপডেট ধরো।

---

## ধাপ ২: `src/types/dashboard.types.ts` — `IAdminDashboardData` যোগ

```ts
// backend stats.service.ts-এর getCommonAdminStats / getSuperAdminStats-এর সাথে মিল রাখো
export interface IAdminDashboardData {
	appointmentCount: number;
	doctorCount: number;
	patientCount: number;
	adminCount: number;
	userCount: number;
	paidPaymentCount: number;
	totalRevenue: number;
	superAdminCount?: number; // শুধু SUPER_ADMIN হলে আসে
	pieChartData: PieChartData[];
	barChartData: BarChartData[];
}
```

- Backend-এর `GET /stats` (server note 34, `stats.service.ts`) ADMIN হলে `getCommonAdminStats`, SUPER_ADMIN হলে তার সাথে `superAdminCount` দেয়। এই interface ঠিক সেই shape।
- `superAdminCount?` → optional, কারণ ADMIN-এর response-এ থাকে না।
- `PieChartData[]` (note 13) আর `BarChartData[]` (note 14) আগের type-ই ব্যবহার → chart component-এ সরাসরি দেওয়া যায়।
- Backend-এ field বদলালে এখানেও বদলাতে হবে (frontend Prisma type দেখতে পায় না)।

---

## ধাপ ৩: `src/services/dashboard.services.ts` — Server Action

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { IAdminDashboardData } from "@/types/dashboard.types";

// catch করি না: error হলে throw হবে, তখন TanStack Query isError দেখাতে পারবে।
// "use server" export public endpoint, তবে backend-এর checkAuth caller-এর নিজের cookie-ই যাচাই করে।
export const getAdminDashboardData = async () => httpClient.get<IAdminDashboardData>("/stats");
```

### কেন `"use server"`?
- `httpClient` এখন `server-only`, তাই client component সরাসরি ব্যবহার করতে পারবে না।
- `"use server"` দিলে এই function একটা **Server Action** হয়ে যায়: server-এ prefetch-এর সময় সাধারণ function-এর মতো চলে, আর client-এ `useQuery` যখন refetch করে, তখন browser থেকে একটা `POST` request গিয়ে **server-এ** চলে → `cookies()` পায় → backend-এ cookie যায়।

```
Server prefetch:   page.tsx → getAdminDashboardData()  (সরাসরি function call)
Client refetch:    useQuery → getAdminDashboardData()  → POST (Server Action) → Next server → backend
```

### কেন `catch` নেই?
Error হলে **throw হতে দাও** → TanStack Query সেটা ধরে `isError: true` করে → UI-তে "Try again" দেখানো যায় (note 12–14-এর recommendation এখানে ✅)। এখানে catch করে `null` ফেরত দিলে Query ভাবত সফল, আর error UI কখনো দেখাত না।

### Public endpoint কিনা?
`"use server"` file-এর export বাইরে থেকে call করা যায়। কিন্তু ভয় নেই: backend-এর `checkAuth` **যে call করছে তার নিজের cookie** যাচাই করে। Patient call করলে patient-এর stats পাবে, অন্যের না।

---

## ধাপ ৪: `src/queries/dashboard.queries.ts` (নতুন)

```ts
import { getAdminDashboardData } from "@/services/dashboard.services";
import { queryOptions } from "@tanstack/react-query";

// server prefetch আর client useQuery দুই জায়গাতেই একই key আর setting
export const adminDashboardQueryOptions = () =>
	queryOptions({
		queryKey: ["dashboard", "admin"],
		queryFn: getAdminDashboardData,
		staleTime: 30 * 1000, // ৩০ সেকেন্ড: এর মধ্যে ফিরে এলে আবার backend-এ যায় না
	});
```

| অংশ | কাজ |
|---|---|
| `queryOptions(...)` | TanStack-এর helper — key, function, setting এক জায়গায়, আর type ঠিক থাকে |
| function কেন (`() => queryOptions(...)`) | doctors-এর মতো পরে parameter লাগলে (যেমন filter) সহজে যোগ করা যায় |
| `queryKey: ["dashboard", "admin"]` | cache-এর নাম। Server prefetch আর client `useQuery` **একই key** দিলেই client cache থেকে data পায় |
| `queryFn` | data আনার function (ধাপ ৩) |
| `staleTime: 30 * 1000` | ৩০ সেকেন্ড data "তাজা" ধরা হয় → এর মধ্যে page-এ ফিরে এলে বা tab বদলে এলে আবার backend-এ যায় না। Dashboard-এর সংখ্যা বদলায়, তাই doctors-এর (১ ঘণ্টা) চেয়ে কম |

> একই options server আর client দুই জায়গায় → key বা setting ভুলে আলাদা হওয়ার সুযোগ নেই।

---

## ধাপ ৫: `src/components/modules/Dashboard/AdminDashboardContent.tsx`

```tsx
"use client";

import AppointmentBarChart, { AppointmentBarChartSkeleton } from "@/components/shared/charts/AppointmentBarChart";
import AppointmentPieChart, { AppointmentPieChartSkeleton } from "@/components/shared/charts/AppointmentPieChart";
import StatsCard, { StatsCardSkeleton } from "@/components/shared/cards/StatsCard";
import { Button } from "@/components/ui/button";
import { Card, CardContent } from "@/components/ui/card";
import { adminDashboardQueryOptions } from "@/queries/dashboard.queries";
import { useQuery } from "@tanstack/react-query";
import { CalendarDays, Stethoscope, Users, Wallet } from "lucide-react";

// narrowSymbol: "BDT 12,345"-এর বদলে "৳12,345"
const currencyFormatter = new Intl.NumberFormat("en-US", {
	style: "currency",
	currency: "BDT",
	currencyDisplay: "narrowSymbol",
	maximumFractionDigits: 0,
});

export default function AdminDashboardContent() {
	const { data: response, isError, refetch, isFetching } = useQuery(adminDashboardQueryOptions());
	const stats = response?.data;

	if (!stats) {
		if (isError) {
			return (
				<Card>
					<CardContent className="flex flex-col items-center gap-3 py-10 text-center">
						<p className="text-sm text-muted-foreground">Couldn&apos;t load dashboard data.</p>
						<Button variant="outline" onClick={() => refetch()} disabled={isFetching}>
							{isFetching ? "Retrying..." : "Try again"}
						</Button>
					</CardContent>
				</Card>
			);
		}

		return <AdminDashboardSkeleton />;
	}

	const statCards = [
		{
			title: "Total Appointments",
			value: stats.appointmentCount,
			icon: CalendarDays,
			description: "All appointments booked",
		},
		{ title: "Patients", value: stats.patientCount, icon: Users, description: "Registered patients" },
		{ title: "Doctors", value: stats.doctorCount, icon: Stethoscope, description: "Active doctors" },
		{
			title: "Total Revenue",
			value: currencyFormatter.format(stats.totalRevenue),
			icon: Wallet,
			description: `From ${stats.paidPaymentCount} paid payments`,
		},
	];

	return (
		<div className="space-y-6">
			<div className="grid gap-4 sm:grid-cols-2 xl:grid-cols-4">
				{statCards.map((card) => (
					<StatsCard key={card.title} {...card} />
				))}
			</div>

			<div className="grid gap-4 lg:grid-cols-7">
				<AppointmentBarChart data={stats.barChartData} className="lg:col-span-4" />
				<AppointmentPieChart data={stats.pieChartData} className="lg:col-span-3" />
			</div>
		</div>
	);
}

// আসল layout-এর মতো একই grid, তাই load শেষে কিছু লাফায় না
export function AdminDashboardSkeleton() {
	return (
		<div className="space-y-6">
			<div className="grid gap-4 sm:grid-cols-2 xl:grid-cols-4">
				{Array.from({ length: 4 }, (_, index) => (
					<StatsCardSkeleton key={index} />
				))}
			</div>

			<div className="grid gap-4 lg:grid-cols-7">
				<AppointmentBarChartSkeleton className="lg:col-span-4" />
				<AppointmentPieChartSkeleton className="lg:col-span-3" />
			</div>
		</div>
	);
}
```

### কেন `"use client"`?
`useQuery` hook। আর `StatsCard`-এ icon (function) পাঠানো হচ্ছে — client-এর ভিতরে client, তাই সমস্যা নেই (note 12)।

### `useQuery` থেকে কী নেওয়া হলো

| নাম | মানে |
|---|---|
| `data: response` | backend-এর পুরো response `{ success, message, data }` → নাম বদলে `response` |
| `response?.data` → `stats` | আসল dashboard data (`IAdminDashboardData`) |
| `isError` | fetch fail হয়েছে |
| `refetch` | আবার আনার function ("Try again" button) |
| `isFetching` | এখন fetch চলছে (retry-র সময় button disable) |

### ৩টা অবস্থা

```
stats নেই + isError  → Card: "Couldn't load dashboard data." + [Try again]
stats নেই            → <AdminDashboardSkeleton />  (প্রথমবার load)
stats আছে            → আসল dashboard
```
- **`stats` থাকলে আগে দেখাও:** background refetch fail করলেও (`isError` true) পুরনো data থাকলে সেটাই দেখায় — হঠাৎ পুরো dashboard error-এ বদলে যায় না।
- `Couldn&apos;t` → JSX-এ সরাসরি `'` লিখলে ESLint (`react/no-unescaped-entities`) error দেয়, তাই `&apos;`।

### টাকা format

```ts
new Intl.NumberFormat("en-US", { style: "currency", currency: "BDT", currencyDisplay: "narrowSymbol", maximumFractionDigits: 0 });
// 540000 → "৳540,000"
```
- `currencyDisplay: "narrowSymbol"` → `"BDT 540,000"`-এর বদলে `"৳540,000"`।
- `maximumFractionDigits: 0` → পয়সা নেই।
- Format করা string `StatsCard`-এ যায় → card শুধু দেখায় (note 12-এর নিয়ম)।

### `statCards` array + `map`
৪টা card-এর data একটা array-তে, তারপর `map` করে `<StatsCard key={card.title} {...card} />`।
- `{...card}` → object-এর সব field (`title`, `value`, `icon`, `description`) prop হিসেবে ছড়িয়ে দেয়।
- নতুন card যোগ করতে শুধু array-তে একটা object।

### Layout

| অংশ | Class | ফল |
|---|---|---|
| Stats card | `grid gap-4 sm:grid-cols-2 xl:grid-cols-4` | mobile ১টা, `sm` ২টা, `xl` ৪টা এক সারিতে |
| Chart | `grid gap-4 lg:grid-cols-7` + bar `lg:col-span-4` + pie `lg:col-span-3` | বড় screen-এ bar ৪/৭ চওড়া, pie ৩/৭ |

```
xl:
┌────────┬────────┬────────┬────────┐
│ Appts  │Patients│Doctors │Revenue │
└────────┴────────┴────────┴────────┘
┌───────────────────────┬───────────────┐
│ Bar (4/7)             │ Pie (3/7)     │
└───────────────────────┴───────────────┘
```

### `AdminDashboardSkeleton`
আসল layout-এর **হুবহু একই grid** + প্রতিটা component-এর skeleton → data এলে কিছু লাফায় না। এটা export করা, কারণ page-এর `<Suspense fallback>`-এও লাগে।

---

## ধাপ ৬: `src/app/(dashboardLayout)/admin/dashboard/page.tsx`

তোমার consultation page যেভাবে লেখা (note 06), সেই একই pattern:

```tsx
import AdminDashboardContent, { AdminDashboardSkeleton } from "@/components/modules/Dashboard/AdminDashboardContent";
import { getQueryClient } from "@/lib/getQueryClient";
import { adminDashboardQueryOptions } from "@/queries/dashboard.queries";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import { Suspense } from "react";

async function AdminDashboardSection() {
	const queryClient = getQueryClient();

	// fail হলেও page ভাঙবে না: client-এ useQuery আবার চেষ্টা করবে বা error দেখাবে
	await queryClient.query(adminDashboardQueryOptions()).catch(noop);

	return (
		<HydrationBoundary state={dehydrate(queryClient)}>
			<AdminDashboardContent />
		</HydrationBoundary>
	);
}

export default function AdminDashboardPage() {
	return (
		<div className="space-y-6">
			<div>
				<h1 className="text-2xl font-semibold tracking-tight">Dashboard</h1>
				<p className="text-sm text-muted-foreground">Overview of appointments, users and revenue</p>
			</div>

			{/* Suspense: heading সঙ্গে সঙ্গে দেখায়, data আসা পর্যন্ত skeleton */}
			<Suspense fallback={<AdminDashboardSkeleton />}>
				<AdminDashboardSection />
			</Suspense>
		</div>
	);
}
```

### দুই ভাগ কেন — `AdminDashboardPage` আর `AdminDashboardSection`?

| Component | কী করে | কখন দেখায় |
|---|---|---|
| `AdminDashboardPage` | heading + `<Suspense>` | **সাথে সাথে** |
| `AdminDashboardSection` (`async`) | backend থেকে data এনে client component-এ দেয় | data এলে (ততক্ষণ fallback skeleton) |

- `async` component-কে `<Suspense>`-এর ভিতরে রাখলে Next.js heading আর skeleton আগে পাঠিয়ে দেয় (streaming), data এলে বাকিটা। User ফাঁকা সাদা page দেখে না।
- `cacheComponents: true`-তে cookie/dynamic data `<Suspense>`-এর ভিতরে থাকা ভালো (note 10)।

### Prefetch লাইন

```ts
await queryClient.query(adminDashboardQueryOptions()).catch(noop);
```
- `getQueryClient()` → server-এ **প্রতিটা request-এ নতুন** client (note 05) — এক user-এর data অন্য user পাবে না।
- `queryClient.query(...)` → data এনে cache-এ রাখে।
- `.catch(noop)` → fail করলে page ভাঙবে না (`noop` = কিছু না করা function)। Error-টা client-এ `useQuery` আবার চেষ্টা করে সামলায় → তারপর "Try again"।
- `dehydrate(queryClient)` → cache-কে সাধারণ JSON বানায় → `HydrationBoundary` সেটা browser-এর QueryClient-এ বসায় → `useQuery` সাথে সাথে data পায়।

---

## ধাপ ৭: Data কীভাবে আসে (পুরো ছবি)

```
Browser → /admin/dashboard
  → proxy: token verify (আর দরকার হলে refresh)
  → page (server): heading + skeleton সঙ্গে সঙ্গে পাঠায়
  → AdminDashboardSection: httpClient + cookie → backend /stats → data
  → HydrationBoundary: data browser-এর QueryClient-এ পৌঁছায়
  → AdminDashboardContent: useQuery cache থেকেই data পায় (আবার fetch করে না)
  → ৩০ সেকেন্ড পর tab-এ ফিরে এলে: শুধু তখনই আবার আনে
```

---

## 📁 এই নোটের file-গুলো

```
src/
├── lib/axios/httpClient.ts                    ← আপডেট: cookie পাঠায়, server-only
├── types/dashboard.types.ts                   ← + IAdminDashboardData
├── services/dashboard.services.ts             ← নতুন: "use server" → GET /stats
├── queries/dashboard.queries.ts               ← নতুন: adminDashboardQueryOptions
├── components/modules/Dashboard/
│   └── AdminDashboardContent.tsx              ← নতুন: useQuery + card + chart + skeleton
└── app/(dashboardLayout)/admin/dashboard/page.tsx   ← prefetch + Suspense
```

---

## 🧪 Test

Backend চালু রাখো, ADMIN হয়ে login।

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | `/admin/dashboard` খোলো | heading সাথে সাথে, তারপর ৪টা card + bar + pie আসল সংখ্যা দিয়ে |
| ২ | DevTools → Network (Fetch/XHR) | page load-এ browser থেকে `/stats` call **নেই** (server-এ prefetch হয়েছে) |
| ৩ | Backend terminal | `GET /api/v1/stats` 200 |
| ৪ | Revenue card | `৳540,000`-এর মতো |
| ৫ | অন্য tab-এ গিয়ে ৩০ সেকেন্ডের পর ফিরে এসো | Network-এ একটা `POST` (Server Action) → নতুন data |
| ৬ | ৩০ সেকেন্ডের আগে ফিরে এসো | কোনো request নেই |
| ৭ | Backend বন্ধ করে page reload | skeleton → (কয়েকবার retry-র পর) "Couldn't load dashboard data." |
| ৮ | Backend চালু করে "Try again" | "Retrying..." → dashboard |
| ৯ | SUPER_ADMIN হয়ে | একই dashboard (`superAdminCount` এখন দেখানো হয় না) |

---

## 🔜 পরে যা যোগ হবে
- [ ] Doctor dashboard (`/doctor/dashboard`) আর Patient dashboard (`/dashboard`) — একই ৬ ধাপ, backend `/stats` role অনুযায়ী আলাদা data দেয়
- [ ] SUPER_ADMIN হলে `superAdminCount`-এর card
- [ ] অন্য protected page (appointments, doctors management) — `httpClient` এখন cookie পাঠায়, তাই শুধু service + query + page

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Error দেখাতে কয়েক সেকেন্ড লাগে** — TanStack Query client-এ default-এ fail হলে **৩ বার retry** করে (১s, ২s, ৪s অপেক্ষা)। তাই backend বন্ধ থাকলে "Couldn't load" আসতে ~৭ সেকেন্ড skeleton দেখায়। Dashboard-এর জন্য ১ বারই যথেষ্ট:
   ```ts
   queryOptions({
   	queryKey: ["dashboard", "admin"],
   	queryFn: getAdminDashboardData,
   	staleTime: 30 * 1000,
   	retry: 1,
   });
   ```

2. **401 হলে retry অর্থহীন** — session শেষ হলে backend 401 দেয়, আবার চেষ্টা করলেও 401। তখন login-এ পাঠানো ভালো। পরে অনেক page হলে `getQueryClient`-এর default-এ একবারেই নিয়ম দিতে পারো (যেমন 401/403 হলে retry না)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Client থেকে Server Action একটার পর একটা চলে** — Next.js একই browser থেকে একসাথে কয়েকটা Server Action এলে **লাইনে দাঁড় করায়** (parallel না)। Dashboard-এ একটাই query, তাই সমস্যা নেই। কিন্তু পরে একটা page-এ অনেকগুলো query client-এ একসাথে refetch হলে ধীর লাগতে পারে — তখন বেশিরভাগ data server prefetch-এই আনো।

4. **Query key-তে user নেই** — `["dashboard", "admin"]` সব admin-এর জন্য একই key। Browser-এর cache একজন user-এরই, আর logout-এ `queryClient.clear()` (note 11) cache মোছে, তাই অন্য user আগের data দেখবে না। Logout-এ `clear()` কখনো সরিয়ো না।

5. **`AdminDashboardContent` শুধু ADMIN shape বোঝে** — backend `/stats` DOCTOR/PATIENT-কে অন্য shape দেয়। Proxy শুধু ADMIN/SUPER_ADMIN-কে `/admin/dashboard`-এ ঢুকতে দেয়, তাই এখন ঠিক আছে। Doctor dashboard বানানোর সময় আলাদা interface (`IDoctorDashboardData`) + আলাদা query key (`["dashboard", "doctor"]`) দিও।
