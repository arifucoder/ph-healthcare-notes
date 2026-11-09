# 24 - Doctor Details Page (Public Profile + Reviews)

> Public `/consultation/doctor/[id]` page: doctor-এর profile, about, সাম্প্রতিক ৫টা review, আর ডান পাশে booking card (এখন placeholder, পরের ধাপে আসল booking)।
> আগের note: 23 (consultation list থেকে card-এ ক্লিক করে এখানে আসা)।

```
/consultation/doctor/:id  →  UUID check  →  Promise.all( GET /doctors/:id , GET /reviews?doctorId=.. )  →  Profile + Booking + Reviews
                                 ↓ ভুল id / 404                         ↓ review fail → null (page ভাঙে না)
                              not-found.tsx                          ↓ বাকি error → error.tsx
```

## ফাইলগুলো

```
src/types/review.types.ts                       (নতুন)
src/services/doctor.services.ts                 (একটা function যোগ)
src/services/review.services.ts                 (নতুন)
src/components/shared/StarRating.tsx            (নতুন)
src/components/modules/Consultation/DoctorDetails/
  ├─ DoctorProfile.tsx
  ├─ DoctorReviews.tsx
  └─ DoctorBookingCard.tsx                       (এখন placeholder, পরের ধাপে আসল booking)
src/app/(commonLayout)/consultation/doctor/[id]/page.tsx
```

> সব component **server component** (কোনো `"use client"` নেই) — শুধু data দেখায়, click/state নেই, তাই browser-এ এদের JS যায় না। booking card পরে client হবে।

---

## ধাপ ১: `src/types/review.types.ts` (নতুন)

```ts
// backend reviewPublicInclude: patient-এর শুধু নাম আর ছবি (email/phone কখনো না)
export interface IReview {
	id: string;
	rating: number;
	comment: string | null;
	createdAt: string;
	patient: {
		name: string;
		profilePhoto: string | null;
	};
}
```

- public page-এ patient-এর private তথ্য (email, phone) দেখানো যাবে না — backend-ই শুধু `name` আর `profilePhoto` পাঠায়, type-ও তাই।
- `createdAt` JSON-এ string হয়ে আসে (Date object না)।

---

## ধাপ ২: `src/services/doctor.services.ts` — file-এর শেষে যোগ করো

```ts
// public route: login ছাড়াও চলে। না পেলে backend 404 দেয়।
export const getDoctorById = async (id: string) => httpClient.get<IDoctor>(`/doctors/${encodeURIComponent(id)}`);
```

- `encodeURIComponent` → id-তে `/` বা `?` থাকলেও URL ভাঙে না (অন্য route-এ চলে যায় না)।

---

## ধাপ ৩: `src/services/review.services.ts` (নতুন)

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { IReview } from "@/types/review.types";

// public route: ?doctorId=...&limit=5&sortBy=createdAt&sortOrder=desc
export const getReviews = async (queryString: string) =>
	httpClient.get<IReview[]>(queryString ? `/reviews?${queryString}` : "/reviews");
```

- doctor service-এর মতোই pattern: query string বাইরে থেকে বানিয়ে পাঠানো।
- response-এ `data` (review list) আর `meta.total` (মোট কতগুলো) দুটোই আসে।

---

## ধাপ ৪: `src/components/shared/StarRating.tsx` (নতুন)

```tsx
import { cn } from "@/lib/utils";
import { Star } from "lucide-react";

interface StarRatingProps {
	value: number; // 0–5
	className?: string;
}

// screen reader পুরোটা একবারে পড়ে ("4.5 out of 5"), ৫টা আলাদা icon না
export default function StarRating({ value, className }: StarRatingProps) {
	const filled = Math.round(value);

	return (
		<span role="img" aria-label={`${value.toFixed(1)} out of 5`} className={cn("inline-flex gap-0.5", className)}>
			{[1, 2, 3, 4, 5].map((star) => (
				<Star
					key={star}
					aria-hidden
					className={cn(
						"size-4",
						star <= filled ? "fill-amber-400 text-amber-400" : "fill-muted text-muted-foreground/30",
					)}
				/>
			))}
		</span>
	);
}
```

- `Math.round` → 4.3 হলে ৪টা, 4.6 হলে ৫টা তারা ভরা।
- `role="img"` + `aria-label` → পুরো তারার সারি একটা ছবি হিসেবে পড়া হয়; প্রতিটা icon `aria-hidden`।
- `shared/`-এ রাখা হয়েছে কারণ profile আর review দুই জায়গাতেই লাগে (পরে doctor card-এও লাগতে পারে)।

---

## ধাপ ৫: `Consultation/DoctorDetails/DoctorProfile.tsx`

```tsx
import { GENDER_LABELS } from "@/components/modules/Admin/DoctorsManagement/doctorsTable.config";
import StarRating from "@/components/shared/StarRating";
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Badge } from "@/components/ui/badge";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { formatCurrency, getInitials } from "@/lib/formatters";
import type { IDoctor } from "@/types/doctor.types";
import { BadgeCheck, Briefcase, Building2, GraduationCap, MapPin, User, type LucideIcon } from "lucide-react";
import type { ReactNode } from "react";

