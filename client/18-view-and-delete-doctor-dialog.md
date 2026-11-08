# 18 - Doctor Table: View Dialog ও Delete (Row Action + useMutation + Toast)

> Note 16–17-এর doctor table-এর প্রতিটা row-এর শেষে "⋯" menu: **View** (doctor-এর পুরো তথ্য dialog-এ) আর **Delete** (নিশ্চিত করে soft delete)। সাথে কয়েকটা reusable জিনিস: `useRowAction` (কোন row-এ কোন dialog), `useActionMutation` (Server Action + toast + table refresh), `getActionErrorMessage`।

## 🧭 এক নজরে

```
"⋯" চাপো → View / Delete
  → useRowAction: state = { type: "view" | "delete", row, open: true }
  → <ViewDoctorDialog {...dialogProps("view")} />     → শুধু দেখায়
  → <DeleteDoctorDialog {...dialogProps("delete")} /> → নিশ্চিত করলে:
```
```
Delete বাটন
  → useMutation → deleteDoctorAction (Server Action, id validate করে)
     → doctor.services → httpClient (+ cookie) → backend DELETE /doctors/:id
  → সফল: invalidateQueries(["doctors"]) → table নতুন data আনে → dialog বন্ধ → toast
  → ব্যর্থ: toast.error(message), dialog খোলা থাকে
```

---

# ভাগ ১: View Dialog

## ধাপ ১: Install

```shell
pnpm dlx shadcn@latest add dialog
```

## ধাপ ২: `src/types/doctor.types.ts` — `IDoctor`-এ দুটো field যোগ

```ts
export interface IDoctor {
	id: string;
	name: string;
	email: string;
	profilePhoto?: string | null;
	contactNumber?: string | null; // ✅ নতুন
	registrationNumber: string; // ✅ নতুন
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

- `contactNumber` → DB-তে optional (`String?`), তাই `?` + `| null`।
- `registrationNumber` → DB-তে বাধ্যতামূলক আর unique।

> ✅ এই দুটো field backend-এর list API (`GET /doctors`)-তে আসে (`fields` param না দিলে পুরো doctor আসে)।

## ধাপ ৩: `src/hooks/useRowAction.ts` (নতুন) — কোন row-এ কোন dialog

```ts
"use client";

import { useCallback, useMemo, useState } from "react";

export type RowActionType = "view" | "edit" | "delete";

interface RowActionState<TData> {
	type: RowActionType;
	row: TData;
	open: boolean;
}

// একটাই state: "কোন row-এ কোন কাজ"। একসাথে দুটো dialog খোলা থাকা অসম্ভব।
export function useRowAction<TData>() {
	const [state, setState] = useState<RowActionState<TData> | null>(null);

	// setState stable, তাই actions-ও stable থাকে → DataTable-এর columns বারবার নতুন হয় না
	const actions = useMemo(
		() => ({
			onView: (row: TData) => setState({ type: "view", row, open: true }),
			onEdit: (row: TData) => setState({ type: "edit", row, open: true }),
			onDelete: (row: TData) => setState({ type: "delete", row, open: true }),
		}),
		[],
	);

	// বন্ধ করার সময় row রেখে দিই: close animation চলাকালীন content হারায় না
	const close = useCallback(() => setState((prev) => (prev ? { ...prev, open: false } : prev)), []);

	// নির্দিষ্ট dialog-এর জন্য props: <ViewDialog {...dialogProps("view")} />
	const dialogProps = useCallback(
		(type: RowActionType) => ({
			row: state?.type === type ? state.row : null,
			open: state?.type === type && state.open,
			onOpenChange: (open: boolean) => {
				if (!open) close();
			},
		}),
		[state, close],
	);

	return { actions, close, dialogProps };
}
```

### সমস্যাটা
প্রতিটা dialog-এর জন্য আলাদা state (`viewOpen`, `viewRow`, `deleteOpen`, `deleteRow` …) রাখলে state অনেক, আর ভুল করে দুটো dialog একসাথে খোলা থাকতে পারে।

### সমাধান: একটাই state

```ts
{ type: "view", row: doctor, open: true }   // অথবা null
```
"কোন row-এ কোন কাজ" — একটাই মান, তাই একসাথে দুটো dialog খোলা **অসম্ভব**।

| অংশ | কাজ |
|---|---|
| `actions` (`onView`, `onEdit`, `onDelete`) | DataTable-এর "⋯" menu-তে যায়। `useMemo([])` → সবসময় একই object (setState stable), তাই DataTable-এর columns বারবার নতুন হয় না |
| `close()` | `open: false`, কিন্তু **`row` রেখে দেয়** → dialog বন্ধ হওয়ার animation-এর সময় ভিতরের লেখা হঠাৎ ফাঁকা হয়ে যায় না |
| `dialogProps("view")` | ঐ dialog-এর জন্য `{ row, open, onOpenChange }` — type মিললে row আর open, নাহলে `null`/`false` |

```tsx
<ViewDoctorDialog {...rowAction.dialogProps("view")} />
// = <ViewDoctorDialog row={doctor বা null} open={true/false} onOpenChange={...} />
```
- Generic `<TData>` → doctor, patient, appointment যেকোনো table-এ।

## ধাপ ৪: `doctorsTable.config.ts` — শেষে যোগ

```ts
export const GENDER_LABELS: Record<string, string> = Object.fromEntries(
	GENDER_OPTIONS.map((option) => [option.value, option.label]),
);
```

- `GENDER_OPTIONS` (`[{ label: "Male", value: "MALE" }, ...]`) থেকে উল্টো map: `{ MALE: "Male", FEMALE: "Female" }`।
- `Object.fromEntries` → `[key, value]` জোড়ার list থেকে object।
- একই তথ্য দুবার হাতে না লিখে একটা থেকে আরেকটা বানানো → নতুন gender যোগ করলে এক জায়গায়।

## ধাপ ৫: `ViewDoctorDialog.tsx` (নতুন)

`src/components/modules/Admin/DoctorsManagement/ViewDoctorDialog.tsx`

```tsx
"use client";

