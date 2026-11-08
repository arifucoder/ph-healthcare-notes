# 16 - TanStack Table: Admin Doctors List (Data Layer + Reusable DataTable)

> Module: 48-7 (Introduction To Tanstack Table), 48-8 (Setting Up Server Action And Interface To Get All Doctors)

> Admin-এর `/admin/dashboard/doctors-management` page-এ সব doctor-এর একটা **table**। এই নোটে: data আনা (prefetch), আর একটা **reusable `DataTable`** (TanStack Table v9 + shadcn `table`) যেটা পরে appointments, patients সব table-এ ব্যবহার হবে।
>
> এই topic বড়, তাই ৩ ভাগে:
> - **16 (এই নোট):** data layer + DataTable + columns
> - **17:** search, sort, pagination, filter (URL state)
> - **18:** View dialog আর Delete

## 🧭 এক নজরে

```
/admin/dashboard/doctors-management?page=1&searchTerm=...
  → page.tsx (Server)
       → buildQueryString(searchParams, allowlist, defaults) → "page=1&limit=10"
       → prefetch: doctors + specialties (একসাথে)
       → <HydrationBoundary> → <DoctorsTable /> ("use client")
            → useSearchParams → একই buildQueryString → একই cache key → data cache থেকে
            → <DataTable data columns getRowId ... />
                 → useTable (TanStack) → header + row
                 → প্রতিটা cell: doctorColumns (UserInfoCell, Badge, DateCell, formatCurrency)
```

### TanStack Table কী?
👉 https://tanstack.com/table/latest/docs/overview

- **Headless** table library: নিজে কোনো HTML/CSS দেয় না। শুধু হিসাব করে দেয় — কোন column, কোন row, কোন cell-এ কী, sort কোন দিকে।
- আমরা সেই হিসাব নিয়ে shadcn-এর `<Table>`, `<TableRow>` দিয়ে UI বানাই। তাই দেখতে যেমন খুশি, logic লিখতে হয় না।
- Project-এ **v9** (`useTable`, `tableFeatures`, `table.FlexRender`)। ইন্টারনেটের বেশিরভাগ উদাহরণ v8-এর (`useReactTable`, `getCoreRowModel`, `flexRender`) — নাম আলাদা, মিলিয়ে নিও।

---

## ধাপ ০: Install

```shell
pnpm add @tanstack/react-table
pnpm dlx shadcn@latest add table
```

- `@tanstack/react-table` → table-এর logic।
- shadcn `table` → `<Table>`, `<TableHeader>`, `<TableRow>`, `<TableHead>`, `<TableCell>` (শুধু style করা HTML table)।

---

## ধাপ ১: `src/types/doctor.types.ts` — `IDoctor` আপডেট

```ts
// backend prisma enum Gender-এর সাথে মিল (OTHER নেই)
export type Gender = "MALE" | "FEMALE";

// backend doctorSelectableFields + specialties include-এর সাথে মিল রাখো
export interface IDoctor {
	id: string;
	name: string;
	email: string;
	profilePhoto?: string | null;
	gender: Gender;
	address?: string | null;
	appointmentFee: number;
	qualification: string;
	designation: string;
	currentWorkingPlace: string;
	experience: number;
	averageRating: number;
	createdAt: string;
	specialties: Array<{
		specialty: { id: string; title: string };
	}>;
}
```

- Backend-এর `getAllDoctors` পুরো doctor + `include` (`user`, `specialties.specialty`) পাঠায়। (`doctorSelectableFields` শুধু `?fields=` দিলে কাজ করে, আমরা দিই না।)
- `Gender` → backend-এর Prisma enum-এ শুধু `MALE`, `FEMALE` (`OTHER` নেই)।
- `?` + `| null` → field DB-তে খালি থাকতে পারে (backend `null` পাঠায়)।
- `createdAt: string` → JSON-এ Date আসে না, ISO string আসে (note 14)।
- `specialties` → doctor আর specialty-র মাঝে একটা join table, তাই `specialties[].specialty.title`।

> নোট 18-এ এখানে আরো ২টা field (`contactNumber`, `registrationNumber`) যোগ হবে।

---

## ধাপ ২: `src/lib/buildQueryString.ts` (নতুন) — reusable

এটা তোমার consultation page-এর buildQueryString-কে সব page-এর জন্য reusable বানানো version। Server আর client দুজনেই এটা ব্যবহার করবে, তাই একই URL থেকে হুবহু একই query string, আর একই cache key তৈরি হবে।

```ts
type SearchParamsInput = URLSearchParams | Record<string, string | string[] | undefined>;

// server-এ searchParams আসে object হিসেবে, client-এ URLSearchParams হিসেবে; দুটোকেই এক রূপে আনা
const toEntries = (input: SearchParamsInput): Array<[string, string]> => {
	if (input instanceof URLSearchParams) return [...input.entries()];

	return Object.entries(input).flatMap(([key, raw]) => {
		if (raw === undefined) return [];
		return (Array.isArray(raw) ? raw : [raw]).map((value): [string, string] => [key, value]);
	});
};

/**
 * শুধু allowlist-এর key backend-এ যায় (বাকি সব বাদ)।
 * default (যেমন page=1) বসানো থাকে, যাতে "/doctors" আর "/doctors?page=1" একই cache key পায়।
 */
export const buildQueryString = (
	input: SearchParamsInput,
	allowedKeys: ReadonlySet<string>,
	defaults: Record<string, string> = {},
): string => {
	const result = new URLSearchParams();

	for (const [key, value] of toEntries(input)) {
		const trimmed = value.trim();
		if (allowedKeys.has(key) && trimmed) result.append(key, trimmed);
	}

	for (const [key, value] of Object.entries(defaults)) {
		if (!result.has(key)) result.set(key, value);
	}

	return result.toString();
};
```

