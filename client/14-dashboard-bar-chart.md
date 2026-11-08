# 14 - Dashboard Bar Chart — মাস অনুযায়ী Appointment (Recharts)

> Module: 48-3 (Creating Bar Chart Component For Dashboard)

> গত ১২ মাসে প্রতি মাসে কতগুলো appointment হয়েছে, সেটা একটা **bar chart** দিয়ে দেখানো। Pie chart (note 13)-এর মতোই কাঠামো: Card + Recharts + custom tooltip + skeleton। নতুন জিনিস: **তারিখ format** (UTC), X/Y axis, grid।

## 🧭 এক নজরে

```
Backend stats → barChartData: [{ month: "2025-12-01T00:00:00.000Z", count: 3 }, ... ১২টা]
  → <AppointmentBarChart data={barChartData} />   ("use client")
       ├─ প্রতিটা month (ISO string) → new Date() → "Dec 25" (axis) + "December 2025" (tooltip)
       ├─ সব মাসে 0?  → "No appointment data available yet."
       └─ Recharts <BarChart>: grid + X axis (মাস) + Y axis (সংখ্যা) + tooltip + bar
```

### দেখতে কেমন

```
┌──────────────────────────────────────────────────────┐
│ Appointment Trends                                   │
│ Appointments per month, last 12 months               │
│                                                      │
│ 30 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│ 20 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▆▆ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▇▇ ─  │
│ 10 ─ ▃▃ ─ ─ ▅▅ ─ ─ ─ ─ ─ ██ ▄▄ ─ ─ ─ ─ ▆▆ ─ ─ ██ ─  │
│  0 ─ ██ ── ██ ▂▂ ── ── ─ ██ ██ ▃▃ ── ── ██ ▄▄ ██ ─  │
│     Dec  Jan  Feb  Mar  Apr  May  Jun  Jul ... Nov   │
└──────────────────────────────────────────────────────┘
```

> ✏️ এই নোটে নতুন package লাগে না — `recharts` আগেই install করা (note 13), chart-এর রং-ও আগেই বদলানো (`--chart-1` নীল)।

---

## ধাপ ১: `src/types/dashboard.types.ts` — যোগ করো

```ts
// API JSON-এ Date আসে না, ISO string আসে: "2026-11-01T00:00:00.000Z"
export interface BarChartData {
	month: string;
	count: number;
}
```

### `month: string` কেন, `Date` না?

- Backend-এ (`stats.service.ts`-এর `getMonthlyAppointmentCount`) PostgreSQL প্রতিটা মাসের **শুরুর মুহূর্ত** দেয় (Date)।
- কিন্তু API দিয়ে আসার সময় সব কিছু **JSON** হয়ে আসে, আর JSON-এ `Date` বলে কিছু নেই → `res.json()` Date-কে **ISO string** বানিয়ে পাঠায়: `"2026-11-01T00:00:00.000Z"`।
- Frontend-এ type `Date` লিখলে TypeScript ভাববে `.getMonth()` চলবে, কিন্তু আসলে string → runtime error। তাই সত্যি যা আসে, type-এ তাই: `string`। দরকার হলে component-এ নিজে `new Date(...)`।

### Backend কী পাঠায়

```sql
generate_series(DATE_TRUNC('month', NOW()) - INTERVAL '11 months', DATE_TRUNC('month', NOW()), INTERVAL '1 month')
LEFT JOIN appointments ...
```
- `generate_series` → গত ১১ মাস + এই মাস = **সবসময় ১২টা মাস**।
- `LEFT JOIN` → যে মাসে কোনো appointment নেই, সেটাও আসে `count: 0` নিয়ে। তাই chart-এ মাস বাদ পড়ে না, ফাঁকা মাসে bar-ও থাকে না।
- `ORDER BY month ASC` → পুরনো থেকে নতুন, frontend-এ আবার sort করতে হয় না।

---

## ধাপ ২: `src/components/shared/charts/AppointmentBarChart.tsx`

