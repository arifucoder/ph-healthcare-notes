# 20 - Create Doctor (একই DoctorForm দিয়ে Create + Edit)

> Doctor table-এর toolbar-এ **"Add doctor"** button → dialog-এ form → নতুন doctor account (email + temporary password + registration no. + specialty + profile)। Note 19-এর `DoctorForm`-কে এবার **`mode: "create" | "edit"`** দিয়ে দুই কাজের উপযোগী করা হলো — একটা form, দুটো dialog।

## 🧭 এক নজরে

```
"Add doctor" চাপা → CreateDoctorDialog → DoctorForm mode="create" (ফাঁকা)
  → টাইপ করার সময়: createDoctorFormSchema (account + profile, string-ভিত্তিক)
  → Submit: toCreateDoctorPayload → { password, doctor: {...}, specialties: [id] }
  → createDoctorAction: createDoctorPayloadSchema দিয়ে আবার যাচাই → POST /user/create-doctor
  → সফল: ["doctors"] invalidate → dialog বন্ধ (form unmount) → toast "Doctor account created"
  → ব্যর্থ (যেমন email আগে থেকে আছে): লাল toast, form-এর মান থাকে
```

### Create বনাম Edit — এক নজরে

| | Create (এই নোট) | Edit (note 19) |
|---|---|---|
| শুরুর মান | `EMPTY_DOCTOR_FORM` | `doctorToFormValues(row)` |
| Account section (email, password, reg. no., specialties) | ✅ দেখায় + যাচাই | ❌ লুকানো, যাচাই নেই |
| Profile photo | ❌ (backend create-এ নেয় না) | ✅ |
| Schema | `createDoctorFormSchema` | `editDoctorFormSchema` |
| পাঠায় | পুরো payload, `POST /user/create-doctor` | শুধু বদলানো field, `PATCH /doctors/:id` |
| খোলে | toolbar-এর button (`DialogTrigger`) | row-এর "⋯" (`useRowAction`) |

> ✏️ নোট 19-এর code থেকে কয়েকটা নাম বদলেছে: `DoctorProfileFormValues` → **`DoctorFormValues`**, `doctorProfileFormSchema` → **`createDoctorFormSchema` / `editDoctorFormSchema`**, `EMPTY_DOCTOR_PROFILE` → **`EMPTY_DOCTOR_FORM`**। নিচের file-গুলো পুরো দেওয়া আছে, সেগুলোই চূড়ান্ত।

---

## ধাপ ১: `src/types/doctor.types.ts` (পুরো file)

```ts
// backend prisma enum Gender-এর সাথে মিল (OTHER নেই)
export type Gender = "MALE" | "FEMALE";

// backend doctorInclude (user + specialties) অনুযায়ী
export interface IDoctor {
	id: string;
	name: string;
	email: string;
	profilePhoto?: string | null;
	contactNumber?: string | null;
	registrationNumber: string;
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

// create আর edit দুজনেরই কমন profile field (backend schema অনুযায়ী)
export interface IDoctorProfilePayload {
	name: string;
	contactNumber: string;
	address?: string;
	profilePhoto?: string; // শুধু edit-এ (backend create-এ নেয় না)
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

// backend POST /user/create-doctor
export interface ICreateDoctorPayload {
	password: string;
	doctor: Omit<IDoctorProfilePayload, "profilePhoto"> & {
		email: string;
		registrationNumber: string;
	};
	specialties: string[]; // specialty id (uuid)
}
```

| Type | কাজ |
|---|---|
| `IDoctor` | table row — backend `getAllDoctors`-এর `include` (`user` + `specialties`) অনুযায়ী পুরো doctor, তাই `contactNumber`, `registrationNumber` আসে |
| `IDoctorProfilePayload` | create + edit-এর কমন profile; `profilePhoto` শুধু edit-এ |
| `IUpdateDoctorPayload` | `{ doctor: Partial<...> }` (note 19) |
| **`ICreateDoctorPayload`** (নতুন) | backend `POST /user/create-doctor`-এর body |

### `ICreateDoctorPayload` ভেঙে বোঝা

```ts
{
	password: string;
	doctor: Omit<IDoctorProfilePayload, "profilePhoto"> & { email: string; registrationNumber: string };
	specialties: string[];
}
```
- **`Omit<T, "profilePhoto">`** → profile-এর সব field, শুধু `profilePhoto` বাদ।
- **`& { ... }`** (intersection) → তার সাথে `email` আর `registrationNumber` যোগ।
- `specialties` → specialty-র **id** (uuid), title না।
- `password` আলাদা, `doctor`-এর বাইরে → backend আগে Better Auth দিয়ে user account বানায় (password), তারপর Doctor row (server note 10)।

---

## ধাপ ২: `src/zod/doctor.validation.ts` (পুরো file)