### কেন এত জরুরি — cache key ⭐

```
Server (page.tsx):  searchParams (object)        → buildQueryString → "page=1&limit=10" → key ["doctors", "page=1&limit=10"]
Client (table):     useSearchParams (URLSearchParams) → buildQueryString → "page=1&limit=10" → key ["doctors", "page=1&limit=10"] ✅ একই
```
- Key একটুও আলাদা হলে (`"limit=10&page=1"` বনাম `"page=1&limit=10"`, বা default বসানো/না বসানো) client prefetch-এর data খুঁজে পাবে না → **আবার fetch** করবে।
- একই function, একই নিয়ম → দুই জায়গায় হুবহু একই string।

### ভেঙে বোঝা

| অংশ | কাজ |
|---|---|
| `SearchParamsInput` | server-এ `searchParams` আসে **object** (`{ page: "1", gender: ["MALE"] }`), client-এ **`URLSearchParams`** — দুটোই চলবে |
| `toEntries` | দুই রকমকেই `[key, value]` জোড়ার list বানায়। Array মান (একই key একাধিকবার, যেমন একাধিক specialty) আলাদা আলাদা জোড়া হয় |
| `flatMap` | প্রতিটা key থেকে ০, ১ বা অনেক জোড়া → সব একটা list-এ |
| `allowedKeys.has(key)` | **allowlist**: শুধু চেনা key backend-এ যায়। কেউ URL-এ `?isDeleted=true` বা আজেবাজে কিছু লিখলে বাদ |
| `trimmed` খালি হলে বাদ | `?searchTerm=` (ফাঁকা) পাঠানো হয় না |
| `defaults` | URL-এ না থাকলে বসায় → `/doctors-management` আর `?page=1` একই key পায় |

---

## ধাপ ৩: `doctorsTable.config.ts` (নতুন)

`src/components/modules/Admin/DoctorsManagement/doctorsTable.config.ts`

```ts
// admin doctor table-এর URL param নিয়ম এক জায়গায়: page (server) আর table (client) দুজনেই ব্যবহার করবে
export const DOCTOR_TABLE_DEFAULTS = { page: "1", limit: "10" } as const;

export const SPECIALTY_FILTER_KEY = "specialties.specialty.title";

// backend doctor.constant.ts-এর searchable/filterable/sortable field অনুযায়ী
export const DOCTOR_TABLE_QUERY_KEYS: ReadonlySet<string> = new Set([
	"page",
	"limit",
	"sortBy",
	"sortOrder",
	"searchTerm",
	"gender",
	SPECIALTY_FILTER_KEY,
	"appointmentFee[gte]",
	"appointmentFee[lte]",
]);
```

- **কেন আলাদা file:** server page আর client table **দুজনেই** এই নিয়ম লাগে। এক জায়গায় রাখলে দুই জায়গায় আলাদা হওয়ার সুযোগ নেই।
- `.ts` (`.tsx` না) আর `"use client"` নেই → server থেকেও import করা যায়।
- Key-গুলো backend-এর `doctor.constant.ts` থেকে: searchable → `searchTerm`, filterable → `gender`, `specialties.specialty.title`, `appointmentFee`, sortable → `sortBy`।
- `appointmentFee[gte]` / `[lte]` → backend-এর QueryBuilder-এর range filter (≥ আর ≤)।
- `SPECIALTY_FILTER_KEY` আলাদা constant → লম্বা string বারবার লিখতে হয় না, typo হয় না।

> নোট 17-এ এই file-এ filter-এর জন্য আরো constant যোগ হবে।

---

## ধাপ ৪: Page — `src/app/(dashboardLayout)/admin/dashboard/doctors-management/page.tsx`

```tsx
import DoctorsTable from "@/components/modules/Admin/DoctorsManagement/DoctorsTable";
import {
	DOCTOR_TABLE_DEFAULTS,
	DOCTOR_TABLE_QUERY_KEYS,
} from "@/components/modules/Admin/DoctorsManagement/doctorsTable.config";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { buildQueryString } from "@/lib/buildQueryString";
import { getQueryClient } from "@/lib/getQueryClient";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import { Suspense } from "react";

type SearchParams = Record<string, string | string[] | undefined>;

async function DoctorsSection({ searchParams }: { searchParams: Promise<SearchParams> }) {
	const queryString = buildQueryString(await searchParams, DOCTOR_TABLE_QUERY_KEYS, DOCTOR_TABLE_DEFAULTS);
	const queryClient = getQueryClient();

	// দুটো একসাথে (teacher-এর code-এ একটার পর একটা await হচ্ছিল)
	await Promise.all([
		queryClient.query(doctorsQueryOptions(queryString)).catch(noop),
		queryClient.query(specialtiesQueryOptions()).catch(noop),
	]);

	return (
		<HydrationBoundary state={dehydrate(queryClient)}>
			<DoctorsTable />
		</HydrationBoundary>
	);
}

export default function DoctorsManagementPage({ searchParams }: PageProps<"/admin/dashboard/doctors-management">) {
	return (
		<div className="space-y-6">
			<div>
				<h1 className="text-2xl font-semibold tracking-tight">Doctors Management</h1>
				<p className="text-sm text-muted-foreground">View, add, edit and remove doctors</p>
			</div>

			<Suspense fallback={<LoadingSpinner text="Loading doctors..." />}>
				<DoctorsSection searchParams={searchParams} />
			</Suspense>
		</div>
	);
}
```

