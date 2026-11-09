# 22 - Doctor: My Schedules (Admin-এর Slot থেকে নিজের Slot বাছাই)

> Doctor-এর `/doctor/dashboard/my-schedules` page। Admin যে ৩০ মিনিটের slot বানিয়েছে (note 21), doctor সেখান থেকে **নিজের সময় বেছে নেয়** — যখন রোগী তাকে book করতে পারবে। নিজের নেওয়া slot-এর table, filter (Status + Date), আর slot সরানো।
>
> Backend: server note 26 (`/doctor-schedules/...`, সব route `checkAuth(DOCTOR)`)।

## 🧭 এক নজরে

```
page.tsx (Server) → prefetch GET /doctor-schedules/my-doctor-schedules?page=1&limit=10
  → DoctorSchedulesTable ("use client")
       ├─ Toolbar: [Status ▾] [📅 Date] [✕ Reset]                         [📅+ Add slots]
       ├─ DataTable: Date | Time | Duration | Status | Added on | ⋯ (শুধু Remove)
       └─ Pagination

"Add slots" → একটা দিন বাছো
  → GET /schedules?startDate=D&endDate=D&limit=100        (admin-এর ওই দিনের সব slot)
  → GET /doctor-schedules/my-doctor-schedules?...D...      (ওই দিনে আমার নেওয়াগুলো)
  → grid: [09:00 ✓ Added] [09:30] [10:00 ✓selected] ...    (অতীতের slot দেখায়ই না)
  → "Add 3 slots" → POST /doctor-schedules/create-my-doctor-schedule { scheduleIds }
  → invalidate ["my-doctor-schedules"] → table + dialog-এর query দুটোই নতুন

Remove → booked হলে আটকানো, না হলে DELETE /doctor-schedules/delete-my-doctor-schedule/:scheduleId
```

### Admin (note 21) বনাম Doctor (এই নোট)

| | Admin — Schedules | Doctor — My Schedules |
|---|---|---|
| Table-এর row | `ISchedule` (slot নিজেই) | `IDoctorSchedule` (আমি ↔ slot-এর join) |
| Row id | `schedule.id` | `scheduleId` (doctorId সবার একই) |
| Create | date range + time window → নতুন slot **বানায়** | আগের slot থেকে **বাছে** |
| Edit | ✅ | ❌ (শুধু Remove) |
| Date filter backend-এ | `?startDate=&endDate=` (backend নিজে handle করে) | `?schedule.startDateTime[gte]=&[lt]=` (frontend ISO বানায়) |
| Status filter | ❌ | ✅ Available / Booked |

---

## ধাপ ০: ফাইলের তালিকা

```
src/lib/scheduleTime.ts                    ← ৩টা function যোগ
src/types/doctorSchedule.types.ts
src/zod/doctorSchedule.validation.ts
src/services/doctorSchedule.services.ts
src/actions/doctorSchedule.actions.ts
src/queries/doctorSchedule.queries.ts
src/components/modules/Doctor/DoctorSchedules/
  ├─ doctorSchedulesTable.config.ts
  ├─ doctorSchedulesColumns.tsx
  ├─ AddSchedulesDialog.tsx
  ├─ RemoveScheduleDialog.tsx
  └─ DoctorSchedulesTable.tsx
src/app/(dashboardLayout)/doctor/dashboard/my-schedules/page.tsx
```

> নতুন shadcn component লাগে না। Shared component আগের: `DataTable`, `DataTablePagination`, `DataTableSelectFilter` (note 17), `DataTableDateRangeFilter` (note 21)।

---

## ধাপ ১: `src/lib/scheduleTime.ts` — file-এর শেষে যোগ করো

```ts
const CLINIC_UTC_OFFSET = "+06:00";

// ঢাকার আজকের তারিখ "YYYY-MM-DD" (browser যেখানেই থাকুক)
export const getClinicToday = () => toClinicDateTimeParts(new Date().toISOString()).date;

// "2026-11-08" → ঢাকার ওই দিনের রাত 12টা, ISO string।
// Prisma DateTime শুধু তারিখ ("2026-11-08") নেয় না, পুরো ISO লাগে।
export const clinicDayStartIso = (date: string) => new Date(`${date}T00:00:00${CLINIC_UTC_OFFSET}`).toISOString();

// "2026-11-08" → "2026-11-09" (UTC-তে হিসাব, তাই timezone-এর প্রভাব নেই)
export const addDaysToDateString = (date: string, days: number) => {
	const value = new Date(`${date}T00:00:00Z`);
	value.setUTCDate(value.getUTCDate() + days);
	return value.toISOString().slice(0, 10);
};
```

| Function | Input → Output | কেন |
|---|---|---|
| `getClinicToday()` | — → `"2026-11-08"` | ঢাকার আজকের তারিখ (রাত ১টায় লন্ডন থেকে খুললেও ঢাকার তারিখ)। Date picker-এর `min` আর default |
| `clinicDayStartIso(date)` | `"2026-11-08"` → `"2026-11-07T18:00:00.000Z"` | ঢাকার ৮ Nov রাত ১২টা = UTC-তে ৭ Nov সন্ধ্যা ৬টা |
| `addDaysToDateString(date, n)` | `"2026-11-08", 1` → `"2026-11-09"` | পরের দিন (মাস/বছর পার হলেও ঠিক) |

### `clinicDayStartIso` কেন লাগে?
- Admin-এর `/schedules`-এ backend নিজে `startDate`/`endDate`-কে ঢাকার দিনে বদলায় (note 21)।
- কিন্তু `/doctor-schedules/my-doctor-schedules`-এ এমন কিছু নেই — সাধারণ QueryBuilder filter: `schedule.startDateTime[gte]=...`। আর **Prisma `DateTime` শুধু `"2026-11-08"` নেয় না, পুরো ISO লাগে**।
- তাই frontend নিজে ঢাকার দিনের শুরুর ISO বানায়: `"2026-11-08T00:00:00+06:00"` → `toISOString()` → UTC।

### `addDaysToDateString` — UTC-তে কেন?
`T00:00:00Z` দিয়ে UTC-তে বানিয়ে `setUTCDate` → browser-এর timezone আর daylight saving দিনের হিসাবে ঢোকে না। `.slice(0, 10)` → ISO-র প্রথম ১০ অক্ষর `"2026-11-09"`। (Backend-এর `schedule.utils.ts`-এর একই নামের function-এর মতো।)

- `CLINIC_UTC_OFFSET` export করা না → শুধু এই file-এর ভিতরে লাগে।

---

