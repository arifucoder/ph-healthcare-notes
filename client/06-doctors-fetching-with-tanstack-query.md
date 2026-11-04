# 06 - TanStack Query দিয়ে Doctors Fetching (SSR + Hydration)

Doctors গুলো teacher-কে follow করে `src/app/(commonLayout)/consultation/page.tsx`-এ দেখাব।

## 🧩 পুরো pattern এক নজরে (যেকোনো project-এ এটাই)

```
১. types      → data দেখতে কেমন          (IDoctor)
২. services   → API call                 (getDoctors → httpClient.get)
৩. queries    → queryKey + queryFn + cache time একসাথে  (doctorsQueryOptions)
৪. client     → useQuery দিয়ে data দেখানো  (DoctorsList — "use client")
৫. server page→ আগেই data এনে client-এ পাঠানো (query + dehydrate + HydrationBoundary + Suspense)
```

```
Browser request /consultation
  → Server: DoctorsSection  →  queryClient.query(doctorsQueryOptions)  → API থেকে data
           → dehydrate(cache) → HTML-এর সাথে পাঠায়
  → Browser: HydrationBoundary cache বসায় → DoctorsList-এর useQuery cache-এ data পায় → fetch ছাড়াই সাথে সাথে দেখায় ✅
```

> নতুন কোনো resource (যেমন specialties, appointments) — এই ৫টা ধাপ একই ক্রমে বানালেই হবে।

| Layer | Folder | কেন আলাদা |
|---|---|---|
| Type | `src/types/` | সব জায়গায় একই type, autocomplete |
| Service | `src/services/` | API URL এক জায়গায় — বদলালে এক জায়গায় |
| Query options | `src/queries/` | server (prefetch) আর client (useQuery) **একই** key/fn ব্যবহার করে — key মিলতেই হবে |
| Component | `src/components/modules/<Feature>/` | UI + interaction |
| Page | `src/app/.../page.tsx` | server-এ data আনা |

---

## ধাপ ১: Types

`src/types/specialty.types.ts`
```ts
export interface ISpecialty {
	id: string;
	title: string;
	icon?: string;
}
```

`src/types/doctor.types.ts`
```ts
export interface IDoctor {
	id: string;
	name: string;
	email: string;
	profilePhoto?: string;
	appointmentFee: number;
	designation: string;
	currentWorkingPlace: string;
	experience: number;
	averageRating: number;
	specialties: Array<{
		specialty: { id: string; title: string };
	}>;
}
```

> ✏️ `experience` আর `averageRating` নোটে ছিল না — কিন্তু `DoctorCard` (ধাপ ৭) এ দুটো ব্যবহার করে, তাই এগুলো ছাড়া `tsc`-এ error দেয় (`Property 'experience' does not exist on type 'IDoctor'` — project-এ চালিয়ে দেখা হয়েছে)। Server-এর `doctor.prisma`-তেও দুটো আছে।

`specialties`-এর shape এমন কেন? Server-এ doctor আর specialty-র মাঝে একটা join table (`DoctorSpecialty`) আছে, আর server `include: { specialties: { include: { specialty: true } } }` দিয়ে পাঠায় — তাই প্রতিটা item-এর ভেতরে আবার `specialty`।

## ধাপ ২: Services — API call

`src/services/doctor.services.ts`
```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { IDoctor } from "@/types/doctor.types";

export const getDoctors = async (queryString: string) => {
	return httpClient.get<IDoctor[]>(queryString ? `/doctors?${queryString}` : "/doctors");
};
```

`src/services/specialty.services.ts`
```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { ISpecialty } from "@/types/specialty.types";

export const getAllSpecialties = async () => {
	return httpClient.get<ISpecialty[]>("/specialties");
};
```

- `queryString` = `page=1&limit=12&searchTerm=...` → request যায় `/doctors?page=1&limit=12...`
- **`"use server"` কেন:** এই function গুলো সবসময় **server-এ** চলবে — client component থেকে call করলেও (Next.js নিজে server-এ পাঠিয়ে চালায়, এদের **Server Action** বলে)। সুবিধা: পরে `httpClient`-এ `cookies()` দিয়ে token যোগ হলে সেটা server-এই কাজ করবে, browser থেকে API URL/token দেখা যাবে না।

## ধাপ ৩: Query options — key + fn + cache এক জায়গায়

