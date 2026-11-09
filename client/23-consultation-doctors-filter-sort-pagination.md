# 23 - Consultation Page: Doctors Filter, Sort ও Pagination (Public)

> Public `/consultation` page-এ doctor card-এর list-এ এখন **search, gender, fee range, specialty (একাধিক), sort আর pagination**।
> Admin doctor table-এর (note 17) একই hook (`useTableUrlState`), একই filter component, একই backend endpoint (`/doctors`) — শুধু UI টা table-এর বদলে card grid।
> আগের version: note 06 (SSR + hydration দিয়ে doctors fetching)।

```
URL (?searchTerm=..&gender=..&page=2)  →  buildQueryString (allowlist + default)  →  useQuery(["doctors", qs])  →  card grid
        ↑ filter/sort/page বদলালে pushState (server আবার চলে না)
```

| | Admin table (note 17) | Consultation (এই note) |
|---|---|---|
| UI | TanStack Table | `DoctorCard` grid (3 column) |
| `limit` default | 10 | 12 (3-column grid-এ ৪ সারি) |
| Specialty filter | multi-select dropdown | chip button (toggle) |
| Sort | column header ক্লিক | একটা dropdown (`"field:order"`) |
| Pagination | `DataTablePagination` | ছোট `DoctorsPagination` (Previous / Next) |

---

## ধাপ ১: `src/components/modules/Consultation/consultation.config.ts` (নতুন)

Filter key আর default এক জায়গায়। Server page আর client `DoctorsList` দুজনেই এটা থেকে query string বানায় → **একই cache key**, তাই SSR-এর data client-এ আবার fetch হয় না।

```ts
import {
	DOCTOR_FILTER_KEYS,
	FEE_MAX_KEY,
	FEE_MIN_KEY,
	GENDER_OPTIONS,
	SPECIALTY_FILTER_KEY,
} from "@/components/modules/Admin/DoctorsManagement/doctorsTable.config";

// filter-এর নিয়ম admin doctor table-এর সাথে এক: একই backend endpoint (/doctors)
export { FEE_MAX_KEY, FEE_MIN_KEY, GENDER_OPTIONS, SPECIALTY_FILTER_KEY };

export const CONSULTATION_DEFAULTS = { page: "1", limit: "12" } as const; // 3-column grid → 12

export const CONSULTATION_FILTER_KEYS = DOCTOR_FILTER_KEYS;

// page (server) আর DoctorsList (client) দুজনেই এটা দিয়ে query string বানায় → একই cache key
export const CONSULTATION_QUERY_KEYS: ReadonlySet<string> = new Set([
	"page",
	"limit",
	"sortBy",
	"sortOrder",
	...CONSULTATION_FILTER_KEYS,
]);

// "field:order": একটা dropdown-এ দুটো মান (backend doctorSortableFields অনুযায়ী)
export const SORT_OPTIONS = [
	{ label: "Fee: low to high", value: "appointmentFee:asc" },
	{ label: "Fee: high to low", value: "appointmentFee:desc" },
	{ label: "Most experienced", value: "experience:desc" },
	{ label: "Top rated", value: "averageRating:desc" },
	{ label: "Name: A–Z", value: "name:asc" },
];
```

- `DOCTOR_FILTER_KEYS` = `searchTerm`, `gender`, `specialties.specialty.title`, `appointmentFee[gte]`, `appointmentFee[lte]` (note 17-এর config থেকে)।
- `CONSULTATION_QUERY_KEYS` = **allowlist** — URL-এ অন্য কোনো param থাকলে (যেমন `utm_source`) backend-এ যায় না, cache key-ও নষ্ট হয় না।
- `SORT_OPTIONS`-এর value `"field:order"` — একটা dropdown-এ `sortBy` আর `sortOrder` দুটোই ধরা যায়। field-গুলো backend-এর sortable field list-এ থাকতে হবে।

---

## ধাপ ২: `src/components/modules/Consultation/DoctorsPagination.tsx` (নতুন)

Card list-এর জন্য ছোট pagination — শুধু Previous / Next আর "Page X of Y"।

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { ChevronLeft, ChevronRight } from "lucide-react";

interface DoctorsPaginationProps {
	page: number;
	totalPages: number;
	disabled: boolean;
	onPageChange: (page: number) => void;
}

