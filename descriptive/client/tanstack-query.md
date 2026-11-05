# TanStack Query — সহজ ভাষায় (Teacher Part-8 অনুযায়ী)

👉 https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr#prefetching-and-de-hydrating-data

> এই নোটের লক্ষ্য: **বোঝা**। তাই code ছোট রাখা হয়েছে।
> Level 1-এ এক file-এ সব করে browser console-এ doctors দেখব, Level 2-এ teacher-এর মতো server-এ prefetch করব।

---

## ১. TanStack Query কেন লাগে?

সাধারণ React-এ data আনতে আমরা নিজে লিখি: `useState` (data) + `useState` (loading) + `useState` (error) + `useEffect` (fetch)।
প্রতিটা page-এ বারবার একই কাজ, আর data **cache** হয় না — page বদলে ফিরে এলে আবার fetch।

TanStack Query এই সব নিজে করে দেয়:

| তুমি দাও | TanStack Query দেয় |
|---|---|
| একটা নাম (`queryKey`) | `data`, `isLoading`, `isError` |
| data আনার function (`queryFn`) | cache, auto refetch, একই data দুইবার fetch না করা |

> সহজ উপমা: **QueryClient = একটা আলমারি (cache)**। `queryKey` = ড্রয়ারের নাম। `queryFn` = দোকান থেকে জিনিস আনার লোক।
> `useQuery` আগে ড্রয়ারে দেখে — থাকলে সেখান থেকে দেয়, না থাকলে লোক পাঠায়।

---

## ২. ৫টা শব্দ মনে রাখলেই হবে

| শব্দ | কাজ |
|---|---|
| `QueryClient` | cache store (আলমারি) |
| `QueryClientProvider` | পুরো app-কে আলমারিটা দেখতে দেয় (React context) |
| `queryKey` | data-র নাম/ID, যেমন `["doctors"]` |
| `queryFn` | data আনার async function |
| `useQuery` | component-এর ভেতর data চাওয়ার hook |

```
QueryClient বানাও → Provider দিয়ে wrap করো → useQuery({ queryKey, queryFn }) → data
```

---

## ৩. Level 1 — এক file-এ সব (শুধু client side)

নতুন একটা test page বানাও: `src/app/tanstack-test/page.tsx` → browser-এ `/tanstack-test` খোলো → **browser console** (F12) দেখো।

```tsx
"use client"; // hook ব্যবহার করব, তাই client component

import { QueryClient, QueryClientProvider, useQuery } from "@tanstack/react-query";
import { httpClient } from "@/lib/axios/httpClient";

// ধাপ ১: আলমারি (cache) বানাও
const queryClient = new QueryClient();

// ধাপ ২: data আনার function (queryFn)
const fetchDoctors = async () => {
  return httpClient.get("/doctors"); // { success, message, data: [...] }
};

// ধাপ ৩: useQuery দিয়ে data চাও
function Doctors() {
  const { data, isLoading, isError } = useQuery({
    queryKey: ["doctors"], // ড্রয়ারের নাম
    queryFn: fetchDoctors, // না থাকলে এটা দিয়ে আনবে
  });

  console.log("doctors:", data);

  if (isLoading) return <p>Loading...</p>;
  if (isError) return <p>কিছু ভুল হয়েছে</p>;
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}

// ধাপ ৪: Provider দিয়ে wrap করো
export default function TanstackTestPage() {
  return (
    <QueryClientProvider client={queryClient}>
      <Doctors />
    </QueryClientProvider>
  );
}
```

### Console-এ কী দেখবে?

```
doctors: undefined          ← প্রথম render, তখনো data আসেনি (isLoading = true)
doctors: { success: true, data: [...] }   ← fetch শেষ, আবার render
```

> - `useQuery` অবশ্যই Provider-এর **ভেতরের** component-এ থাকতে হবে — তাই `Doctors` আলাদা function।
> - আসল project-এ Provider একবারই `layout.tsx`-এ থাকে (teacher-এর `QueryProviders`), তখন প্রতিটা page-এ শুধু ধাপ ২ + ৩ লাগে।
> - Browser থেকে সরাসরি API call হচ্ছে, তাই server-এ CORS-এ client URL allow থাকতে হবে। CORS error এলে `fetchDoctors`-এর জায়গায় teacher-এর `getDoctors` (`_actions.ts`) ব্যবহার করো।

