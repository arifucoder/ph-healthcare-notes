# 10 - Dashboard Root Layout, Sidebar ও Mobile Navbar

> Module: 47-3 (Making Dashboard Sidebar Component Part-1)

> Note 09-এ role অনুযায়ী menu-র **data** বানিয়েছিলাম। এই নোটে সেই data দিয়ে dashboard-এর **কাঠামো (shell)** বানানো হলো: বাঁয়ে sidebar (desktop), উপরে navbar (mobile-এ hamburger menu), মাঝখানে page।

## 🧭 এক নজরে

```
Request: /admin/dashboard
  → proxy.ts (note 08): login আছে? role ঠিক? token refresh
  → (dashboardLayout)/layout.tsx  [Server]
       → getSessionUser()  → cookie থেকে accessToken → verify → { name, email, role }
       → getNavItemsByRole(role), getDefaultDashboardRoute(role)
       → <DashboardSidebar />  (desktop, md+)        ┐ একই props
       → <DashboardNavbar />   (mobile Sheet + user) ┘ { user, navItems, dashboardHome }
       → <main>{children}</main>  ← আসল page
```

### Screen-এ কেমন দেখায়

```
Desktop (md+)                              Mobile
┌──────────┬─────────────────────────┐     ┌───────────────────────┐
│ Phdoc    │              👤 Arif    │     │ ☰  Phdoc     👤 Arif  │ ← Navbar
│──────────│─────────────────────────│     │───────────────────────│
│ Home     │                         │     │                       │
│ Dashboard│        <main>           │     │       <main>          │
│ ...      │       page এখানে        │     │                       │
│──────────│                         │     │  ☰ চাপলে বাঁ দিক থেকে  │
│ 👤 Arif  │                         │     │  Sheet-এ menu খোলে    │
└──────────┴─────────────────────────┘     └───────────────────────┘
 Sidebar        Navbar + main
```

### কোন file কী কাজ করে

| File | কাজ | Server / Client |
|---|---|---|
| `types/auth.types.ts` | `ISessionUser` যোগ (UI-তে যতটুকু লাগে) | type |
| `types/dashboard.types.ts` | `IDashboardShellProps` যোগ | type |
| `lib/session.ts` | cookie থেকে logged-in user বের করা | Server (`server-only`) |
| `app/(dashboardLayout)/layout.tsx` | সব জোড়া লাগানো | Server |
| `Dashboard/DashboardNav.tsx` | menu link-গুলো + active link | **Client** (`usePathname`) |
| `Dashboard/DashboardUserInfo.tsx` | avatar + নাম + role | দুই জায়গাতেই চলে |
| `Dashboard/DashboardSidebar.tsx` | desktop sidebar | Server |
| `Dashboard/DashboardNavbar.tsx` | উপরের bar + mobile Sheet menu | **Client** (`useState`) |

---

## ধাপ ১: shadcn `sheet` install

```shell
pnpm dlx shadcn@latest add sheet
```

👉 https://ui.shadcn.com/docs/components/sheet

- **Sheet** = পাশ থেকে slide করে আসা panel (Radix Dialog-এর উপর বানানো)। Mobile-এ sidebar-এর জায়গায় এটা ব্যবহার করব।

> ✏️ নোটে ছিল না: এই নোটে `separator` component-ও লাগে (`DashboardNav`-এ)। `avatar` আর `button` আগেই install করা আছে (note 01, 06)। Project-এ `separator` আছে; নতুন project-এ না থাকলে:
> ```shell
> pnpm dlx shadcn@latest add separator
> ```

---

## ধাপ ২: `src/types/auth.types.ts` (পুরো file)

```ts
export type UserRole = "SUPER_ADMIN" | "ADMIN" | "DOCTOR" | "PATIENT";
export type UserStatus = "ACTIVE" | "BLOCKED" | "DELETED";

export interface IAuthUser {
	id: string;
	name: string;
	email: string;
	emailVerified: boolean;
	image: string | null;
	role: UserRole;
	status: UserStatus;
	needPasswordChange: boolean;
	isDeleted: boolean;
	deletedAt: string | null;
	createdAt: string;
	updatedAt: string;
}

export interface ILoginResponse {
	token: string;
	accessToken: string;
	refreshToken: string;
	redirect: boolean;
	url?: string;
	user: IAuthUser;
}

// UI-তে যতটুকু লাগে শুধু ততটুকু (client-এ কম data যায়)
export interface ISessionUser {
	name: string;
	email: string;
	role: UserRole;
}
```

