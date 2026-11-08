# 12 - Dashboard Stats Card (Reusable Card + Skeleton)

> Module: 48-1 (Creating Stats Card Component For Dashboard)

> Dashboard-এর উপরে যে ছোট ছোট সংখ্যার card থাকে ("Total Appointments: 1,250"), সেটার জন্য একটা **reusable** component। Admin, doctor, patient — সব dashboard-এ এটাই ব্যবহার হবে। সাথে data load হওয়ার সময়ের জন্য একটা **skeleton** version।

## 🧭 এক নজরে

```
Dashboard page
  → data আসছে (isPending)?  → <StatsCardSkeleton />  (ধূসর ঝিলমিল box)
  → data এসেছে              → <StatsCard title value icon description />
                                 └─ number হলে 1250 → "1,250" (নিজেই format)
                                 └─ string হলে (যেমন "৳ 50,000") যেমন আছে তেমন
```

### দেখতে কেমন

```
┌────────────────────────────────┐
│ Total Appointments      ┌────┐ │  ← CardHeader: title + CardAction (icon)
│                         │ 📅 │ │
│                         └────┘ │
│ 1,250                          │  ← CardContent: বড় সংখ্যা
│ Number of appointments …       │  ← description (optional)
└────────────────────────────────┘
```

---

## ধাপ ১: `src/components/shared/cards/StatsCard.tsx`

> ✏️ এই নোটে `card` আর `skeleton` shadcn component লাগে। `card` আগে install করা আছে (note 06)। Project-এ `skeleton`-ও আছে; নতুন project-এ না থাকলে:
> ```shell
> pnpm dlx shadcn@latest add card skeleton
> ```

```tsx
import { Card, CardAction, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Skeleton } from "@/components/ui/skeleton";
import { cn } from "cn";
import type { LucideIcon } from "lucide-react";

// ⚠️ এই file-এ "use client" দেবে না।
// তাহলে Server Component থেকে icon (function) পাঠানো যাবে না।

interface StatsCardProps {
	title: string;
	value: number | string; // টাকার মতো format করা মান string হিসেবে পাঠাও
	icon: LucideIcon;
	description?: string;
	className?: string;
}

// locale fixed রাখা: server আর browser-এ একই output → hydration mismatch হয় না
const numberFormatter = new Intl.NumberFormat("en-US");

const formatValue = (value: number | string) => (typeof value === "number" ? numberFormatter.format(value) : value);

export default function StatsCard({ title, value, icon: Icon, description, className }: StatsCardProps) {
	return (
		<Card className={cn("transition-shadow hover:shadow-md", className)}>
			<CardHeader>
				<CardTitle className="text-sm font-medium text-muted-foreground">{title}</CardTitle>
				<CardAction>
					<div className="flex size-9 items-center justify-center rounded-lg bg-primary/10 text-primary">
						<Icon className="size-5" aria-hidden />
					</div>
				</CardAction>
			</CardHeader>

			<CardContent className="space-y-1">
				{/* tabular-nums: সংখ্যা বদলালে width লাফায় না */}
				<p className="text-2xl font-bold tabular-nums">{formatValue(value)}</p>
				{description && <p className="text-xs text-muted-foreground">{description}</p>}
			</CardContent>
		</Card>
	);
}

// data load হওয়ার সময় "0"-এর বদলে এটা দেখাও
export function StatsCardSkeleton() {
	return (
		<Card>
			<CardHeader>
				<Skeleton className="h-4 w-24" />
				<CardAction>
					<Skeleton className="size-9 rounded-lg" />
				</CardAction>
			</CardHeader>

			<CardContent className="space-y-2">
				<Skeleton className="h-7 w-16" />
				<Skeleton className="h-3 w-32" />
			</CardContent>
		</Card>
	);
}
```

---

## ধাপ ২: ভেঙে বোঝা

### `shared/cards/` কেন, `modules/Dashboard/` না?
এই card কোনো একটা feature-এর না — admin, doctor, patient সবার dashboard-এ লাগবে, পরে অন্য জায়গাতেও লাগতে পারে। একাধিক জায়গায় লাগে এমন component → `shared/`, আর ধরন অনুযায়ী folder (`cards/`)। (ধাপ ৪-এর structure দেখো।)

### ⚠️ `"use client"` কেন দেবে না — icon-এর রহস্য ⭐

এখানে `icon: LucideIcon` — মানে আস্ত **component (function)** prop হিসেবে নেওয়া হচ্ছে, note 09-এর মতো নাম (string) না।