### সমস্যা কোথায়?
Page খুললে প্রথমে **"Loading..."** দেখায়, তারপর browser data আনে। Server HTML-এ doctors নেই → ধীর প্রথম load, SEO খারাপ।
👉 সমাধান: Level 2 — data **server-এ আগেই** এনে রাখা।

---

## ৪. Level 2 — Server-এ prefetch + dehydrate + hydrate (Teacher-এর way)

### আইডিয়া (উপমা)

| শব্দ | মানে |
|---|---|
| `prefetchQuery` | server-এ আগেই data এনে server-এর আলমারিতে রাখা |
| `dehydrate` | আলমারির data-কে "শুকিয়ে" JSON বানানো, যাতে HTML-এর সাথে browser-এ পাঠানো যায় |
| `HydrationBoundary` | browser-এ সেই JSON-এ "পানি দিয়ে" আবার browser-এর আলমারিতে বসানো |
| `useQuery` (same key) | আলমারিতে data পেয়ে যায় → **fetch ছাড়াই** সাথে সাথে দেখায় |

```
Server page:  new QueryClient → prefetchQuery(["doctors"]) → dehydrate → <HydrationBoundary>
Browser:      HydrationBoundary cache বসায় → useQuery(["doctors"]) cache-এ পায় → Loading নেই ✅
```

> 🔑 সবচেয়ে জরুরি নিয়ম: server-এর `queryKey` আর client-এর `queryKey` **হুবহু এক** হতে হবে (`["doctors"]`)। না মিললে client আবার fetch করবে।

### ধাপ ১: data আনার function — `consultation/_actions.ts`

```ts
"use server"; // server action — server-এ চলে, client থেকেও call করা যায়

import { httpClient } from "@/lib/axios/httpClient";

export const getDoctors = async () => {
  const doctors = await httpClient.get("/doctors");
  console.log(doctors, "server"); // এটা দেখাবে terminal-এ, browser-এ না
  return doctors;
};
```

### ধাপ ২: Server page-এ prefetch — `consultation/page.tsx`

```tsx
import DoctorsList from "@/components/modules/Consultation/DoctorsList";
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query";
import { getDoctors } from "./_actions";

const ConsultationPage = async () => {
  const queryClient = new QueryClient(); // প্রতি request-এ নতুন আলমারি

  await queryClient.prefetchQuery({
    queryKey: ["doctors"],
    queryFn: getDoctors,
  });

  return (
    // dehydrate করা data props হিসেবে পাঠানো — HydrationBoundary client-এ বসাবে
    <HydrationBoundary state={dehydrate(queryClient)}>
      <DoctorsList />
    </HydrationBoundary>
  );
};

export default ConsultationPage;
```

### ধাপ ৩: Client component-এ useQuery — `DoctorsList.tsx`

```tsx
/* eslint-disable @typescript-eslint/no-explicit-any */
"use client";
import { getDoctors } from "@/app/(commonLayout)/consultation/_actions";
import { useQuery } from "@tanstack/react-query";

const DoctorsList = () => {
  const { data } = useQuery({
    queryKey: ["doctors"], // server-এর সাথে same key
    queryFn: () => getDoctors(),
  });

  console.log(data); // browser console-এ — প্রথমবারেই data থাকবে (undefined না)

  return (
    <div>
      {data.data.map((doctor: any) => (
        <div key={doctor.id}>{doctor.name}</div>
      ))}
    </div>
  );
};

export default DoctorsList;
```

### 🧪 নিজে পরীক্ষা করো (teacher-এর commented code)

`DoctorsList`-এ teacher একটা **non-prefetched** query রেখেছেন — key আলাদা (`["doctors-non-prefetched"]`), server-এ prefetch হয়নি।

| | `["doctors"]` (prefetched) | `["doctors-non-prefetched"]` |
|---|---|---|
| প্রথম `console.log` | data সাথে সাথে ✅ | `undefined` |
| Network tab | browser-এ কোনো request নেই | browser থেকে নতুন request |
| Terminal-এ `"server"` log | page load-এ একবার | — |