import DateCell from "@/components/shared/table/cells/DateCell";
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Separator } from "@/components/ui/separator";
import { formatCurrency, getInitials } from "@/lib/formatters";
import type { IDoctor } from "@/types/doctor.types";
import { ExternalLink, Star } from "lucide-react";
import Link from "next/link";
import type { ReactNode } from "react";
import { GENDER_LABELS } from "./doctorsTable.config";

interface ViewDoctorDialogProps {
	row: IDoctor | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

// <dl>: label আর value-র জোড়া, screen reader সঠিকভাবে পড়ে
const DetailItem = ({ label, children, wide }: { label: string; children: ReactNode; wide?: boolean }) => (
	<div className={wide ? "sm:col-span-2" : undefined}>
		<dt className="text-xs text-muted-foreground">{label}</dt>
		<dd className="mt-0.5 text-sm font-medium break-words">{children}</dd>
	</div>
);

const EMPTY = <span className="font-normal text-muted-foreground">—</span>;

export default function ViewDoctorDialog({ row: doctor, open, onOpenChange }: ViewDoctorDialogProps) {
	return (
		<Dialog open={open} onOpenChange={onOpenChange}>
			<DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-lg">
				{doctor && (
					<>
						<DialogHeader>
							<div className="flex items-center gap-4">
								<Avatar className="size-16">
									{doctor.profilePhoto && <AvatarImage src={doctor.profilePhoto} alt="" />}
									<AvatarFallback className="bg-primary/10 text-lg font-semibold text-primary">
										{getInitials(doctor.name)}
									</AvatarFallback>
								</Avatar>

								<div className="min-w-0 text-left">
									<DialogTitle className="truncate">{doctor.name}</DialogTitle>
									<DialogDescription>
										{doctor.designation} · {doctor.currentWorkingPlace}
									</DialogDescription>
									<p className="mt-1 inline-flex items-center gap-1 text-sm tabular-nums">
										<Star className="size-3.5 fill-amber-400 text-amber-400" aria-hidden />
										{doctor.averageRating.toFixed(1)}
										<span className="sr-only">out of 5 rating</span>
									</p>
								</div>
							</div>
						</DialogHeader>

						{doctor.specialties.length > 0 && (
							<div className="flex flex-wrap gap-1.5">
								{doctor.specialties.map(({ specialty }) => (
									<Badge key={specialty.id} variant="secondary">
										{specialty.title}
									</Badge>
								))}
							</div>
						)}

						<Separator />

						<dl className="grid grid-cols-1 gap-x-6 gap-y-4 sm:grid-cols-2">
							<DetailItem label="Email">{doctor.email}</DetailItem>
							<DetailItem label="Phone">{doctor.contactNumber || EMPTY}</DetailItem>
							<DetailItem label="Gender">{GENDER_LABELS[doctor.gender] ?? doctor.gender}</DetailItem>
							<DetailItem label="Experience">{doctor.experience} years</DetailItem>
							<DetailItem label="Appointment fee">{formatCurrency(doctor.appointmentFee)}</DetailItem>
							<DetailItem label="Registration no.">{doctor.registrationNumber}</DetailItem>
							<DetailItem label="Qualification" wide>
								{doctor.qualification}
							</DetailItem>
							<DetailItem label="Address" wide>
								{doctor.address || EMPTY}
							</DetailItem>
							<DetailItem label="Joined">
								<DateCell value={doctor.createdAt} />
							</DetailItem>
						</dl>

						<DialogFooter>
							<Button variant="outline" asChild>
								<Link href={`/consultation/doctor/${doctor.id}`}>
									<ExternalLink />
									Public profile
								</Link>
							</Button>
						</DialogFooter>
					</>
				)}
			</DialogContent>
		</Dialog>
	);
}
```

### ভেঙে বোঝা

| অংশ | কাজ |
|---|---|
| `row: doctor` | prop-এর নাম `row` (useRowAction থেকে আসে), ভিতরে নাম বদলে `doctor` → পড়তে সহজ |
| `{doctor && (...)}` | `row` null হলে কিছু দেখায় না (Dialog নিজে `open` দিয়ে খোলে/বন্ধ হয়) |
| `max-h-[90dvh] overflow-y-auto` | ছোট screen-এ dialog-এর ভিতরে scroll, screen-এর বাইরে যায় না |
| `DialogTitle` / `DialogDescription` | Radix-এ বাধ্যতামূলক (note 10-এর Sheet-এর মতো) — এখানে চোখে দেখা যায়: নাম, আর পদবি · কর্মস্থল |
| `text-left` | shadcn DialogHeader mobile-এ মাঝখানে রাখে, avatar-এর পাশে বাঁয়ে সারিবদ্ধ দেখাতে |
| Rating-এ `sr-only` "out of 5 rating" | screen reader শুধু "4.5" না, "4.5 out of 5 rating" পড়ে |
| Specialty `Badge` | না থাকলে পুরো অংশই নেই |

### `DetailItem` আর `<dl>`
```tsx
<dl>                          // description list
	<div><dt>Email</dt><dd>doctor@mail.com</dd></div>
</dl>
```
- `<dl>` / `<dt>` (term) / `<dd>` (description) → HTML-এর সঠিক উপায় "label: value" জোড়ার জন্য। Screen reader জোড়া হিসেবে পড়ে।
- `wide` → `sm:col-span-2` → Qualification, Address-এর মতো লম্বা লেখা দুই column জুড়ে।
- `break-words` → লম্বা email/address পরের লাইনে ভাঙে।
- `EMPTY` → ফাঁকা হলে হালকা `—` (একবার বানিয়ে বারবার)।
- `GENDER_LABELS[doctor.gender] ?? doctor.gender` → `"MALE"` → `"Male"`, অচেনা হলে আসল মান।
- `<DateCell />` → table-এর একই cell dialog-এও।

### Footer
`Button asChild` + `Link` → button-এর মতো দেখতে, কিন্তু আসলে link → `/consultation/doctor/{id}` (public profile)।

> ✏️ Public doctor profile page (`consultation/doctor/[id]`) এখনো placeholder — পরে বানানো হবে।

## ধাপ ৬: `DoctorsTable.tsx`-এ View জোড়া

Import:
```tsx
import { useRowAction } from "@/hooks/useRowAction";
import ViewDoctorDialog from "./ViewDoctorDialog";
```

Component-এর ভেতরে, useQuery-গুলোর পরে:
```tsx
	const rowAction = useRowAction<IDoctor>();