**নতুন: `ISessionUser`** — শুধু `name`, `email`, `role`।

- Layout থেকে এই user object **Client Component**-এ (Navbar) prop হিসেবে যায়। Server থেকে client-এ যা পাঠানো হয়, তা HTML-এর ভিতরে page-এ চলে যায় (যে কেউ দেখতে পারে)।
- তাই পুরো token payload (`userId`, `status`, `isDeleted`…) না পাঠিয়ে **UI-তে যতটুকু লাগে শুধু ততটুকু** → কম data, কম ঝুঁকি।

---

## ধাপ ৩: `src/types/dashboard.types.ts` (পুরো file)

```ts
import type { NavIconName } from "@/lib/iconMapper";
import type { ISessionUser } from "@/types/auth.types";

export interface NavItem {
	title: string;
	href: string;
	icon: NavIconName; // string না: ভুল নাম compile-এর সময়েই ধরা পড়বে
}

export interface NavSection {
	title?: string;
	items: NavItem[];
}

// layout → Sidebar/Navbar-এ একই props যায়
export interface IDashboardShellProps {
	user: ISessionUser;
	navItems: NavSection[];
	dashboardHome: string;
}
```

**নতুন: `IDashboardShellProps`** — Sidebar আর Navbar দুটোই হুবহু একই ৩টা prop পায়। একবার type লিখে দুই জায়গায় ব্যবহার।

| Prop | কী | উদাহরণ (ADMIN) |
|---|---|---|
| `user` | `ISessionUser` | `{ name: "Arif", email: "...", role: "ADMIN" }` |
| `navItems` | `NavSection[]` (note 09) | common + admin menu |
| `dashboardHome` | role-এর dashboard path | `"/admin/dashboard"` |

---

## ধাপ ৪: `src/lib/session.ts` (নতুন) — logged-in user কে?

```ts
import type { ISessionUser } from "@/types/auth.types";
import { connection } from "next/server";
import { cache } from "react";
import "server-only";
import { AUTH_COOKIES } from "./cookieConfig";
import { getCookie } from "./cookieUtils";
import { verifyAccessToken } from "./jwtUtils";

// cache(): একই request-এ যত component call করুক, verify হবে একবারই।
// proxy refresh করলে request-এর cookie আপডেট করে দেয়, তাই এখানে নতুন token-ই পাওয়া যায়।
export const getSessionUser = cache(async (): Promise<ISessionUser | null> => {
    const accessToken = await getCookie(AUTH_COOKIES.accessToken);
    if (!accessToken) return null;

    // jwt.verify() ভেতরে Date.now() দিয়ে exp check করে।
    // connection(): Next-কে বলে এটা prerender-এ না, আসল request-এর সময় চালাও।
    await connection();

    const payload = verifyAccessToken(accessToken);
    if (!payload) return null;

    return { name: payload.name, email: payload.email, role: payload.role };
});
```

### ভেঙে বোঝা

| অংশ | কেন |
|---|---|
| `import "server-only"` | cookie আর secret-এর কাজ, client bundle-এ গেলে build error (note 07) |
| `getCookie(AUTH_COOKIES.accessToken)` | note 07-এর `cookieUtils` → `cookies()` দিয়ে পড়া |
| `verifyAccessToken` | note 08-এর `jwtUtils` → signature + মেয়াদ check, ভুল হলে `null` |
| return শুধু ৩টা field | ধাপ ২-এর `ISessionUser` |

### `cache()` — একই request-এ একবারই verify

`cache` React-এর function। একই request-এর মধ্যে `getSessionUser()` যতবার, যত component থেকেই call করা হোক, **ভিতরের কাজ একবারই চলে**, বাকিবার আগের ফল ফেরত দেয়।

```
একটা request:
  layout.tsx       → getSessionUser()  → cookie পড়া + verify ✅ (আসল কাজ)
  page.tsx         → getSessionUser()  → আগের ফল ♻️
  কোনো component  → getSessionUser()  → আগের ফল ♻️
পরের request → আবার নতুন করে
```