| Parent | StatsCard-এ `"use client"` নেই | StatsCard-এ `"use client"` দিলে |
|---|---|---|
| Server Component (`page.tsx`) | ✅ চলে — দুজনেই server-এ, function পাঠাতে সমস্যা নেই | ❌ Error — server থেকে client-এ function পাঠানো যায় না |
| Client Component (TanStack Query দিয়ে data আনে) | ✅ চলে — client-এর ভিতরে import হলে এটাও client হয়ে যায় | ✅ চলে |

তাই `"use client"` **না দিলে দুই জায়গাতেই চলে**। এই component-এ কোনো hook নেই, তাই client হওয়ার দরকারও নেই।

**Note 09 (sidebar) বনাম এখানে — দুই রকম কেন?**

| | Sidebar menu (note 09) | StatsCard |
|---|---|---|
| Icon কীভাবে | নাম (`"House"`) → `getIconComponent()` | সরাসরি component (`CalendarDays`) |
| কারণ | data server layout থেকে **client** `DashboardNav`-এ যায় (ওটা `usePathname`-এর জন্য client-ই হতে হবে) | StatsCard নিজে server/client যেকোনোটা হতে পারে, তাই function পাঠানো যায় |
| লাভ | serializable | সহজ: যেকোনো lucide icon সরাসরি, mapper-এ যোগ করতে হয় না |

### Props

| Prop | Type | কাজ |
|---|---|---|
| `title` | `string` | উপরের ছোট লেখা |
| `value` | `number \| string` | বড় সংখ্যা। number হলে নিজেই `1,250` বানায়, string হলে যেমন আছে তেমন দেখায় |
| `icon` | `LucideIcon` | ডান কোণের icon |
| `description?` | `string` | নিচের ছোট লেখা (optional) |
| `className?` | `string` | বাইরে থেকে বাড়তি style (যেমন রং বদলানো) — `cn()` দিয়ে জোড়া লাগে |

- `icon: Icon` → destructure করার সময় নাম বদলে বড় হাতের `Icon`, কারণ JSX-এ component-এর নাম বড় হাতের অক্ষরে শুরু হতে হয় (`<Icon />`; ছোট হাতের `<icon />` হলে React এটাকে HTML tag ভাবত)।

### `Intl.NumberFormat("en-US")` — locale কেন fixed?

```ts
const numberFormatter = new Intl.NumberFormat("en-US");
numberFormatter.format(1250000); // "1,250,000"
```

- **Locale না দিলে** server (Node-এর locale) আর browser (user-এর locale, যেমন `bn-BD` → `১২,৫০,০০০`) আলাদা লেখা বানাতে পারে। Server-এর HTML আর browser-এর render না মিললে React **hydration mismatch** error দেয়।
- `"en-US"` fixed → সব জায়গায় হুবহু একই output।
- File-এর উপরে (component-এর বাইরে) একবার বানানো → প্রতিবার render-এ নতুন formatter বানাতে হয় না।

### `formatValue`
```ts
typeof value === "number" ? numberFormatter.format(value) : value
```
- Number → comma দিয়ে format।
- String → কিছু না করে দেখায়। টাকা, শতাংশ (`"85%"`) বা অন্য বিশেষ format **parent নিজে বানিয়ে string পাঠাবে**। এতে card সহজ থাকে, সব রকম format-এর logic card-এ ঢোকাতে হয় না।

### Card-এর গঠন (shadcn)

| অংশ | কাজ |
|---|---|
| `CardHeader` | উপরের সারি। ভিতরে `CardAction` থাকলে নিজেই ২ column grid হয়ে যায় (বাঁয়ে title, ডানে action) |
| `CardTitle` | title (`text-sm text-muted-foreground` দিয়ে ছোট আর হালকা) |
| `CardAction` | header-এর ডান কোণ — এখানে icon box |
| `CardContent` | বড় সংখ্যা + description |

### Tailwind class

| Class | কাজ |
|---|---|
| `transition-shadow hover:shadow-md` | mouse নিলে হালকা ছায়া (ধীরে ধীরে) |
| `size-9 rounded-lg bg-primary/10 text-primary` | ৩৬px-এর গোল কোণের box, primary রঙের ১০% হালকা background, icon primary রঙে |
| `tabular-nums` | সব অঙ্ক সমান চওড়া → সংখ্যা বদলালে (যেমন `999` → `1,000`) লেখা এদিক-ওদিক লাফায় না |
| `aria-hidden` (icon) | শুধু সাজসজ্জা, screen reader title পড়বে |

### `StatsCardSkeleton` — loading-এর সময়