`PageProps<"/admin/dashboard/doctors-management">` type-টা Next নিজে বানায়, তবে ওই folder-এ page file তৈরি হওয়ার পরে `pnpm dev` চালু থাকলে। তাই প্রথমে লাল দাগ দেখালে dev server একবার restart করো।

- Pattern হুবহু admin dashboard (note 15) আর consultation (note 06)-এর মতো: heading সাথে সাথে, `<Suspense>`-এ async section।
- `searchParams` **Promise** (Next.js 15+), তাই `await searchParams`। আর এটা `DoctorsSection`-এর ভিতরে await করা → page-এর heading আটকে থাকে না।
- **`Promise.all`** → doctors আর specialties **একসাথে** আনা (teacher-এর code-এ একটার পর একটা)। Specialties লাগবে filter-এর option-এ (note 17)।
- `.catch(noop)` → fail হলেও page ভাঙে না, client `useQuery` সামলায়।

---

## ধাপ ৫: `DoctorsTable.tsx` — আপাতত test (data আসছে কিনা)

ধাপ ৯-এ এটা আসল table হয়ে যাবে। এখন শুধু দেখবো data আসছে কিনা।

```tsx
"use client";

import { buildQueryString } from "@/lib/buildQueryString";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { useQuery } from "@tanstack/react-query";
import { useSearchParams } from "next/navigation";
import { DOCTOR_TABLE_DEFAULTS, DOCTOR_TABLE_QUERY_KEYS } from "./doctorsTable.config";

export default function DoctorsTable() {
	const searchParams = useSearchParams();

	// server-এর মতো একই নিয়মে বানানো, তাই একই cache key, আর প্রথমবার আবার fetch হয় না
	const queryString = buildQueryString(searchParams, DOCTOR_TABLE_QUERY_KEYS, DOCTOR_TABLE_DEFAULTS);

	const { data: response, isFetching, isError } = useQuery(doctorsQueryOptions(queryString));

	if (isError) return <p className="text-sm text-destructive">Failed to load doctors.</p>;

	return (
		<div className="space-y-2 text-sm">
			<p className="text-muted-foreground">
				Query: <code>{queryString}</code> · Total: {response?.meta?.total ?? 0} {isFetching && "· refreshing..."}
			</p>
			<ul className="list-inside list-disc">
				{response?.data.map((doctor) => (
					<li key={doctor.id}>
						{doctor.name}: {doctor.specialties.map((s) => s.specialty.title).join(", ") || "No specialty"}
					</li>
				))}
			</ul>
		</div>
	);
}
```

- `useSearchParams()` → client-এ URL-এর query (`URLSearchParams`)।
- একই `buildQueryString` → server-এর মতো একই key → প্রথমবার আবার fetch হয় না।
- `response?.meta?.total` → backend paginated list-এর সাথে `meta: { page, limit, total, totalPages }` পাঠায় (note 04-এর `ApiResponse`)।
- **Test:** page খোলো → `Query: limit=10&page=1 · Total: N` আর doctor-দের নাম + specialty দেখা গেলে data layer ঠিক। Network tab-এ browser থেকে আলাদা doctor request থাকবে না।

> ✏️ `useSearchParams()` ব্যবহার করা client component-কে Next.js একটা `<Suspense>`-এর ভিতরে চায় — page-এ আগে থেকেই `<Suspense>` আছে, তাই ঠিক আছে।

---

## ধাপ ৬: (ঐচ্ছিক) Consultation page-এ একই helper

`app/(commonLayout)/consultation/page.tsx`-এর নিজের লেখা `buildQueryString` function মুছে, এভাবে বদলাও:

```tsx
import { buildQueryString } from "@/lib/buildQueryString";

const ALLOWED_QUERY_KEYS = new Set(["page", "limit", "sortBy", "sortOrder", "searchTerm"]);

// DoctorsSection-এর ভেতরে:
const queryString = buildQueryString(await searchParams, ALLOWED_QUERY_KEYS, { page: "1", limit: "12" });
```

> Project-এ এটা করা হয়ে গেছে।

---

## ধাপ ৭: Reusable DataTable — কী কী বানাচ্ছি

```
components/shared/table/
├── dataTableFeatures.ts     ← v9 feature + column type
├── DataTable.tsx            ← শুধু table: header, row, loading, empty, sort icon, row action
└── cells/
    ├── UserInfoCell.tsx
    └── DateCell.tsx
lib/formatters.ts             ← টাকা আর সংখ্যার format (dashboard-ও ব্যবহার করতে পারবে)
modules/Admin/DoctorsManagement/
├── doctorsColumns.tsx
└── DoctorsTable.tsx          ← ধাপ 1-এর তালিকার বদলে আসল table
```

