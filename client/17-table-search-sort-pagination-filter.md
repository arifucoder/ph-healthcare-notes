# 17 - Table-এর Search, Sort, Pagination ও Filter (URL State)

> Note 16-এর doctor table-এ এবার: **search** (টাইপ থামলে খোঁজে), **sort** (header চাপলে), **pagination** (page + rows), আর ৩ রকম **filter** (Gender, Specialties, Fee range)। সব অবস্থা থাকে **URL-এ** — তাই reload, back button, link share সব কাজ করে।

## 🧭 এক নজরে

```
User filter বা sort বা page বদলায়
  → useTableUrlState: window.history.pushState (শুধু URL বদলায়)
  → useSearchParams নতুন মান দেয় → queryString বদলায়
  → useQuery নতুন key দেখে একবার fetch করে
  → keepPreviousData: নতুন data না আসা পর্যন্ত পুরনোটা হালকা হয়ে থাকে
```

### কেন state URL-এ, `useState`-এ না?

| `useState` | URL (`?page=2&gender=MALE`) |
|---|---|
| reload করলে সব হারায় | reload-এ একই অবস্থা |
| link কাউকে পাঠালে সে প্রথম page দেখে | একই filter করা table দেখে |
| back button আগের page-এ (অন্য route) চলে যায় | back button আগের filter-এ ফেরে |
| Server prefetch জানে না কী filter | Server `searchParams` থেকে **একই** data prefetch করে (note 16) |

### Toolbar দেখতে কেমন

```
┌──────────────────────────┐   [Gender ▾] [≡ Specialties 2] [⇆ Fee 500–1500] [✕ Reset]
│ 🔍 Search name, email... │
└──────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────┐
│ Doctor ↕ │ Specialties │ Designation │ Experience ↑ │ Fee ↕ │ Rating ↕ │ ⋯│
│ ...                                                                      │
└──────────────────────────────────────────────────────────────────────────┘
Showing 11–20 of 57          Rows [10 ▾]   Page 2 of 6   [«] [‹] [›] [»]
```

---

## ধাপ ০: Install

```shell
pnpm dlx shadcn@latest add popover
```
Fee range filter-এর popup-এর জন্য shadcn `popover`। (`select`, `dropdown-menu`, `input`, `label`, `badge` আগেই আছে।)

---

## ধাপ ১: `src/hooks/useTableUrlState.ts` (নতুন) — মূল hook ⭐

```ts
"use client";

import type { SortingState } from "@tanstack/react-table";
import { usePathname, useSearchParams } from "next/navigation";
import { useCallback, useMemo } from "react";

type ParamValue = string | string[] | undefined;

interface SetParamsOptions {
	resetPage?: boolean; // filter/sort/search বদলালে page 1-এ ফেরা
	replace?: boolean; // search-এ প্রতি অক্ষরে history ভরে না যায়
}

const toPositiveInt = (value: string | null, fallback: number) => {
	const parsed = Number(value);
	return Number.isInteger(parsed) && parsed > 0 ? parsed : fallback;
};

export function useTableUrlState({ defaultLimit }: { defaultLimit: number }) {
	const searchParams = useSearchParams();
	const pathname = usePathname();

	// pushState শুধু URL বদলায়: Next.js useSearchParams আপডেট করে, কিন্তু server-এ আবার request যায় না।
	// নতুন data আনে শুধু useQuery (teacher-এর router.refresh()-এ দুবার আসত)।
	const setParams = useCallback(
		(update: (params: URLSearchParams) => void, { resetPage = true, replace = false }: SetParamsOptions = {}) => {
			const params = new URLSearchParams(window.location.search);
			update(params);
			if (resetPage) params.delete("page"); // default = 1, URL পরিষ্কার থাকে

			const query = params.toString();
			const url = query ? `${pathname}?${query}` : pathname;
			window.history[replace ? "replaceState" : "pushState"](null, "", url);
		},
		[pathname],
	);

	// ---------- read ----------
	const page = toPositiveInt(searchParams.get("page"), 1);
	const limit = toPositiveInt(searchParams.get("limit"), defaultLimit);
	const searchTerm = searchParams.get("searchTerm") ?? "";

	const sortBy = searchParams.get("sortBy");
	const sortOrder = searchParams.get("sortOrder");
	const sorting = useMemo<SortingState>(
		() => (sortBy && (sortOrder === "asc" || sortOrder === "desc") ? [{ id: sortBy, desc: sortOrder === "desc" }] : []),
		[sortBy, sortOrder],
	);

	// ---------- write ----------
	const setSorting = useCallback(
		(next: SortingState) =>
			setParams((params) => {
				const [first] = next;
				if (first) {
					params.set("sortBy", first.id);
					params.set("sortOrder", first.desc ? "desc" : "asc");
				} else {
					params.delete("sortBy");
					params.delete("sortOrder");
				}
			}),
		[setParams],
	);

	const setPage = useCallback(
		(nextPage: number) => setParams((params) => params.set("page", String(nextPage)), { resetPage: false }),
		[setParams],
	);

	const setLimit = useCallback(
		(nextLimit: number) => setParams((params) => params.set("limit", String(nextLimit))),
		[setParams],
	);

	const setSearchTerm = useCallback(
		(term: string) =>
			setParams(
				(params) => {
					if (term) params.set("searchTerm", term);
					else params.delete("searchTerm");
				},
				{ replace: true },
			),
		[setParams],
	);

	// একসাথে একাধিক key (যেমন fee-র min আর max) → একটাই URL পরিবর্তন
	const setFilters = useCallback(
		(updates: Record<string, ParamValue>) =>
			setParams((params) => {
				for (const [key, value] of Object.entries(updates)) {
					params.delete(key);
					const values = value === undefined ? [] : Array.isArray(value) ? value : [value];
					for (const item of values) {
						if (item) params.append(key, item);
					}
				}
			}),
		[setParams],
	);

	const clearParams = useCallback(
		(keys: readonly string[]) =>
			setParams((params) => {
				for (const key of keys) params.delete(key);
			}),
		[setParams],
	);

	return {
		searchParams,
		page,
		limit,
		searchTerm,
		sorting,
		setSorting,
		setPage,
		setLimit,
		setSearchTerm,
		setFilters,
		clearParams,
		getFilter: (key: string) => searchParams.get(key) ?? undefined,
		getFilterList: (key: string) => searchParams.getAll(key),
	};
}
```

