# 21 - Admin Schedule Management (৩০ মিনিটের Slot: Table + Create/Edit/View/Delete)

> Admin-এর `/admin/dashboard/schedules-management` page। **Schedule** = clinic-এর একটা ৩০ মিনিটের সময়-slot (যেমন ৮ Nov, 09:00–09:30)। Admin একসাথে অনেক slot বানায় (date range × প্রতিদিনের সময়), পরে doctor-রা এর থেকে নিজের slot বেছে নেয় (server note 26), আর patient সেই slot-এ appointment নেয়।
>
> Doctor management (note 16–20)-এর **হুবহু একই pattern** — DataTable, useTableUrlState, useRowAction, useActionMutation, form + dialog। তাই এই নোটে মূলত **schedule-এর নতুন জিনিস**: timezone (ঢাকা), date/time input, date range filter, slot গোনা, booked slot lock।

## 🧭 এক নজরে

```
page.tsx (Server) → prefetch /schedules?include=doctorSchedules&limit=10&page=1
  → SchedulesTable ("use client")
       ├─ Toolbar: [📅 Date: from → to] [✕ Reset]                    [+ Create schedule]
       ├─ DataTable: Date | Time | Duration | Doctors | Booked | Created | ⋯
       │     ⋯ → View (doctor list) / Edit (একটা slot সরানো) / Delete
       └─ Pagination

Create: startDate–endDate × startTime–endTime (প্রতিদিন) → ৩০ মিনিট করে কেটে অনেক slot
        → backend createManyAndReturn + skipDuplicates → নতুনগুলো ফেরত → "N schedule slots created"
Edit/Delete: slot booked হলে → UI-তেই আটকানো (backend-ও 409 দেয়)
```

### Backend (server note 25) যা দেয়

| Method | URL | কে | কাজ |
|---|---|---|---|
| `GET` | `/schedules?startDate=&endDate=&include=doctorSchedules` | ADMIN, SUPER_ADMIN, DOCTOR | list (date filter, sort, page) |
| `POST` | `/schedules` | ADMIN, SUPER_ADMIN | range থেকে অনেক slot |
| `PATCH` | `/schedules/:id` | ADMIN, SUPER_ADMIN | একটা slot-এর নতুন সময় (appointment থাকলে 409) |
| `DELETE` | `/schedules/:id` | ADMIN, SUPER_ADMIN | মোছা (appointment থাকলে 409), data ফেরত দেয় না |

---

## ধাপ ০: ফাইলের তালিকা (এই ক্রমে লেখো)

```
src/types/schedule.types.ts
src/lib/scheduleTime.ts
src/zod/schedule.validation.ts
src/services/schedule.services.ts
src/actions/schedule.actions.ts
src/queries/schedule.queries.ts
src/components/shared/table/filters/DataTableDateRangeFilter.tsx
src/components/modules/Admin/SchedulesManagement/
  ├─ schedulesTable.config.ts
  ├─ schedule.utils.ts
  ├─ schedulesColumns.tsx
  ├─ ScheduleForm.tsx
  ├─ CreateScheduleDialog.tsx
  ├─ EditScheduleDialog.tsx
  ├─ DeleteScheduleDialog.tsx
  ├─ ViewScheduleDialog.tsx
  └─ SchedulesTable.tsx
src/app/(dashboardLayout)/admin/dashboard/schedules-management/page.tsx
```

| ভাগ | File | Doctor-এর কোনটার মতো |
|---|---|---|
| Data | types, services, actions, queries | note 16, 18 |
| Shared নতুন | `lib/scheduleTime.ts`, `DataTableDateRangeFilter.tsx` | — (নতুন) |
| Validation | `zod/schedule.validation.ts` | note 19 (তবে এখানে একটাই schema) |
| Module | config, utils, columns, form, ৪টা dialog, table | note 16–20 |
| Page | `schedules-management/page.tsx` | note 16 |

> নতুন shadcn component লাগে না — `alert`, `dialog`, `alert-dialog`, `popover`, `badge`, `separator` আগেই আছে।

---

## ধাপ ১: `src/types/schedule.types.ts`

```ts
// backend scheduleIncludeConfig অনুযায়ী: doctorSchedules → doctor (appointments include হয় না)
export interface IScheduleDoctor {
	id: string;
	name: string;
	email: string;
	profilePhoto?: string | null;
}

export interface IDoctorScheduleLink {
	doctorId: string;
	scheduleId: string;
	isBooked: boolean;
	doctor: IScheduleDoctor;
}

export interface ISchedule {
	id: string;
	startDateTime: string;
	endDateTime: string;
	createdAt: string;
	updatedAt: string;
	doctorSchedules?: IDoctorScheduleLink[]; // ?include=doctorSchedules দিলে তবেই আসে
}

// backend create আর update দুটোই একই ৪টা field নেয়
export interface ISchedulePayload {
	startDate: string; // "YYYY-MM-DD"
	endDate: string;
	startTime: string; // "HH:mm"
	endTime: string;
}
```

- `ISchedule` → একটা slot। `startDateTime`/`endDateTime` ISO string (UTC, যেমন `"2026-11-08T03:00:00.000Z"` = ঢাকার সকাল ৯টা)।
- **`doctorSchedules?`** → optional, কারণ backend শুধু `?include=doctorSchedules` দিলে পাঠায় (`dynamicInclude`)। আমরা সবসময় চাই (ধাপ ৮)।
- `IDoctorScheduleLink` → join table (`DoctorSchedules`): কোন doctor এই slot নিয়েছে, আর booked কিনা।
- **`ISchedulePayload`** → create আর update দুটোই একই ৪টা string field (`"YYYY-MM-DD"`, `"HH:mm"`)।

---

## ধাপ ২: `src/lib/scheduleTime.ts` — ঢাকার সময়ের সব হিসাব ⭐

```ts
// backend schedule.constant.ts-এর সাথে মিল: clinic-এর সময় সবসময় ঢাকা (+06:00, DST নেই)
export const CLINIC_TIME_ZONE = "Asia/Dhaka";
export const SCHEDULE_INTERVAL_MINUTES = 30;

// locale + timeZone fixed: server (UTC) আর browser-এ একই output → hydration mismatch হয় না
const timeFormatter = new Intl.DateTimeFormat("en-US", {
	hour: "2-digit",
	minute: "2-digit",
	hour12: true,
	timeZone: CLINIC_TIME_ZONE,
});

const dateFormatter = new Intl.DateTimeFormat("en-GB", {
	day: "2-digit",
	month: "short",
	year: "numeric",
	timeZone: CLINIC_TIME_ZONE,
});

// en-CA → "2026-11-08" format, input type="date"-এর সাথে মেলে
const partsFormatter = new Intl.DateTimeFormat("en-CA", {
	year: "numeric",
	month: "2-digit",
	day: "2-digit",
	hour: "2-digit",
	minute: "2-digit",
	hourCycle: "h23",
	timeZone: CLINIC_TIME_ZONE,
});

const toValidDate = (value: string | null | undefined) => {
	if (!value) return null;
	const date = new Date(value);
	return Number.isNaN(date.getTime()) ? null : date;
};

export const formatClinicTime = (value: string | null | undefined) => {
	const date = toValidDate(value);
	return date ? timeFormatter.format(date) : "—";
};

export const formatClinicDateTime = (value: string | null | undefined) => {
	const date = toValidDate(value);
	return date ? `${dateFormatter.format(date)}, ${timeFormatter.format(date)}` : "—";
};

// ISO → { date: "2026-11-08", time: "09:30" } ঢাকার সময়ে (edit form-এর input-এর জন্য)
// teacher-এর date-fns format() browser-এর timezone নিত, ঢাকার বাইরে থাকলে ভুল সময় বসত
export const toClinicDateTimeParts = (value: string) => {
	const date = toValidDate(value);
	if (!date) return { date: "", time: "" };

	const parts = Object.fromEntries(partsFormatter.formatToParts(date).map(({ type, value: part }) => [type, part]));

	return {
		date: `${parts.year}-${parts.month}-${parts.day}`,
		time: `${parts.hour}:${parts.minute}`,
	};
};

export const getDurationMinutes = (start: string, end: string) => {
	const startDate = toValidDate(start);
	const endDate = toValidDate(end);
	if (!startDate || !endDate) return null;

	const minutes = Math.round((endDate.getTime() - startDate.getTime()) / 60_000);
	return minutes > 0 ? minutes : null;
};

// "09:30" → 570
export const timeToMinutes = (time: string) => {
	const [hours = 0, minutes = 0] = time.split(":").map(Number);
	return hours * 60 + minutes;
};
```

