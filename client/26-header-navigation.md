# 26 - Header Navigation (Public Navbar: Desktop + Mobile + Auth)

> Public page-গুলোর (`(commonLayout)`) উপরের header: logo, menu link (active link চিহ্নিত), mobile-এ পাশ থেকে আসা Sheet menu, আর ডানে login থাকলে user avatar dropdown, না থাকলে "Login" button।
> Dashboard-এর dropdown (note 11) এখানেও reuse।

```
Navbar (server, static)
 ├─ MobileNav (client: Sheet open/close) ──┐
 ├─ Logo                                   ├─ দুজনেই একই NavLinks + publicNavItems
 ├─ NavLinks (client: usePathname → active)┘
 └─ <Suspense> NavbarAuth (async server: cookie → UserDropdown / Login)
```

## Folder structure

```
src/
├─ lib/publicNavItems.ts                      ← menu config (একমাত্র জায়গা)
└─ components/shared/Navbar/
   ├─ Navbar.tsx        (server, মূল কাঠামো)
   ├─ NavLinks.tsx      (client, active link)
   ├─ MobileNav.tsx     (client, mobile Sheet)
   └─ NavbarAuth.tsx    (async server, avatar বা Login)
```

> ✏️ নোটে tree-তে `src/` দুবার ছিল (`src/src/components/...`) — ঠিক করা হয়েছে।

| file | কোথায় চলে | কেন |
|---|---|---|
| `Navbar.tsx` | server | শুধু কাঠামো, কোনো state নেই → JS bundle-এ যায় না |
| `NavLinks.tsx` | client | `usePathname()` (কোন page খোলা) একটা hook |
| `MobileNav.tsx` | client | Sheet খোলা/বন্ধ `useState` |
| `NavbarAuth.tsx` | async server | cookie পড়ে user বের করে |

---

## ধাপ ১: `src/lib/publicNavItems.ts`

```ts
export interface PublicNavItem {
	title: string;
	href: string;
}

// নতুন menu লাগলে শুধু এখানে এক লাইন যোগ করো, desktop আর mobile দুটোতেই চলে আসবে
export const publicNavItems: PublicNavItem[] = [
	{ title: "Home", href: "/" },
	{ title: "Consultation", href: "/consultation" },
	{ title: "Health Plans", href: "/health-plans" },
	{ title: "Medicine", href: "/medicine" },
	{ title: "Diagnostics", href: "/diagnostics" },
	{ title: "NGOs", href: "/ngos" },
];
```

- dashboard-এর `navItems.ts`-এর (note 9) মতোই — menu data আর UI আলাদা।

---

## ধাপ ২: `NavLinks.tsx`

```tsx
"use client";

import type { PublicNavItem } from "@/lib/publicNavItems";
import { cn } from "@/lib/utils";
import Link from "next/link";
import { usePathname } from "next/navigation";

interface NavLinksProps {
	navItems: PublicNavItem[];
	onNavigate?: () => void; // mobile Sheet বন্ধ করার জন্য
	className?: string;
}

// "/" শুধু হুবহু মিললে active, বাকিগুলো sub-page-এও (যেমন /consultation/doctor/123)
const isActiveLink = (pathname: string, href: string) =>
	href === "/" ? pathname === "/" : pathname === href || pathname.startsWith(`${href}/`);

export default function NavLinks({ navItems, onNavigate, className }: NavLinksProps) {
	const pathname = usePathname();

	return (
		<ul className={cn("flex items-center gap-1", className)}>
			{navItems.map((item) => {
				const isActive = isActiveLink(pathname, item.href);

				return (
					<li key={item.href}>
						<Link
							href={item.href}
							onClick={onNavigate}
							aria-current={isActive ? "page" : undefined}
							className={cn(
								"block rounded-md px-3 py-2 text-sm font-medium transition-colors",
								isActive
									? "bg-primary/10 text-primary"
									: "text-muted-foreground hover:bg-accent hover:text-foreground",
							)}
						>
							{item.title}
						</Link>
					</li>
				);
			})}
		</ul>
	);
}
```