- তাই পরে কোনো page-এ user লাগলে নিশ্চিন্তে `await getSessionUser()` call করা যায়, prop হিসেবে নামিয়ে আনতে হয় না।
- `cache` শুধু **একটা request-এর ভিতরে** কাজ করে, এক user-এর data আরেক user পাবে না।

### Proxy refresh করলে নতুন token পাই কীভাবে?

Note 08-এর `next()` function proxy-তে refresh হওয়া নতুন token **request-এর cookie-তেও** বসিয়ে দেয়। তাই এখানে `getCookie()` সবসময় নতুন token পায়, পুরনো expired token না।

### `connection()` কেন?

Project-এ `next.config.ts`-এ **`cacheComponents: true`** আছে। এতে Next.js build-এর সময় যতটা সম্ভব page আগে থেকে বানিয়ে (prerender) রাখতে চায়। `jwt.verify()` ভিতরে `Date.now()` দিয়ে `exp` check করে, আর "এখনকার সময়" build-এর সময় জানা সম্ভব না। `await connection()` Next-কে বলে: "এই অংশ আসল request আসার পরেই চালাও"।

---

## ধাপ ৫: `src/app/(dashboardLayout)/layout.tsx` (পুরো file)

```tsx
import DashboardNavbar from "@/components/modules/Dashboard/DashboardNavbar";
import DashboardSidebar from "@/components/modules/Dashboard/DashboardSidebar";
import { getDefaultDashboardRoute } from "@/lib/authUtils";
import { getNavItemsByRole } from "@/lib/navItems";
import { getSessionUser } from "@/lib/session";
import { redirect } from "next/navigation";

export default async function RootDashboardLayout({ children }: LayoutProps<"/">) {
	const user = await getSessionUser();

	// proxy আগেই আটকায়; এটা দ্বিতীয় সুরক্ষা (defense in depth)
	if (!user) redirect("/login");

	// একবার হিসাব করে দুই component-কে পাঠানো
	const navItems = getNavItemsByRole(user.role);
	const dashboardHome = getDefaultDashboardRoute(user.role);

	return (
		<div className="flex h-dvh overflow-hidden">
			<DashboardSidebar user={user} navItems={navItems} dashboardHome={dashboardHome} />

			<div className="flex min-w-0 flex-1 flex-col overflow-hidden">
				<DashboardNavbar user={user} navItems={navItems} dashboardHome={dashboardHome} />

				<main className="flex-1 overflow-y-auto bg-muted/10 p-4 md:p-6">{children}</main>
			</div>
		</div>
	);
}
```

### ভেঙে বোঝা

- **`async` Server Component** → সরাসরি `await getSessionUser()` করা যায়, `useEffect`/loading state লাগে না।
- **`if (!user) redirect("/login")`** → proxy আগেই আটকায়। তবুও এটা **দ্বিতীয় দরজা (defense in depth)**: কখনো proxy-র matcher ভুল হলে বা proxy বাদ পড়লেও dashboard খুলবে না। `redirect()` throw করে, তাই এর পরে TypeScript জানে `user` আর `null` না।
- **একবার হিসাব** → `navItems` আর `dashboardHome` layout-এ একবার বানিয়ে দুই component-কে পাঠানো। প্রতিটা component নিজে হিসাব করলে একই কাজ দুবার হত।
- **কোথায় বসে:** `(dashboardLayout)` route group-এর layout, তাই `/admin/dashboard/...`, `/doctor/dashboard/...`, `/dashboard/...`, `/my-profile`, `/change-password` — সবগুলোতে একই sidebar/navbar।

### Layout-এর Tailwind class

| Class | কাজ |
|---|---|
| `flex h-dvh overflow-hidden` | পুরো screen-এর উচ্চতা, sidebar বাঁয়ে আর বাকিটা ডানে। পুরো page scroll হয় না |
| `h-dvh` (`h-screen` না) | mobile browser-এ address bar উঠলে-নামলে উচ্চতা ঠিক থাকে |
| `min-w-0 flex-1` | ডান অংশ বাকি জায়গা নেয়। `min-w-0` না দিলে লম্বা content (table) flex item-কে চওড়া করে layout ভেঙে দেয় |
| `main`-এ `flex-1 overflow-y-auto` | **শুধু page অংশ scroll** হয়, sidebar আর navbar জায়গায় থাকে |
| `p-4 md:p-6` | mobile-এ কম, desktop-এ বেশি padding |