export default function DoctorsPagination({ page, totalPages, disabled, onPageChange }: DoctorsPaginationProps) {
	if (totalPages <= 1) return null;

	return (
		<nav aria-label="Doctors pagination" className="flex items-center justify-center gap-3">
			<Button variant="outline" disabled={disabled || page <= 1} onClick={() => onPageChange(page - 1)}>
				<ChevronLeft />
				Previous
			</Button>
			<span className="text-sm tabular-nums text-muted-foreground">
				Page {page} of {totalPages}
			</span>
			<Button variant="outline" disabled={disabled || page >= totalPages} onClick={() => onPageChange(page + 1)}>
				Next
				<ChevronRight />
			</Button>
		</nav>
	);
}
```

- page ১টা হলে কিছুই দেখায় না (`return null`)।
- `disabled` = fetch চলার সময় বারবার ক্লিক বন্ধ।
- `tabular-nums` → সংখ্যা বদলালে লেখা নড়ে না।
- `<nav aria-label>` → screen reader বোঝে এটা pagination।

---

## ধাপ ৩: `src/components/modules/Consultation/DoctorsList.tsx` (পুরোটা বদলাও)

```tsx
"use client";

import DoctorCard from "@/components/modules/Consultation/DoctorCard";
import DataTableSearch from "@/components/shared/table/DataTableSearch";
import DataTableRangeFilter from "@/components/shared/table/filters/DataTableRangeFilter";
import DataTableSelectFilter from "@/components/shared/table/filters/DataTableSelectFilter";
import { Button } from "@/components/ui/button";
import { useTableUrlState } from "@/hooks/useTableUrlState";
import { buildQueryString } from "@/lib/buildQueryString";
import { cn } from "@/lib/utils";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import { keepPreviousData, useQuery } from "@tanstack/react-query";
import { X } from "lucide-react";
import { useEffect } from "react";
import {
	CONSULTATION_DEFAULTS,
	CONSULTATION_FILTER_KEYS,
	CONSULTATION_QUERY_KEYS,
	FEE_MAX_KEY,
	FEE_MIN_KEY,
	GENDER_OPTIONS,
	SORT_OPTIONS,
	SPECIALTY_FILTER_KEY,
} from "./consultation.config";
import DoctorsPagination from "./DoctorsPagination";

