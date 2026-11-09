# 11 - Logout Action ও User Dropdown

> Note 10-এর dashboard navbar-এর ডান কোণে এখন একটা **user dropdown**: নাম, email, role, My Profile, Change Password আর **Logout**। Logout একটা **Server Action**, যেটা backend-এর session মোছে আর browser-এর ৩টা cookie মুছে `/login`-এ পাঠায়।

## 🧭 এক নজরে

```
Avatar চাপো → Dropdown খোলে
  → "Logout" চাপো (UserDropdown, client)
      → event.preventDefault()  → menu খোলা থাকে, "Logging out..." + spinner
      → queryClient.clear()     → আগের user-এর TanStack Query cache মোছা
      → logoutAction()  ── Server Action (server-এ চলে) ──┐
                                                          ▼
              session cookie আছে? → POST /auth/logout (backend DB থেকে session মোছে)
              (backend fail হলেও থামবে না)
              → clearAuthCookies()  → ৩টা cookie মোছা
              → redirect("/login")
```

### কোন file-এ কী

| File | কী করা হলো |
|---|---|
| `actions/auth.actions.ts` | শেষে `logoutAction` যোগ |
| `lib/authUtils.ts` | শেষে `formatRole` helper যোগ (সবাই এখান থেকে নেবে) |
| `Dashboard/DashboardUserInfo.tsx` | নিজের `formatRole` মুছে `authUtils` থেকে import |
| `Dashboard/UserDropdown.tsx` | **নতুন** — avatar + dropdown menu + logout |
| `Dashboard/DashboardNavbar.tsx` | ডান কোণে `UserDropdown` বসানো |

---

## ধাপ ০: shadcn `dropdown-menu` install

> ✏️ নোটে ছিল না: `UserDropdown`-এ `dropdown-menu` component লাগে। Project-এ আছে; নতুন project-এ না থাকলে:

```shell
pnpm dlx shadcn@latest add dropdown-menu
```
👉 https://ui.shadcn.com/docs/components/dropdown-menu

---

## ধাপ ১: `src/actions/auth.actions.ts` — শেষে `logoutAction` যোগ

```ts
import { AUTH_COOKIES } from "@/lib/cookieConfig";
import { clearAuthCookies, getCookie } from "@/lib/cookieUtils";


export async function logoutAction(): Promise<void> {
    const sessionToken = await getCookie(AUTH_COOKIES.sessionToken);

    if (sessionToken) {
        try {
            // backend DB থেকে session মুছে দেয় → token চুরি হলেও আর কাজ করবে না
            await httpClient.post("/auth/logout", undefined, {
                headers: { Cookie: `${AUTH_COOKIES.sessionToken}=${sessionToken}` },
            });
        } catch {
            // backend fail হলেও (httpClient নিজেই log করে) নিজের দিকের cookie মুছে logout করাতেই হবে
        }
    }

    await clearAuthCookies();
    redirect("/login");
}
```

> ✏️ নোটে import-এর তালিকায় ছিল না, কিন্তু এই file-এ আগে থেকেই আছে (note 07): `import { httpClient } from "@/lib/axios/httpClient";` আর `import { redirect } from "next/navigation";`। File-এর একদম উপরে `"use server";` আছে বলেই এটা Server Action।

### ধাপে ধাপে

