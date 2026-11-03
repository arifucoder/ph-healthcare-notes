# 02 - Project Structure, Routing ও Layout

> Module: 44-03, 44-05

```
Route plan (public / auth / private) → route group দিয়ে folder সাজানো → page.tsx → layout.tsx
```

## ধাপ ১: Route plan — কোন page কে দেখতে পারবে

আমাদের ২ ধরনের route থাকবে: **public** আর **private**।

### 🌐 Public pages
যে কেউ যেকোনো সময় visit করতে পারবে।

| Page | Route |
|---|---|
| Home | `/` |
| Consultation | `/consultation` |
| Contact us | `/contact` |
| Policy | `/policy` |

### 🔑 Authentication pages
এগুলোও public, কিন্তু **special public page** — শুধু **unauthenticated** (login না করা) user এগুলো দেখতে পারবে। Login করার পর user login সহ বাকি এই page গুলোতে আর যেতে পারবে না।

```
/login
/register
/forget-password
/verify
/reset-password
```

### 🔒 Private pages
Private page-এ আবার ২ ধরনের protected page থাকবে:

**১) Common protected page** — login করা যেকোনো user (যেকোনো role):
1. change password
2. my profile

**২) Role based protected page**

| Role | Route | কাজ |
|---|---|---|
| ADMIN, SUPER_ADMIN | `/admin/dashboard` | statistical data |
| | `/admin/dashboard/admin-management` | |
| | `/admin/dashboard/schedule-management` | |
| | `/admin/dashboard/doctor-management` | |
| | `/admin/dashboard/appointment-management` | |
| DOCTOR | `/doctor/dashboard` | |
| | `/doctor/dashboard/appointments` | |
| | `/doctor/dashboard/my-schedules` | |
| | `/doctor/dashboard/prescriptions` | |
| PATIENT | `/dashboard` | |
| | `/dashboard/book-appointment` | |
| | `/dashboard/health-record` | |
| | `/dashboard/my-appointments` | |
| | `/dashboard/my-prescriptions` | |

## ধাপ ২: `src/app` folder structure (44-03 Routing Setup For Public Routes)

`src/app`-এর ভেতরে folder ও file গুলো:

```
src/app/
├── (commonLayout)/
├── (dashboardLayout)/
├── _actions/
│   ├── appointment.actions.ts
│   └── rag.action.ts
├── error.tsx
├── favicon.ico
├── globals.css
├── layout.tsx
├── loading.tsx
└── not-found.tsx
```

### Route group `()` কী?

Folder name `()` parenthesis-এর ভেতরে দিলে **URL-এ ওই folder name skip হয়ে যায়**। এটা শুধু সাজানোর জন্য, যাতে arrangement সুন্দর থাকে।

```
src/app/(commonLayout)/consultation/page.tsx   →   /consultation   ((commonLayout) URL-এ নেই)
```

| Folder | কী থাকবে |
|---|---|
| `(commonLayout)` | public page গুলো — এগুলোতে sidebar থাকবে না |
| `(dashboardLayout)` | protected page গুলো — এগুলোতে sidebar থাকতে পারে |

> `_` দিয়ে শুরু folder (যেমন `_actions`) = **private folder** — Next.js এটাকে route বানায় না।

## ধাপ ৩: `(commonLayout)`-এর ভেতরে auth ও public pages

`(commonLayout)`-এর ভেতরে `(authRouteGroup)` নামে একটা folder নিব:

```
(commonLayout)/
├── (authRouteGroup)/
│   ├── login/
│   ├── register/
│   ├── reset-password/
│   ├── verify-email/
│   └── forgot-password/
├── consultation/
├── diagnostics/
├── health-plans/
├── medicine/
└── ngos/
```

`(commonLayout)`-এর ভেতরে grouping ছাড়া page গুলো: `consultation`, `diagnostics`, `health-plans`, `medicine`, `ngos`।

**প্রত্যেকটা folder-এর জন্য `page.tsx` নামে file লাগবে** — `page.tsx` না থাকলে ওই route খুলবে না।

### Naming convention

Teacher page component-এর নামের শেষে `Page` রাখেন, যেমন login → `LoginPage`:

```tsx
export default function LoginPage() {
	return (
		<div>
			<h1>This is Login component</h1>
		</div>
	);
}
```

### Page-এ কখনো `"use client"` না