| File | কাজ | কে ব্যবহার করবে |
|---|---|---|
| `dataTableFeatures.ts` | TanStack v9-এর feature আর column-এর type | সব table |
| `DataTable.tsx` | শুধু table: header, row, loading, empty, sort icon, row action menu | সব table |
| `cells/UserInfoCell.tsx` | avatar + নাম + email | doctor, patient, admin table |
| `cells/DateCell.tsx` | তারিখ সুন্দর করে | সব table |
| `lib/formatters.ts` | টাকা, সংখ্যা, নামের আদ্যক্ষর | table, dashboard, dialog |
| `doctorsColumns.tsx` | **শুধু doctor-এর** column | doctor table |
| `DoctorsTable.tsx` | data আনা + DataTable জোড়া | doctor page |

**নিয়ম:** `shared/table/`-এ কোনো doctor-এর কথা নেই। Doctor-এর জিনিস শুধু `modules/Admin/DoctorsManagement/`-এ। তাই appointments table বানাতে শুধু নতুন columns + নতুন `XxxTable.tsx`।

---

## ধাপ ৮: Shared table অংশ

### ৮.১ `src/components/shared/table/dataTableFeatures.ts`

```ts
import { type ColumnDef, rowSortingFeature, tableFeatures } from "@tanstack/react-table";

// v9: যে feature লাগবে শুধু সেটাই যোগ হয় (bundle ছোট থাকে)।
// sorting/pagination backend করে (manual), তাই sortedRowModel বা paginatedRowModel লাগে না।
export const dataTableFeatures = tableFeatures({ rowSortingFeature });

export type DataTableFeatures = typeof dataTableFeatures;

// প্রতিটা table-এর column এই type দিয়ে লিখবে
export type DataTableColumnDef<TData> = ColumnDef<DataTableFeatures, TData>;
```

- **v9-এর নতুন নিয়ম:** কোন feature (sorting, filtering, pagination, row selection…) লাগবে, সেটা নিজে বলে দিতে হয়। যা লাগে না সেটা bundle-এ ঢোকে না।
- আমরা শুধু **`rowSortingFeature`** নিচ্ছি — header-এ sort-এর অবস্থা (↑↓) দেখানোর জন্য।
- **Sorting/pagination আসলে backend করে (manual)** — তাই v8-এর মতো `getSortedRowModel` / `getPaginationRowModel` লাগে না। Table শুধু দেখায়।
- `DataTableColumnDef<TData>` → প্রতিটা table-এর column এই type দিয়ে লিখলে feature-এর type মিলে থাকে (`ColumnDef<features, data>`)।

### ৮.২ `src/components/shared/table/DataTable.tsx`

