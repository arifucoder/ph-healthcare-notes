# 04 - Axios দিয়ে httpClient

```
.env (API URL) → api.types.ts (response type) → httpClient.ts (axios instance + get/post/...) → component/service থেকে call
```

## Axios কী, আর কেন fetch না

Axios হলো একটা **HTTP client** — browser বা server থেকে API-তে request পাঠানোর library। আমাদের project-এ data fetching-এর জন্য `fetch` না, **axios** ব্যবহার করব।

| বিষয় | `fetch` | `axios` |
|---|---|---|
| JSON | নিজে `await res.json()` করতে হয় | নিজেই JSON বানিয়ে `response.data`-তে দেয় |
| Error | 404/500-এও error throw করে না, নিজে `res.ok` check করতে হয় | 4xx/5xx হলে নিজেই error throw করে → সরাসরি `catch`-এ যায় |
| Base URL | প্রতিবার পুরো URL লিখতে হয় | `axios.create({ baseURL })` একবার দিলেই হয় |
| Timeout | নিজে বানাতে হয় | `timeout: 30000` |
| Common header | প্রতিটা request-এ দিতে হয় | instance-এ একবার |

## ধাপ ১: Install

👉 https://axios.rest/pages/getting-started/first-steps.html

```shell
pnpm install axios
```

## ধাপ ২: কীভাবে ব্যবহার করে — example দেখা

👉 https://axios.rest/pages/getting-started/examples/typescript.html

Example-এ ২ ভাবেই দেখানো আছে: `.then()` দিয়ে আর `try/catch` দিয়ে। আমরা TypeScript ব্যবহার করছি, তাই শুধু **`async/await` + `try/catch`** ব্যবহার করব (পড়তে সহজ, type-ও ভালো কাজ করে)।

```ts
// .then() — আমরা এটা ব্যবহার করব না
axios.get("/users").then((res) => console.log(res.data)).catch(console.error);

// try/catch ✅
try {
	const res = await axios.get<User[]>("/users");
	console.log(res.data);
} catch (error) {
	console.error(error);
}
```

## ধাপ ৩: Response-এর type — `src/types/api.types.ts`

আমাদের server সবসময় একই shape-এ response পাঠায় (`sendResponse` → `{ success, message, meta, data }`)। সেই shape-টাই এখানে TypeScript type হিসেবে লিখে রাখছি:

```ts
export interface PaginationMeta {
	page: number;
	limit: number;
	total: number;
	totalPages: number;
}

export interface ApiResponse<TData = unknown> {
	success: true;
	message: string;
	data: TData;
	meta?: PaginationMeta;
}

export interface ApiErrorResponse {
	success: false;
	message: string;
}
```

| Type | কাজ |
|---|---|
| `PaginationMeta` | list API-তে page info (server-এর `IQueryMeta`-এর মতো) |
| `ApiResponse<TData>` | সফল response। `TData` = generic — `data`-র ভেতরে কী আসবে সেটা call করার সময় বলে দিব (যেমন `Specialty[]`) |
| `ApiErrorResponse` | error response (`success: false`) |

## ধাপ ৪: httpClient — `src/lib/axios/httpClient.ts`