```tsx
"use client";

import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Skeleton } from "@/components/ui/skeleton";
import type { BarChartData } from "@/types/dashboard.types";
import { Bar, BarChart, CartesianGrid, ResponsiveContainer, Tooltip, XAxis, YAxis } from "recharts";

// timeZone: "UTC": backend মাসের শুরু UTC-তে পাঠায়।
// local সময়ে format করলে UTC-র পেছনের দেশে "Oct" দেখাত, আর server/browser-এ আলাদা হতো।
const axisMonthFormatter = new Intl.DateTimeFormat("en-US", { month: "short", year: "2-digit", timeZone: "UTC" });
const fullMonthFormatter = new Intl.DateTimeFormat("en-US", { month: "long", year: "numeric", timeZone: "UTC" });
const numberFormatter = new Intl.NumberFormat("en-US");

const TICK_STYLE = { fill: "var(--muted-foreground)", fontSize: 12 };

interface ChartItem {
	label: string; // axis-এ: "Nov 26"
	fullLabel: string; // tooltip-এ: "November 2026"
	appointments: number;
}

interface AppointmentBarChartProps {
	data: BarChartData[];
	title?: string;
	description?: string;
	className?: string;
}

export default function AppointmentBarChart({
	data,
	title = "Appointment Trends",
	description = "Appointments per month, last 12 months",
	className,
}: AppointmentBarChartProps) {
	const chartData: ChartItem[] = data.map((item) => {
		const date = new Date(item.month);
		return {
			label: axisMonthFormatter.format(date),
			fullLabel: fullMonthFormatter.format(date),
			appointments: item.count,
		};
	});

	const hasData = chartData.some((item) => item.appointments > 0);

	// screen reader-এর জন্য chart-এর লিখিত বিবরণ
	const summary = chartData.map((item) => `${item.fullLabel}: ${item.appointments}`).join(", ");

	return (
		<Card className={className}>
			<CardHeader>
				<CardTitle>{title}</CardTitle>
				<CardDescription>{description}</CardDescription>
			</CardHeader>

			<CardContent>
				{!hasData ? (
					<div className="flex h-72 items-center justify-center text-sm text-muted-foreground">
						No appointment data available yet.
					</div>
				) : (
					<div className="h-72 sm:h-80" role="img" aria-label={`Monthly appointments. ${summary}`}>
						<ResponsiveContainer width="100%" height="100%">
							<BarChart data={chartData} margin={{ top: 8, right: 8, left: 0, bottom: 0 }}>
								{/* হালকা, শুধু অনুভূমিক grid: data-কে প্রাধান্য দেয় */}
								<CartesianGrid vertical={false} stroke="var(--border)" />

								<XAxis
									dataKey="label"
									tickLine={false}
									axisLine={false}
									tickMargin={8}
									minTickGap={8} // জায়গা কম হলে কিছু label নিজে থেকে লুকায়
									tick={TICK_STYLE}
								/>

								<YAxis
									tickLine={false}
									axisLine={false}
									allowDecimals={false} // ২.৫টা appointment হয় না
									width={32}
									tick={TICK_STYLE}
								/>

								<Tooltip
									cursor={{ fill: "var(--muted)", opacity: 0.6 }}
									content={({ active, payload }) => {
										const item = payload?.[0]?.payload as ChartItem | undefined;
										if (!active || !item) return null;

										return (
											<div className="rounded-lg border bg-popover px-3 py-2 text-xs text-popover-foreground shadow-md">
												<p className="font-medium">{item.fullLabel}</p>
												<p className="mt-1 tabular-nums text-muted-foreground">
													{numberFormatter.format(item.appointments)} appointments
												</p>
											</div>
										);
									}}
								/>

								<Bar dataKey="appointments" fill="var(--chart-1)" radius={[4, 4, 0, 0]} maxBarSize={48} />
							</BarChart>
						</ResponsiveContainer>
					</div>
				)}
			</CardContent>
		</Card>
	);
}

// skeleton-এ bar-এর উচ্চতা fixed (random দিলে server/browser-এ আলাদা হয়ে hydration mismatch হবে)
const SKELETON_BAR_HEIGHTS = ["40%", "65%", "50%", "80%", "35%", "70%", "55%", "90%", "45%", "60%", "75%", "50%"];

export function AppointmentBarChartSkeleton({ className }: { className?: string }) {
	return (
		<Card className={className}>
			<CardHeader>
				<Skeleton className="h-5 w-40" />
				<Skeleton className="h-4 w-56" />
			</CardHeader>
			<CardContent>
				<div className="flex h-72 items-end gap-2 sm:h-80">
					{SKELETON_BAR_HEIGHTS.map((height, index) => (
						<Skeleton key={index} className="flex-1 rounded-b-none" style={{ height }} />
					))}
				</div>
			</CardContent>
		</Card>
	);
}
```

---

## ধাপ ৩: ভেঙে বোঝা