```tsx
"use client";

import { Button } from "@/components/ui/button";
import {
	DropdownMenu,
	DropdownMenuContent,
	DropdownMenuItem,
	DropdownMenuSeparator,
	DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { Skeleton } from "@/components/ui/skeleton";
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table";
import { functionalUpdate, type SortDirection, type SortingState, useTable } from "@tanstack/react-table";
import { cn } from "cn";
import { ArrowDown, ArrowUp, ArrowUpDown, Eye, MoreHorizontal, Pencil, Trash2 } from "lucide-react";
import { useMemo } from "react";
import { type DataTableColumnDef, dataTableFeatures } from "./dataTableFeatures";

export interface DataTableRowActions<TData> {
	onView?: (row: TData) => void;
	onEdit?: (row: TData) => void;
	onDelete?: (row: TData) => void;
}

interface DataTableProps<TData> {
	data: TData[];
	columns: DataTableColumnDef<TData>[];
	getRowId: (row: TData) => string; // index-এর বদলে আসল id: delete-এর পর ভুল row-এ state যায় না
	isLoading?: boolean; // প্রথমবার, এখনো কোনো data নেই
	isFetching?: boolean; // পুরনো data দেখাচ্ছে, নতুন আসছে
	emptyMessage?: string;
	sorting?: {
		state: SortingState;
		onSortingChange: (state: SortingState) => void;
	};
	actions?: DataTableRowActions<TData>;
}

const SKELETON_ROW_COUNT = 5;

const SortIcon = ({ direction }: { direction: false | SortDirection }) => {
	if (direction === "asc") return <ArrowUp className="size-3.5" aria-hidden />;
	if (direction === "desc") return <ArrowDown className="size-3.5" aria-hidden />;
	return <ArrowUpDown className="size-3.5 opacity-50" aria-hidden />;
};

const toAriaSort = (direction: false | SortDirection) =>
	direction === "asc" ? "ascending" : direction === "desc" ? "descending" : undefined;

// row-এর শেষে "⋯" menu: যে action দেওয়া হয়েছে শুধু সেটাই দেখায়
const createActionsColumn = <TData,>(actions: DataTableRowActions<TData>): DataTableColumnDef<TData> => ({
	id: "actions",
	header: () => <span className="sr-only">Actions</span>,
	enableSorting: false,
	cell: ({ row }) => (
		<DropdownMenu>
			<DropdownMenuTrigger asChild>
				<Button variant="ghost" size="icon" aria-label="Open row actions">
					<MoreHorizontal />
				</Button>
			</DropdownMenuTrigger>
			<DropdownMenuContent align="end">
				{actions.onView && (
					<DropdownMenuItem onSelect={() => actions.onView?.(row.original)}>
						<Eye />
						View
					</DropdownMenuItem>
				)}
				{actions.onEdit && (
					<DropdownMenuItem onSelect={() => actions.onEdit?.(row.original)}>
						<Pencil />
						Edit
					</DropdownMenuItem>
				)}
				{actions.onDelete && (
					<>
						<DropdownMenuSeparator />
						<DropdownMenuItem variant="destructive" onSelect={() => actions.onDelete?.(row.original)}>
							<Trash2 />
							Delete
						</DropdownMenuItem>
					</>
				)}
			</DropdownMenuContent>
		</DropdownMenu>
	),
});

export default function DataTable<TData>({
	data,
	columns,
	getRowId,
	isLoading = false,
	isFetching = false,
	emptyMessage = "No results found.",
	sorting,
	actions,
}: DataTableProps<TData>) {
	const allColumns = useMemo(
		() => (actions ? [...columns, createActionsColumn(actions)] : columns),
		[columns, actions],
	);

	const table = useTable({
		features: dataTableFeatures,
		columns: allColumns,
		data,
		getRowId,
		manualSorting: true, // sort backend করে, table শুধু state দেখায়
		enableSorting: Boolean(sorting),
		state: { sorting: sorting?.state ?? [] },
		onSortingChange: (updater) => {
			if (sorting) sorting.onSortingChange(functionalUpdate(updater, sorting.state));
		},
	});

	const rows = table.getRowModel().rows;

	return (
		<div className="overflow-hidden rounded-lg border">
			<Table aria-busy={isLoading || isFetching}>
				<TableHeader>
					{table.getHeaderGroups().map((headerGroup) => (
						<TableRow key={headerGroup.id}>
							{headerGroup.headers.map((header) => {
								const direction = header.column.getIsSorted();

								return (
									<TableHead key={header.id} aria-sort={toAriaSort(direction)}>
										{header.isPlaceholder ? null : header.column.getCanSort() ? (
											<Button
												variant="ghost"
												size="sm"
												className="-ml-2.5"
												onClick={header.column.getToggleSortingHandler()}
											>
												<table.FlexRender header={header} />
												<SortIcon direction={direction} />
											</Button>
										) : (
											<table.FlexRender header={header} />
										)}
									</TableHead>
								);
							})}
						</TableRow>
					))}
				</TableHeader>

				{/* নতুন data আসার সময় পুরনোটা হালকা দেখায়, লাফ দেয় না */}
				<TableBody className={cn("transition-opacity", isFetching && !isLoading && "opacity-60")}>
					{isLoading ? (
						Array.from({ length: SKELETON_ROW_COUNT }, (_, rowIndex) => (
							<TableRow key={rowIndex}>
								{allColumns.map((_, cellIndex) => (
									<TableCell key={cellIndex}>
										<Skeleton className="h-5 w-full" />
									</TableCell>
								))}
							</TableRow>
						))
					) : rows.length === 0 ? (
						<TableRow>
							<TableCell colSpan={allColumns.length} className="h-24 text-center text-muted-foreground">
								{emptyMessage}
							</TableCell>
						</TableRow>
					) : (
						rows.map((row) => (
							<TableRow key={row.id}>
								{row.getAllCells().map((cell) => (
									<TableCell key={cell.id}>
										<table.FlexRender cell={cell} />
									</TableCell>
								))}
							</TableRow>
						))
					)}
				</TableBody>
			</Table>
		</div>
	);
}
```

#### Props

| Prop | কাজ |
|---|---|
| `data` | row-এর list (এক page-এর) |
| `columns` | কোন column কী দেখাবে |
| `getRowId` | প্রতিটা row-এর **আসল id**। Index দিলে delete-এর পর row সরে গেলে menu/state ভুল row-এ চলে যেত |
| `isLoading` | প্রথমবার, এখনো কোনো data নেই → skeleton row |
| `isFetching` | পুরনো data দেখাচ্ছে, নতুন আসছে → হালকা (opacity-60) |
| `emptyMessage` | কোনো row না থাকলে |
| `sorting?` | `{ state, onSortingChange }` — দিলে sort চালু, না দিলে বন্ধ (note 17) |
| `actions?` | `{ onView, onEdit, onDelete }` — দিলে শেষে "⋯" menu (note 18) |

#### Generic `<TData>`
`DataTable<TData>` → যেকোনো ধরনের data (`IDoctor`, পরে `IAppointment`) নিতে পারে, আর TypeScript প্রতিটা ক্ষেত্রে ঠিক type বোঝে। `.tsx` file-এ arrow function-এ `<TData,>` (কমা সহ) লিখতে হয়, নাহলে TypeScript এটাকে JSX tag ভাবে।

#### `useTable(...)`

| Option | কাজ |
|---|---|
| `features: dataTableFeatures` | ধাপ ৮.১-এর feature |
| `columns`, `data`, `getRowId` | উপরে |
| `manualSorting: true` | table নিজে data sort করবে না, শুধু state দেখাবে — sort backend করে |
| `enableSorting: Boolean(sorting)` | `sorting` prop না দিলে কোনো column-এ sort button নেই |
| `state: { sorting }` | sort-এর অবস্থা **বাইরে থেকে** (URL থেকে, note 17) — controlled |
| `onSortingChange` | user header চাপলে নতুন state বাইরে পাঠানো |

**`functionalUpdate(updater, sorting.state)`** → TanStack কখনো সরাসরি নতুন মান দেয়, কখনো একটা function (`old => new`)। `functionalUpdate` দুই ক্ষেত্রেই আসল নতুন মান বের করে দেয়।