```ts
import type { ApiResponse } from "@/types/api.types";
import axios, { type AxiosInstance, type Method } from "axios";
import { connection } from "next/server";

const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL;

if (!API_BASE_URL) {
	throw new Error("NEXT_PUBLIC_API_BASE_URL is not defined in environment variables");
}

export interface ApiRequestOptions {
	params?: Record<string, unknown>;
	headers?: Record<string, string>;
}

// পরে cookie যোগ করার সময় শুধু এই function বদলাতে হবে
const axiosInstance = async (): Promise<AxiosInstance> => {
	// axios ভেতরে Date.now() ব্যবহার করে, যেটা Next.js চেনে না।
	// connection() Next-কে বলে: এই request আসল request-time-এ চালাও, prerender কোরো না।
	await connection();

	return axios.create({
		baseURL: API_BASE_URL,
		timeout: 30000,
		headers: {
			"Content-Type": "application/json",
		},
	});
};

const request = async <TData>(
	method: Method,
	endpoint: string,
	data?: unknown,
	options?: ApiRequestOptions,
): Promise<ApiResponse<TData>> => {
	try {
		const instance = await axiosInstance();
		const response = await instance.request<ApiResponse<TData>>({
			method,
			url: endpoint,
			data,
			params: options?.params,
			headers: options?.headers,
		});
		return response.data;
	} catch (error) {
		console.error(`${method} request to ${endpoint} failed:`, error);
		throw error;
	}
};

export const httpClient = {
	get: <TData>(endpoint: string, options?: ApiRequestOptions) => request<TData>("GET", endpoint, undefined, options),

	post: <TData>(endpoint: string, data: unknown, options?: ApiRequestOptions) =>
		request<TData>("POST", endpoint, data, options),

	put: <TData>(endpoint: string, data: unknown, options?: ApiRequestOptions) =>
		request<TData>("PUT", endpoint, data, options),

	patch: <TData>(endpoint: string, data: unknown, options?: ApiRequestOptions) =>
		request<TData>("PATCH", endpoint, data, options),

	delete: <TData>(endpoint: string, options?: ApiRequestOptions) =>
		request<TData>("DELETE", endpoint, undefined, options),
};

```

### প্রতিটা অংশ কী করছে

| অংশ | কাজ |
|---|---|
| `API_BASE_URL` + `throw` | `.env` থেকে server-এর URL নেয়। না থাকলে শুরুতেই পরিষ্কার error দেয়, যাতে পরে "request কোথায় যাচ্ছে" খুঁজতে না হয় |
| `ApiRequestOptions` | চাইলে extra `params` (URL query: `?page=1&limit=10`) আর `headers` পাঠানো যায় |
| `axiosInstance()` | baseURL, timeout (৩০ সেকেন্ড), JSON header সহ একটা axios তৈরি করে। এটা `async` রাখা হয়েছে কারণ পরে এখানে `await cookies()` দিয়ে token যোগ হবে |
| `request()` | **সব method-এর আসল কাজ এক জায়গায়** — instance বানায়, request পাঠায়, শুধু `response.data` (server-এর JSON) ফেরত দেয়। Error হলে log করে আবার `throw` করে, যাতে যে call করেছে সে-ও error জানতে পারে |
| `httpClient` | `get/post/put/patch/delete` — সবাই ভেতরে `request()` call করে, শুধু method বদলায়। `get`/`delete`-এ body থাকে না তাই `undefined` |

> 💡 Teacher-এর version-এ ৫টা আলাদা function-এ (`httpGet`, `httpPost` ...) একই try/catch ৫ বার লেখা ছিল। এখানে একটা `request()` function দিয়ে বাকিগুলো বানানো — কাজ একই, code কম, বদলাতে হলে এক জায়গায়।

## ধাপ ৫: `.env`

```dotenv
NEXT_PUBLIC_API_BASE_URL=http://localhost:5000/api/v1
```

- `5000` = আমাদের server-এর `PORT`, `/api/v1` = server-এর `IndexRoutes`-এর prefix।
- `NEXT_PUBLIC_` দিয়ে শুরু হলে variable টা **browser-এও** পাওয়া যায় (client component থেকে call করতে লাগবে)। এটা ছাড়া শুধু server-এ পাওয়া যায়।
- ⚠️ `.env` বদলালে `pnpm dev` **restart** করতে হয়।

## ধাপ ৬: Normal usage

```tsx
import { httpClient } from "@/lib/axios/httpClient";

type Specialty = { id: string; title: string };

async function SpecialtiesList() {
	const res = await httpClient.get<Specialty[]>("/specialties");
	return (
		<ul>
			{res.data.map((s) => (
				<li key={s.id}>{s.title}</li>
			))}
		</ul>
	);
}
```

- `httpClient.get<Specialty[]>` → generic-এ বলে দিলাম `data` হবে `Specialty[]`, তাই `s.id`, `s.title`-এ autocomplete আর type check পাওয়া যায়।
- Request যায়: `http://localhost:5000/api/v1/specialties`
- `res` = server-এর পুরো JSON (`{ success, message, data }`), তাই list পেতে `res.data`।