### ভেঙে বোঝা

**`isActiveLink` — কোন link active**

| pathname | Home (`/`) | Consultation (`/consultation`) |
|---|---|---|
| `/` | ✅ | ❌ |
| `/consultation` | ❌ | ✅ |
| `/consultation/doctor/123` | ❌ | ✅ (sub-page) |
| `/consultationxyz` | ❌ | ❌ (`${href}/` চেক, তাই ভুল মিল হয় না) |

- `/` আলাদা কারণ সব path-ই `/` দিয়ে শুরু — `startsWith` দিলে Home সবসময় active থাকত।
- `aria-current="page"` → screen reader বলে "current page"।
- `onNavigate` optional — desktop-এ লাগে না, mobile-এ Sheet বন্ধ করে।
- `className` বাইরে থেকে → একই component desktop-এ পাশাপাশি, mobile-এ খাড়া।

---

## ধাপ ৩: `MobileNav.tsx`

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { Sheet, SheetContent, SheetDescription, SheetTitle, SheetTrigger } from "@/components/ui/sheet";
import type { PublicNavItem } from "@/lib/publicNavItems";
import { Menu } from "lucide-react";
import Link from "next/link";
import { useState } from "react";
import NavLinks from "./NavLinks";

export default function MobileNav({ navItems }: { navItems: PublicNavItem[] }) {
	const [open, setOpen] = useState(false);
	const closeMenu = () => setOpen(false);

	return (
		<Sheet open={open} onOpenChange={setOpen}>
			<SheetTrigger asChild>
				<Button variant="ghost" size="icon" className="md:hidden" aria-label="Open menu">
					<Menu />
				</Button>
			</SheetTrigger>

			<SheetContent side="left" className="w-72 gap-0 p-0">
				<SheetTitle className="sr-only">Main menu</SheetTitle>
				<SheetDescription className="sr-only">Site pages</SheetDescription>

				<div className="flex h-16 items-center border-b px-6">
					<Link href="/" onClick={closeMenu} className="text-xl font-bold text-primary">
						Phdoc
					</Link>
				</div>

				<nav aria-label="Mobile navigation" className="p-3">
					{/* একই NavLinks, শুধু খাড়া করে সাজানো */}
					<NavLinks navItems={navItems} onNavigate={closeMenu} className="flex-col items-stretch" />
				</nav>
			</SheetContent>
		</Sheet>
	);
}
```

### ভেঙে বোঝা

- **controlled Sheet (`open` + `onOpenChange`)** — নিজে state রাখলে link-এ ক্লিক করে বন্ধ করা যায়। না রাখলে page বদলায় কিন্তু Sheet খোলাই থাকে (client navigation-এ component unmount হয় না)।
- **`md:hidden`** → menu button শুধু ছোট screen-এ।
- **`SheetTitle` + `SheetDescription` (`sr-only`)** — Radix dialog-এ title/description না দিলে console-এ accessibility warning আসে; চোখে দেখা যায় না, screen reader পড়ে।
- **`side="left"`** → বাম দিক থেকে আসে (menu button-ও বামে)।
- **`items-stretch`** → খাড়া list-এ প্রতিটা link পুরো চওড়া, ক্লিক করা সহজ।
- ডানের avatar/Login header-এই থাকে, তাই Sheet-এ আলাদা করে রাখা লাগেনি।

---

## ধাপ ৪: `NavbarAuth.tsx`

```tsx
import UserDropdown from "@/components/modules/Dashboard/UserDropdown";
import { Button } from "@/components/ui/button";
import { getDefaultDashboardRoute } from "@/lib/authUtils";
import { getSessionUser } from "@/lib/session";
import Link from "next/link";