#### Header render
- `table.getHeaderGroups()` → header-এর সারি। `header.isPlaceholder` → group header-এর ফাঁকা ঘর (এখানে নেই, তবু নিয়মমতো)।
- `header.column.getCanSort()` → sort করা যায় এমন column-এ header-টা একটা `Button` + `SortIcon` (↑ asc, ↓ desc, ↕ কিছু না)।
- `getToggleSortingHandler()` → চাপলে: কিছু না → asc → desc → কিছু না।
- `<table.FlexRender header={header} />` → column-এর `header` (string বা function) render করে (v8-এর `flexRender`)।
- `aria-sort` → screen reader বলে "ascending/descending"।

#### Body — ৩ অবস্থা
```
isLoading           → ৫টা skeleton row (প্রতিটা column-এ একটা ধূসর লাইন)
rows.length === 0   → একটা row, সব column জুড়ে (colSpan) emptyMessage
বাকি                → আসল row → row.getAllCells() → <table.FlexRender cell={cell} />
```
- `isFetching && !isLoading` → `opacity-60` + `transition-opacity` → page বদলালে table ফাঁকা হয়ে লাফায় না, হালকা হয়ে নতুন data-র অপেক্ষা।
- `aria-busy` → screen reader বোঝে data load হচ্ছে।

#### `createActionsColumn` — "⋯" menu
- `actions` দেওয়া থাকলে column list-এর শেষে একটা extra column যোগ হয় (`useMemo` দিয়ে, যাতে প্রতি render-এ নতুন না হয়)।
- `header` → `sr-only` "Actions" (চোখে দেখা যায় না, screen reader পড়ে)।
- যে action দেওয়া হয়েছে শুধু সেই item দেখায় (`actions.onView && ...`)। Delete আলাদা separator-এর পরে, লাল।
- `row.original` → ওই row-এর আসল data (`IDoctor`)।
- বিস্তারিত ব্যবহার note 18-এ।

---

## ধাপ ৯: Formatter আর cell

### ৯.১ `src/lib/formatters.ts`

```ts
// locale fixed: server আর browser-এ একই output (hydration mismatch হয় না)
const currencyFormatter = new Intl.NumberFormat("en-US", {
	style: "currency",
	currency: "BDT",
	currencyDisplay: "narrowSymbol", // "৳1,500"
	maximumFractionDigits: 0,
});

const numberFormatter = new Intl.NumberFormat("en-US");

export const formatCurrency = (amount: number) => currencyFormatter.format(amount);
export const formatNumber = (value: number) => numberFormatter.format(value);

// "Dr. John Doe" → "JD"
export const getInitials = (name: string) =>
	name
		.replace(/^dr\.?\s*/i, "")
		.split(/\s+/)
		.filter(Boolean)
		.slice(0, 2)
		.map((word) => word[0]?.toUpperCase())
		.join("");
```

পরে `AdminDashboardContent`-এর নিজের `currencyFormatter` মুছে এই `formatCurrency` ব্যবহার করতে পারো। এটা ঐচ্ছিক।

- Locale fixed `"en-US"` → hydration mismatch হয় না (note 12)।
- `getInitials("Dr. John Doe")` → `"JD"`:

| ধাপ | ফল |
|---|---|
| `.replace(/^dr\.?\s*/i, "")` | শুরুর `Dr.` / `dr` / `DR ` বাদ → `"John Doe"` |
| `.split(/\s+/)` | এক বা একাধিক space দিয়ে ভাগ → `["John", "Doe"]` |
| `.filter(Boolean)` | ফাঁকা অংশ বাদ |
| `.slice(0, 2)` | প্রথম ২টা শব্দ |
| `.map((word) => word[0]?.toUpperCase())` | প্রথম অক্ষর বড় হাতের (`?.` কারণ `noUncheckedIndexedAccess`) |
| `.join("")` | `"JD"` |

### ৯.২ `src/components/shared/table/cells/UserInfoCell.tsx`

```tsx
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { getInitials } from "@/lib/formatters";

interface UserInfoCellProps {
	name: string;
	email: string;
	image?: string | null;
}

export default function UserInfoCell({ name, email, image }: UserInfoCellProps) {
	return (
		<div className="flex min-w-0 items-center gap-3">
			<Avatar>
				{/* alt="": নাম পাশেই লেখা, তাই screen reader-কে দুবার পড়াই না */}
				{image && <AvatarImage src={image} alt="" />}
				<AvatarFallback className="bg-primary/10 text-xs font-semibold text-primary">{getInitials(name)}</AvatarFallback>
			</Avatar>

			<div className="min-w-0">
				<p className="truncate font-medium">{name}</p>
				<p className="truncate text-xs text-muted-foreground">{email}</p>
			</div>
		</div>
	);
}
```

- ছবি থাকলে `AvatarImage`, না থাকলে (বা load না হলে) `AvatarFallback`-এ আদ্যক্ষর।
- `alt=""` → নাম পাশেই লেখা, তাই screen reader-কে দুবার পড়াই না।
- `min-w-0` + `truncate` → লম্বা নাম/email table ভাঙে না।
- `"use client"` নেই → hook নেই, client table-এর ভিতরে client হিসেবেই চলে।

### ৯.৩ `src/components/shared/table/cells/DateCell.tsx`