- Data আসার আগে `0` দেখালে user ভুল বুঝতে পারে ("আমার কোনো appointment নেই?")। তাই আসল card-এর **একই আকারের** ধূসর box দেখানো।
- একই `Card` / `CardHeader` / `CardAction` / `CardContent` কাঠামো → data এলে card-এর আকার বদলায় না, page লাফায় না।
- একই file-এ **named export** (`export function`), আর `StatsCard` হলো **default export**:
  ```ts
  import StatsCard, { StatsCardSkeleton } from "@/components/shared/cards/StatsCard";
  ```

---

## ধাপ ৩: ব্যবহার (পরে admin dashboard-এ)

```tsx
import StatsCard, { StatsCardSkeleton } from "@/components/shared/cards/StatsCard";
import { CalendarDays, Users } from "lucide-react";

// loading হলে
{isPending ? (
	<StatsCardSkeleton />
) : (
	<StatsCard
		title="Total Appointments"
		value={data?.appointmentCount ?? 0}
		icon={CalendarDays}
		description="Number of appointments scheduled"
	/>
)}

// টাকা: নিজে format করে string পাঠাও
<StatsCard title="Total Revenue" value={`৳ ${revenue.toLocaleString("en-US")}`} icon={Wallet} />
```

> ✏️ উদাহরণে `Wallet` icon ব্যবহার হয়েছে কিন্তু import-এ ছিল না। লাগবে: `import { CalendarDays, Users, Wallet } from "lucide-react";`

- `isPending` আসবে TanStack Query-র `useQuery` থেকে (note 05, 06) → তাই যে component এটা লিখবে সেটা **client component** (`"use client"`)। StatsCard-এ `"use client"` না থাকলেও সমস্যা নেই, client parent-এর ভিতরে এটা নিজেই client হিসেবে চলবে।
- `data?.appointmentCount ?? 0` → data না থাকলে (`undefined`) `0`।
- **টাকা:** `৳ ` আর comma parent নিজে বানিয়ে string পাঠাচ্ছে → card শুধু দেখায়।

### কয়েকটা card পাশাপাশি (grid)

> ✏️ এটা নোটে ছিল না — সাধারণত dashboard-এ এভাবে বসানো হয়:

```tsx
<div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
	<StatsCard title="Total Doctors" value={120} icon={Stethoscope} />
	<StatsCard title="Total Patients" value={3450} icon={Users} />
	<StatsCard title="Appointments" value={1250} icon={CalendarDays} />
	<StatsCard title="Revenue" value="৳ 5,40,000" icon={Wallet} />
</div>
```
Mobile-এ ১টা, `sm`-এ ২টা, `lg`-এ ৪টা card এক সারিতে।

---

## ধাপ ৪: Final structure (তোমার project-এর জন্য)

```
src/components/
├── ui/                          ← শুধু shadcn (হাত দেবে না)
│
├── shared/                      ← একাধিক জায়গায় লাগে, ধরন অনুযায়ী
│   ├── cards/
│   │   └── StatsCard.tsx
│   ├── charts/
│   │   ├── AppointmentBarChart.tsx
│   │   └── AppointmentPieChart.tsx
│   ├── form/                    ← আগে থেকেই আছে
│   │   ├── AppField.tsx
│   │   └── AppSubmitButton.tsx
│   ├── table/
│   │   ├── DataTable.tsx
│   │   ├── DataTableFilters.tsx
│   │   ├── DataTablePagination.tsx
│   │   ├── DataTableSearch.tsx
│   │   └── cells/               ← teacher-এর "cell", table-এর ভেতরে
│   │       ├── DateCell.tsx
│   │       ├── StatusBadgeCell.tsx
│   │       └── UserInfoCell.tsx
│   ├── layout/
│   │   ├── PublicNavbar.tsx
│   │   └── PublicFooter.tsx
│   └── LoadingSpinner.tsx
│
└── modules/                     ← feature অনুযায়ী
    ├── Auth/
    ├── Consultation/
    ├── Home/
    ├── Dashboard/               ← shell + প্রতিটা role-এর dashboard content
    │   ├── DashboardSidebar.tsx, DashboardNavbar.tsx, ...
    │   └── AdminDashboardContent.tsx   (পরে)
    ├── Admin/
    │   ├── DoctorsManagement/
    │   └── SchedulesManagement/
    ├── Doctor/
    └── Patient/
```

### নিয়মটা

