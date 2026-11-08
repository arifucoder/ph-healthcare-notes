# 13 - Dashboard Pie (Donut) Chart — Recharts

> Module: 48-2 (Creating Pie Chart Component For Dashboard)

> Appointment-গুলো কোন status-এ কতটা (Scheduled, In Progress, Completed, Canceled), সেটা একটা **donut chart** দিয়ে দেখানো। মাঝখানে মোট সংখ্যা, পাশে legend-এ প্রতিটা status-এর আসল সংখ্যা আর শতাংশ। Loading-এর জন্য skeleton।

## 🧭 এক নজরে

```
Backend GET stats → pieChartData: [{ status: "SCHEDULED", count: 40 }, { status: "COMPLETED", count: 0 }, ...]
  → <AppointmentPieChart data={pieChartData} />   ("use client")
       ├─ total = সব count-এর যোগফল
       ├─ total 0?  → "No appointment data available yet."
       ├─ chartData = count > 0 গুলো → { name, value, fill }  → Recharts <Pie>
       ├─ মাঝখানে Total
       └─ Legend: সব status (0 সহ) → রং + নাম + সংখ্যা + %
```

### দেখতে কেমন

```
┌───────────────────────────────────────────────┐
│ Appointment Status                            │
│ Distribution of appointments by status        │
│                                               │
│      ╭─────╮          ■ Scheduled   40  40%   │
│    ╱  120  ╲          ■ In Progress 10  10%   │
│   │  Total  │         ■ Completed   45  45%   │
│    ╲       ╱          ■ Canceled     5   5%   │
│      ╰─────╯                                  │
└───────────────────────────────────────────────┘
   Donut (hover → tooltip)     Legend
```

---

## ধাপ ১: `recharts` install

```shell
pnpm add recharts
```
👉 https://recharts.org

- **Recharts** = React-এর chart library। Chart গুলো SVG দিয়ে আঁকা, React component-এর মতো লেখা যায় (`<PieChart><Pie /></PieChart>`)।
- Project-এ এখন `recharts` v3।
- shadcn-এর chart-ও ভিতরে Recharts ব্যবহার করে — আমরা shadcn-এর chart component না নিয়ে সরাসরি Recharts + নিজের Card দিয়ে বানিয়েছি।

---

## ধাপ ২: `src/app/globals.css` — chart-এর রং বদলাও

shadcn-এর default chart রং সব **ধূসর** (`oklch(... 0 0)` — শেষের `0` মানে কোনো রং নেই)। পাশাপাশি slice আলাদা করা যায় না, তাই রঙিন palette দিয়ে বদলানো।

আমি রংগুলো একটা validator দিয়ে পরীক্ষা করেছি। সাধারণ চোখে আর colorblind (বর্ণান্ধ) চোখে, light আর dark দুই mode-এই পাশাপাশি slice আলাদা করে চেনা যায়।

### `:root` (light mode) — line ~70–74

**আগে (মুছে দাও):**
```css
:root {
	--chart-1: oklch(0.87 0 0);
	--chart-2: oklch(0.556 0 0);
	--chart-3: oklch(0.439 0 0);
	--chart-4: oklch(0.371 0 0);
	--chart-5: oklch(0.269 0 0);
}
```

**এখন:**
```css
:root {
	--chart-1: #2a78d6; /* blue */
	--chart-2: #eb6834; /* orange */
	--chart-3: #1baf7a; /* aqua */
	--chart-4: #eda100; /* yellow */
	--chart-5: #e87ba4; /* magenta */
}
```

### `.dark` (dark mode)

```css
.dark {
	--chart-1: #3987e5;
	--chart-2: #d95926;
	--chart-3: #199e70;
	--chart-4: #c98500;
	--chart-5: #d55181;
}
```

> ✏️ তোমার নোটের CSS অংশ একটু জড়ানো ছিল ("eta replace koro" / "eta diye")। Project-এর `globals.css` দেখে আগে/পরে পরিষ্কার করে লেখা হয়েছে — মানগুলো হুবহু একই। শুধু `--chart-1` … `--chart-5` লাইনগুলো বদলাবে, `:root` / `.dark`-এর বাকি variable যেমন আছে থাকবে।