	// আপাতত শুধু View (Edit আর Delete ধাপ 6–7-এ যোগ হবে)
	const tableActions = useMemo(() => ({ onView: rowAction.actions.onView }), [rowAction.actions]);
```

`<DataTable ... />`-এ:
```tsx
				actions={tableActions}
```

`<DataTablePagination ... />`-এর পরে, শেষের `</div>`-এর আগে:
```tsx
<ViewDoctorDialog {...rowAction.dialogProps("view")} />

```

- **`tableActions`-এ শুধু `onView`** → DataTable-এর menu-তে শুধু "View" দেখায় (যে action দেওয়া হয় শুধু সেটা)।
- `useMemo(..., [rowAction.actions])` → stable, DataTable-এর `useMemo`-তে columns বারবার নতুন হয় না।

### পুরো `DoctorsTable.tsx` (View সহ)

```tsx
"use client";

import DataTable from "@/components/shared/table/DataTable";
import DataTablePagination from "@/components/shared/table/DataTablePagination";
import DataTableSearch from "@/components/shared/table/DataTableSearch";
import DataTableMultiSelectFilter from "@/components/shared/table/filters/DataTableMultiSelectFilter";
import DataTableRangeFilter from "@/components/shared/table/filters/DataTableRangeFilter";
import DataTableSelectFilter from "@/components/shared/table/filters/DataTableSelectFilter";
import { Button } from "@/components/ui/button";
import { useRowAction } from "@/hooks/useRowAction";
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
import ViewDoctorDialog from "./ViewDoctorDialog";

// component-এর বাইরে: প্রতি render-এ নতুন function তৈরি হয় না
const getDoctorId = (doctor: IDoctor) => doctor.id;

export default function DoctorsTable() {
	const table = useTableUrlState({ defaultLimit: Number(DOCTOR_TABLE_DEFAULTS.limit) });
	const rowAction = useRowAction<IDoctor>();

	// server page-এর মতো একই নিয়ম → একই cache key, প্রথমবার আবার fetch হয় না
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

	// আপাতত শুধু View (Delete ধাপ 6-এ, Edit ধাপ 7-এ যোগ হবে)
	const tableActions = useMemo(() => ({ onView: rowAction.actions.onView }), [rowAction.actions]);

	const hasActiveFilters = DOCTOR_FILTER_KEYS.some((key) => table.searchParams.has(key));

	if (isError && !response) {
		return <p className="text-sm text-destructive">Failed to load doctors.</p>;
	}

	return (
		<div className="space-y-4">
			{/* Toolbar: search + filters */}
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
				actions={tableActions}
			/>

			<DataTablePagination meta={response?.meta} onPageChange={table.setPage} onLimitChange={table.setLimit} />

			{/* Row action dialogs */}
			<ViewDoctorDialog {...rowAction.dialogProps("view")} />
		</div>
	);
}
```

---

# ভাগ ২: Delete

## ধাপ ৭: `src/app/layout.tsx` — Toaster বসাও

```tsx
import { Toaster } from "@/components/ui/sonner";

// body te bosaite hobe
<body className="min-h-full flex flex-col">
<QueryProvider>{children}</QueryProvider>
<Toaster richColors closeButton position="top-right" />
</body>
```

- **Sonner** = toast (কোণে ছোট notification) library, shadcn-এর `sonner` component এটাই।
- `<Toaster />` root layout-এ একবার → app-এর যেকোনো জায়গা থেকে `toast.success(...)` / `toast.error(...)` call করলে এখানে দেখায়।
- `richColors` → success সবুজ, error লাল। `closeButton` → ✕। `position="top-right"`।

> Project-এ বসানো আছে। নতুন project-এ `pnpm dlx shadcn@latest add sonner`।

## ধাপ ৮: `src/types/api.types.ts` — `ActionResult` যোগ

```ts
// server action সবসময় এর একটা return করে, throw করে না
// (production-এ thrown error-এর message client পর্যন্ত পৌঁছায় না)
export type ActionResult<TData> = ApiResponse<TData> | ApiErrorResponse;
```

### কেন Server Action error **throw** না করে **return** করে? ⭐

| Throw করলে | Return করলে (`{ success: false, message }`) |
|---|---|
| Production-এ Next.js নিরাপত্তার জন্য Server Action-এর error message **লুকিয়ে ফেলে** → client পায় সাধারণ "An error occurred…" | আমাদের নিজের message client পর্যন্ত পৌঁছায় → toast-এ "Doctor not found" |
| Development-এ ঠিক দেখায়, production-এ আলাদা → বিভ্রান্তিকর | দুই জায়গায় একই |

- `ActionResult<TData>` = সফল হলে `ApiResponse<TData>` (`success: true`, `data`), ব্যর্থ হলে `ApiErrorResponse` (`success: false`, `message`)।
- `success`-এর মান দেখে TypeScript বুঝে যায় কোনটা (discriminated union): `if (!result.success)` এর পরে `result.data` নিরাপদ।

> (Note 15-এর `getAdminDashboardData` throw করে, কারণ সেটা query — সেখানে শুধু "Try again" দেখাই, message না।)

## ধাপ ৯: `src/lib/getActionErrorMessage.ts` (নতুন)

```ts
import type { ApiErrorResponse } from "@/types/api.types";
import axios from "axios";

const GENERIC_ERROR = "Something went wrong. Please try again.";

export const getActionErrorMessage = (error: unknown, context = "action"): string => {
	if (axios.isAxiosError<ApiErrorResponse>(error)) {
		const { response } = error;

		if (!response) {
			return error.code === "ECONNABORTED"
				? "The server took too long to respond. Please try again."
				: "Unable to reach the server. Please check your connection.";
		}

		// 5xx-এর ভেতরের details user-কে দেখানো হয় না
		if (response.status >= 500) return GENERIC_ERROR;

		return response.data?.message ?? GENERIC_ERROR;
	}

	console.error(`[${context}]`, error instanceof Error ? error.message : "Unknown error");
	return GENERIC_ERROR;
};
```

এরপর `auth.actions.ts`-এ নিজের `getErrorMessage` আর `GENERIC_ERROR` মুছে দাও। উপরে এটা import করো:

```ts
import { getActionErrorMessage } from "@/lib/getActionErrorMessage";
```

আর ভেতরে `getErrorMessage(error)`-এর বদলে লেখো `getActionErrorMessage(error, "auth.actions")`। File-এর ভেতরে `axios` import-এর আর কোনো ব্যবহার না থাকলে ওটাও মুছে দাও।

| অবস্থা | User দেখে |
|---|---|
| Server-এ পৌঁছানোই যায়নি, timeout (`ECONNABORTED`) | "The server took too long to respond…" |
| Server-এ পৌঁছানো যায়নি (network) | "Unable to reach the server…" |
| 5xx | সাধারণ message (ভিতরের details লুকানো) |
| 4xx (যেমন 404, 403) | backend-এর `message` |
| axios-এর বাইরের error | সাধারণ message + server log (`[context] ...`) |

- `context` → log-এ কোন action থেকে error (`[deleteDoctorAction] ...`)।
- Login action (note 07) আর doctor action এখন একই function → এক জায়গায় নিয়ম।

## ধাপ ১০: `src/services/doctor.services.ts` (পুরো file)

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { IDoctor } from "@/types/doctor.types";

// service: শুধু backend-এ request পাঠায়। validation আর error handling action-এ হয়।

export const getDoctors = async (queryString: string) =>
	httpClient.get<IDoctor[]>(queryString ? `/doctors?${queryString}` : "/doctors");

export const deleteDoctor = async (id: string) => httpClient.delete<IDoctor>(`/doctors/${encodeURIComponent(id)}`);
```

- **Service = শুধু backend-এ request।** Validation আর error handling action-এ।
- `encodeURIComponent(id)` → id-তে `/` বা `?` থাকলে URL ভাঙে না / অন্য endpoint-এ যায় না।
- Backend: `DELETE /doctors/:id` → `checkAuth(ADMIN, SUPER_ADMIN)` → soft delete (`isDeleted: true`, user-এর session মুছে)।

## ধাপ ১১: `src/actions/doctor.actions.ts` (নতুন)

```ts
"use server";

import { getActionErrorMessage } from "@/lib/getActionErrorMessage";
import { deleteDoctor } from "@/services/doctor.services";
import type { ActionResult } from "@/types/api.types";
import type { IDoctor } from "@/types/doctor.types";
import * as z from "zod";

// "use server" file-এর প্রতিটা export একটা public endpoint:
// যে কেউ যেকোনো মান পাঠাতে পারে, তাই input-কে unknown ধরে validate করি।
// কে delete করতে পারবে সেটা backend-এর checkAuth (ADMIN/SUPER_ADMIN) ঠিক করে।
const doctorIdSchema = z.uuid("Invalid doctor id");

export async function deleteDoctorAction(id: unknown): Promise<ActionResult<IDoctor>> {
	const parsedId = doctorIdSchema.safeParse(id);

	if (!parsedId.success) {
		return { success: false, message: parsedId.error.issues[0]?.message ?? "Invalid doctor id" };
	}

	try {
		return await deleteDoctor(parsedId.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "deleteDoctorAction") };
	}
}
```

### Service আর Action — পার্থক্য

| | Service (`services/`) | Action (`actions/`) |
|---|---|---|
| কাজ | backend-এ request | input validate + error কে message বানানো |
| Error | throw করে | **return** করে (`ActionResult`) |
| কে call করে | query (`useQuery`), action | client component (`useMutation`) |

- **`id: unknown`** → `"use server"` export public endpoint, যে কেউ যা খুশি পাঠাতে পারে। তাই type বিশ্বাস না করে **Zod দিয়ে check**।
- `z.uuid()` → backend-এর doctor id `uuid(7)`। ভুল id হলে backend পর্যন্ত যায়ই না।
- `safeParse` → throw না করে `{ success, data | error }`।
- `return await` → try-এর ভিতরে `await` দিলে তবেই error `catch`-এ ধরা পড়ে।
- **কে delete করতে পারবে** → frontend ঠিক করে না, backend-এর `checkAuth` (ADMIN/SUPER_ADMIN)। Patient কোনোভাবে call করলেও backend 403 → toast-এ message।

## ধাপ ১২: `src/queries/doctor.queries.ts` (পুরো file)

```ts
import { getDoctors } from "@/services/doctor.services";
import { queryOptions } from "@tanstack/react-query";

// mutation-এর পর ["doctors"] invalidate করলে সব doctor list (যেকোনো filter বা page) আবার আসবে
export const DOCTORS_QUERY_KEY = ["doctors"] as const;

export const doctorsQueryOptions = (queryString: string) =>
	queryOptions({
		queryKey: [...DOCTORS_QUERY_KEY, queryString],
		queryFn: () => getDoctors(queryString),
		staleTime: 1000 * 60 * 60, // 1 ঘণ্টা (বদলালে invalidate করি, তাই লম্বা রাখা নিরাপদ)
		gcTime: 1000 * 60 * 60 * 6, // 6 ঘণ্টা
	});
```

- **`DOCTORS_QUERY_KEY = ["doctors"]`** আলাদা constant → query আর mutation দুজনেই একই key ব্যবহার করে।
- Query key: `["doctors", "page=1&limit=10"]`, `["doctors", "page=2&..."]` …
- `invalidateQueries({ queryKey: ["doctors"] })` → `["doctors"]` দিয়ে **শুরু হওয়া সব** query (যেকোনো page/filter) "পুরনো" হয়ে যায় → এখন যেটা screen-এ আছে সেটা সাথে সাথে আবার আসে, বাকিগুলো পরে খুললে।
- `staleTime` ১ ঘণ্টা → doctor list কম বদলায়, আর বদলালে আমরা নিজেই invalidate করি, তাই নিরাপদ।

## ধাপ ১৩: `src/hooks/useActionMutation.ts` (নতুন)

```ts
"use client";

import type { ActionResult } from "@/types/api.types";
import { type QueryKey, useMutation, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

interface UseActionMutationOptions<TVariables, TData> {
	action: (variables: TVariables) => Promise<ActionResult<TData>>;
	invalidateKey: QueryKey; // সফল হলে কোন data আবার আনতে হবে
	successMessage: string | ((data: TData) => string);
	onSuccess?: (data: TData) => void;
}

export function useActionMutation<TVariables, TData>({
	action,
	invalidateKey,
	successMessage,
	onSuccess,
}: UseActionMutationOptions<TVariables, TData>) {
	const queryClient = useQueryClient();

	return useMutation({
		mutationFn: async (variables: TVariables) => {
			const result = await action(variables);
			// action error return করে (throw করে না); এখানে throw করলে React Query isError বোঝে
			if (!result.success) throw new Error(result.message);
			return result.data;
		},
		onSuccess: async (data) => {
			// await: table নতুন data পাওয়া পর্যন্ত বাটন "loading" থাকে, তারপর dialog বন্ধ হয়
			await queryClient.invalidateQueries({ queryKey: invalidateKey });
			toast.success(typeof successMessage === "function" ? successMessage(data) : successMessage);
			onSuccess?.(data);
		},
		onError: (error) => {
			toast.error(error.message);
		},
	});
}
```

যেকোনো "Server Action → toast → table refresh" কাজের জন্য একটা reusable hook (delete, পরে create, edit, status বদল…)।

| অংশ | কাজ |
|---|---|
| `action` | Server Action (যেমন `deleteDoctorAction`) |
| `mutationFn` | action call → `success: false` হলে **এখানে throw** → TanStack Query `isError` + `onError` বোঝে |
| `invalidateKey` | সফল হলে কোন data আবার আনতে হবে |
| `onSuccess` → `await invalidateQueries` | table নতুন data না পাওয়া পর্যন্ত `isPending` true থাকে → button "Deleting…" → তারপর toast + dialog বন্ধ। (await না দিলে dialog বন্ধ হয়ে যেত, কিন্তু table-এ মোছা doctor আরো এক মুহূর্ত থাকত) |
| `successMessage` | string, অথবা ফেরত data থেকে বানানো function |
| `onSuccess?` | বাড়তি কাজ (dialog বন্ধ) |
| `onError` | `toast.error(error.message)` → action-এর message (ধাপ ৯) |

```
Action  ──return { success: false, message }──►  mutationFn throw  ──►  onError → toast.error
        ──return { success: true, data }─────►  return data       ──►  onSuccess → invalidate → toast.success → close
```

## ধাপ ১৪: `DeleteDoctorDialog.tsx` (নতুন)

`src/components/modules/Admin/DoctorsManagement/DeleteDoctorDialog.tsx`

```tsx
"use client";

import { deleteDoctorAction } from "@/actions/doctor.actions";
import {
	AlertDialog,
	AlertDialogCancel,
	AlertDialogContent,
	AlertDialogDescription,
	AlertDialogFooter,
	AlertDialogHeader,
	AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { useActionMutation } from "@/hooks/useActionMutation";
import { DOCTORS_QUERY_KEY } from "@/queries/doctor.queries";
import type { IDoctor } from "@/types/doctor.types";
import { LoaderCircle, Trash2 } from "lucide-react";

interface DeleteDoctorDialogProps {
	row: IDoctor | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

export default function DeleteDoctorDialog({ row: doctor, open, onOpenChange }: DeleteDoctorDialogProps) {
	const { mutate, isPending } = useActionMutation({
		action: deleteDoctorAction,
		invalidateKey: DOCTORS_QUERY_KEY,
		successMessage: `${doctor?.name ?? "Doctor"} has been deleted`,
		onSuccess: () => onOpenChange(false),
	});

	return (
		// delete চলার সময় Esc বা বাইরে click করে বন্ধ করা যাবে না
		<AlertDialog open={open} onOpenChange={(next) => !isPending && onOpenChange(next)}>
			<AlertDialogContent>
				<AlertDialogHeader>
					<AlertDialogTitle>Delete this doctor?</AlertDialogTitle>
					<AlertDialogDescription>
						<strong className="text-foreground">{doctor?.name}</strong> will be removed and can no longer log in.
						Their past appointments and prescriptions stay in the records.
					</AlertDialogDescription>
				</AlertDialogHeader>

				<AlertDialogFooter>
					<AlertDialogCancel disabled={isPending}>Cancel</AlertDialogCancel>
					{/* AlertDialogAction না: ওটা click করলেই dialog বন্ধ করে দেয়, delete শেষ হওয়ার আগেই */}
					<Button variant="destructive" disabled={isPending || !doctor} onClick={() => doctor && mutate(doctor.id)}>
						{isPending ? <LoaderCircle className="animate-spin" /> : <Trash2 />}
						{isPending ? "Deleting..." : "Delete"}
					</Button>
				</AlertDialogFooter>
			</AlertDialogContent>
		</AlertDialog>
	);
}
```

Backend soft delete করে, তাই লেখাটা সেই অনুযায়ী রেখেছি: পুরনো রেকর্ড থাকবে, শুধু doctor আর login করতে পারবে না।

| অংশ | কেন |
|---|---|
| `AlertDialog` (Dialog না) | বিপজ্জনক কাজের নিশ্চিতকরণ — বাইরে click করে বন্ধ হয় না, focus "Cancel"-এ থাকে |
| `onOpenChange={(next) => !isPending && onOpenChange(next)}` | delete চলার সময় Esc / Cancel দিয়ে বন্ধ করা যাবে না (মাঝপথে বন্ধ হলে user বুঝবে না হলো কিনা) |
| **`Button` (AlertDialogAction না)** | `AlertDialogAction` click করলেই dialog বন্ধ করে দেয় — delete শেষ হওয়ার আগেই। নিজের Button দিয়ে আমরা ঠিক করি কখন বন্ধ হবে (সফল হলে `onSuccess`-এ) |
| `disabled={isPending \|\| !doctor}` | দুবার চাপলে দুবার delete হবে না |
| `mutate(doctor.id)` | শুধু id পাঠানো (পুরো object না) |
| Spinner + "Deleting…" | কাজ চলছে বোঝায় |
| ব্যর্থ হলে | toast.error, dialog **খোলা থাকে** → আবার চেষ্টা বা Cancel |

## ধাপ ১৫: `DoctorsTable.tsx`-এ Delete জোড়া — ৩টা পরিবর্তন

**ক) Import যোগ করো:**
```tsx
import { useEffect, useMemo } from "react";
import DeleteDoctorDialog from "./DeleteDoctorDialog";
```
(`useMemo`-এর লাইনটা `useEffect` যোগ করে বদলে দাও)