### সমস্যাটা: সময় কোন timezone-এ?

```
DB-তে:      2026-11-08T03:00:00.000Z   (UTC)
ঢাকায়:     ৮ Nov, 09:00 AM  (UTC+6)
লন্ডনে:     ৮ Nov, 03:00 AM
Server-এ:   server-এর timezone (Vercel = UTC)
```
- Clinic বাংলাদেশে, তাই slot-এর সময় **সবসময় ঢাকার সময়ে** দেখাতে হবে — admin যে দেশ থেকেই খুলুক।
- `timeZone: CLINIC_TIME_ZONE` ("Asia/Dhaka") দিলে server আর browser দুই জায়গায় একই লেখা → hydration mismatch নেই (note 14, 16-এর মতো)।
- Backend-এর `CLINIC_UTC_OFFSET = "+06:00"` আর `SCHEDULE_INTERVAL_MINUTES = 30`-এর সাথে মিল।

### Function-গুলো

| Function | Input | Output | কোথায় |
|---|---|---|---|
| `formatClinicTime` | ISO | `"09:00 AM"` | table-এর Time column, dialog |
| `formatClinicDateTime` | ISO | `"08 Nov 2026, 09:00 AM"` | toast, delete dialog |
| `toClinicDateTimeParts` | ISO | `{ date: "2026-11-08", time: "09:00" }` | edit form-এর input |
| `getDurationMinutes` | start, end | `30` (ভুল হলে `null`) | Duration column |
| `timeToMinutes` | `"09:30"` | `570` | validation, slot গোনা |

- `toValidDate` → ফাঁকা বা ভাঙা তারিখে `null` → `"—"` (ভাঙা Date-এ `Intl` RangeError throw করে, note 14)।

### `toClinicDateTimeParts` — কেন `formatToParts`?

Edit form-এ `<input type="date">` চায় `"2026-11-08"`, আর `<input type="time">` চায় `"09:00"` — **ঢাকার সময়ে**।

```ts
partsFormatter.formatToParts(date)
// [{ type: "year", value: "2026" }, { type: "literal", value: "-" }, { type: "month", value: "11" }, ...]
Object.fromEntries(... map(({ type, value }) => [type, value]))
// { year: "2026", month: "11", day: "08", hour: "09", minute: "00", literal: ... }
```
- `en-CA` → সংখ্যার format `YYYY-MM-DD`-এর মতো (২ অঙ্ক)।
- **`hourCycle: "h23"`** → ০–২৩ ঘণ্টা (রাত ১২টা = `"00"`, `"24"` না)।
- Teacher-এর `date-fns format()` **browser-এর timezone** নিত → admin ঢাকার বাইরে থাকলে form-এ ভুল সময় বসত, আর save করলে slot সরে যেত।

### `timeToMinutes`
```ts
const [hours = 0, minutes = 0] = time.split(":").map(Number);
```
- `= 0` default → `noUncheckedIndexedAccess`-এ array-এর item `undefined` হতে পারে, তাই।

---

## ধাপ ৩: `src/zod/schedule.validation.ts`

```ts
import { SCHEDULE_INTERVAL_MINUTES, timeToMinutes } from "@/lib/scheduleTime";
import * as z from "zod";

// backend schedule.validation.ts-এর একই নিয়ম।
// input type="date" → "YYYY-MM-DD", type="time" → "HH:mm": form-এর মান আর payload একই string,
// তাই doctor-এর মতো আলাদা form/server schema লাগে না, একই schema দুই জায়গায় চলে।

const dateField = (label: string) =>
	z.string().min(1, `${label} is required`).pipe(z.iso.date(`${label} is invalid`));

const timeField = (label: string) =>
	z
		.string()
		.min(1, `${label} is required`)
		.pipe(z.iso.time({ precision: -1, error: `${label} must be in HH:mm format` }));

const scheduleShape = z.object({
	startDate: dateField("Start date"),
	endDate: dateField("End date"),
	startTime: timeField("Start time"),
	endTime: timeField("End time"),
});

// Create: date range × প্রতিদিনের time window → অনেকগুলো slot
export const createScheduleSchema = scheduleShape
	.refine((data) => data.endDate >= data.startDate, {
		message: "End date can't be before start date",
		path: ["endDate"],
	})
	// backend: একটাও slot না হলে 400 দেয়, তাই এখানেই আটকাই
	.refine((data) => timeToMinutes(data.endTime) - timeToMinutes(data.startTime) >= SCHEDULE_INTERVAL_MINUTES, {
		message: `End time must be at least ${SCHEDULE_INTERVAL_MINUTES} minutes after start time`,
		path: ["endTime"],
	});

// Update: একটা slot-এর নতুন শুরু আর শেষ
export const updateScheduleSchema = scheduleShape.refine(
	(data) => `${data.endDate}T${data.endTime}` > `${data.startDate}T${data.startTime}`,
	{ message: "End must be after start", path: ["endTime"] },
);

export type ScheduleFormValues = z.input<typeof createScheduleSchema>;
```

### এখানে একটাই schema কেন (doctor-এ দুটো ছিল)?

| | Doctor (note 19) | Schedule |
|---|---|---|
| Form-এর মান | `"5"` (string) | `"2026-11-08"`, `"09:00"` |
| Backend-এর মান | `5` (number) | `"2026-11-08"`, `"09:00"` — **একই string** |
| Schema | form (string) + server (আসল type) — দুটো | **একটাই**, browser আর Server Action দুই জায়গায় |

`<input type="date">` আর `type="time"` নিজেই backend-এর format দেয়, তাই রূপান্তর লাগে না।

### Field

| অংশ | কাজ |
|---|---|
| `dateField` → `.min(1).pipe(z.iso.date())` | আগে "required", তারপর `YYYY-MM-DD` ঠিক কিনা (note 20-এর email-এর মতো `pipe`) |
| `timeField` → `z.iso.time({ precision: -1 })` | `precision: -1` = শুধু `HH:mm` (সেকেন্ড ছাড়া) — backend-এর একই নিয়ম |

### Create বনাম Update

| Schema | নিয়ম | মানে |
|---|---|---|
| `createScheduleSchema` | `endDate >= startDate` | date range ঠিক |
| | `endTime - startTime >= 30 মিনিট` | প্রতিদিন অন্তত ১টা slot হয় — নাহলে backend 400 দিত, এখানেই আটকানো |
| `updateScheduleSchema` | `"endDateT endTime" > "startDateT startTime"` | একটা slot, শেষ শুরুর পরে (string তুলনা কাজ করে কারণ format fixed: `2026-11-08T09:30`) |

- **Create-এ time মানে "প্রতিদিনের window"** (প্রতিদিন ০৯:০০–১২:০০), **Update-এ time মানে "এই slot-এর আসল শুরু/শেষ"**।
- `refine`-এর `path: ["endDate"]` / `["endTime"]` → error ওই field-এর নিচে দেখায়।
- `"YYYY-MM-DD"` string `>=` দিয়ে তুলনা নিরাপদ — অভিধানের ক্রম আর তারিখের ক্রম একই।

---

## ধাপ ৪: `src/services/schedule.services.ts`

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { ISchedule, ISchedulePayload } from "@/types/schedule.types";

// service: শুধু backend-এ request পাঠায়। validation আর error handling action-এ হয়।

export const getSchedules = async (queryString: string) =>
	httpClient.get<ISchedule[]>(queryString ? `/schedules?${queryString}` : "/schedules");