---

## ধাপ ৬: `DashboardNav.tsx` (client) — menu link

`src/components/modules/Dashboard/DashboardNav.tsx`

```tsx
"use client";

import { Separator } from "@/components/ui/separator";
import { getIconComponent } from "@/lib/iconMapper";
import type { NavSection } from "@/types/dashboard.types";
import { cn } from "cn";
import Link from "next/link";
import { usePathname } from "next/navigation";

interface DashboardNavProps {
	navItems: NavSection[];
	dashboardHome: string;
	onNavigate?: () => void; // mobile Sheet বন্ধ করার জন্য
}

// "/" আর dashboard home শুধু হুবহু মিললে active।
// বাকিগুলো sub-page-এও active, যেমন /admin/dashboard/admins-management/123।
const isActiveLink = (pathname: string, href: string, dashboardHome: string) => {
	if (href === "/" || href === dashboardHome) return pathname === href;
	return pathname === href || pathname.startsWith(`${href}/`);
};

export default function DashboardNav({ navItems, dashboardHome, onNavigate }: DashboardNavProps) {
	const pathname = usePathname();

	return (
		<nav aria-label="Dashboard navigation" className="space-y-4">
			{navItems.map((section, index) => (
				<div key={section.title ?? `section-${index}`}>
					{index > 0 && <Separator className="mb-4" />}

					{section.title && (
						<h4 className="mb-2 px-3 text-xs font-semibold uppercase text-muted-foreground">{section.title}</h4>
					)}

					<ul className="space-y-1">
						{section.items.map((item) => {
							const Icon = getIconComponent(item.icon);
							const isActive = isActiveLink(pathname, item.href, dashboardHome);

							return (
								<li key={item.href}>
									<Link
										href={item.href}
										onClick={onNavigate}
										aria-current={isActive ? "page" : undefined}
										className={cn(
											"flex items-center gap-3 rounded-lg px-3 py-2 text-sm font-medium transition-colors",
											isActive
												? "bg-primary text-primary-foreground"
												: "text-muted-foreground hover:bg-accent hover:text-accent-foreground",
										)}
									>
										<Icon className="size-4 shrink-0" aria-hidden />
										<span className="truncate">{item.title}</span>
									</Link>
								</li>
							);
						})}
					</ul>
				</div>
			))}
		</nav>
	);
}
```

### কেন `"use client"`?
Active link বুঝতে এখন কোন page খোলা আছে জানতে হয় → `usePathname()` hook। Hook শুধু Client Component-এ চলে। এজন্যই note 09-এ icon-এর নাম (string) রাখা হয়েছিল — server layout থেকে এই client component-এ শুধু string যায়, আর এখানে `getIconComponent(item.icon)` দিয়ে আসল icon হয়।

### `isActiveLink` — কোন link highlight হবে

| pathname | href | ফল | কেন |
|---|---|---|---|
| `/admin/dashboard` | `/admin/dashboard` (home) | ✅ | হুবহু মিলেছে |
| `/admin/dashboard/admins-management` | `/admin/dashboard` (home) | ❌ | home শুধু হুবহু মিললে |
| `/admin/dashboard/admins-management` | `/admin/dashboard/admins-management` | ✅ | হুবহু |
| `/admin/dashboard/admins-management/123` | `/admin/dashboard/admins-management` | ✅ | sub-page |
| `/admin/dashboard/admins-management-x` | `/admin/dashboard/admins-management` | ❌ | `${href}/` দেওয়ায় `-x` মিলবে না |
| যেকোনো page | `/` (Home) | শুধু `/`-এ ✅ | নাহলে Home সব page-এ active থাকত |

> Note 09-এর recommendation ১ এখানে ✅ (Done) — home আর dashboard-এ exact match।

### বাকি অংশ