আমরা আমাদের page গুলোকে **server side rendering** করব। তাই page component-এর ভেতরে কখনোই `"use client"` directive ব্যবহার করব না। Client component লাগলে (যেমন form, button click, `useState`) আলাদা client component বানাব, তারপর সেটা page file-এ এনে বসাব।

```tsx
// src/components/modules/auth/LoginForm.tsx
"use client";
export default function LoginForm() { /* useState, onSubmit ... */ }

// src/app/(commonLayout)/(authRouteGroup)/login/page.tsx  (server component)
import LoginForm from "@/components/modules/auth/LoginForm";
export default function LoginPage() {
	return <LoginForm />;
}
```

> ✏️ উপরের উদাহরণটা Claude যোগ করেছে, pattern টা বোঝানোর জন্য — এটা নোটে ছিল না।

### 🆚 Server Side Rendering vs Client Side Rendering

| বিষয় | Server (Server Component / SSR) | Client (Client Component / CSR) |
|---|---|---|
| কোথায় render হয় | server-এ, browser পায় তৈরি HTML | browser-এ, JS download হওয়ার পর |
| প্রথম load | দ্রুত — HTML সাথে সাথে দেখা যায় | ধীর — JS load + data fetch শেষে দেখা যায় |
| SEO | ভালো — Google পুরো content দেখে | দুর্বল — শুরুতে খালি HTML |
| Browser-এ JS bundle | কম (component-এর code browser-এ যায় না) | বেশি |
| Data fetching | সরাসরি `async/await` দিয়ে, server থেকে | `useEffect` / TanStack Query দিয়ে |
| Secret (API key, token) | নিরাপদ — browser-এ যায় না | expose হওয়ার ঝুঁকি |
| Interactivity (`onClick`, `useState`, `useEffect`) | ❌ পারে না | ✅ পারে |
| Browser API (`window`, `localStorage`) | ❌ | ✅ |
| কখন ব্যবহার | page, layout, data দেখানো, static content | form, modal, dropdown, live UI |

**কোনটা ভালো?** কোনোটাই সবসময় ভালো না — দুটো মিলিয়ে ব্যবহার করাই best। Default server, যেখানে interaction লাগবে শুধু সেখানে client।

**কেন teacher page-এ client component রাখেন না — সুবিধা:**
1. **SEO ও দ্রুত প্রথম load** — page server-এ render হয়, তাই content সাথে সাথে দেখা যায় (healthcare site-এ doctor/consultation page Google-এ আসা দরকার)।
2. **JS bundle ছোট** — শুধু ছোট client অংশটুকুর (যেমন form) JS browser-এ যায়, পুরো page-এর না।
3. **Page-এ server feature ব্যবহার করা যায়** — `async` page, server থেকে সরাসরি data fetch, `metadata` export, cookie পড়া। `"use client"` দিলে এগুলো সব হারিয়ে যায় (client component-এ `metadata` export করা যায় না)।
4. **নিরাপত্তা** — token/secret server-এই থাকে।
5. **Clean structure** — page = data + layout, client component = interaction। কোন code কোথায় চলে বোঝা সহজ।

> 💡 ছোট্ট সংশোধন: Next.js-এ client component-ও প্রথমবার server-এ pre-render (SSR) হয়, পরে browser-এ hydrate হয়। আসল পার্থক্য হলো client component-এর JS browser-এ যায় আর browser-এ চলে, server component-এর JS যায় না।

## ধাপ ৪: Dynamic route — doctor profile

`/consultation/doctor/:id` দিয়ে doctor-এর profile দেখাব। তাই folder:

```
src/app/(commonLayout)/consultation/doctor/[id]/page.tsx
```

`[id]` = **dynamic segment** — URL-এর ওই অংশ যা খুশি হতে পারে (`/consultation/doctor/123`)।

```tsx
export default function ConsultationDoctorByIdPage() {
	return (
		<div>
			<h1>This is DoctorPage component</h1>
		</div>
	);
}
```

> 💡 পরে `id` পড়তে (Next.js 15+ এ `params` একটা Promise):
> ```tsx
> export default async function ConsultationDoctorByIdPage({ params }: PageProps<"/consultation/doctor/[id]">) {
> 	const { id } = await params;
> 	return <h1>Doctor: {id}</h1>;
> }
> ```

## ধাপ ৫: `(dashboardLayout)`-এর ভেতরে protected pages