**খ) `tableActions`-এ `onDelete` যোগ করো, আর তার নিচে effect বসাও:**
```tsx
	const tableActions = useMemo(
		() => ({ onView: rowAction.actions.onView, onDelete: rowAction.actions.onDelete }),
		[rowAction.actions],
	);

	// শেষ page-এর শেষ doctor delete হলে ফাঁকা page-এ আটকে না থেকে আগের page-এ যাওয়া
	const meta = response?.meta;
	const { setPage } = table;
	useEffect(() => {
		if (meta && meta.totalPages > 0 && meta.page > meta.totalPages) setPage(meta.totalPages);
	}, [meta, setPage]);
```

**গ) `<ViewDoctorDialog ... />`-এর নিচে:**
```tsx
			<DeleteDoctorDialog {...rowAction.dialogProps("delete")} />
```

### শেষ page-এর effect কেন?
```
৫৭ জন doctor, limit ১০ → ৬টা page, শেষ page-এ ৭ জন
শেষ page-এ গিয়ে ৭ জনকেই delete → total ৫০ → এখন ৫টা page
কিন্তু URL-এ এখনো page=6 → ফাঁকা table ❌
effect: meta.page (6) > meta.totalPages (5) → setPage(5) ✅
```
- `meta.totalPages > 0` → সব doctor মুছে গেলে (`totalPages` 0) page 0-তে পাঠানো হয় না।
- `const { setPage } = table;` আলাদা করে নেওয়া → effect-এর dependency-তে পুরো `table` object না দিয়ে শুধু stable `setPage` (`table` প্রতি render-এ নতুন object)।