// server component: interactivity নেই, তাই "use client" লাগে না (JS bundle-এ যায় না)

const Stat = ({ label, value }: { label: string; value: ReactNode }) => (
	<div className="rounded-xl bg-muted/50 px-4 py-3 text-center">
		<p className="text-lg font-semibold tabular-nums">{value}</p>
		<p className="text-xs text-muted-foreground">{label}</p>
	</div>
);

const InfoItem = ({ icon: Icon, label, children }: { icon: LucideIcon; label: string; children: ReactNode }) => (
	<div className="flex gap-3">
		<Icon className="mt-0.5 size-4 shrink-0 text-muted-foreground" aria-hidden />
		<div className="min-w-0">
			<dt className="text-xs text-muted-foreground">{label}</dt>
			<dd className="text-sm font-medium break-words">{children}</dd>
		</div>
	</div>
);

export default function DoctorProfile({ doctor }: { doctor: IDoctor }) {
	const hasRating = doctor.averageRating > 0;

	return (
		<>
			{/* ---------- Header ---------- */}
			<Card className="relative overflow-hidden">
				<div className="pointer-events-none absolute inset-x-0 top-0 h-24 bg-linear-to-r from-cyan-50 via-sky-50 to-blue-50 dark:from-cyan-950/40 dark:via-sky-950/30 dark:to-blue-950/40" />

				<CardContent className="relative space-y-6 pt-6">
					<div className="flex flex-col gap-5 sm:flex-row sm:items-end">
						<Avatar className="size-24 shadow-sm ring-4 ring-background">
							{doctor.profilePhoto && <AvatarImage src={doctor.profilePhoto} alt="" />}
							<AvatarFallback className="bg-primary/10 text-2xl font-semibold text-primary">
								{getInitials(doctor.name)}
							</AvatarFallback>
						</Avatar>

						<div className="min-w-0 space-y-1">
							<h1 className="text-2xl font-semibold tracking-tight sm:text-3xl">{doctor.name}</h1>
							<p className="text-muted-foreground">
								{doctor.designation} · {doctor.currentWorkingPlace}
							</p>
							<div className="flex items-center gap-2 text-sm">
								{hasRating ? (
									<>
										<StarRating value={doctor.averageRating} />
										<span className="tabular-nums text-muted-foreground">{doctor.averageRating.toFixed(1)}</span>
									</>
								) : (
									<span className="text-muted-foreground">No ratings yet</span>
								)}
							</div>
						</div>
					</div>

					{doctor.specialties.length > 0 && (
						<ul className="flex flex-wrap gap-2" aria-label="Specialties">
							{doctor.specialties.map(({ specialty }) => (
								<li key={specialty.id}>
									<Badge variant="secondary">{specialty.title}</Badge>
								</li>
							))}
						</ul>
					)}

					<div className="grid grid-cols-3 gap-3">
						<Stat label="Experience" value={`${doctor.experience} yrs`} />
						<Stat label="Fee" value={formatCurrency(doctor.appointmentFee)} />
						<Stat label="Rating" value={hasRating ? doctor.averageRating.toFixed(1) : "—"} />
					</div>
				</CardContent>
			</Card>

			{/* ---------- About ---------- */}
			<Card>
				<CardHeader>
					<CardTitle>About</CardTitle>
				</CardHeader>
				<CardContent>
					<dl className="grid gap-5 sm:grid-cols-2">
						<InfoItem icon={GraduationCap} label="Qualification">
							{doctor.qualification}
						</InfoItem>
						<InfoItem icon={Briefcase} label="Experience">
							{doctor.experience} years
						</InfoItem>
						<InfoItem icon={Building2} label="Current workplace">
							{doctor.currentWorkingPlace}
						</InfoItem>
						<InfoItem icon={BadgeCheck} label="Registration no.">
							{doctor.registrationNumber}
						</InfoItem>
						<InfoItem icon={User} label="Gender">
							{GENDER_LABELS[doctor.gender] ?? doctor.gender}
						</InfoItem>
						{doctor.address && (
							<InfoItem icon={MapPin} label="Chamber address">
								{doctor.address}
							</InfoItem>
						)}
					</dl>
				</CardContent>
			</Card>
		</>
	);
}
```

### ভেঙে বোঝা

- **`Stat` আর `InfoItem`** — একই design বারবার লেখা না লাগে তাই ছোট helper component (এই file-এর ভেতরেই, কারণ বাইরে লাগে না)।
- **`icon: Icon`** — prop-এর নাম বদলে বড় হাতের `Icon` করা, কারণ JSX-এ component-এর নাম বড় হাতের অক্ষরে শুরু হতে হয় (`<Icon />`)।
- **`hasRating`** — কোনো review না থাকলে `averageRating = 0`। তখন ০টা তারা না দেখিয়ে "No ratings yet" আর stat-এ "—"।
- **`<dl>` / `<dt>` / `<dd>`** — label-value জোড়ার জন্য সঠিক HTML (description list)।
- **`AvatarImage alt=""`** — নাম ঠিক পাশেই `<h1>`-এ আছে, তাই ছবির আলাদা বর্ণনা দরকার নেই (screen reader নাম দুবার পড়ে না)।
- **`GENDER_LABELS[...] ?? doctor.gender`** — `MALE` → "Male"; label না পেলে আসল মানটাই।
- **`min-w-0` + `break-words`** — লম্বা নাম/ঠিকানায় layout ভেঙে বাইরে যায় না।
- **`address`** optional — থাকলেই দেখায়।

---

## ধাপ ৬: `Consultation/DoctorDetails/DoctorReviews.tsx`

```tsx
import StarRating from "@/components/shared/StarRating";
import DateCell from "@/components/shared/table/cells/DateCell";
import { Avatar, AvatarFallback, AvatarImage } from "@/components/ui/avatar";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { getInitials } from "@/lib/formatters";
import type { IReview } from "@/types/review.types";