## ধাপ ২: `src/types/doctorSchedule.types.ts`

```ts
import type { ISchedule } from "./schedule.types";

// backend getMyDoctorSchedules → include { schedule: true }
export interface IDoctorSchedule {
	doctorId: string;
	scheduleId: string;
	isBooked: boolean;
	createdAt: string;
	updatedAt: string;
	schedule: Omit<ISchedule, "doctorSchedules">;
}

export interface ICreateDoctorSchedulePayload {
	scheduleIds: string[];
}
```

- `IDoctorSchedule` = backend-এর `DoctorSchedules` row + `include: { schedule: true }` (server note 26-এর `getMyDoctorSchedules`)।
- **`Omit<ISchedule, "doctorSchedules">`** → `ISchedule`-এর (note 21) সব field, শুধু `doctorSchedules` বাদ (এখানে nested-এর ভিতরে আবার nested আসে না)। নতুন করে সব field লিখতে হয় না।
- `ICreateDoctorSchedulePayload` → `{ scheduleIds: string[] }` (backend body)।

---

## ধাপ ৩: `src/zod/doctorSchedule.validation.ts`

```ts
import * as z from "zod";

// server action-এ ব্যবহার হয় (form নেই, doctor button চেপে slot বাছে)
export const createDoctorSchedulePayloadSchema = z.object({
	scheduleIds: z
		.array(z.uuid("Invalid schedule id"))
		.min(1, "Select at least one slot")
		.max(100, "You can add at most 100 slots at once"),
});
```

- **Form নেই** — doctor button চেপে slot বাছে, তাই শুধু Server Action-এর জন্য schema।
- `z.uuid()` প্রতিটা id, অন্তত ১টা, সর্বোচ্চ ১০০টা (একদিনে সর্বোচ্চ ৪৮টা slot — ২৪ ঘণ্টা × ২, তাই ১০০ যথেষ্ট; কেউ হাজারটা id পাঠিয়ে backend চাপে ফেলতে পারবে না)।

---

## ধাপ ৪: `src/services/doctorSchedule.services.ts`

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { ICreateDoctorSchedulePayload, IDoctorSchedule } from "@/types/doctorSchedule.types";

// সব route checkAuth(DOCTOR): logged-in doctor নিজেরটাই দেখে/বদলায়

export const getMyDoctorSchedules = async (queryString: string) =>
	httpClient.get<IDoctorSchedule[]>(
		queryString ? `/doctor-schedules/my-doctor-schedules?${queryString}` : "/doctor-schedules/my-doctor-schedules",
	);

// skipDuplicates: আগে নেওয়া slot আবার পাঠালে error না
export const createMyDoctorSchedules = async (payload: ICreateDoctorSchedulePayload) =>
	httpClient.post<IDoctorSchedule[]>("/doctor-schedules/create-my-doctor-schedule", payload);

export const deleteMyDoctorSchedule = async (scheduleId: string) =>
	httpClient.delete<null>(`/doctor-schedules/delete-my-doctor-schedule/${encodeURIComponent(scheduleId)}`);
```

| Function | Backend | নোট |
|---|---|---|
| `getMyDoctorSchedules` | `GET /my-doctor-schedules` | logged-in doctor-এর নিজেরগুলো (backend cookie থেকে doctor চেনে, note 15) |
| `createMyDoctorSchedules` | `POST /create-my-doctor-schedule` | `skipDuplicates` → আগে নেওয়া slot আবার পাঠালে error না |
| `deleteMyDoctorSchedule` | `DELETE /delete-my-doctor-schedule/:scheduleId` | `data` ফেরত দেয় না → `<null>` |

> ⚠️ Delete route-এর param `:scheduleId` — backend controller-এ `getParam(req, "scheduleId")` (server note 26-এর bug fix)। এই fix না থাকলে এখান থেকে delete সবসময় `400` দিত।

## ধাপ ৫: `src/actions/doctorSchedule.actions.ts`

```ts
"use server";

import { getActionErrorMessage } from "@/lib/getActionErrorMessage";
import { createMyDoctorSchedules, deleteMyDoctorSchedule } from "@/services/doctorSchedule.services";
import type { ActionResult } from "@/types/api.types";
import type { IDoctorSchedule } from "@/types/doctorSchedule.types";
import { createDoctorSchedulePayloadSchema } from "@/zod/doctorSchedule.validation";
import * as z from "zod";

const scheduleIdSchema = z.uuid("Invalid schedule id");

const firstIssue = (error: z.ZodError) => error.issues[0]?.message ?? "Invalid input";

export async function createMyDoctorSchedulesAction(payload: unknown): Promise<ActionResult<IDoctorSchedule[]>> {
	const parsed = createDoctorSchedulePayloadSchema.safeParse(payload);
	if (!parsed.success) {
		return { success: false, message: firstIssue(parsed.error) };
	}

	try {
		return await createMyDoctorSchedules(parsed.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "createMyDoctorSchedulesAction") };
	}
}

export async function deleteMyDoctorScheduleAction(scheduleId: unknown): Promise<ActionResult<unknown>> {
	const parsed = scheduleIdSchema.safeParse(scheduleId);
	if (!parsed.success) {
		return { success: false, message: firstIssue(parsed.error) };
	}

	try {
		return await deleteMyDoctorSchedule(parsed.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "deleteMyDoctorScheduleAction") };
	}
}
```

- আগের সব action-এর একই pattern (note 18, 20, 21)।
- Doctor নয় এমন কেউ call করলে backend `checkAuth(DOCTOR)` → 403 → toast।

## ধাপ ৬: `src/queries/doctorSchedule.queries.ts`

```ts
import { getMyDoctorSchedules } from "@/services/doctorSchedule.services";
import { queryOptions } from "@tanstack/react-query";

// invalidate করলে table আর "Add slots" dialog-এর day-query দুটোই আবার আসে
export const MY_DOCTOR_SCHEDULES_QUERY_KEY = ["my-doctor-schedules"] as const;

export const myDoctorSchedulesQueryOptions = (queryString: string) =>
	queryOptions({
		queryKey: [...MY_DOCTOR_SCHEDULES_QUERY_KEY, queryString],
		queryFn: () => getMyDoctorSchedules(queryString),
		// রোগী book করলে isBooked বদলায় (doctor invalidate করে না), তাই ছোট রাখা
		staleTime: 1000 * 60 * 5,
		gcTime: 1000 * 60 * 30,
	});
