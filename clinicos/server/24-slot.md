# Doctor Slot module

## ১. Module-এর file গুলো

```text
src/app/module/slot/
├─ slot.route.ts
├─ slot.controller.ts
├─ slot.service.ts        (open, manage list, public list, block, unblock, delete)
├─ slot.validation.ts     (Zod: HH:mm, dates 1–14, breaks, slotMinutes)
├─ slot.interface.ts
├─ slot.utils.ts          (buildSlots: pure function, এখানে DB নেই)
└─ slot.utils.test.ts
src/app/config/slot.config.ts   (hospital-এর নিয়ম: allowed minutes, max days, default breaks)
```

## ২. Package install

1. `apps/api`-তে গিয়ে package install করো:

```bash
cd apps/api
pnpm add @date-fns/tz date-fns
pnpm add -D vitest
```

2. `package.json`-এর `scripts`-এ যোগ করো:

```json
"test": "vitest run",
"test:watch": "vitest"
```

> **নোট:** Code-এ `Array.prototype.toSorted()` ব্যবহার হয়েছে। এর জন্য Node 20+ আর `tsconfig.json`-এর `lib`/`target`-এ `ES2023` (বা তার পরের) লাগবে।

## ৩. Prisma

1. `prisma/schema/enums.prisma`-এর শেষে যোগ করো:

```prisma
enum SlotStatus {
  OPEN
  BLOCKED
}
```

2. নতুন file বানাও:

**File:** `prisma/schema/scheduling.prisma`

```prisma
model DoctorSlot {
  id          String     @id @default(uuid())
  doctorId    String
  startAt     DateTime
  endAt       DateTime
  status      SlotStatus @default(OPEN)
  blockReason String?
  blockedAt   DateTime?
  blockedById String?
  createdById String
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  doctor DoctorProfile @relation(fields: [doctorId], references: [id])

  @@unique([doctorId, startAt])
  @@index([status, startAt])
  @@map("doctor_slots")
}
```

3. `clinic.prisma`-এর `DoctorProfile`-এ, `doctorSpecialties`-এর নিচে যোগ করো:

```prisma
  slots             DoctorSlot[]
```

4. তারপর চালাও:

```bash
pnpm format && pnpm migrate --name add_doctor_slot
```

## ৪. Slot config

**File:** `src/app/config/slot.config.ts`

```ts
// Hospital-এর নিয়ম: doctor এই সীমার ভেতরে নিজের সময় বাছাই করেন
export const SLOT_CONFIG = {
	allowedSlotMinutes: [10, 15, 20, 30] as readonly number[],
	maxDatesPerRequest: 14,
	maxSlotsPerRequest: 1000,
	maxBreaks: 5,
	openMaxDaysAhead: 30,
	bookingWindowDays: 14,
	manageMaxRangeDays: 31,
	minLeadMinutes: 60,
};
```

## ৫. Slot utils (pure function, DB বা env নেই)

**File:** `src/app/module/slot/slot.utils.ts`