export default function DoctorsList() {
	// URL = source of truth। pushState শুধু URL বদলায়, server আবার চলে না; নতুন data আনে useQuery।
	const table = useTableUrlState({ defaultLimit: Number(CONSULTATION_DEFAULTS.limit) });
	const queryString = buildQueryString(table.searchParams, CONSULTATION_QUERY_KEYS, CONSULTATION_DEFAULTS);

	const {
		data: doctorsRes,
		isFetching,
		isError,
	} = useQuery({
		...doctorsQueryOptions(queryString),
		placeholderData: keepPreviousData, // নতুন result আসা পর্যন্ত পুরনোটা দেখায়
	});
	const { data: specialtiesRes } = useQuery(specialtiesQueryOptions());

	const doctors = doctorsRes?.data ?? [];
	const meta = doctorsRes?.meta;
	const specialties = specialtiesRes?.data ?? [];

	// filter দেওয়ার পর page সংখ্যা কমে গেলে শেষ page-এ ফেরা
	const { setPage } = table;
	useEffect(() => {
		if (meta && meta.totalPages > 0 && meta.page > meta.totalPages) setPage(meta.totalPages);
	}, [meta, setPage]);

	// ---------- specialty (একাধিক) ----------
	const selectedSpecialties = table.getFilterList(SPECIALTY_FILTER_KEY);
	const toggleSpecialty = (title: string) =>
		table.setFilters({
			[SPECIALTY_FILTER_KEY]: selectedSpecialties.includes(title)
				? selectedSpecialties.filter((item) => item !== title)
				: [...selectedSpecialties, title],
		});

	// ---------- sort: "field:order" ↔ SortingState ----------
	const [currentSort] = table.sorting;
	const sortValue = currentSort ? `${currentSort.id}:${currentSort.desc ? "desc" : "asc"}` : undefined;
	const handleSortChange = (value: string | undefined) => {
		if (!value) return table.setSorting([]);
		const [id = "createdAt", order] = value.split(":");
		table.setSorting([{ id, desc: order === "desc" }]);
	};

	const hasActiveFilters = CONSULTATION_FILTER_KEYS.some((key) => table.searchParams.has(key)) || Boolean(currentSort);

	const clearAll = () => table.clearParams([...CONSULTATION_FILTER_KEYS, "sortBy", "sortOrder"]);

	const handlePageChange = (page: number) => {
		table.setPage(page);
		window.scrollTo({ top: 0, behavior: "smooth" });
	};

	return (
		<section className="mx-auto w-full max-w-7xl space-y-6 px-4 py-8">
			<div className="relative overflow-hidden rounded-2xl border bg-linear-to-br from-cyan-50 via-white to-blue-50 p-6 dark:from-cyan-950/40 dark:via-background dark:to-blue-950/40">
				<div className="absolute -right-10 -top-10 h-40 w-40 rounded-full bg-blue-200/30 blur-2xl" />
				<div className="absolute -bottom-10 -left-10 h-36 w-36 rounded-full bg-cyan-200/30 blur-2xl" />
				<div className="relative space-y-3">
					<h1 className="text-2xl font-semibold tracking-tight sm:text-3xl">Consult With Our Specialists</h1>
					<p className="max-w-3xl text-sm text-muted-foreground sm:text-base">
						Discover trusted doctors, compare experience and fees, and open detailed profiles to find the right
						specialist.
					</p>
				</div>
			</div>

			{/* ---------- Filters ---------- */}
			<div className="space-y-4 rounded-2xl border bg-card p-4 shadow-sm sm:p-5">
				<div className="flex flex-wrap items-center gap-2">
					<DataTableSearch
						value={table.searchTerm}
						onChange={table.setSearchTerm}
						placeholder="Search name, designation, specialty..."
						debounceMs={500}
					/>

					<DataTableSelectFilter
						label="Gender"
						allLabel="All genders"
						value={table.getFilter("gender")}
						options={GENDER_OPTIONS}
						onChange={(value) => table.setFilters({ gender: value })}
					/>

					<DataTableRangeFilter
						label="Fee (৳)"
						min={table.getFilter(FEE_MIN_KEY)}
						max={table.getFilter(FEE_MAX_KEY)}
						onChange={({ min, max }) => table.setFilters({ [FEE_MIN_KEY]: min, [FEE_MAX_KEY]: max })}
					/>

					<div className="ml-auto flex items-center gap-2">
						<span className="text-sm text-muted-foreground">Sort</span>
						<DataTableSelectFilter
							label="Sort doctors"
							allLabel="Newest"
							value={sortValue}
							options={SORT_OPTIONS}
							onChange={handleSortChange}
						/>
					</div>
				</div>

				{specialties.length > 0 && (
					<div className="space-y-2">
						<p className="text-sm font-medium">Specialties</p>
						<div className="flex flex-wrap gap-2">
							{specialties.map((specialty) => {
								const isSelected = selectedSpecialties.includes(specialty.title);
								return (
									<Button
										key={specialty.id}
										type="button"
										size="sm"
										variant={isSelected ? "default" : "outline"}
										aria-pressed={isSelected}
										className="rounded-full"
										onClick={() => toggleSpecialty(specialty.title)}
									>
										{specialty.title}
									</Button>
								);
							})}
						</div>
					</div>
				)}

				{hasActiveFilters && (
					<div className="flex justify-end">
						<Button type="button" variant="ghost" size="sm" onClick={clearAll}>
							<X />
							Clear all
						</Button>
					</div>
				)}
			</div>

			{/* ---------- Results ---------- */}
			{isError && !doctorsRes ? (
				<p className="rounded-2xl border border-dashed p-10 text-center text-destructive">
					Couldn&apos;t load doctors. Please try again.
				</p>
			) : doctors.length === 0 ? (
				<div className="rounded-2xl border border-dashed p-10 text-center text-muted-foreground">
					No doctors found. Try changing the filters.
				</div>
			) : (
				<div
					aria-busy={isFetching}
					className={cn("grid gap-5 transition-opacity sm:grid-cols-2 lg:grid-cols-3", isFetching && "opacity-60")}
				>
					{doctors.map((doctor) => (
						<DoctorCard key={doctor.id} doctor={doctor} />
					))}
				</div>
			)}

			<div className="space-y-3">
				<DoctorsPagination
					page={meta?.page ?? 1}
					totalPages={meta?.totalPages ?? 1}
					disabled={isFetching}
					onPageChange={handlePageChange}
				/>
				{meta && (
					<p className="text-center text-sm text-muted-foreground">
						{meta.total} {meta.total === 1 ? "doctor" : "doctors"} found
					</p>
				)}
			</div>
		</section>
	);
}
```

### ভেঙে বোঝা

**১. URL → query string → useQuery**
- `useTableUrlState` URL থেকে page/search/sort/filter পড়ে, আর লেখে `history.pushState` দিয়ে (note 17)। তাই filter বদলালে **server component আবার চলে না**, শুধু `useSearchParams` আপডেট হয়।
- `buildQueryString` allowlist-এর key রাখে + default (`page=1`, `limit=12`) বসায় → নতুন `queryKey` → `useQuery` নিজেই নতুন data আনে।

**২. `placeholderData: keepPreviousData`**
- নতুন filter-এর data আসা পর্যন্ত পুরনো card-গুলোই থাকে (খালি হয়ে লাফায় না)।
- তখন `isFetching = true` → grid `opacity-60` আর `aria-busy`, pagination button disabled।

**৩. শেষ page ঠিক করা (`useEffect`)**
- ধরো page 5-এ আছো, তারপর filter দিলে মোট page হলো 2 → backend `page: 5, totalPages: 2` দেয় → `setPage(2)`।
- `totalPages > 0` চেক: কোনো result না থাকলে (0 page) loop হয় না।
- `setPage`-কে আলাদা করে destructure করা হয়েছে যাতে dependency-তে পুরো `table` object (প্রতি render-এ নতুন) না যায়।

**৪. Specialty chip (একাধিক বাছাই)**
- URL-এ একই key একাধিকবার: `?specialties.specialty.title=Cardiology&specialties.specialty.title=Neurology`।
- `getFilterList` = `searchParams.getAll(key)`; toggle = থাকলে বাদ, না থাকলে যোগ → `setFilters` array পেলে প্রতিটা `append` করে।
- `aria-pressed` → screen reader বোঝে button টা on/off toggle।

**৫. Sort dropdown ↔ `SortingState`**
- hook-এর sort state TanStack Table-এর `SortingState` (`[{ id, desc }]`)।
- পড়ার সময়: `{ id: "appointmentFee", desc: false }` → `"appointmentFee:asc"` (dropdown-এর value)।
- লেখার সময়: `"experience:desc".split(":")` → `setSorting([{ id: "experience", desc: true }])` → URL-এ `sortBy` + `sortOrder`।
- "Newest" (`allLabel`) বাছলে value `undefined` → `setSorting([])` → backend-এর default sort।

**৬. Clear all**
- সব filter key + `sortBy`/`sortOrder` মুছে দেয়; `page`-ও মুছে যায় (`resetPage` default `true`)। কোনো filter/sort না থাকলে button দেখায় না।

**৭. Page বদলালে উপরে scroll**
- `handlePageChange` → `setPage` + `window.scrollTo({ top: 0, behavior: "smooth" })`।

**৮. Result-এর ৩ অবস্থা**

| অবস্থা | কী দেখায় |
|---|---|
| error এবং কোনো data নেই | লাল "Couldn't load doctors" |
| data আছে কিন্তু খালি | "No doctors found. Try changing the filters." |
| data আছে | card grid (fetch চলার সময় ঝাপসা) |

---

## ধাপ ৪: `src/app/(commonLayout)/consultation/page.tsx` (পুরোটা বদলাও)

```tsx
import {
	CONSULTATION_DEFAULTS,
	CONSULTATION_QUERY_KEYS,
} from "@/components/modules/Consultation/consultation.config";
import DoctorsList from "@/components/modules/Consultation/DoctorsList";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { buildQueryString } from "@/lib/buildQueryString";
import { getQueryClient } from "@/lib/getQueryClient";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import { Suspense } from "react";