```

- **`MY_DOCTOR_SCHEDULES_QUERY_KEY = ["my-doctor-schedules"]`** → এই একটা key দিয়ে শুরু **দুটো** query:
  - table-এর list (`["my-doctor-schedules", "page=1&limit=10"]`)
  - Add dialog-এর "ওই দিনে আমার slot" (`["my-doctor-schedules", "limit=100&schedule.startDateTime..."]`)
  - Add/Remove-এর পর একবার invalidate → দুটোই নতুন।
- `staleTime` ৫ মিনিট → রোগী book করলে `isBooked` বদলায়, doctor-এর cache invalidate হয় না (অন্য user-এর কাজ), তাই বেশিক্ষণ পুরনো না থাকুক।

---

## ধাপ ৭: `doctorSchedulesTable.config.ts` — URL ↔ backend অনুবাদ ⭐

`src/components/modules/Doctor/DoctorSchedules/doctorSchedulesTable.config.ts`

```ts
import { buildQueryString } from "@/lib/buildQueryString";
import { addDaysToDateString, clinicDayStartIso } from "@/lib/scheduleTime";

export const MY_SCHEDULE_TABLE_DEFAULTS = { page: "1", limit: "10" } as const;

export const BOOKED_KEY = "isBooked";
export const FROM_KEY = "from";
export const TO_KEY = "to";

// "Reset" চাপলে এগুলো মুছবে
export const MY_SCHEDULE_FILTER_KEYS = [BOOKED_KEY, FROM_KEY, TO_KEY] as const;

// এগুলো URL থেকে সরাসরি backend-এ যায়
// (backend sortable: createdAt, schedule.startDateTime, schedule.endDateTime)
const PASS_THROUGH_KEYS: ReadonlySet<string> = new Set(["page", "limit", "sortBy", "sortOrder", BOOKED_KEY]);

// backend QueryBuilder "true"/"false" string-কে boolean বানায়
export const STATUS_OPTIONS = [
	{ label: "Available", value: "false" },
	{ label: "Booked", value: "true" },
];

type SearchParamsInput = URLSearchParams | Record<string, string | string[] | undefined>;

const ISO_DATE_REGEX = /^\d{4}-\d{2}-\d{2}$/;

const readDateParam = (input: SearchParamsInput, key: string) => {
	const raw = input instanceof URLSearchParams ? input.get(key) : input[key];
	const value = Array.isArray(raw) ? raw[0] : raw;
	return value && ISO_DATE_REGEX.test(value) ? value : undefined;
};

/**
 * URL-এ সুন্দর ?from=2026-11-08&to=2026-11-10 থাকে,
 * backend-এ যায় schedule.startDateTime[gte]/[lt] = ঢাকার দিনের শুরু (ISO)।
 * server page আর client table দুজনেই এটা ডাকে → একই cache key।
 */
export const buildMyScheduleQueryString = (input: SearchParamsInput) => {
	const params = new URLSearchParams(buildQueryString(input, PASS_THROUGH_KEYS, MY_SCHEDULE_TABLE_DEFAULTS));

	const from = readDateParam(input, FROM_KEY);
	const to = readDateParam(input, TO_KEY);

	if (from) params.set("schedule.startDateTime[gte]", clinicDayStartIso(from));
	// "to" দিনটাও ধরা হবে → পরের দিনের রাত 12টার আগে পর্যন্ত
	if (to) params.set("schedule.startDateTime[lt]", clinicDayStartIso(addDaysToDateString(to, 1)));

	return params.toString();
};
```

### URL-এ যা থাকে বনাম backend-এ যা যায়

```
Browser URL (পড়তে সুন্দর):
  /doctor/dashboard/my-schedules?isBooked=false&from=2026-11-08&to=2026-11-10&sortBy=schedule.startDateTime&sortOrder=asc

buildMyScheduleQueryString → backend:
  page=1&limit=10&isBooked=false&sortBy=schedule.startDateTime&sortOrder=asc
  &schedule.startDateTime[gte]=2026-11-07T18:00:00.000Z     ← ৮ Nov ঢাকার রাত ১২টা
  &schedule.startDateTime[lt]=2026-11-10T18:00:00.000Z      ← ১১ Nov ঢাকার রাত ১২টা (১০ Nov পুরোটা ধরা)
```

| Key | URL-এ | Backend-এ |
|---|---|---|
| `page`, `limit`, `sortBy`, `sortOrder`, `isBooked` | হুবহু | হুবহু (**PASS_THROUGH_KEYS**) |
| `from` | `2026-11-08` | `schedule.startDateTime[gte]` = ঢাকার দিনের শুরু ISO |
| `to` | `2026-11-10` | `schedule.startDateTime[lt]` = **পরের দিনের** শুরু ISO |
| `schedule.startDateTime[gte]` (কেউ হাতে URL-এ লিখলে) | — | **বাদ** (allowlist-এ নেই) |

### ভেঙে বোঝা

- **`buildQueryString(input, PASS_THROUGH_KEYS, DEFAULTS)`** (note 16) → সাধারণ key-গুলো আগের মতো allowlist + default।
- তারপর **`from`/`to` আলাদা করে পড়ে অনুবাদ** → `params.set(...)`।
- **`readDateParam`** → server-এ `searchParams` object, client-এ `URLSearchParams` (note 16-এর `toEntries`-এর মতো) — দুটোই পড়ে; array হলে প্রথমটা; `YYYY-MM-DD` না হলে বাদ (`"abc"` হলে `new Date` ভেঙে যেত)।
- **`[lt]` (less than) পরের দিনের শুরু** → `to` দিনটার শেষ মুহূর্ত পর্যন্ত সব ধরা পড়ে (`[lte] ২৩:৫৯:৫৯.৯৯৯` লেখার চেয়ে পরিষ্কার)।
- **Server page আর client table দুজনেই এই function ডাকে** → হুবহু একই string → একই cache key (note 16-এর মূল নিয়ম)।
- `STATUS_OPTIONS` → `value: "false"` / `"true"` string; backend QueryBuilder `"true"`/`"false"`-কে boolean বানায়।
- `MY_SCHEDULE_FILTER_KEYS` → Reset-এ মোছে (`isBooked`, `from`, `to`); page/limit/sort থাকে।

---

## ধাপ ৮: `doctorSchedulesColumns.tsx`

```tsx
import DateCell from "@/components/shared/table/cells/DateCell";
import type { DataTableColumnDef } from "@/components/shared/table/dataTableFeatures";
import { Badge } from "@/components/ui/badge";
import { formatClinicTime, getDurationMinutes } from "@/lib/scheduleTime";
import type { IDoctorSchedule } from "@/types/doctorSchedule.types";