```ts
import { TZDate } from "@date-fns/tz";
import { format } from "date-fns";

export type TTimeRange = { startTime: string; endTime: string };
export type TSlotTime = { startAt: Date; endAt: Date };
export type TMinuteRange = { start: number; end: number };

export type TBuildSlotsInput = TTimeRange & {
	date: string;
	slotMinutes: number;
	breaks: TTimeRange[];
	timeZone: string;
};

const MINUTES_PER_DAY = 24 * 60;

export const toMinutes = (time: string): number => {
	const [hours = 0, minutes = 0] = time.split(":").map(Number);
	return hours * 60 + minutes;
};

// Shift-এর তারিখের মাঝরাত থেকে কত মিনিট। end <= start হলে রাতের shift → পরের দিন
export const toShiftWindow = ({ startTime, endTime }: TTimeRange): TMinuteRange => {
	const start = toMinutes(startTime);
	let end = toMinutes(endTime);
	if (end <= start) end += MINUTES_PER_DAY;
	return { start, end };
};

// রাতের shift-এ shift শুরুর আগের সময় মানে মাঝরাতের পরে (02:00 → 26:00)
export const toBreakWindow = (range: TTimeRange, shift: TMinuteRange): TMinuteRange => {
	let start = toMinutes(range.startTime);
	if (start < shift.start) start += MINUTES_PER_DAY;
	let end = toMinutes(range.endTime);
	while (end <= start) end += MINUTES_PER_DAY;
	return { start, end };
};

// Shift থেকে break কেটে ফেলে segment বানায়: [09–11], [11:10–13]
export const subtractBreaks = (shift: TMinuteRange, breaks: TMinuteRange[]): TMinuteRange[] => {
	const sorted = breaks.toSorted((a, b) => a.start - b.start);
	const segments: TMinuteRange[] = [];
	let cursor = shift.start;

	for (const breakWindow of sorted) {
		if (breakWindow.start > cursor) segments.push({ start: cursor, end: breakWindow.start });
		cursor = Math.max(cursor, breakWindow.end);
	}
	if (cursor < shift.end) segments.push({ start: cursor, end: shift.end });

	return segments;
};

// Clinic-এর wall time ("2026-11-21" + মিনিট) → আসল UTC মুহূর্ত
const wallTimeToUtc = (date: string, minuteOfDay: number, timeZone: string): Date => {
	const [year = 0, month = 1, day = 1] = date.split("-").map(Number);
	const dayOffset = Math.floor(minuteOfDay / MINUTES_PER_DAY);
	const minutes = minuteOfDay % MINUTES_PER_DAY;
	const wall = new TZDate(year, month - 1, day + dayOffset, Math.floor(minutes / 60), minutes % 60, 0, 0, timeZone);
	return new Date(wall.getTime());
};

export const buildSlots = ({
	date,
	startTime,
	endTime,
	slotMinutes,
	breaks,
	timeZone,
}: TBuildSlotsInput): TSlotTime[] => {
	const shift = toShiftWindow({ startTime, endTime });
	const segments = subtractBreaks(
		shift,
		breaks.map((range) => toBreakWindow(range, shift)),
	);

	const slots: TSlotTime[] = [];
	for (const segment of segments) {
		for (let start = segment.start; start + slotMinutes <= segment.end; start += slotMinutes) {
			slots.push({
				startAt: wallTimeToUtc(date, start, timeZone),
				endAt: wallTimeToUtc(date, start + slotMinutes, timeZone),
			});
		}
	}
	return slots;
};

export const hasOverlap = (a: TSlotTime, b: TSlotTime): boolean => a.startAt < b.endAt && a.endAt > b.startAt;

export const clinicDateOf = (instant: Date, timeZone: string): string =>
	format(new TZDate(instant, timeZone), "yyyy-MM-dd");

export const formatClinicTime = (instant: Date, timeZone: string): string =>
	format(new TZDate(instant, timeZone), "EEE d MMM, HH:mm");

export const clinicDayStartUtc = (date: string, timeZone: string): Date => wallTimeToUtc(date, 0, timeZone);

export const addDaysToDate = (date: string, days: number): string => {
	const value = new Date(`${date}T00:00:00Z`);
	value.setUTCDate(value.getUTCDate() + days);
	return value.toISOString().slice(0, 10);
};

export const groupByClinicDate = <T extends { startAt: Date }>(slots: T[], timeZone: string) => {
	const groups = new Map<string, T[]>();
	for (const slot of slots) {
		const key = clinicDateOf(slot.startAt, timeZone);
		const list = groups.get(key);
		if (list) list.push(slot);
		else groups.set(key, [slot]);
	}
	return [...groups].map(([date, items]) => ({ date, slots: items }));
};
```

## ৬. Slot validation

**File:** `src/app/module/slot/slot.validation.ts`

> **নোট:** আগের একটা `dateRangeQuerySchema`-এর জায়গায় এখন দুটো schema: `manageSlotQuerySchema` (`from`/`to` দুটোই আবশ্যক) আর `publicSlotQuerySchema` (`from`/`to` optional)।

