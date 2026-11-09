# 25 - Doctor Booking (খালি Slot বাছাই + Pay Now / Pay Later)

> Doctor details page-এর (note 24) placeholder booking card এখন আসল: আগামী ১৪ দিনের খালি slot → দিন বাছাই → সময় বাছাই → **Pay now** (Stripe) বা **Pay later**।
> Login না থাকলে "Log in to book" (login-এর পর এই page-এ, একই দিনে ফেরত); patient ছাড়া অন্য role হলে book করা যায় না।
> Backend অংশ (public available route + Stripe cancel_url): server note 42।

```
server page: আজ (ঢাকা) + session user + slot prefetch  →  HydrationBoundary  →  DoctorBookingCard (client)
   দিন chip → সময় chip → Pay now  → bookAppointmentAction → Stripe checkout (window.location)
                        → Pay later → bookAppointmentAction → /dashboard/my-appointments
   409 (অন্য কেউ এইমাত্র book করেছে) → selection মুছে slot তালিকা আবার আনা
```

## ফাইলের তালিকা

```
src/lib/scheduleTime.ts                       ← ৩টা জিনিস যোগ
src/types/doctorSchedule.types.ts             ← IAvailableSlot যোগ
src/types/appointment.types.ts                (নতুন)
src/zod/appointment.validation.ts             (নতুন)
src/services/doctorSchedule.services.ts       ← ১টা function যোগ
src/services/appointment.services.ts          (নতুন)
src/actions/appointment.actions.ts            (নতুন)
src/queries/doctorSchedule.queries.ts         ← ১টা queryOptions যোগ
Consultation/DoctorDetails/DoctorBookingCard.tsx   (পুরোটা বদলাও)
consultation/doctor/[id]/page.tsx                  (পুরোটা বদলাও)
```

---

## ধাপ ১: `src/lib/scheduleTime.ts` — শেষে যোগ করো

```ts
// consultation page-এ রোগী কত দিন আগে পর্যন্ত slot দেখবে
export const BOOKING_WINDOW_DAYS = 14;

const dayPartsFormatter = new Intl.DateTimeFormat("en-GB", {
	weekday: "short",
	day: "numeric",
	month: "short",
	timeZone: CLINIC_TIME_ZONE,
});

// "2026-11-09" → { weekday: "Mon", day: "9", month: "Nov" } (দিনের chip-এর জন্য)
// দুপুর 12টা ধরি: কোনো timezone-এই তারিখ সরে যায় না
export const formatClinicDayParts = (date: string) => {
	const parts = Object.fromEntries(
		dayPartsFormatter
			.formatToParts(new Date(`${date}T12:00:00${CLINIC_UTC_OFFSET}`))
			.map(({ type, value: part }) => [type, part]),
	);

	return { weekday: parts.weekday ?? "", day: parts.day ?? "", month: parts.month ?? "" };
};
```

- `formatToParts` → `[{ type: "weekday", value: "Mon" }, { type: "literal", value: " " }, ...]`; `Object.fromEntries` দিয়ে `{ weekday, day, month, literal }` object বানানো, তারপর শুধু দরকারি তিনটা।
- **দুপুর ১২টা কেন:** রাত ১২টা ধরলে অন্য timezone-এ আগের দিন হয়ে যেতে পারে; দুপুরে ±১২ ঘণ্টার মধ্যে তারিখ বদলায় না।
- `CLINIC_UTC_OFFSET` (`"+06:00"`) এই file-এই আগে থেকে আছে।
- ১৪ দিন backend-এর ৩১ দিনের সীমার মধ্যে (server note 42)।

---

## ধাপ ২: `src/types/doctorSchedule.types.ts` — শেষে যোগ করো

```ts
// public available route: রোগীর শুধু সময়টুকু লাগে
export interface IAvailableSlot {
	id: string; // scheduleId
	startDateTime: string;
	endDateTime: string;
}
```

---

## ধাপ ৩: `src/types/appointment.types.ts` (নতুন)

```ts
export type PaymentMode = "pay-now" | "pay-later";

export interface IBookAppointmentPayload {
	doctorId: string;
	scheduleId: string;
}

// backend: pay-now → { appointment, payment, paymentUrl }, pay-later → { appointment, payment }
export interface IBookAppointmentResult {
	appointment: {
		id: string;
		status: string;
		paymentStatus: string;
	};
	paymentUrl?: string | null; // Stripe session.url null হতে পারে
}
```

- type-এ শুধু যা frontend ব্যবহার করে; `payment` object লাগে না তাই রাখা হয়নি।
- `paymentUrl` থাকলে pay-now, না থাকলে pay-later — card এটা দেখেই ঠিক করে কোথায় যাবে।