```
(dashboardLayout)/
├── (commonProtectedLayout)/
│   ├── change-password/page.tsx
│   ├── my-profile/page.tsx
│   └── layout.tsx
├── admin/dashboard/
├── dashboard/
├── doctor/dashboard/
└── layout.tsx
```

### `admin/dashboard`-এর ভেতরে

```
admin/dashboard/
├── admins-management/            page.tsx, loading.tsx
├── appointments-management/      page.tsx, loading.tsx
├── doctor-schedules-managament/  page.tsx, loading.tsx
├── doctor-specialties-management/ page.tsx, loading.tsx
├── doctors-management/           _action.ts, page.tsx, loading.tsx
├── patients-management/          page.tsx, loading.tsx
├── payments-management/          page.tsx, loading.tsx
├── prescriptions-management/     page.tsx, loading.tsx
├── reviews-management/           page.tsx, loading.tsx
├── schedules-management/         page.tsx, loading.tsx
├── specialties-management/       page.tsx, loading.tsx
├── layout.tsx
├── loading.tsx
└── page.tsx
```

### `dashboard` (patient)-এর ভেতরে

```
dashboard/
├── appointments/page.tsx
├── book-appointments/page.tsx
└── my-appointments/page.tsx
```

### `doctor/dashboard`-এর ভেতরে

```
doctor/dashboard/
├── appointments/   page.tsx, loading.tsx
├── my-reviews/     page.tsx, loading.tsx
├── my-schedules/   _action.ts, page.tsx, loading.tsx
├── prescriptions/  page.tsx, loading.tsx
├── layout.tsx
├── loading.tsx
└── page.tsx
```

### Special file গুলোর কাজ

| File | কাজ |
|---|---|
| `page.tsx` | ওই route-এর UI (এটা থাকলেই route খোলে) |
| `layout.tsx` | ওই folder ও তার ভেতরের সব page-এর common wrapper (sidebar, navbar) |
| `loading.tsx` | page load/data fetch হওয়ার সময় দেখায় (skeleton/spinner) |
| `error.tsx` | error হলে দেখায় (এটা অবশ্যই `"use client"`) |
| `not-found.tsx` | 404 page |
| `_action.ts` | server action — route হয় না |

## ধাপ ৬: Layout setup (44-05 Layout Setup In The Project)

### Root layout

App-এর entry file হলো `src/app/layout.tsx`। এখানে `<html>` আর `<body>`-এর ভেতরে `children` থাকে। এটাকে **global layout / root layout** বলে। Browser থেকে যেকোনো page-এ ঢুকলে সেই page এই layout-এর ভেতরেই load হয় — মানে **এই layout সব page-এ load হয়**।

Layout গুলো nested হয়:

```
RootLayout → (dashboardLayout)/layout → admin/dashboard/layout → page
```

### Route group-এ page.tsx — conflict

ধরো `src/app/(commonLayout)`-এর ভেতরে `layout.tsx` আর `page.tsx` নিলাম। সাথে সাথে **error** দিবে, কারণ `src/app`-এর ভেতরে আগে থেকেই `page.tsx` আছে। Route group URL-এ কিছু যোগ করে না, তাই দুটো `page.tsx`-ই `/` route — একই level-এ চলে এসেছে।

```
src/app/page.tsx                  →  /
src/app/(commonLayout)/page.tsx   →  /     ❌ দুটো একই route
```

**সমাধান:** বাইরের `src/app/page.tsx` delete করে দিব। কিন্তু তখন আবার error দিবে, কারণ `src/app/(commonLayout)`-এর `layout.tsx` empty — layout file-এ default export করা component লাগবেই। তাই এটা add করতে হবে:

```tsx
export default function CommonLayout({ children }: LayoutProps<"/">) {
	return (
		<>
			<h1>This is CommonLayout component</h1>
			{children}
		</>
	);
}
```

> `LayoutProps<"/">` — Next.js-এর global type, generic-এ ওই layout-এর route দিতে হয় (যেমন admin-এ `LayoutProps<"/admin/dashboard">`)। Route group-এর layout `/`-তেই বসে, তাই `"/"`।

## 🔜 পরে যা যোগ হবে