// ⚠️ sortable column-এর id = backend-এর sortBy field
// nested field, তাই accessorKey-এর বদলে id + accessorFn
export const doctorScheduleColumns: DataTableColumnDef<IDoctorSchedule>[] = [
	{
		id: "schedule.startDateTime",
		accessorFn: (row) => row.schedule.startDateTime,
		header: "Date",
		cell: ({ row }) => <DateCell value={row.original.schedule.startDateTime} />,
	},
	{
		id: "time",
		header: "Time",
		enableSorting: false,
		cell: ({ row }) => (
			<span className="whitespace-nowrap tabular-nums">
				{formatClinicTime(row.original.schedule.startDateTime)} – {formatClinicTime(row.original.schedule.endDateTime)}
			</span>
		),
	},
	{
		id: "duration",
		header: "Duration",
		enableSorting: false,
		cell: ({ row }) => {
			const minutes = getDurationMinutes(row.original.schedule.startDateTime, row.original.schedule.endDateTime);
			return minutes ? <span className="tabular-nums">{minutes} min</span> : <span className="text-muted-foreground">—</span>;
		},
	},
	{
		accessorKey: "isBooked",
		header: "Status",
		enableSorting: false,
		cell: ({ row }) =>
			row.original.isBooked ? <Badge>Booked</Badge> : <Badge variant="outline">Available</Badge>,
	},
	{
		accessorKey: "createdAt",
		header: "Added on",
		cell: ({ row }) => <DateCell value={row.original.createdAt} />,
	},
];
```

### Nested field-এ sort — `id` + `accessorFn`

```ts
{ id: "schedule.startDateTime", accessorFn: (row) => row.schedule.startDateTime, ... }
```
- Date আসলে `row.schedule.startDateTime` (nested)। `accessorKey: "schedule.startDateTime"` দিলে TanStack নিজে dot দিয়ে ভিতরে ঢোকে — কিন্তু type-check-এ nested path নিয়ে ঝামেলা হতে পারে।
- তাই **`id`** = backend-এর sortBy নাম (`"schedule.startDateTime"` — `doctorScheduleSortableFields`-এ আছে), আর **`accessorFn`** = মান কোথা থেকে।
- Header চাপলে URL-এ `sortBy=schedule.startDateTime` → backend বোঝে (note 17-এর নিয়ম: sortable column-এর id = backend sortBy)।

| Column | Sort | দেখায় |
|---|---|---|
| Date | ✅ | `08 Nov 2026` |
| Time | ❌ | `09:00 AM – 09:30 AM` |
| Duration | ❌ | `30 min` |
| Status (`isBooked`) | ❌ | **Booked** (ভরাট) / Available (outline) |
| Added on (`createdAt`) | ✅ | কবে doctor slot-টা নিয়েছে |

---

## ধাপ ৯: `AddSchedulesDialog.tsx` — slot বাছাই ⭐

`src/components/modules/Doctor/DoctorSchedules/AddSchedulesDialog.tsx`

```tsx
"use client";

import { createMyDoctorSchedulesAction } from "@/actions/doctorSchedule.actions";
import { Button } from "@/components/ui/button";
import {
    Dialog,
    DialogContent,
    DialogDescription,
    DialogFooter,
    DialogHeader,
    DialogTitle,
    DialogTrigger,
} from "@/components/ui/dialog";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { useActionMutation } from "@/hooks/useActionMutation";
import {
    addDaysToDateString,
    clinicDayStartIso,
    formatClinicTime,
    getClinicToday,
    SCHEDULE_INTERVAL_MINUTES,
} from "@/lib/scheduleTime";
import { MY_DOCTOR_SCHEDULES_QUERY_KEY, myDoctorSchedulesQueryOptions } from "@/queries/doctorSchedule.queries";
import { schedulesQueryOptions } from "@/queries/schedule.queries";
import { useQuery } from "@tanstack/react-query";
import { CalendarPlus, Check, LoaderCircle } from "lucide-react";
import { useMemo, useState } from "react";

const ISO_DATE_REGEX = /^\d{4}-\d{2}-\d{2}$/;

// ওই দিনের admin-তৈরি সব slot (একদিনে সর্বোচ্চ 48টা, তাই limit 100 যথেষ্ট)
const daySlotsQueryString = (date: string) =>
    new URLSearchParams({
        startDate: date,
        endDate: date,
        limit: "100",
        sortBy: "startDateTime",
        sortOrder: "asc",
    }).toString();

// ওই দিনে আমি আগে থেকে কোনগুলো নিয়েছি
const myDaySlotsQueryString = (date: string) =>
    new URLSearchParams({
        limit: "100",
        "schedule.startDateTime[gte]": clinicDayStartIso(date),
        "schedule.startDateTime[lt]": clinicDayStartIso(addDaysToDateString(date, 1)),
    }).toString();