```ts
import { SLOT_CONFIG } from "@/app/config/slot.config";
import * as z from "zod";
import { toBreakWindow, toShiftWindow } from "./slot.utils";

const timeSchema = z.string().regex(/^([01]\d|2[0-3]):[0-5]\d$/, "Time must be in HH:mm (24-hour) format");
const clinicDateSchema = z.iso.date("Date must be in YYYY-MM-DD format");
const timeRangeSchema = z.object({ startTime: timeSchema, endTime: timeSchema });
const slotIdsSchema = z
	.array(z.uuid("Invalid slot id"))
	.min(1, "Select at least one slot")
	.max(SLOT_CONFIG.maxSlotsPerRequest)
	.transform((ids) => [...new Set(ids)]);

const openSlotsSchema = z
	.object({
		dates: z
			.array(clinicDateSchema)
			.min(1, "Select at least one date")
			.max(SLOT_CONFIG.maxDatesPerRequest, `At most ${SLOT_CONFIG.maxDatesPerRequest} dates per request`)
			.transform((dates) => [...new Set(dates)].toSorted()),
		startTime: timeSchema,
		endTime: timeSchema,
		slotMinutes: z
			.number()
			.int()
			.refine((value) => SLOT_CONFIG.allowedSlotMinutes.includes(value), {
				error: `Slot length must be one of ${SLOT_CONFIG.allowedSlotMinutes.join(", ")} minutes`,
			}),
		breaks: z.array(timeRangeSchema).max(SLOT_CONFIG.maxBreaks, `At most ${SLOT_CONFIG.maxBreaks} breaks`).default([]),
	})
	.superRefine((data, ctx) => {
		if (data.startTime === data.endTime) {
			ctx.addIssue({ code: "custom", path: ["endTime"], message: "End time must be different from start time" });
			return;
		}

		const shift = toShiftWindow(data);
		if (shift.end - shift.start < data.slotMinutes) {
			ctx.addIssue({ code: "custom", path: ["endTime"], message: "Shift is shorter than one slot" });
		}

		const windows = data.breaks.map((range, index) => ({ index, range, window: toBreakWindow(range, shift) }));

		for (const { index, range, window } of windows) {
			if (range.startTime === range.endTime) {
				ctx.addIssue({ code: "custom", path: ["breaks", index], message: "Break start and end are the same" });
			} else if (window.start <= shift.start || window.end >= shift.end) {
				ctx.addIssue({ code: "custom", path: ["breaks", index], message: "Break must be inside the shift" });
			}
		}

		const sorted = windows.toSorted((a, b) => a.window.start - b.window.start);
		sorted.forEach((current, i) => {
			const next = sorted[i + 1];
			if (next && next.window.start < current.window.end) {
				ctx.addIssue({ code: "custom", path: ["breaks", next.index], message: "Breaks must not overlap" });
			}
		});
	});

const blockSlotsSchema = z
	.object({
		slotIds: slotIdsSchema.optional(),
		from: z.iso
			.datetime({ offset: true })
			.transform((value) => new Date(value))
			.optional(),
		to: z.iso
			.datetime({ offset: true })
			.transform((value) => new Date(value))
			.optional(),
		reason: z.string().trim().min(3, "Reason must be at least 3 characters").max(200),
	})
	.superRefine((data, ctx) => {
		const byIds = data.slotIds !== undefined;
		const byRange = data.from !== undefined || data.to !== undefined;

		if (byIds === byRange) {
			ctx.addIssue({ code: "custom", path: ["slotIds"], message: "Send either slotIds or a from–to range" });
			return;
		}
		if (byRange && (!data.from || !data.to || data.from >= data.to)) {
			ctx.addIssue({ code: "custom", path: ["to"], message: "'to' must be after 'from'" });
		}
	});

const unblockSlotsSchema = z.object({ slotIds: slotIdsSchema });

const doctorParamSchema = z.object({ doctorId: z.union([z.literal("me"), z.uuid("Invalid doctor id")]) });
const publicDoctorParamSchema = z.object({ doctorId: z.uuid("Invalid doctor id") });
const slotParamSchema = doctorParamSchema.extend({ slotId: z.uuid("Invalid slot id") });

// Doctor/admin: calendar-এর week/month সবসময় জানা, অতীতও দেখা লাগে → দুটোই আবশ্যক
const manageSlotQuerySchema = z
	.object({ from: clinicDateSchema, to: clinicDateSchema })
	.refine((query) => query.from <= query.to, { error: "'from' must be on or before 'to'", path: ["to"] });

// Public: না দিলে আজ থেকে booking window (১৪ দিন)
const publicSlotQuerySchema = z
	.object({ from: clinicDateSchema.optional(), to: clinicDateSchema.optional() })
	.refine((query) => !query.from || !query.to || query.from <= query.to, {
		error: "'from' must be on or before 'to'",
		path: ["to"],
	});

export const SlotValidation = {
	openSlotsSchema,
	blockSlotsSchema,
	unblockSlotsSchema,
	doctorParamSchema,
	publicDoctorParamSchema,
	slotParamSchema,
	manageSlotQuerySchema,
	publicSlotQuerySchema,
};
```