### তারিখ format — `timeZone: "UTC"` কেন জরুরি ⭐

```ts
const axisMonthFormatter = new Intl.DateTimeFormat("en-US", { month: "short", year: "2-digit", timeZone: "UTC" });
const fullMonthFormatter = new Intl.DateTimeFormat("en-US", { month: "long", year: "numeric", timeZone: "UTC" });
```

Backend পাঠায় `"2026-11-01T00:00:00.000Z"` = **UTC সময়ে** নভেম্বরের ১ তারিখ রাত ১২টা। শেষের `Z` মানে UTC।

| কোথায় format করা হচ্ছে | `timeZone` না দিলে (local সময়) | `timeZone: "UTC"` দিলে |
|---|---|---|
| বাংলাদেশ (UTC+6) | ১ নভেম্বর সকাল ৬টা → **Nov** ✅ | Nov ✅ |
| আমেরিকা (UTC-5) | ৩১ **অক্টোবর** সন্ধ্যা ৭টা → **Oct** ❌ | Nov ✅ |
| Server (Node, যেকোনো timezone) | server-এর timezone অনুযায়ী | Nov ✅ |

- UTC-র **পেছনের** দেশে (আমেরিকা) প্রতিটা মাস এক মাস আগের নামে দেখাত।
- Server আর browser-এর timezone আলাদা হলে দুই জায়গায় আলাদা লেখা → **hydration mismatch**।
- `timeZone: "UTC"` → backend যে timezone-এ পাঠিয়েছে, সেভাবেই পড়া → সবখানে একই।
- `"en-US"` fixed → note 12-এর মতো, locale-এর কারণে আলাদা হয় না।

| Formatter | Output | কোথায় |
|---|---|---|
| `axisMonthFormatter` (`short`, `2-digit`) | `"Nov 26"` | X axis-এ (জায়গা কম) |
| `fullMonthFormatter` (`long`, `numeric`) | `"November 2026"` | tooltip আর screen reader-এ |

**Year কেন দেখানো?** ১২ মাসের মধ্যে দুটো বছর থাকে (Dec 25 … Nov 26)। শুধু "Dec" লিখলে কোন বছরের বোঝা যেত না।

### `TICK_STYLE`
```ts
const TICK_STYLE = { fill: "var(--muted-foreground)", fontSize: 12 };
```
X আর Y দুই axis-এর লেখার একই style — একবার লিখে দুই জায়গায়। `var(--muted-foreground)` → হালকা রং, dark mode-এ নিজেই বদলায়।

### Data তৈরি

```ts
const chartData = data.map((item) => {
	const date = new Date(item.month);
	return { label: ..., fullLabel: ..., appointments: item.count };
});
```
- প্রতিটা item থেকে একবারই `Date` বানিয়ে দুটো label।
- `count` → `appointments` নাম কেন? Recharts-এর default tooltip/legend-এ `dataKey`-এর নামই দেখায় — অর্থবোধক নাম রাখা ভালো অভ্যাস।

```ts
const hasData = chartData.some((item) => item.appointments > 0);
```
- **`some`** → অন্তত একটা মাসে appointment থাকলেই `true`।
- Pie chart-এ `total === 0` check করা হয়েছিল। এখানে `data.length === 0` দিয়ে check করলে হবে না — backend **সবসময় ১২টা মাস পাঠায়** (0 সহ), তাই array কখনো খালি না। ১২টা ফাঁকা bar-এর চেয়ে "No data" লেখা ভালো।

### `<BarChart>`-এর অংশ