interface DoctorReviewsProps {
	reviews: IReview[] | null; // null = load হয়নি (page ভাঙে না, শুধু এই অংশে message)
	total: number;
}

export default function DoctorReviews({ reviews, total }: DoctorReviewsProps) {
	return (
		<Card>
			<CardHeader>
				<CardTitle>Patient reviews</CardTitle>
				{total > 0 && (
					<CardDescription>
						{total} {total === 1 ? "review" : "reviews"}
						{reviews && total > reviews.length && ` · showing latest ${reviews.length}`}
					</CardDescription>
				)}
			</CardHeader>

			<CardContent>
				{reviews === null ? (
					<p className="text-sm text-muted-foreground">Reviews couldn&apos;t be loaded right now.</p>
				) : reviews.length === 0 ? (
					<p className="text-sm text-muted-foreground">No reviews yet.</p>
				) : (
					<ul className="divide-y">
						{reviews.map((review) => (
							<li key={review.id} className="flex gap-3 py-4 first:pt-0 last:pb-0">
								<Avatar className="size-9">
									{review.patient.profilePhoto && <AvatarImage src={review.patient.profilePhoto} alt="" />}
									<AvatarFallback className="text-xs">{getInitials(review.patient.name)}</AvatarFallback>
								</Avatar>

								<div className="min-w-0 flex-1 space-y-1">
									<div className="flex flex-wrap items-center justify-between gap-2">
										<p className="text-sm font-medium">{review.patient.name}</p>
										<span className="text-xs text-muted-foreground">
											<DateCell value={review.createdAt} />
										</span>
									</div>
									<StarRating value={review.rating} />
									{review.comment && <p className="text-sm text-muted-foreground">{review.comment}</p>}
								</div>
							</li>
						))}
					</ul>
				)}
			</CardContent>
		</Card>
	);
}
```

### ভেঙে বোঝা

**`reviews`-এর তিন অবস্থা:**

| মান | মানে | দেখায় |
|---|---|---|
| `null` | review API fail করেছে | "Reviews couldn't be loaded right now." |
| `[]` | কোনো review নেই | "No reviews yet." |
| `[...]` | আছে | list |

- `null` আর `[]` আলাদা রাখা হয়েছে যাতে "review নেই" আর "load হয়নি" গুলিয়ে না যায়।
- **"12 reviews · showing latest 5"** — মোট সংখ্যা (`meta.total`) list-এর চেয়ে বেশি হলে user বুঝবে সব দেখানো হচ্ছে না।
- **`DateCell` reuse** — table-এর cell, কিন্তু timezone fix করা (`Asia/Dhaka`) তারিখ format এখানেও কাজে লাগে।
- `divide-y` + `first:pt-0 last:pb-0` → review-গুলোর মাঝে দাগ, কিন্তু প্রথম আর শেষে বাড়তি ফাঁকা নেই।

---

## ধাপ ৭: `Consultation/DoctorDetails/DoctorBookingCard.tsx` (এখন placeholder)

```tsx
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/components/ui/card";
import { Separator } from "@/components/ui/separator";
import { formatCurrency } from "@/lib/formatters";
import { SCHEDULE_INTERVAL_MINUTES } from "@/lib/scheduleTime";
import type { IDoctor } from "@/types/doctor.types";
import { CalendarClock } from "lucide-react";