`src/queries/doctor.queries.ts`
```ts
import { getDoctors } from "@/services/doctor.services";
import { queryOptions } from "@tanstack/react-query";

export const doctorsQueryOptions = (queryString: string) =>
	queryOptions({
		queryKey: ["doctors", queryString],
		queryFn: () => getDoctors(queryString),
		staleTime: 1000 * 60 * 60, // 1 ঘণ্টা
		gcTime: 1000 * 60 * 60 * 6, // 6 ঘণ্টা
	});
```

`src/queries/specialty.queries.ts`
```ts
import { getAllSpecialties } from "@/services/specialty.services";
import { queryOptions } from "@tanstack/react-query";

export const specialtiesQueryOptions = () =>
	queryOptions({
		queryKey: ["specialties"],
		queryFn: getAllSpecialties,
		staleTime: 1000 * 60 * 60 * 6,
		gcTime: 1000 * 60 * 60 * 24,
	});
```

| Option | মানে |
|---|---|
| `queryOptions()` | TanStack-এর helper — option গুলো এক object-এ রাখে আর type ঠিক রাখে। Server আর client দুই জায়গায় **একই function** call করলে key কখনো ভুল হয় না |
| `queryKey: ["doctors", queryString]` | cache-এর নাম। `queryString` key-তে আছে, তাই page 1 আর page 2 আলাদা আলাদা cache |
| `staleTime` | এতক্ষণ data "fresh" — আবার API-তে যাবে না। Doctor list ১ ঘণ্টা, specialty কম বদলায় তাই ৬ ঘণ্টা |
| `gcTime` | কোনো component ব্যবহার না করলে cache এতক্ষণ memory-তে থাকে, তারপর মুছে যায় (gc = garbage collection) |

> `staleTime` এখানে দিলে `getQueryClient.ts`-এর default (১ মিনিট) override হয়।

## ধাপ ৪: Client component — `useQuery` দিয়ে দেখানো

`src/components/modules/Consultation/DoctorsList.tsx`
```tsx
"use client";

import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import { useQuery } from "@tanstack/react-query";

export default function DoctorsList({ queryString }: { queryString: string }) {
	const { data: doctorsRes, isFetching } = useQuery(doctorsQueryOptions(queryString));
	const { data: specialtiesRes } = useQuery(specialtiesQueryOptions());

	const doctors = doctorsRes?.data ?? [];
	const specialties = specialtiesRes?.data ?? [];

	return (
		<section className="space-y-6 p-6">
			<div className="flex flex-wrap gap-2">
				{specialties.map((s) => (
					<span key={s.id} className="rounded-full border px-3 py-1 text-sm">
						{s.title}
					</span>
				))}
			</div>

			{isFetching && <p className="text-sm text-muted-foreground">Updating...</p>}

			{doctors.length === 0 ? (
				<p>No doctors found.</p>
			) : (
				<ul className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
					{doctors.map((doctor) => (
						<li key={doctor.id} className="rounded-lg border p-4">
							<h3 className="font-semibold">{doctor.name}</h3>
							<p className="text-sm text-muted-foreground">{doctor.designation}</p>
							<p className="text-sm">Fee: ৳{doctor.appointmentFee}</p>
						</li>
					))}
				</ul>
			)}
		</section>
	);
}
```

- `useQuery(doctorsQueryOptions(queryString))` — page যে key দিয়ে prefetch করেছে, এখানেও **একই key** → cache-এ data আগে থেকেই আছে, তাই প্রথমবার কোনো loading ছাড়াই দেখায়।
- `doctorsRes` = পুরো response `{ success, message, data, meta }` → list পেতে `.data`।
- `?? []` — data না থাকলে খালি array, যাতে `.map` crash না করে।
- `isFetching` — background-এ নতুন করে আনার সময় `true` ("Updating..." দেখাই)। (`isLoading` শুধু প্রথমবার, যখন cache-এ কিছুই নেই।)

## ধাপ ৫: Server page — prefetch + hydrate