### কেন পরে TanStack Query

এটা server component-এ সরাসরি call। কিন্তু client side-এ আমরা **axios + TanStack Query** একসাথে ব্যবহার করব — axios request পাঠাবে, আর TanStack Query সেটা manage করবে:

| TanStack Query যা দেয় | মানে |
|---|---|
| Caching | একই data আবার লাগলে API-তে না গিয়ে memory থেকে দেয় → দ্রুত |
| Revalidate / refetch | data পুরোনো হলে, বা tab-এ ফিরে এলে নিজে আবার আনে |
| `isLoading`, `isError` | loading/error state নিজে `useState` দিয়ে বানাতে হয় না |
| Mutation + invalidate | নতুন specialty add করলে list নিজে নিজে update হয় |
| Retry | fail করলে নিজে কয়েকবার চেষ্টা করে |
| Deduping | একই সময়ে ৩টা component একই data চাইলে request যায় ১ বার |

```tsx
// পরে এরকম হবে (client component)
const { data, isLoading } = useQuery({
	queryKey: ["specialties"],
	queryFn: () => httpClient.get<Specialty[]>("/specialties"),
});
```

## 🔜 পরে যা যোগ হবে

Teacher-এর শেষ version-এ `axiosInstance()`-এ এগুলো যোগ হয় (আমাদের code-এ সেই জায়গা ফাঁকা রাখা আছে):

- [ ] `cookies()` থেকে `accessToken`/`refreshToken` পড়ে request-এর `Cookie` header-এ পাঠানো (server component থেকে call করলে browser-এর cookie নিজে যায় না)
- [ ] Access token শেষ হওয়ার আগে refresh token দিয়ে নতুন token আনা (`tryRefreshToken`)
- [ ] TanStack Query-এর `QueryClientProvider` setup আর `useQuery` / `useMutation`
- [ ] `services/` folder-এ API function (যেমন `getSpecialties()`)

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

1. **`SpecialtiesList` অবশ্যই `<Suspense>`-এর ভেতরে রাখো** — আমাদের `next.config.ts`-এ `cacheComponents: true`, তাই `await` থাকা component সরাসরি page-এ বসালে `uncached data` error দিবে (নোট 03, ধাপ ৪):

   ```tsx
   export default function SpecialtiesPage() {
   	return (
   		<Suspense fallback={<LoadingSpinner text="Loading specialties..." />}>
   			<SpecialtiesList />
   		</Suspense>
   	);
   }
   ```

### 🟡 ভালো practice

2. **Server-এর error message বের করার helper** — axios error-এর ভেতরে server-এর `message` লুকানো থাকে (`error.response.data.message`)। Toast/form-এ দেখাতে এটা লাগবে:

   ```ts
   // src/lib/axios/getErrorMessage.ts
   import axios from "axios";
   import type { ApiErrorResponse } from "@/types/api.types";

   export const getErrorMessage = (error: unknown): string => {
   	if (axios.isAxiosError<ApiErrorResponse>(error)) {
   		return error.response?.data?.message ?? error.message;
   	}
   	return error instanceof Error ? error.message : "Something went wrong";
   };
   ```

3. **`.env.example` বানাও** — `.gitignore`-এ `.env*` আছে, তাই `.env` GitHub-এ যাবে না। অন্য কেউ (বা নতুন computer-এ তুমি) clone করলে কোন variable লাগবে বুঝতে:

   ```dotenv
   # .env.example
   NEXT_PUBLIC_API_BASE_URL=http://localhost:5000/api/v1
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **Browser থেকে call করলে `withCredentials: true` লাগবে** — server-এ `cors({ credentials: true })` আছে, কিন্তু browser cookie (login token) পাঠাবে শুধু তখনই যখন axios-এও এটা থাকে:

   ```ts
   axios.create({ baseURL: API_BASE_URL, withCredentials: true, ... });
   ```

5. **`TData`-র type গুলো `src/types/`-এ রাখো** — `type Specialty` component-এর ভেতরে না রেখে `src/types/specialty.types.ts`-এ রাখলে সব জায়গায় reuse হবে।