// TODO (পরের ধাপ): এটা client component হবে:
// দিন বাছাই → doctor-এর খালি slot → patient হলে book, না হলে login-এ পাঠানো
export default function DoctorBookingCard({ doctor }: { doctor: IDoctor }) {
	return (
		<Card>
			<CardHeader>
				<CardTitle>Book an appointment</CardTitle>
				<CardDescription>Pick a time that works for you with {doctor.name}.</CardDescription>
			</CardHeader>

			<CardContent className="space-y-4">
				<dl className="space-y-2 text-sm">
					<div className="flex items-center justify-between">
						<dt className="text-muted-foreground">Consultation fee</dt>
						<dd className="text-lg font-semibold tabular-nums">{formatCurrency(doctor.appointmentFee)}</dd>
					</div>
					<div className="flex items-center justify-between">
						<dt className="text-muted-foreground">Duration</dt>
						<dd className="font-medium">{SCHEDULE_INTERVAL_MINUTES} minutes</dd>
					</div>
				</dl>

				<Separator />

				<div className="flex flex-col items-center gap-2 rounded-lg border border-dashed p-6 text-center">
					<CalendarClock className="size-6 text-muted-foreground" aria-hidden />
					<p className="text-sm text-muted-foreground">Available time slots will appear here.</p>
				</div>
			</CardContent>

			<CardFooter>
				<Button className="w-full" disabled>
					Book appointment
				</Button>
			</CardFooter>
		</Card>
	);
}
```

- `SCHEDULE_INTERVAL_MINUTES` (= 30) — admin-এর slot যে মাপে বানানো (note 21), সেখান থেকেই নেওয়া; সংখ্যা hard-code করা হয়নি।
- button এখন `disabled` — slot বাছাই না হওয়া পর্যন্ত book করা যাবে না।

---

## ধাপ ৮: `src/app/(commonLayout)/consultation/doctor/[id]/page.tsx` (পুরোটা বদলাও)

```tsx
import DoctorBookingCard from "@/components/modules/Consultation/DoctorDetails/DoctorBookingCard";
import DoctorProfile from "@/components/modules/Consultation/DoctorDetails/DoctorProfile";
import DoctorReviews from "@/components/modules/Consultation/DoctorDetails/DoctorReviews";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { Button } from "@/components/ui/button";
import { getDoctorById } from "@/services/doctor.services";
import { getReviews } from "@/services/review.services";
import axios from "axios";
import { ArrowLeft } from "lucide-react";
import Link from "next/link";
import { notFound } from "next/navigation";
import { Suspense } from "react";
import * as z from "zod";

const REVIEW_PREVIEW_LIMIT = "5";

// ভুল id বা মুছে ফেলা doctor → 404 page (error box না)
const getDoctorOrNotFound = async (id: string) => {
	// UUID না হলে backend-এ request পাঠানোরই দরকার নেই
	if (!z.uuid().safeParse(id).success) notFound();

	try {
		const response = await getDoctorById(id);
		return response.data;
	} catch (error) {
		if (axios.isAxiosError(error) && error.response?.status === 404) notFound();
		throw error; // বাকি error (500, network) → error.tsx
	}
};