### `setParams` — সব write-এর মূল

```ts
const params = new URLSearchParams(window.location.search); // এখনকার URL
update(params);                                             // যা বদলাতে চাই
if (resetPage) params.delete("page");                       // filter বদলালে page 1
window.history[replace ? "replaceState" : "pushState"](null, "", url);
```

**`router.push` কেন না, `window.history.pushState` কেন?**

| `router.push(url)` / `router.refresh()` | `window.history.pushState` |
|---|---|
| Next.js server-এ আবার page request পাঠায় (server component আবার চলে, prefetch আবার) | শুধু browser-এর URL বদলায়, **server-এ কিছু যায় না** |
| তারপর client `useQuery`-ও fetch করে → **দুবার** data আনা | Next.js নিজেই `useSearchParams` আপডেট করে → `useQuery` নতুন key → **একবার** fetch |

> Next.js (14.1+) `pushState`/`replaceState` ধরে নিয়ে `useSearchParams` sync করে — এটা অফিসিয়াল উপায়।

**`window.location.search` কেন, `searchParams` (hook-এর) কেন না?** একই মুহূর্তে দুটো জিনিস বদলালে (যেমন min আর max) hook-এর মান তখনো পুরনো থাকতে পারে। `window.location`-এ সবসময় সবচেয়ে নতুন URL।

**`resetPage` (default `true`)** → search/filter/sort বদলালে page ৩-এ থেকে গেলে হয়তো ৩ নম্বর page-ই নেই। তাই `page` মুছে দেওয়া (= ১)। শুধু `setPage`-এ `resetPage: false`।

**`replace`** → search-এ `replaceState`: প্রতিটা খোঁজ history-তে নতুন entry না, নাহলে back button চাপতে চাপতে হাত ব্যথা।

### Read অংশ

| মান | কোথা থেকে | নিরাপত্তা |
|---|---|---|
| `page`, `limit` | `toPositiveInt(...)` | `?page=abc` বা `-5` হলে default (১ / ১০) |
| `searchTerm` | না থাকলে `""` | — |
| `sorting` | `sortBy` + `sortOrder` → `[{ id: "appointmentFee", desc: true }]` | `sortOrder` শুধু `asc`/`desc` হলে; `useMemo` → একই মানে একই array (DataTable অকারণে re-render হয় না) |

### Write function

| Function | URL-এ কী করে | page reset? |
|---|---|---|
| `setSorting(state)` | `sortBy`, `sortOrder` বসায় বা মোছে | ✅ |
| `setPage(n)` | `page=n` | ❌ |
| `setLimit(n)` | `limit=n` | ✅ |
| `setSearchTerm(term)` | `searchTerm` (ফাঁকা হলে মোছে), **replace** | ✅ |
| `setFilters({ key: value })` | একসাথে কয়েকটা key; value string, string[] বা `undefined` (মোছা) | ✅ |
| `clearParams(keys)` | দেওয়া key-গুলো মোছে (Reset) | ✅ |
| `getFilter(key)` / `getFilterList(key)` | একটা মান / সব মান (multi-select) | — |

- `setFilters`-এ প্রথমে `params.delete(key)` তারপর `append` → পুরনো মান মুছে নতুন। Array হলে একই key একাধিকবার: `?specialties.specialty.title=Cardiology&specialties.specialty.title=Neurology`।
- প্রতিটা function `useCallback` → stable, তাই child component-এ prop হিসেবে দিলে অকারণে নতুন হয় না (search-এর effect-এর জন্য জরুরি, ধাপ ২)।

---

## ধাপ ২: `src/components/shared/table/DataTableSearch.tsx` — debounce search