export default function AddSchedulesDialog() {
    const [open, setOpen] = useState(false);
    const [today, setToday] = useState("");
    const [date, setDate] = useState("");
    const [openedAt, setOpenedAt] = useState(0);
    const [selectedIds, setSelectedIds] = useState<ReadonlySet<string>>(() => new Set());

    const isValidDate = ISO_DATE_REGEX.test(date);

    // enabled: false দিলেও queryKey প্রতি render-এ তৈরি হয়,
    // তাই খালি/ভুল তারিখে Date বানানোর আগেই থামাই
    const slotsQueryString = isValidDate ? daySlotsQueryString(date) : "";
    const mySlotsQueryString = isValidDate ? myDaySlotsQueryString(date) : "";

    const slotsQuery = useQuery({
        ...schedulesQueryOptions(slotsQueryString),
        enabled: open && isValidDate,
    });

    const mineQuery = useQuery({
        ...myDoctorSchedulesQueryOptions(mySlotsQueryString),
        enabled: open && isValidDate,
    });

    const mySlotIds = useMemo(
        () => new Set((mineQuery.data?.data ?? []).map((item) => item.scheduleId)),
        [mineQuery.data],
    );

    // backend অতীতের slot নেয় না (400), তাই দেখাই-ই না
    const slots = useMemo(
        () => (slotsQuery.data?.data ?? []).filter((slot) => new Date(slot.startDateTime).getTime() > openedAt),
        [slotsQuery.data, openedAt],
    );

    const selectableIds = slots.filter((slot) => !mySlotIds.has(slot.id)).map((slot) => slot.id);
    const allSelected = selectableIds.length > 0 && selectableIds.every((id) => selectedIds.has(id));

    const { mutate, isPending } = useActionMutation({
        action: (scheduleIds: string[]) => createMyDoctorSchedulesAction({ scheduleIds }),
        invalidateKey: MY_DOCTOR_SCHEDULES_QUERY_KEY,
        successMessage: (added) => `${added.length} ${added.length === 1 ? "slot" : "slots"} added to your schedule`,
        onSuccess: () => setOpen(false),
    });

    // Date.now() render-এর মধ্যে না, event handler-এ (React purity rule)
    const handleOpenChange = (nextOpen: boolean) => {
        if (isPending) return;

        if (nextOpen) {
            const clinicToday = getClinicToday();
            setToday(clinicToday);
            setDate(clinicToday);
            setOpenedAt(Date.now());
        }

        setSelectedIds(new Set());
        setOpen(nextOpen);
    };

    const handleDateChange = (nextDate: string) => {
        setDate(nextDate);
        setSelectedIds(new Set()); // অন্য দিনের selection রাখি না
    };

    const toggleSlot = (id: string) =>
        setSelectedIds((prev) => {
            const next = new Set(prev);
            if (next.has(id)) next.delete(id);
            else next.add(id);
            return next;
        });

    const toggleAll = () => setSelectedIds(allSelected ? new Set() : new Set(selectableIds));

    const isLoadingSlots = slotsQuery.isLoading || mineQuery.isLoading;

    const renderSlots = () => {
        if (!isValidDate) {
            return <p className="text-sm text-muted-foreground">Pick a day to see open slots.</p>;
        }

        if (isLoadingSlots) {
            return (
                <p className="flex items-center gap-2 text-sm text-muted-foreground">
                    <LoaderCircle className="size-4 animate-spin" aria-hidden />
                    Loading slots...
                </p>
            );
        }

        if (slotsQuery.isError || mineQuery.isError) {
            return <p className="text-sm text-destructive">Couldn&apos;t load slots for this day. Please try again.</p>;
        }

        if (slots.length === 0) {
            return (
                <p className="rounded-md border border-dashed p-4 text-center text-sm text-muted-foreground">
                    No upcoming slots on this day. Pick another day, or ask an admin to create schedules.
                </p>
            );
        }

        return (
            <ul className="grid grid-cols-2 gap-2 sm:grid-cols-3">
                {slots.map((slot) => {
                    const isMine = mySlotIds.has(slot.id);
                    const isSelected = selectedIds.has(slot.id);

                    return (
                        <li key={slot.id}>
                            <Button
                                type="button"
                                variant={isMine ? "secondary" : isSelected ? "default" : "outline"}
                                disabled={isMine || isPending}
                                aria-pressed={isMine || isSelected}
                                onClick={() => toggleSlot(slot.id)}
                                className="h-auto w-full flex-col gap-0.5 py-2"
                            >
                                <span className="tabular-nums">{formatClinicTime(slot.startDateTime)}</span>
                                <span className="text-xs font-normal opacity-80">
									{isMine ? (
                                        <span className="inline-flex items-center gap-1">
											<Check className="size-3" aria-hidden />
											Added
										</span>
                                    ) : (
                                        `to ${formatClinicTime(slot.endDateTime)}`
                                    )}
								</span>
                            </Button>
                        </li>
                    );
                })}
            </ul>
        );
    };

    return (
        <Dialog open={open} onOpenChange={handleOpenChange}>
            <DialogTrigger asChild>
                <Button>
                    <CalendarPlus />
                    Add slots
                </Button>
            </DialogTrigger>

            <DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-xl">
                <DialogHeader>
                    <DialogTitle>Add slots to my schedule</DialogTitle>
                    <DialogDescription>
                        Pick a day, then choose the {SCHEDULE_INTERVAL_MINUTES}-minute slots when patients can book you. Times are
                        Bangladesh time (GMT+6).
                    </DialogDescription>
                </DialogHeader>

                <div className="flex items-end justify-between gap-3">
                    <div className="space-y-1">
                        <Label htmlFor="slot-day">Day</Label>
                        <Input
                            id="slot-day"
                            type="date"
                            min={today}
                            value={date}
                            disabled={isPending}
                            onChange={(event) => handleDateChange(event.target.value)}
                        />
                    </div>

                    {selectableIds.length > 0 && (
                        <Button type="button" variant="ghost" size="sm" onClick={toggleAll} disabled={isPending}>
                            {allSelected ? "Clear all" : "Select all"}
                        </Button>
                    )}
                </div>

                {renderSlots()}

                <DialogFooter>
                    <Button type="button" variant="outline" disabled={isPending} onClick={() => handleOpenChange(false)}>
                        Cancel
                    </Button>
                    <Button type="button" disabled={isPending || selectedIds.size === 0} onClick={() => mutate([...selectedIds])}>
                        {isPending ? <LoaderCircle className="animate-spin" /> : <Check />}
                        {isPending
                            ? "Adding..."
                            : selectedIds.size > 0
                                ? `Add ${selectedIds.size} ${selectedIds.size === 1 ? "slot" : "slots"}`
                                : "Add slots"}
                    </Button>
                </DialogFooter>
            </DialogContent>
        </Dialog>
    );
}