## ৭. Slot interface

**File:** `src/app/module/slot/slot.interface.ts`

```ts
import type * as z from "zod";
import type { SlotValidation } from "./slot.validation";

export type TOpenSlotsPayload = z.infer<typeof SlotValidation.openSlotsSchema>;
export type TBlockSlotsPayload = z.infer<typeof SlotValidation.blockSlotsSchema>;
export type TUnblockSlotsPayload = z.infer<typeof SlotValidation.unblockSlotsSchema>;
export type TManageSlotQuery = z.infer<typeof SlotValidation.manageSlotQuerySchema>;
export type TPublicSlotQuery = z.infer<typeof SlotValidation.publicSlotQuerySchema>;
```

## ৮. Slot service

**File:** `src/app/module/slot/slot.service.ts`

> **নোট:** Public list এখন `{ from, to, days }` ফেরত দেয় (আগে শুধু দিনের array)। `from`/`to` না দিলে আজ থেকে ১৪ দিন (আজ + পরের ১৩ দিন) ধরা হয়।

```ts
import { envVars } from "@/app/config/env";
import { SLOT_CONFIG } from "@/app/config/slot.config";
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import type { TAuthPayload } from "@/app/utils/token";
import type { Prisma } from "@/generated/prisma/client";
import { Role, SlotStatus } from "@/generated/prisma/enums";
import httpStatus from "http-status";
import type {
	TBlockSlotsPayload,
	TManageSlotQuery,
	TOpenSlotsPayload,
	TPublicSlotQuery,
	TUnblockSlotsPayload,
} from "./slot.interface";
import {
	addDaysToDate,
	buildSlots,
	clinicDateOf,
	clinicDayStartUtc,
	formatClinicTime,
	groupByClinicDate,
	hasOverlap,
} from "./slot.utils";

const TZ = envVars.CLINIC_TIME_ZONE;
const DAY_MS = 24 * 60 * 60 * 1000;

type TDateRange = { from: string; to: string };

// "me" বা id → আসল doctorProfile.id। Doctor শুধু নিজেরটা, admin যেকোনো doctor-এর
const resolveDoctorId = async (authUser: TAuthPayload, doctorIdParam: string): Promise<string> => {
	if (authUser.role === Role.DOCTOR) {
		const own = await prisma.doctorProfile.findUnique({
			where: { userId: authUser.userId },
			select: { id: true },
		});
		if (!own) throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
		if (doctorIdParam !== "me" && doctorIdParam !== own.id) {
			throw new AppError(httpStatus.FORBIDDEN, "You can only manage your own slots");
		}
		return own.id;
	}

	if (doctorIdParam === "me") {
		throw new AppError(httpStatus.BAD_REQUEST, "'me' works only for doctors — use a doctor id");
	}

	const doctor = await prisma.doctorProfile.findFirst({
		where: { id: doctorIdParam, user: { isDeleted: false } },
		select: { id: true },
	});
	if (!doctor) throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
	return doctor.id;
};

// Clinic-এর তারিখ (inclusive) → UTC range, দিনের সীমা সহ
const toUtcRange = ({ from, to }: TDateRange, maxDays: number) => {
	const days = (Date.parse(to) - Date.parse(from)) / DAY_MS + 1;
	if (days > maxDays) {
		throw new AppError(httpStatus.BAD_REQUEST, `Date range can be at most ${maxDays} days`);
	}
	return { gte: clinicDayStartUtc(from, TZ), lt: clinicDayStartUtc(addDaysToDate(to, 1), TZ) };
};

const openSlots = async (authUser: TAuthPayload, doctorIdParam: string, payload: TOpenSlotsPayload) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);
	const now = new Date();
	const today = clinicDateOf(now, TZ);
	const lastDate = addDaysToDate(today, SLOT_CONFIG.openMaxDaysAhead);

	const pastDate = payload.dates.find((date) => date < today);
	if (pastDate) throw new AppError(httpStatus.BAD_REQUEST, `${pastDate} is in the past`);

	const tooFar = payload.dates.find((date) => date > lastDate);
	if (tooFar) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			`Slots can be opened at most ${SLOT_CONFIG.openMaxDaysAhead} days ahead (until ${lastDate})`,
		);
	}

	const allSlots = payload.dates.flatMap((date) =>
		buildSlots({
			date,
			startTime: payload.startTime,
			endTime: payload.endTime,
			slotMinutes: payload.slotMinutes,
			breaks: payload.breaks,
			timeZone: TZ,
		}),
	);
	const futureSlots = allSlots.filter((slot) => slot.startAt > now);

	const [firstSlot] = futureSlots;
	const lastSlot = futureSlots.at(-1);
	if (!firstSlot || !lastSlot) {
		throw new AppError(httpStatus.BAD_REQUEST, "All selected times are in the past");
	}
	if (futureSlots.length > SLOT_CONFIG.maxSlotsPerRequest) {
		throw new AppError(
			httpStatus.BAD_REQUEST,
			`This would create ${futureSlots.length} slots; the limit is ${SLOT_CONFIG.maxSlotsPerRequest} per request`,
		);
	}

	const created = await prisma.$transaction(async (tx) => {
		// একই doctor-এর দুটো open request একসাথে এলে একটা আরেকটার জন্য অপেক্ষা করে
		await tx.$executeRaw`SELECT pg_advisory_xact_lock(hashtext(${doctorId}))`;

		const existing = await tx.doctorSlot.findMany({
			where: { doctorId, startAt: { lt: lastSlot.endAt }, endAt: { gt: firstSlot.startAt } },
			select: { startAt: true, endAt: true },
		});

		const conflicts = futureSlots.filter((slot) => existing.some((other) => hasOverlap(slot, other)));
		const [firstConflict] = conflicts;
		if (firstConflict) {
			throw new AppError(
				httpStatus.CONFLICT,
				`${conflicts.length} new slot(s) overlap existing slots — first at ${formatClinicTime(firstConflict.startAt, TZ)}`,
			);
		}

		const result = await tx.doctorSlot.createMany({
			data: futureSlots.map((slot) => ({ doctorId, ...slot, createdById: authUser.userId })),
		});
		return result.count;
	});

	return { created, skippedPast: allSlots.length - futureSlots.length };
};

const getPublicSlots = async (doctorId: string, query: TPublicSlotQuery) => {
	const doctor = await prisma.doctorProfile.findFirst({
		where: { id: doctorId, user: { isDeleted: false, isActive: true } },
		select: { id: true },
	});
	if (!doctor) throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");

	const now = new Date();
	const today = clinicDateOf(now, TZ);
	// ১৪ দিনের window = আজ + পরের ১৩ দিন (আজ নিজেও ১ দিন)
	const lastBookableDate = addDaysToDate(today, SLOT_CONFIG.bookingWindowDays - 1);

	// অতীতের from দিলেও আজ থেকে ধরা হয় — অতীতের slot এমনিতেও book করা যায় না
	const from = query.from && query.from > today ? query.from : today;
	const to = query.to ?? lastBookableDate;

	if (from > lastBookableDate || to > lastBookableDate) {
		throw new AppError(httpStatus.BAD_REQUEST, `Booking is open until ${lastBookableDate}`);
	}
	if (from > to) {
		throw new AppError(httpStatus.BAD_REQUEST, "'from' must be on or before 'to'");
	}

	const range = toUtcRange({ from, to }, SLOT_CONFIG.bookingWindowDays);
	const earliest = new Date(now.getTime() + SLOT_CONFIG.minLeadMinutes * 60_000);

	const slots = await prisma.doctorSlot.findMany({
		where: { doctorId, AND: [{ startAt: range }, { startAt: { gte: earliest } }] },
		// দিন ৩: OPEN slot-এ active appointment থাকলে (booked) বাদ দিতে হবে
		select: { id: true, startAt: true, endAt: true, status: true, blockReason: true },
		orderBy: { startAt: "asc" },
	});

	const items = slots.map(({ status, blockReason, ...slot }) =>
		status === SlotStatus.BLOCKED
			? { ...slot, status: "BLOCKED" as const, blockReason }
			: { ...slot, status: "AVAILABLE" as const },
	);

	// from/to ফেরত দেওয়া হয় → frontend জানে calendar-এর কোন দিনগুলো দেখাবে
	return { from, to, days: groupByClinicDate(items, TZ) };
};

const getManageSlots = async (authUser: TAuthPayload, doctorIdParam: string, query: TManageSlotQuery) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);
	const range = toUtcRange(query, SLOT_CONFIG.manageMaxRangeDays);

	const slots = await prisma.doctorSlot.findMany({
		where: { doctorId, startAt: range },
		// দিন ৩: active appointment include করে BOOKED (রোগী + appointment number) দেখাবে
		select: { id: true, startAt: true, endAt: true, status: true, blockReason: true, blockedAt: true },
		orderBy: { startAt: "asc" },
	});

	return groupByClinicDate(slots, TZ);
};

// Prisma-তে undefined মানে "filter নেই"। তাই range ফাঁকা থাকলে সব slot block হয়ে যেত — এখানে আটকানো হলো
const toTargetFilter = (payload: TBlockSlotsPayload): Prisma.DoctorSlotWhereInput => {
	if (payload.slotIds) return { id: { in: payload.slotIds } };
	if (payload.from && payload.to) return { startAt: { lt: payload.to }, endAt: { gt: payload.from } };
	throw new AppError(httpStatus.BAD_REQUEST, "Send either slotIds or a from–to range");
};

const blockSlots = async (authUser: TAuthPayload, doctorIdParam: string, payload: TBlockSlotsPayload) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);
	const target = toTargetFilter(payload);
	const now = new Date();

	const result = await prisma.$transaction(async (tx) => {
		const blocked = await tx.doctorSlot.updateManyAndReturn({
			where: { doctorId, status: SlotStatus.OPEN, AND: [{ startAt: { gt: now } }, target] },
			data: {
				status: SlotStatus.BLOCKED,
				blockReason: payload.reason,
				blockedAt: now,
				blockedById: authUser.userId,
			},
			select: { id: true },
		});

		if (blocked.length === 0) {
			throw new AppError(httpStatus.BAD_REQUEST, "No open future slots matched — they may be past or already blocked");
		}

		// দিন ৩: এই slot-গুলোর active appointment → CANCELLED (cancelReason = payload.reason),
		//        কাদের email যাবে সেটা return করো; audit log-ও এখানে (একই tx)
		return { blocked: blocked.length, cancelledAppointments: 0 };
	});

	// দিন ৩: commit-এর পরে প্রতিটা রোগীকে appointment-cancelled email + SSE event
	return result;
};

const unblockSlots = async (authUser: TAuthPayload, doctorIdParam: string, payload: TUnblockSlotsPayload) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);

	const result = await prisma.doctorSlot.updateMany({
		where: { doctorId, id: { in: payload.slotIds }, status: SlotStatus.BLOCKED, startAt: { gt: new Date() } },
		data: { status: SlotStatus.OPEN, blockReason: null, blockedAt: null, blockedById: null },
	});

	if (result.count === 0) {
		throw new AppError(httpStatus.BAD_REQUEST, "No blocked future slots matched");
	}
	return { unblocked: result.count };
};

const deleteSlot = async (authUser: TAuthPayload, doctorIdParam: string, slotId: string) => {
	const doctorId = await resolveDoctorId(authUser, doctorIdParam);

	const slot = await prisma.doctorSlot.findFirst({
		where: { id: slotId, doctorId },
		select: { id: true, startAt: true },
	});
	if (!slot) throw new AppError(httpStatus.NOT_FOUND, "Slot not found");
	if (slot.startAt <= new Date()) {
		throw new AppError(httpStatus.BAD_REQUEST, "Past slots can't be deleted");
	}

	// দিন ৩: কোনো appointment থাকলে 409 "This slot has bookings — block it instead"
	await prisma.doctorSlot.delete({ where: { id: slot.id } });
};

export const SlotService = {
	openSlots,
	getPublicSlots,
	getManageSlots,
	blockSlots,
	unblockSlots,
	deleteSlot,
};
```