async function DoctorDetails({ params }: { params: Promise<{ id: string }> }) {
	const { id } = await params;

	const reviewsQuery = new URLSearchParams({
		doctorId: id,
		limit: REVIEW_PREVIEW_LIMIT,
		sortBy: "createdAt",
		sortOrder: "desc",
	}).toString();

	// দুটো একসাথে। review না এলে পুরো page ভাঙে না → null।
	const [doctor, reviewsResponse] = await Promise.all([
		getDoctorOrNotFound(id),
		getReviews(reviewsQuery).catch(() => null),
	]);

	return (
		// DOM ক্রম: profile → booking → reviews। mobile-এ booking উপরের দিকেই থাকে।
		// desktop-এ grid দিয়ে booking ডান column-এ, পুরো উচ্চতা জুড়ে, sticky।
		<div className="grid gap-6 lg:grid-cols-[minmax(0,1fr)_22rem]">
			<div className="space-y-6 lg:col-start-1">
				<DoctorProfile doctor={doctor} />
			</div>

			<aside className="lg:col-start-2 lg:row-span-2 lg:row-start-1" aria-label="Book an appointment">
				<div className="lg:sticky lg:top-20">
					<DoctorBookingCard doctor={doctor} />
				</div>
			</aside>

			<div className="lg:col-start-1">
				<DoctorReviews reviews={reviewsResponse?.data ?? null} total={reviewsResponse?.meta?.total ?? 0} />
			</div>
		</div>
	);
}