---

## ধাপ ৪: `src/zod/appointment.validation.ts` (নতুন)

```ts
import * as z from "zod";

// backend bookAppointmentValidationSchema-র সাথে মিল
export const bookAppointmentPayloadSchema = z.object({
	doctorId: z.uuid("Invalid doctor"),
	scheduleId: z.uuid("Select a time slot"),
});

export const paymentModeSchema = z.enum(["pay-now", "pay-later"], "Invalid payment option");
```

- Zod v4-এ error message সরাসরি string হিসেবে দেওয়া যায় (`z.uuid("...")`, `z.enum([...], "...")`)।

---

## ধাপ ৫: `src/services/doctorSchedule.services.ts` — শেষে যোগ করো

```ts
// public route: login ছাড়াও চলে
export const getAvailableDoctorSlots = async (doctorId: string, startDate: string, endDate: string) =>
	httpClient.get<IAvailableSlot[]>(`/doctor-schedules/doctor/${encodeURIComponent(doctorId)}/available`, {
		params: { startDate, endDate },
	});
```

উপরের import-এ `IAvailableSlot` যোগ করো:

```ts
import type { IAvailableSlot, ICreateDoctorSchedulePayload, IDoctorSchedule } from "@/types/doctorSchedule.types";
```

- `params` → axios নিজেই `?startDate=...&endDate=...` বানায় আর encode করে।

---

## ধাপ ৬: `src/services/appointment.services.ts` (নতুন)

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { IBookAppointmentPayload, IBookAppointmentResult } from "@/types/appointment.types";

// দুটোই checkAuth(PATIENT)। slot booked থাকলে backend 409 দেয়।

// appointment + Stripe checkout URL
export const bookAppointment = async (payload: IBookAppointmentPayload) =>
	httpClient.post<IBookAppointmentResult>("/appointments/book-appointment", payload);

// appointment UNPAID থাকে, ৩০ মিনিটের মধ্যে pay না করলে cron cancel করে
export const bookAppointmentWithPayLater = async (payload: IBookAppointmentPayload) =>
	httpClient.post<IBookAppointmentResult>("/appointments/book-appointment-with-pay-later", payload);
```

- login cookie `httpClient` নিজেই পাঠায় (server-এ চলে), তাই backend বোঝে কে book করছে।

---

## ধাপ ৭: `src/actions/appointment.actions.ts` (নতুন)

```ts
"use server";

import { getActionErrorMessage } from "@/lib/getActionErrorMessage";
import { bookAppointment, bookAppointmentWithPayLater } from "@/services/appointment.services";
import type { ActionResult } from "@/types/api.types";
import type { IBookAppointmentResult } from "@/types/appointment.types";
import { bookAppointmentPayloadSchema, paymentModeSchema } from "@/zod/appointment.validation";

// public endpoint: input unknown। কে book করতে পারবে (শুধু PATIENT) সেটা backend checkAuth ঠিক করে।
export async function bookAppointmentAction(
	payload: unknown,
	paymentMode: unknown,
): Promise<ActionResult<IBookAppointmentResult>> {
	const parsedPayload = bookAppointmentPayloadSchema.safeParse(payload);
	if (!parsedPayload.success) {
		return { success: false, message: parsedPayload.error.issues[0]?.message ?? "Invalid input" };
	}

	const parsedMode = paymentModeSchema.safeParse(paymentMode);
	if (!parsedMode.success) {
		return { success: false, message: "Invalid payment option" };
	}

	try {
		return parsedMode.data === "pay-now"
			? await bookAppointment(parsedPayload.data)
			: await bookAppointmentWithPayLater(parsedPayload.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "bookAppointmentAction") };
	}
}
```

### ভেঙে বোঝা

- **parameter `unknown` কেন:** server action যে কেউ সরাসরি call করতে পারে (browser devtools থেকে), তাই TypeScript type-এ ভরসা না করে Zod দিয়ে আসল check।
- **দুই ধাপ validation:** payload (`doctorId`, `scheduleId`) আর payment mode আলাদা — ভুল হলে প্রথম error message ফেরত।
- **throw না, `{ success: false }` return:** action error object ফেরত দেয়; `useActionMutation` সেটা দেখে নিজে throw করে (note 18/19)।
- **একটা action, দুটো service:** mode অনুযায়ী `book-appointment` বা `book-appointment-with-pay-later`।
- `getActionErrorMessage` → backend-এর message (যেমন 409 "already booked") user-এর কাছে পৌঁছায়।

---

## ধাপ ৮: `src/queries/doctorSchedule.queries.ts` — শেষে যোগ করো

```ts
// book হলে invalidate → booked slot তালিকা থেকে সরে যায়
export const AVAILABLE_SLOTS_QUERY_KEY = ["available-slots"] as const;