// backend createManyAndReturn + skipDuplicates → শুধু নতুন তৈরি হওয়া slot-গুলো ফেরত দেয়
export const createSchedule = async (payload: ISchedulePayload) => httpClient.post<ISchedule[]>("/schedules", payload);

export const updateSchedule = async (id: string, payload: ISchedulePayload) =>
	httpClient.patch<ISchedule>(`/schedules/${encodeURIComponent(id)}`, payload);

// backend data পাঠায় না
export const deleteSchedule = async (id: string) => httpClient.delete<null>(`/schedules/${encodeURIComponent(id)}`);
```

- Doctor-এর service-এর (note 18, 20) মতো: শুধু request।
- `createSchedule` → `ISchedule[]` (**শুধু নতুন তৈরি হওয়া**; আগে থেকে থাকা slot `skipDuplicates` দিয়ে বাদ)।
- `deleteSchedule` → backend `data` পাঠায় না, তাই `<null>`।

## ধাপ ৫: `src/actions/schedule.actions.ts`

```ts
"use server";

import { getActionErrorMessage } from "@/lib/getActionErrorMessage";
import { createSchedule, deleteSchedule, updateSchedule } from "@/services/schedule.services";
import type { ActionResult } from "@/types/api.types";
import type { ISchedule } from "@/types/schedule.types";
import { createScheduleSchema, updateScheduleSchema } from "@/zod/schedule.validation";
import * as z from "zod";

// public endpoint: input unknown ধরে validate করি। কে পারবে সেটা backend checkAuth (ADMIN/SUPER_ADMIN) ঠিক করে।
const scheduleIdSchema = z.uuid("Invalid schedule id");

const firstIssue = (error: z.ZodError) => error.issues[0]?.message ?? "Invalid input";

export async function createScheduleAction(payload: unknown): Promise<ActionResult<ISchedule[]>> {
	const parsed = createScheduleSchema.safeParse(payload);
	if (!parsed.success) {
		return { success: false, message: firstIssue(parsed.error) };
	}

	try {
		return await createSchedule(parsed.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "createScheduleAction") };
	}
}

export async function updateScheduleAction(id: unknown, payload: unknown): Promise<ActionResult<ISchedule>> {
	const parsedId = scheduleIdSchema.safeParse(id);
	if (!parsedId.success) {
		return { success: false, message: firstIssue(parsedId.error) };
	}

	const parsedPayload = updateScheduleSchema.safeParse(payload);
	if (!parsedPayload.success) {
		return { success: false, message: firstIssue(parsedPayload.error) };
	}

	try {
		return await updateSchedule(parsedId.data, parsedPayload.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "updateScheduleAction") };
	}
}

export async function deleteScheduleAction(id: unknown): Promise<ActionResult<unknown>> {
	const parsedId = scheduleIdSchema.safeParse(id);
	if (!parsedId.success) {
		return { success: false, message: firstIssue(parsedId.error) };
	}

	try {
		return await deleteSchedule(parsedId.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "deleteScheduleAction") };
	}
}
```

- Doctor actions-এর (note 20) হুবহু pattern: `unknown` → `safeParse` → service → `getActionErrorMessage`।
- **একই `createScheduleSchema` / `updateScheduleSchema`** browser-এর form-এও চলে, এখানেও (ধাপ ৩)।
- Backend-এর 409 ("Cannot update a schedule that already has appointments", "A schedule already exists at this time") → toast-এ সেই message।

## ধাপ ৬: `src/queries/schedule.queries.ts`

```ts
import { getSchedules } from "@/services/schedule.services";
import { queryOptions } from "@tanstack/react-query";

// ["schedules"] invalidate করলে সব page/filter-এর list আবার আসবে
export const SCHEDULES_QUERY_KEY = ["schedules"] as const;

export const schedulesQueryOptions = (queryString: string) =>
	queryOptions({
		queryKey: [...SCHEDULES_QUERY_KEY, queryString],
		queryFn: () => getSchedules(queryString),
		// doctor-দের booking অন্য জায়গা থেকে হয় (admin invalidate করে না), তাই doctors-এর চেয়ে ছোট
		staleTime: 1000 * 60 * 5,
		gcTime: 1000 * 60 * 30,
	});
```

- `SCHEDULES_QUERY_KEY = ["schedules"]` → mutation-এর পর invalidate (note 18-এর মতো)।
- **`staleTime` ৫ মিনিট** (doctor-এ ১ ঘণ্টা): doctor slot বেছে নিলে বা patient book করলে admin-এর page-এর cache invalidate হয় না (অন্য জায়গা থেকে বদলায়), তাই Doctors/Booked সংখ্যা বেশিক্ষণ পুরনো না থাকুক।

---

## ধাপ ৭: `src/components/shared/table/filters/DataTableDateRangeFilter.tsx` (নতুন)

```tsx
"use client";

import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { CalendarRange } from "lucide-react";
import { type FormEvent, useId, useState } from "react";

interface DateRange {
	from?: string; // "YYYY-MM-DD"
	to?: string;
}

interface DataTableDateRangeFilterProps extends DateRange {
	label: string;
	onChange: (range: DateRange) => void;
}

