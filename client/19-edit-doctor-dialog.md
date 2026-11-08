# 19 - Edit Doctor (TanStack Form + দুই রকম Zod Schema + শুধু বদলানো field পাঠানো)

> Note 16–18-এর doctor table-এর "⋯" menu-তে এবার **Edit**। Dialog-এ একটা form, যেটা row-এর data দিয়ে আগে থেকে ভরা। Save চাপলে **শুধু যে field বদলেছে** সেটুকু backend-এ `PATCH` হয়। Form (`DoctorForm`) এমনভাবে বানানো যে পরে **Create doctor**-এও একই form ব্যবহার হবে।

## 🧭 এক নজরে

```
Edit চাপা → EditDoctorDialog → DoctorForm (row-এর data দিয়ে আগে থেকে ভরা)
  → টাইপ করার সময়: form schema validate (string-ভিত্তিক)
  → Submit: string থেকে আসল type (Number ইত্যাদি) → শুধু যা বদলেছে সেটুকু বের করা
     → কিছু বদলায়নি? → "No changes" toast, request যায় না
  → updateDoctorAction: id + payload আবার validate (server schema) → backend PATCH
  → সফল: list invalidate → dialog বন্ধ → toast
```

### কোন file-এ কী

| File | কাজ |
|---|---|
| `types/doctor.types.ts` | `IDoctorProfilePayload`, `IUpdateDoctorPayload` |
| `zod/doctor.validation.ts` (নতুন) | **form schema** (browser, string) + **server schema** (action, আসল type) |
| `services/doctor.services.ts` | `updateDoctor` → `PATCH /doctors/:id` |
| `actions/doctor.actions.ts` | `updateDoctorAction` → id + payload validate |
| `shared/form/AppSelectField.tsx` (নতুন) | Select-এর জন্য `AppField`-এর মতো field |
| `doctorForm.utils.ts` (নতুন) | row → form মান, form মান → payload, কী বদলেছে |
| `DoctorForm.tsx` (নতুন) | form UI (create + edit দুজনের) |
| `EditDoctorDialog.tsx` (নতুন) | dialog + submit logic |
| `DoctorsTable.tsx` | Edit action + dialog জোড়া |

---

## ধাপ ১: `src/types/doctor.types.ts` — শেষে যোগ করো

```ts
// create আর edit দুজনেরই কমন profile field (backend schema অনুযায়ী)
export interface IDoctorProfilePayload {
	name: string;
	contactNumber: string;
	address?: string;
	profilePhoto?: string;
	gender: Gender;
	experience: number;
	appointmentFee: number;
	qualification: string;
	designation: string;
	currentWorkingPlace: string;
}

// backend PATCH /doctors/:id → { doctor: {...} }, শুধু যা বদলেছে
export interface IUpdateDoctorPayload {
	doctor: Partial<IDoctorProfilePayload>;
}
```

- **`IDoctorProfilePayload`** → backend-এ যা পাঠানো যায় (`IDoctor` না — ওতে `id`, `averageRating`, `createdAt`-এর মতো field আছে যা admin বদলাতে পারে না)।
- Create আর Edit দুজনেই একই profile field → একবার লিখে দুই জায়গায়।
- **`IUpdateDoctorPayload`** → backend-এর `updateDoctorValidationSchema` body চায় `{ doctor: {...} }` আকারে।
- **`Partial<...>`** → সব field optional → শুধু যা বদলেছে।

---

## ধাপ ২: `src/zod/doctor.validation.ts` (নতুন) — দুই রকম schema ⭐