type SearchParams = Record<string, string | string[] | undefined>;

// server শুধু প্রথমবার data আনে (loading ছাড়া প্রথম render)।
// এরপর filter বদলালে client-এর useQuery নিজেই আনে, server আবার চলে না।
async function DoctorsSection({ searchParams }: { searchParams: Promise<SearchParams> }) {
	const queryString = buildQueryString(await searchParams, CONSULTATION_QUERY_KEYS, CONSULTATION_DEFAULTS);
	const queryClient = getQueryClient();

	await Promise.all([
		queryClient.query(doctorsQueryOptions(queryString)).catch(noop),
		queryClient.query(specialtiesQueryOptions()).catch(noop),
	]);

	return (
		<HydrationBoundary state={dehydrate(queryClient)}>
			<DoctorsList />
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

- **`searchParams` একটা Promise** (Next.js 15+)। `await` টা `DoctorsSection`-এর ভেতরে, তাই page-এর বাকি অংশ সাথে সাথে আসে আর এই অংশে `Suspense` fallback (spinner) দেখায়।
- **`Suspense` দরকার কেন:** `DoctorsList` `useSearchParams()` ব্যবহার করে — Next.js-এ এটা `Suspense`-এর ভেতরে থাকা লাগে।
- **একই query string:** server `await searchParams` (object) থেকে, client `useSearchParams()` (`URLSearchParams`) থেকে — `buildQueryString` দুটোকেই এক রূপে আনে → একই `queryKey` → hydrate হওয়া data সরাসরি কাজে লাগে, client আবার fetch করে না।
- **doctors আর specialties একসাথে** (`Promise.all`) — একটার জন্য আরেকটা অপেক্ষা করে না।
- **`queryClient.query(...).catch(noop)`:** fetch করে cache-এ রাখে; error হলে page crash করে না (`noop` = কিছু না করা function)। তখন client-এর `useQuery` নিজে আবার চেষ্টা করে, আর না পারলে "Couldn't load doctors" দেখায়।
- **`HydrationBoundary` + `dehydrate`:** server-এর cache client-এ পাঠানো (note 06)।
- **`PageProps<"/consultation">`:** Next.js-এর auto-generated type (route অনুযায়ী `searchParams`-এর type)।

---

## 🧪 Test (browser-এ)

| কী করবে | URL / ফল |
|---|---|
| page খোলো | `/consultation` → backend-এ `GET /api/v1/doctors?page=1&limit=12` |
| "card" লিখে search | ৫০০ms পর `?searchTerm=card` (history-তে নতুন entry হয় না) |
| Gender: Female | `?gender=FEMALE`, page 1-এ ফেরে |
| Fee 500–1000 | `?appointmentFee[gte]=500&appointmentFee[lte]=1000` |
| ২টা specialty chip | একই key দুবার URL-এ |
| Sort: Fee low to high | `?sortBy=appointmentFee&sortOrder=asc` |
| Next | `?page=2`, উপরে scroll হয় |
| Browser Back | আগের filter ফিরে আসে (cache থেকে, তাই তাৎক্ষণিক) |
| Clear all | শুধু `/consultation` |
| Network tab | filter বদলালে শুধু `/doctors?...` request; page-এর RSC request যায় না |

---

## ✅ নতুন project-এ implement করার checklist

- [ ] config file: allowlist key + default (`page`, `limit`) + sort options
- [ ] server page: `await searchParams` → `buildQueryString` → prefetch → `HydrationBoundary`
- [ ] `useSearchParams` ব্যবহার করা component-কে `Suspense`-এ রাখো
- [ ] client: একই `buildQueryString` → `useQuery` + `keepPreviousData`
- [ ] filter/sort বদলালে page 1-এ ফেরা; বেশি page হলে শেষ page-এ ফেরা
- [ ] empty / error / loading (ঝাপসা) তিনটা অবস্থা

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Error হলে "Try again" button** — এখন শুধু লেখা দেখায়, user-কে page reload করতে হয়। `useQuery` থেকে `refetch` নিয়ে button দাও:
   ```tsx
   const { data: doctorsRes, isFetching, isError, refetch } = useQuery({ ... });
   // ...
   <div className="space-y-3 rounded-2xl border border-dashed p-10 text-center">
   	<p className="text-destructive">Couldn&apos;t load doctors.</p>
   	<Button variant="outline" onClick={() => refetch()}>Try again</Button>
   </div>
   ```

2. **Specialty chip-গুলোকে group হিসেবে চিহ্নিত করা** — screen reader বুঝবে এগুলো একসাথে একটা filter:
   ```tsx
   <div role="group" aria-label="Filter by specialty" className="flex flex-wrap gap-2">
   ```

3. **Filter বদলানোর সময় count-ও ঝাপসা করা** — `keepPreviousData`-এর কারণে নতুন result আসা পর্যন্ত "X doctors found" পুরনো সংখ্যা দেখায়। grid-এর মতো `isFetching && "opacity-60"` দিলে বোঝা যায় এটা আপডেট হচ্ছে।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **Admin-এর key বদলালে এটাও বদলাবে** — `consultation.config.ts` admin-এর `doctorsTable.config`-থেকে import করে। ভালো (এক জায়গায় নিয়ম), কিন্তু কখনো public page-এ আলাদা filter লাগলে (যেমন শুধু active doctor) এখানে নিজস্ব key যোগ করতে হবে — admin file-এ না।

5. **Sort field backend-এ থাকতে হবে** — `SORT_OPTIONS`-এ নতুন field দিলে (যেমন `averageRating`) সেটা backend-এর sortable fields list-এ আছে কিনা দেখে নিও; না থাকলে backend হয় ignore করবে নয়তো error দেবে।

6. **অনেক page হলে** — শুধু Previous/Next-এ ২০ নম্বর page-এ যেতে কষ্ট। doctor বাড়লে shadcn-এর `Pagination` component দিয়ে page number (১ … ৪ ৫ ৬ … ২০) দেখানো যায়।

---

## 🔜 পরে যা যোগ হবে

- [ ] Doctor details page থেকে appointment booking
- [ ] Doctor card-এ available schedule দেখানো