`src/app/(commonLayout)/consultation/page.tsx`
```tsx
import DoctorsList from "@/components/modules/Consultation/DoctorsList";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { getQueryClient } from "@/lib/getQueryClient";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import { Suspense } from "react";

const DEFAULT_PAGE = "1";
const DEFAULT_LIMIT = "12";

const ALLOWED_QUERY_KEYS = new Set(["page", "limit", "sortBy", "sortOrder", "searchTerm"]);

type SearchParams = Record<string, string | string[] | undefined>;

function buildQueryString(params: SearchParams) {
	const result = new URLSearchParams();

	for (const [key, raw] of Object.entries(params)) {
		if (!ALLOWED_QUERY_KEYS.has(key) || raw === undefined) continue;

		const values = Array.isArray(raw) ? raw : [raw];
		for (const value of values) {
			const trimmed = value.trim();
			if (trimmed) result.append(key, trimmed);
		}
	}

	// URL-এ না থাকলে default বসাও, যাতে cache key সবসময় একই থাকে
	if (!result.has("page")) result.set("page", DEFAULT_PAGE);
	if (!result.has("limit")) result.set("limit", DEFAULT_LIMIT);

	return result.toString();
}

async function DoctorsSection({ searchParams }: { searchParams: Promise<SearchParams> }) {
	const queryString = buildQueryString(await searchParams);
	const queryClient = getQueryClient();

	await Promise.all([
		queryClient.query(doctorsQueryOptions(queryString)).catch(noop),
		queryClient.query(specialtiesQueryOptions()).catch(noop),
	]);

	return (
		<HydrationBoundary state={dehydrate(queryClient)}>
			<DoctorsList queryString={queryString} />
		</HydrationBoundary>
	);
}

export default function ConsultationPage({ searchParams }: PageProps<"/consultation">) {
	return (
		<Suspense fallback={<LoadingSpinner text="Loading doctors..." />}>
			<DoctorsSection searchParams={searchParams} />
		</Suspense>
	);
}
```

### ভেঙে বোঝা

**`buildQueryString`** — URL-এর `?page=2&searchTerm=ali` থেকে query string বানায়:

| কাজ | কেন |
|---|---|
| শুধু `ALLOWED_QUERY_KEYS` নেয় | URL-এ আজেবাজে param (`?abc=1`) দিলে API-তে যাবে না, আর cache key-ও নষ্ট হবে না |
| `Array.isArray(raw)` | `?page=1&page=2`-এর মতো একই key বারবার থাকলে array আসে |
| `trim()` + খালি বাদ | `?searchTerm=` (খালি) পাঠাবে না |
| default `page=1`, `limit=12` | `/consultation` আর `/consultation?page=1&limit=12` একই data — তাই একই key হওয়া উচিত। নাহলে একই data দুবার cache হতো |

```
/consultation?searchTerm=ali&abc=x   →   "searchTerm=ali&page=1&limit=12"
```

**`DoctorsSection`** (async server component):

| লাইন | কাজ |
|---|---|
| `await searchParams` | Next.js 15+ এ `searchParams` একটা Promise |
| `getQueryClient()` | server-এ এই request-এর জন্য নতুন QueryClient |
| `queryClient.query(...)` | server-এ data এনে cache-এ রাখে (আগে নাম ছিল `prefetchQuery`/`fetchQuery` — v5.105-এ এগুলো deprecated, `query` নতুন নাম) |
| `.catch(noop)` | API fail করলে page crash না করে চুপ থাকে → client-এ `useQuery` নিজে আবার চেষ্টা করবে। `noop` = কিছু না করা function |
| `Promise.all` | doctors আর specialties **একসাথে** আনে (একটার পর একটা না) → দ্রুত |
| `dehydrate(queryClient)` | cache-কে JSON বানায় |
| `<HydrationBoundary state>` | সেই JSON browser-এর QueryClient-এ বসিয়ে দেয় → ভেতরের `useQuery` cache পেয়ে যায় |

**`ConsultationPage`** — page নিজে `async` না; ধীর কাজটা (`DoctorsSection`) `<Suspense>`-এর ভেতরে। কারণ `cacheComponents: true` (নোট 03, ধাপ ৪)। Data আসা পর্যন্ত spinner দেখায়।

## ধাপ ৬: Design — shadcn card

```shell
pnpm dlx shadcn@latest add card
pnpm dlx shadcn@latest add avatar badge
```

> ✏️ `avatar` আর `badge` add করার command নোটে ছিল না — `DoctorCard` এ দুটো import করে (তোমার project-এ আগেই আছে)।

## ধাপ ৭: `DoctorCard.tsx` (নতুন)