```ts
import * as z from "zod";

const BD_PHONE_REGEX = /^(\+?880|0)1[3-9]\d{8}$/;

// ---------------- Form schema (browser) ----------------
// input field সবসময় string দেয়, তাই সব মান string হিসেবে যাচাই হয়।
// optional field-এ "" মানে "দেওয়া হয়নি"।

const requiredText = (label: string, min: number, max: number) =>
	z
		.string()
		.trim()
		.min(1, `${label} is required`)
		.min(min, `${label} must be at least ${min} characters`)
		.max(max, `${label} must be at most ${max} characters`);

export const doctorProfileFormSchema = z.object({
	name: requiredText("Name", 3, 30),
	contactNumber: z.string().trim().regex(BD_PHONE_REGEX, "Enter a valid Bangladeshi phone number (01XXXXXXXXX)"),
	gender: z.union([z.enum(["MALE", "FEMALE"]), z.literal("")]).refine((value) => value !== "", "Select a gender"),
	designation: requiredText("Designation", 2, 50),
	currentWorkingPlace: requiredText("Working place", 2, 50),
	qualification: requiredText("Qualification", 2, 50),
	experience: z.string().trim().regex(/^\d+$/, "Experience must be a whole number (years)"),
	appointmentFee: z
		.string()
		.trim()
		.regex(/^\d+(\.\d{1,2})?$/, "Fee must be a positive number"),
	address: z.union([
		z.literal(""),
		z.string().trim().min(10, "Address must be at least 10 characters").max(100, "Address must be at most 100 characters"),
	]),
	profilePhoto: z.union([
		z.literal(""),
		z.url({ protocol: /^https?$/, error: "Must be a valid http/https URL" }),
	]),
});

export type DoctorProfileFormValues = z.input<typeof doctorProfileFormSchema>;

// ---------------- Server schema (server action) ----------------
// Server Action public endpoint, তাই আসল type আবার যাচাই হয়।
// z.object অজানা key বাদ দেয়: কেউ averageRating বা isDeleted পাঠালেও backend-এ যাবে না।

export const doctorProfilePayloadSchema = z.object({
	name: z.string().trim().min(3).max(30),
	contactNumber: z.string().trim().regex(BD_PHONE_REGEX, "Invalid Bangladeshi phone number"),
	address: z.string().trim().min(10).max(100).optional(),
	profilePhoto: z.url({ protocol: /^https?$/ }).optional(),
	gender: z.enum(["MALE", "FEMALE"]),
	experience: z.int().nonnegative(),
	appointmentFee: z.number().nonnegative(),
	qualification: z.string().trim().min(2).max(50),
	designation: z.string().trim().min(2).max(50),
	currentWorkingPlace: z.string().trim().min(2).max(50),
});

export const updateDoctorPayloadSchema = z.object({
	doctor: doctorProfilePayloadSchema
		.partial()
		.refine((doctor) => Object.keys(doctor).length > 0, "Nothing to update"),
});
```

### দুটো schema কেন?

| | Form schema (`doctorProfileFormSchema`) | Server schema (`doctorProfilePayloadSchema`) |
|---|---|---|
| কোথায় চলে | browser, টাইপ করার সময় | Server Action-এ, backend-এ পাঠানোর আগে |
| মানের type | **সব string** (input সবসময় string দেয়: `"5"`, `""`) | **আসল type** (`5` number, `undefined`) |
| Optional field | `""` মানে "দেওয়া হয়নি" | `undefined` মানে "দেওয়া হয়নি" |
| কেন | user-কে সুন্দর error message দেখাতে | Server Action public — কেউ সরাসরি call করে আজেবাজে পাঠালেও আটকাতে |

```
Input "5"  → form schema: string, শুধু অঙ্ক? ✅ → Number("5") → 5 → server schema: int, ≥ 0? ✅ → backend
```

### Form schema-র কৌশল

| Field | নিয়ম | কেন এভাবে |
|---|---|---|
| `requiredText(label, min, max)` | trim → ফাঁকা না → min → max | ৫টা text field-এ একই নিয়ম, একটা helper। প্রথমে `min(1, "is required")` → ফাঁকা হলে "required", কম হলে "at least N" — আলাদা message |
| `contactNumber` | `BD_PHONE_REGEX` | `01XXXXXXXXX`, `+8801…`, `8801…` — backend-এর একই regex |
| `gender` | `"MALE" \| "FEMALE" \| ""` + `refine(≠ "")` | শুরুতে form-এ কিছু বাছা থাকে না (`""`), তাই type-এ `""` রাখতে হয়; কিন্তু submit-এ `""` হলে "Select a gender" |
| `experience` | `/^\d+$/` | শুধু পূর্ণ সংখ্যা (string হিসেবে) |
| `appointmentFee` | `/^\d+(\.\d{1,2})?$/` | `500` বা `500.50` (সর্বোচ্চ ২ দশমিক) |
| `address`, `profilePhoto` | `z.union([z.literal(""), ...])` | **ফাঁকা অথবা** ঠিক মান। ফাঁকা থাকলে min-10 error দেয় না |
| `profilePhoto` | `z.url({ protocol: /^https?$/ })` | শুধু http/https (`javascript:` বা `ftp:` না) |

- **`z.input<typeof schema>`** → form-এর মানের type (`DoctorProfileFormValues`)। `input` (output না), কারণ form schema-তে কোনো transform নেই, আর form যা ধরে রাখে সেটাই input।

### Server schema-র কৌশল

- **`z.object` অজানা key বাদ দেয়** → কেউ `{ averageRating: 5, isDeleted: false }` পাঠালে parse-এর পর সেগুলো আর থাকে না, backend-এ যায় না।
- `z.int().nonnegative()` → পূর্ণ সংখ্যা, ≥ ০।
- **`updateDoctorPayloadSchema`** → `.partial()` (সব optional) + `.refine(কমপক্ষে ১টা key)` → খালি `{ doctor: {} }` হলে "Nothing to update"।
- `doctorProfilePayloadSchema` আলাদা রাখা → পরে Create-এ (সব field বাধ্যতামূলক) সরাসরি ব্যবহার।

