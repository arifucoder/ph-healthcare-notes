# 09 - Dashboard Nav Items (Role অনুযায়ী Sidebar Menu)

> Dashboard-এর sidebar-এ কোন menu দেখাব, সেটা **role অনুযায়ী** আলাদা। এই নোটে শুধু menu-র **data** বানানো হয়েছে: type, icon আর role অনুযায়ী list। Sidebar UI (component) পরের নোটে।

## 🧭 এক নজরে

```
role (token থেকে)  →  getNavItemsByRole(role)
                        ├─ getCommonNavItems(role)   ← সবার: Home, Dashboard, My Profile, Change Password
                        └─ roleNavItems[role]        ← ADMIN / DOCTOR / PATIENT-এর নিজের menu
                     →  NavSection[]  →  Sidebar প্রতিটা item-এর icon নাম দিয়ে getIconComponent(name)
```

### কোন file কী কাজ করে

| File | কাজ |
|---|---|
| `src/types/dashboard.types.ts` | একটা menu item আর section দেখতে কেমন (type) |
| `src/lib/iconMapper.ts` | icon-এর **নাম (string)** → আসল lucide icon component |
| `src/lib/navItems.ts` | কোন role কোন menu পাবে |

---

## ধাপ ১: Type — `src/types/dashboard.types.ts` (নতুন)

```ts
import type { NavIconName } from "@/lib/iconMapper";

export interface NavItem {
	title: string;
	href: string;
	icon: NavIconName; // string না: ভুল নাম compile-এর সময়েই ধরা পড়বে
}

export interface NavSection {
	title?: string;
	items: NavItem[];
}
```

- **`NavItem`** → একটা link: লেখা (`title`), কোথায় যাবে (`href`), কোন icon (`icon`)।
- **`NavSection`** → কয়েকটা item-এর একটা group, যেমন "User Management"। `title?` optional, কারণ প্রথম group-এর (Home, Dashboard…) উপরে কোনো heading থাকে না।
- `icon: NavIconName` → এটা `"House" | "User" | "Calendar" | ...`-এর মতো union type (ধাপ ২-এ বানানো)। `icon: "Hom"` (typo) লিখলে সাথে সাথে **TypeScript error**। শুধু `string` দিলে typo ধরা পড়ত না, sidebar-এ icon আসত না।
- `import type` → শুধু type আনছি, runtime-এ কিছু import হয় না।

```
NavSection[]
 ├─ { items: [Home, Dashboard, My Profile] }           ← title নেই
 ├─ { title: "Settings", items: [Change Password] }
 └─ { title: "User Management", items: [Admins, Doctors, Patients] }
```

---

## ধাপ ২: Icon Mapper — `src/lib/iconMapper.ts` (নতুন)

```ts
import {
	Activity,
	Calendar,
	CalendarClock,
	CircleHelp,
	ClipboardList,
	Clock,
	CreditCard,
	FileText,
	Hospital,
	House,
	KeyRound,
	LayoutDashboard,
	type LucideIcon,
	Shield,
	Star,
	Stethoscope,
	User,
	Users,
} from "lucide-react";

// শুধু যে icon লাগে সেগুলোই import → পুরো lucide library bundle-এ ঢোকে না
const navIcons = {
	Activity,
	Calendar,
	CalendarClock,
	ClipboardList,
	Clock,
	CreditCard,
	FileText,
	Hospital,
	House,
	KeyRound,
	LayoutDashboard,
	Shield,
	Star,
	Stethoscope,
	User,
	Users,
} satisfies Record<string, LucideIcon>;

// নতুন icon লাগলে উপরে import করে navIcons-এ যোগ করো, type নিজেই আপডেট হবে
export type NavIconName = keyof typeof navIcons;

export const getIconComponent = (name: NavIconName): LucideIcon => navIcons[name] ?? CircleHelp;
```

### কেন icon-এর নাম (string), সরাসরি component না? ⭐

Menu data-তে সরাসরি `icon: House` (component) রাখলে সহজ মনে হয়। কিন্তু:

- Dashboard layout সাধারণত **Server Component** (cookie থেকে role পড়ে)। Sidebar হবে **Client Component** (`"use client"`), কারণ এখন কোন page খোলা আছে (active link) জানতে `usePathname()` hook লাগে।
- Server Component থেকে Client Component-এ prop হিসেবে শুধু **serializable** জিনিস পাঠানো যায় (string, number, plain object, array)। **Function বা component পাঠানো যায় না** → error।
- তাই data-তে শুধু নাম `"House"` (string) যায়, আর Client Component-এর ভিতরে `getIconComponent("House")` দিয়ে আসল icon বানানো হয়।

```
Server layout ──(navItems: { icon: "House" } ✅ string)──► Client Sidebar ──getIconComponent──► <House />
Server layout ──(navItems: { icon: House } ❌ function)──► Error
```

### ভেঙে বোঝা

| অংশ | মানে |
|---|---|
| আলাদা আলাদা named import | Bundler (tree-shaking) শুধু এই ১৭টা icon রাখে। `import * as Icons from "lucide-react"` করে `Icons[name]` করলে হাজারখানেক icon সব bundle-এ ঢুকে যেত |
| `type LucideIcon` | প্রতিটা lucide icon component-এর type |
| `satisfies Record<string, LucideIcon>` | check করে যে প্রতিটা value একটা icon, **কিন্তু** object-এর আসল key-গুলো (`"House"`, `"User"`…) type-এ রেখে দেয় |
| `keyof typeof navIcons` | object-এর সব key দিয়ে union: `"Activity" \| "Calendar" \| ...` → এটাই `NavIconName` |
| `?? CircleHelp` | কোনোভাবে নাম না মিললে (?) icon দেখাবে, sidebar ভাঙবে না |

**`satisfies` বনাম `: Record<string, LucideIcon>`**

| লেখা | `NavIconName` কী হবে |
|---|---|
| `const navIcons: Record<string, LucideIcon> = {...}` | `string` → typo ধরা পড়বে না ❌ |
| `const navIcons = {...} satisfies Record<string, LucideIcon>` | `"Activity" \| "Calendar" \| ...` ✅ |

**নতুন icon যোগ করার নিয়ম:** lucide থেকে import → `navIcons`-এ যোগ → ব্যস। `NavIconName` নিজেই আপডেট হবে।
👉 icon খুঁজতে: https://lucide.dev/icons

---

## ধাপ ৩: Nav Items — `src/lib/navItems.ts`