## ৯. Slot controller

**File:** `src/app/module/slot/slot.controller.ts`

> **নোট:** `dateRangeQuerySchema` আর নেই, তাই public list-এ `publicSlotQuerySchema` আর manage list-এ `manageSlotQuerySchema` দিয়ে query parse করা হয়।

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { getAuthUser } from "@/app/shared/getAuthUser";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { SlotService } from "./slot.service";
import { SlotValidation } from "./slot.validation";

const openSlots = catchAsync(async (req, res) => {
	const { doctorId } = SlotValidation.doctorParamSchema.parse(req.params);
	const result = await SlotService.openSlots(getAuthUser(req), doctorId, req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: `${result.created} slots opened`,
		data: result,
	});
});

const getPublicSlots = catchAsync(async (req, res) => {
	const { doctorId } = SlotValidation.publicDoctorParamSchema.parse(req.params);
	const query = SlotValidation.publicSlotQuerySchema.parse(req.query);
	const result = await SlotService.getPublicSlots(doctorId, query);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Slots retrieved successfully", data: result });
});

const getManageSlots = catchAsync(async (req, res) => {
	const { doctorId } = SlotValidation.doctorParamSchema.parse(req.params);
	const query = SlotValidation.manageSlotQuerySchema.parse(req.query);
	const result = await SlotService.getManageSlots(getAuthUser(req), doctorId, query);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Slots retrieved successfully", data: result });
});