> Backend-এর `doctor.validation.ts`-এর নিয়মের সাথে মিল রাখা হয়েছে (নাম ৩–৩০, address ১০–১০০, …)। Backend-এ বদলালে এখানেও।

---

## ধাপ ৩: `src/services/doctor.services.ts` — `updateDoctor`

Import-এ type যোগ করো:
```ts
import type { IDoctor, IUpdateDoctorPayload } from "@/types/doctor.types";
```

শেষে:
```ts
export const updateDoctor = async (id: string, payload: IUpdateDoctorPayload) =>
	httpClient.patch<IDoctor>(`/doctors/${encodeURIComponent(id)}`, payload);
```

- Backend: `PATCH /doctors/:id` → `checkAuth(ADMIN, SUPER_ADMIN)` → `validateRequest(updateDoctorValidationSchema)` → update।
- **`PATCH`** (PUT না) → শুধু পাঠানো field বদলায়, বাকিগুলো যেমন আছে।

---

## ধাপ ৪: `src/actions/doctor.actions.ts` — `updateDoctorAction`

Import আপডেট করো:
```ts
import { deleteDoctor, updateDoctor } from "@/services/doctor.services";
import { updateDoctorPayloadSchema } from "@/zod/doctor.validation";
```

শেষে:
```ts
export async function updateDoctorAction(id: unknown, payload: unknown): Promise<ActionResult<IDoctor>> {
	const parsedId = doctorIdSchema.safeParse(id);
	if (!parsedId.success) {
		return { success: false, message: "Invalid doctor id" };
	}

	const parsedPayload = updateDoctorPayloadSchema.safeParse(payload);
	if (!parsedPayload.success) {
		return { success: false, message: parsedPayload.error.issues[0]?.message ?? "Invalid input" };
	}

	try {
		return await updateDoctor(parsedId.data, parsedPayload.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "updateDoctorAction") };
	}
}
```

- Delete action-এর (note 18) মতোই: `unknown` → Zod → ভুল হলে `{ success: false, message }` return (throw না)।
- **id আর payload দুটোই** validate।
- `parsedPayload.data` পাঠানো (আসল `payload` না) → অজানা key ছেঁটে ফেলা version।
- `issues[0]?.message` → প্রথম error-এর message toast-এ।

---

## ধাপ ৫: `src/components/shared/form/AppSelectField.tsx` (নতুন)

Gender-এর জন্য লাগবে। AppField-এর মতো একই error দেখানোর নিয়ম মেনে বানানো, পরে অন্য form-এও কাজে লাগবে।

```tsx
"use client";

import { Field, FieldError, FieldLabel } from "@/components/ui/field";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import type { AnyFieldApi } from "@tanstack/react-form";

interface AppSelectFieldProps {
	field: AnyFieldApi;
	label: string;
	options: Array<{ label: string; value: string }>;
	placeholder?: string;
}

export default function AppSelectField({ field, label, options, placeholder = "Select..." }: AppSelectFieldProps) {
	const { meta } = field.state;

	// AppField-এর মতো: field ছেড়ে গেলে বা submit চাপলে তবেই error দেখায়
	const showError = !meta.isValid && (meta.isBlurred || field.form.state.submissionAttempts > 0);
	const errorId = `${field.name}-error`;

	return (
		<Field data-invalid={showError}>
			<FieldLabel htmlFor={field.name}>{label}</FieldLabel>

			<Select
				name={field.name}
				value={field.state.value ?? ""}
				onValueChange={(value) => field.handleChange(value)}
				onOpenChange={(open) => {
					if (!open) field.handleBlur(); // menu বন্ধ হলে "blurred" ধরা হয়
				}}
			>
				<SelectTrigger
					id={field.name}
					className="w-full"
					aria-invalid={showError}
					aria-describedby={showError ? errorId : undefined}
				>
					<SelectValue placeholder={placeholder} />
				</SelectTrigger>
				<SelectContent>
					{options.map((option) => (
						<SelectItem key={option.value} value={option.value}>
							{option.label}
						</SelectItem>
					))}
				</SelectContent>
			</Select>

			{showError && <FieldError id={errorId} errors={meta.errors} />}
		</Field>
	);
}
```

| অংশ | কাজ |
|---|---|
| `field: AnyFieldApi` | TanStack Form-এর field (note 07-এর `AppField`-এর মতো) |
| `showError` | field ছেড়ে গেলে (`isBlurred`) বা submit চাপলে তবেই error — টাইপ/বাছাইয়ের মাঝখানে লাল দেখায় না |
| `value={field.state.value ?? ""}` | `""` হলে Select placeholder দেখায় |
| `onValueChange` → `handleChange` | বাছাই করলে form-এর মান বদলায় |
| **`onOpenChange` → `handleBlur`** | Select-এ input-এর মতো `onBlur` নেই। Menu বন্ধ হওয়াকে "blur" ধরা হয় → তারপর error দেখানো শুরু |
| `id={field.name}` (trigger-এ) | `FieldLabel htmlFor` → label চাপলে Select-এ focus |
| `aria-invalid`, `aria-describedby` | screen reader error পড়ে (AppField-এর মতো) |