| অংশ | Props | কাজ |
|---|---|---|
| `BarChart` | `data`, `margin={{ top: 8, right: 8, left: 0, bottom: 0 }}` | chart-এর চারপাশে একটু জায়গা, বাঁয়ে 0 (Y axis-এর জায়গা আলাদা) |
| `CartesianGrid` | `vertical={false}`, `stroke="var(--border)"` | শুধু **অনুভূমিক** হালকা দাগ → সংখ্যা মেলানো সহজ, কিন্তু চোখে data-ই আগে পড়ে |
| `XAxis` | `dataKey="label"` | নিচে মাসের নাম |
| | `tickLine={false}` `axisLine={false}` | ছোট কাঁটা আর axis-এর দাগ নেই → পরিষ্কার |
| | `tickMargin={8}` | লেখা আর chart-এর মাঝে ফাঁক |
| | `minTickGap={8}` | ছোট screen-এ জায়গা না থাকলে কিছু label নিজে থেকে লুকায় (একটার উপর আরেকটা পড়ে না) |
| `YAxis` | `allowDecimals={false}` | `2.5`-এর মতো ভগ্নাংশ দাগ নেই — ২.৫টা appointment হয় না |
| | `width={32}` | Y axis-এর জন্য ৩২px (default বেশি জায়গা নেয়) |
| `Tooltip` | `cursor={{ fill: "var(--muted)", opacity: 0.6 }}` | hover করা মাসের পেছনে হালকা ধূসর column |
| | `content={...}` | নিজের tooltip (pie-এর মতোই theme): `November 2026` / `25 appointments` |
| `Bar` | `dataKey="appointments"` | কোন field দিয়ে bar-এর উচ্চতা |
| | `fill="var(--chart-1)"` | নীল (note 13-এ বদলানো রং) |
| | `radius={[4, 4, 0, 0]}` | উপরের দুই কোণ গোল, নিচের দুই কোণ সোজা (ক্রম: উপর-বাম, উপর-ডান, নিচ-ডান, নিচ-বাম) |
| | `maxBarSize={48}` | data কম থাকলে bar অনেক মোটা হয়ে যায় না |

### Height
`h-72 sm:h-80` → mobile-এ ২৮৮px, বড় screen-এ ৩২০px। `ResponsiveContainer`-এর জন্য parent-এ উচ্চতা বাধ্যতামূলক (note 13)। খালি অবস্থার div-ও `h-72` → card-এর আকার একই থাকে।

### Accessibility
`role="img"` + `aria-label` → "Monthly appointments. December 2025: 3, January 2026: 0, …" — screen reader পুরো chart-এর তথ্য পড়ে শোনায় (tooltip-এর `fullLabel` এখানেও কাজে লাগল)।

### `AppointmentBarChartSkeleton`

```ts
const SKELETON_BAR_HEIGHTS = ["40%", "65%", "50%", ...]; // ১২টা
```
- Loading-এর সময় ১২টা ধূসর bar → দেখেই বোঝা যায় এখানে bar chart আসবে।
- **উচ্চতা fixed কেন, `Math.random()` না?** Server-এ একবার random, browser-এ আরেকবার random → আলাদা HTML → **hydration mismatch**। Fixed list দিলে দুই জায়গায় একই।
- `flex items-end gap-2` → bar-গুলো নিচে সারিবদ্ধ, `flex-1` → সব সমান চওড়া।
- `rounded-b-none` → আসল bar-এর মতো শুধু উপরে গোল।
- `style={{ height }}` → উচ্চতা আলাদা আলাদা, তাই Tailwind class না দিয়ে inline style।
- `key={index}` → list কখনো বদলায় না (fixed), তাই index key ঠিক আছে।

---

## ধাপ ৪: ব্যবহার (পরে AdminDashboardContent-এ)

```tsx
import AppointmentBarChart, { AppointmentBarChartSkeleton } from "@/components/shared/charts/AppointmentBarChart";

{isPending ? (
	<AppointmentBarChartSkeleton className="lg:col-span-4" />
) : (
	<AppointmentBarChart data={data?.barChartData ?? []} className="lg:col-span-4" />
)}
```

- `lg:col-span-4` → বড় screen-এ grid-এর ৪ column জুড়ে (bar chart চওড়া হলে ভালো দেখায়)। Pie chart `lg:col-span-2` (note 13) — দুটো মিলিয়ে ৬ column-এর grid হতে পারে:

```
lg grid (৬ column):
┌──────────────────────────────┬──────────────────┐
│ Bar chart (col-span-4)       │ Pie (col-span-2) │
└──────────────────────────────┴──────────────────┘
```

> ✏️ এটা নোটে ছিল না, শুধু ধারণা: parent-এ `className="grid gap-4 lg:grid-cols-6"`।

---

## 📊 Pie বনাম Bar — তুলনা

| | Pie (note 13) | Bar (এই নোট) |
|---|---|---|
| কী দেখায় | ভাগ (কোন status কত %) | সময়ের সাথে বদল (মাস অনুযায়ী) |
| Data | `{ status, count }[]` | `{ month, count }[]` |
| রং | status অনুযায়ী ৪টা | একটাই (`--chart-1`) |
| খালি check | `total === 0` | `some(count > 0)` না হলে |
| বিশেষ | মাঝে Total, legend | UTC তারিখ format, axis, grid |
| Skeleton | গোল + ৪টা লাইন | fixed উচ্চতার ১২টা bar |