export const availableSlotsQueryOptions = (doctorId: string, startDate: string, endDate: string) =>
	queryOptions({
		queryKey: [...AVAILABLE_SLOTS_QUERY_KEY, doctorId, startDate, endDate],
		queryFn: () => getAvailableDoctorSlots(doctorId, startDate, endDate),
		staleTime: 1000 * 30, // অন্য রোগী যেকোনো সময় book করতে পারে, তাই ছোট
	});
```

import লাইনটা এভাবে বদলাও:

```ts
import { getAvailableDoctorSlots, getMyDoctorSchedules } from "@/services/doctorSchedule.services";
```

- `staleTime` মাত্র ৩০ সেকেন্ড (doctor list-এ ১ ঘণ্টা ছিল) — slot-এর তথ্য দ্রুত পুরনো হয়।
- `["available-slots"]` invalidate করলে যেকোনো doctor/তারিখের slot query আবার আসে।

---

## ধাপ ৯: `Consultation/DoctorDetails/DoctorBookingCard.tsx` (পুরোটা বদলাও)

```tsx
"use client";

import { bookAppointmentAction } from "@/actions/appointment.actions";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardDescription, CardFooter, CardHeader, CardTitle } from "@/components/ui/card";
import { Separator } from "@/components/ui/separator";
import { useActionMutation } from "@/hooks/useActionMutation";
import { formatCurrency } from "@/lib/formatters";
import {
	addDaysToDateString,
	BOOKING_WINDOW_DAYS,
	formatClinicDayParts,
	formatClinicTime,
	SCHEDULE_INTERVAL_MINUTES,
	toClinicDateTimeParts,
} from "@/lib/scheduleTime";
import { cn } from "@/lib/utils";
import { AVAILABLE_SLOTS_QUERY_KEY, availableSlotsQueryOptions } from "@/queries/doctorSchedule.queries";
import type { PaymentMode } from "@/types/appointment.types";
import type { UserRole } from "@/types/auth.types";
import type { IDoctor } from "@/types/doctor.types";
import type { IAvailableSlot } from "@/types/doctorSchedule.types";
import { useQuery } from "@tanstack/react-query";
import { CalendarClock, CalendarX, CreditCard, LoaderCircle, LogIn } from "lucide-react";
import { usePathname, useRouter } from "next/navigation";
import { useMemo, useState } from "react";

const UNPAID_TIMEOUT_MINUTES = 30; // backend UNPAID_APPOINTMENT_TIMEOUT_MINUTES

interface DoctorBookingCardProps {
	doctor: Pick<IDoctor, "id" | "name" | "appointmentFee">;
	viewerRole: UserRole | null; // null = login করা নেই
	today: string; // server থেকে (ঢাকার তারিখ) → server আর browser-এ একই, hydration mismatch নেই
	initialDate?: string; // login থেকে ফিরলে ?date=...
}