---

## ধাপ ৬: `doctorForm.utils.ts` (নতুন) — মান রূপান্তর

`src/components/modules/Admin/DoctorsManagement/doctorForm.utils.ts`

```ts
import type { IDoctor, IDoctorProfilePayload } from "@/types/doctor.types";
import type { DoctorProfileFormValues } from "@/zod/doctor.validation";

// Create form-এর শুরুর ফাঁকা মান (ভাগ 2b-তে লাগবে)
export const EMPTY_DOCTOR_PROFILE: DoctorProfileFormValues = {
	name: "",
	contactNumber: "",
	gender: "",
	designation: "",
	currentWorkingPlace: "",
	qualification: "",
	experience: "",
	appointmentFee: "",
	address: "",
	profilePhoto: "",
};

// table-এর row → form-এর string মান (null হলে "")
export const doctorToFormValues = (doctor: IDoctor): DoctorProfileFormValues => ({
	name: doctor.name,
	contactNumber: doctor.contactNumber ?? "",
	gender: doctor.gender,
	designation: doctor.designation,
	currentWorkingPlace: doctor.currentWorkingPlace,
	qualification: doctor.qualification,
	experience: String(doctor.experience),
	appointmentFee: String(doctor.appointmentFee),
	address: doctor.address ?? "",
	profilePhoto: doctor.profilePhoto ?? "",
});

// form-এর string → backend-এর আসল type ("" হলে পাঠাই না)
export const toDoctorProfilePayload = (values: DoctorProfileFormValues): IDoctorProfilePayload => {
	// schema আগেই "" আটকায়; এই লাইন শুধু TypeScript-কে নিশ্চিত করে
	if (values.gender === "") throw new Error("Gender is required");

	return {
		name: values.name.trim(),
		contactNumber: values.contactNumber.trim(),
		gender: values.gender,
		designation: values.designation.trim(),
		currentWorkingPlace: values.currentWorkingPlace.trim(),
		qualification: values.qualification.trim(),
		experience: Number(values.experience),
		appointmentFee: Number(values.appointmentFee),
		address: values.address.trim() || undefined,
		profilePhoto: values.profilePhoto.trim() || undefined,
	};
};

// শুধু যে field-গুলো বদলেছে (PATCH-এ সব পাঠানোর দরকার নেই)
export const getChangedFields = <T extends object>(next: T, previous: T): Partial<T> =>
	Object.fromEntries(
		Object.entries(next).filter(
			([key, value]) => value !== undefined && value !== previous[key as keyof T],
		),
	) as Partial<T>;
```

> ✏️ File-টা `.ts` (তোমার নোটের code block-এ `tsx` লেখা ছিল) — ভিতরে JSX নেই।

### তিন রকম মান, তিনটা function

```
IDoctor (table row)                  DoctorProfileFormValues (form)        IDoctorProfilePayload (backend)
experience: 5            ─ doctorToFormValues ─►  experience: "5"   ─ toDoctorProfilePayload ─►  experience: 5
address: null                                     address: ""                                    address: undefined
profilePhoto: "https://…"                         profilePhoto: "https://…"                      profilePhoto: "https://…"
```

| Function | কাজ |
|---|---|
| `EMPTY_DOCTOR_PROFILE` | Create form-এর শুরুর ফাঁকা মান (পরের নোটে) |
| `doctorToFormValues(doctor)` | row → form: সংখ্যা → `String()`, `null` → `""` |
| `toDoctorProfilePayload(values)` | form → backend: trim, `Number()`, `""` → `undefined` (পাঠাই না) |
| `getChangedFields(next, previous)` | শুধু যে key-এর মান আলাদা, সেগুলো |

- `if (values.gender === "") throw` → schema আগেই `""` আটকায়, তাই এটা কখনো চলবে না। কিন্তু এই লাইনের পরে TypeScript জানে `gender` হলো `"MALE" | "FEMALE"` (type narrowing) → return type মেলে।

### `getChangedFields` — ভেঙে বোঝা

```ts
Object.entries(next)                                   // [["name", "Dr. X"], ["experience", 6], ...]
	.filter(([key, value]) => value !== undefined && value !== previous[key])  // বদলেছে + ফাঁকা না
Object.fromEntries(...)                                // আবার object
```