| Variable | Light | Dark | কোন status |
|---|---|---|---|
| `--chart-1` | `#2a78d6` 🔵 | `#3987e5` | Scheduled |
| `--chart-2` | `#eb6834` 🟠 | `#d95926` | In Progress |
| `--chart-3` | `#1baf7a` 🟢 | `#199e70` | Completed |
| `--chart-4` | `#eda100` 🟡 | `#c98500` | Canceled |
| `--chart-5` | `#e87ba4` 🩷 | `#d55181` | (এখন ব্যবহার নেই, পরে লাগলে) |

- **CSS variable কেন:** component-এ `var(--chart-1)` লিখলে mode বদলালে রং নিজেই বদলায়, component-এ কিছু করতে হয় না।
- Dark mode-এর রং একটু আলাদা, যাতে কালো background-এ চোখে লাগে না আর slice-এর মাঝের ফাঁক (`--card` রঙের) ঠিকমতো দেখা যায়।
- `@theme inline`-এ আগে থেকেই `--color-chart-1: var(--chart-1)` আছে, তাই Tailwind-এও `bg-chart-1`, `text-chart-3` এসব class ব্যবহার করা যায়।

---

## ধাপ ৩: `src/types/dashboard.types.ts` — শেষে যোগ করো

```ts
// backend-এর prisma enum AppointmentStatus-এর সাথে মিল রাখো
export type AppointmentStatus = "SCHEDULED" | "INPROGRESS" | "COMPLETED" | "CANCELED";

export interface PieChartData {
	status: AppointmentStatus;
	count: number;
}
```

- Backend-এর `prisma/schema/enums.prisma`-এর `enum AppointmentStatus`-এ ঠিক এই ৪টা মান (`SCHEDULED`, `INPROGRESS`, `COMPLETED`, `CANCELED`)। Frontend Prisma import করতে পারে না, তাই হাতে একই union লিখতে হয় — **বানান হুবহু** মেলাতে হবে।
- `PieChartData` = backend-এর stats API-এর `pieChartData`-এর একটা item।
- Backend (`stats.service.ts`-এর `getAppointmentStatusDistribution`) **সব status** পাঠায়, কোনোটার appointment না থাকলে `count: 0`। তাই frontend-এ legend-এ সবসময় ৪টা লাইন থাকে।

---

## ধাপ ৪: `src/components/shared/charts/AppointmentPieChart.tsx`