```ts
import * as z from "zod";

const BD_PHONE_REGEX = /^(\+?880|0)1[3-9]\d{8}$/;

// =============== Form schema (browser) ===============
// input field string দেয়, তাই মানগুলো string হিসেবে যাচাই হয়। optional field-এ "" = দেওয়া হয়নি।

const requiredText = (label: string, min: number, max: number) =>
	z
		.string()
		.trim()
		.min(1, `${label} is required`)
		.min(min, `${label} must be at least ${min} characters`)
		.max(max, `${label} must be at most ${max} characters`);

// create আর edit দুজনেরই কমন অংশ
const doctorProfileFormShape = {
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
	profilePhoto: z.union([z.literal(""), z.url({ protocol: /^https?$/, error: "Must be a valid http/https URL" })]),
};

// Create: account-এর field-গুলোও যাচাই হয় (backend createDoctorValidationSchema অনুযায়ী)
export const createDoctorFormSchema = z.object({
	...doctorProfileFormShape,
	email: z.string().trim().min(1, "Email is required").pipe(z.email("Invalid email address")),
	password: z
		.string()
		.min(6, "Password must be at least 6 characters")
		.max(32, "Password must be at most 32 characters"),
	registrationNumber: z.string().trim().min(1, "Registration number is required").max(50, "Too long"),
	specialties: z.array(z.string()).min(1, "Select at least one specialty"),
});

// Edit: account-এর field form-এ থাকে (একই form, একই type), কিন্তু বদলানো যায় না, তাই যাচাইও নেই
export const editDoctorFormSchema = z.object({
	...doctorProfileFormShape,
	email: z.string(),
	password: z.string(),
	registrationNumber: z.string(),
	specialties: z.array(z.string()),
});

// দুই schema-র input type একই, তাই একটাই form type
export type DoctorFormValues = z.input<typeof createDoctorFormSchema>;

// =============== Server schema (server action) ===============
// Server Action public endpoint, তাই আসল type আবার যাচাই হয়।
// z.object অজানা key বাদ দেয়: কেউ averageRating বা isDeleted পাঠালেও backend-এ যাবে না।

const doctorProfilePayloadSchema = z.object({
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

export const createDoctorPayloadSchema = z.object({
	password: z.string().min(6, "Password must be at least 6 characters").max(32, "Password must be at most 32 characters"),
	doctor: doctorProfilePayloadSchema.omit({ profilePhoto: true }).extend({
		email: z.email("Invalid email address").toLowerCase(),
		registrationNumber: z.string().trim().min(1, "Registration number is required"),
	}),
	specialties: z
		.array(z.uuid("Invalid specialty"))
		.min(1, "Select at least one specialty")
		.refine((ids) => new Set(ids).size === ids.length, "Duplicate specialties are not allowed"),
});
```

### Form schema — `doctorProfileFormShape` ভাগাভাগি

```ts
const doctorProfileFormShape = { name: ..., contactNumber: ..., ... };   // শুধু object, z.object না

createDoctorFormSchema = z.object({ ...doctorProfileFormShape, email, password, registrationNumber, specialties });
editDoctorFormSchema   = z.object({ ...doctorProfileFormShape, email: z.string(), password: z.string(), ... });
```
- Profile-এর নিয়ম একবার লিখে দুই schema-তে spread (`...`)।
- **Edit schema-তেও account field আছে**, কিন্তু শুধু `z.string()` (কোনো নিয়ম নেই)। কারণ:
  - একটাই form component, একটাই মানের type (`DoctorFormValues`) — edit form-এও email, password key থাকে (লুকানো)।
  - Edit-এ এগুলো বদলানো যায় না, password ফাঁকা (`""`) — নিয়ম দিলে "Password must be at least 6" error-এ edit আটকে যেত।
- `DoctorFormValues = z.input<typeof createDoctorFormSchema>` → দুই schema-র input type একই, তাই একটা type।

### Create-এর account field

| Field | নিয়ম | কেন |
|---|---|---|
| `email` | `.trim().min(1, "required").pipe(z.email(...))` | **`pipe`** → আগে "ফাঁকা কিনা" check, পাস করলে তবেই email format। ফাঁকা হলে "Email is required" (শুধু "Invalid email" না) |
| `password` | ৬–৩২ | backend-এর `createDoctorValidationSchema`-এর মতো। `trim` নেই — password-এ space ইচ্ছাকৃত হতে পারে |
| `registrationNumber` | ১–৫০ | |
| `specialties` | `z.array(z.string()).min(1)` | অন্তত একটা specialty |

### Server schema — `createDoctorPayloadSchema`

```ts
doctor: doctorProfilePayloadSchema.omit({ profilePhoto: true }).extend({ email, registrationNumber })
```
- **`.omit()`** → schema থেকে field বাদ; **`.extend()`** → নতুন field যোগ। (TypeScript-এর `Omit` + `&`-এর Zod version।)
- `z.email().toLowerCase()` → `Doctor@Mail.com` → `doctor@mail.com` (একই email দুবার আলাদা account না হয়)।
- `specialties` → `z.uuid()` (প্রতিটা আসল id কিনা) + `.min(1)` + **duplicate check**: `new Set(ids).size === ids.length` (Set-এ duplicate থাকে না, তাই size কম হলে duplicate ছিল)।
- `doctorProfilePayloadSchema` এখন export করা হয় না (শুধু এই file-এর ভিতরে লাগে)।