```
previous: { name: "Dr. A", experience: 5, appointmentFee: 500 }
next:     { name: "Dr. A", experience: 6, appointmentFee: 500 }
changes:  { experience: 6 }   → PATCH body: { doctor: { experience: 6 } }
```
- Generic `<T extends object>` → যেকোনো object-এ চলে।
- `value !== undefined` → ফাঁকা optional field পাঠানো হয় না।
- `!==` শুধু সাধারণ মান (string, number) তুলনা করে — এখানে সব field তাই, ঠিক আছে।

---

## ধাপ ৭: `DoctorForm.tsx` (নতুন) — form UI

`src/components/modules/Admin/DoctorsManagement/DoctorForm.tsx`

এখন শুধু profile-এর field আছে। ভাগ 2b-তে Create-এর জন্য এখানেই account section (email, password, registration number, specialties) যোগ হবে।

```tsx
"use client";

import AppField from "@/components/shared/form/AppField";
import AppSelectField from "@/components/shared/form/AppSelectField";
import AppSubmitButton from "@/components/shared/form/AppSubmitButton";
import { Button } from "@/components/ui/button";
import { type DoctorProfileFormValues, doctorProfileFormSchema } from "@/zod/doctor.validation";
import { useForm } from "@tanstack/react-form";
import { GENDER_OPTIONS } from "./doctorsTable.config";

interface DoctorFormProps {
	defaultValues: DoctorProfileFormValues;
	onSubmit: (values: DoctorProfileFormValues) => void;
	onCancel: () => void;
	isPending: boolean;
	submitLabel: string;
	pendingLabel: string;
}

export default function DoctorForm({
	defaultValues,
	onSubmit,
	onCancel,
	isPending,
	submitLabel,
	pendingLabel,
}: DoctorFormProps) {
	const form = useForm({
		defaultValues,
		validators: { onChange: doctorProfileFormSchema },
		onSubmit: ({ value }) => onSubmit(value),
	});

	return (
		<form
			noValidate
			className="space-y-6"
			onSubmit={(event) => {
				event.preventDefault();
				event.stopPropagation();
				void form.handleSubmit();
			}}
		>
			{/* fieldset disabled: save চলার সময় সব input একসাথে বন্ধ */}
			<fieldset disabled={isPending} className="grid gap-4 sm:grid-cols-2">
				<legend className="sr-only">Doctor profile</legend>

				<div className="sm:col-span-2">
					<form.Field name="name">
						{(field) => <AppField field={field} label="Full name" placeholder="Dr. Jane Doe" autoComplete="name" />}
					</form.Field>
				</div>

				<form.Field name="contactNumber">
					{(field) => (
						<AppField field={field} label="Phone" type="tel" inputMode="tel" placeholder="01XXXXXXXXX" autoComplete="tel" />
					)}
				</form.Field>

				<form.Field name="gender">
					{(field) => (
						<AppSelectField field={field} label="Gender" options={GENDER_OPTIONS} placeholder="Select gender" />
					)}
				</form.Field>

				<form.Field name="designation">
					{(field) => <AppField field={field} label="Designation" placeholder="Senior Consultant" />}
				</form.Field>

				<form.Field name="currentWorkingPlace">
					{(field) => <AppField field={field} label="Working place" placeholder="Dhaka Medical College" />}
				</form.Field>

				<div className="sm:col-span-2">
					<form.Field name="qualification">
						{(field) => <AppField field={field} label="Qualification" placeholder="MBBS, FCPS (Medicine)" />}
					</form.Field>
				</div>

				<form.Field name="experience">
					{(field) => <AppField field={field} label="Experience (years)" type="number" inputMode="numeric" min={0} />}
				</form.Field>

				<form.Field name="appointmentFee">
					{(field) => <AppField field={field} label="Appointment fee (৳)" type="number" inputMode="decimal" min={0} />}
				</form.Field>

				<div className="sm:col-span-2">
					<form.Field name="address">
						{(field) => <AppField field={field} label="Address (optional)" autoComplete="street-address" />}
					</form.Field>
				</div>

				<div className="sm:col-span-2">
					<form.Field name="profilePhoto">
						{(field) => (
							<AppField field={field} label="Profile photo URL (optional)" type="url" inputMode="url" placeholder="https://..." />
						)}
					</form.Field>
				</div>
			</fieldset>

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

### Props — form নিজে কিছু জানে না

| Prop | Edit-এ | Create-এ (পরে) |
|---|---|---|
| `defaultValues` | row থেকে | `EMPTY_DOCTOR_PROFILE` |
| `onSubmit` | বদলানো field বের করে PATCH | পুরো payload POST |
| `submitLabel` / `pendingLabel` | "Save changes" / "Saving..." | "Create doctor" / "Creating..." |

Form শুধু UI + validation; **কী করবে সেটা বাইরে থেকে** → একটা form দুই কাজে।

### `useForm`
- `defaultValues` → শুরুর মান।
- `validators: { onChange: doctorProfileFormSchema }` → প্রতিটা পরিবর্তনে পুরো form Zod দিয়ে check (TanStack Form সরাসরি Zod schema নেয়)। Error দেখানো হয় শুধু blur/submit-এর পরে (`AppField`-এর নিয়ম)।
- `onSubmit` → validation পাস করলে তবেই চলে।

### `<form onSubmit>`

```tsx
event.preventDefault();      // page reload না
event.stopPropagation();     // dialog-এর ভিতরের form, বাইরের কোনো form-এ submit ছড়ায় না
void form.handleSubmit();    // Promise দেয়; "void" → ইচ্ছা করে await করছি না (lint-এর floating promise নিয়ম)
```
- `noValidate` → browser-এর নিজের validation popup (যেমন `type="url"`-এর) বন্ধ, শুধু আমাদের Zod-এর message।

### `<fieldset disabled={isPending}>`
- Save চলার সময় **ভিতরের সব input একসাথে disabled** — প্রতিটা field-এ আলাদা `disabled` লিখতে হয় না।
- `<legend className="sr-only">` → fieldset-এর নাম (screen reader)।
- `grid gap-4 sm:grid-cols-2` → বড় screen-এ ২ column; `sm:col-span-2` দিয়ে লম্বা field (নাম, qualification, address, photo) পুরো চওড়া।

### Input-এর ছোট ছোট সুবিধা

| Field | Prop | লাভ |
|---|---|---|
| Name | `autoComplete="name"` | browser autofill |
| Phone | `type="tel"` `inputMode="tel"` | mobile-এ সংখ্যার keyboard |
| Experience | `type="number"` `inputMode="numeric"` `min={0}` | পূর্ণ সংখ্যার keyboard |
| Fee | `inputMode="decimal"` | দশমিক বিন্দুসহ keyboard |
| Address | `autoComplete="street-address"` | autofill |
| Photo | `type="url"` `inputMode="url"` | `.com`, `/` সহ keyboard |

- `AppField`-এ বাকি সব prop সরাসরি `<Input>`-এ যায় (`...inputProps`), তাই এগুলো দেওয়া যায়।
- `AppSubmitButton className="w-auto"` → login form-এ পুরো চওড়া ছিল (`w-full`), এখানে ছোট।

---

## ধাপ ৮: `EditDoctorDialog.tsx` (নতুন)

`src/components/modules/Admin/DoctorsManagement/EditDoctorDialog.tsx`

```tsx
"use client";