---

## 🧪 Test

সাময়িক `admin/dashboard/page.tsx`-এ:

```tsx
import AppointmentBarChart, { AppointmentBarChartSkeleton } from "@/components/shared/charts/AppointmentBarChart";

const sample = [
	{ month: "2025-12-01T00:00:00.000Z", count: 3 },
	{ month: "2026-01-01T00:00:00.000Z", count: 0 },
	{ month: "2026-02-01T00:00:00.000Z", count: 12 },
	{ month: "2026-11-01T00:00:00.000Z", count: 25 },
];

export default function AdminDashboardPage() {
	return (
		<div className="grid gap-4">
			<AppointmentBarChart data={sample} />
			<AppointmentBarChartSkeleton />
		</div>
	);
}
```

| # | কী দেখবে | ঠিক থাকলে |
|---|---|---|
| ১ | X axis | `Dec 25`, `Jan 26`, `Feb 26`, `Nov 26` |
| ২ | `Jan 26` | কোনো bar নেই (0) |
| ৩ | Y axis | শুধু পূর্ণ সংখ্যা (0, 5, 10…), ভগ্নাংশ নেই |
| ৪ | Bar-এ hover | পেছনে ধূসর column + `November 2026` / `25 appointments` |
| ৫ | Browser অনেক ছোট করো | কিছু মাসের label লুকায়, একটার উপর আরেকটা পড়ে না |
| ৬ | সব `count: 0` দাও | "No appointment data available yet." |
| ৭ | DevTools → Sensors → Location/timezone `America/New_York` করে reload | তবুও `Nov 26` (Oct না) |
| ৮ | Skeleton | ১২টা ভিন্ন উচ্চতার ধূসর bar, reload করলেও একই |
| ৯ | Console | hydration warning নেই |

---

## 🔜 পরে যা যোগ হবে
- [ ] `AdminDashboardContent.tsx` — backend stats API থেকে data এনে StatsCard + Bar + Pie একসাথে
- [ ] Doctor / Patient dashboard-এ নিজের appointment-এর chart
- [ ] চাইলে "গত ৬ মাস / ১২ মাস" filter

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **ভাঙা তারিখ এলে পুরো dashboard crash করবে** — `month`-এ ভুল string এলে `new Date("bad")` হয় `Invalid Date`, আর `Intl.DateTimeFormat.format(Invalid Date)` **`RangeError` throw** করে → page error। Backend ঠিক থাকলে হবে না, কিন্তু নিরাপদ থাকতে ভাঙা item বাদ দাও:
   ```ts
   const chartData: ChartItem[] = data
   	.map((item) => ({ date: new Date(item.month), count: item.count }))
   	.filter(({ date }) => !Number.isNaN(date.getTime()))
   	.map(({ date, count }) => ({
   		label: axisMonthFormatter.format(date),
   		fullLabel: fullMonthFormatter.format(date),
   		appointments: count,
   	}));
   ```

2. ✅ (Done — note 15) **Error হলে "No data" দেখায়** — `data?.barChartData ?? []`-এ API fail করলেও খালি → "No appointment data available yet."। Note 12, 13-এর মতো `isError` আলাদা দেখাও।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **UTC ধরে নেওয়াটা database-এর timezone-এর উপর নির্ভর করে** — Backend-এর `DATE_TRUNC('month', NOW())` PostgreSQL-এর session timezone অনুযায়ী মাসের শুরু বানায়। Database UTC-তে থাকলে (সাধারণত তাই থাকে) মাস আসে `...-01T00:00:00.000Z` → এখানকার `timeZone: "UTC"` ঠিক। কিন্তু database-এর timezone যদি `Asia/Dhaka` হয়, তাহলে নভেম্বরের শুরু আসবে `"2026-10-31T18:00:00.000Z"` → frontend UTC-তে পড়ে দেখাবে **Oct** (untested)। Chart-এ মাস এক মাস পিছিয়ে দেখালে প্রথমে database-এর timezone check করো (`SHOW timezone;`)।

4. **`count` আর `appointments` দুই নাম** — backend-এ `count`, chart-এ `appointments`। এখন কোনো সমস্যা নেই। শুধু মনে রাখো, `<Bar dataKey="appointments">` বদলালে `ChartItem`-এর field-ও বদলাতে হবে।