| Folder | কী রাখবে | উদাহরণ |
|---|---|---|
| `ui/` | শুধু shadcn-এর component, **হাত দেবে না** (আবার `add` করলে বদলে যায়) | `button.tsx`, `card.tsx` |
| `shared/` | একাধিক feature-এ লাগে, **ধরন** অনুযায়ী folder | `cards/StatsCard`, `form/AppField`, `table/DataTable` |
| `modules/` | একটা নির্দিষ্ট **feature**-এর component | `Auth/LoginForm`, `Dashboard/DashboardSidebar` |

**প্রশ্ন করো:** "এটা কি শুধু এক feature-এর?" → `modules/<Feature>/`। "কয়েক জায়গায় লাগবে?" → `shared/<ধরন>/`।

### এখন project-এ কী আছে

| ✅ আছে | 🔜 পরে বানানো হবে |
|---|---|
| `shared/cards/StatsCard.tsx` | `shared/table/` (DataTable, filter, pagination, search, cells) |
| `shared/charts/AppointmentBarChart.tsx`, `AppointmentPieChart.tsx` | `shared/layout/` (PublicNavbar, PublicFooter) |
| `shared/form/AppField.tsx`, `AppSubmitButton.tsx` | `modules/Home/` |
| `shared/LoadingSpinner.tsx` | `modules/Dashboard/AdminDashboardContent.tsx` |
| `modules/Auth/`, `modules/Consultation/`, `modules/Dashboard/` | `modules/Admin/`, `modules/Doctor/`, `modules/Patient/` |

---

## 🧪 Test

এখনো কোনো dashboard-এ বসানো হয়নি। দেখতে চাইলে সাময়িক `src/app/(dashboardLayout)/admin/dashboard/page.tsx`-এ:

```tsx
import StatsCard, { StatsCardSkeleton } from "@/components/shared/cards/StatsCard";
import { CalendarDays, Users } from "lucide-react";

export default function AdminDashboardPage() {
	return (
		<div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-4">
			<StatsCard title="Total Patients" value={3450} icon={Users} description="All registered patients" />
			<StatsCard title="Appointments" value={1250} icon={CalendarDays} />
			<StatsCardSkeleton />
		</div>
	);
}
```

| # | কী দেখবে | ঠিক থাকলে |
|---|---|---|
| ১ | `3450` | `3,450` দেখায় |
| ২ | description ছাড়া card | নিচের লাইন নেই, ভাঙা দেখায় না |
| ৩ | Skeleton | আসল card-এর মতোই আকার, ধূসর ঝিলমিল |
| ৪ | Mouse নাও | হালকা ছায়া |
| ৫ | Browser ছোট-বড় করো | ১ → ২ → ৪ column |
| ৬ | Console | hydration mismatch-এর কোনো warning নেই |

> এটা Server Component page থেকে icon পাঠানো — চললে প্রমাণ হলো `"use client"` না দেওয়ার কারণটা ঠিক।

---

## 🔜 পরে যা যোগ হবে
- [ ] Backend-এর dashboard stats API (server note 34) থেকে data এনে admin/doctor/patient dashboard-এ card বসানো
- [ ] Chart component (`AppointmentBarChart`, `AppointmentPieChart`)
- [ ] `AdminDashboardContent.tsx` — card + chart একসাথে

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. ✅ (Done — note 15-এর `AdminDashboardContent`-এ `isError` + Try again) **Error হলে `0` দেখানো বিভ্রান্তিকর** — `data?.appointmentCount ?? 0`-এ API fail করলেও `0` দেখাবে, user ভাববে সত্যিই কিছু নেই। `isError` আলাদা করে দেখাও:
   ```tsx
   {isPending ? (
   	<StatsCardSkeleton />
   ) : isError ? (
   	<StatsCard title="Total Appointments" value="—" icon={CalendarDays} description="Could not load data" />
   ) : (
   	<StatsCard title="Total Appointments" value={data.appointmentCount} icon={CalendarDays} />
   )}
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

2. **Skeleton-এ description-এর লাইন সবসময় থাকে** — আসল card-এ description না থাকলে skeleton আসল card-এর চেয়ে একটু লম্বা দেখাবে, data এলে সামান্য ছোট হবে। চাইলে `StatsCardSkeleton`-এ একটা `withDescription` prop দিয়ে ঐ লাইন বাদ দিতে পারো। খুব ছোট বিষয়।

3. **টাকার জন্য `Intl` দিয়েও format করা যায়** — একই নিয়মে (locale fixed):
   ```ts
   const bdtFormatter = new Intl.NumberFormat("en-US", { style: "currency", currency: "BDT", maximumFractionDigits: 0 });
   bdtFormatter.format(540000); // "BDT 540,000"
   ```
   `৳` চিহ্ন চাইলে এখনকার মতো `৳ ${...}` লেখাই সহজ।