---

## ধাপ ৩: `src/services/doctor.services.ts` (পুরো file)

```ts
"use server";

import { httpClient } from "@/lib/axios/httpClient";
import type { ICreateDoctorPayload, IDoctor, IUpdateDoctorPayload } from "@/types/doctor.types";

// service: শুধু backend-এ request পাঠায়। validation আর error handling action-এ হয়।

export const getDoctors = async (queryString: string) =>
	httpClient.get<IDoctor[]>(queryString ? `/doctors?${queryString}` : "/doctors");

// backend route: /user (s ছাড়া), teacher-এর /users না
export const createDoctor = async (payload: ICreateDoctorPayload) =>
	httpClient.post<unknown>("/user/create-doctor", payload);

export const updateDoctor = async (id: string, payload: IUpdateDoctorPayload) =>
	httpClient.patch<IDoctor>(`/doctors/${encodeURIComponent(id)}`, payload);

export const deleteDoctor = async (id: string) => httpClient.delete<IDoctor>(`/doctors/${encodeURIComponent(id)}`);
```

- Backend route **`/user/create-doctor`** (`s` ছাড়া — teacher-এর `/users` না)। `checkAuth(ADMIN, SUPER_ADMIN)` + `validateRequest(createDoctorValidationSchema)`।
- `httpClient.post<unknown>` → ফেরত data-র shape আমাদের দরকার নেই (শুধু "সফল হলো কিনা"), তাই `unknown`।
- এক file-এ চারটা: get, create, update, delete।

---

## ধাপ ৪: `src/actions/doctor.actions.ts` (পুরো file)

```ts
"use server";

import { getActionErrorMessage } from "@/lib/getActionErrorMessage";
import { createDoctor, deleteDoctor, updateDoctor } from "@/services/doctor.services";
import type { ActionResult } from "@/types/api.types";
import type { IDoctor } from "@/types/doctor.types";
import { createDoctorPayloadSchema, updateDoctorPayloadSchema } from "@/zod/doctor.validation";
import * as z from "zod";

// "use server" file-এর প্রতিটা export একটা public endpoint:
// যে কেউ যেকোনো মান পাঠাতে পারে, তাই input-কে unknown ধরে validate করি।
// কে create, update বা delete করতে পারবে, সেটা backend-এর checkAuth (ADMIN/SUPER_ADMIN) ঠিক করে।
const doctorIdSchema = z.uuid("Invalid doctor id");

const firstIssue = (error: z.ZodError) => error.issues[0]?.message ?? "Invalid input";

export async function createDoctorAction(payload: unknown): Promise<ActionResult<unknown>> {
	const parsed = createDoctorPayloadSchema.safeParse(payload);
	if (!parsed.success) {
		return { success: false, message: firstIssue(parsed.error) };
	}

	try {
		return await createDoctor(parsed.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "createDoctorAction") };
	}
}

export async function updateDoctorAction(id: unknown, payload: unknown): Promise<ActionResult<IDoctor>> {
	const parsedId = doctorIdSchema.safeParse(id);
	if (!parsedId.success) {
		return { success: false, message: "Invalid doctor id" };
	}

	const parsedPayload = updateDoctorPayloadSchema.safeParse(payload);
	if (!parsedPayload.success) {
		return { success: false, message: firstIssue(parsedPayload.error) };
	}

	try {
		return await updateDoctor(parsedId.data, parsedPayload.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "updateDoctorAction") };
	}
}

export async function deleteDoctorAction(id: unknown): Promise<ActionResult<IDoctor>> {
	const parsedId = doctorIdSchema.safeParse(id);
	if (!parsedId.success) {
		return { success: false, message: firstIssue(parsedId.error) };
	}

	try {
		return await deleteDoctor(parsedId.data);
	} catch (error) {
		return { success: false, message: getActionErrorMessage(error, "deleteDoctorAction") };
	}
}
```

- **`firstIssue(error)`** → তিনটা action-এ একই "প্রথম Zod error-এর message" লাইন বারবার না লিখে helper।
- তিনটা action একই নিয়মে: `unknown` → `safeParse` → ভুল হলে `{ success: false, message }` → service → error হলে `getActionErrorMessage` (note 18)।
- Backend-এ email বা registration number আগে থেকে থাকলে backend-এর message (যেমন "Email already exists") toast-এ আসে।

---

## ধাপ ৫: `src/components/shared/form/AppMultiSelectField.tsx` (নতুন)

Specialty বাছাইয়ের জন্য। প্রতিটা specialty একটা toggle বাটন। নতুন কোনো shadcn component install করতে হয় না।

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { FieldError, FieldLegend, FieldSet } from "@/components/ui/field";
import type { AnyFieldApi } from "@tanstack/react-form";
import { Check } from "lucide-react";