```tsx
"use client";

import { Input } from "@/components/ui/input";
import { Search } from "lucide-react";
import { useEffect, useState } from "react";

interface DataTableSearchProps {
	value: string; // URL-এর মান
	onChange: (value: string) => void; // অবশ্যই stable (useCallback) হতে হবে
	placeholder?: string;
	debounceMs?: number;
}

export default function DataTableSearch({
	value,
	onChange,
	placeholder = "Search...",
	debounceMs = 400,
}: DataTableSearchProps) {
	const [inputValue, setInputValue] = useState(value);
	const [syncedValue, setSyncedValue] = useState(value);

	// URL বাইরে থেকে বদলালে (যেমন "Reset") input-ও মিলিয়ে নেওয়া।
	// React-এর নিয়ম: prop থেকে state ঠিক করতে effect না, render-এর সময়েই।
	if (value !== syncedValue) {
		setSyncedValue(value);
		setInputValue(value);
	}

	// টাইপ থামার debounceMs পরে একবারই URL বদলায় (প্রতি অক্ষরে API call হয় না)
	useEffect(() => {
		const trimmed = inputValue.trim();
		if (trimmed === value) return;

		const timer = setTimeout(() => onChange(trimmed), debounceMs);
		return () => clearTimeout(timer);
	}, [inputValue, value, onChange, debounceMs]);

	return (
		<div className="relative w-full sm:max-w-xs">
			<Search
				className="pointer-events-none absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground"
				aria-hidden
			/>
			<Input
				type="search"
				value={inputValue}
				onChange={(event) => setInputValue(event.target.value)}
				placeholder={placeholder}
				aria-label={placeholder}
				className="pl-8"
			/>
		</div>
	);
}
```

### দুটো মান: input-এর আর URL-এর

```
টাইপ: "c" → "ca" → "car" → "card"   (inputValue — প্রতি অক্ষরে বদলায়, শুধু screen-এ)
            ⏱ ৪০০ms কিছু টাইপ না করলে
                                     → onChange("card") → URL ?searchTerm=card → একবার API call
```
- **Debounce:** প্রতি অক্ষরে API call না করে, টাইপ থামার ৪০০ms পরে একবারই।
- `useEffect`-এ `setTimeout`, আর cleanup-এ `clearTimeout` → নতুন অক্ষর এলে আগের timer বাতিল, নতুন করে ৪০০ms।
- `if (trimmed === value) return` → URL-এ আগে থেকেই একই মান থাকলে কিছু করে না (অকারণ loop নেই)।
- `onChange` **stable হতেই হবে** (`useCallback`) — নাহলে প্রতি render-এ effect আবার চলে, timer বারবার শুরু হয়। `table.setSearchTerm` stable, তাই ঠিক আছে।

### বাইরে থেকে URL বদলালে (Reset / back button)

```ts
if (value !== syncedValue) {
	setSyncedValue(value);
	setInputValue(value);
}
```
- "Reset" চাপলে URL-এর `searchTerm` মুছে যায়, কিন্তু input-এ তখনো পুরনো লেখা। তাই prop (`value`) বদলালে input-ও মিলিয়ে নেওয়া।
- React-এর নিয়ম: prop থেকে state ঠিক করতে **`useEffect` না, render-এর সময়েই** (`syncedValue` দিয়ে আগের prop মনে রেখে)। Effect দিলে একবার ভুল মান দেখিয়ে তারপর ঠিক হত।
- 👉 https://react.dev/learn/you-might-not-need-an-effect#adjusting-some-state-when-a-prop-changes

### বাকি
- `type="search"` → mobile keyboard-এ "Search" বোতাম, আর কিছু browser-এ ✕ (clear)।
- Icon `absolute` + input-এ `pl-8` → icon input-এর ভিতরে বাঁয়ে।
- `aria-label={placeholder}` → label নেই, তাই screen reader-এর জন্য।

---

## ধাপ ৩: `src/components/shared/table/DataTablePagination.tsx`

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { formatNumber } from "@/lib/formatters";
import type { PaginationMeta } from "@/types/api.types";
import { ChevronLeft, ChevronRight, ChevronsLeft, ChevronsRight } from "lucide-react";

const PAGE_SIZE_OPTIONS = [10, 20, 50]; // backend maxLimit 100-এর মধ্যে

interface DataTablePaginationProps {
	meta?: PaginationMeta;
	onPageChange: (page: number) => void;
	onLimitChange: (limit: number) => void;
}