---

## 📁 এই নোটের file

```
src/
├── app/layout.tsx                                     ← + <Toaster />
├── types/
│   ├── api.types.ts                                   ← + ActionResult
│   └── doctor.types.ts                                ← + contactNumber, registrationNumber
├── lib/getActionErrorMessage.ts                       ← নতুন (auth.actions-ও ব্যবহার করে)
├── hooks/
│   ├── useRowAction.ts                                ← নতুন
│   └── useActionMutation.ts                           ← নতুন
├── services/doctor.services.ts                        ← + deleteDoctor
├── actions/doctor.actions.ts                          ← নতুন: deleteDoctorAction
├── queries/doctor.queries.ts                          ← + DOCTORS_QUERY_KEY
└── components/modules/Admin/DoctorsManagement/
    ├── doctorsTable.config.ts                         ← + GENDER_LABELS
    ├── ViewDoctorDialog.tsx                           ← নতুন
    ├── DeleteDoctorDialog.tsx                         ← নতুন
    └── DoctorsTable.tsx                               ← actions + dialogs + শেষ page effect
```

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | একটা row-এর "⋯" | View, (separator), Delete (লাল) |
| ২ | View | dialog: avatar, নাম, পদবি, rating, specialty, তথ্যের grid |
| ৩ | Esc / বাইরে click | View বন্ধ, বন্ধের animation-এ লেখা ফাঁকা হয় না |
| ৪ | Public profile | `/consultation/doctor/{id}`-এ যায় |
| ৫ | Delete → Cancel | কিছু হয় না |
| ৬ | Delete → Delete | "Deleting…" → table থেকে doctor নেই → dialog বন্ধ → সবুজ toast "X has been deleted" |
| ৭ | Delete চলার সময় Esc | বন্ধ হয় না |
| ৮ | Backend বন্ধ রেখে Delete | লাল toast "Unable to reach the server…", dialog খোলা |
| ৯ | শেষ page-এর শেষ doctor delete | আগের page-এ চলে যায় |
| ১০ | DB (`pnpm studio`) | doctor-এর `isDeleted: true`, row মোছেনি |
| ১১ | Mobile-এ View | dialog-এর ভিতরে scroll, তথ্য এক column-এ |
| ১২ | Login page-এ ভুল password | message আগের মতোই আসে (`getActionErrorMessage` বদলের পর) |