| # | লাইন | কেন |
|---|---|---|
| ১ | `getCookie(AUTH_COOKIES.sessionToken)` | Backend-এর `/auth/logout` (server note 19) **`better-auth.session_token` cookie** থেকে session চেনে |
| ২ | `if (sessionToken)` | cookie না থাকলে backend-এ পাঠানোর কিছু নেই (backend 401 দিত) |
| ৩ | `headers: { Cookie: ... }` | Server Action server-এ চলে, browser না — তাই browser-এর cookie নিজে থেকে যায় না। হাতে `Cookie` header বানিয়ে পাঠাতে হয় (proxy-র refresh call-এর মতো, note 08) |
| ৪ | `httpClient.post(url, undefined, {...})` | body নেই → দ্বিতীয় argument `undefined`, তৃতীয়টায় options |
| ৫ | খালি `catch {}` | Backend down থাকলেও user-কে logout করাতেই হবে। `httpClient` নিজেই error log করে (note 04), তাই এখানে আবার log লাগে না |
| ৬ | `clearAuthCookies()` | note 07-এর function → ৩টা cookie একসাথে মোছে (🔄 note 28: এখন cookie-র `domain` দিয়েই মোছে, নইলে production-এ logout কাজ করত না) |
| ৭ | `redirect("/login")` | try/catch-এর **বাইরে**, কারণ `redirect()` ভিতরে একটা বিশেষ error throw করে — catch-এর ভিতরে থাকলে ধরা পড়ে যেত (note 07-এর login action-এও একই নিয়ম) |

### Backend-এ মোছা কেন জরুরি, শুধু cookie মুছলে হয় না?

| শুধু cookie মোছা | Backend-এও session মোছা |
|---|---|
| এই browser থেকে logout | এই browser থেকে logout |
| কেউ আগে token কপি করে থাকলে (চুরি) সে এখনো ব্যবহার করতে পারবে — session DB-তে আছে | DB-তে session নেই → চুরি হওয়া token-ও অকেজো (backend-এর `checkAuth` প্রতিটা request-এ session check করে) |

### CSRF থেকে সুরক্ষা
এটা Server Action হওয়ায় Next.js নিজেই CSRF থেকে রক্ষা করে: অন্য site থেকে কেউ এই action call করে তোমাকে জোর করে logout করাতে পারবে না। (Server Action শুধু `POST`-এ চলে, আর Next.js request-এর `Origin` header নিজের site-এর সাথে মিলিয়ে দেখে।)

---

## ধাপ ২: `src/lib/authUtils.ts` — শেষে `formatRole` helper

```ts
// "SUPER_ADMIN" → "super admin" (CSS capitalize দিয়ে দেখাবে "Super Admin")
export const formatRole = (role: UserRole) => role.toLowerCase().replaceAll("_", " ");
```

তারপর `DashboardUserInfo.tsx`-এ নিজের `formatRole` মুছে এটা import করো:

```tsx
import { formatRole } from "@/lib/authUtils";
```

- আগে `formatRole` শুধু `DashboardUserInfo`-এর ভিতরে ছিল। এখন `UserDropdown`-এও লাগে → **এক জায়গায় রেখে দুই জায়গায় import** (একই কোড দুবার লিখলে একটা বদলে আরেকটা ভুলে যাওয়ার সুযোগ থাকে)।
- `replace` → **`replaceAll`**: সব `_` বদলায় (note 10-এর recommendation ৪ ✅)।
- `authUtils.ts`-এ রাখার কারণ: এখানে আগে থেকেই role-সংক্রান্ত সব helper আছে, আর এই file-এ `server-only` নেই (client component-ও import করতে পারে)।

---

## ধাপ ৩: `src/components/modules/Dashboard/UserDropdown.tsx` (নতুন)

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
import { KeyRound, LoaderCircle, LogOut, User } from "lucide-react";
import Link from "next/link";
import { useTransition } from "react";