const blockSlots = catchAsync(async (req, res) => {
	const { doctorId } = SlotValidation.doctorParamSchema.parse(req.params);
	const result = await SlotService.blockSlots(getAuthUser(req), doctorId, req.body);

	sendResponse(res, { statusCode: httpStatus.OK, message: `${result.blocked} slots blocked`, data: result });
});

const unblockSlots = catchAsync(async (req, res) => {
	const { doctorId } = SlotValidation.doctorParamSchema.parse(req.params);
	const result = await SlotService.unblockSlots(getAuthUser(req), doctorId, req.body);

	sendResponse(res, { statusCode: httpStatus.OK, message: `${result.unblocked} slots unblocked`, data: result });
});

const deleteSlot = catchAsync(async (req, res) => {
	const { doctorId, slotId } = SlotValidation.slotParamSchema.parse(req.params);
	await SlotService.deleteSlot(getAuthUser(req), doctorId, slotId);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Slot deleted successfully", data: null });
});

export const SlotController = {
	openSlots,
	getPublicSlots,
	getManageSlots,
	blockSlots,
	unblockSlots,
	deleteSlot,
};
```

## ১০. Slot route

**File:** `src/app/module/slot/slot.route.ts`

```ts
import { checkAuth, STAFF } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Router } from "express";
import { SlotController } from "./slot.controller";
import { SlotValidation } from "./slot.validation";