export default function DoctorBookingCard({ doctor, viewerRole, today, initialDate }: DoctorBookingCardProps) {
	const router = useRouter();
	const pathname = usePathname();
	const endDate = addDaysToDateString(today, BOOKING_WINDOW_DAYS - 1);

	const [pickedDate, setPickedDate] = useState<string | null>(initialDate ?? null);
	const [selectedSlotId, setSelectedSlotId] = useState<string | null>(null);

	// server prefetch করেছে → প্রথম render-এই data থাকে
	const { data, isPending, isError, refetch } = useQuery(availableSlotsQueryOptions(doctor.id, today, endDate));

	// দিন অনুযায়ী ভাগ (backend আগেই সময় অনুযায়ী সাজিয়ে দেয়, তাই Map-এর ক্রমও ঠিক থাকে)
	const slotsByDate = useMemo(() => {
		const map = new Map<string, IAvailableSlot[]>();
		for (const slot of data?.data ?? []) {
			const { date } = toClinicDateTimeParts(slot.startDateTime);
			const list = map.get(date);
			if (list) list.push(slot);
			else map.set(date, [slot]);
		}
		return map;
	}, [data]);

	const availableDates = [...slotsByDate.keys()];

	// derived state: বাছা দিনে আর slot না থাকলে (অন্য কেউ book করেছে) প্রথম খালি দিনে চলে যায়।
	// effect দিয়ে state sync করার দরকার নেই।
	const activeDate = pickedDate && slotsByDate.has(pickedDate) ? pickedDate : (availableDates[0] ?? null);
	const daySlots = activeDate ? (slotsByDate.get(activeDate) ?? []) : [];
	const selectedSlot = daySlots.find((slot) => slot.id === selectedSlotId) ?? null;

	const {
		mutate,
		isPending: isBooking,
		variables,
	} = useActionMutation({
		action: ({ scheduleId, mode }: { scheduleId: string; mode: PaymentMode }) =>
			bookAppointmentAction({ doctorId: doctor.id, scheduleId }, mode),
		invalidateKey: AVAILABLE_SLOTS_QUERY_KEY,
		successMessage: (result) =>
			result.paymentUrl
				? "Slot reserved. Taking you to payment..."
				: `Appointment booked. Please pay within ${UNPAID_TIMEOUT_MINUTES} minutes.`,
		onSuccess: (result) => {
			// Stripe অন্য site, তাই router না, পুরো page navigation
			if (result.paymentUrl) {
				window.location.assign(result.paymentUrl);
				return;
			}
			router.push("/dashboard/my-appointments");
		},
	});

	const handleBook = (mode: PaymentMode) => {
		if (!selectedSlot) return;

		mutate(
			{ scheduleId: selectedSlot.id, mode },
			{
				// 409 (অন্য কেউ এইমাত্র book করেছে) → নতুন তালিকা আনো, selection মুছে দাও
				onError: () => {
					setSelectedSlotId(null);
					void refetch();
				},
			},
		);
	};

	const handleLogin = () => {
		// login-এর পর এই page, এই দিনেই ফেরত (authUtils.getPostLoginRedirect নিরাপদে check করে)
		const backTo = activeDate ? `${pathname}?date=${activeDate}` : pathname;
		router.push(`/login?redirect=${encodeURIComponent(backTo)}`);
	};

	const handlePickDate = (date: string) => {
		setPickedDate(date);
		setSelectedSlotId(null);
	};

	// ---------- slot অংশ ----------
	const renderSlots = () => {
		if (isPending) {
			return (
				<p className="flex items-center justify-center gap-2 py-6 text-sm text-muted-foreground">
					<LoaderCircle className="size-4 animate-spin" aria-hidden />
					Loading available slots...
				</p>
			);
		}

		if (isError) {
			return (
				<div className="space-y-2 py-4 text-center">
					<p className="text-sm text-destructive">Couldn&apos;t load available slots.</p>
					<Button variant="outline" size="sm" onClick={() => void refetch()}>
						Try again
					</Button>
				</div>
			);
		}

		if (availableDates.length === 0) {
			return (
				<div className="flex flex-col items-center gap-2 rounded-lg border border-dashed p-6 text-center">
					<CalendarX className="size-6 text-muted-foreground" aria-hidden />
					<p className="text-sm text-muted-foreground">
						No open slots in the next {BOOKING_WINDOW_DAYS} days. Please check back later.
					</p>
				</div>
			);
		}

		return (
			<div className="space-y-4">
				{/* দিন */}
				<div>
					<p className="mb-2 text-sm font-medium">Choose a day</p>
					<div role="group" aria-label="Available days" className="-mx-1 flex gap-2 overflow-x-auto px-1 pb-1">
						{availableDates.map((date) => {
							const { weekday, day, month } = formatClinicDayParts(date);
							const isActive = date === activeDate;

							return (
								<Button
									key={date}
									type="button"
									variant={isActive ? "default" : "outline"}
									aria-pressed={isActive}
									disabled={isBooking}
									onClick={() => handlePickDate(date)}
									className="h-auto w-16 shrink-0 flex-col gap-0 py-2"
								>
									<span className="text-xs font-normal opacity-80">{weekday}</span>
									<span className="text-lg leading-tight font-semibold tabular-nums">{day}</span>
									<span className="text-xs font-normal opacity-80">{month}</span>
								</Button>
							);
						})}
					</div>
				</div>

				{/* সময় */}
				<div>
					<p className="mb-2 text-sm font-medium">
						Choose a time <span className="font-normal text-muted-foreground">(Bangladesh time)</span>
					</p>
					<div role="group" aria-label="Available times" className="grid grid-cols-3 gap-2">
						{daySlots.map((slot) => {
							const isSelected = slot.id === selectedSlotId;

							return (
								<Button
									key={slot.id}
									type="button"
									size="sm"
									variant={isSelected ? "default" : "outline"}
									aria-pressed={isSelected}
									disabled={isBooking}
									onClick={() => setSelectedSlotId(isSelected ? null : slot.id)}
									className="tabular-nums"
								>
									{formatClinicTime(slot.startDateTime)}
								</Button>
							);
						})}
					</div>
				</div>
			</div>
		);
	};

	// ---------- নিচের button অংশ (কে দেখছে তার উপর নির্ভর করে) ----------
	const renderActions = () => {
		if (!viewerRole) {
			return (
				<Button className="w-full" onClick={handleLogin}>
					<LogIn />
					Log in to book
				</Button>
			);
		}

		if (viewerRole !== "PATIENT") {
			return (
				<p className="w-full rounded-md bg-muted px-3 py-2 text-center text-sm text-muted-foreground">
					Only patient accounts can book appointments.
				</p>
			);
		}

		const isPayNowPending = isBooking && variables?.mode === "pay-now";
		const isPayLaterPending = isBooking && variables?.mode === "pay-later";

		return (
			<div className="w-full space-y-2">
				<Button className="w-full" disabled={!selectedSlot || isBooking} onClick={() => handleBook("pay-now")}>
					{isPayNowPending ? <LoaderCircle className="animate-spin" /> : <CreditCard />}
					{isPayNowPending ? "Reserving..." : `Pay ${formatCurrency(doctor.appointmentFee)} & book`}
				</Button>
				<Button
					variant="outline"
					className="w-full"
					disabled={!selectedSlot || isBooking}
					onClick={() => handleBook("pay-later")}
				>
					{isPayLaterPending && <LoaderCircle className="animate-spin" />}
					{isPayLaterPending ? "Booking..." : "Book now, pay later"}
				</Button>
				<p className="text-center text-xs text-muted-foreground">
					Unpaid bookings are cancelled after {UNPAID_TIMEOUT_MINUTES} minutes.
				</p>
			</div>
		);
	};

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

				{renderSlots()}

				{/* বাছাইয়ের সারাংশ: কী book হচ্ছে স্পষ্ট দেখায় */}
				<div
					aria-live="polite"
					className={cn(
						"flex items-center gap-2 rounded-md px-3 py-2 text-sm",
						selectedSlot ? "bg-primary/10 text-foreground" : "bg-muted text-muted-foreground",
					)}
				>
					<CalendarClock className="size-4 shrink-0" aria-hidden />
					{selectedSlot && activeDate ? (
						<span>
							{(() => {
								const { weekday, day, month } = formatClinicDayParts(activeDate);
								return `${weekday}, ${day} ${month}`;
							})()}{" "}
							· {formatClinicTime(selectedSlot.startDateTime)} – {formatClinicTime(selectedSlot.endDateTime)}
						</span>
					) : (
						<span>No time selected</span>
					)}
				</div>
			</CardContent>

			<CardFooter>{renderActions()}</CardFooter>
		</Card>
	);
}
```

### ভেঙে বোঝা

**১. Props — server থেকে কী আসে**

| prop | কেন |
|---|---|
| `doctor` (`Pick<...>`) | শুধু `id`, `name`, `appointmentFee` — পুরো doctor object client-এ পাঠানো লাগে না |
| `viewerRole` | `null` = login নেই; role দেখে নিচের button ঠিক হয় |
| `today` | ঢাকার আজকের তারিখ server-এ ঠিক হয় → server আর browser-এ একই query key, hydration mismatch নেই |
| `initialDate` | login থেকে ফিরলে `?date=...` |

**২. Slot-গুলো দিন অনুযায়ী ভাগ (`slotsByDate`)**
- backend একটা লম্বা list দেয় (সময় অনুযায়ী সাজানো)। `toClinicDateTimeParts` দিয়ে প্রতিটা slot-এর **ঢাকার** তারিখ বের করে `Map<date, slot[]>`।
- `Map` ঢোকানোর ক্রম মনে রাখে → দিনগুলোও ক্রমে থাকে। `useMemo` → `data` না বদলালে আবার হিসাব হয় না।
- যে দিনে slot নেই সেই দিনের chip-ই দেখায় না।

**৩. Derived state (effect ছাড়া)**
- state-এ রাখা হয় শুধু user যা বেছেছে: `pickedDate`, `selectedSlotId`।
- `activeDate` = বাছা দিন যদি এখনো তালিকায় থাকে, নইলে প্রথম খালি দিন। কেউ শেষ slot book করে দিলে নিজে থেকেই অন্য দিনে চলে যায়।
- `selectedSlot` = বাছা id এখনো ওই দিনের তালিকায় আছে কিনা — না থাকলে `null`, তাই button disabled।
- `useEffect` দিয়ে state মেলাতে গেলে এক render ভুল দেখায় + বাড়তি render; render-এর সময় হিসাব করাই সহজ।

**৪. Booking mutation (`useActionMutation`)**
- একই mutation দুই mode-এ; `variables.mode` দেখে কোন button-এ spinner দেখাবে।
- সফল হলে আগে `["available-slots"]` invalidate (hook নিজে করে), তারপর:
  - `paymentUrl` আছে → `window.location.assign` (Stripe অন্য website, তাই Next router না)।
  - নেই (pay later) → `/dashboard/my-appointments`।
- `mutate(..., { onError })` → hook-এর toast-এর পাশাপাশি এখানে selection মুছে নতুন তালিকা আনা (409 = অন্য কেউ এইমাত্র book করেছে)।

**৫. Login-এ পাঠানো আর ফেরত আসা**
- `/login?redirect=/consultation/doctor/<id>?date=2026-11-12` — `encodeURIComponent` দিয়ে, কারণ ভেতরের `?` আর `=` বাইরের URL ভাঙাতো।
- login-এর পর `getPostLoginRedirect` (authUtils) দেখে নেয় এটা নিজের site-এর path কিনা (open redirect আটকায়)।

**৬. কে দেখছে অনুযায়ী button**

| viewer | footer |
|---|---|
| login নেই | "Log in to book" |
| DOCTOR / ADMIN | "Only patient accounts can book appointments." |
| PATIENT | "Pay ৳X & book" + "Book now, pay later" (slot না বাছলে disabled) |

> UI-তে লুকানো শুধু সুবিধার জন্য — আসল নিরাপত্তা backend-এর `checkAuth(PATIENT)`।

**৭. Accessibility**
- দিন আর সময়ের chip `role="group"` + `aria-label`, প্রতিটা button `aria-pressed` (toggle)।
- সারাংশ বাক্সে `aria-live="polite"` → সময় বাছলে screen reader বলে দেয় কী বাছা হলো।
- "(Bangladesh time)" লেখা → বিদেশ থেকে দেখলেও বোঝা যায় সময় কোন timezone-এর।

**৮. ছোট জিনিস**
- দিনের chip-এর সারি `overflow-x-auto` → ১৪ দিন হলেও পাশে scroll; `-mx-1 px-1` focus ring কাটা যায় না।
- সময় আবার ক্লিক করলে বাছাই উঠে যায় (`isSelected ? null : slot.id`)।
- booking চলার সময় সব chip `disabled` → মাঝপথে বাছাই বদলানো যায় না।

---

## ধাপ ১০: `consultation/doctor/[id]/page.tsx` (পুরোটা বদলাও)

```tsx
import DoctorBookingCard from "@/components/modules/Consultation/DoctorDetails/DoctorBookingCard";
import DoctorProfile from "@/components/modules/Consultation/DoctorDetails/DoctorProfile";
import DoctorReviews from "@/components/modules/Consultation/DoctorDetails/DoctorReviews";
import LoadingSpinner from "@/components/shared/LoadingSpinner";
import { Button } from "@/components/ui/button";
import { getQueryClient } from "@/lib/getQueryClient";
import { addDaysToDateString, BOOKING_WINDOW_DAYS, getClinicToday } from "@/lib/scheduleTime";
import { getSessionUser } from "@/lib/session";
import { availableSlotsQueryOptions } from "@/queries/doctorSchedule.queries";
import { getDoctorById } from "@/services/doctor.services";
import { getReviews } from "@/services/review.services";
import { dehydrate, HydrationBoundary, noop } from "@tanstack/react-query";
import axios from "axios";
import { ArrowLeft } from "lucide-react";
import Link from "next/link";
import { notFound } from "next/navigation";
import { connection } from "next/server";
import { Suspense } from "react";
import * as z from "zod";