interface AppMultiSelectFieldProps {
	field: AnyFieldApi; // value: string[]
	label: string;
	options: Array<{ label: string; value: string }>;
	emptyText?: string;
}

export default function AppMultiSelectField({
	field,
	label,
	options,
	emptyText = "No options available",
}: AppMultiSelectFieldProps) {
	const selected: string[] = field.state.value ?? [];
	const { meta } = field.state;
	const showError = !meta.isValid && (meta.isTouched || field.form.state.submissionAttempts > 0);
	const errorId = `${field.name}-error`;

	const toggle = (value: string) => {
		const next = selected.includes(value) ? selected.filter((item) => item !== value) : [...selected, value];
		field.handleChange(next);
		field.handleBlur();
	};

	return (
		// fieldset + legend: পুরো বাটনের দলটার একটাই নাম (screen reader-এর জন্য)
		<FieldSet data-invalid={showError}>
			<FieldLegend className="text-sm font-medium">{label}</FieldLegend>

			{options.length === 0 ? (
				<p className="text-sm text-muted-foreground">{emptyText}</p>
			) : (
				<div className="flex flex-wrap gap-2" aria-describedby={showError ? errorId : undefined}>
					{options.map((option) => {
						const isSelected = selected.includes(option.value);

						return (
							<Button
								key={option.value}
								type="button"
								size="sm"
								variant={isSelected ? "default" : "outline"}
								aria-pressed={isSelected} // toggle বাটন: বাছাই করা কিনা screen reader বলে দেয়
								onClick={() => toggle(option.value)}
							>
								{isSelected && <Check />}
								{option.label}
							</Button>
						);
					})}
				</div>
			)}

			{showError && <FieldError id={errorId} errors={meta.errors} />}
		</FieldSet>
	);
}
```

```
Specialties
[✓ Cardiology]  [ Neurology ]  [✓ Pediatrics]  [ Dermatology ]
 (default রং)    (outline)      (default রং)     (outline)
```

| অংশ | কাজ |
|---|---|
| `field.state.value` | `string[]` (বাছাই করা id) |
| `toggle(value)` | থাকলে বাদ (`filter`), না থাকলে যোগ (`[...selected, value]`) → নতুন array → `handleChange` |
| `handleBlur()` toggle-এর পরে | button-এ blur-এর ধারণা নেই, তাই প্রথম click-কেই "ছুঁয়েছে" ধরা |
| `showError`-এ `isTouched` (`isBlurred` না) | `handleChange` হলে `isTouched` true → সব বাদ দিলে সাথে সাথে "Select at least one specialty" |
| `variant` default / outline | বাছাই করা ভরাট রঙে, বাকিগুলো শুধু border |
| `aria-pressed` | toggle button — screen reader "pressed / not pressed" বলে |
| `<Check />` | বাছাই করা হলে ✓ — শুধু রং দিয়ে না, চিহ্ন দিয়েও বোঝা যায় |
| `type="button"` | form-এর ভিতরে button-এর default `submit` — না দিলে প্রতি click-এ form submit হত! |
| `FieldSet` + `FieldLegend` | অনেকগুলো button-এর একটাই নাম ("Specialties") — screen reader-এর জন্য (input না, তাই `<label>` চলে না) |
| `options.length === 0` | specialty নেই → "No specialties yet. Create a specialty first." |

**কেন dropdown না, toggle button?** Specialty সাধারণত কম (১০–২০টা), সবগুলো একবারে চোখে পড়ে, এক click-এ বাছাই, নতুন component install লাগে না।

---

## ধাপ ৬: `doctorForm.utils.ts` (পুরো file)

`src/components/modules/Admin/DoctorsManagement/doctorForm.utils.ts`

```ts
import type { ICreateDoctorPayload, IDoctor, IDoctorProfilePayload } from "@/types/doctor.types";
import type { DoctorFormValues } from "@/zod/doctor.validation";