export default function DataTablePagination({ meta, onPageChange, onLimitChange }: DataTablePaginationProps) {
	if (!meta || meta.total === 0) return null;

	const { page, limit, total, totalPages } = meta;
	const from = (page - 1) * limit + 1;
	const to = Math.min(page * limit, total);
	const canGoBack = page > 1;
	const canGoForward = page < totalPages;

	return (
		<div className="flex flex-col-reverse items-center justify-between gap-3 sm:flex-row">
			<p className="text-sm tabular-nums text-muted-foreground">
				Showing {formatNumber(from)}–{formatNumber(to)} of {formatNumber(total)}
			</p>

			<div className="flex flex-wrap items-center justify-center gap-4">
				<div className="flex items-center gap-2">
					<span className="text-sm text-muted-foreground">Rows</span>
					<Select value={String(limit)} onValueChange={(value) => onLimitChange(Number(value))}>
						<SelectTrigger className="w-18" aria-label="Rows per page">
							<SelectValue />
						</SelectTrigger>
						<SelectContent>
							{PAGE_SIZE_OPTIONS.map((size) => (
								<SelectItem key={size} value={String(size)}>
									{size}
								</SelectItem>
							))}
						</SelectContent>
					</Select>
				</div>

				<span className="text-sm tabular-nums">
					Page {page} of {totalPages}
				</span>

				<div className="flex items-center gap-1">
					<Button variant="outline" size="icon" onClick={() => onPageChange(1)} disabled={!canGoBack} aria-label="First page">
						<ChevronsLeft />
					</Button>
					<Button variant="outline" size="icon" onClick={() => onPageChange(page - 1)} disabled={!canGoBack} aria-label="Previous page">
						<ChevronLeft />
					</Button>
					<Button variant="outline" size="icon" onClick={() => onPageChange(page + 1)} disabled={!canGoForward} aria-label="Next page">
						<ChevronRight />
					</Button>
					<Button variant="outline" size="icon" onClick={() => onPageChange(totalPages)} disabled={!canGoForward} aria-label="Last page">
						<ChevronsRight />
					</Button>
				</div>
			</div>
		</div>
	);
}
```

- **`meta` backend থেকে** (`{ page, limit, total, totalPages }`) — নিজে হিসাব করি না।
- `meta.total === 0` → কিছুই দেখায় না (খালি table-এ pagination অর্থহীন)।
- `from`/`to`: page ২, limit ১০, total ৫৭ → `from = 11`, `to = min(20, 57) = 20` → "Showing 11–20 of 57"। শেষ page-এ `to = 57`।
- ৪টা button: প্রথম « / আগের ‹ / পরের › / শেষ »। প্রথম page-এ « ‹ disabled, শেষ page-এ › » disabled।
- **Rows per page:** `PAGE_SIZE_OPTIONS = [10, 20, 50]` (backend-এর `maxLimit` ১০০-এর মধ্যে)। Select-এর মান string, তাই `String(limit)` আর `Number(value)`।
- `flex-col-reverse sm:flex-row` → mobile-এ button উপরে, "Showing…" নিচে; বড় screen-এ পাশাপাশি।
- Icon-only button-এ `aria-label`।

---

## ধাপ ৪: Filter — ৩টা ছোট component

তিন রকম filter, প্রতিটা আলাদা ছোট component:

```
components/shared/table/filters/
├── DataTableSelectFilter.tsx       ← একটা মান (Gender)
├── DataTableMultiSelectFilter.tsx  ← একাধিক মান (Specialties)
└── DataTableRangeFilter.tsx        ← min থেকে max (Fee)
```

| Filter | কখন | UI | URL |
|---|---|---|---|
| Select | একটা মান বাছাই | shadcn `Select` | `?gender=MALE` |
| Multi-select | একাধিক মান | `DropdownMenu` + checkbox | `?specialties.specialty.title=A&...=B` |
| Range | min থেকে max | `Popover` + ২টা input + Apply | `?appointmentFee[gte]=500&appointmentFee[lte]=1500` |

সবগুলো **`shared/table/filters/`-এ** — doctor-এর কিছু জানে না, শুধু `label`, `options`, `value`, `onChange`। পরে patient table-এ একই component।

### ৪.১ `DataTableSelectFilter.tsx`

```tsx
"use client";

import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";

export interface FilterOption {
	label: string;
	value: string;
}

// Radix Select-এ খালি string মান দেওয়া যায় না, তাই "সব"-এর জন্য আলাদা মান
const ALL_VALUE = "__all__";

interface DataTableSelectFilterProps {
	label: string;
	allLabel: string;
	value?: string;
	options: FilterOption[];
	onChange: (value: string | undefined) => void;
}

export default function DataTableSelectFilter({ label, allLabel, value, options, onChange }: DataTableSelectFilterProps) {
	return (
		<Select value={value ?? ALL_VALUE} onValueChange={(next) => onChange(next === ALL_VALUE ? undefined : next)}>
			<SelectTrigger className="w-36" aria-label={label}>
				<SelectValue />
			</SelectTrigger>
			<SelectContent>
				<SelectItem value={ALL_VALUE}>{allLabel}</SelectItem>
				{options.map((option) => (
					<SelectItem key={option.value} value={option.value}>
						{option.label}
					</SelectItem>
				))}
			</SelectContent>
		</Select>
	);
}
```

- `FilterOption` = `{ label, value }` → অন্য filter-ও এই type ব্যবহার করে (export)।
- **`ALL_VALUE = "__all__"`**: Radix Select-এ `value=""` দেওয়া যায় না (error)। তাই "সব"-এর জন্য একটা বিশেষ মান, আর বাইরে পাঠানোর সময় সেটাকে `undefined` বানানো → URL থেকে key মুছে যায়।

```
URL-এ নেই        → value undefined → Select দেখায় ALL_VALUE ("All genders")
"Male" বাছাই     → onChange("MALE") → ?gender=MALE
"All genders"    → onChange(undefined) → gender মুছে যায়
```

### ৪.২ `DataTableMultiSelectFilter.tsx`

```tsx
"use client";