---

## 🔜 পরে যা যোগ হবে
- [x] Edit doctor (`onEdit` + `EditDoctorDialog`) — note 19। Create doctor — note 20 — project-এ `DoctorForm`, `CreateDoctorDialog`, `EditDoctorDialog` আছে, পরের নোটে
- [ ] Public doctor profile page (`/consultation/doctor/[id]`)
- [ ] অন্য table-এ একই `useRowAction` + `useActionMutation`

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ সংশোধিত

1. ~~Phone আর Registration no. সবসময় ফাঁকা দেখাবে~~
> ✏️ সংশোধন: আগে এখানে লেখা ছিল যে list API এই field পাঠায় না — সেটা **ভুল** ছিল। Backend-এর `QueryBuilder`-এ `doctorSelectableFields` শুধু তখনই কাজ করে যখন URL-এ `?fields=...` দেওয়া হয়; আমরা `fields` পাঠাই না, তাই `GET /doctors` পুরো doctor (`contactNumber`, `registrationNumber` সহ) + `user` + `specialties` পাঠায়। তাই View dialog-এ Phone আর Registration no. ঠিকমতো দেখায়, backend-এ কিছু বদলাতে হবে না।

### 🟡 ভালো practice

2. **Delete-এর success message render-এর সময় বানানো** — `successMessage: \`${doctor?.name} has been deleted\`` প্রতি render-এ নতুন হয়। এখন ঠিক আছে, কারণ dialog বন্ধের পরও `row` থাকে (useRowAction)। কিন্তু ফেরত data থেকে বানালে আরো নিরাপদ: `successMessage: (deleted) => \`${deleted.name} has been deleted\`` — backend delete-এর পর doctor ফেরত দেয় কিনা নিশ্চিত হয়ে নিও।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **নিজের account delete-এর মতো কাজ backend-এ আটকাও** — এখন শুধু doctor delete, তাই ঠিক আছে। পরে admin delete বানালে "নিজেকে delete করা যাবে না" নিয়মটা backend-এ রাখো, frontend-এ button লুকানো যথেষ্ট না।

4. **`useActionMutation` সব mutation-এর জন্য** — Create/Edit-এও একই hook: `action`, `invalidateKey`, `successMessage`, `onSuccess: () => close()`। নতুন করে toast/invalidate লিখতে হবে না।