export default function UserDropdown({ user }: { user: ISessionUser }) {
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

### কেন `"use client"`?
Dropdown খোলা/বন্ধ, `useTransition`, `useQueryClient` — সব hook, তাই client component।

### Client component থেকে Server Action call করা যায়?
হ্যাঁ। `"use server"` file-এর function client component-এ import করলে Next.js আসলে function-টা পাঠায় না, একটা **reference** পাঠায়। Call করলে browser থেকে server-এ একটা `POST` request যায়, আর কাজটা server-এ হয় (cookie মোছা, backend call)।

### `handleLogout` — ভেঙে বোঝা

```ts
const handleLogout = (event: Event) => {
	event.preventDefault();
	startTransition(async () => {
		queryClient.clear();
		await logoutAction();
	});
};
```

| অংশ | কেন |
|---|---|
| `onSelect` (onClick না) | Radix-এর menu item-এ `onSelect` mouse click **আর** keyboard (Enter/Space) দুটোই ধরে |
| `event: Event` | `onSelect` সাধারণ DOM `Event` দেয় (React-এর `MouseEvent` না) |
| `event.preventDefault()` | Item select হলে Radix নিজে menu বন্ধ করে দেয়। আটকালে menu খোলা থাকে, user "Logging out..." দেখতে পায় |
| `useTransition` → `isLoggingOut` | Action চলাকালীন `true` → item `disabled` + spinner। দুবার চাপলে দুবার logout হবে না। Redirect শেষ না হওয়া পর্যন্ত `true` থাকে (note 07-এর login form-এর মতো) |
| `queryClient.clear()` | TanStack Query-র cache browser-এর memory-তে থাকে। না মুছলে একই browser-এ অন্য কেউ login করলে এক মুহূর্তের জন্য **আগের user-এর data** দেখতে পারত |
| `await logoutAction()` | Server-এ logout, তারপর `redirect` → Next.js নিজেই `/login`-এ নিয়ে যায় |

### বাকি অংশ

| অংশ | কাজ |
|---|---|
| `DropdownMenuTrigger asChild` + `Button` | আমাদের `Button`-কেই trigger বানায় |
| `aria-label="Open user menu"` | mobile-এ শুধু avatar দেখা যায় (লেখা নেই), তাই screen reader-এর জন্য নাম |
| `hidden ... md:block` (নাম) | Mobile-এ শুধু avatar, desktop-এ avatar + নাম |
| `max-w-32 truncate` | লম্বা নাম `...` হয়ে যাবে |
| `align="end"` | Menu trigger-এর ডান কিনারা বরাবর খোলে → screen-এর বাইরে যায় না |
| `DropdownMenuLabel` | উপরে নাম, email, role (click করা যায় না, শুধু তথ্য) |
| `DropdownMenuItem asChild` + `Link` | পুরো item-টাই link → keyboard-এ Enter দিলেও page বদলায়, আর link-এ চাপলে menu নিজেই বন্ধ হয় |
| `variant="destructive"` | Logout লাল রঙে (বিপজ্জনক কাজ বোঝাতে) |
| `LoaderCircle` + `animate-spin` | ঘুরতে থাকা spinner |

---

## ধাপ ৪: `DashboardNavbar.tsx`-এ বসাও

```tsx
import UserDropdown from "./UserDropdown";

<div className="ml-auto flex items-center gap-3">
    <NotificationDropdown />
    <UserDropdown user={user} />
</div>
```

- আগে navbar-এর ডানে ছিল `<DashboardUserInfo user={user} />`। এখন সেই জায়গায় **`UserDropdown`**। ফলে desktop-এ user info আর দুবার দেখা যায় না: sidebar-এর নিচে শুধু তথ্য, আর navbar-এ dropdown (note 10-এর recommendation ১ ✅)।
- Mobile Sheet-এর ভিতরের নিচের `DashboardUserInfo` আগের মতোই থাকে।
- `flex items-center gap-3` → notification আর user dropdown পাশাপাশি।

> ✏️ `NotificationDropdown` এই নোটে বানানো হয়নি — এটা আলাদা component (`Dashboard/NotificationDropdown.tsx`, এখন mock data দিয়ে চলে)। নতুন project-এ ওটা না বানিয়ে থাকলে এই লাইনটা বাদ দাও, শুধু `<UserDropdown user={user} />` রাখো।

---

## 📐 Navbar এখন

```
Desktop:  [                                   🔔  (A) Arif ▾ ]
Mobile:   [ ☰  Phdoc                           🔔  (A) ▾     ]

(A) Arif ▾ চাপলে:
┌──────────────────────────┐
│ Arif                     │
│ arif@mail.com            │
│ Admin                    │
├──────────────────────────┤
│ 👤 My Profile            │
│ 🔑 Change Password       │
├──────────────────────────┤
│ ⎋ Logout          (লাল) │
└──────────────────────────┘
```

---

## 🧪 Test

| # | কী করবে | কী হওয়ার কথা |
|---|---|---|
| ১ | Dashboard-এ avatar চাপো | Dropdown: নাম, email, role, ২টা link, Logout |
| ২ | Keyboard: avatar-এ Tab → Enter → ↓ ↓ → Enter | Menu খোলে, item বদলায়, link-এ যায় |
| ৩ | My Profile চাপো | `/my-profile`, menu বন্ধ |
| ৪ | Logout চাপো | Menu খোলা থাকে, "Logging out..." + spinner → `/login` |
| ৫ | DevTools → Application → Cookies | `accessToken`, `refreshToken`, `better-auth.session_token` তিনটাই নেই |
| ৬ | Backend DB (`pnpm studio`) → `session` table | ওই session-এর row নেই |
| ৭ | Logout-এর পর URL-এ `/admin/dashboard` লেখো | proxy `/login?redirect=...`-এ পাঠায় |
| ৮ | Backend বন্ধ করে logout | তবুও cookie মুছে `/login` (terminal-এ `[httpClient] POST /auth/logout failed` log) |
| ৯ | Mobile view-এ avatar চাপো | নাম ছাড়া শুধু avatar, menu ঠিকমতো খোলে |

---

## 🔜 পরে যা যোগ হবে
- [ ] "সব device থেকে logout" (`POST /auth/logout-all`, server note 19) — Change Password page বা settings-এ
- [ ] My Profile আর Change Password page
- [ ] Notification backend API + TanStack Query (এখন mock)
- [ ] User-এর ছবি থাকলে `AvatarImage`

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Avatar-এর কোড দুই জায়গায় একই** — `UserDropdown` আর `DashboardUserInfo` দুটোতেই একই `Avatar` + `AvatarFallback` + `charAt(0)`। পরে ছবি (`AvatarImage`) যোগ করলে দুই জায়গায় বদলাতে হবে। একটা ছোট component বানিয়ে দুই জায়গায় ব্যবহার করলে সুবিধা:
   ```tsx
   // src/components/modules/Dashboard/UserAvatar.tsx
   import { Avatar, AvatarFallback } from "@/components/ui/avatar";

   export default function UserAvatar({ name }: { name: string }) {
   	return (
   		<Avatar>
   			<AvatarFallback className="bg-primary/10 text-sm font-semibold text-primary">
   				{name.charAt(0).toUpperCase()}
   			</AvatarFallback>
   		</Avatar>
   	);
   }
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

2. **Logout-এর পর browser-এর Back button** — Next.js আগের দেখা page memory-তে (router cache) রাখে, তাই logout করে Back চাপলে কখনো কখনো dashboard-এর পুরনো UI এক ঝলক দেখা যেতে পারে (untested)। নতুন কোনো data আসবে না (cookie নেই, backend আটকাবে), আর reload বা অন্য link-এ গেলেই proxy `/login`-এ পাঠাবে। তাই নিরাপত্তার সমস্যা না, শুধু দেখতে।

3. **`logoutAction` একটা public endpoint** — `"use server"` file-এর প্রতিটা export করা function বাইরে থেকে call করা যায়। Logout-এর ক্ষেত্রে এটা নিরাপদ (নিজের cookie-ই মোছে, অন্যের কিছু করতে পারে না)। কিন্তু পরে যেকোনো action-এ (যেমন delete doctor) **action-এর ভিতরেই** check রাখবে, শুধু UI-তে button লুকানো যথেষ্ট না।