import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import {
	DropdownMenu,
	DropdownMenuCheckboxItem,
	DropdownMenuContent,
	DropdownMenuItem,
	DropdownMenuSeparator,
	DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { ListFilter } from "lucide-react";
import type { FilterOption } from "./DataTableSelectFilter";

interface DataTableMultiSelectFilterProps {
	label: string;
	values: string[];
	options: FilterOption[];
	onChange: (values: string[]) => void;
}

export default function DataTableMultiSelectFilter({ label, values, options, onChange }: DataTableMultiSelectFilterProps) {
	const selected = new Set(values);

	const toggle = (value: string) => {
		const next = new Set(selected);
		if (next.has(value)) next.delete(value);
		else next.add(value);
		onChange([...next]);
	};

	return (
		<DropdownMenu>
			<DropdownMenuTrigger asChild>
				<Button variant="outline" className="border-dashed">
					<ListFilter />
					{label}
					{selected.size > 0 && <Badge variant="secondary">{selected.size}</Badge>}
				</Button>
			</DropdownMenuTrigger>

			<DropdownMenuContent align="start" className="max-h-72 w-56 overflow-y-auto">
				{options.length === 0 ? (
					<p className="px-2 py-1.5 text-sm text-muted-foreground">No options</p>
				) : (
					options.map((option) => (
						<DropdownMenuCheckboxItem
							key={option.value}
							checked={selected.has(option.value)}
							onCheckedChange={() => toggle(option.value)}
							onSelect={(event) => event.preventDefault()} // একাধিক বাছতে menu খোলা থাকে
						>
							{option.label}
						</DropdownMenuCheckboxItem>
					))
				)}

				{selected.size > 0 && (
					<>
						<DropdownMenuSeparator />
						<DropdownMenuItem onSelect={() => onChange([])} className="justify-center">
							Clear
						</DropdownMenuItem>
					</>
				)}
			</DropdownMenuContent>
		</DropdownMenu>
	);
}
```

- `new Set(values)` → "এটা বাছা আছে কিনা" দ্রুত দেখা (`selected.has`)।
- `toggle` → থাকলে বাদ, না থাকলে যোগ → নতুন array বাইরে।
- **`onSelect={(event) => event.preventDefault()}`** → item চাপলে menu বন্ধ হয় না, তাই একসাথে কয়েকটা বাছা যায় (note 11-এর logout-এর মতো একই কৌশল)।
- Button-এ বাছাই করা সংখ্যার `Badge` (যেমন `2`), আর `border-dashed` → "এটা filter" বোঝায়।
- `max-h-72 overflow-y-auto` → অনেক specialty হলে menu-র ভিতরে scroll।
- কিছু বাছা থাকলে নিচে "Clear"।

### ৪.৩ `DataTableRangeFilter.tsx`

```tsx
"use client";

import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { SlidersHorizontal } from "lucide-react";
import { type FormEvent, useId, useState } from "react";

interface Range {
	min?: string;
	max?: string;
}

interface DataTableRangeFilterProps extends Range {
	label: string;
	onChange: (range: Range) => void;
}

export default function DataTableRangeFilter({ label, min, max, onChange }: DataTableRangeFilterProps) {
	const id = useId();
	const [open, setOpen] = useState(false);
	const [draftMin, setDraftMin] = useState(min ?? "");
	const [draftMax, setDraftMax] = useState(max ?? "");

	// popover খোলার সময় URL-এর বর্তমান মান দিয়ে শুরু
	const handleOpenChange = (nextOpen: boolean) => {
		if (nextOpen) {
			setDraftMin(min ?? "");
			setDraftMax(max ?? "");
		}
		setOpen(nextOpen);
	};

	const isInvalid = draftMin !== "" && draftMax !== "" && Number(draftMin) > Number(draftMax);
	const isActive = Boolean(min || max);

	// Apply চাপলে তবেই URL বদলায় (টাইপ করার সময় প্রতি অক্ষরে API call হয় না)
	const handleSubmit = (event: FormEvent) => {
		event.preventDefault();
		if (isInvalid) return;
		onChange({ min: draftMin || undefined, max: draftMax || undefined });
		setOpen(false);
	};

	return (
		<Popover open={open} onOpenChange={handleOpenChange}>
			<PopoverTrigger asChild>
				<Button variant="outline" className="border-dashed">
					<SlidersHorizontal />
					{label}
					{isActive && (
						<Badge variant="secondary" className="tabular-nums">
							{min ?? "0"}–{max ?? "∞"}
						</Badge>
					)}
				</Button>
			</PopoverTrigger>

			<PopoverContent align="start" className="w-64">
				<form onSubmit={handleSubmit} className="space-y-3">
					<div className="grid grid-cols-2 gap-2">
						<div className="space-y-1">
							<Label htmlFor={`${id}-min`}>Min</Label>
							<Input
								id={`${id}-min`}
								type="number"
								inputMode="numeric"
								min={0}
								value={draftMin}
								onChange={(event) => setDraftMin(event.target.value)}
							/>
						</div>
						<div className="space-y-1">
							<Label htmlFor={`${id}-max`}>Max</Label>
							<Input
								id={`${id}-max`}
								type="number"
								inputMode="numeric"
								min={0}
								value={draftMax}
								onChange={(event) => setDraftMax(event.target.value)}
							/>
						</div>
					</div>

					{isInvalid && <p className="text-xs text-destructive">Min can&apos;t be greater than max.</p>}

					<div className="flex justify-end gap-2">
						<Button
							type="button"
							variant="ghost"
							size="sm"
							onClick={() => {
								onChange({});
								setOpen(false);
							}}
						>
							Clear
						</Button>
						<Button type="submit" size="sm" disabled={isInvalid}>
							Apply
						</Button>
					</div>
				</form>
			</PopoverContent>
		</Popover>
	);
}
```

- **Draft state:** টাইপ করার সময় শুধু `draftMin`/`draftMax` বদলায়। **Apply চাপলে তবেই** URL → প্রতি অক্ষরে API call হয় না।
- `handleOpenChange` → popover খোলার মুহূর্তে URL-এর বর্তমান মান দিয়ে draft শুরু (আগেরবার টাইপ করে Apply না করে বন্ধ করলে সেটা মুছে যায়)।
- `isInvalid` → min > max হলে লাল লেখা আর Apply disabled।
- `<form onSubmit>` → input-এ Enter চাপলেও Apply। `event.preventDefault()` → page reload হয় না।
- `useId()` → প্রতিটা filter-এর input-এর unique id, `Label htmlFor`-এর জন্য (একই page-এ দুটো range filter থাকলেও id মেলে না)।
- Button-এ `500–1500`, শুধু min থাকলে `500–∞`, শুধু max থাকলে `0–1500`।
- `type="number"` + `inputMode="numeric"` → mobile-এ সংখ্যার keyboard।
- "Clear" → `onChange({})` → দুটোই `undefined` → URL থেকে মোছে।

---

## ধাপ ৫: সব জোড়া দেওয়া

### ৫.১ `doctorsTable.config.ts` (পুরো file)

```ts
// admin doctor table-এর URL param নিয়ম এক জায়গায়: page (server) আর table (client) দুজনেই ব্যবহার করে
export const DOCTOR_TABLE_DEFAULTS = { page: "1", limit: "10" } as const;

export const SPECIALTY_FILTER_KEY = "specialties.specialty.title";
export const FEE_MIN_KEY = "appointmentFee[gte]";
export const FEE_MAX_KEY = "appointmentFee[lte]";

// "Reset" চাপলে এগুলো মুছবে (page, limit, sort থাকবে)
export const DOCTOR_FILTER_KEYS = ["searchTerm", "gender", SPECIALTY_FILTER_KEY, FEE_MIN_KEY, FEE_MAX_KEY] as const;

// backend doctor.constant.ts-এর searchable/filterable/sortable field অনুযায়ী
export const DOCTOR_TABLE_QUERY_KEYS: ReadonlySet<string> = new Set([
	"page",
	"limit",
	"sortBy",
	"sortOrder",
	...DOCTOR_FILTER_KEYS,
]);

// backend-এ OTHER নেই
export const GENDER_OPTIONS = [
	{ label: "Male", value: "MALE" },
	{ label: "Female", value: "FEMALE" },
];
```

নোট 16-এর থেকে নতুন:
- `FEE_MIN_KEY`, `FEE_MAX_KEY` → লম্বা string একবার।
- **`DOCTOR_FILTER_KEYS`** → "Reset" চাপলে এগুলো মুছবে, কিন্তু page, limit, sort থাকবে। `as const` → `readonly` tuple।
- `DOCTOR_TABLE_QUERY_KEYS` এখন `...DOCTOR_FILTER_KEYS` দিয়ে বানানো → নতুন filter যোগ করলে এক জায়গায় লিখলেই allowlist-এও ঢুকে যায়।
- `GENDER_OPTIONS` → Select filter-এর option (backend-এ `OTHER` নেই)।

### ৫.২ `DoctorsTable.tsx` (পুরো file)

```tsx
"use client";

import DataTable from "@/components/shared/table/DataTable";
import DataTablePagination from "@/components/shared/table/DataTablePagination";
import DataTableSearch from "@/components/shared/table/DataTableSearch";
import DataTableMultiSelectFilter from "@/components/shared/table/filters/DataTableMultiSelectFilter";
import DataTableRangeFilter from "@/components/shared/table/filters/DataTableRangeFilter";
import DataTableSelectFilter from "@/components/shared/table/filters/DataTableSelectFilter";
import { Button } from "@/components/ui/button";
import { useTableUrlState } from "@/hooks/useTableUrlState";
import { buildQueryString } from "@/lib/buildQueryString";
import { doctorsQueryOptions } from "@/queries/doctor.queries";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import type { IDoctor } from "@/types/doctor.types";
import { keepPreviousData, useQuery } from "@tanstack/react-query";
import { X } from "lucide-react";
import { useMemo } from "react";
import { doctorColumns } from "./doctorsColumns";
import {
	DOCTOR_FILTER_KEYS,
	DOCTOR_TABLE_DEFAULTS,
	DOCTOR_TABLE_QUERY_KEYS,
	FEE_MAX_KEY,
	FEE_MIN_KEY,
	GENDER_OPTIONS,
	SPECIALTY_FILTER_KEY,
} from "./doctorsTable.config";

const getDoctorId = (doctor: IDoctor) => doctor.id;

export default function DoctorsTable() {
	const table = useTableUrlState({ defaultLimit: Number(DOCTOR_TABLE_DEFAULTS.limit) });

	// server page-এর মতো একই নিয়ম → একই cache key
	const queryString = buildQueryString(table.searchParams, DOCTOR_TABLE_QUERY_KEYS, DOCTOR_TABLE_DEFAULTS);

	const { data: response, isPending, isFetching, isError } = useQuery({
		...doctorsQueryOptions(queryString),
		placeholderData: keepPreviousData, // নতুন data আসা পর্যন্ত পুরনোটা দেখায় (table ফাঁকা হয়ে লাফায় না)
	});

	const { data: specialtiesResponse } = useQuery(specialtiesQueryOptions());

	const specialtyOptions = useMemo(
		() => (specialtiesResponse?.data ?? []).map((specialty) => ({ label: specialty.title, value: specialty.title })),
		[specialtiesResponse],
	);

	const hasActiveFilters = DOCTOR_FILTER_KEYS.some((key) => table.searchParams.has(key));

	if (isError && !response) {
		return <p className="text-sm text-destructive">Failed to load doctors.</p>;
	}

	return (
		<div className="space-y-4">
			{/* Toolbar */}
			<div className="flex flex-col gap-3 lg:flex-row lg:items-center lg:justify-between">
				<DataTableSearch
					value={table.searchTerm}
					onChange={table.setSearchTerm}
					placeholder="Search name, email, specialty..."
				/>

				<div className="flex flex-wrap items-center gap-2">
					<DataTableSelectFilter
						label="Gender"
						allLabel="All genders"
						value={table.getFilter("gender")}
						options={GENDER_OPTIONS}
						onChange={(value) => table.setFilters({ gender: value })}
					/>

					<DataTableMultiSelectFilter
						label="Specialties"
						values={table.getFilterList(SPECIALTY_FILTER_KEY)}
						options={specialtyOptions}
						onChange={(values) => table.setFilters({ [SPECIALTY_FILTER_KEY]: values })}
					/>

					<DataTableRangeFilter
						label="Fee"
						min={table.getFilter(FEE_MIN_KEY)}
						max={table.getFilter(FEE_MAX_KEY)}
						onChange={({ min, max }) => table.setFilters({ [FEE_MIN_KEY]: min, [FEE_MAX_KEY]: max })}
					/>

					{hasActiveFilters && (
						<Button variant="ghost" onClick={() => table.clearParams(DOCTOR_FILTER_KEYS)}>
							<X />
							Reset
						</Button>
					)}
				</div>
			</div>

			<DataTable
				data={response?.data ?? []}
				columns={doctorColumns}
				getRowId={getDoctorId}
				isLoading={isPending}
				isFetching={isFetching}
				emptyMessage={hasActiveFilters ? "No doctors match these filters." : "No doctors found."}
				sorting={{ state: table.sorting, onSortingChange: table.setSorting }}
			/>

			<DataTablePagination meta={response?.meta} onPageChange={table.setPage} onLimitChange={table.setLimit} />
		</div>
	);
}
```

#### ভেঙে বোঝা

| অংশ | কাজ |
|---|---|
| `useTableUrlState({ defaultLimit: 10 })` | URL পড়া/লেখার সব function একটা object-এ (`table.xxx`) |
| `buildQueryString(table.searchParams, ...)` | server-এর মতো একই নিয়ম → একই cache key (note 16) |
| `...doctorsQueryOptions(queryString)` + **`placeholderData: keepPreviousData`** | key বদলালে (page ২ → ৩) নতুন data আসা পর্যন্ত **পুরনো data দেখায়** + `isFetching` true → DataTable হালকা (opacity) করে। না দিলে প্রতিবার table ফাঁকা → skeleton → data, লাফাত |
| `useQuery(specialtiesQueryOptions())` | specialty list (server-এ prefetch করা, note 16) → filter-এর option |
| `specialtyOptions` (`useMemo`) | `{ label: title, value: title }` — backend title দিয়ে filter করে (`specialties.specialty.title`) |
| `hasActiveFilters` | কোনো filter key URL-এ আছে? → Reset button দেখানো আর empty message বদলানো |
| `sorting={{ state, onSortingChange }}` | DataTable-এ sort চালু (note 16-এর `sorting` prop) |
| `<DataTablePagination meta=... />` | backend-এর `meta` দিয়ে |

#### Filter-এর `onChange`

```tsx
onChange={(value) => table.setFilters({ gender: value })}
onChange={(values) => table.setFilters({ [SPECIALTY_FILTER_KEY]: values })}
onChange={({ min, max }) => table.setFilters({ [FEE_MIN_KEY]: min, [FEE_MAX_KEY]: max })}
```
- `[SPECIALTY_FILTER_KEY]` → computed key: variable-এর মান (`"specialties.specialty.title"`) object-এর key হয়।
- Fee-তে min আর max **একটাই `setFilters`-এ** → URL একবারই বদলায়, একবারই fetch।

#### Toolbar layout
`flex-col lg:flex-row lg:justify-between` → mobile-এ search উপরে, filter নিচে; বড় screen-এ search বাঁয়ে, filter ডানে। Filter-গুলো `flex-wrap` → জায়গা না থাকলে পরের লাইনে।

---

## 🔄 পুরো চক্র — একটা উদাহরণ

```
User "Fee" header চাপে
  → DataTable: getToggleSortingHandler → onSortingChange(updater)
  → functionalUpdate → [{ id: "appointmentFee", desc: false }]
  → table.setSorting → setParams → ?sortBy=appointmentFee&sortOrder=asc (page মুছে)
  → pushState → useSearchParams নতুন
  → buildQueryString → "limit=10&page=1&sortBy=appointmentFee&sortOrder=asc"
  → useQuery: নতুন key → getDoctors() (Server Action) → httpClient + cookie → backend
  → ততক্ষণ: keepPreviousData → পুরনো row হালকা
  → data এল → নতুন row, header-এ ↑
Reload করলে → server page একই searchParams → একই queryString → prefetch → একই table ✅
```

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | Search-এ "car" টাইপ | থামার ~০.৪s পরে URL `?searchTerm=car`, একটাই request |
| ২ | দ্রুত টাইপ করে মুছে দাও | মাঝের অক্ষরগুলোর জন্য request হয় না |
| ৩ | "Fee" header ৩ বার চাপো | ↑ asc → ↓ desc → ↕ (sort মুছে যায়) |
| ৪ | Page ২-এ গিয়ে gender filter দাও | page ১-এ ফিরে আসে |
| ৫ | Specialties-এ ২টা বাছো | menu খোলা থাকে, button-এ `2`, URL-এ key দুবার |
| ৬ | Fee: min 1500, max 500 | লাল লেখা, Apply disabled |
| ৭ | Fee: 500–1500 → Apply | button-এ `500–1500` badge |
| ৮ | Reset | সব filter আর search মুছে, sort/limit থাকে, search box-ও খালি |
| ৯ | Page বদলানোর সময় | table ফাঁকা হয় না, হালকা হয়ে নতুন data |
| ১০ | Filter করা অবস্থায় reload | একই filter করা table (server prefetch থেকে) |
| ১১ | Back button | আগের filter/page-এ ফেরে |
| ১২ | URL-এ `?page=abc&foo=bar` লেখো | page 1, `foo` backend-এ যায় না |
| ১৩ | কিছু মেলে না এমন filter | "No doctors match these filters." |

---

## 🔜 পরে যা যোগ হবে
- [ ] Row action: View, Delete → note 18; Edit, Create → পরের নোট
- [ ] অন্য table (patients, appointments) — একই hook, একই filter component, শুধু নতুন config + columns

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **URL-এ হাতে বড় `limit` লিখলে Select ফাঁকা দেখাবে** — `?limit=30` লিখলে backend ৩০টা দেয়, কিন্তু Select-এর option-এ ৩০ নেই, তাই trigger ফাঁকা দেখায় (untested)। `useTableUrlState`-এ অচেনা limit হলে default ধরে নিলে ঠিক থাকে:
   ```ts
   const rawLimit = toPositiveInt(searchParams.get("limit"), defaultLimit);
   const limit = [10, 20, 50].includes(rawLimit) ? rawLimit : defaultLimit;
   ```
   (তাহলে `PAGE_SIZE_OPTIONS` একটা shared জায়গা থেকে export করে দুই জায়গায় ব্যবহার করো।)

2. **Fee-তে ঋণাত্মক সংখ্যা** — `min={0}` শুধু browser-এর arrow-এ কাজ করে, হাতে `-100` লেখা যায়। `isInvalid`-এ যোগ করো: `Number(draftMin) < 0 || Number(draftMax) < 0`।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Specialty filter title দিয়ে** — দুটো specialty-র একই title হলে বা title বদলালে পুরনো link কাজ করবে না। Backend `specialties.specialty.title` দিয়েই filter করে, তাই এখন এটাই ঠিক। পরে backend id দিয়ে filter সমর্থন করলে `value: specialty.id` করা নিরাপদ।

4. **`getFilter`/`getFilterList` প্রতি render-এ নতুন function** — এখন কোনো সমস্যা নেই (এগুলো শুধু render-এ call হয়)। কিন্তু কখনো `useEffect`-এর dependency-তে দিলে বারবার চলবে — তখন `useCallback` লাগবে।