```tsx
"use client";

import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Skeleton } from "@/components/ui/skeleton";
import type { AppointmentStatus, PieChartData } from "@/types/dashboard.types";
import { Pie, PieChart, ResponsiveContainer, Tooltip } from "recharts";

// রং status-এর সাথে বাঁধা, array-র ক্রমের সাথে না।
// ক্রম বা সংখ্যা বদলালেও "Completed" সবসময় একই রঙে থাকে।
const STATUS_CONFIG: Record<AppointmentStatus, { label: string; color: string }> = {
	SCHEDULED: { label: "Scheduled", color: "var(--chart-1)" },
	INPROGRESS: { label: "In Progress", color: "var(--chart-2)" },
	COMPLETED: { label: "Completed", color: "var(--chart-3)" },
	CANCELED: { label: "Canceled", color: "var(--chart-4)" },
};

// locale fixed: server আর browser-এ একই output
const numberFormatter = new Intl.NumberFormat("en-US");
const percentFormatter = new Intl.NumberFormat("en-US", { style: "percent", maximumFractionDigits: 0 });

interface ChartItem {
	name: string;
	value: number;
	fill: string; // recharts প্রতিটা slice-এ এই রং নিজেই বসায়
}

interface AppointmentPieChartProps {
	data: PieChartData[];
	title?: string;
	description?: string;
	className?: string;
}

export default function AppointmentPieChart({
	data,
	title = "Appointment Status",
	description = "Distribution of appointments by status",
	className,
}: AppointmentPieChartProps) {
	const total = data.reduce((sum, item) => sum + item.count, 0);

	// 0-এর slice আঁকার কিছু নেই; তবে নিচের তালিকায় সব status দেখাবে
	const chartData: ChartItem[] = data
		.filter((item) => item.count > 0)
		.map((item) => ({
			name: STATUS_CONFIG[item.status].label,
			value: item.count,
			fill: STATUS_CONFIG[item.status].color,
		}));

	// screen reader-এর জন্য chart-এর লিখিত বিবরণ
	const summary = data.map((item) => `${STATUS_CONFIG[item.status].label}: ${item.count}`).join(", ");

	return (
		<Card className={className}>
			<CardHeader>
				<CardTitle>{title}</CardTitle>
				<CardDescription>{description}</CardDescription>
			</CardHeader>

			<CardContent>
				{total === 0 ? (
					<div className="flex h-64 items-center justify-center text-sm text-muted-foreground">
						No appointment data available yet.
					</div>
				) : (
					<div className="grid items-center gap-6 sm:grid-cols-[1fr_auto]">
						{/* Donut + মাঝখানে মোট সংখ্যা */}
						<div className="relative h-64" role="img" aria-label={`Appointment status. Total ${total}. ${summary}`}>
							<ResponsiveContainer width="100%" height="100%">
								<PieChart>
									<Pie
										data={chartData}
										dataKey="value"
										nameKey="name"
										innerRadius="62%"
										outerRadius="90%"
										stroke="var(--card)" // slice-এর মাঝে card-রঙের ফাঁক
										strokeWidth={2}
									/>
									<Tooltip
										cursor={false}
										content={({ active, payload }) => {
											const item = payload?.[0]?.payload as ChartItem | undefined;
											if (!active || !item) return null;

											return (
												<div className="rounded-lg border bg-popover px-3 py-2 text-xs text-popover-foreground shadow-md">
													<div className="flex items-center gap-2">
														<span className="size-2.5 rounded-sm" style={{ backgroundColor: item.fill }} />
														<span className="font-medium">{item.name}</span>
													</div>
													<p className="mt-1 tabular-nums text-muted-foreground">
														{numberFormatter.format(item.value)} ({percentFormatter.format(item.value / total)})
													</p>
												</div>
											);
										}}
									/>
								</PieChart>
							</ResponsiveContainer>

							<div className="pointer-events-none absolute inset-0 flex flex-col items-center justify-center">
								<span className="text-2xl font-bold tabular-nums">{numberFormatter.format(total)}</span>
								<span className="text-xs text-muted-foreground">Total</span>
							</div>
						</div>

						{/* Legend + আসল সংখ্যা: শুধু রং দেখে না, লেখা পড়েও বোঝা যায় */}
						<ul className="min-w-48 space-y-2 text-sm">
							{data.map((item) => {
								const config = STATUS_CONFIG[item.status];

								return (
									<li key={item.status} className="flex items-center gap-3">
										<span aria-hidden className="size-3 shrink-0 rounded-sm" style={{ backgroundColor: config.color }} />
										<span className="flex-1 text-muted-foreground">{config.label}</span>
										<span className="font-medium tabular-nums">{numberFormatter.format(item.count)}</span>
										<span className="w-10 text-right text-xs tabular-nums text-muted-foreground">
											{percentFormatter.format(item.count / total)}
										</span>
									</li>
								);
							})}
						</ul>
					</div>
				)}
			</CardContent>
		</Card>
	);
}

// data load হওয়ার সময়
export function AppointmentPieChartSkeleton({ className }: { className?: string }) {
	return (
		<Card className={className}>
			<CardHeader>
				<Skeleton className="h-5 w-40" />
				<Skeleton className="h-4 w-56" />
			</CardHeader>
			<CardContent className="grid items-center gap-6 sm:grid-cols-[1fr_auto]">
				<Skeleton className="mx-auto size-56 rounded-full" />
				<div className="min-w-48 space-y-3">
					{Array.from({ length: 4 }, (_, index) => (
						<Skeleton key={index} className="h-4 w-full" />
					))}
				</div>
			</CardContent>
		</Card>
	);
}

```