// Create form-এর শুরুর ফাঁকা মান
export const EMPTY_DOCTOR_FORM: DoctorFormValues = {
	email: "",
	password: "",
	registrationNumber: "",
	specialties: [],
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
export const doctorToFormValues = (doctor: IDoctor): DoctorFormValues => ({
	email: doctor.email,
	password: "",
	registrationNumber: doctor.registrationNumber,
	specialties: doctor.specialties.map(({ specialty }) => specialty.id),
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
export const toDoctorProfilePayload = (values: DoctorFormValues): IDoctorProfilePayload => {
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

// Create: backend-এর { password, doctor, specialties } আকারে
export const toCreateDoctorPayload = (values: DoctorFormValues): ICreateDoctorPayload => {
	// backend create-এ profilePhoto নেয় না, তাই বাদ
	const { profilePhoto: _profilePhoto, ...profile } = toDoctorProfilePayload(values);

	return {
		password: values.password,
		doctor: {
			...profile,
			email: values.email.trim().toLowerCase(),
			registrationNumber: values.registrationNumber.trim(),
		},
		specialties: values.specialties,
	};
};

// Edit: শুধু যে field-গুলো বদলেছে (PATCH-এ সব পাঠানোর দরকার নেই)
export const getChangedFields = <T extends object>(next: T, previous: T): Partial<T> =>
	Object.fromEntries(
		Object.entries(next).filter(([key, value]) => value !== undefined && value !== previous[key as keyof T]),
	) as Partial<T>;
```

> ✏️ File-টা `.ts` (নোটের code block-এ `tsx` লেখা ছিল)।

### নতুন / বদলানো

| Function | কী বদলেছে |
|---|---|
| `EMPTY_DOCTOR_FORM` | এখন account field-ও (`email: ""`, `password: ""`, `specialties: []`, …) |
| `doctorToFormValues` | account field-ও ভরে (edit-এ লুকানো থাকে, schema check করে না) — `password: ""`, specialty-র **id** |
| `toDoctorProfilePayload` | আগের মতোই |
| **`toCreateDoctorPayload`** (নতুন) | form → backend-এর create shape |
| `getChangedFields` | আগের মতোই (শুধু edit) |

### `toCreateDoctorPayload`

```ts
const { profilePhoto: _profilePhoto, ...profile } = toDoctorProfilePayload(values);
```
- **Rest destructuring:** `profilePhoto` বের করে নিয়ে বাকি সব `profile`-এ → `profile`-এ photo নেই।
- `profilePhoto: _profilePhoto` → বের করা মানটার নাম বদলে `_profilePhoto` (`_` = "ইচ্ছা করে ব্যবহার করছি না")।
- Profile-এর রূপান্তর (`Number()`, trim, `""` → `undefined`) আবার না লিখে `toDoctorProfilePayload` reuse।
- `email.trim().toLowerCase()` → server schema-ও lowercase করে, এখানেও করা।
- `password` trim করা হয় না।

```
Form values                         → ICreateDoctorPayload
{ email: " Dr@Mail.com ",             { password: "secret12",
  password: "secret12",                 doctor: { name, contactNumber, gender, ..., email: "dr@mail.com", registrationNumber: "BMDC-1" },
  specialties: ["uuid-1"], ... }        specialties: ["uuid-1"] }
```

---

## ধাপ ৭: `DoctorForm.tsx` (পুরো file) — create + edit

`src/components/modules/Admin/DoctorsManagement/DoctorForm.tsx`

```tsx
"use client";

import AppField from "@/components/shared/form/AppField";
import AppMultiSelectField from "@/components/shared/form/AppMultiSelectField";
import AppSelectField from "@/components/shared/form/AppSelectField";
import AppSubmitButton from "@/components/shared/form/AppSubmitButton";
import { Button } from "@/components/ui/button";
import { Separator } from "@/components/ui/separator";
import { specialtiesQueryOptions } from "@/queries/specialty.queries";
import { createDoctorFormSchema, type DoctorFormValues, editDoctorFormSchema } from "@/zod/doctor.validation";
import { useForm } from "@tanstack/react-form";
import { useQuery } from "@tanstack/react-query";
import { Eye, EyeOff } from "lucide-react";
import { useMemo, useState } from "react";
import { GENDER_OPTIONS } from "./doctorsTable.config";

interface DoctorFormProps {
	mode: "create" | "edit";
	defaultValues: DoctorFormValues;
	onSubmit: (values: DoctorFormValues) => void;
	onCancel: () => void;
	isPending: boolean;
	submitLabel: string;
	pendingLabel: string;
}

export default function DoctorForm({
	mode,
	defaultValues,
	onSubmit,
	onCancel,
	isPending,
	submitLabel,
	pendingLabel,
}: DoctorFormProps) {
	const isCreate = mode === "create";
	const [showPassword, setShowPassword] = useState(false);

	// table page-এ আগেই cache-এ আছে, তাই নতুন request যায় না
	const { data: specialtiesResponse } = useQuery({ ...specialtiesQueryOptions(), enabled: isCreate });
	const specialtyOptions = useMemo(
		() => (specialtiesResponse?.data ?? []).map((specialty) => ({ label: specialty.title, value: specialty.id })),
		[specialtiesResponse],
	);

	const form = useForm({
		defaultValues,
		validators: { onChange: isCreate ? createDoctorFormSchema : editDoctorFormSchema },
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
			<fieldset disabled={isPending} className="space-y-6">
				{/* ---------- Account (শুধু create) ---------- */}
				{isCreate && (
					<>
						<div className="grid gap-4 sm:grid-cols-2">
							<div className="sm:col-span-2">
								<form.Field name="email">
									{(field) => (
										<AppField
											field={field}
											label="Email"
											type="email"
											inputMode="email"
											autoComplete="off"
											autoCapitalize="none"
											spellCheck={false}
											placeholder="doctor@example.com"
										/>
									)}
								</form.Field>
							</div>

							<form.Field name="password">
								{(field) => (
									<AppField
										field={field}
										label="Temporary password"
										type={showPassword ? "text" : "password"}
										autoComplete="new-password"
										endAdornment={
											<Button
												type="button"
												variant="ghost"
												size="icon-sm"
												onClick={() => setShowPassword((prev) => !prev)}
												aria-label={showPassword ? "Hide password" : "Show password"}
											>
												{showPassword ? <EyeOff /> : <Eye />}
											</Button>
										}
									/>
								)}
							</form.Field>

							<form.Field name="registrationNumber">
								{(field) => <AppField field={field} label="Registration no." placeholder="BMDC-12345" />}
							</form.Field>

							<div className="sm:col-span-2">
								<form.Field name="specialties">
									{(field) => (
										<AppMultiSelectField
											field={field}
											label="Specialties"
											options={specialtyOptions}
											emptyText="No specialties yet. Create a specialty first."
										/>
									)}
								</form.Field>
							</div>
						</div>

						<p className="text-xs text-muted-foreground">
							The doctor will be asked to change this password on first login.
						</p>

						<Separator />
					</>
				)}

				{/* ---------- Profile (create + edit) ---------- */}
				<div className="grid gap-4 sm:grid-cols-2">
					<div className="sm:col-span-2">
						<form.Field name="name">
							{(field) => <AppField field={field} label="Full name" placeholder="Dr. Jane Doe" autoComplete="off" />}
						</form.Field>
					</div>

					<form.Field name="contactNumber">
						{(field) => <AppField field={field} label="Phone" type="tel" inputMode="tel" placeholder="01XXXXXXXXX" />}
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
							{(field) => <AppField field={field} label="Address (optional)" />}
						</form.Field>
					</div>

					{/* backend create-এ photo নেয় না, তাই শুধু edit-এ */}
					{!isCreate && (
						<div className="sm:col-span-2">
							<form.Field name="profilePhoto">
								{(field) => (
									<AppField
										field={field}
										label="Profile photo URL (optional)"
										type="url"
										inputMode="url"
										placeholder="https://..."
									/>
								)}
							</form.Field>
						</div>
					)}
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

### `mode` prop
- `isCreate = mode === "create"` → এক জায়গায় হিসাব, তারপর পুরো form-এ ব্যবহার।
- **Schema mode অনুযায়ী:** `validators: { onChange: isCreate ? createDoctorFormSchema : editDoctorFormSchema }`।
- **Account section** (`{isCreate && (...)}`) শুধু create-এ। **Profile photo** (`{!isCreate && (...)}`) শুধু edit-এ।

### Specialty-র option

```ts
useQuery({ ...specialtiesQueryOptions(), enabled: isCreate });
```
- `enabled: isCreate` → edit mode-এ specialty লাগে না, তাই query চলেই না।
- Table page-এ specialty আগেই prefetch করা (note 16, filter-এর জন্য) → **cache-এ আছে**, নতুন request যায় না।
- Option-এর `value: specialty.id` — table filter-এ (note 17) `value: specialty.title`, কারণ backend filter title দিয়ে করে, কিন্তু create id চায়।

### Password field — দেখাও/লুকাও

```tsx
<AppField type={showPassword ? "text" : "password"} endAdornment={<Button ...>{showPassword ? <EyeOff /> : <Eye />}</Button>} />
```
- `AppField`-এর `endAdornment` (note 07-এর login form-এর মতো) → input-এর ডান পাশে চোখের icon।
- `type="button"` → form-এর ভিতরে `<button>`-এর default `submit`, তাই এটা না দিলে চোখে click করলেই form submit হত। `aria-label` অবস্থা অনুযায়ী বদলায়।
- `autoComplete="new-password"` → browser admin-এর নিজের saved password এখানে বসিয়ে দেয় না, বরং নতুন password suggest করতে পারে।
- Label "Temporary password" + নিচে লেখা: doctor প্রথম login-এ এটা বদলাতে বাধ্য হবে (backend `needPasswordChange: true` দেয় → note 08-এর proxy `/change-password`-এ পাঠায়)।

### Email field
| Prop | কেন |
|---|---|
| `type="email"` `inputMode="email"` | `@` সহ keyboard |
| `autoComplete="off"` | admin-এর নিজের email autofill না হয় (এটা অন্য মানুষের account) |
| `autoCapitalize="none"` | mobile-এ প্রথম অক্ষর বড় হাতের না হয় |
| `spellCheck={false}` | email-এর নিচে লাল দাগ না |

### Layout
- `fieldset` এখন `space-y-6` → ভিতরে দুটো grid (account, profile), মাঝে `Separator`।
- Name-এ `autoComplete="off"` (note 19-এ `"name"` ছিল) → admin-এর নিজের নাম autofill না হয়।

---

## ধাপ ৮: `CreateDoctorDialog.tsx` (নতুন)

`src/components/modules/Admin/DoctorsManagement/CreateDoctorDialog.tsx`

```tsx
"use client";

import { createDoctorAction } from "@/actions/doctor.actions";
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
import { DOCTORS_QUERY_KEY } from "@/queries/doctor.queries";
import type { ICreateDoctorPayload } from "@/types/doctor.types";
import type { DoctorFormValues } from "@/zod/doctor.validation";
import { Plus } from "lucide-react";
import { useState } from "react";
import DoctorForm from "./DoctorForm";
import { EMPTY_DOCTOR_FORM, toCreateDoctorPayload } from "./doctorForm.utils";

export default function CreateDoctorDialog() {
	const [open, setOpen] = useState(false);

	const { mutate, isPending } = useActionMutation({
		action: (payload: ICreateDoctorPayload) => createDoctorAction(payload),
		invalidateKey: DOCTORS_QUERY_KEY,
		successMessage: "Doctor account created",
		onSuccess: () => setOpen(false), // dialog বন্ধ হলে form unmount হয় → পরের বার ফাঁকা form
	});

	const handleSubmit = (values: DoctorFormValues) => mutate(toCreateDoctorPayload(values));

	return (
		// save চলার সময় Esc বা বাইরে click করে বন্ধ করা যাবে না
		<Dialog open={open} onOpenChange={(next) => !isPending && setOpen(next)}>
			<DialogTrigger asChild>
				<Button>
					<Plus />
					Add doctor
				</Button>
			</DialogTrigger>

			<DialogContent className="max-h-[90dvh] overflow-y-auto sm:max-w-2xl">
				<DialogHeader>
					<DialogTitle>Add doctor</DialogTitle>
					<DialogDescription>Create a doctor account. They can log in with this email and password.</DialogDescription>
				</DialogHeader>

				<DoctorForm
					mode="create"
					defaultValues={EMPTY_DOCTOR_FORM}
					onSubmit={handleSubmit}
					onCancel={() => setOpen(false)}
					isPending={isPending}
					submitLabel="Create doctor"
					pendingLabel="Creating..."
				/>
			</DialogContent>
		</Dialog>
	);
}
```

### Edit dialog-এর সাথে পার্থক্য

| | Create | Edit |
|---|---|---|
| খোলা/বন্ধ state | **নিজের** `useState(false)` | বাইরে থেকে (`useRowAction`) |
| খোলার উপায় | `DialogTrigger asChild` → "Add doctor" button নিজেই | row-এর "⋯" menu |
| row | নেই | আছে |

- **`DialogTrigger asChild`** → button চাপলে Radix নিজেই `setOpen(true)` করে।
- `onOpenChange={(next) => !isPending && setOpen(next)}` → তৈরি চলার সময় বন্ধ করা যাবে না।
- `onSuccess: () => setOpen(false)` → dialog বন্ধ হলে Radix `DialogContent`-কে **unmount** করে → `DoctorForm`-এর state মুছে যায় → পরেরবার খুললে **ফাঁকা form** (আলাদা reset লাগে না)।
- ব্যর্থ হলে dialog খোলা থাকে, টাইপ করা সব মান থাকে → ভুল ঠিক করে আবার চেষ্টা।
- `handleSubmit` → `mutate(toCreateDoctorPayload(values))`।

### EditDoctorDialog (পুরো file)

```tsx
"use client";

import { updateDoctorAction } from "@/actions/doctor.actions";
import { Dialog, DialogContent, DialogDescription, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { useActionMutation } from "@/hooks/useActionMutation";
import { DOCTORS_QUERY_KEY } from "@/queries/doctor.queries";
import type { IDoctor, IUpdateDoctorPayload } from "@/types/doctor.types";
import type { DoctorFormValues } from "@/zod/doctor.validation";
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

	const handleSubmit = (values: DoctorFormValues) => {
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

নোট 19 থেকে যা বদলেছে: `DoctorFormValues` type, আর `<DoctorForm mode="edit" ... />`। বাকি সব একই।

---

## ধাপ ৯: `DoctorsTable.tsx` (পুরো file)

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
import CreateDoctorDialog from "./CreateDoctorDialog";
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

	// filter-এ title দিয়ে খোঁজা হয় (backend specialties.specialty.title)
	const specialtyOptions = useMemo(
		() => (specialtiesResponse?.data ?? []).map((specialty) => ({ label: specialty.title, value: specialty.title })),
		[specialtiesResponse],
	);

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
			{/* Toolbar: search + filters + add */}
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

					<CreateDoctorDialog />
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
				actions={rowAction.actions}
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

নোট 19 থেকে যা বদলেছে:
- `import CreateDoctorDialog from "./CreateDoctorDialog";`
- Toolbar-এর filter-গুলোর শেষে **`<CreateDoctorDialog />`** → "Add doctor" button।
- `actions={rowAction.actions}` সরাসরি (আলাদা `tableActions` variable বাদ)।
- Specialty option-এর উপরে comment: filter title দিয়ে (create-এ id দিয়ে)।

---

## 📁 Doctor management-এর সব file (চূড়ান্ত)

```
src/
├── types/doctor.types.ts                    ← IDoctor, IDoctorProfilePayload, IUpdateDoctorPayload, ICreateDoctorPayload
├── zod/doctor.validation.ts                 ← create/edit form schema + create/update payload schema
├── services/doctor.services.ts              ← get, create, update, delete
├── actions/doctor.actions.ts                ← createDoctorAction, updateDoctorAction, deleteDoctorAction
├── queries/doctor.queries.ts                ← DOCTORS_QUERY_KEY, doctorsQueryOptions
├── components/shared/form/
│   ├── AppField.tsx, AppSubmitButton.tsx    ← (note 07)
│   ├── AppSelectField.tsx                   ← (note 19)
│   └── AppMultiSelectField.tsx              ← নতুন
└── components/modules/Admin/DoctorsManagement/
    ├── doctorsTable.config.ts, doctorsColumns.tsx, DoctorsTable.tsx
    ├── doctorForm.utils.ts, DoctorForm.tsx
    ├── CreateDoctorDialog.tsx               ← নতুন
    ├── ViewDoctorDialog.tsx, EditDoctorDialog.tsx, DeleteDoctorDialog.tsx
```

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | "Add doctor" | dialog: account section (email, password, reg. no., specialty) + separator + profile; photo field নেই |
| ২ | কিছু না লিখে "Create doctor" | প্রতিটা বাধ্যতামূলক field-এ error, "Select at least one specialty" |
| ৩ | Email "abc" | "Invalid email address"; ফাঁকা হলে "Email is required" |
| ৪ | Password ৪ অক্ষর | "Password must be at least 6 characters" |
| ৫ | চোখের icon | password দেখা/লুকানো |
| ৬ | Specialty একটা বেছে আবার বাদ | সাথে সাথে "Select at least one specialty" |
| ৭ | সব ঠিক → Create | "Creating…" → dialog বন্ধ → "Doctor account created" → table-এ নতুন doctor |
| ৮ | আবার "Add doctor" | form ফাঁকা |
| ৯ | আগের email দিয়ে আবার create | লাল toast (backend-এর message), dialog খোলা, মান থাকে |
| ১০ | নতুন doctor দিয়ে login | `/change-password`-এ পাঠায় (needPasswordChange) |
| ১১ | Edit dialog খোলো | account section নেই, photo আছে, আগের মতো কাজ করে |
| ১২ | Network → Edit-এর সময় specialties request | নেই (`enabled: false`) |

---

## 🔜 পরে যা যোগ হবে
- [ ] Doctor-এর specialty বদলানো (edit-এ এখন যায় না)
- [ ] Profile photo upload (backend multer, server note 23) — এখন শুধু edit-এ URL
- [ ] একই pattern-এ Admin management (`/admin/dashboard/admins-management`)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **`_profilePhoto`-তে ESLint warning আসে** — Client project-এ ESLint (`eslint-config-next`) চলে, আর এর `@typescript-eslint/no-unused-vars` নিয়ম `_` দিয়ে শুরু নাম ছাড় দেয় না। আমি চালিয়ে দেখেছি:
   ```
   doctorForm.utils.ts  62:24  warning  '_profilePhoto' is assigned a value but never used
   ```
   `eslint.config.mjs`-এ নিয়মটা ঠিক করে দাও (তাহলে পরে সব জায়গায় `_` চলবে, আর rest destructuring-এ নাম না বদলালেও চলবে):
   ```js
   const eslintConfig = defineConfig([
   	...nextVitals,
   	...nextTs,
   	...pluginQuery.configs["flat/recommended"],
   	{
   		rules: {
   			"@typescript-eslint/no-unused-vars": [
   				"warn",
   				{ argsIgnorePattern: "^_", varsIgnorePattern: "^_", ignoreRestSiblings: true },
   			],
   		},
   	},
   	globalIgnores([...]),
   ]);
   ```

2. **Temporary password admin-কে নিজে doctor-কে জানাতে হবে** — এখন account তৈরির পর doctor-কে email-এ কিছু যায় না। Admin password ভুলে গেলে doctor login করতে পারবে না (তখন forgot password দিয়ে reset করতে হবে)। পরে backend থেকে "welcome email + temporary password" পাঠানো যায় (server-এ nodemailer আছে, note 20)।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **Specialty অনেক হলে toggle button জায়গা নেবে** — ৩০–৪০টার বেশি হলে note 17-এর মতো `DropdownMenu` + checkbox, বা search সহ list ভালো। এখন কম, তাই ঠিক আছে।

4. **একই form দুই mode-এ — কতদূর?** এখন `mode` দিয়ে দুটো section লুকানো/দেখানো সহজ। কিন্তু create আর edit-এর পার্থক্য যত বাড়বে (আলাদা field, আলাদা নিয়ম), এক form-এ তত `if` জমবে। তখন কমন অংশ (`DoctorProfileFields`) আলাদা component করে দুটো আলাদা form বানানো পরিষ্কার হবে।