const REVIEW_PREVIEW_LIMIT = "5";
const ISO_DATE_REGEX = /^\d{4}-\d{2}-\d{2}$/;

type SearchParams = Record<string, string | string[] | undefined>;

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

async function DoctorDetails({
	params,
	searchParams,
}: {
	params: Promise<{ id: string }>;
	searchParams: Promise<SearchParams>;
}) {
	const [{ id }, query] = await Promise.all([params, searchParams]);

	// connection(): Next-কে বলে "এই অংশ prerender করো না, প্রতিটা request-এ চালাও"।
	// params/searchParams await করাই যথেষ্ট না (prerender-এ ওদের fallback থাকতে পারে)।
	// এর পরে new Date() নিরাপদ: "আজ" সবসময় আসল আজকের তারিখ।
	await connection();

	// ঢাকার তারিখ server-এ ঠিক হয় → client-এ একই মান যায়, hydration mismatch নেই
	const today = getClinicToday();
	const endDate = addDaysToDateString(today, BOOKING_WINDOW_DAYS - 1);

	// login থেকে ফিরলে ?date=... দিয়ে আগের বাছা দিনটা খোলে
	const requestedDate = typeof query.date === "string" && ISO_DATE_REGEX.test(query.date) ? query.date : undefined;

	const reviewsQuery = new URLSearchParams({
		doctorId: id,
		limit: REVIEW_PREVIEW_LIMIT,
		sortBy: "createdAt",
		sortOrder: "desc",
	}).toString();

	const queryClient = getQueryClient();

	// সব একসাথে। review বা slot না এলে page ভাঙে না।
	const [doctor, reviewsResponse, viewer] = await Promise.all([
		getDoctorOrNotFound(id),
		getReviews(reviewsQuery).catch(() => null),
		getSessionUser(),
		queryClient.query(availableSlotsQueryOptions(id, today, endDate)).catch(noop),
	]);

	return (
		// DOM ক্রম: profile → booking → reviews (mobile-এ booking উপরের দিকে থাকে)।
		// desktop-এ grid দিয়ে booking ডান column-এ, পুরো উচ্চতা জুড়ে, sticky।
		<div className="grid gap-6 lg:grid-cols-[minmax(0,1fr)_22rem]">
			<div className="space-y-6 lg:col-start-1">
				<DoctorProfile doctor={doctor} />
			</div>

			<aside className="lg:col-start-2 lg:row-span-2 lg:row-start-1" aria-label="Book an appointment">
				<div className="lg:sticky lg:top-20">
					<HydrationBoundary state={dehydrate(queryClient)}>
						{/* client component-এ শুধু দরকারি field পাঠাই (পুরো doctor object না) */}
						<DoctorBookingCard
							doctor={{ id: doctor.id, name: doctor.name, appointmentFee: doctor.appointmentFee }}
							viewerRole={viewer?.role ?? null}
							today={today}
							initialDate={requestedDate}
						/>
					</HydrationBoundary>
				</div>
			</aside>

			<div className="lg:col-start-1">
				<DoctorReviews reviews={reviewsResponse?.data ?? null} total={reviewsResponse?.meta?.total ?? 0} />
			</div>
		</div>
	);
}