- [ ] `(commonLayout)/layout.tsx`-এ Navbar + Footer
- [ ] `(dashboardLayout)/layout.tsx`-এ Sidebar (role অনুযায়ী menu)
- [ ] `proxy.ts` দিয়ে auth ও role based route protection
- [ ] বাকি folder-এ `page.tsx`, `loading.tsx` বানানো (`dashboard/`, `doctor/dashboard/` এখনো খালি)

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **Route plan আর folder name মিলছে না** — plan-এ `/forget-password`, `/verify`, কিন্তু folder `forgot-password`, `verify-email`। Plan-এ `/contact`, `/policy` আছে কিন্তু folder নেই, আর folder-এ `diagnostics`, `medicine`, `ngos` আছে যা plan-এ নেই। একটা নাম ঠিক করে সব জায়গায় একই রাখো, নাহলে পরে redirect/link ভুল route-এ যাবে।

2. **Route protection শুধু folder দিয়ে হয় না** — `(dashboardLayout)` নামে রাখলেই page protected হয় না। Next.js 16-এ `src/proxy.ts` (আগের `middleware.ts`) দিয়ে check করতে হবে:

   ```ts
   // src/proxy.ts
   import { NextResponse, type NextRequest } from "next/server";

   const authRoutes = ["/login", "/register", "/forgot-password", "/reset-password", "/verify-email"];

   export function proxy(req: NextRequest) {
   	const token = req.cookies.get("accessToken")?.value;
   	const { pathname } = req.nextUrl;

   	if (token && authRoutes.includes(pathname)) return NextResponse.redirect(new URL("/", req.url));
   	if (!token && !authRoutes.includes(pathname)) return NextResponse.redirect(new URL(`/login?redirect=${pathname}`, req.url));
   	// role check: token decode করে admin/doctor/patient route মিলিয়ে দেখা
   	return NextResponse.next();
   }

   export const config = {
   	matcher: ["/login", "/register", "/forgot-password", "/reset-password", "/verify-email",
   		"/dashboard/:path*", "/admin/:path*", "/doctor/:path*", "/my-profile", "/change-password"],
   };
   ```

### 🟡 ভালো practice

3. **Typo ঠিক করো** — `doctor-schedules-managament` → `doctor-schedules-management` (folder name-ই URL হয়ে যায়)। Plan-এ `appointmet-management` → `appointment-management`। নোটে `patients-management` দুবার লেখা ছিল, একটা রেখেছি।

4. **Patient dashboard-এ একই জিনিস দুবার** — `appointments` আর `my-appointments` দুটোই আছে, আর plan-এর `health-record`, `my-prescriptions` folder-এ নেই। `book-appointment` (plan) বনাম `book-appointments` (folder) — একটা ঠিক করো।

5. **Plan আর folder-এর মধ্যে schedule** — admin-এ `schedules-management` আর `doctor-schedules-managament` দুটো আছে। কোনটার কী কাজ, নোটে লিখে রাখো।

### 🔵 পরে কাজে লাগবে / জেনে রাখো

6. **Role → route একটা constant file-এ রাখো** — sidebar menu আর `proxy.ts` দুই জায়গায় একই list লাগবে:

   ```ts
   // src/constants/routes.ts
   export const roleBasedRoutes = {
   	SUPER_ADMIN: "/admin/dashboard",
   	ADMIN: "/admin/dashboard",
   	DOCTOR: "/doctor/dashboard",
   	PATIENT: "/dashboard",
   } as const;
   ```

7. **`(authRouteGroup)`-এর জন্য আলাদা `layout.tsx`** — login/register page-এ সাধারণত navbar লাগে না, মাঝখানে একটা card থাকে। তখন `(authRouteGroup)/layout.tsx` দিয়ে সেটা সব auth page-এ একবারে দেওয়া যায়।

## 🏗️ Claude-এর Suggested Best Layout Structure

তোমার এখনকার structure ঠিক আছে। নিচেরটা সেটার উপরেই বানানো, শুধু role অনুযায়ী layout আর `src/`-এর বাকি folder গুলো পরিষ্কার করা হয়েছে:

```
src/
├── app/
│   ├── (commonLayout)/                  # Navbar + Footer, sidebar নেই
│   │   ├── layout.tsx
│   │   ├── page.tsx                     # /  (home)
│   │   ├── (authRouteGroup)/            # শুধু unauthenticated user
│   │   │   ├── layout.tsx               # মাঝখানে card, navbar ছাড়া
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   ├── forgot-password/page.tsx
│   │   │   ├── reset-password/page.tsx
│   │   │   └── verify-email/page.tsx
│   │   ├── consultation/
│   │   │   ├── page.tsx
│   │   │   └── doctor/[id]/page.tsx
│   │   ├── contact/page.tsx
│   │   ├── policy/page.tsx
│   │   ├── diagnostics/page.tsx
│   │   ├── health-plans/page.tsx
│   │   ├── medicine/page.tsx
│   │   └── ngos/page.tsx
│   │
│   ├── (dashboardLayout)/               # Sidebar + Topbar
│   │   ├── layout.tsx
│   │   ├── (commonProtectedLayout)/     # যেকোনো logged-in user
│   │   │   ├── layout.tsx
│   │   │   ├── my-profile/page.tsx
│   │   │   └── change-password/page.tsx
│   │   ├── admin/dashboard/             # ADMIN, SUPER_ADMIN
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx  loading.tsx
│   │   │   ├── admins-management/
│   │   │   ├── doctors-management/
│   │   │   ├── patients-management/
│   │   │   ├── specialties-management/
│   │   │   ├── schedules-management/
│   │   │   ├── appointments-management/
│   │   │   ├── prescriptions-management/
│   │   │   ├── payments-management/
│   │   │   └── reviews-management/
│   │   ├── doctor/dashboard/            # DOCTOR
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx  loading.tsx
│   │   │   ├── appointments/
│   │   │   ├── my-schedules/
│   │   │   ├── prescriptions/
│   │   │   └── my-reviews/
│   │   └── dashboard/                   # PATIENT
│   │       ├── layout.tsx
│   │       ├── page.tsx  loading.tsx
│   │       ├── book-appointment/
│   │       ├── my-appointments/
│   │       ├── my-prescriptions/
│   │       └── health-record/
│   │
│   ├── layout.tsx                       # root: html, body, font, Providers
│   ├── error.tsx                        # "use client"
│   ├── loading.tsx
│   ├── not-found.tsx
│   └── globals.css
│
├── actions/                             # src/actions (app/-এর বাইরে, তাই route হয় না) — server actions ("use server")
│   ├── auth.actions.ts
│   └── appointment.actions.ts
├── components/
│   ├── ui/                              # shadcn (হাত দিয়ে edit কম)
│   ├── shared/                          # Navbar, Footer, Sidebar, Logo
│   └── modules/                         # feature অনুযায়ী client component
│       ├── auth/LoginForm.tsx
│       ├── doctor/DoctorCard.tsx
│       └── appointment/BookingForm.tsx
├── constants/routes.ts                  # role → route, sidebar menu
├── hooks/                               # custom hooks (useUser ...)
├── lib/
│   ├── utils.ts                         # cn()
│   ├── axios.ts                         # axios instance (baseURL)
│   └── query-client.ts
├── providers/QueryProvider.tsx          # "use client" — QueryClientProvider
├── services/                            # API call function (auth.service.ts ...)
├── types/                               # TypeScript types (IUser, IDoctor ...)
├── zod/                                 # form validation schema (auth.validation.ts ...)
└── proxy.ts                             # auth + role based route protection
```

**মূল নিয়ম গুলো:**

| নিয়ম | কেন |
|---|---|
| `app/`-এ শুধু route file (`page`, `layout`, `loading`, `error`) | কোন folder route, সেটা এক নজরে বোঝা যায় |
| page = server component, interaction = `components/modules/` | উপরের SSR সুবিধা গুলো পাওয়া যায় |
| প্রতিটা role-এর dashboard-এ নিজের `layout.tsx` | role অনুযায়ী আলাদা sidebar menu |
| `services/` (API call) আলাদা, component-এ সরাসরি axios না | একই API অনেক জায়গায় reuse, URL বদলালে এক জায়গায় বদলাতে হয় |
| route protection `proxy.ts`-এ, layout-এ না | page render হওয়ার আগেই redirect হয়ে যায় |

> ⚠️ খেয়াল করো: `actions/`, `components/`, `lib/` ইত্যাদি `app/`-এর **পাশে** (`src/`-এর ভেতরে), `app/`-এর ভেতরে না। শুধু `app/`-এর ভেতরের folder route হয়। আর `app/`-এর ভেতরেও কোনো folder তখনই route হয় যখন তাতে `page.tsx` বা `route.ts` থাকে — তবুও নিশ্চিত থাকতে ভেতরে রাখলে `_` দিয়ে নাম দাও (`_actions`)।
>
> `app/_actions` রাখাও ঠিক আছে (teacher-এর মতো) — `src/actions` শুধু `app/` folder টা route-only রাখার জন্য suggestion।