```

> ✏️ তোমার code-এ এই file-এ কিছু লাইন space আর কিছু tab দিয়ে indent করা (copy-paste থেকে)। কাজে সমস্যা নেই; editor-এ format (Format Document) চালালে ঠিক হয়ে যাবে।

### State

| State | কাজ |
|---|---|
| `open` | dialog খোলা? |
| `today` | ঢাকার আজ — date input-এর `min` |
| `date` | বাছাই করা দিন |
| `openedAt` | dialog খোলার মুহূর্ত (ms) — এর আগের slot দেখায় না |
| `selectedIds` | বাছাই করা slot-এর id (`Set`) |

### দুটো query — "কী আছে" আর "আমার কী আছে"

```ts
slotsQuery → schedulesQueryOptions(daySlotsQueryString(date))      // admin-এর সব slot, ওই দিন
mineQuery  → myDoctorSchedulesQueryOptions(myDaySlotsQueryString(date))  // আমার নেওয়া, ওই দিন
```
- Doctor-ও `GET /schedules` পড়তে পারে (backend: `checkAuth(ADMIN, SUPER_ADMIN, DOCTOR)`)।
- `limit: "100"` → একদিনে সর্বোচ্চ ৪৮টা slot, সব এক page-এ।
- `mySlotIds` (`Set`) → কোন slot আগেই নেওয়া → grid-এ "✓ Added", disabled।
- **`enabled: open && isValidDate`** → dialog বন্ধ থাকলে বা তারিখ অর্ধেক লেখা থাকলে request যায় না।
- **`isValidDate ? ... : ""`** → `enabled: false` দিলেও `queryOptions(...)` প্রতি render-এ চলে (key বানাতে)। অর্ধেক লেখা তারিখে `clinicDayStartIso("2026-1")` → `new Date(...)` → **Invalid Date → `toISOString()` throw** → page crash। তাই ভুল তারিখে string বানানোর আগেই থামানো।
- Query key `["schedules", ...]` → admin-এর table-এর সাথে একই prefix; admin slot বদলালে (একই browser-এ না হলেও) staleTime পরে আবার আসে।

### অতীতের slot বাদ

```ts
slots = slotsQuery.data.filter((slot) => new Date(slot.startDateTime).getTime() > openedAt);
```
- Backend অতীতের slot নিলে `400` দেয় (server note 26-এর `ensureSchedulesAvailable`)। তাই দেখানোই হয় না।
- আজকের দিন বাছলে এখনকার আগের সময়গুলো লুকানো থাকে।

### `Date.now()` কেন event handler-এ? (React purity)

```ts
const handleOpenChange = (nextOpen) => {
	if (nextOpen) {
		setToday(getClinicToday());
		setDate(clinicToday);
		setOpenedAt(Date.now());   // ← এখানে
	}
	...
};
```
- React-এর নিয়ম: **render "pure"** হতে হবে — একই props/state-এ একই ফল। Render-এর ভিতরে `Date.now()` প্রতিবার আলাদা → React-এর lint (`react-hooks/purity`) আপত্তি করে, আর server/browser-এ আলাদা হয়ে hydration mismatch হতে পারে।
- তাই **dialog খোলার মুহূর্তে একবার** সময় নিয়ে state-এ রাখা → render শুধু state পড়ে।
- `useState("")` দিয়ে শুরু → server render-এ কোনো সময় নেই, mismatch নেই।

### বাছাই

| Function | কাজ |
|---|---|
| `handleDateChange` | দিন বদলালে selection মুছে দেয় (আগের দিনের slot ভুল করে পাঠানো না হয়) |
| `toggleSlot(id)` | `Set` কপি করে যোগ/বাদ (state সরাসরি বদলানো যায় না, তাই `new Set(prev)`) |
| `selectableIds` | এই দিনের slot − আমার আগে নেওয়া |
| `allSelected` | সব selectable বাছা? → "Select all" / "Clear all" |
| `toggleAll` | সব বাছো / সব মোছো |

- `useState<ReadonlySet<string>>(() => new Set())` → `ReadonlySet` type → ভুল করে `selectedIds.add()` লিখলে TypeScript আপত্তি করে (সবসময় নতুন Set বানাতে হবে)। `() => new Set()` → শুধু প্রথম render-এ বানায়।

### Slot button-এর ৩ অবস্থা

| অবস্থা | `variant` | disabled | নিচের লেখা |
|---|---|---|---|
| আগেই নেওয়া (`isMine`) | `secondary` | ✅ | ✓ Added |
| এখন বাছা (`isSelected`) | `default` (ভরাট) | ❌ | `to 09:30 AM` |
| বাছা না | `outline` | ❌ | `to 09:30 AM` |

- `aria-pressed` → toggle button (note 20-এর `AppMultiSelectField`-এর মতো)।
- `type="button"` → dialog-এ form নেই, তবু অভ্যাস।
- `h-auto flex-col` → দুই লাইনের button (সময় + নিচে ছোট লেখা)।

### `renderSlots()` — ৫ অবস্থা

```
তারিখ ঠিক না       → "Pick a day to see open slots."
load হচ্ছে         → spinner "Loading slots..."
error             → লাল "Couldn't load slots..."
slot নেই          → dashed box "No upcoming slots on this day... ask an admin"
আছে               → grid (mobile ২, sm ৩ column)
```
- JSX-এর ভিতরে অনেক `? :` না লিখে আলাদা function → পড়তে সহজ।
- **`isLoading`** (isPending না) → `enabled: false` অবস্থায় `isPending` সবসময় true থাকে, কিন্তু `isLoading` শুধু আসলে fetch চলার সময় true।

### Mutation
- `action: (scheduleIds) => createMyDoctorSchedulesAction({ scheduleIds })` → Set থেকে array (`[...selectedIds]`) → `{ scheduleIds }`।
- `successMessage` → `"3 slots added to your schedule"` (ফেরত array-এর length)।
- Button-এর লেখা বদলায়: "Add slots" → "Add 3 slots" → "Adding..."।
- `handleOpenChange`-এর শুরুতে `if (isPending) return;` → save চলার সময় বন্ধ হয় না।

---

## ধাপ ১০: `RemoveScheduleDialog.tsx`

```tsx
"use client";

import { deleteMyDoctorScheduleAction } from "@/actions/doctorSchedule.actions";
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
import { MY_DOCTOR_SCHEDULES_QUERY_KEY } from "@/queries/doctorSchedule.queries";
import type { IDoctorSchedule } from "@/types/doctorSchedule.types";
import { LoaderCircle, Trash2 } from "lucide-react";