---

## ধাপ ৫: ভেঙে বোঝা

### কেন `"use client"`?
Recharts browser-এ চলে: `ResponsiveContainer` parent-এর আকার মেপে chart আঁকে, tooltip mouse-এর নড়াচড়া শোনে — এগুলো server-এ সম্ভব না।

**StatsCard (note 12)-এর সাথে পার্থক্য:** এই component শুধু `data` (সাধারণ array — serializable) নেয়, কোনো function/component prop নেয় না। তাই `"use client"` থাকলেও Server Component থেকে নিশ্চিন্তে ব্যবহার করা যায়।

### `STATUS_CONFIG` — রং status-এর সাথে বাঁধা ⭐

```ts
const STATUS_CONFIG: Record<AppointmentStatus, { label: string; color: string }> = { ... };
```

| ভুল উপায় | এই উপায় |
|---|---|
| `COLORS = ["blue", "orange", ...]` আর `COLORS[index]` | `STATUS_CONFIG[item.status].color` |
| Backend ক্রম বদলালে বা 0-এর slice বাদ দিলে **রং সরে যায়** — আজ "Completed" সবুজ, কাল কমলা | "Completed" **সবসময়** সবুজ |

- `Record<AppointmentStatus, ...>` → ৪টা status-এর প্রতিটার জন্য config থাকতেই হবে। Backend-এ নতুন status যোগ করে frontend-এর type-এ যোগ করলে এখানে না লিখলে TypeScript error।
- `label` → `"INPROGRESS"`-এর বদলে মানুষের পড়ার মতো `"In Progress"`।

### Formatter (locale fixed)
- `numberFormatter` → `1250` → `"1,250"` (note 12-এর মতো, server আর browser-এ একই output)।
- `percentFormatter` → `{ style: "percent", maximumFractionDigits: 0 }` → `0.4` → `"40%"`। নিজে `* 100` আর `"%"` জুড়তে হয় না।

### Data তৈরি — ৩টা জিনিস

```ts
const total = data.reduce((sum, item) => sum + item.count, 0);
```
**`reduce`** → array-এর সব `count` যোগ করে একটা সংখ্যা (শুরু `0` থেকে)।

```ts
const chartData = data.filter((item) => item.count > 0).map((item) => ({ name, value, fill }));
```
- `filter` → 0-এর slice আঁকার কিছু নেই (Recharts 0-এর জন্য ফাঁকা দাগ আঁকতে পারে)।
- `map` → Recharts যে shape চায় সেভাবে: `name` (tooltip-এ নাম), `value` (slice-এর আকার), `fill` (রং — Recharts প্রতিটা slice-এ data-র `fill` নিজেই বসায়, আলাদা `<Cell>` লাগে না)।
- কিন্তু **legend-এ `data` (সব status)** ব্যবহার হয়, `chartData` না → 0-এর status-ও তালিকায় দেখায়।

```ts
const summary = data.map((item) => `${label}: ${count}`).join(", ");
```
→ `"Scheduled: 40, In Progress: 10, Completed: 45, Canceled: 5"` — screen reader-এর জন্য।

### খালি অবস্থা
`total === 0` হলে chart না এঁকে একটা লেখা। একই উচ্চতা (`h-64`) → card-এর আকার ঠিক থাকে। (০ দিয়ে ভাগ করে `NaN%` দেখানোর ঝামেলাও এড়ানো যায়।)

### Layout
`grid sm:grid-cols-[1fr_auto]` → mobile-এ chart-এর নিচে legend, `sm` থেকে পাশাপাশি (chart বাকি জায়গা, legend যতটুকু লাগে)।

### `<Pie>` props