// cookie পড়ে, তাই Navbar-এ এটা Suspense-এর ভেতরে রাখা হয়েছে
export default async function NavbarAuth() {
	const user = await getSessionUser();

	if (!user) {
		return (
			<Button asChild>
				<Link href="/login">Login</Link>
			</Button>
		);
	}

	return <UserDropdown user={user} dashboardHome={getDefaultDashboardRoute(user.role)} />;
}
```

- `getSessionUser` `cache()`-এ wrap করা — একই request-এ page (যেমন doctor page, note 25) আবার call করলেও একবারই চলে।
- `getDefaultDashboardRoute(role)` → role অনুযায়ী dashboard (admin → `/admin/dashboard`, doctor → `/doctor/dashboard`, patient → `/dashboard`)।
- async server component → client-এ কোনো fetch/loading state লাগে না।

---

## ধাপ ৫: `Navbar.tsx`

```tsx
import { Skeleton } from "@/components/ui/skeleton";
import { publicNavItems } from "@/lib/publicNavItems";
import Link from "next/link";
import { Suspense } from "react";
import MobileNav from "./MobileNav";
import NavbarAuth from "./NavbarAuth";
import NavLinks from "./NavLinks";

export default function Navbar() {
	return (
		<header className="sticky top-0 z-50 border-b bg-background/80 backdrop-blur">
			<div className="container mx-auto flex h-16 items-center gap-2 px-4">
				<MobileNav navItems={publicNavItems} />

				{/* বাম পাশে logo */}
				<Link href="/" className="text-xl font-bold text-primary">
					Phdoc
				</Link>

				{/* ডান পাশে menu + user */}
				<div className="ml-auto flex items-center gap-4">
					<nav aria-label="Main navigation" className="hidden md:block">
						<NavLinks navItems={publicNavItems} />
					</nav>

					{/* cookie পড়া অংশটা শুধু এখানে dynamic, বাকি header static থাকে */}
					<Suspense fallback={<Skeleton className="size-9 rounded-full" />}>
						<NavbarAuth />
					</Suspense>
				</div>
			</div>
		</header>
	);
}
```

### ভেঙে বোঝা

- **`sticky top-0 z-50`** → scroll করলেও উপরে থাকে, বাকি content-এর উপরে। (doctor page-এর booking card `top-20` = এই `h-16` header-এর নিচে।)
- **`bg-background/80 backdrop-blur`** → আধা-স্বচ্ছ, পেছনের content ঝাপসা দেখায়।
- **`ml-auto`** → menu + user ডান দিকে ঠেলে দেয়।
- **`hidden md:block`** → desktop menu ছোট screen-এ লুকানো (তখন MobileNav)।
- **দুটো `<nav>`-এ আলাদা `aria-label`** (Main / Mobile) → screen reader আলাদা করে চেনে।
- **`Suspense` শুধু `NavbarAuth`-এর চারপাশে** — `cacheComponents: true`-তে cookie পড়া অংশ Suspense-এ থাকতে হয়। তাই logo আর menu সাথে সাথে (static shell) আসে, শুধু avatar-এর জায়গায় গোল skeleton, তারপর আসল avatar/Login।

---

## ধাপ ৬: `src/app/(commonLayout)/layout.tsx`

```tsx
import Navbar from "@/components/shared/Navbar/Navbar";

export default function CommonLayout({ children }: LayoutProps<"/">) {
	return (
		<>
			<Navbar />
			<main className="flex-1">{children}</main>
		</>
	);
}
```

> ✏️ নোটে import ছিল `@/components/modules/Home/Navbar/Navbar`, কিন্তু file আছে `components/shared/Navbar/` — import ঠিক করা হয়েছে (project-এর সাথে মিলিয়ে)। project-এ এখন `<Footer />`-ও আছে (`@/components/shared/Footer/Footer`), সেটা আলাদা note-এ।

- `main`-এ `flex-1` → root layout-এর `<body className="min-h-full flex flex-col">`-এর সাথে মিলে content কম হলেও footer নিচে থাকে।
- `<main>` landmark → screen reader সরাসরি মূল content-এ যেতে পারে।

---

## ধাপ ৭: `src/components/modules/Dashboard/UserDropdown.tsx`

> ✏️ নোটে path ছিল `src/app/components/modules/dashboard/...` — আসল path `src/components/modules/Dashboard/UserDropdown.tsx`।

```tsx
"use client";