👉 এটা দেখলেই prefetch-এর পার্থক্য পরিষ্কার হয়ে যাবে।

---

## ৫. Teacher-এর `QueryProvider.tsx` — এক লাইনে প্রতিটা অংশ

`layout.tsx`-এ `<QueryProviders>{children}</QueryProviders>` দিয়ে পুরো app wrap করা।

| অংশ | কেন |
|---|---|
| `"use client"` | Provider ভেতরে React context ব্যবহার করে, তাই client-এ চলতে হবে |
| `makeQueryClient()` | default setting সহ নতুন `QueryClient` বানায় |
| `staleTime: 60 * 1000` | data ১ মিনিট "fresh" ধরা হবে → server থেকে আসা data client সাথে সাথে আবার fetch করবে না |
| `isServer` হলে নতুন client | server-এ প্রতি request-এ নতুন — এক user-এর data অন্য user-এর কাছে যাবে না |
| browser-এ `browserQueryClient` একবারই | browser-এ একটাই আলমারি থাকবে, বারবার নতুন বানালে cache হারিয়ে যাবে |
| `useState` ব্যবহার না করা | Suspense হলে React প্রথম render ফেলে দিতে পারে, তখন useState-এর client-ও হারায় |
| `ReactQueryStreamedHydration` | (experimental package) `useSuspenseQuery` দিয়ে client component থেকেই server-এ data stream করার জন্য। উপরের `prefetchQuery` pattern-এর জন্য এটা বাধ্যতামূলক না |

---

## ৬. Cheat sheet

**`useQuery` কী ফেরত দেয়:**

| জিনিস | মানে |
|---|---|
| `data` | পাওয়া data (প্রথমে `undefined` হতে পারে) |
| `isLoading` | প্রথমবার আনছে, cache-এ কিছু নেই |
| `isFetching` | যেকোনো fetch চলছে (background refetch-সহ) |
| `isError` / `error` | fetch fail করেছে |
| `refetch()` | হাতে করে আবার আনো |

**দুইটা time:**

| | মানে | default |
|---|---|---|
| `staleTime` | কতক্ষণ data fresh — এর মধ্যে refetch হবে না | `0` |
| `gcTime` | কেউ ব্যবহার না করলে কতক্ষণ পর cache থেকে মুছবে | ৫ মিনিট |

**`queryKey` নিয়ম:**
- সবসময় array: `["doctors"]`
- filter/id থাকলে key-তে দাও: `["doctors", page]`, `["doctor", id]` → আলাদা value = আলাদা cache

**Console কোথায় দেখবে:**
- Server component / server action-এর `console.log` → **terminal** (`pnpm dev`)
- `"use client"` component-এর `console.log` → **browser console**

---

## 🔜 পরে যা যোগ হবে
- [ ] `useMutation` + `invalidateQueries` (POST/PATCH/DELETE-এর পর list refresh) — `client/05-tanstack-query-setup.md`-এ ছোট উদাহরণ আছে
- [ ] `queryOptions` দিয়ে key + fn এক জায়গায় রাখা — `client/06-doctors-fetching-with-tanstack-query.md`
- [ ] Pagination / filter-সহ query key

---

## 🤖 Claude Recommendation
> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice
1. **`data.data.map` → `data?.data?.map`** — prefetch fail করলে বা key না মিললে প্রথম render-এ `data` থাকবে `undefined`, তখন page crash করবে।
   ```tsx
   {data?.data?.map((doctor: any) => <div key={doctor.id}>{doctor.name}</div>)}
   ```
2. **queryKey + queryFn এক জায়গায় রাখো** — server আর client-এ দুইবার `["doctors"]` লিখলে typo হলে prefetch কাজ করবে না (কোনো error ছাড়াই)।
   ```ts
   export const doctorsQuery = { queryKey: ["doctors"], queryFn: getDoctors };
   // server: prefetchQuery(doctorsQuery)   client: useQuery(doctorsQuery)
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো
3. **Devtools** — `@tanstack/react-query-devtools` দিলে browser-এ আলমারির ভেতরে কী আছে (key, fresh/stale) চোখে দেখা যায়; শেখার সময় খুব কাজের।