```ts
import type { UserRole } from "@/types/auth.types";
import type { NavSection } from "@/types/dashboard.types";
import { getDefaultDashboardRoute } from "./authUtils";

export const getCommonNavItems = (role: UserRole): NavSection[] => [
	{
		items: [
			{ title: "Home", href: "/", icon: "House" },
			{ title: "Dashboard", href: getDefaultDashboardRoute(role), icon: "LayoutDashboard" },
			{ title: "My Profile", href: "/my-profile", icon: "User" },
		],
	},
	{
		title: "Settings",
		items: [{ title: "Change Password", href: "/change-password", icon: "KeyRound" }],
	},
];

export const doctorNavItems: NavSection[] = [
	{
		title: "Patient Management",
		items: [
			{ title: "Appointments", href: "/doctor/dashboard/appointments", icon: "Calendar" },
			{ title: "My Schedules", href: "/doctor/dashboard/my-schedules", icon: "Clock" },
			{ title: "Prescriptions", href: "/doctor/dashboard/prescriptions", icon: "FileText" },
			{ title: "My Reviews", href: "/doctor/dashboard/my-reviews", icon: "Star" },
		],
	},
];

export const adminNavItems: NavSection[] = [
	{
		title: "User Management",
		items: [
			{ title: "Admins", href: "/admin/dashboard/admins-management", icon: "Shield" },
			{ title: "Doctors", href: "/admin/dashboard/doctors-management", icon: "Stethoscope" },
			{ title: "Patients", href: "/admin/dashboard/patients-management", icon: "Users" },
		],
	},
	{
		title: "Hospital Management",
		items: [
			{ title: "Appointments", href: "/admin/dashboard/appointments-management", icon: "Calendar" },
			{ title: "Schedules", href: "/admin/dashboard/schedules-management", icon: "Clock" },
			{ title: "Specialties", href: "/admin/dashboard/specialties-management", icon: "Hospital" },
			{ title: "Doctor Schedules", href: "/admin/dashboard/doctor-schedules-management", icon: "CalendarClock" },
			{ title: "Doctor Specialties", href: "/admin/dashboard/doctor-specialties-management", icon: "Stethoscope" },
			{ title: "Payments", href: "/admin/dashboard/payments-management", icon: "CreditCard" },
			{ title: "Prescriptions", href: "/admin/dashboard/prescriptions-management", icon: "FileText" },
			{ title: "Reviews", href: "/admin/dashboard/reviews-management", icon: "Star" },
		],
	},
];

export const patientNavItems: NavSection[] = [
	{
		title: "Appointments",
		items: [
			{ title: "My Appointments", href: "/dashboard/my-appointments", icon: "Calendar" },
			{ title: "Book Appointment", href: "/dashboard/book-appointments", icon: "ClipboardList" },
		],
	},
	{
		title: "Medical Records",
		items: [
			{ title: "My Prescriptions", href: "/dashboard/my-prescriptions", icon: "FileText" },
			{ title: "Health Records", href: "/dashboard/health-records", icon: "Activity" },
		],
	},
];

// Record<UserRole, ...>: নতুন role যোগ হলে এখানে না লিখলে TypeScript error দেবে
const roleNavItems: Record<UserRole, NavSection[]> = {
	SUPER_ADMIN: adminNavItems,
	ADMIN: adminNavItems,
	DOCTOR: doctorNavItems,
	PATIENT: patientNavItems,
};

export const getNavItemsByRole = (role: UserRole): NavSection[] => [
	...getCommonNavItems(role),
	...roleNavItems[role],
];
```

### ভেঙে বোঝা

**`getCommonNavItems(role)` — function কেন, সাধারণ array না?**
"Dashboard" link প্রতিটা role-এর জন্য আলাদা (`/admin/dashboard`, `/doctor/dashboard`, `/dashboard`)। তাই role নিয়ে `getDefaultDashboardRoute(role)` (note 07-এর `authUtils.ts`) থেকে ঠিক link বসানো হয়। Dashboard-এর path এক জায়গাতেই লেখা থাকে — proxy আর sidebar দুজনেই একই function ব্যবহার করে।

**Role-এর list-গুলো সাধারণ array** (`doctorNavItems` ইত্যাদি), কারণ এগুলো role-এর উপর নির্ভর করে না।

**`roleNavItems: Record<UserRole, NavSection[]>`**
- `Record<UserRole, ...>` মানে `UserRole`-এর **প্রতিটা** value-র জন্য key থাকতেই হবে।
- পরে নতুন role (যেমন `"NURSE"`) যোগ করলে এখানে না লিখলে TypeScript error → menu দিতে ভুলে যাওয়ার সুযোগ নেই।
- `SUPER_ADMIN` আর `ADMIN` একই menu পায় (proxy-তেও SUPER_ADMIN-কে ADMIN ধরা হয়, note 08)।
- `if/else` বা `switch`-এর চেয়ে ছোট আর পরিষ্কার।

**`getNavItemsByRole(role)`** → common section + role-এর section, spread (`...`) দিয়ে একটা array।

### কে কী menu পায়

| Role | Common (সবাই) | নিজের |
|---|---|---|
| ADMIN / SUPER_ADMIN | Home, Dashboard → `/admin/dashboard`, My Profile, Change Password | **User Management** (Admins, Doctors, Patients) + **Hospital Management** (Appointments, Schedules, Specialties, Doctor Schedules, Doctor Specialties, Payments, Prescriptions, Reviews) |
| DOCTOR | Home, Dashboard → `/doctor/dashboard`, My Profile, Change Password | **Patient Management** (Appointments, My Schedules, Prescriptions, My Reviews) |
| PATIENT | Home, Dashboard → `/dashboard`, My Profile, Change Password | **Appointments** (My Appointments, Book Appointment) + **Medical Records** (My Prescriptions, Health Records) |

### Nav item-এর `href` আর proxy-র route মিলছে তো?