import { logoutAction } from "@/actions/auth.actions";
import { Avatar, AvatarFallback } from "@/components/ui/avatar";
import { Button } from "@/components/ui/button";
import {
	DropdownMenu,
	DropdownMenuContent,
	DropdownMenuItem,
	DropdownMenuLabel,
	DropdownMenuSeparator,
	DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import { formatRole } from "@/lib/authUtils";
import type { ISessionUser } from "@/types/auth.types";
import { useQueryClient } from "@tanstack/react-query";
import { KeyRound, LayoutDashboard, LoaderCircle, LogOut, User } from "lucide-react";

import Link from "next/link";
import { useTransition } from "react";

interface UserDropdownProps {
	user: ISessionUser;
	dashboardHome?: string;
}

export default function UserDropdown({ user, dashboardHome }: UserDropdownProps) {
	const queryClient = useQueryClient();
	const [isLoggingOut, startTransition] = useTransition();

	const handleLogout = (event: Event) => {
		event.preventDefault(); // menu খোলা রেখে "Logging out..." দেখানো
		startTransition(async () => {
			queryClient.clear(); // আগের user-এর cached data মুছে ফেলা
			await logoutAction();
		});
	};

	return (
		<DropdownMenu>
			<DropdownMenuTrigger asChild>
				<Button variant="ghost" className="h-auto gap-2 rounded-full p-1 md:pr-3" aria-label="Open user menu">
					<Avatar>
						<AvatarFallback className="bg-primary/10 text-sm font-semibold text-primary">
							{user.name.charAt(0).toUpperCase()}
						</AvatarFallback>
					</Avatar>
					<span className="hidden max-w-32 truncate text-sm font-medium md:block">{user.name}</span>
				</Button>
			</DropdownMenuTrigger>

			<DropdownMenuContent align="end" className="w-56">
				<DropdownMenuLabel className="space-y-1">
					<p className="truncate text-sm font-medium">{user.name}</p>
					<p className="truncate text-xs font-normal text-muted-foreground">{user.email}</p>
					<p className="text-xs font-normal capitalize text-primary">{formatRole(user.role)}</p>
				</DropdownMenuLabel>

				<DropdownMenuSeparator />

				{/* asChild: পুরো item-টাই link, keyboard-এ Enter দিলেও কাজ করে */}
				{dashboardHome && (
					<DropdownMenuItem asChild>
						<Link href={dashboardHome}>
							<LayoutDashboard />
							Dashboard
						</Link>
					</DropdownMenuItem>
				)}
				<DropdownMenuItem asChild>
					<Link href="/my-profile">
						<User />
						My Profile
					</Link>
				</DropdownMenuItem>

				<DropdownMenuItem asChild>
					<Link href="/change-password">
						<KeyRound />
						Change Password
					</Link>
				</DropdownMenuItem>

				<DropdownMenuSeparator />

				<DropdownMenuItem variant="destructive" disabled={isLoggingOut} onSelect={handleLogout}>
					{isLoggingOut ? <LoaderCircle className="animate-spin" /> : <LogOut />}
					{isLoggingOut ? "Logging out..." : "Logout"}
				</DropdownMenuItem>
			</DropdownMenuContent>
		</DropdownMenu>
	);
}
```

### আগের (note 11) থেকে কী নতুন

| নতুন | কেন |
|---|---|
| `dashboardHome?` prop + "Dashboard" item | public header থেকে নিজের dashboard-এ যাওয়া; dashboard-এর ভেতরে না দিলে item দেখায় না |
| avatar-এর পাশে নাম (`hidden md:block`) | desktop-এ কে login আছে চোখে পড়ে; mobile-এ শুধু avatar (জায়গা কম) |
| `aria-label="Open user menu"` | mobile-এ শুধু অক্ষর দেখায়, তাই button-এর নাম দরকার |

- **`DropdownMenuItem asChild` + `Link`** → item-টাই আসল `<a>`; keyboard, middle-click (নতুন tab) সব কাজ করে।
- **Logout:** `event.preventDefault()` → menu বন্ধ হয় না, "Logging out..." দেখায়। `useTransition` → pending state। `queryClient.clear()` → আগের user-এর cache (যেমন তার appointment) পরের user দেখে না।
- `formatRole("SUPER_ADMIN")` → `"super admin"`, `capitalize` → "Super Admin"।

---

## 🧪 Test (browser-এ)

| কী করবে | ফল |
|---|---|
| `/consultation/doctor/<id>` খোলো | "Consultation" link active (sub-page) |
| `/` খোলো | শুধু "Home" active |
| login ছাড়া | ডানে "Login" button |
| login করে | avatar + নাম (desktop), dropdown-এ নাম, email, role, Dashboard |
| PATIENT-এ "Dashboard" | `/dashboard`; DOCTOR-এ `/doctor/dashboard` |
| mobile width (< 768px) | desktop menu লুকানো, বামে ☰ button |
| ☰ → একটা link | page বদলায় আর Sheet বন্ধ হয় |
| Tab key দিয়ে ঘোরা | link আর dropdown keyboard-এ চলে |
| hard reload | logo/menu সাথে সাথে, avatar-এর জায়গায় এক মুহূর্ত skeleton |
| scroll | header উপরে আটকে থাকে, পেছন ঝাপসা |

---

## ✅ নতুন project-এ implement করার checklist

- [ ] menu এক config file-এ; desktop আর mobile একই `NavLinks`
- [ ] active link: `/` হুবহু, বাকিগুলো `startsWith(href + "/")`, `aria-current`
- [ ] mobile Sheet controlled, link-এ ক্লিকে বন্ধ; `SheetTitle` (sr-only)
- [ ] cookie পড়া অংশ (avatar/Login) আলাদা async component, ছোট `Suspense`-এ
- [ ] logout-এ query cache clear

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Header আর page-এর চওড়া মেলানো** — header-এ `container` (Tailwind v4-এ বড় screen-এ max 1536px), কিন্তু consultation/doctor page-এ `max-w-7xl` (1280px)। বড় monitor-এ logo আর page-এর content এক লাইনে থাকে না। দুই জায়গায় একই দাও:
   ```tsx
   <div className="mx-auto flex h-16 w-full max-w-7xl items-center gap-2 px-4">
   ```

2. **Skeleton-এর মাপ** — fallback গোল `size-9`, কিন্তু login না থাকলে আসে চওড়া "Login" button, আর login থাকলে desktop-এ avatar + নাম। জায়গা বদলে menu একটু সরে যায় (layout shift)। ছোট বিষয়; চাইলে `w-20 h-9 rounded-md` এর মতো মাঝামাঝি মাপ দাও।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

3. **"Skip to content" link** — keyboard user-কে প্রতি page-এ ৬টা menu link পার হতে হয়। header-এর শুরুতে লুকানো link দিলে এক Tab-এ মূল content-এ যাওয়া যায়:
   ```tsx
   <a href="#main" className="sr-only focus:not-sr-only focus:absolute focus:left-4 focus:top-4 focus:z-50 focus:rounded-md focus:bg-background focus:px-3 focus:py-2">
   	Skip to content
   </a>
   // layout-এ: <main id="main" className="flex-1">
   ```

4. **Avatar-এ ছবি** — এখন শুধু নামের প্রথম অক্ষর। `ISessionUser`-এ ছবি যোগ হলে `AvatarImage` দেওয়া যাবে (doctor profile-এ যেমন আছে, note 24)।

5. **Logout fail হলে** — `queryClient.clear()` আগে হয়, তাই `logoutAction` fail করলে cache মুছে যায় কিন্তু user login-ই থাকে। ক্ষতি নেই (data আবার আসবে), শুধু জেনে রাখো।

---

## 🔜 পরে যা যোগ হবে

- [ ] Footer (project-এ আছে, আলাদা note)
- [ ] Health Plans / Medicine / Diagnostics / NGOs page-এর আসল content
- [ ] dark mode toggle (header-এ)