```tsx
// timeZone fixed: server (UTC) আর browser (ঢাকা)-এ একই তারিখ দেখায়
const dateFormatter = new Intl.DateTimeFormat("en-GB", {
	day: "2-digit",
	month: "short",
	year: "numeric",
	timeZone: "Asia/Dhaka",
});

export default function DateCell({ value }: { value: string | null | undefined }) {
	const date = value ? new Date(value) : null;

	// ভুল তারিখে Intl RangeError throw করে, তাই আগেই check
	if (!date || Number.isNaN(date.getTime())) {
		return <span className="text-muted-foreground">—</span>;
	}

	return (
		<time dateTime={value ?? undefined} className="whitespace-nowrap">
			{dateFormatter.format(date)}
		</time>
	);
}
```

- `en-GB` + `day: "2-digit", month: "short", year: "numeric"` → `"08 Nov 2026"`।
- **`timeZone: "Asia/Dhaka"`** → server (সাধারণত UTC) আর browser-এ একই তারিখ। নাহলে রাত ১২টার কাছাকাছি সময়ে server এক তারিখ, browser আরেক → hydration mismatch।
  - Bar chart-এ (note 14) `UTC` কারণ সেখানে backend মাসের শুরু UTC-তে দেয়। এখানে আসল মুহূর্ত (`createdAt`), তাই ব্যবহারকারীর দেশের (ঢাকা) তারিখ দেখানোই ঠিক।
- ভুল/ফাঁকা তারিখে `—` (note 14-এর মতো: ভাঙা Date-এ `Intl` RangeError throw করে)।
- `<time dateTime={...}>` → HTML-এর সঠিক tag, machine-readable আসল মান।
- `whitespace-nowrap` → তারিখ দুই লাইনে ভাঙে না।

---

## ধাপ ১০: `doctorsColumns.tsx` — doctor-এর column

`src/components/modules/Admin/DoctorsManagement/doctorsColumns.tsx`

```tsx
import DateCell from "@/components/shared/table/cells/DateCell";
import UserInfoCell from "@/components/shared/table/cells/UserInfoCell";
import type { DataTableColumnDef } from "@/components/shared/table/dataTableFeatures";
import { Badge } from "@/components/ui/badge";
import { formatCurrency } from "@/lib/formatters";
import type { IDoctor } from "@/types/doctor.types";
import { Star } from "lucide-react";

// ⚠️ sortable column-এর accessorKey = backend-এর sortBy field
// (doctorSortableFields: createdAt, name, experience, appointmentFee, averageRating)
export const doctorColumns: DataTableColumnDef<IDoctor>[] = [
	{
		accessorKey: "name",
		header: "Doctor",
		cell: ({ row }) => (
			<UserInfoCell name={row.original.name} email={row.original.email} image={row.original.profilePhoto} />
		),
	},
	{
		id: "specialties",
		header: "Specialties",
		enableSorting: false,
		cell: ({ row }) => {
			const { specialties } = row.original;

			if (specialties.length === 0) {
				return <span className="text-xs text-muted-foreground">No specialties</span>;
			}

			return (
				<div className="flex max-w-60 flex-wrap gap-1">
					{specialties.map(({ specialty }) => (
						<Badge key={specialty.id} variant="secondary">
							{specialty.title}
						</Badge>
					))}
				</div>
			);
		},
	},
	{
		id: "designation",
		header: "Designation",
		enableSorting: false, // backend-এ sortable না
		cell: ({ row }) => (
			<div className="min-w-0">
				<p className="truncate">{row.original.designation}</p>
				<p className="truncate text-xs text-muted-foreground">{row.original.currentWorkingPlace}</p>
			</div>
		),
	},
	{
		accessorKey: "experience",
		header: "Experience",
		cell: ({ row }) => <span className="tabular-nums">{row.original.experience} yrs</span>,
	},
	{
		accessorKey: "appointmentFee",
		header: "Fee",
		cell: ({ row }) => <span className="tabular-nums">{formatCurrency(row.original.appointmentFee)}</span>,
	},
	{
		accessorKey: "averageRating",
		header: "Rating",
		cell: ({ row }) => (
			<span className="inline-flex items-center gap-1 tabular-nums">
				<Star className="size-3.5 fill-amber-400 text-amber-400" aria-hidden />
				{row.original.averageRating.toFixed(1)}
			</span>
		),
	},
	{
		accessorKey: "createdAt",
		header: "Joined",
		cell: ({ row }) => <DateCell value={row.original.createdAt} />,
	},
];
```

### দুই রকম column

| ধরন | উদাহরণ | কখন |
|---|---|---|
| `accessorKey: "name"` | Doctor, Experience, Fee, Rating, Joined | data-র একটা field-এর সাথে বাঁধা। **Sort করা যায়** |
| `id: "specialties"` + `enableSorting: false` | Specialties, Designation | কয়েকটা field মিলিয়ে বা হিসাব করে দেখানো, backend-এ sort হয় না |

⚠️ **sortable column-এর `accessorKey` = backend-এর `sortBy` field।** User "Fee" header চাপলে URL-এ `sortBy=appointmentFee` যায় (note 17) — backend-এর `doctorSortableFields`-এ (`createdAt`, `name`, `experience`, `appointmentFee`, `averageRating`) না থাকলে কাজ করবে না।

### প্রতিটা column

| Column | Cell |
|---|---|
| Doctor | `UserInfoCell` (ছবি + নাম + email) |
| Specialties | প্রতিটা `Badge`, `max-w-60 flex-wrap` → অনেক হলে পরের লাইনে। নেই হলে "No specialties" |
| Designation | পদবি + নিচে হালকা করে কর্মস্থল |
| Experience | `5 yrs` |
| Fee | `formatCurrency` → `৳1,500` |
| Rating | হলুদ তারা + `4.5` (`toFixed(1)` → সবসময় এক দশমিক) |
| Joined | `DateCell` |