| href শুরু | Proxy-তে (note 08) কার | ✅ |
|---|---|---|
| `/admin/dashboard/...` | `ADMIN` | ✅ |
| `/doctor/dashboard/...` | `DOCTOR` | ✅ |
| `/dashboard/...` | `PATIENT` | ✅ |
| `/my-profile`, `/change-password` | `COMMON` | ✅ |
| `/` | public | ✅ |

> Menu লুকানো শুধু **UX**। কেউ URL হাতে লিখে অন্য role-এর page খুললে proxy আটকায়, আর data আটকায় backend-এর `checkAuth`।

---

## ✅ নতুন menu item যোগ করার checklist

1. দরকার হলে `iconMapper.ts`-এ icon import + `navIcons`-এ যোগ।
2. `navItems.ts`-এ ঠিক role-এর ঠিক section-এ `{ title, href, icon }`।
3. `href` যেন proxy-র ঠিক role-এর path দিয়ে শুরু হয় (উপরের table)।
4. `src/app/(dashboardLayout)/...`-এ সেই page বানাও (নাহলে 404)।

---

## 🧪 Test (sidebar বানানোর আগে)

এখন শুধু data, তাই TypeScript দিয়েই check:
- `navItems.ts`-এ একটা `icon: "Hom"` লিখে দেখো → red underline (`"Hom"` is not assignable to type `NavIconName`)। তারপর ঠিক করো।
- `roleNavItems` থেকে `DOCTOR` লাইনটা সাময়িক মুছে দেখো → error (property `DOCTOR` is missing)।
- `pnpm build` বা `pnpm lint` চালিয়ে কোনো error নেই নিশ্চিত হও।

---

## 🔜 পরে যা যোগ হবে
- [ ] Dashboard layout (Server Component): cookie থেকে access token → `verifyAccessToken` → `role` → `getNavItemsByRole(role)`
- [ ] Sidebar (Client Component, shadcn `sidebar`): `usePathname()` দিয়ে active link, `getIconComponent(item.icon)` দিয়ে icon
- [ ] Mobile-এ sidebar (sheet/drawer)
- [ ] Menu-র প্রতিটা page (এখন বেশিরভাগ page নেই → 404)

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Active link check-এ `"/dashboard"` সাবধানে** — Sidebar-এ "এই link active কিনা" দেখতে শুধু `pathname.startsWith(href)` করলে patient-এর `/dashboard/my-appointments` খুললে **"Dashboard" আর "My Appointments" দুটোই** active দেখাবে (আর `"/"` (Home) সব page-এ active হবে)। Dashboard আর Home-এর জন্য exact match:
   ```ts
   const isActive = (pathname: string, href: string, isRoot: boolean) =>
   	isRoot ? pathname === href : pathname === href || pathname.startsWith(`${href}/`);
   ```
   Home আর Dashboard-এ `isRoot = true`। (পরের নোটে sidebar বানানোর সময় কাজে লাগবে।)

2. **Menu-র বেশিরভাগ page এখনো নেই** — project-এ এখন শুধু `admin/dashboard/admins-management` আছে। Sidebar বানালে বাকি link-এ click করলে 404 দেখাবে। Page না বানানো পর্যন্ত এটা স্বাভাবিক — শুধু মনে রাখো, ভুল ভেবে debug করতে যেও না।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`?? CircleHelp` এখন আসলে কখনো চলবে না** — `name`-এর type `NavIconName`, তাই TypeScript ভুল নাম ঢুকতেই দেয় না। Fallback কাজে লাগবে যদি কখনো menu data backend বা JSON থেকে আসে (তখন type check runtime-এ থাকে না)। রেখে দেওয়ায় ক্ষতি নেই।

4. **Role কোথা থেকে নেবে** — Sidebar-এর জন্য role নিতে আবার `/auth/me` call করার দরকার নেই। Layout-এ `getCookie(AUTH_COOKIES.accessToken)` → `verifyAccessToken()` (note 08-এর `jwtUtils.ts`) → `user.role`। Proxy আগেই token refresh করে request-এ বসিয়ে দেয়, তাই layout সবসময় নতুন token পায়।