| অংশ | কাজ |
|---|---|
| `onNavigate?` | mobile Sheet-এ link চাপলে Sheet বন্ধ করার function। Desktop-এ দেওয়া হয় না (`?` optional) |
| `key={section.title ?? \`section-${index}\`}` | প্রথম section-এর title নেই, তাই index দিয়ে key |
| `{index > 0 && <Separator />}` | প্রথম section ছাড়া প্রতিটার উপরে দাগ |
| `const Icon = getIconComponent(...)` | বড় হাতের `Icon` নাম দিলে JSX-এ `<Icon />` লেখা যায় |
| `aria-current="page"` | screen reader বলে "এটা বর্তমান page" — accessibility |
| `cn(...)` | active হলে `bg-primary`, না হলে হালকা রং + hover। `cn` class জোড়া লাগায় |
| `aria-hidden` (icon) | icon শুধু সাজসজ্জা, পাশে লেখা আছে, তাই screen reader পড়বে না |
| `truncate` | লম্বা নাম `...` হয়ে যাবে, লাইন ভাঙবে না |

---

## ধাপ ৭: `DashboardUserInfo.tsx` — avatar + নাম + role

`src/components/modules/Dashboard/DashboardUserInfo.tsx`

```tsx
import { Avatar, AvatarFallback } from "@/components/ui/avatar";
import type { ISessionUser } from "@/types/auth.types";

// "SUPER_ADMIN" → "super admin"
const formatRole = (role: ISessionUser["role"]) => role.toLowerCase().replace("_", " ");

export default function DashboardUserInfo({ user }: { user: ISessionUser }) {
	return (
		<div className="flex min-w-0 items-center gap-3">
			<Avatar>
				<AvatarFallback className="bg-primary/10 text-sm font-semibold text-primary">
					{user.name.charAt(0).toUpperCase()}
				</AvatarFallback>
			</Avatar>

			<div className="min-w-0">
				<p className="truncate text-sm font-medium">{user.name}</p>
				<p className="text-xs capitalize text-muted-foreground">{formatRole(user.role)}</p>
			</div>
		</div>
	);
}
```

- `"use client"` **নেই**, তবুও দুই জায়গায় চলে: Server `DashboardSidebar`-এর ভিতরে Server Component হিসেবে, আর Client `DashboardNavbar`-এর ভিতরে client হিসেবে। কোনো hook নেই বলে এটা সম্ভব।
- `AvatarFallback` → ছবি নেই, তাই নামের প্রথম অক্ষর (`"arif"` → `"A"`)।
- `formatRole` → `"SUPER_ADMIN"` → `"super admin"`, তারপর CSS `capitalize` → **Super Admin**।
- `min-w-0` + `truncate` → লম্বা নাম sidebar-এর বাইরে যাবে না।

---

## ধাপ ৮: `DashboardSidebar.tsx` (server, desktop)

`src/components/modules/Dashboard/DashboardSidebar.tsx`

```tsx
import type { IDashboardShellProps } from "@/types/dashboard.types";
import Link from "next/link";
import DashboardNav from "./DashboardNav";
import DashboardUserInfo from "./DashboardUserInfo";

export default function DashboardSidebar({ user, navItems, dashboardHome }: IDashboardShellProps) {
	return (
		// mobile-এ লুকানো, md থেকে দেখায়; mobile-এ Navbar-এর Sheet কাজ করে
		<aside className="hidden w-64 shrink-0 flex-col border-r bg-background md:flex">
			<div className="flex h-16 items-center border-b px-6">
				<Link href={dashboardHome} className="text-xl font-bold text-primary">
					Phdoc
				</Link>
			</div>

			<div className="flex-1 overflow-y-auto px-3 py-4">
				<DashboardNav navItems={navItems} dashboardHome={dashboardHome} />
			</div>

			<div className="border-t p-4">
				<DashboardUserInfo user={user} />
			</div>
		</aside>
	);
}
```

- **Server Component** (`"use client"` নেই) — নিজে কোনো hook ব্যবহার করে না। ভিতরের `DashboardNav` client, এভাবে Server-এর ভিতরে Client রাখা যায়।
- `hidden md:flex` → mobile-এ লুকানো, `md` (768px) থেকে দেখা যায়।
- `w-64 shrink-0` → সবসময় ২৫৬px, ছোট হবে না।
- ৩ ভাগ: উপরে logo (`h-16`, navbar-এর সমান উচ্চতা → দাগ এক লাইনে মেলে), মাঝে menu (`flex-1 overflow-y-auto` → menu লম্বা হলে শুধু এই অংশ scroll), নিচে user info।

---

## ধাপ ৯: `DashboardNavbar.tsx` (client, mobile menu)