export default function DataTableDateRangeFilter({ label, from, to, onChange }: DataTableDateRangeFilterProps) {
	const id = useId();
	const [open, setOpen] = useState(false);
	const [draftFrom, setDraftFrom] = useState(from ?? "");
	const [draftTo, setDraftTo] = useState(to ?? "");

	// popover খোলার সময় URL-এর বর্তমান মান দিয়ে শুরু
	const handleOpenChange = (nextOpen: boolean) => {
		if (nextOpen) {
			setDraftFrom(from ?? "");
			setDraftTo(to ?? "");
		}
		setOpen(nextOpen);
	};

	// "YYYY-MM-DD" string compare-ই যথেষ্ট
	const isInvalid = draftFrom !== "" && draftTo !== "" && draftTo < draftFrom;
	const isActive = Boolean(from || to);

	const handleSubmit = (event: FormEvent) => {
		event.preventDefault();
		if (isInvalid) return;
		onChange({ from: draftFrom || undefined, to: draftTo || undefined });
		setOpen(false);
	};

	return (
		<Popover open={open} onOpenChange={handleOpenChange}>
			<PopoverTrigger asChild>
				<Button variant="outline" className="border-dashed">
					<CalendarRange />
					{label}
					{isActive && (
						<Badge variant="secondary" className="tabular-nums">
							{from ?? "…"} → {to ?? "…"}
						</Badge>
					)}
				</Button>
			</PopoverTrigger>

			<PopoverContent align="start" className="w-72">
				<form onSubmit={handleSubmit} className="space-y-3">
					<div className="grid grid-cols-2 gap-2">
						<div className="space-y-1">
							<Label htmlFor={`${id}-from`}>From</Label>
							<Input
								id={`${id}-from`}
								type="date"
								max={draftTo || undefined}
								value={draftFrom}
								onChange={(event) => setDraftFrom(event.target.value)}
							/>
						</div>
						<div className="space-y-1">
							<Label htmlFor={`${id}-to`}>To</Label>
							<Input
								id={`${id}-to`}
								type="date"
								min={draftFrom || undefined}
								value={draftTo}
								onChange={(event) => setDraftTo(event.target.value)}
							/>
						</div>
					</div>

					{isInvalid && <p className="text-xs text-destructive">&quot;To&quot; can&apos;t be before &quot;From&quot;.</p>}

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

Note 17-এর `DataTableRangeFilter` (Fee min–max)-এর **হুবহু কাঠামো**, শুধু `type="number"`-এর বদলে `type="date"`:

| অংশ | কাজ |
|---|---|
| draft state + Apply | টাইপ/বাছাইয়ের সময় URL বদলায় না, Apply চাপলে তবেই |
| `handleOpenChange` | খোলার মুহূর্তে URL-এর মান দিয়ে draft শুরু |
| `draftTo < draftFrom` | `"YYYY-MM-DD"` string তুলনাই যথেষ্ট |
| **`max={draftTo}` / `min={draftFrom}`** | date picker-এ ভুল দিক ধূসর হয়ে যায় — ভুল বাছাই করাই কঠিন |
| Badge | `2026-11-01 → 2026-11-30`, একদিক ফাঁকা হলে `…` |
| `&quot;` `&apos;` | JSX-এ `"` আর `'` সরাসরি লিখলে ESLint error (`react/no-unescaped-entities`) |

- `shared/`-এ, কারণ পরে appointment, payment table-এও date filter লাগবে।

---

## ধাপ ৮: Module — config, utils, columns

### ৮.১ `schedulesTable.config.ts`

```ts
// admin schedule table-এর URL param নিয়ম: page (server) আর table (client) দুজনেই ব্যবহার করে

// include: backend default-এ কোনো relation দেয় না, তাই doctorSchedules চেয়ে নিই।
// "include" allowlist-এ নেই → URL দিয়ে বদলানো যায় না, কিন্তু default হিসেবে সবসময় যায়
export const SCHEDULE_TABLE_DEFAULTS = { page: "1", limit: "10", include: "doctorSchedules" } as const;

export const START_DATE_KEY = "startDate";
export const END_DATE_KEY = "endDate";

// backend-এ searchableFields নেই, তাই search box নেই। date range দিয়ে filter হয়।
export const SCHEDULE_FILTER_KEYS = [START_DATE_KEY, END_DATE_KEY] as const;

// backend scheduleSortableFields: startDateTime, endDateTime, createdAt
export const SCHEDULE_TABLE_QUERY_KEYS: ReadonlySet<string> = new Set([
	"page",
	"limit",
	"sortBy",
	"sortOrder",
	...SCHEDULE_FILTER_KEYS,
]);
```

**`include: "doctorSchedules"` default-এ কেন?** ⭐
- Backend `?include=doctorSchedules` না দিলে কোন doctor slot নিয়েছে, booked কিনা — কিছুই পাঠায় না।
- `buildQueryString` (note 16) default-গুলো সবসময় বসায় → প্রতিটা request-এ `include=doctorSchedules` যায়।
- কিন্তু **allowlist-এ (`SCHEDULE_TABLE_QUERY_KEYS`) `include` নেই** → কেউ URL-এ `?include=appointments` লিখলে বাদ পড়ে, আর default আবার বসে। URL দিয়ে বদলানো যায় না।

```
URL:  /schedules-management?startDate=2026-11-01&include=hack
→ buildQueryString → "startDate=2026-11-01&page=1&limit=10&include=doctorSchedules"
```

- **Search box নেই**, কারণ backend-এ schedule-এর `searchableFields` নেই (খোঁজার মতো লেখা নেই)। Filter = শুধু date range।
- `START_DATE_KEY`/`END_DATE_KEY` → backend নিজে handle করে: ঢাকার `startDate` দিনের শুরু থেকে `endDate` দিনের শেষ পর্যন্ত।

### ৮.২ `schedule.utils.ts`

```ts
import { SCHEDULE_INTERVAL_MINUTES, timeToMinutes, toClinicDateTimeParts } from "@/lib/scheduleTime";
import type { ISchedule } from "@/types/schedule.types";
import { createScheduleSchema, type ScheduleFormValues } from "@/zod/schedule.validation";

export const EMPTY_SCHEDULE_FORM: ScheduleFormValues = {
	startDate: "",
	endDate: "",
	startTime: "",
	endTime: "",
};

export const scheduleToFormValues = (schedule: ISchedule): ScheduleFormValues => {
	const start = toClinicDateTimeParts(schedule.startDateTime);
	const end = toClinicDateTimeParts(schedule.endDateTime);

	return { startDate: start.date, startTime: start.time, endDate: end.date, endTime: end.time };
};

export const isSameFormValues = (a: ScheduleFormValues, b: ScheduleFormValues) =>
	(Object.keys(a) as Array<keyof ScheduleFormValues>).every((key) => a[key] === b[key]);

export const getBookedCount = (schedule: ISchedule) =>
	schedule.doctorSchedules?.filter((link) => link.isBooked).length ?? 0;

// booked slot-এ appointment আছে → backend update/delete 409 দেয়। আগেই UI-তে জানিয়ে দিই।
export const isScheduleLocked = (schedule: ISchedule) => getBookedCount(schedule) > 0;

// backend-এর loop-এর একই হিসাব: দিন × প্রতিদিনের slot
export const countSlots = (values: ScheduleFormValues) => {
	const parsed = createScheduleSchema.safeParse(values);
	if (!parsed.success) return 0;

	const { startDate, endDate, startTime, endTime } = parsed.data;
	const days =
		Math.round((Date.parse(`${endDate}T00:00:00Z`) - Date.parse(`${startDate}T00:00:00Z`)) / 86_400_000) + 1;
	const slotsPerDay = Math.floor((timeToMinutes(endTime) - timeToMinutes(startTime)) / SCHEDULE_INTERVAL_MINUTES);

	return days * slotsPerDay;
};
```

| Function | কাজ |
|---|---|
| `EMPTY_SCHEDULE_FORM` | create-এর ফাঁকা মান |
| `scheduleToFormValues` | ISO → ঢাকার সময়ের `date`/`time` (ধাপ ২) → edit form |
| `isSameFormValues` | ৪টা field-ই একই? → "No changes" (doctor-এর `getChangedFields`-এর ছোট version — এখানে backend সব field চায়, তাই "কোনটা বদলেছে" লাগে না, শুধু "কিছু বদলেছে কিনা") |
| `getBookedCount` | কয়জন doctor-এর এই slot booked |
| **`isScheduleLocked`** | booked থাকলে `true` → Edit/Delete UI-তেই আটকানো |
| **`countSlots`** | create-এর আগে কয়টা slot হবে |

#### `countSlots` — backend-এর loop-এর একই হিসাব

```
startDate 2026-11-08, endDate 2026-11-10, 09:00 – 12:00
days        = (১০ Nov − ৮ Nov) / ১ দিন + ১ = ৩
slotsPerDay = floor((720 − 540) / 30)       = ৬
total       = ৩ × ৬                          = ১৮
```
- আগে `createScheduleSchema.safeParse` → form ঠিক না থাকলে `0` (অর্ধেক লেখা তারিখে আজেবাজে সংখ্যা দেখায় না)।
- দিন গোনায় `T00:00:00Z` (UTC) → browser-এর timezone বা daylight saving দিনের হিসাব নষ্ট করে না।
- `86_400_000` = ১ দিনের millisecond (`_` শুধু পড়ার সুবিধা)।
- `floor` → শেষের অসম্পূর্ণ ৩০ মিনিট (যেমন ০৯:০০–১০:৪৫ → ৩টা, শেষের ১৫ মিনিট বাদ) — backend-ও `slotEnd <= dayEnd` দিয়ে বাদ দেয়।

### ৮.৩ `schedulesColumns.tsx`

```tsx
import DateCell from "@/components/shared/table/cells/DateCell";
import type { DataTableColumnDef } from "@/components/shared/table/dataTableFeatures";
import { Badge } from "@/components/ui/badge";
import { formatClinicTime, getDurationMinutes } from "@/lib/scheduleTime";
import type { ISchedule } from "@/types/schedule.types";
import { getBookedCount } from "./schedule.utils";

const EMPTY = <span className="text-muted-foreground">—</span>;

// ⚠️ sortable column-এর accessorKey = backend-এর sortBy field (startDateTime, createdAt)
export const scheduleColumns: DataTableColumnDef<ISchedule>[] = [
	{
		accessorKey: "startDateTime",
		header: "Date",
		cell: ({ row }) => <DateCell value={row.original.startDateTime} />,
	},
	{
		id: "time",
		header: "Time",
		enableSorting: false,
		cell: ({ row }) => (
			<span className="whitespace-nowrap tabular-nums">
				{formatClinicTime(row.original.startDateTime)} – {formatClinicTime(row.original.endDateTime)}
			</span>
		),
	},
	{
		id: "duration",
		header: "Duration",
		enableSorting: false,
		cell: ({ row }) => {
			const minutes = getDurationMinutes(row.original.startDateTime, row.original.endDateTime);
			return minutes ? <span className="tabular-nums">{minutes} min</span> : EMPTY;
		},
	},
	{
		id: "doctors",
		header: "Doctors",
		enableSorting: false,
		cell: ({ row }) => {
			const total = row.original.doctorSchedules?.length ?? 0;

			if (total === 0) {
				return <span className="text-xs text-muted-foreground">Not assigned</span>;
			}

			return (
				<Badge variant="secondary">
					{total} {total === 1 ? "doctor" : "doctors"}
				</Badge>
			);
		},
	},
	{
		id: "booked",
		header: "Booked",
		enableSorting: false,
		cell: ({ row }) => {
			const booked = getBookedCount(row.original);
			return booked > 0 ? <Badge>{booked} booked</Badge> : EMPTY;
		},
	},
	{
		accessorKey: "createdAt",
		header: "Created",
		cell: ({ row }) => <DateCell value={row.original.createdAt} />,
	},
];
```

| Column | Sort? | দেখায় |
|---|---|---|
| Date (`startDateTime`) | ✅ | `08 Nov 2026` (`DateCell`, আগেই Asia/Dhaka) |
| Time | ❌ | `09:00 AM – 09:30 AM` |
| Duration | ❌ | `30 min` |
| Doctors | ❌ | `3 doctors` badge / "Not assigned" |
| Booked | ❌ | `1 booked` (ভরাট badge) / `—` |
| Created (`createdAt`) | ✅ | তারিখ |

- Sortable `accessorKey` = backend-এর `scheduleSortableFields` (`startDateTime`, `createdAt`) — note 16-এর নিয়ম।
- `{total === 1 ? "doctor" : "doctors"}` → এক হলে একবচন।

---

## ধাপ ৯: `ScheduleForm.tsx` — create + edit

```tsx
"use client";

import AppField from "@/components/shared/form/AppField";
import AppSubmitButton from "@/components/shared/form/AppSubmitButton";
import { Button } from "@/components/ui/button";
import { SCHEDULE_INTERVAL_MINUTES } from "@/lib/scheduleTime";
import { createScheduleSchema, type ScheduleFormValues, updateScheduleSchema } from "@/zod/schedule.validation";
import { useForm } from "@tanstack/react-form";
import { countSlots } from "./schedule.utils";

interface ScheduleFormProps {
	mode: "create" | "edit";
	defaultValues: ScheduleFormValues;
	onSubmit: (values: ScheduleFormValues) => void;
	onCancel: () => void;
	isPending: boolean;
	submitLabel: string;
	pendingLabel: string;
}

export default function ScheduleForm({
	mode,
	defaultValues,
	onSubmit,
	onCancel,
	isPending,
	submitLabel,
	pendingLabel,
}: ScheduleFormProps) {
	const isCreate = mode === "create";

	// form-level schema: refine (end ≥ start) চলে, error নিজের path-এর field-এ বসে।
	// teacher-এর field-level `.shape.x` দিলে refine কখনো চলত না।
	const form = useForm({
		defaultValues,
		validators: { onChange: isCreate ? createScheduleSchema : updateScheduleSchema },
		onSubmit: ({ value }) => onSubmit(value),
	});

	return (
		<form
			noValidate
			className="space-y-5"
			onSubmit={(event) => {
				event.preventDefault();
				event.stopPropagation();
				void form.handleSubmit();
			}}
		>
			{/* fieldset disabled: save চলার সময় সব input একসাথে বন্ধ */}
			<fieldset disabled={isPending} className="grid gap-4 sm:grid-cols-2">
				<form.Field name="startDate">
					{(field) => <AppField field={field} label="Start date" type="date" />}
				</form.Field>

				<form.Field name="endDate">{(field) => <AppField field={field} label="End date" type="date" />}</form.Field>

				<form.Field name="startTime">
					{(field) => <AppField field={field} label={isCreate ? "Daily start time" : "Start time"} type="time" />}
				</form.Field>

				<form.Field name="endTime">
					{(field) => <AppField field={field} label={isCreate ? "Daily end time" : "End time"} type="time" />}
				</form.Field>
			</fieldset>

			{/* create-এ কয়টা slot হবে আগেই দেখাই */}
			{isCreate && (
				<form.Subscribe selector={(state) => state.values}>
					{(values) => {
						const count = countSlots(values);

						return (
							<p aria-live="polite" className="rounded-md bg-muted px-3 py-2 text-sm text-muted-foreground">
								{count > 0 ? (
									<>
										Up to <strong className="tabular-nums text-foreground">{count}</strong> slots of{" "}
										{SCHEDULE_INTERVAL_MINUTES} minutes will be created. Existing slots are skipped.
									</>
								) : (
									`Pick a date range and a daily time window. Each slot is ${SCHEDULE_INTERVAL_MINUTES} minutes.`
								)}
							</p>
						);
					}}
				</form.Subscribe>
			)}

			<p className="text-xs text-muted-foreground">All times are Bangladesh time (GMT+6).</p>

			<div className="flex justify-end gap-2">
				<Button type="button" variant="outline" onClick={onCancel} disabled={isPending}>
					Cancel
				</Button>
				<AppSubmitButton isPending={isPending} pendingLabel={pendingLabel} className="w-auto">
					{submitLabel}
				</AppSubmitButton>
			</div>
		</form>
	);
}
```

### Doctor form (note 20)-এর মতো
- `mode` → schema বদলায়, label বদলায় ("Daily start time" বনাম "Start time")।
- `fieldset disabled`, `noValidate`, `void form.handleSubmit()`, `AppSubmitButton className="w-auto"`।

### Form-level schema কেন (field-level না)?
```ts
validators: { onChange: isCreate ? createScheduleSchema : updateScheduleSchema }
```
- `refine` (end ≥ start) **দুটো field মিলিয়ে** check করে — এটা শুধু পুরো object-এর schema-তে চলে।
- Teacher field-level দিয়েছিল (`validators: { onChange: schema.shape.startDate }`) → প্রতিটা field আলাদা check, `refine` **কখনো চলত না** → end-এর আগে start দিলেও form পাস।
- Form-level-এ `refine`-এর `path: ["endTime"]` থাকায় error ঠিক `endTime` field-এর নিচে বসে।

### `<form.Subscribe>` — কয়টা slot হবে (live)

```tsx
<form.Subscribe selector={(state) => state.values}>
	{(values) => <p>Up to {countSlots(values)} slots ...</p>}
</form.Subscribe>
```
- **`form.Subscribe`** → form-এর state-এর একটা অংশ শুনে **শুধু এই ছোট অংশটা** re-render করে (পুরো form না)।
- `selector={(state) => state.values}` → সব মান।
- **"Up to"** → আগে থেকে থাকা slot বাদ পড়বে (`skipDuplicates`), তাই আসল সংখ্যা কম হতে পারে।
- `aria-live="polite"` → সংখ্যা বদলালে screen reader পড়ে শোনায়।

### "All times are Bangladesh time (GMT+6)."
Admin দেশের বাইরে থাকলেও বুঝবে, input-এ যা লিখছে সেটা ঢাকার সময়।

---

## ধাপ ১০: Dialog — ৪টা

### ১০.১ `CreateScheduleDialog.tsx`

```tsx
"use client";

import { createScheduleAction } from "@/actions/schedule.actions";
import { Button } from "@/components/ui/button";
import {
	Dialog,
	DialogContent,
	DialogDescription,
	DialogHeader,
	DialogTitle,
	DialogTrigger,
} from "@/components/ui/dialog";
import { useActionMutation } from "@/hooks/useActionMutation";
import { SCHEDULE_INTERVAL_MINUTES } from "@/lib/scheduleTime";
import { SCHEDULES_QUERY_KEY } from "@/queries/schedule.queries";
import type { ScheduleFormValues } from "@/zod/schedule.validation";
import { Plus } from "lucide-react";
import { useState } from "react";
import { EMPTY_SCHEDULE_FORM } from "./schedule.utils";
import ScheduleForm from "./ScheduleForm";

export default function CreateScheduleDialog() {
	const [open, setOpen] = useState(false);

	const { mutate, isPending } = useActionMutation({
		action: (payload: ScheduleFormValues) => createScheduleAction(payload),
		invalidateKey: SCHEDULES_QUERY_KEY,
		// skipDuplicates: সব slot আগে থেকেই থাকলে ফাঁকা array আসে
		successMessage: (created) =>
			created.length > 0
				? `${created.length} schedule slots created`
				: "These slots already exist. Nothing new was created.",
		onSuccess: () => setOpen(false), // dialog বন্ধ হলে form unmount হয় → পরের বার ফাঁকা form
	});

	return (
		<Dialog open={open} onOpenChange={(next) => !isPending && setOpen(next)}>
			<DialogTrigger asChild>
				<Button>
					<Plus />
					Create schedule
				</Button>
			</DialogTrigger>

			<DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-lg">
				<DialogHeader>
					<DialogTitle>Create schedule</DialogTitle>
					<DialogDescription>
						Generate {SCHEDULE_INTERVAL_MINUTES}-minute slots for every day in a date range.
					</DialogDescription>
				</DialogHeader>

				<ScheduleForm
					mode="create"
					defaultValues={EMPTY_SCHEDULE_FORM}
					onSubmit={(values) => mutate(values)}
					onCancel={() => setOpen(false)}
					isPending={isPending}
					submitLabel="Create slots"
					pendingLabel="Creating..."
				/>
			</DialogContent>
		</Dialog>
	);
}
```

- Doctor-এর Create dialog (note 20)-এর মতো: নিজের `useState`, `DialogTrigger`, বন্ধ হলে form unmount → পরেরবার ফাঁকা।
- **`successMessage` function** → `created.length` (backend শুধু **নতুনগুলো** ফেরত দেয়):
  - `> 0` → `"18 schedule slots created"`
  - `0` → `"These slots already exist. Nothing new was created."` (error না — সফল, শুধু নতুন কিছু হয়নি)
- `onSubmit={(values) => mutate(values)}` → form-এর মানই payload (রূপান্তর লাগে না, ধাপ ৩)।

### ১০.২ `EditScheduleDialog.tsx`

```tsx
"use client";

import { updateScheduleAction } from "@/actions/schedule.actions";
import { Alert, AlertDescription, AlertTitle } from "@/components/ui/alert";
import { Dialog, DialogContent, DialogDescription, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { useActionMutation } from "@/hooks/useActionMutation";
import { formatClinicDateTime } from "@/lib/scheduleTime";
import { SCHEDULES_QUERY_KEY } from "@/queries/schedule.queries";
import type { ISchedule } from "@/types/schedule.types";
import type { ScheduleFormValues } from "@/zod/schedule.validation";
import { CalendarX } from "lucide-react";
import { useMemo } from "react";
import { toast } from "sonner";
import { isSameFormValues, isScheduleLocked, scheduleToFormValues } from "./schedule.utils";
import ScheduleForm from "./ScheduleForm";

interface EditScheduleDialogProps {
	row: ISchedule | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

export default function EditScheduleDialog({ row: schedule, open, onOpenChange }: EditScheduleDialogProps) {
	const { mutate, isPending } = useActionMutation({
		action: ({ id, payload }: { id: string; payload: ScheduleFormValues }) => updateScheduleAction(id, payload),
		invalidateKey: SCHEDULES_QUERY_KEY,
		successMessage: (updated) => `Schedule moved to ${formatClinicDateTime(updated.startDateTime)}`,
		onSuccess: () => onOpenChange(false),
	});

	// useMemo: প্রতি render-এ নতুন object দিলে form নিজেকে reset করতে পারে
	const defaultValues = useMemo(() => (schedule ? scheduleToFormValues(schedule) : null), [schedule]);
	const isLocked = schedule ? isScheduleLocked(schedule) : false;

	const handleSubmit = (values: ScheduleFormValues) => {
		if (!schedule || !defaultValues) return;

		if (isSameFormValues(values, defaultValues)) {
			toast.info("No changes to save");
			return;
		}

		mutate({ id: schedule.id, payload: values });
	};

	return (
		<Dialog open={open} onOpenChange={(next) => !isPending && onOpenChange(next)}>
			<DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-lg">
				<DialogHeader>
					<DialogTitle>Edit schedule</DialogTitle>
					<DialogDescription>Change the start and end of this slot.</DialogDescription>
				</DialogHeader>

				{isLocked && (
					<Alert variant="destructive">
						<CalendarX />
						<AlertTitle>This slot is booked</AlertTitle>
						<AlertDescription>
							A patient already has an appointment in this slot, so its time can&apos;t be changed.
						</AlertDescription>
					</Alert>
				)}

				{schedule && defaultValues && !isLocked && (
					// key: অন্য schedule খুললে form একদম নতুন করে শুরু হয়
					<ScheduleForm
						key={schedule.id}
						mode="edit"
						defaultValues={defaultValues}
						onSubmit={handleSubmit}
						onCancel={() => onOpenChange(false)}
						isPending={isPending}
						submitLabel="Save changes"
						pendingLabel="Saving..."
					/>
				)}
			</DialogContent>
		</Dialog>
	);
}
```

- **`isLocked` হলে form-ই দেখায় না** → লাল `Alert`: "This slot is booked" — backend 409 দেওয়ার আগেই admin বুঝে যায়।
- `isSameFormValues` → কিছু না বদলালে "No changes to save" (request যায় না)।
- `successMessage` → `"Schedule moved to 09 Nov 2026, 10:00 AM"` (ফেরত পাওয়া **নতুন** সময়)।
- `useMemo` + `key={schedule.id}` → note 19-এর একই কারণ।

### ১০.৩ `DeleteScheduleDialog.tsx`

```tsx
"use client";

import { deleteScheduleAction } from "@/actions/schedule.actions";
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
import { formatClinicDateTime, formatClinicTime } from "@/lib/scheduleTime";
import { SCHEDULES_QUERY_KEY } from "@/queries/schedule.queries";
import type { ISchedule } from "@/types/schedule.types";
import { LoaderCircle, Trash2 } from "lucide-react";
import { isScheduleLocked } from "./schedule.utils";

interface DeleteScheduleDialogProps {
	row: ISchedule | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

export default function DeleteScheduleDialog({ row: schedule, open, onOpenChange }: DeleteScheduleDialogProps) {
	const { mutate, isPending } = useActionMutation({
		action: deleteScheduleAction,
		invalidateKey: SCHEDULES_QUERY_KEY,
		successMessage: "Schedule deleted",
		onSuccess: () => onOpenChange(false),
	});

	const isLocked = schedule ? isScheduleLocked(schedule) : false;
	const doctorCount = schedule?.doctorSchedules?.length ?? 0;

	return (
		<AlertDialog open={open} onOpenChange={(next) => !isPending && onOpenChange(next)}>
			<AlertDialogContent>
				<AlertDialogHeader>
					<AlertDialogTitle>{isLocked ? "Can't delete this slot" : "Delete this schedule?"}</AlertDialogTitle>
					<AlertDialogDescription>
						<strong className="text-foreground">
							{formatClinicDateTime(schedule?.startDateTime)} – {formatClinicTime(schedule?.endDateTime)}
						</strong>{" "}
						{isLocked
							? "has a booked appointment, so it can't be removed."
							: doctorCount > 0
								? `will be removed. ${doctorCount} assigned ${doctorCount === 1 ? "doctor" : "doctors"} will lose this slot.`
								: "will be removed."}
					</AlertDialogDescription>
				</AlertDialogHeader>

				<AlertDialogFooter>
					<AlertDialogCancel disabled={isPending}>{isLocked ? "Close" : "Cancel"}</AlertDialogCancel>
					{/* AlertDialogAction না: ওটা click করলেই dialog বন্ধ করে দেয়, delete শেষ হওয়ার আগেই */}
					{!isLocked && (
						<Button
							variant="destructive"
							disabled={isPending || !schedule}
							onClick={() => schedule && mutate(schedule.id)}
						>
							{isPending ? <LoaderCircle className="animate-spin" /> : <Trash2 />}
							{isPending ? "Deleting..." : "Delete"}
						</Button>
					)}
				</AlertDialogFooter>
			</AlertDialogContent>
		</AlertDialog>
	);
}
```

| অবস্থা | Title | Description | Button |
|---|---|---|---|
| Booked | "Can't delete this slot" | "…has a booked appointment, so it can't be removed." | শুধু **Close** |
| Doctor আছে, booked না | "Delete this schedule?" | "…3 assigned doctors will lose this slot." | Cancel + Delete |
| কেউ নেয়নি | "Delete this schedule?" | "…will be removed." | Cancel + Delete |

- Admin মোছার আগে জানে কয়জন doctor-এর slot চলে যাবে।
- `action: deleteScheduleAction` সরাসরি (একটা argument, তাই wrapper লাগে না)।
- Nested ternary (`isLocked ? … : doctorCount > 0 ? … : …`) → ৩টা অবস্থা।
- `AlertDialogAction` না, নিজের `Button` → note 18-এর একই কারণ।

### ১০.৪ `ViewScheduleDialog.tsx`

```tsx
"use client";

import DateCell from "@/components/shared/table/cells/DateCell";
import UserInfoCell from "@/components/shared/table/cells/UserInfoCell";
import { Badge } from "@/components/ui/badge";
import { Dialog, DialogContent, DialogDescription, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Separator } from "@/components/ui/separator";
import { formatClinicTime, getDurationMinutes } from "@/lib/scheduleTime";
import type { ISchedule } from "@/types/schedule.types";
import type { ReactNode } from "react";
import { getBookedCount } from "./schedule.utils";

interface ViewScheduleDialogProps {
	row: ISchedule | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

const DetailItem = ({ label, children }: { label: string; children: ReactNode }) => (
	<div>
		<dt className="text-xs text-muted-foreground">{label}</dt>
		<dd className="mt-0.5 text-sm font-medium">{children}</dd>
	</div>
);

// row-এ দরকারি সব data আগেই আছে (include=doctorSchedules), তাই teacher-এর মতো আলাদা getById request লাগে না
export default function ViewScheduleDialog({ row: schedule, open, onOpenChange }: ViewScheduleDialogProps) {
	const links = schedule?.doctorSchedules ?? [];
	const duration = schedule ? getDurationMinutes(schedule.startDateTime, schedule.endDateTime) : null;

	return (
		<Dialog open={open} onOpenChange={onOpenChange}>
			<DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-lg">
				{schedule && (
					<>
						<DialogHeader>
							<DialogTitle>Schedule details</DialogTitle>
							<DialogDescription>
								{formatClinicTime(schedule.startDateTime)} – {formatClinicTime(schedule.endDateTime)} (Bangladesh time)
							</DialogDescription>
						</DialogHeader>

						<dl className="grid grid-cols-2 gap-x-6 gap-y-4">
							<DetailItem label="Date">
								<DateCell value={schedule.startDateTime} />
							</DetailItem>
							<DetailItem label="Duration">{duration ? `${duration} minutes` : "—"}</DetailItem>
							<DetailItem label="Doctors assigned">{links.length}</DetailItem>
							<DetailItem label="Booked">{getBookedCount(schedule)}</DetailItem>
							<DetailItem label="Created">
								<DateCell value={schedule.createdAt} />
							</DetailItem>
						</dl>

						<Separator />

						<section className="space-y-3">
							<h3 className="text-sm font-semibold">Assigned doctors</h3>

							{links.length === 0 ? (
								<p className="text-sm text-muted-foreground">No doctor has picked this slot yet.</p>
							) : (
								<ul className="space-y-2">
									{links.map(({ doctor, doctorId, isBooked }) => (
										<li key={doctorId} className="flex items-center justify-between gap-3 rounded-md border p-2">
											<UserInfoCell name={doctor.name} email={doctor.email} image={doctor.profilePhoto} />
											<Badge variant={isBooked ? "default" : "outline"}>{isBooked ? "Booked" : "Available"}</Badge>
										</li>
									))}
								</ul>
							)}
						</section>
					</>
				)}
			</DialogContent>
		</Dialog>
	);
}
```

- **আলাদা request লাগে না** → row-এ আগেই `doctorSchedules` + `doctor` আছে (`include`, ধাপ ৮.১)। Teacher View খুললে `getById` call করত।
- Doctor-দের list → `UserInfoCell` (note 16) + "Booked"/"Available" badge।
- `<dl>`/`<dt>`/`<dd>` + `DetailItem` → note 18-এর মতো।

---

## ধাপ ১১: `SchedulesTable.tsx`

```tsx
"use client";

import DataTable from "@/components/shared/table/DataTable";
import DataTablePagination from "@/components/shared/table/DataTablePagination";
import DataTableDateRangeFilter from "@/components/shared/table/filters/DataTableDateRangeFilter";
import { Button } from "@/components/ui/button";
import { useRowAction } from "@/hooks/useRowAction";
import { useTableUrlState } from "@/hooks/useTableUrlState";
import { buildQueryString } from "@/lib/buildQueryString";
import { schedulesQueryOptions } from "@/queries/schedule.queries";
import type { ISchedule } from "@/types/schedule.types";
import { keepPreviousData, useQuery } from "@tanstack/react-query";
import { X } from "lucide-react";
import { useEffect } from "react";
import CreateScheduleDialog from "./CreateScheduleDialog";
import DeleteScheduleDialog from "./DeleteScheduleDialog";
import EditScheduleDialog from "./EditScheduleDialog";
import { scheduleColumns } from "./schedulesColumns";
import {
	END_DATE_KEY,
	SCHEDULE_FILTER_KEYS,
	SCHEDULE_TABLE_DEFAULTS,
	SCHEDULE_TABLE_QUERY_KEYS,
	START_DATE_KEY,
} from "./schedulesTable.config";
import ViewScheduleDialog from "./ViewScheduleDialog";

const getScheduleId = (schedule: ISchedule) => schedule.id;

export default function SchedulesTable() {
	const table = useTableUrlState({ defaultLimit: Number(SCHEDULE_TABLE_DEFAULTS.limit) });
	const rowAction = useRowAction<ISchedule>();

	// server page-এর একই নিয়ম → একই cache key, প্রথমবার আবার fetch হয় না
	const queryString = buildQueryString(table.searchParams, SCHEDULE_TABLE_QUERY_KEYS, SCHEDULE_TABLE_DEFAULTS);

	const {
		data: response,
		isPending,
		isFetching,
		isError,
	} = useQuery({
		...schedulesQueryOptions(queryString),
		placeholderData: keepPreviousData,
	});

	// শেষ page-এর শেষ slot delete হলে আগের page-এ ফেরা
	const meta = response?.meta;
	const { setPage } = table;
	useEffect(() => {
		if (meta && meta.totalPages > 0 && meta.page > meta.totalPages) setPage(meta.totalPages);
	}, [meta, setPage]);

	const hasActiveFilters = SCHEDULE_FILTER_KEYS.some((key) => table.searchParams.has(key));

	if (isError && !response) {
		return <p className="text-sm text-destructive">Failed to load schedules.</p>;
	}

	return (
		<div className="space-y-4">
			{/* Toolbar: date filter + create */}
			<div className="flex flex-wrap items-center justify-between gap-3">
				<div className="flex flex-wrap items-center gap-2">
					<DataTableDateRangeFilter
						label="Date"
						from={table.getFilter(START_DATE_KEY)}
						to={table.getFilter(END_DATE_KEY)}
						onChange={({ from, to }) => table.setFilters({ [START_DATE_KEY]: from, [END_DATE_KEY]: to })}
					/>

					{hasActiveFilters && (
						<Button variant="ghost" onClick={() => table.clearParams(SCHEDULE_FILTER_KEYS)}>
							<X />
							Reset
						</Button>
					)}
				</div>

				<CreateScheduleDialog />
			</div>

			<DataTable
				data={response?.data ?? []}
				columns={scheduleColumns}
				getRowId={getScheduleId}
				isLoading={isPending}
				isFetching={isFetching}
				emptyMessage={hasActiveFilters ? "No schedules in this date range." : "No schedules yet."}
				sorting={{ state: table.sorting, onSortingChange: table.setSorting }}
				actions={rowAction.actions}
			/>

			<DataTablePagination meta={response?.meta} onPageChange={table.setPage} onLimitChange={table.setLimit} />

			<ViewScheduleDialog {...rowAction.dialogProps("view")} />
			<EditScheduleDialog {...rowAction.dialogProps("edit")} />
			<DeleteScheduleDialog {...rowAction.dialogProps("delete")} />
		</div>
	);
}
```

Doctor table (note 20)-এর সাথে পার্থক্য:

| | Doctor | Schedule |
|---|---|---|
| Search | ✅ | ❌ (backend-এ searchable নেই) |
| Filter | Gender, Specialty, Fee | শুধু **Date range** |
| আলাদা `useQuery` (specialty) | ✅ | ❌ |
| Toolbar | `lg:flex-row` | `flex-wrap justify-between` (জিনিস কম) |
| Create button | filter-এর পাশে | ডান কোণে আলাদা |

বাকি সব একই: `useTableUrlState`, `buildQueryString`, `keepPreviousData`, `useRowAction`, শেষ page-এর `useEffect`, ৩টা dialog।

---

## ধাপ ১২: `src/app/(dashboardLayout)/admin/dashboard/schedules-management/page.tsx`

```tsx
import SchedulesTable from "@/components/modules/Admin/SchedulesManagement/SchedulesTable";
import {
	SCHEDULE_TABLE_DEFAULTS,
	SCHEDULE_TABLE_QUERY_KEYS,
} from "@/components/modules/Admin/SchedulesManagement/schedulesTable.config";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { buildQueryString } from "@/lib/buildQueryString";
import { getQueryClient } from "@/lib/getQueryClient";
import { schedulesQueryOptions } from "@/queries/schedule.queries";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import { Suspense } from "react";

type SearchParams = Record<string, string | string[] | undefined>;

async function SchedulesSection({ searchParams }: { searchParams: Promise<SearchParams> }) {
	const queryString = buildQueryString(await searchParams, SCHEDULE_TABLE_QUERY_KEYS, SCHEDULE_TABLE_DEFAULTS);
	const queryClient = getQueryClient();

	await queryClient.query(schedulesQueryOptions(queryString)).catch(noop);

	return (
		<HydrationBoundary state={dehydrate(queryClient)}>
			<SchedulesTable />
		</HydrationBoundary>
	);
}

export default function SchedulesManagementPage({ searchParams }: PageProps<"/admin/dashboard/schedules-management">) {
	return (
		<div className="space-y-6">
			<div>
				<h1 className="text-2xl font-semibold tracking-tight">Schedules Management</h1>
				<p className="text-sm text-muted-foreground">Create and manage 30-minute time slots doctors can pick</p>
			</div>

			<Suspense fallback={<LoadingSpinner text="Loading schedules..." />}>
				<SchedulesSection searchParams={searchParams} />
			</Suspense>
		</div>
	);
}
```

- Doctor page (note 16)-এর মতো, শুধু একটা prefetch (schedule)।
- Sidebar-এর "Schedules" link (note 09: `/admin/dashboard/schedules-management`) এখন কাজ করে।
- `PageProps<"/admin/dashboard/schedules-management">` লাল দেখালে `pnpm dev` restart (note 16)।

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | Sidebar → Schedules | table: Date, Time, Duration, Doctors, Booked, Created |
| ২ | Create → ৮–১০ Nov, ০৯:০০–১২:০০ | নিচে "Up to **18** slots of 30 minutes…" |
| ৩ | Create slots | "18 schedule slots created", table-এ নতুন slot |
| ৪ | একই জিনিস আবার create | "These slots already exist. Nothing new was created." |
| ৫ | End date < start date | "End date can't be before start date" |
| ৬ | ০৯:০০–০৯:২০ | "End time must be at least 30 minutes after start time" |
| ৭ | Time column | ঢাকার সময় (`09:00 AM`) — computer-এর timezone বদলে reload করলেও একই |
| ৮ | Edit → ১ ঘণ্টা পরে সরাও | "Schedule moved to …" |
| ৯ | Edit-এ কিছু না বদলে Save | "No changes to save" |
| ১০ | Booked slot-এ Edit | লাল Alert, form নেই |
| ১১ | Booked slot-এ Delete | "Can't delete this slot", শুধু Close |
| ১২ | Doctor নেওয়া slot-এ Delete | "2 assigned doctors will lose this slot." |
| ১৩ | View | doctor list + Booked/Available |
| ১৪ | Date filter ১–৩০ Nov | badge `2026-11-01 → 2026-11-30`, শুধু ওই দিনের slot |
| ১৫ | "Date" header চাপো | sort asc/desc |
| ১৬ | URL-এ `&include=appointments` লেখো | বাদ পড়ে, Doctors column ঠিক থাকে |

---

## 🔜 পরে যা যোগ হবে
- [x] Doctor dashboard: নিজের slot বাছাই — note 22 · (`/doctor-schedules/create-my-doctor-schedule`, server note 26)
- [ ] Patient: doctor-এর খালি slot দেখে appointment
- [ ] Appointment / payment table-এ একই `DataTableDateRangeFilter`

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **অতীতের তারিখে slot বানানো যায়** — frontend বা backend কেউ আটকায় না, অথচ doctor অতীতের slot নিতে পারে না (server note 26-এর `ensureSchedulesAvailable` ভবিষ্যৎ ছাড়া reject করে)। ফলে অকেজো slot জমে। Create-এ আজকের (ঢাকার) তারিখের আগের দিন আটকাও:
   ```ts
   // scheduleTime.ts
   export const getClinicToday = () => toClinicDateTimeParts(new Date().toISOString()).date;

   // schedule.validation.ts → createScheduleSchema-এ আরেকটা refine
   .refine((data) => data.startDate >= getClinicToday(), {
   	message: "Start date can't be in the past",
   	path: ["startDate"],
   })
   ```
   আর form-এ `<AppField type="date" min={getClinicToday()} />`। Backend-এও একই check রাখা ভালো (Server Action-কে বাইপাস করা যায় না, কিন্তু backend সরাসরি call করা যায়)।

2. **খুব বড় range-এ হাজার হাজার slot** — ১ বছর × প্রতিদিন ২৪ ঘণ্টা = প্রায় ১৭,৫০০ slot এক request-এ। ভুল করে হয়ে গেলে মুছতেও কষ্ট। `countSlots` তো আছেই — একটা সীমা দাও (যেমন ৫০০টা বা ৩১ দিন):
   ```ts
   .refine((data) => countDays(data) <= 31, { message: "Pick at most 31 days at a time", path: ["endDate"] })
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Lock-এর শর্ত frontend আর backend-এ আলাদা জিনিস দেখে** — frontend দেখে `doctorSchedules[].isBooked`, backend দেখে `appointments`-এর সংখ্যা। Booking-এর সময় দুটোই একসাথে বদলায়, তাই এখন মিলবে। কখনো না মিললে (যেমন appointment cancel-এ `isBooked` false না হলে) UI এক কথা বলবে আর backend আরেক — তখন backend-এর 409 message-ই toast-এ আসবে, তাই ক্ষতি নেই।

4. **রাত পার হওয়া window (যেমন ২২:০০–০২:০০) create-এ চলে না** — `endTime > startTime` দরকার। Clinic রাতে খোলা থাকলে দুবারে বানাতে হবে (২২:০০–২৩:৫৯ আর পরের দিন ০০:০০–০২:০০)। এখনকার জন্য ঠিক আছে।