| Prop | কাজ |
|---|---|
| `dataKey="value"` | কোন field দিয়ে slice-এর আকার |
| `nameKey="name"` | কোন field নাম |
| `innerRadius="62%"` | মাঝখান ফাঁকা → **donut**। `0` হলে সাধারণ pie |
| `outerRadius="90%"` | বাইরের মাপ, একটু জায়গা রেখে (hover-এ কাটা না যায়) |
| `stroke="var(--card)"` + `strokeWidth={2}` | প্রতিটা slice-এর চারপাশে card-এর রঙের দাগ → slice-এর মাঝে **ফাঁক** দেখায়, পাশাপাশি রং আলাদা বোঝা যায় |

### `ResponsiveContainer`
`width="100%" height="100%"` → parent div-এর মাপ নেয়। তাই parent-এ উচ্চতা দিতেই হবে (`h-64`), নাহলে chart-এর উচ্চতা 0 → কিছুই দেখা যায় না।

### Custom `Tooltip`

```tsx
<Tooltip cursor={false} content={({ active, payload }) => { ... }} />
```
- Recharts-এর default tooltip দেখতে আলাদা, আমাদের theme-এর সাথে মেলে না। তাই `content`-এ নিজের JSX → `bg-popover`, `border`, `shadow-md` (shadcn-এর popover-এর মতো)।
- `active` → mouse কোনো slice-এর উপর আছে কিনা।
- `payload?.[0]?.payload` → hover করা slice-এর আসল data (আমাদের `ChartItem`)। `?.` কারণ শুরুতে `payload` খালি থাকে।
- `cursor={false}` → hover-এ পেছনের হাইলাইট বন্ধ।
- দেখায়: রঙের ছোট box + নাম + `45 (45%)`।

### মাঝখানে Total

```tsx
<div className="pointer-events-none absolute inset-0 flex flex-col items-center justify-center">
```
- Parent `relative`, এটা `absolute inset-0` → chart-এর ঠিক উপরে পুরো জায়গা জুড়ে, আর `flex` দিয়ে একদম মাঝে।
- **`pointer-events-none`** → এই লেখা mouse ধরে না, তাই নিচের slice-এ hover করলে tooltip ঠিকমতো আসে।

### Accessibility — শুধু রং দেখে না

| অংশ | কেন |
|---|---|
| `role="img"` + `aria-label` | Screen reader-এর কাছে পুরো chart একটা ছবি, যার বিবরণ: "Appointment status. Total 100. Scheduled: 40, ..." |
| Legend-এ আসল সংখ্যা + % | রং চিনতে না পারলেও (colorblind) লেখা পড়ে বোঝা যায় |
| Legend-এর রঙের box-এ `aria-hidden` | box শুধু সাজসজ্জা, পাশে নাম আছে |

### Legend
- `key={item.status}` → status unique, তাই key হিসেবে ভালো।
- `flex-1` (নাম) → নাম বাকি জায়গা নেয়, সংখ্যা ডানে ঠেলে যায়।
- `w-10 text-right` (%) → সব শতাংশ একই চওড়া জায়গায় ডানে সারিবদ্ধ।

### `AppointmentPieChartSkeleton`
- একই Card কাঠামো: title/description-এর জায়গায় ২টা লাইন, গোল (`rounded-full`) skeleton chart-এর জায়গায়, ৪টা লাইন legend-এর জায়গায়।
- `Array.from({ length: 4 }, (_, index) => ...)` → ৪টা element-এর array বানিয়ে map — একই JSX ৪ বার না লিখে।
- `className` prop → grid-এ `lg:col-span-2` দিলে আসল chart যেখানে বসবে, skeleton-ও ঠিক সেখানে।

---

## ধাপ ৬: ব্যবহার (পরে AdminDashboardContent-এ)

```tsx
import AppointmentPieChart, { AppointmentPieChartSkeleton } from "@/components/shared/charts/AppointmentPieChart";

{isPending ? (
	<AppointmentPieChartSkeleton className="lg:col-span-2" />
) : (
	<AppointmentPieChart data={data?.pieChartData ?? []} className="lg:col-span-2" />
)}
```