// mount path-এ :doctorId আছে → mergeParams না দিলে req.params.doctorId = undefined
const router = Router({ mergeParams: true });

router.get("/", SlotController.getPublicSlots);

router.use(checkAuth(...STAFF));

router.post("/", validateRequest(SlotValidation.openSlotsSchema), SlotController.openSlots);
router.get("/manage", SlotController.getManageSlots);
router.post("/block", validateRequest(SlotValidation.blockSlotsSchema), SlotController.blockSlots);
router.post("/unblock", validateRequest(SlotValidation.unblockSlotsSchema), SlotController.unblockSlots);
router.delete("/:slotId", SlotController.deleteSlot);

export const SlotRoutes = router;
```

## ১১. Route register করা

**File:** `src/app/routes/index.ts` — `/doctors`-এর **আগে** বসাতে হবে:

```ts
import { SlotRoutes } from "../module/slot/slot.route";
// ...
	{ path: "/doctors/:doctorId/slots", route: SlotRoutes },
	{ path: "/doctors", route: DoctorRoutes },
```

> **⚠️ কেন আগে:** `DoctorRoutes`-এ `router.use(checkAuth(...ADMINS))` আছে, যেটা path ছাড়া চলে। `/doctors` আগে mount হলে `/doctors/abc/slots`-এর request আগে `DoctorRoutes`-এ ঢুকবে আর ওই middleware চলবে। তখন public slot list পেতে গিয়েও রোগী 401 পাবে।

## ১২. Unit test

**File:** `src/app/module/slot/slot.utils.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { buildSlots, type TSlotTime } from "./slot.utils";