interface RemoveScheduleDialogProps {
	row: IDoctorSchedule | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

export default function RemoveScheduleDialog({ row: doctorSchedule, open, onOpenChange }: RemoveScheduleDialogProps) {
	const { mutate, isPending } = useActionMutation({
		action: deleteMyDoctorScheduleAction,
		invalidateKey: MY_DOCTOR_SCHEDULES_QUERY_KEY,
		successMessage: "Slot removed from your schedule",
		onSuccess: () => onOpenChange(false),
	});

	// backend booked slot remove করতে দেয় না (400), তাই আগেই আটকাই
	const isBooked = doctorSchedule?.isBooked ?? false;

	return (
		<AlertDialog open={open} onOpenChange={(next) => !isPending && onOpenChange(next)}>
			<AlertDialogContent>
				<AlertDialogHeader>
					<AlertDialogTitle>{isBooked ? "Can't remove this slot" : "Remove this slot?"}</AlertDialogTitle>
					<AlertDialogDescription>
						<strong className="text-foreground">
							{formatClinicDateTime(doctorSchedule?.schedule.startDateTime)} –{" "}
							{formatClinicTime(doctorSchedule?.schedule.endDateTime)}
						</strong>{" "}
						{isBooked
							? "already has a patient appointment, so it can't be removed."
							: "will be removed. Patients won't be able to book you at this time."}
					</AlertDialogDescription>
				</AlertDialogHeader>

				<AlertDialogFooter>
					<AlertDialogCancel disabled={isPending}>{isBooked ? "Close" : "Cancel"}</AlertDialogCancel>
					{!isBooked && (
						<Button
							variant="destructive"
							disabled={isPending || !doctorSchedule}
							onClick={() => doctorSchedule && mutate(doctorSchedule.scheduleId)}
						>
							{isPending ? <LoaderCircle className="animate-spin" /> : <Trash2 />}
							{isPending ? "Removing..." : "Remove"}
						</Button>
					)}
				</AlertDialogFooter>
			</AlertDialogContent>
		</AlertDialog>
	);
}
```

- Admin-এর `DeleteScheduleDialog` (note 21)-এর ছোট version: booked হলে "Can't remove this slot" + শুধু Close।
- `mutate(doctorSchedule.scheduleId)` → **scheduleId** পাঠানো (doctorId backend নিজে cookie থেকে জানে)।
- Description → "Patients won't be able to book you at this time." — doctor বোঝে remove মানে কী।

---

## ধাপ ১১: `DoctorSchedulesTable.tsx`

```tsx
"use client";

import DataTable from "@/components/shared/table/DataTable";
import DataTablePagination from "@/components/shared/table/DataTablePagination";
import DataTableDateRangeFilter from "@/components/shared/table/filters/DataTableDateRangeFilter";
import DataTableSelectFilter from "@/components/shared/table/filters/DataTableSelectFilter";
import { Button } from "@/components/ui/button";
import { useRowAction } from "@/hooks/useRowAction";
import { useTableUrlState } from "@/hooks/useTableUrlState";
import { myDoctorSchedulesQueryOptions } from "@/queries/doctorSchedule.queries";
import type { IDoctorSchedule } from "@/types/doctorSchedule.types";
import { keepPreviousData, useQuery } from "@tanstack/react-query";
import { X } from "lucide-react";
import { useEffect, useMemo } from "react";
import AddSchedulesDialog from "./AddSchedulesDialog";
import { doctorScheduleColumns } from "./doctorSchedulesColumns";
import {
	BOOKED_KEY,
	buildMyScheduleQueryString,
	FROM_KEY,
	MY_SCHEDULE_FILTER_KEYS,
	MY_SCHEDULE_TABLE_DEFAULTS,
	STATUS_OPTIONS,
	TO_KEY,
} from "./doctorSchedulesTable.config";
import RemoveScheduleDialog from "./RemoveScheduleDialog";

// নিজের list-এ scheduleId-ই unique (doctorId সবার একই)
const getRowId = (item: IDoctorSchedule) => item.scheduleId;

export default function DoctorSchedulesTable() {
	const table = useTableUrlState({ defaultLimit: Number(MY_SCHEDULE_TABLE_DEFAULTS.limit) });
	const rowAction = useRowAction<IDoctorSchedule>();

	const queryString = buildMyScheduleQueryString(table.searchParams);

	const {
		data: response,
		isPending,
		isFetching,
		isError,
	} = useQuery({
		...myDoctorSchedulesQueryOptions(queryString),
		placeholderData: keepPreviousData,
	});

	const meta = response?.meta;
	const { setPage } = table;
	useEffect(() => {
		if (meta && meta.totalPages > 0 && meta.page > meta.totalPages) setPage(meta.totalPages);
	}, [meta, setPage]);

	// এখানে edit করার কিছু নেই, শুধু remove। DataTable শুধু দেওয়া action-ই দেখায়।
	const actions = useMemo(() => ({ onDelete: rowAction.actions.onDelete }), [rowAction.actions]);

	const hasActiveFilters = MY_SCHEDULE_FILTER_KEYS.some((key) => table.searchParams.has(key));

	if (isError && !response) {
		return <p className="text-sm text-destructive">Failed to load your schedules.</p>;
	}

	return (
		<div className="space-y-4">
			<div className="flex flex-wrap items-center justify-between gap-3">
				<div className="flex flex-wrap items-center gap-2">
					<DataTableSelectFilter
						label="Status"
						allLabel="All slots"
						value={table.getFilter(BOOKED_KEY)}
						options={STATUS_OPTIONS}
						onChange={(value) => table.setFilters({ [BOOKED_KEY]: value })}
					/>

					<DataTableDateRangeFilter
						label="Date"
						from={table.getFilter(FROM_KEY)}
						to={table.getFilter(TO_KEY)}
						onChange={({ from, to }) => table.setFilters({ [FROM_KEY]: from, [TO_KEY]: to })}
					/>

					{hasActiveFilters && (
						<Button variant="ghost" onClick={() => table.clearParams(MY_SCHEDULE_FILTER_KEYS)}>
							<X />
							Reset
						</Button>
					)}
				</div>

				<AddSchedulesDialog />
			</div>

			<DataTable
				data={response?.data ?? []}
				columns={doctorScheduleColumns}
				getRowId={getRowId}
				isLoading={isPending}
				isFetching={isFetching}
				emptyMessage={hasActiveFilters ? "No slots match these filters." : "You haven't added any slots yet."}
				sorting={{ state: table.sorting, onSortingChange: table.setSorting }}
				actions={actions}
			/>

			<DataTablePagination meta={response?.meta} onPageChange={table.setPage} onLimitChange={table.setLimit} />

			<RemoveScheduleDialog {...rowAction.dialogProps("delete")} />
		</div>
	);
}
```

| অংশ | Admin table থেকে পার্থক্য |
|---|---|
| `getRowId = item.scheduleId` | নিজের list-এ doctorId সবার একই, তাই scheduleId-ই unique |
| `buildMyScheduleQueryString(table.searchParams)` | `buildQueryString`-এর বদলে নিজের (ধাপ ৭) |
| `actions = { onDelete }` শুধু | Edit/View নেই — DataTable শুধু দেওয়া action দেখায় (menu-তে শুধু "Delete") |
| Filter | Status (`DataTableSelectFilter`) + Date (`DataTableDateRangeFilter`) — দুটোই আগের shared component |
| Toolbar ডানে | `<AddSchedulesDialog />` |

বাকি সব একই: `useTableUrlState`, `keepPreviousData`, শেষ page-এর `useEffect`, `useRowAction`।

> ✏️ "⋯" menu-তে item-এর লেখা "Delete" (DataTable-এর default) আর dialog-এ "Remove" — চাইলে পরে DataTable-এ label বদলানোর option যোগ করা যায়।

---

## ধাপ ১২: `src/app/(dashboardLayout)/doctor/dashboard/my-schedules/page.tsx`

```tsx
import DoctorSchedulesTable from "@/components/modules/Doctor/DoctorSchedules/DoctorSchedulesTable";
import { buildMyScheduleQueryString } from "@/components/modules/Doctor/DoctorSchedules/doctorSchedulesTable.config";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { getQueryClient } from "@/lib/getQueryClient";
import { myDoctorSchedulesQueryOptions } from "@/queries/doctorSchedule.queries";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import { Suspense } from "react";

type SearchParams = Record<string, string | string[] | undefined>;

async function MySchedulesSection({ searchParams }: { searchParams: Promise<SearchParams> }) {
	const queryString = buildMyScheduleQueryString(await searchParams);
	const queryClient = getQueryClient();

	await queryClient.query(myDoctorSchedulesQueryOptions(queryString)).catch(noop);

	return (
		<HydrationBoundary state={dehydrate(queryClient)}>
			<DoctorSchedulesTable />
		</HydrationBoundary>
	);
}

export default function MySchedulesPage({ searchParams }: PageProps<"/doctor/dashboard/my-schedules">) {
	return (
		<div className="space-y-6">
			<div>
				<h1 className="text-2xl font-semibold tracking-tight">My Schedules</h1>
				<p className="text-sm text-muted-foreground">Choose when patients can book you, and manage your slots</p>
			</div>

			<Suspense fallback={<LoadingSpinner text="Loading your schedules..." />}>
				<MySchedulesSection searchParams={searchParams} />
			</Suspense>
		</div>
	);
}
```

- আগের সব page-এর মতো (note 16, 21)। শুধু `buildMyScheduleQueryString(await searchParams)`।
- Sidebar-এর doctor menu-র "My Schedules" (note 09: `/doctor/dashboard/my-schedules`) এখন কাজ করে।
- Proxy (note 08) শুধু DOCTOR-কে `/doctor/dashboard/...`-এ ঢুকতে দেয়।

---

## 🧪 Test

আগে admin হয়ে (note 21) আজ আর কাল কিছু slot বানিয়ে রাখো। তারপর DOCTOR হয়ে login।

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | Sidebar → My Schedules | খালি: "You haven't added any slots yet." |
| ২ | Add slots | dialog, Day = আজ (ঢাকা), এখনকার পরের slot-গুলো grid-এ |
| ৩ | Date picker-এ গতকাল | ধূসর (বাছা যায় না) |
| ৪ | ৩টা slot বাছো | ভরাট, button "Add 3 slots" |
| ৫ | Select all → Clear all | সব বাছে / সব মোছে |
| ৬ | Add | "3 slots added to your schedule", table-এ ৩টা Available |
| ৭ | আবার Add slots, একই দিন | ওই ৩টা "✓ Added", disabled |
| ৮ | দিন বদলাও | selection মুছে যায়, নতুন দিনের slot |
| ৯ | Admin slot বানায়নি এমন দিন | "No upcoming slots on this day…" |
| ১০ | Status → Booked | শুধু booked (এখন খালি) |
| ১১ | Date filter ৮–১০ Nov | URL-এ `?from=2026-11-08&to=2026-11-10`, Network-এ `schedule.startDateTime[gte]=…T18:00:00.000Z` |
| ১২ | "Date" header চাপো | `sortBy=schedule.startDateTime` |
| ১৩ | ⋯ → Delete → Remove | "Slot removed from your schedule" |
| ১৪ | Patient book করেছে এমন slot Remove | "Can't remove this slot", শুধু Close |
| ১৫ | Admin হয়ে Schedules table | ওই slot-এর Doctors column-এ এই doctor |

---

## 🔜 পরে যা যোগ হবে
- [ ] Patient: doctor-এর Available slot দেখে appointment নেওয়া (তখন `isBooked: true`)
- [ ] Doctor dashboard-এ "আজকের slot" / upcoming appointment
- [ ] DataTable-এ action-এর label বদলানোর option ("Delete" → "Remove")

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Default-এ পুরনো (অতীতের) slot আগে দেখায়** — backend-এর `defaultSortBy: "schedule.startDateTime"` (asc), তাই কয়েক সপ্তাহ পর table-এর প্রথম page ভরে থাকবে শেষ হয়ে যাওয়া slot-এ, আর doctor-কে আজকের slot খুঁজতে পরের page-এ যেতে হবে। Default-এ আজ থেকে দেখাতে `buildMyScheduleQueryString`-এ:
   ```ts
   const from = readDateParam(input, FROM_KEY) ?? getClinicToday();
   ```
   (তখন "পুরনো slot দেখতে" date filter-এ আগের তারিখ দিতে হবে।) সাবধান: এই function client-এর render-এও চলে, তাই ভিতরে `Date.now()` থাকলে React-এর purity lint আপত্তি করতে পারে, আর ঠিক রাত ১২টার মুহূর্তে server আর browser আলাদা দিন ধরে cache key না মিলতে পারে (untested)। আরো পরিষ্কার উপায়: backend-এর `getMyDoctorSchedules`-এ default হিসেবে "আজ থেকে" filter রাখা।

2. **Dialog অনেকক্ষণ খোলা রাখলে অতীতের slot বাছা যায়** — `openedAt` dialog খোলার সময় একবার নেওয়া। ০৯:২৫-এ খুলে ০৯:৪০-এ "Add" চাপলে ০৯:৩০-এর slot অতীত → backend `400 Invalid or past schedule id(s): <uuid>` — message-এ uuid, doctor বুঝবে না। সহজ সমাধান: Add চাপার মুহূর্তে আবার filter করো:
   ```ts
   onClick={() => {
   	const now = Date.now();
   	const validIds = slots.filter((s) => selectedIds.has(s.id) && new Date(s.startDateTime).getTime() > now).map((s) => s.id);
   	mutate(validIds);
   }}
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Schedule-এর timezone-এর হিসাব এখন দুই জায়গায়** — admin-এর date filter backend করে (`startDate`/`endDate`), doctor-এরটা frontend করে (`clinicDayStartIso`)। দুটোই ঢাকার `+06:00` ধরে, তাই মেলে। পরে backend-এর doctor-schedule-এও `startDate`/`endDate` সমর্থন যোগ করলে frontend-এর এই অনুবাদ লাগবে না, আর নিয়ম এক জায়গায় থাকবে।

4. **`CLINIC_UTC_OFFSET` আর `CLINIC_TIME_ZONE` দুটো আলাদা constant** — বাংলাদেশে daylight saving নেই, তাই `"+06:00"` সবসময় ঠিক। অন্য দেশের clinic-এ (DST আছে) fixed offset ভুল হবে — তখন date library (যেমন `date-fns-tz`) লাগবে।