- `isPending` → TanStack Query (তাই parent client component)।
- `className="lg:col-span-2"` → বড় screen-এ grid-এর ২ column জুড়ে।
- `title`, `description`-এর default মান আছে, তাই না দিলেও চলে। Doctor-এর dashboard-এ চাইলে `title="My Appointments"` দিয়ে বদলানো যায়।

---

## 🧪 Test

এখনো কোনো dashboard-এ বসানো হয়নি। সাময়িক `admin/dashboard/page.tsx`-এ (Server Component থেকেই চলবে):

```tsx
import AppointmentPieChart, { AppointmentPieChartSkeleton } from "@/components/shared/charts/AppointmentPieChart";

export default function AdminDashboardPage() {
	return (
		<div className="grid gap-4 lg:grid-cols-2">
			<AppointmentPieChart
				data={[
					{ status: "SCHEDULED", count: 40 },
					{ status: "INPROGRESS", count: 10 },
					{ status: "COMPLETED", count: 45 },
					{ status: "CANCELED", count: 0 },
				]}
			/>
			<AppointmentPieChartSkeleton />
		</div>
	);
}
```

| # | কী দেখবে | ঠিক থাকলে |
|---|---|---|
| ১ | Donut-এর মাঝে | `95` / Total |
| ২ | Slice | ৩টা (Canceled 0, তাই নেই), মাঝে সাদা ফাঁক |
| ৩ | Legend | ৪টা লাইন, Canceled `0` `0%` |
| ৪ | Slice-এ hover | নিজের রঙের tooltip: `Completed 45 (47%)` |
| ৫ | মাঝের Total-এর উপর দিয়ে mouse নিয়ে slice-এ | tooltip ঠিক আসে |
| ৬ | সব count `0` দাও | "No appointment data available yet." |
| ৭ | Browser ছোট করো | legend chart-এর নিচে চলে যায় |
| ৮ | `<html className="dark">` দিয়ে দেখো | dark-এর রং, ফাঁক card-এর রঙে |
| ৯ | Console | hydration বা `width(0) height(0)` warning নেই |

---

## 🔜 পরে যা যোগ হবে
- [x] Bar chart (`AppointmentBarChart`) — note 14
- [ ] `AdminDashboardContent.tsx` — stats card + pie + bar একসাথে, backend stats API থেকে data
- [ ] Doctor / Patient dashboard-এ একই chart (নিজের appointment)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Backend নতুন status পাঠালে page crash করবে** — backend-এ নতুন status (যেমন `"NO_SHOW"`) যোগ হলো কিন্তু frontend-এর type/config এখনো আপডেট হয়নি → `STATUS_CONFIG["NO_SHOW"]` হবে `undefined` → `.label` পড়তে গিয়ে পুরো dashboard error। TypeScript এটা ধরতে পারে না (runtime data)। অচেনা status বাদ দিলে নিরাপদ:
   ```ts
   const knownData = data.filter((item) => item.status in STATUS_CONFIG);
   // তারপর total, chartData, summary, legend — সব জায়গায় data-র বদলে knownData
   ```

2. ✅ (Done — note 15) **Error হলে "No appointment data" দেখায়** — `data?.pieChartData ?? []`-এ API fail করলেও খালি array → "No appointment data available yet."। User ভাববে সত্যিই কোনো appointment নেই। Note 12-এর মতো `isError` আলাদা দেখাও।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **শতাংশের যোগফল সবসময় ১০০% হয় না** — round করার কারণে (যেমন 33%, 33%, 33% = 99%)। এটা স্বাভাবিক, সব chart-এ হয়। মোট সংখ্যা মাঝখানে আর আসল সংখ্যা legend-এ আছে, তাই বিভ্রান্তি কম।

4. **Mobile-এ tooltip** — touch screen-এ hover নেই, slice-এ tap করলে tooltip আসে। তবে legend-এ সব সংখ্যা আগেই লেখা আছে, তাই mobile user কিছু হারায় না।