`src/components/modules/Consultation/DoctorCard.tsx`
```tsx
import Link from "next/link";
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardFooter, CardHeader } from "@/components/ui/card";
import type { IDoctor } from "@/types/doctor.types";

const getDoctorInitials = (name: string) => {
	const parts = name.trim().split(/\s+/);
	const initials = parts.slice(0, 2).map((part) => part[0]?.toUpperCase() ?? "");
	return initials.join("") || "DR";
};

export default function DoctorCard({ doctor }: { doctor: IDoctor }) {
	const specialtyTitles = doctor.specialties.map((s) => s.specialty.title);

	return (
		<Card className="relative h-full gap-4 overflow-hidden rounded-2xl transition-all duration-200 hover:-translate-y-0.5 hover:shadow-md">
			<div className="pointer-events-none absolute left-0 top-0 h-1 w-full bg-linear-to-r from-cyan-500 via-sky-500 to-blue-500 opacity-80" />

			<CardHeader className="flex items-start gap-3">
				<Avatar className="size-14 ring-2 ring-blue-100">
					<AvatarImage src={doctor.profilePhoto ?? undefined} alt={doctor.name} />
					<AvatarFallback>{getDoctorInitials(doctor.name)}</AvatarFallback>
				</Avatar>

				<div className="min-w-0 space-y-1">
					<h3 className="truncate text-base font-semibold">{doctor.name}</h3>
					<p className="truncate text-xs text-muted-foreground">{doctor.designation || "N/A"}</p>
					<p className="text-xs text-muted-foreground">{doctor.currentWorkingPlace || "N/A"}</p>
					<p className="truncate text-xs text-muted-foreground">{doctor.email}</p>
				</div>
			</CardHeader>

			<CardContent className="space-y-3">
				<div className="grid gap-2 rounded-lg bg-muted/40 p-3 text-sm">
					<p>
						<span className="font-medium">Experience:</span> {doctor.experience ?? 0} years
					</p>
					<p>
						<span className="font-medium">Fee:</span> ৳{doctor.appointmentFee}
					</p>
					<p>
						<span className="font-medium">Rating:</span> {doctor.averageRating.toFixed(1)}
					</p>
				</div>

				<div className="flex flex-wrap gap-2">
					{specialtyTitles.length > 0 ? (
						specialtyTitles.slice(0, 3).map((title) => (
							<Badge key={`${doctor.id}-${title}`} variant="secondary">
								{title}
							</Badge>
						))
					) : (
						<Badge variant="secondary">No specialties</Badge>
					)}
				</div>
			</CardContent>

			<CardFooter className="mt-auto">
				<Button asChild className="w-full">
					<Link href={`/consultation/doctor/${doctor.id}`}>View Details</Link>
				</Button>
			</CardFooter>
		</Card>
	);
}
```

- এখানে কোনো `"use client"` নেই — কোনো hook/click handler নেই। তবে client component (`DoctorsList`) থেকে import হচ্ছে, তাই এটাও client-এ চলবে (নোট 05, ধাপ ৪-এর নিয়ম)।
- `getDoctorInitials("Abdul Karim")` → `"AK"` — ছবি না থাকলে Avatar-এ এটা দেখায়।
- `slice(0, 3)` — বেশি specialty থাকলে প্রথম ৩টা।
- `Button asChild` — Button-এর design, কিন্তু আসলে `<Link>` render হয় → `/consultation/doctor/:id` (নোট 02-এর dynamic route)।

## ধাপ ৮: `DoctorsList.tsx`-এ card ব্যবহার (পুরো code)

```tsx
"use client";

import { useQuery } from "@tanstack/react-query";
import { Badge } from "@/components/ui/badge";
import DoctorCard from "@/components/modules/Consultation/DoctorCard";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";

export default function DoctorsList({ queryString }: { queryString: string }) {
	const { data: doctorsRes, isFetching } = useQuery(doctorsQueryOptions(queryString));
	const { data: specialtiesRes } = useQuery(specialtiesQueryOptions());

	const doctors = doctorsRes?.data ?? [];
	const meta = doctorsRes?.meta;
	const specialties = specialtiesRes?.data ?? [];

	return (
		<section className="mx-auto w-full max-w-7xl space-y-6 px-4 py-8">
			<div className="relative overflow-hidden rounded-2xl border bg-linear-to-br from-cyan-50 via-white to-blue-50 p-6 dark:from-cyan-950/40 dark:via-background dark:to-blue-950/40">
				<div className="absolute -right-10 -top-10 h-40 w-40 rounded-full bg-blue-200/30 blur-2xl" />
				<div className="absolute -bottom-10 -left-10 h-36 w-36 rounded-full bg-cyan-200/30 blur-2xl" />
				<div className="relative space-y-3">
					<h1 className="text-2xl font-semibold tracking-tight sm:text-3xl">Consult With Our Specialists</h1>
					<p className="max-w-3xl text-sm text-muted-foreground sm:text-base">
						Discover trusted doctors, compare experience and fees, and open detailed profiles to find the
						right specialist.
					</p>
				</div>
			</div>

			{specialties.length > 0 && (
				<div className="rounded-2xl border bg-card p-4 shadow-sm sm:p-5">
					<p className="mb-3 text-sm font-medium">Specialties</p>
					<div className="flex flex-wrap gap-2">
						{specialties.map((s) => (
							<Badge key={s.id} variant="outline">
								{s.title}
							</Badge>
						))}
					</div>
				</div>
			)}

			{isFetching && <p className="text-sm text-muted-foreground">Updating doctors...</p>}

			{doctors.length === 0 ? (
				<div className="rounded-2xl border border-dashed p-10 text-center text-muted-foreground">
					No doctors found.
				</div>
			) : (
				<div className="grid gap-5 sm:grid-cols-2 lg:grid-cols-3">
					{doctors.map((doctor) => (
						<DoctorCard key={doctor.id} doctor={doctor} />
					))}
				</div>
			)}

			<p className="text-center text-sm text-muted-foreground">
				Total {meta?.total ?? doctors.length} doctors
			</p>
		</section>
	);
}
```