`src/components/modules/Dashboard/DashboardNavbar.tsx`

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { Sheet, SheetContent, SheetDescription, SheetTitle, SheetTrigger } from "@/components/ui/sheet";
import type { IDashboardShellProps } from "@/types/dashboard.types";
import { Menu } from "lucide-react";
import Link from "next/link";
import { useState } from "react";
import DashboardNav from "./DashboardNav";
import DashboardUserInfo from "./DashboardUserInfo";

export default function DashboardNavbar({ user, navItems, dashboardHome }: IDashboardShellProps) {
	const [open, setOpen] = useState(false);
	const closeMenu = () => setOpen(false);

	return (
		<header className="flex h-16 shrink-0 items-center gap-3 border-b bg-background px-4 md:px-6">
			{/* শুধু mobile-এ */}
			<Sheet open={open} onOpenChange={setOpen}>
				<SheetTrigger asChild>
					<Button variant="ghost" size="icon" className="md:hidden" aria-label="Open navigation menu">
						<Menu />
					</Button>
				</SheetTrigger>

				<SheetContent side="left" className="flex w-72 flex-col gap-0 p-0">
					{/* screen reader-এর জন্য দরকার, নইলে Radix console-এ warning দেয় */}
					<SheetTitle className="sr-only">Navigation menu</SheetTitle>
					<SheetDescription className="sr-only">Dashboard pages</SheetDescription>

					<div className="flex h-16 items-center border-b px-6">
						<Link href={dashboardHome} onClick={closeMenu} className="text-xl font-bold text-primary">
							Phdoc
						</Link>
					</div>

					<div className="flex-1 overflow-y-auto px-3 py-4">
						<DashboardNav navItems={navItems} dashboardHome={dashboardHome} onNavigate={closeMenu} />
					</div>

					<div className="border-t p-4">
						<DashboardUserInfo user={user} />
					</div>
				</SheetContent>
			</Sheet>

			<Link href={dashboardHome} className="text-lg font-bold text-primary md:hidden">
				Phdoc
			</Link>

			<div className="ml-auto">
				<DashboardUserInfo user={user} />
			</div>
		</header>
	);
}
```

### কেন `"use client"`?
Sheet খোলা/বন্ধ (`open`) state রাখতে `useState`।

### Sheet কীভাবে কাজ করে

```
☰ (SheetTrigger) চাপলে → open = true → বাঁ দিক থেকে SheetContent
  menu-র link চাপলে → onNavigate = closeMenu → open = false → Sheet বন্ধ, নতুন page
  বাইরে চাপলে / Esc → onOpenChange(false) → বন্ধ
```

| অংশ | কাজ |
|---|---|
| `open` + `onOpenChange` (controlled) | নিজের state দিয়ে চালানো, যাতে link চাপলে code থেকে বন্ধ করা যায়। না করলে নতুন page-এ গিয়েও Sheet খোলা থাকত |
| `SheetTrigger asChild` | আলাদা button না বানিয়ে আমাদের `Button`-কেই trigger বানায় |
| `className="md:hidden"` (button) | hamburger শুধু mobile-এ |
| `aria-label` | icon-only button, screen reader-এর জন্য নাম |
| `side="left"` | বাঁ দিক থেকে আসে (sidebar-এর মতো) |
| `SheetTitle` + `SheetDescription` `sr-only` | Radix Dialog-এ title বাধ্যতামূলক (accessibility), নাহলে console-এ warning। `sr-only` → চোখে দেখা যায় না, শুধু screen reader পড়ে |
| ভিতরের অংশ | Sidebar-এর মতোই: logo + `DashboardNav` (+ `onNavigate`) + user info |
| `ml-auto` | user info ডান কোণে ঠেলে দেয় |

---

## 📐 Server আর Client — কে কোথায়

```
layout.tsx (Server)  ── getSessionUser() ──► user, navItems, dashboardHome
 ├─ DashboardSidebar (Server)
 │    ├─ DashboardNav (Client: usePathname)
 │    └─ DashboardUserInfo (Server হিসেবে)
 ├─ DashboardNavbar (Client: useState)
 │    ├─ Sheet → DashboardNav (Client)
 │    └─ DashboardUserInfo (Client হিসেবে)
 └─ main → page