- `cell: ({ row }) => ...` → `row.original` = পুরো `IDoctor`।
- `tabular-nums` → সংখ্যা সারিবদ্ধ (note 12)।
- `.tsx` কারণ JSX আছে, কিন্তু `"use client"` লাগে না — শুধু client `DoctorsTable` import করে।

---

## ধাপ ১১: `DoctorsTable.tsx` — আসল table (ধাপ ৫-এর বদলে)

```tsx
"use client";

import DataTable from "@/components/shared/table/DataTable";
import { buildQueryString } from "@/lib/buildQueryString";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import type { IDoctor } from "@/types/doctor.types";
import { useQuery } from "@tanstack/react-query";
import { useSearchParams } from "next/navigation";
import { doctorColumns } from "./doctorsColumns";
import { DOCTOR_TABLE_DEFAULTS, DOCTOR_TABLE_QUERY_KEYS } from "./doctorsTable.config";

const getDoctorId = (doctor: IDoctor) => doctor.id;

export default function DoctorsTable() {
	const searchParams = useSearchParams();
	const queryString = buildQueryString(searchParams, DOCTOR_TABLE_QUERY_KEYS, DOCTOR_TABLE_DEFAULTS);

	const { data: response, isPending, isFetching, isError } = useQuery(doctorsQueryOptions(queryString));

	if (isError && !response) {
		return <p className="text-sm text-destructive">Failed to load doctors.</p>;
	}

	return (
		<DataTable
			data={response?.data ?? []}
			columns={doctorColumns}
			getRowId={getDoctorId}
			isLoading={isPending}
			isFetching={isFetching}
			emptyMessage="No doctors found."
		/>
	);
}
```

- `getDoctorId` **component-এর বাইরে** → প্রতি render-এ নতুন function বানায় না।
- `isPending` → প্রথমবার data নেই → skeleton। (Prefetch সফল হলে প্রথম থেকেই data থাকে, skeleton দেখা যায় না।)
- `isError && !response` → শুধু তখনই error, যখন দেখানোর মতো কোনো data নেই। পুরনো data থাকলে সেটাই দেখায় (note 15-এর মতো)।
- এখনো sort/search/pagination নেই → note 17।

---

## 📁 এই নোট শেষে file

```
src/
├── types/doctor.types.ts                          ← IDoctor আপডেট
├── lib/
│   ├── buildQueryString.ts                        ← নতুন (server + client)
│   └── formatters.ts                              ← নতুন
├── components/shared/table/
│   ├── dataTableFeatures.ts                       ← নতুন
│   ├── DataTable.tsx                              ← নতুন
│   └── cells/{UserInfoCell,DateCell}.tsx          ← নতুন
├── components/modules/Admin/DoctorsManagement/
│   ├── doctorsTable.config.ts                     ← নতুন
│   ├── doctorsColumns.tsx                         ← নতুন
│   └── DoctorsTable.tsx                           ← নতুন
└── app/(dashboardLayout)/admin/dashboard/doctors-management/page.tsx
```

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | ADMIN হয়ে sidebar → Doctors | heading + table, ১০টা doctor |
| ২ | Network tab | browser থেকে doctors-এর আলাদা request নেই (server prefetch) |
| ৩ | ছবি নেই এমন doctor | avatar-এ আদ্যক্ষর (`Dr.` বাদ দিয়ে) |
| ৪ | Specialty নেই এমন doctor | "No specialties" |
| ৫ | Fee / Rating / Joined | `৳1,500`, ⭐ `4.5`, `08 Nov 2026` |
| ৬ | Browser ছোট করো | table পাশে scroll হয় (shadcn `Table` নিজে `overflow-x-auto` wrapper দেয়) |
| ৭ | Backend বন্ধ করে reload | "Failed to load doctors." |

---

## 🔜 পরে যা যোগ হবে
- [ ] Search, sort, pagination, filter → note 17
- [ ] View dialog, Delete → note 18
- [ ] Create / Edit doctor (project-এ `CreateDoctorDialog`, `EditDoctorDialog`, `DoctorForm` আছে — পরের নোটে)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

1. **v8-এর tutorial দেখে লিখলে মিলবে না** — v9-এ নাম বদলেছে। মিলিয়ে নেওয়ার ছোট তালিকা:

   | v8 | v9 (আমাদের) |
   |---|---|
   | `useReactTable({ getCoreRowModel: getCoreRowModel() })` | `useTable({ features })` |
   | `flexRender(header.column.columnDef.header, header.getContext())` | `<table.FlexRender header={header} />` |
   | `ColumnDef<TData>` | `ColumnDef<Features, TData>` |
   | `getVisibleCells()` | `getAllCells()` (আমরা column hide করি না) |

2. **`formatters.ts` এখন একমাত্র জায়গা হোক** — `StatsCard` (note 12), pie/bar chart (note 13, 14) আর `AdminDashboardContent` (note 15) প্রত্যেকে নিজের `Intl.NumberFormat` বানায়। পরে সময় পেলে সবাইকে `formatNumber` / `formatCurrency` import করাও — format বদলাতে হলে এক জায়গায় বদলালেই হবে।