export default function ConsultationDoctorPage({ params, searchParams }: PageProps<"/consultation/doctor/[id]">) {
	return (
		<section className="mx-auto w-full max-w-7xl space-y-6 px-4 py-8">
			<Button asChild variant="ghost" size="sm" className="-ml-2">
				<Link href="/consultation">
					<ArrowLeft />
					All doctors
				</Link>
			</Button>

			{/* cacheComponents: request data (params, cookie, সময়) Suspense-এর ভেতরে পড়তে হয় */}
			<Suspense fallback={<LoadingSpinner text="Loading doctor..." />}>
				<DoctorDetails params={params} searchParams={searchParams} />
			</Suspense>
		</section>
	);
}
```

### আগের version (note 24) থেকে কী বদলালো

| নতুন | কেন |
|---|---|
| `searchParams` নেওয়া, `?date=` পড়া | login থেকে ফিরে আগের বাছা দিন খোলা; regex দিয়ে শুধু `YYYY-MM-DD` মানা |
| `await connection()` | এরপর `new Date()` (আজকের তারিখ) নিরাপদ — `cacheComponents`-এ prerender-এর সময়ের "আজ" আটকে থাকে না |
| `getClinicToday()` server-এ | ঢাকার আজ; client-কে prop হিসেবে → একই query key, hydration mismatch নেই |
| `getSessionUser()` | কে দেখছে (role) → booking card-এর button ঠিক করা (`cache()`-এ wrap করা, তাই একই request-এ বারবার call হলেও একবারই চলে) |
| `queryClient.query(availableSlots...)` + `HydrationBoundary` | slot আগেই server-এ আনা → card প্রথম render-এই slot দেখায়, loading নেই |
| `.catch(noop)` | slot না এলে page ভাঙে না; client-এর `useQuery` আবার চেষ্টা করে, না পারলে "Try again" |
| `doctor={{ id, name, appointmentFee }}` | client component-এ যা পাঠাও তা HTML-এ serialize হয় — ছোট রাখা ভালো |

- চারটা কাজ (`doctor`, `reviews`, `session`, `slots`) এক `Promise.all`-এ → একটার জন্য আরেকটা অপেক্ষা করে না। চতুর্থটার ফল লাগে না (cache-এ যায়), তাই destructure-এ তিনটা।

---

## 🧪 Test (browser-এ)

| কী করবে | ফল |
|---|---|
| login ছাড়া doctor page খোলো | slot দেখা যায়, নিচে "Log in to book" |
| একটা দিন বেছে "Log in to book" | `/login?redirect=...%3Fdate%3D2026-11-12` → login → একই page, একই দিন খোলা |
| DOCTOR/ADMIN হিসেবে খোলো | "Only patient accounts can book appointments." |
| PATIENT: সময় না বেছে | দুই button disabled, "No time selected" |
| সময় বাছো | সারাংশে "Wed, 12 Nov · 10:00 AM – 10:30 AM" |
| Pay now | toast "Slot reserved..." → Stripe checkout page |
| Stripe-এ cancel | `/dashboard/my-appointments?...` (server note 42-এর cancel_url) |
| Pay later | toast "Please pay within 30 minutes" → `/dashboard/my-appointments` |
| দুই browser-এ একই slot book | দ্বিতীয়টায় 409 toast, selection মুছে নতুন তালিকা (slot টা আর নেই) |
| slot নেই এমন doctor | "No open slots in the next 14 days." |
| backend-এর slot route বন্ধ | "Couldn't load available slots." + Try again |
| `?date=abc` দিয়ে খোলো | regex-এ বাদ → প্রথম খালি দিন খোলে |

---

## ✅ নতুন project-এ implement করার checklist

- [ ] backend-এ public "available slots" route (date range + সীমা)
- [ ] "আজ" server-এ হিসাব করে client-কে prop (timezone fixed)
- [ ] slot prefetch + `HydrationBoundary`; ছোট `staleTime`
- [ ] user-এর বাছাই state-এ, বাকি সব derived (effect ছাড়া)
- [ ] action-এ `unknown` input + Zod; role check backend-এ
- [ ] 409 হলে refetch + selection মুছে দেওয়া
- [ ] login redirect-এ ফেরত path + safe check

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Stripe redirect-এর আগে invalidate-এর অপেক্ষা বাদ** — `useActionMutation` সফল হলে আগে `await invalidateQueries` (slot আবার fetch) করে, তারপর `onSuccess`। Pay-now-এ user তো page ছেড়েই যাচ্ছে, তাই এই fetch অপ্রয়োজনীয় দেরি। hook-এ একটা option দেওয়া যায়:
   ```ts
   // useActionMutation-এ
   invalidateKey?: QueryKey;
   // ...
   if (invalidateKey) await queryClient.invalidateQueries({ queryKey: invalidateKey });
   ```
   তারপর card-এ pay-now হলে `onSuccess`-এ redirect আগে, invalidate পরে (বা না)। ছোট বিষয়, test করে দেখো দেরি চোখে পড়ে কিনা।

2. **শুধু 409-এ refetch** — এখন যেকোনো error-এ (network, 401) selection মুছে যায়। network error-এ user-কে আবার বাছতে হবে, যেটা বিরক্তিকর। action থেকে status code ফেরত দিলে শুধু conflict-এ মোছা যায়। (এখনকার আচরণও ভুল না — নিরাপদ দিকেই।)

3. **Session শেষ হয়ে গেলে (401)** — page খোলার সময় login ছিল, পরে cookie expire হলে book-এ backend 401 দেয়, শুধু toast দেখায়। toast-এর সাথে login-এ পাঠালে ভালো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **Pay-now-এ Stripe cancel করলে slot ৩০ মিনিট আটকে থাকে** — appointment আগেই তৈরি হয়, cron cancel না করা পর্যন্ত অন্যরা slot পায় না (server note 42-এ বিস্তারিত)। `my-appointments` page-এ "Pay now" বা "Cancel" button রাখলে user নিজেই ঠিক করতে পারে।

5. **`UNPAID_TIMEOUT_MINUTES` দুই জায়গায়** — backend আর frontend-এ আলাদা করে ৩০ লেখা। backend বদলালে এটাও বদলাতে হবে (comment-এ লেখা আছে, ভালো)।

6. **Page খোলা রেখে মধ্যরাত পার হলে** — `today` server-এ একবার ঠিক হয়, তাই পরের দিন সকাল পর্যন্ত tab খোলা থাকলে গতকাল থেকে range থাকে। backend অতীতের slot বাদ দেয় (server note 42), তাই ভুল slot দেখায় না — শুধু range একদিন কম। reload করলেই ঠিক।

---

## 🔜 পরে যা যোগ হবে

- [ ] `/dashboard/my-appointments` — নিজের appointment-এর তালিকা, UNPAID হলে "Pay now"
- [ ] `/dashboard/payment/payment-success` — Stripe থেকে ফিরে দেখানো
- [ ] cancel_url থেকে ফিরলে "Payment cancelled" message
- [ ] appointment শেষে review দেওয়া