export default function ConsultationDoctorPage({ params }: PageProps<"/consultation/doctor/[id]">) {
	return (
		<section className="mx-auto w-full max-w-7xl space-y-6 px-4 py-8">
			<Button asChild variant="ghost" size="sm" className="-ml-2">
				<Link href="/consultation">
					<ArrowLeft />
					All doctors
				</Link>
			</Button>

			{/* cacheComponents: cookie/params-এর মতো request data Suspense-এর ভেতরে পড়তে হয় */}
			<Suspense fallback={<LoadingSpinner text="Loading doctor..." />}>
				<DoctorDetails params={params} />
			</Suspense>
		</section>
	);
}
```

### ভেঙে বোঝা

**১. `getDoctorOrNotFound` — error-এর ধরন অনুযায়ী আলাদা page**

| অবস্থা | কী হয় |
|---|---|
| id UUID না (যেমন `/doctor/abc`) | backend-এ না গিয়েই `notFound()` |
| backend 404 (doctor নেই / মুছে ফেলা) | `notFound()` → `src/app/not-found.tsx` |
| 500 / network error | `throw` → `src/app/error.tsx` |

- `z.uuid()` — Zod v4-এর top-level format (`z.string().uuid()`-এর নতুন রূপ)।
- `notFound()` নিজেই throw করে, তাই এর পরের code চলে না (`return` লাগে না)।
- `axios.isAxiosError` কাজ করে কারণ `httpClient` error-টা বদলায় না, শুধু log করে আবার `throw` করে।

**২. doctor আর reviews একসাথে (`Promise.all`)**
- একটার জন্য আরেকটা অপেক্ষা করে না → page তাড়াতাড়ি আসে।
- `getReviews(...).catch(() => null)` → review fail হলে শুধু review অংশে message, পুরো page error-এ যায় না। doctor fail হলে কিন্তু পুরো page (doctor ছাড়া page-এর মানে নেই)।
- review query: `doctorId`, সাম্প্রতিক ৫টা (`sortBy=createdAt&sortOrder=desc&limit=5`) — `URLSearchParams` দিয়ে বানানো, তাই encode নিজে থেকেই হয়।

**৩. Layout — DOM ক্রম বনাম চোখে দেখার ক্রম**

| Screen | ক্রম |
|---|---|
| Mobile (১ column) | Profile → Booking → Reviews (DOM-এর ক্রমেই) |
| Desktop (`lg`, ২ column) | বামে Profile + Reviews, ডানে Booking (sticky) |

- `lg:grid-cols-[minmax(0,1fr)_22rem]` → বাম column বাকি জায়গা, ডান column ২২rem। `minmax(0, ...)` না দিলে লম্বা লেখায় বাম column ফুলে যেতে পারে।
- booking: `lg:col-start-2 lg:row-start-1 lg:row-span-2` → ডান column-এ, উপরের সারি থেকে দুই সারি জুড়ে।
- `lg:sticky lg:top-20` → scroll করলেও booking card চোখের সামনে থাকে (`top-20` = navbar-এর নিচে)।
- DOM-এ booking মাঝখানে রাখায় mobile-এ user reviews পর্যন্ত scroll না করেই book করতে পারে।
- `<aside aria-label>` → screen reader-এ আলাদা অংশ হিসেবে চেনা যায়।

**৪. `Suspense` কেন**
- `next.config.ts`-এ `cacheComponents: true` — `params`, cookie-র মতো request-এর data পড়া অংশ `Suspense`-এর ভেতরে রাখতে হয়।
- তাই "All doctors" back button সাথে সাথে দেখায়, আর doctor-এর অংশে spinner, তারপর data।
- `params` Promise হিসেবেই নিচে পাঠানো, `await` হয় `DoctorDetails`-এর ভেতরে।

**৫. `PageProps<"/consultation/doctor/[id]">`** — Next.js-এর auto-generated type, `params`-এর `{ id: string }` নিজে থেকেই আসে।

---

## 🧪 Test (browser-এ)

| কী করবে | ফল |
|---|---|
| `/consultation` থেকে card-এ ক্লিক | `/consultation/doctor/<uuid>` → profile, reviews, booking card |
| backend call | `GET /api/v1/doctors/:id` আর `GET /api/v1/reviews?doctorId=..&limit=5&sortBy=createdAt&sortOrder=desc` একসাথে |
| `/consultation/doctor/abc` | 404 page (backend-এ request যায় না) |
| ঠিক format-এর কিন্তু নেই এমন UUID | 404 page |
| backend বন্ধ করে খোলো | `error.tsx` |
| review নেই এমন doctor | "No ratings yet", "No reviews yet." |
| ৫টার বেশি review | "12 reviews · showing latest 5" |
| desktop-এ scroll | booking card ডানে আটকে থাকে |
| mobile width | profile → booking → reviews |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Page title-এ doctor-এর নাম (SEO + browser tab)** — এখন সব doctor page-এর title একই। `generateMetadata` দিয়ে নাম দাও। `React.cache` দিয়ে wrap করলে page আর metadata একই request share করে (দুবার API call হয় না):
   ```tsx
   import type { Metadata } from "next";
   import { cache } from "react";

   const getDoctorCached = cache(getDoctorOrNotFound);

   export async function generateMetadata({ params }: PageProps<"/consultation/doctor/[id]">): Promise<Metadata> {
   	const { id } = await params;
   	const doctor = await getDoctorCached(id);
   	return { title: `${doctor.name} · ${doctor.designation}` };
   }
   // DoctorDetails-এ getDoctorOrNotFound(id) → getDoctorCached(id)
   ```
   > `cacheComponents`-এর সাথে metadata-য় request data পড়লে Next.js কিছু অতিরিক্ত নিয়ম চাইতে পারে — test করা হয়নি, build-এ warning আসে কিনা দেখে নিও।

2. **"All reviews" link** — `total > 5` হলে বাকি review দেখার উপায় নেই। পরে আলাদা page বা "Load more" (client component + `useInfiniteQuery`) যোগ করা যায়।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`Suspense`-এর ভেতরে `notFound()` → status code সম্ভবত 200** — page-এর বাইরের অংশ (back button) আগেই stream হয়ে যায়, তাই browser-এ 404 UI দেখালেও HTTP status 404 না-ও হতে পারে (soft 404)। User-এর জন্য সমস্যা না, শুধু SEO-তে প্রভাব। DevTools → Network-এ status দেখে নিও।

4. **`"use server"` file-এর function = public endpoint** — `review.services.ts`-এর `getReviews` client থেকেও server action হিসেবে call করা যায়। এখানে সমস্যা নেই কারণ backend route-ও public; কিন্তু login লাগে এমন data-র service-এ backend-এর auth check-ই আসল নিরাপত্তা, সেটা মনে রেখো।

5. **Review-এর তারিখ "3 days ago" রূপে** — `Intl.RelativeTimeFormat` দিয়ে করা যায়; তবে server আর browser-এর সময়ের পার্থক্যে hydration mismatch এড়াতে server component-এই রাখা ভালো (এখন যেমন আছে)।

---

## 🔜 পরে যা যোগ হবে

- [ ] `DoctorBookingCard` → client component: দিন বাছাই → doctor-এর খালি slot দেখানো
- [ ] patient হলে book, login না থাকলে login page-এ পাঠানো (ফিরে এসে এই page-এ)
- [ ] সব review দেখার page / load more
- [ ] appointment শেষে patient-এর review দেওয়া