const base = { breaks: [], timeZone: "Asia/Dhaka" }; // UTC+6
const starts = (slots: TSlotTime[]) => slots.map((slot) => slot.startAt.toISOString());

describe("buildSlots", () => {
	it("splits a simple shift into equal slots", () => {
		const slots = buildSlots({ ...base, date: "2026-11-21", startTime: "09:00", endTime: "10:00", slotMinutes: 15 });
		expect(slots).toHaveLength(4);
		expect(slots[0]?.startAt.toISOString()).toBe("2026-11-21T03:00:00.000Z");
		expect(slots.at(-1)?.endAt.toISOString()).toBe("2026-11-21T04:00:00.000Z");
	});

	it("runs an overnight shift into the next day", () => {
		const slots = buildSlots({ ...base, date: "2026-11-20", startTime: "22:00", endTime: "06:00", slotMinutes: 30 });
		expect(slots).toHaveLength(16);
		expect(slots.at(-1)?.startAt.toISOString()).toBe("2026-11-20T23:30:00.000Z"); // Sat 05:30 Dhaka
	});

	it("drops the leftover when the range doesn't divide evenly", () => {
		const slots = buildSlots({ ...base, date: "2026-11-21", startTime: "09:00", endTime: "09:50", slotMinutes: 20 });
		expect(slots).toHaveLength(2);
	});

	it("restarts counting right after a break", () => {
		const slots = buildSlots({
			...base,
			date: "2026-11-21",
			startTime: "09:00",
			endTime: "13:00",
			slotMinutes: 15,
			breaks: [{ startTime: "11:00", endTime: "11:10" }],
		});
		expect(slots).toHaveLength(15);
		expect(starts(slots)).not.toContain("2026-11-21T05:00:00.000Z"); // 11:00
		expect(starts(slots)).toContain("2026-11-21T05:10:00.000Z"); // 11:10
	});

	it("handles a break after midnight in an overnight shift", () => {
		const slots = buildSlots({
			...base,
			date: "2026-11-20",
			startTime: "22:00",
			endTime: "06:00",
			slotMinutes: 30,
			breaks: [{ startTime: "02:00", endTime: "02:30" }],
		});
		expect(slots).toHaveLength(15);
		expect(starts(slots)).not.toContain("2026-11-20T20:00:00.000Z"); // Sat 02:00 Dhaka
	});

	it("uses the given time zone, not the server's", () => {
		const slots = buildSlots({
			...base,
			timeZone: "America/New_York",
			date: "2026-11-21",
			startTime: "09:00",
			endTime: "09:15",
			slotMinutes: 15,
		});
		expect(slots[0]?.startAt.toISOString()).toBe("2026-11-21T14:00:00.000Z"); // EST = UTC-5
	});
});
```

## ১৩. Postman দিয়ে test

```text
POST /api/v1/doctors/me/slots          (doctor login)
{ "dates": ["2026-11-22","2026-11-23"], "startTime": "09:00", "endTime": "13:00",
  "slotMinutes": 15, "breaks": [{ "startTime": "11:00", "endTime": "11:15" }] }
→ 201 { created: 30, skippedPast: 0 }   আবার পাঠাও → 409 overlap

GET  /api/v1/doctors/<id>/slots                                      (login ছাড়া, আজ থেকে ১৪ দিন)
GET  /api/v1/doctors/<id>/slots?from=2026-11-22&to=2026-11-23        (login ছাড়া)
→ 200 { from, to, days: [{ date, slots: [...] }] }
GET  /api/v1/doctors/me/slots/manage?from=2026-11-22&to=2026-11-28   (from/to দুটোই লাগবে)
POST /api/v1/doctors/me/slots/block   { "from": "2026-11-23T00:00:00+06:00", "to": "2026-11-24T00:00:00+06:00", "reason": "Doctor is sick" }
POST /api/v1/doctors/me/slots/unblock { "slotIds": ["..."] }
DELETE /api/v1/doctors/me/slots/<slotId>
```