```

নিয়ম: যত বেশি অংশ Server-এ রাখা যায় তত কম JavaScript browser-এ যায়। শুধু hook লাগলেই `"use client"`।

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | ADMIN হয়ে login → `/admin/dashboard` | বাঁয়ে sidebar: Home, Dashboard (highlight), My Profile, Change Password, User Management, Hospital Management |
| ২ | Sidebar-এর নিচে আর navbar-এর ডানে | নামের প্রথম অক্ষরের avatar + নাম + "Admin" |
| ৩ | PATIENT / DOCTOR হয়ে login | নিজের role-এর menu, Dashboard link নিজের dashboard-এ |
| ৪ | `Admins` link চাপো | `/admin/dashboard/admins-management`, এই link highlight, Dashboard আর highlight না |
| ৫ | Browser ছোট করো (768px-এর কম) বা DevTools mobile view | Sidebar লুকায়, উপরে ☰ |
| ৬ | ☰ চাপো → একটা link চাপো | Sheet খোলে, link চাপলে বন্ধ হয়ে নতুন page |
| ৭ | Sheet খোলা অবস্থায় Esc / বাইরে চাপো | বন্ধ |
| ৮ | Console দেখো | `DialogTitle` বা key-এর কোনো warning নেই |
| ৯ | অনেক লম্বা content দিয়ে page | শুধু main scroll হয়, sidebar/navbar জায়গায় থাকে |

> ✏️ এখনো বেশিরভাগ menu-র page নেই, তাই link চাপলে 404 আসবে (note 09) — এটা স্বাভাবিক।

---

## 🔜 পরে যা যোগ হবে
- [ ] Navbar-এ user dropdown: My Profile, Change Password, **Logout** (`clearAuthCookies`)
- [ ] Menu-র প্রতিটা page
- [ ] Dashboard-এর `loading.tsx` (skeleton) — page load-এর সময় sidebar থাকবে, শুধু main-এ loading
- [ ] User-এর ছবি থাকলে `AvatarImage`

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Desktop-এ user info দুবার দেখা যায়** — `md`-এর উপরে sidebar-এর নিচে আর navbar-এর ডানে, দুই জায়গাতেই একই avatar + নাম। পরে navbar-এরটা logout dropdown হবে, তাহলে ঠিক আছে। নাহলে sidebar-এরটা রেখে navbar-এরটা শুধু mobile-এ দেখাতে পারো:
   ```tsx
   <div className="ml-auto md:hidden">
   	<DashboardUserInfo user={user} />
   </div>
   ```

2. **Layout-এর `redirect` শুধু প্রথম load-এ চলে** — Next.js-এ এক page থেকে আরেক page-এ (client navigation) গেলে layout আবার render হয় না, আগেরটা থেকে যায়। তাই dashboard-এর ভিতরে ঘোরাঘুরির সময় আসল পাহারাদার **proxy** (প্রতিটা navigation-এ চলে), layout-এরটা শুধু বাড়তি সুরক্ষা। Page-এর data-র নিরাপত্তা সবসময় backend-এর `checkAuth`।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **`cacheComponents: true` আর cookie** — এই setting-এ `cookies()` (dynamic data) একটা `<Suspense>`-এর ভিতরে থাকতে হয়, নাহলে Next error দেয়। এখন `src/app/loading.tsx` আছে, যেটা নিজেই একটা Suspense boundary, তাই dashboard layout তার ভিতরে পড়ে — সমস্যা নেই। ওই `loading.tsx` মুছে দিলে এই error আসতে পারে (untested), তখন dashboard-এ নিজের `loading.tsx` রাখো।

4. **`formatRole`-এ `replace` শুধু প্রথম `_` বদলায়** — এখনকার role-গুলোতে একটাই `_` (`SUPER_ADMIN`), তাই ঠিক আছে। কখনো `"CLINIC_SUPER_ADMIN"`-এর মতো role এলে `replaceAll("_", " ")` লাগবে।

5. **নাম token থেকে আসে** — user profile-এ নাম বদলালে sidebar-এ পুরনো নাম দেখাবে, যতক্ষণ না token refresh হয় (backend নতুন token-এ DB থেকে নতুন নাম দেয়)। Profile update action-এ নতুন token নিয়ে `setAuthCookies` করলে সাথে সাথে বদলাবে।