import { updateDoctorAction } from "@/actions/doctor.actions";
import { Dialog, DialogContent, DialogDescription, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { useActionMutation } from "@/hooks/useActionMutation";
import { DOCTORS_QUERY_KEY } from "@/queries/doctor.queries";
import type { IDoctor, IUpdateDoctorPayload } from "@/types/doctor.types";
import type { DoctorProfileFormValues } from "@/zod/doctor.validation";
import { useMemo } from "react";
import { toast } from "sonner";
import DoctorForm from "./DoctorForm";
import { doctorToFormValues, getChangedFields, toDoctorProfilePayload } from "./doctorForm.utils";

interface EditDoctorDialogProps {
	row: IDoctor | null;
	open: boolean;
	onOpenChange: (open: boolean) => void;
}

export default function EditDoctorDialog({ row: doctor, open, onOpenChange }: EditDoctorDialogProps) {
	const { mutate, isPending } = useActionMutation({
		action: ({ id, payload }: { id: string; payload: IUpdateDoctorPayload }) => updateDoctorAction(id, payload),
		invalidateKey: DOCTORS_QUERY_KEY,
		successMessage: (updated) => `${updated.name}'s profile has been updated`,
		onSuccess: () => onOpenChange(false),
	});

	// useMemo: প্রতি render-এ নতুন object দিলে form নিজেকে reset করতে পারে
	const defaultValues = useMemo(() => (doctor ? doctorToFormValues(doctor) : null), [doctor]);

	const handleSubmit = (values: DoctorProfileFormValues) => {
		if (!doctor || !defaultValues) return;

		// দুই দিকই একই নিয়মে বদলে তুলনা করা, তাই null বনাম "" বা "500" বনাম 500 ভুলভাবে "বদলেছে" ধরে না
		const changes = getChangedFields(toDoctorProfilePayload(values), toDoctorProfilePayload(defaultValues));

		if (Object.keys(changes).length === 0) {
			toast.info("No changes to save");
			return;
		}

		mutate({ id: doctor.id, payload: { doctor: changes } });
	};

	return (
		// save চলার সময় Esc বা বাইরে click করে বন্ধ করা যাবে না
		<Dialog open={open} onOpenChange={(next) => !isPending && onOpenChange(next)}>
			<DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-2xl">
				<DialogHeader>
					<DialogTitle>Edit doctor</DialogTitle>
					<DialogDescription>
						Update {doctor?.name}&apos;s profile. Email, registration number and specialties can&apos;t be changed here.
					</DialogDescription>
				</DialogHeader>

				{doctor && defaultValues && (
					// key: অন্য doctor খুললে form একদম নতুন করে শুরু হয়
					<DoctorForm
						key={doctor.id}
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

### `useActionMutation` (note 18-এর hook)
- `action` → `updateDoctorAction` দুটো argument নেয়, কিন্তু `mutate` একটাই নেয় → `{ id, payload }` object বানিয়ে ভিতরে ভাগ করা।
- `successMessage: (updated) => ...` → backend-এর ফেরত **নতুন** doctor-এর নাম (নাম বদলালেও নতুনটা দেখায়)।
- সফল হলে `["doctors"]` invalidate → table-এ নতুন মান → dialog বন্ধ → toast।

### `defaultValues` + `useMemo`
- Row থেকে form-এর মান একবার বানানো। **প্রতি render-এ নতুন object** দিলে TanStack Form ভাবতে পারে "default বদলে গেছে" → user-এর টাইপ করা মান reset হয়ে যেতে পারে। `useMemo([doctor])` → doctor একই থাকলে একই object।

### `handleSubmit` — "কিছু বদলায়নি?" ⭐

```ts
const changes = getChangedFields(toDoctorProfilePayload(values), toDoctorProfilePayload(defaultValues));
```
**দুই দিকই একই function দিয়ে বদলে তারপর তুলনা।** সরাসরি তুলনা করলে:

| ভুল তুলনা | কেন ভুল |
|---|---|
| form `"500"` বনাম row `500` | string বনাম number → "বদলেছে" ❌ |
| form `""` বনাম row `null` (address) | "বদলেছে" ❌ |
| form `" Dr. A "` বনাম `"Dr. A"` | space → "বদলেছে" ❌ |

দুই দিকই `toDoctorProfilePayload` → একই রূপ (`500`, `undefined`, trim) → শুধু আসল পরিবর্তন ধরা পড়ে।

- কিছুই না বদলালে → `toast.info("No changes to save")`, **request যায় না**, dialog খোলা থাকে।
- বদলালে → `mutate({ id, payload: { doctor: changes } })`।

### Dialog
- `onOpenChange={(next) => !isPending && onOpenChange(next)}` → save চলার সময় Esc/বাইরে click-এ বন্ধ হয় না (Delete dialog-এর মতো)।
- `sm:max-w-2xl` → form চওড়া, ২ column ধরে।
- Description-এ জানানো: email, registration number, specialty এখানে বদলানো যায় না (backend-এর update schema-তেও নেই)।
- **`key={doctor.id}`** → React-এ `key` বদলালে component একদম নতুন করে তৈরি হয়। Doctor A-এর form-এ কিছু টাইপ করে বন্ধ করে Doctor B খুললে A-এর মান থেকে যায় না।

---

## ধাপ ৯: `DoctorsTable.tsx` (পুরো file: View + Delete + Edit)

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
import { useEffect, useMemo } from "react";
import DeleteDoctorDialog from "./DeleteDoctorDialog";
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
import EditDoctorDialog from "./EditDoctorDialog";
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
		placeholderData: keepPreviousData, // নতুন data আসা পর্যন্ত পুরনোটা দেখায়
	});

	const { data: specialtiesResponse } = useQuery(specialtiesQueryOptions());

	const specialtyOptions = useMemo(
		() => (specialtiesResponse?.data ?? []).map((specialty) => ({ label: specialty.title, value: specialty.title })),
		[specialtiesResponse],
	);

	// View + Edit + Delete (actions stable, তাই columns বারবার নতুন হয় না)
	const tableActions = rowAction.actions;

	// শেষ page-এর শেষ doctor delete হলে ফাঁকা page-এ আটকে না থেকে আগের page-এ যাওয়া
	const meta = response?.meta;
	const { setPage } = table;
	useEffect(() => {
		if (meta && meta.totalPages > 0 && meta.page > meta.totalPages) setPage(meta.totalPages);
	}, [meta, setPage]);

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

			{/* Row action dialogs: একবারে একটাই খোলে */}
			<ViewDoctorDialog {...rowAction.dialogProps("view")} />
			<EditDoctorDialog {...rowAction.dialogProps("edit")} />
			<DeleteDoctorDialog {...rowAction.dialogProps("delete")} />
		</div>
	);
}
```

নোট 18 থেকে যা বদলেছে:
- **`const tableActions = rowAction.actions;`** → এখন তিনটা action-ই লাগে, তাই আলাদা `useMemo` দিয়ে বাছাই করার দরকার নেই। `rowAction.actions` নিজেই stable (note 18, `useMemo([])`)।
- `<EditDoctorDialog {...rowAction.dialogProps("edit")} />` যোগ।
- `import EditDoctorDialog from "./EditDoctorDialog";`
- "⋯" menu-তে এখন: View, Edit, (separator), Delete।

---

## 📁 এই নোটের file

```
src/
├── types/doctor.types.ts                          ← + IDoctorProfilePayload, IUpdateDoctorPayload
├── zod/doctor.validation.ts                       ← নতুন: form + server schema
├── services/doctor.services.ts                    ← + updateDoctor
├── actions/doctor.actions.ts                      ← + updateDoctorAction
├── components/shared/form/AppSelectField.tsx      ← নতুন
└── components/modules/Admin/DoctorsManagement/
    ├── doctorForm.utils.ts                        ← নতুন
    ├── DoctorForm.tsx                             ← নতুন (create-এও লাগবে)
    ├── EditDoctorDialog.tsx                       ← নতুন
    └── DoctorsTable.tsx                           ← + Edit
```

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | "⋯" → Edit | dialog, সব field doctor-এর মান দিয়ে ভরা |
| ২ | কিছু না বদলে Save | "No changes to save" (info toast), Network-এ request নেই |
| ৩ | Experience 5 → 6 → Save | "Saving…" → table-এ 6 → dialog বন্ধ → সবুজ toast |
| ৪ | Network → Server Action-এর payload | শুধু `{ doctor: { experience: 6 } }` |
| ৫ | Name মুছে বাইরে click | "Name is required" |
| ৬ | Name "ab" | "Name must be at least 3 characters" |
| ৭ | Phone "12345" | "Enter a valid Bangladeshi phone number…" |
| ৮ | Fee "500.555" | "Fee must be a positive number" |
| ৯ | Photo "abc" | "Must be a valid http/https URL" |
| ১০ | Gender Select খুলে কিছু না বেছে বন্ধ | (ফাঁকা থাকলে) error দেখায় |
| ১১ | Save চলার সময় | সব input disabled, Esc-এ বন্ধ হয় না |
| ১২ | Doctor A-এ টাইপ করে Cancel → Doctor B Edit | B-এর নিজের মান, A-এর লেখা নেই |
| ১৩ | Backend বন্ধ রেখে Save | লাল toast, dialog খোলা, মান থাকে |

---

## 🔜 পরে যা যোগ হবে
- [x] **Create doctor** — note 20 (ভাগ 2b) — `DoctorForm`-এ account section (email, password, registration number, specialties) + `CreateDoctorDialog` + `EMPTY_DOCTOR_PROFILE`
- [ ] Specialty বদলানো (আলাদা endpoint)
- [ ] Profile photo upload (এখন শুধু URL)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ সংশোধিত

1. ~~Phone ফাঁকা দেখাবে, আর save করা যাবে না~~
> ✏️ সংশোধন: আগে এখানে লেখা ছিল যে list API এই field পাঠায় না — সেটা **ভুল** ছিল। Backend-এর `QueryBuilder`-এ `doctorSelectableFields` শুধু তখনই কাজ করে যখন URL-এ `?fields=...` দেওয়া হয়; আমরা `fields` পাঠাই না, তাই `GET /doctors` পুরো doctor (`contactNumber`, `registrationNumber` সহ) + `user` + `specialties` পাঠায়। তাই Edit form-এ phone doctor-এর আসল নম্বর দিয়েই ভরা থাকে।

### 🟡 ভালো practice

2. **পুরনো ছোট address থাকলে edit আটকে যাবে** — form-এ address ফাঁকা অথবা ১০–১০০ অক্ষর। কোনো doctor-এর DB-তে ১০ অক্ষরের কম address থাকলে (যেমন seed data-য় "Dhaka"), address না ছুঁয়েও save করা যাবে না। চাইলে edit-এ শুধু **বদলানো** field-কে কড়া নিয়মে check করো, অথবা DB-র পুরনো data ঠিক করে নাও।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Optional field মুছে ফাঁকা করা যায় না** — address মুছে Save করলে `""` → `undefined` → `getChangedFields` বাদ দেয় → backend-এ যায়ই না → পুরনো address থেকে যায়। এখন ঠিক আছে (backend-এর schema-ও `null` নেয় না)। পরে "address মুছে দাও" দরকার হলে backend-এ `.nullable()` আর frontend-এ `""` → `null` পাঠাতে হবে।

4. ✅ (Done — note 20) **`DoctorForm` create-এ ব্যবহার করার সময়** — Create-এ পুরো payload পাঠাবে (`getChangedFields` লাগবে না), আর server-এ `doctorProfilePayloadSchema` (partial না) দিয়ে validate করবে।