- আগের `<li>`-এর জায়গায় এখন `<DoctorCard />`।
- `meta?.total` — server-এর pagination meta থেকে মোট doctor সংখ্যা (শুধু এই page-এর না)।

## ✅ নতুন project-এ implement করার checklist

- [ ] `getQueryClient.ts` + `QueryProvider` + layout-এ wrap (নোট 05) — একবারই
- [ ] `types/x.types.ts` — response-এর shape
- [ ] `services/x.services.ts` — `httpClient.get<IX[]>(...)`
- [ ] `queries/x.queries.ts` — `queryOptions({ queryKey: ["x", params], queryFn, staleTime })`
- [ ] `components/modules/X/XList.tsx` — `"use client"` + `useQuery(xQueryOptions(params))`
- [ ] `page.tsx` — `<Suspense>` → async section → `queryClient.query(...)` → `<HydrationBoundary state={dehydrate(queryClient)}>` → `<XList />`
- [ ] Server আর client-এ **একই** `xQueryOptions(params)` call — key না মিললে client আবার fetch করবে

## 🔜 পরে যা যোগ হবে

- [ ] Search box, sort, pagination (URL-এর `searchParams` বদলানো)
- [ ] Specialty দিয়ে filter (`ALLOWED_QUERY_KEYS`-এ নতুন key)
- [ ] `/consultation/doctor/[id]` — single doctor page

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. ✅ (Done) **Project-এর `src/types/doctor.types.ts`-এ `experience` আর `averageRating` যোগ করা হয়েছে** — এখন `tsc` error নেই।

   ~~আগে:~~ — নোটে ঠিক করা হয়েছে (ধাপ ১), কিন্তু project file-এ এখনো পুরোনোটা। `pnpm dev` চলবে, কিন্তু `pnpm build` এই type error-এ fail করবে। দুটো লাইন যোগ করো:

   ```ts
   experience: number;
   averageRating: number;
   ```

### 🟡 ভালো practice

2. **`doctor.experience ?? 0` অপ্রয়োজনীয়** — type-এ `experience: number` (server-এ `@default(0)`), তাই কখনো `undefined` হবে না। শুধু `{doctor.experience}` দিলেই হয়।

3. **Query string বদলালে পুরোনো list ধরে রাখো** — page 2-তে গেলে নতুন key, cache খালি, তাই list এক মুহূর্তের জন্য খালি হয়ে যায়। `placeholderData: keepPreviousData` দিলে নতুন data না আসা পর্যন্ত আগেরটা থাকে:

   ```ts
   import { keepPreviousData, queryOptions } from "@tanstack/react-query";

   queryOptions({
   	queryKey: ["doctors", queryString],
   	queryFn: () => getDoctors(queryString),
   	placeholderData: keepPreviousData,
   	// ...
   });
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **`"use server"` service client থেকে call হলে সেটা Server Action (POST request)** — Next.js server action গুলো **একটা একটা করে** (sequential) চালায়। তাই client-এ একসাথে অনেক query refetch হলে একটু ধীর লাগতে পারে। এখন সমস্যা নেই; পরে ধীর লাগলে কারণ এটাই।

5. **Filter-এর জন্য `ALLOWED_QUERY_KEYS` বাড়াতে হবে** — server-এর `doctorFilterableFields`-এ যে field আছে (যেমন specialty, gender), সেগুলো এখানে যোগ না করলে URL-এ দিলেও API-তে যাবে না।
