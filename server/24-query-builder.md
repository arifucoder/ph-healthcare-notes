# 24 - Query Builder (search, filter, sort, pagination, fields, include)

> Module: 40-04

```
GET /doctors?searchTerm=..&gender=..&sortBy=..&page=.. → req.query → QueryBuilder
→ .where() .search() .filter() .sort() .paginate() .fields() .include() .dynamicInclude()
→ .execute() → Promise.all([findMany, count]) → { data, meta } → sendResponse (meta + data)
```

---

## ধাপ ১: `src/app/interfaces/query.interface.ts`

```ts
// raw req.query — Express-এর query parser string, array বা nested object দিতে পারে
export type IQueryParams = Record<string, unknown>;

export interface IQueryConfig {
	searchableFields?: readonly string[];
	filterableFields?: readonly string[];
	sortableFields?: readonly string[];
	selectableFields?: readonly string[];
	enumFields?: readonly string[]; // value auto UPPERCASE হয়
	caseInsensitiveFields?: readonly string[]; // exact match, কিন্তু ছোট/বড় হাতের অক্ষর ignore
	tieBreakerFields?: readonly string[]; // default ["id"] — composite key model-এ বদলাও
	defaultSortBy?: string;
	defaultLimit?: number;
	maxLimit?: number;
}

export interface IQueryMeta {
	page: number;
	limit: number;
	total: number;
	totalPages: number;
}

export interface IQueryResult<T> {
	data: T[];
	meta: IQueryMeta;
}

// যেকোনো Prisma model delegate (prisma.doctor, prisma.patient ...) এই shape মেনে চলে
export interface PrismaModelDelegate {
	// oxlint-disable-next-line typescript/no-explicit-any
	findMany(args?: any): Promise<unknown[]>;
	// oxlint-disable-next-line typescript/no-explicit-any
	count(args?: any): Promise<number>;
}
```

- `readonly string[]` → constant file-এর `as const` array সরাসরি পাঠানো যায়।
- `tieBreakerFields` → পরে যোগ হয়েছে ([নোট ২৬](./26-doctor-schedule.md)), নিচে `sort()`-এ ব্যাখ্যা।
- `PrismaModelDelegate` → যেকোনো model (`prisma.doctor`, `prisma.patient`) একই class-এ চলে। এখানে `any` লাগে, তাই **oxlint**-এর comment (এটা কাজ করে, eslint-এর মতো অকেজো না)।

---

## ধাপ ২: `src/app/utils/QueryBuilder.ts`

```ts
import type { IQueryConfig, IQueryParams, IQueryResult, PrismaModelDelegate } from "@/app/interfaces/query.interface";

type AnyObject = Record<string, unknown>;

const ALLOWED_OPERATORS = new Set([
	"equals",
	"not",
	"lt",
	"lte",
	"gt",
	"gte",
	"in",
	"notIn",
	"contains",
	"startsWith",
	"endsWith",
]);
const STRING_OPERATORS = new Set(["contains", "startsWith", "endsWith"]);

const isPlainObject = (value: unknown): value is AnyObject =>
	typeof value === "object" && value !== null && !Array.isArray(value);

const toPositiveInt = (value: unknown, fallback: number): number => {
	const num = Math.floor(Number(value));
	return Number.isFinite(num) && num > 0 ? num : fallback;
};

/**
 * "name"                         -> { name: leaf }
 * "user.email"                   -> { user: { email: leaf } }                         (to-one)
 * "specialties.specialty.title"  -> { specialties: { some: { specialty: { title: leaf } } } } (to-many)
 */
const toNested = (path: string, leaf: unknown, useSome = true): AnyObject => {
	const parts = path.split(".");

	return parts.reduceRight<unknown>((node, key, index) => {
		const wrapSome = useSome && index === 0 && parts.length === 3;
		return { [key]: wrapSome ? { some: node } : node };
	}, leaf) as AnyObject;
};

export class QueryBuilder<T, TWhere = AnyObject, TInclude = AnyObject> {
	private conditions: AnyObject[] = [];
	private orderBy: AnyObject[] = [];
	private includeArgs: AnyObject = {};
	private selectArgs: Record<string, true> | undefined;
	private page = 1;
	private limit: number;

	constructor(
		private readonly model: PrismaModelDelegate,
		private readonly queryParams: IQueryParams,
		private readonly config: IQueryConfig = {},
	) {
		this.limit = config.defaultLimit ?? 10;
	}

	// ?searchTerm=john
	search(): this {
		const term = this.getString("searchTerm")?.trim();
		const { searchableFields = [] } = this.config;

		if (!term || searchableFields.length === 0) return this;

		this.conditions.push({
			OR: searchableFields.map((field) => toNested(field, { contains: term, mode: "insensitive" })),
		});

		return this;
	}

	// ?gender=male&specialties.specialty.title=cardiology
	filter(): this {
		const { filterableFields = [], enumFields = [], caseInsensitiveFields = [] } = this.config;

		// শুধু whitelist-এ থাকা field-ই দেখা হয় — বাকি সব query key নিজে থেকেই ignore
		for (const field of filterableFields) {
			const raw = this.queryParams[field];
			if (raw === undefined || raw === "") continue;

			let value = this.parseValue(raw);
			if (value === undefined) continue;

			if (enumFields.includes(field)) {
				value = this.toUpperDeep(value); // male -> MALE
			} else if (caseInsensitiveFields.includes(field) && typeof value === "string") {
				value = { equals: value, mode: "insensitive" }; // cardiology == Cardiology
			}

			this.conditions.push(toNested(field, value));
		}

		return this;
	}

	// ?sortBy=appointmentFee&sortOrder=asc
	sort(): this {
		const { sortableFields = [], defaultSortBy = "createdAt", tieBreakerFields = ["id"] } = this.config;
		const requested = this.getString("sortBy");

		const sortBy = requested && sortableFields.includes(requested) ? requested : defaultSortBy;
		const sortOrder = this.getString("sortOrder")?.toLowerCase() === "asc" ? "asc" : "desc";

		// tie-breaker: একই value-ওয়ালা row page বদলালে duplicate বা miss হবে না
		this.orderBy = [toNested(sortBy, sortOrder, false)];
		for (const field of tieBreakerFields) {
			if (field !== sortBy) this.orderBy.push({ [field]: sortOrder });
		}

		return this;
	}

	// ?page=2&limit=20
	paginate(): this {
		const { defaultLimit = 10, maxLimit = 100 } = this.config;

		this.page = toPositiveInt(this.getString("page"), 1);
		this.limit = Math.min(toPositiveInt(this.getString("limit"), defaultLimit), maxLimit);

		return this;
	}

	// ?fields=name,email,appointmentFee
	fields(): this {
		const raw = this.getString("fields");
		const { selectableFields = [] } = this.config;

		if (!raw) return this;

		const picked = raw
			.split(",")
			.map((field) => field.trim())
			.filter((field) => selectableFields.includes(field));

		if (picked.length > 0) {
			this.selectArgs = Object.fromEntries(picked.map((field) => [field, true as const]));
		}

		return this;
	}

	// code থেকে fixed include
	include(relations: TInclude): this {
		this.includeArgs = { ...this.includeArgs, ...(relations as AnyObject) };
		return this;
	}

	// ?include=user,specialties — শুধু includeConfig-এ থাকা relation allowed
	dynamicInclude(includeConfig: AnyObject, defaultInclude: readonly string[] = []): this {
		const requested =
			this.getString("include")
				?.split(",")
				.map((r) => r.trim()) ?? [];

		for (const relation of [...defaultInclude, ...requested]) {
			if (includeConfig[relation]) this.includeArgs[relation] = includeConfig[relation];
		}

		return this;
	}

	// code থেকে fixed condition, যেমন { isDeleted: false }
	where(condition: TWhere): this {
		this.conditions.push(condition as AnyObject);
		return this;
	}

	async execute(): Promise<IQueryResult<T>> {
		const where = this.buildWhere();

		// select আর include একসাথে Prisma নেয় না — fields দিলে select জিতবে
		const shape = this.selectArgs
			? { select: this.selectArgs }
			: Object.keys(this.includeArgs).length > 0
				? { include: this.includeArgs }
				: {};

		const [data, total] = await Promise.all([
			this.model.findMany({
				where,
				orderBy: this.orderBy.length > 0 ? this.orderBy : undefined,
				skip: (this.page - 1) * this.limit,
				take: this.limit,
				...shape,
			}),
			this.model.count({ where }),
		]);

		return {
			data: data as T[],
			meta: {
				page: this.page,
				limit: this.limit,
				total,
				totalPages: Math.ceil(total / this.limit),
			},
		};
	}

	// ---------- helpers ----------

	private buildWhere(): AnyObject {
		return this.conditions.length > 0 ? { AND: this.conditions } : {};
	}

	private getString(key: string): string | undefined {
		const value = this.queryParams[key];
		return typeof value === "string" ? value : undefined;
	}

	private parseValue(value: unknown): unknown {
		if (Array.isArray(value)) return { in: value.map((v) => this.parseScalar(v)) };
		if (isPlainObject(value)) return this.parseOperators(value);
		return this.parseScalar(value);
	}

	// "male" -> "MALE", { in: ["male"] } -> { in: ["MALE"] }
	private toUpperDeep(value: unknown): unknown {
		if (typeof value === "string") return value.toUpperCase();
		if (Array.isArray(value)) return value.map((v) => this.toUpperDeep(v));
		if (isPlainObject(value)) {
			return Object.fromEntries(Object.entries(value).map(([key, v]) => [key, this.toUpperDeep(v)]));
		}
		return value;
	}

	private parseScalar(value: unknown): unknown {
		if (value === "true") return true;
		if (value === "false") return false;
		if (typeof value === "string" && value.trim() !== "" && !Number.isNaN(Number(value))) {
			return Number(value);
		}
		return value;
	}

	// { gte: "500", lt: "1000", hack: "x" } -> { gte: 500, lt: 1000 }
	private parseOperators(obj: AnyObject): AnyObject | undefined {
		const result: AnyObject = {};

		for (const [operator, raw] of Object.entries(obj)) {
			if (!ALLOWED_OPERATORS.has(operator)) continue;

			if (operator === "in" || operator === "notIn") {
				result[operator] = (Array.isArray(raw) ? raw : [raw]).map((v) => this.parseScalar(v));
			} else if (STRING_OPERATORS.has(operator)) {
				result[operator] = String(raw);
				result.mode = "insensitive";
			} else {
				result[operator] = this.parseScalar(raw);
			}
		}

		return Object.keys(result).length > 0 ? result : undefined;
	}
}
```

| Method | URL | Prisma-তে যা হয় |
|---|---|---|
| `where()` | — (code থেকে) | `{ isDeleted: false }` |
| `search()` | `?searchTerm=john` | `OR: [{ name: { contains, mode: "insensitive" } }, ...]` |
| `filter()` | `?gender=male`, `?appointmentFee[gte]=500` | whitelist field গুলো → `AND`-এ |
| `sort()` | `?sortBy=..&sortOrder=asc` | `orderBy: [{ field }, ...tieBreakerFields]` (default `{ id }`) |
| `paginate()` | `?page=2&limit=5` | `skip`, `take` |
| `fields()` | `?fields=name,email` | `select` |
| `include()` | — (code থেকে) | সবসময় আসা relation |
| `dynamicInclude()` | `?include=reviews` | client চাইলে আসা relation |
| `execute()` | — | `findMany` + `count` একসাথে → `{ data, meta }` |

Helper গুলো:

| Helper | কাজ |
|---|---|
| `toNested` | `"a.b.c"` path → nested Prisma object (৩ ভাগ হলে `some`) |
| `getString` | শুধু string নেয় — array/object এলে `undefined` (`?page[x]=1`-এ crash হয় না) |
| `parseValue` | array → `{ in }`, object → operator, বাকি → scalar |
| `parseScalar` | `"true"` → `true`, `"500"` → `500` |
| `parseOperators` | whitelist operator রাখে, `hack` বাদ দেয় |
| `toUpperDeep` | enum value UPPERCASE (array/object-এর ভেতরেও) |

### `sort()`-এর tie-breaker (update)

> ✏️ `sort()` পরে update হয়েছে ([নোট ২৬](./26-doctor-schedule.md)) — উপরের code এখনকার version।

আগে `sort()` সবসময় `{ id: sortOrder }` যোগ করত:

```ts
// আগের version
if (sortBy !== "id") this.orderBy.push({ id: sortOrder });
```

কিন্তু `DoctorSchedules`-এর মতো composite key model-এ `id` column নেই → Prisma error। এখন tie-breaker config থেকে আসে:

| Model | `tieBreakerFields` | `orderBy` |
|---|---|---|
| Doctor, Schedule, Patient... | দেওয়া নেই → default `["id"]` | `[{ sortBy }, { id }]` |
| DoctorSchedules | `["doctorId", "scheduleId"]` | `[{ schedule: { startDateTime } }, { doctorId }, { scheduleId }]` |

- **Tie-breaker কেন?** অনেক row-এর `createdAt` (বা `appointmentFee`) একই হলে DB প্রতিবার একই ক্রম দেবে এমন নিশ্চয়তা নেই — তখন page 1 আর page 2-তে একই row দুবার আসতে বা কোনোটা বাদ পড়তে পারে। Unique field দিয়ে শেষে আরেকবার sort করলে ক্রম সবসময় একই।
- `field !== sortBy` → `?sortBy=id` দিলে `id` দুবার যোগ হয় না।
- আগের module-গুলোতে কিছু বদলাতে হয়নি — default `["id"]`।

---

## ধাপ ৩: `src/app.ts`-এ query parser চালু করো

`const app = express();`-এর ঠিক নিচে বসাও:

```ts
// Express 5-এর default "simple" parser nested query বোঝে না: appointmentFee[gte]=500-কে object বানাতে "extended" লাগে
app.set("query parser", "extended");
```

| URL | `simple` (default) | `extended` |
|---|---|---|
| `?appointmentFee[gte]=500` | `{ "appointmentFee[gte]": "500" }` | `{ appointmentFee: { gte: "500" } }` |

---

## ধাপ ৪: `src/app/module/doctor/doctor.constant.ts`

```ts
import type { Prisma } from "@/generated/prisma/client";

export const doctorSearchableFields = [
	"name",
	"email",
	"qualification",
	"designation",
	"currentWorkingPlace",
	"specialties.specialty.title",
] as const;

export const doctorFilterableFields = [
	"gender",
	"experience",
	"appointmentFee",
	"averageRating",
	"designation",
	"specialties.specialty.title",
] as const;

export const doctorSortableFields = ["createdAt", "name", "experience", "appointmentFee", "averageRating"] as const;

export const doctorSelectableFields = [
	"id",
	"name",
	"email",
	"profilePhoto",
	"experience",
	"gender",
	"appointmentFee",
	"qualification",
	"designation",
	"currentWorkingPlace",
	"averageRating",
	"createdAt",
] as const;

export const doctorIncludeConfig = {
	appointments: { include: { patient: true, schedule: true } },
	doctorSchedules: { include: { schedule: true } },
	prescriptions: true,
	reviews: true,
} satisfies Prisma.DoctorInclude;

export const doctorEnumFields = ["gender"] as const;

export const doctorCaseInsensitiveFields = ["designation", "specialties.specialty.title"] as const;
```

- `satisfies Prisma.DoctorInclude` → ভুল relation নাম লিখলে TypeScript error দেবে, কিন্তু object-এর আসল type হারায় না।

---

## ধাপ ৫: `doctor.service.ts`

> ✏️ `doctorInclude` আর import গুলো নোটে ছিল না — project থেকে নেওয়া:

```ts
import type { IQueryParams, IQueryResult } from "@/app/interfaces/query.interface";
import { prisma } from "@/app/lib/prisma";
import { QueryBuilder } from "@/app/utils/QueryBuilder";
import type { Prisma } from "@/generated/prisma/client";
import {
	doctorCaseInsensitiveFields,
	doctorEnumFields,
	doctorFilterableFields,
	doctorIncludeConfig,
	doctorSearchableFields,
	doctorSelectableFields,
	doctorSortableFields,
} from "./doctor.constant";

// reusable include, যাতে সব query একই shape ফেরত দেয়
const doctorInclude = {
	user: true,
	specialties: {
		include: {
			specialty: true,
		},
	},
} as const;
```

```ts
type DoctorWithRelations = Prisma.DoctorGetPayload<{ include: typeof doctorInclude }>;

const getAllDoctors = async (query: IQueryParams): Promise<IQueryResult<DoctorWithRelations>> => {
	const result = await new QueryBuilder<DoctorWithRelations, Prisma.DoctorWhereInput, Prisma.DoctorInclude>(
		prisma.doctor,
		query,
		{
			searchableFields: doctorSearchableFields,
			filterableFields: doctorFilterableFields,
			sortableFields: doctorSortableFields,
			selectableFields: doctorSelectableFields,
			enumFields: doctorEnumFields,
			caseInsensitiveFields: doctorCaseInsensitiveFields,
		},
	)
		.where({ isDeleted: false })
		.search()
		.filter()
		.sort()
		.paginate()
		.fields()
		.include(doctorInclude)
		.dynamicInclude(doctorIncludeConfig)
		.execute();

	return result;
};
```

| Generic | মানে |
|---|---|
| `DoctorWithRelations` (`T`) | `data`-র প্রতিটা item-এর type |
| `Prisma.DoctorWhereInput` | `.where()`-এ ভুল field দিলে TypeScript error |
| `Prisma.DoctorInclude` | `.include()`-এ ভুল relation দিলে TypeScript error |

---

## ধাপ ৬: `src/app/shared/sendResponse.ts` (পুরো file)

```ts
import type { IQueryMeta } from "@/app/interfaces/query.interface";
import type { Response } from "express";

interface IResponseData<T> {
	httpStatusCode: number;
	success: boolean;
	message: string;
	meta?: IQueryMeta;
	data?: T;
}

export const sendResponse = <T>(res: Response, responseData: IResponseData<T>) => {
	const { httpStatusCode, success, message, meta, data } = responseData;

	res.status(httpStatusCode).json({
		success,
		message,
		meta,
		data,
	});
};
```

- `meta` optional — অন্য API-তে না দিলে JSON-এ `meta` থাকবেই না (`undefined` বাদ পড়ে)।

---

## ধাপ ৭: `doctor.controller.ts`-এর `getAllDoctors`

```ts
const getAllDoctors = catchAsync(async (req: Request, res: Response) => {
	const result = await DoctorService.getAllDoctors(req.query);

	sendResponse(res, {
		httpStatusCode: httpStatus.OK,
		success: true,
		message: "Doctors fetched successfully",
		meta: result.meta,
		data: result.data,
	});
});
```

---

## ধাপ ৮: চেক করো

```shell
pnpm check   # = pnpm typecheck && pnpm lint
pnpm dev
```

---

## ধাপ ৯: `globalErrorHandler.ts` — ভুল type-এর value-তে `400`

> ✏️ এই ধাপ নোটে ছিল না — Claude Recommendation থেকে যোগ হয়েছে।

Whitelist-এ থাকা field-এ ভুল ধরনের value দিলে QueryBuilder সেটা আটকায় না, Prisma-তে পাঠিয়ে দেয়। Prisma তখন `PrismaClientValidationError` throw করে। আগে এটা সাধারণ `Error` branch-এ গিয়ে `500` হতো — এখন `400`।

| URL | কেন error |
|---|---|
| `?experience=abc` | `"abc"` string, কিন্তু `experience` Int |
| `?gender=abc` | `"ABC"` `Gender` enum-এ নেই |
| `?appointmentFee[gte]=abc` | Int-এ string |

```ts
import { Prisma } from "@/generated/prisma/client"; // ✅ value import (instanceof-এ লাগে), import type না

	// `else if (err instanceof Error)`-এর ঠিক আগে বসাও
	} else if (err instanceof Prisma.PrismaClientValidationError) {
		// ভুল type-এর value (query বা body) → client-এর ভুল, তাই 400
		statusCode = httpStatus.BAD_REQUEST;
		message = "Invalid query or input value";
		errorSources = [{ path: "", message }];
		stack = err.stack;
	}
```

- `PrismaClientValidationError`-ও একটা `Error`, তাই `instanceof Error`-এর **আগে** বসাতে হবে।
- Prisma-র আসল message অনেক লম্বা আর ভেতরের query দেখায়, তাই user-কে ছোট message দেওয়া হলো। Dev-এ পুরো `error` আর `stack` response-এ আসবেই (আগের মতো)।
- `verbatimModuleSyntax: true` থাকায় `import type { Prisma }` দিলে `instanceof`-এ error হবে — তাই সাধারণ `import`।
- এটা শুধু QueryBuilder না, body-তে ভুল type এলেও (যেখানে Zod নেই) `400` দেবে।

---

## Include কীভাবে কাজ করে

```
GET /api/v1/doctors?include=appointments,reviews
```

> ✏️ আগের explanation-এ `dynamicInclude(doctorIncludeConfig, ["user", "specialties"])` লেখা ছিল। এখনকার code-এ `user` আর `specialties` আসে `.include(doctorInclude)` থেকে, আর `dynamicInclude`-এ default দেওয়া নেই। নিচে এখনকার code অনুযায়ী লেখা।

1. `.include(doctorInclude)` → `user` আর `specialties.specialty` **সবসময়** আসে।
2. `.dynamicInclude(doctorIncludeConfig)` → `?include=`-এ থেকে `appointments` আর `reviews` পড়ে। দুটোই `doctorIncludeConfig`-এ আছে, তাই যোগ হয়।
3. Config-এ নেই এমন কিছু (যেমন `?include=hackerTable`) চুপচাপ বাদ পড়ে।

`dynamicInclude`-এর দ্বিতীয় parameter (`defaultInclude`) দিয়েও default relation দেওয়া যায় — তখন সেগুলো `includeConfig` থেকেই নেওয়া হয়:

```ts
.dynamicInclude(doctorIncludeConfig, ["reviews"]) // reviews সবসময়, বাকিগুলো client চাইলে
```

---

## Test (Postman)

Base: `GET http://localhost:5000/api/v1/doctors`

**মূল ৫টা URL**

```
http://localhost:5000/api/v1/doctors?searchTerm=john
http://localhost:5000/api/v1/doctors?gender=MALE&appointmentFee[gte]=500&appointmentFee[lte]=1500
http://localhost:5000/api/v1/doctors?specialties.specialty.title=Cardiology
http://localhost:5000/api/v1/doctors?sortBy=appointmentFee&sortOrder=asc&page=2&limit=5
http://localhost:5000/api/v1/doctors?fields=name,email,appointmentFee
```

**একসাথে সব**

```
http://localhost:5000/api/v1/doctors?searchTerm=a&gender=MALE&appointmentFee[gte]=500&sortBy=experience&sortOrder=desc&page=1&limit=3
```

**Security test** — এগুলো ignore হওয়া উচিত, কোনোটাতেই `500` আসা যাবে না:

```
http://localhost:5000/api/v1/doctors?sortBy=password
http://localhost:5000/api/v1/doctors?page=-3&limit=99999
http://localhost:5000/api/v1/doctors?appointmentFee[hack]=x
http://localhost:5000/api/v1/doctors?experience=abc
http://localhost:5000/api/v1/doctors?gender=abc
http://localhost:5000/api/v1/doctors?appointmentFee[gte]=abc
```

| URL | যা দেখার কথা |
|---|---|
| `?sortBy=password` | `createdAt desc` দিয়ে sort |
| `?page=-3&limit=99999` | `meta`-তে `page: 1`, `limit: 100` |
| `?appointmentFee[hack]=x` | filter ছাড়াই সব doctor |
| `?experience=abc` | `400` "Invalid query or input value" (ধাপ ৯) |
| `?gender=abc` | `400` (ধাপ ৯) |
| `?appointmentFee[gte]=abc` | `400` (ধাপ ৯) |

**`in` filter (array)**

```
http://localhost:5000/api/v1/doctors?gender=MALE&gender=FEMALE
```

**dynamicInclude**

```
http://localhost:5000/api/v1/doctors?include=reviews,doctorSchedules
```

**Case ignore (enum + caseInsensitive)**

```
http://localhost:5000/api/v1/doctors?gender=male
http://localhost:5000/api/v1/doctors?gender=male&gender=Female
http://localhost:5000/api/v1/doctors?specialties.specialty.title=cardiology
http://localhost:5000/api/v1/doctors?sortBy=appointmentFee&sortOrder=ASC
```

> curl দিয়ে test করলে `[` `]`-এর জন্য `curl -g "URL"` দিতে হবে।

---

# QueryBuilder — Short Notes

> ✏️ এই অংশ code update-এর আগে লেখা ছিল — এখনকার code (enumFields, caseInsensitiveFields, `doctorInclude`) অনুযায়ী ঠিক করা হয়েছে।

## এটা কী?

একটা reusable class। `req.query` থেকে **search, filter, sort, pagination, fields, include** পড়ে Prisma-র `findMany` + `count` query বানায়। প্রতিটা module-এ (doctor, patient, admin...) একই class দিয়ে কাজ হয়, তাই code বারবার লিখতে হয় না।

## কেন লাগে?

- প্রতিটা `getAll` API-তে আলাদা করে search/filter/pagination লিখতে হয় না → DRY
- **Whitelist** থাকায় client যা খুশি field দিয়ে filter/sort করতে পারে না → security
- Response-এ `meta` (page, limit, total, totalPages) পাওয়া যায় → frontend-এ pagination সহজ

## কীভাবে কাজ করে (flow)

```
URL → req.query → QueryBuilder
   .where()          → fixed condition (isDeleted: false)
   .search()         → OR [ field contains term ]
   .filter()         → whitelist-এ থাকা field গুলো (enum → UPPERCASE, caseInsensitive → mode)
   .sort()           → orderBy (+ tie-breaker, default id)
   .paginate()       → skip, take
   .fields()         → select
   .include()        → fixed relation
   .dynamicInclude() → client-এর চাওয়া relation
   .execute()        → Promise.all([findMany, count]) → { data, meta }
```

- সব condition একটা `AND: []` array-তে জমা হয়।
- আসল query তৈরি হয় শুধু `execute()`-এ, তাই method-এর order কোনো ব্যাপার না (শুধু `execute()` শেষে)।

## কোন URL দিলে কী হয়

Base: `http://localhost:5000/api/v1/doctors`

| URL query | কী হয় |
|---|---|
| `?searchTerm=john` | searchableFields-এর যেকোনোটায় "john" আছে (case-insensitive) |
| `?gender=MALE` বা `?gender=male` | exact match — `enumFields`-এ আছে বলে auto UPPERCASE |
| `?gender=MALE&gender=FEMALE` | `in: [MALE, FEMALE]` |
| `?appointmentFee[gte]=500&appointmentFee[lte]=1500` | range filter |
| `?designation[contains]=consult` | partial match (case-insensitive) |
| `?designation=Consultant` বা `consultant` | exact match, case ignore (`caseInsensitiveFields`) |
| `?specialties.specialty.title=cardiology` | nested relation filter, case ignore |
| `?sortBy=appointmentFee&sortOrder=asc` (বা `ASC`) | sort (default: `createdAt desc`) |
| `?page=2&limit=5` | pagination (default limit 10, max 100) |
| `?fields=name,email` | শুধু ওই field গুলো আসবে (include বাদ) |
| `?include=reviews,doctorSchedules` | বাড়তি relation আসবে |

> ✏️ আগে উদাহরণ ছিল `?name[contains]=ra`, কিন্তু doctor-এর `filterableFields`-এ `name` নেই — তাই ওটা ignore হতো। `designation` দিয়ে বদলানো হয়েছে।

Allowed operators: `equals, not, lt, lte, gt, gte, in, notIn, contains, startsWith, endsWith`

**ভুল / অচেনা জিনিস চুপচাপ ignore হয়:**

- `?sortBy=password` → `createdAt` দিয়ে sort
- `?page=-3&limit=99999` → page 1, limit 100
- `?isDeleted=true` → whitelist-এ নেই, ignore
- `?appointmentFee[hack]=x` → operator বাদ
- `?include=hackerTable` → config-এ নেই, বাদ

**ভুল type-এর value → `400`** (ধাপ ৯): `?experience=abc`, `?gender=abc`

## Path rule (খুব জরুরি)

| path | মানে | Prisma |
|---|---|---|
| `name` | নিজের field | `{ name: ... }` |
| `user.email` | ২ ভাগ = **to-one** relation | `{ user: { email: ... } }` |
| `specialties.specialty.title` | ৩ ভাগ = **to-many** relation | `{ specialties: { some: { specialty: { title } } } }` |

- ⚠️ to-many relation-এ ২ ভাগের path দিও না (যেমন `specialties.specialtyId`) → error
- ⚠️ `sortableFields`-এ শুধু নিজের field বা to-one path রাখো
- ⚠️ সংখ্যা-ওয়ালা String column (`contactNumber`, `registrationNumber`) `filterableFields`-এ দিও না, কারণ value auto number হয়ে যায়
- ⚠️ `enumFields` আর `caseInsensitiveFields`-এর field `filterableFields`-এও থাকতে হবে, নাহলে কাজ করবে না
- ⚠️ Model-এ `id` না থাকলে (composite key) `tieBreakerFields` দিতেই হবে, যেমন `["doctorId", "scheduleId"]` — নাহলে sort-এ Prisma error

## কোথায় কী যোগ করতে হয়

### একবারই (setup)

1. `src/app.ts` → `app.set("query parser", "extended");` (`[gte]` বোঝার জন্য)
2. `src/app/interfaces/query.interface.ts` → `IQueryParams`, `IQueryConfig`, `IQueryMeta`, `IQueryResult`, `PrismaModelDelegate`
3. `src/app/utils/QueryBuilder.ts` → class
4. `src/app/shared/sendResponse.ts` → `meta?: IQueryMeta` যোগ
5. `src/app/middleware/globalErrorHandler.ts` → `PrismaClientValidationError` → `400`

### প্রতিটা নতুন module-এ (যেমন patient)

**1. `patient.constant.ts`**

```ts
export const patientSearchableFields = ["name", "email"] as const;
export const patientFilterableFields = ["gender"] as const;
export const patientSortableFields = ["createdAt", "name"] as const;
export const patientSelectableFields = ["id", "name", "email"] as const;
export const patientEnumFields = ["gender"] as const;
export const patientIncludeConfig = { appointments: true } satisfies Prisma.PatientInclude;
```

**2. `patient.service.ts`**

```ts
const patientInclude = { user: true } as const;
type PatientType = Prisma.PatientGetPayload<{ include: typeof patientInclude }>;

const getAllPatients = async (query: IQueryParams) => {
	return new QueryBuilder<PatientType, Prisma.PatientWhereInput, Prisma.PatientInclude>(prisma.patient, query, {
		searchableFields: patientSearchableFields,
		filterableFields: patientFilterableFields,
		sortableFields: patientSortableFields,
		selectableFields: patientSelectableFields,
		enumFields: patientEnumFields,
	})
		.where({ isDeleted: false })
		.search()
		.filter()
		.sort()
		.paginate()
		.fields()
		.include(patientInclude)
		.dynamicInclude(patientIncludeConfig)
		.execute();
};
```

> ✏️ `PatientType` আর `patientEnumFields` আগের উদাহরণে define করা ছিল না — যোগ করা হলো।

**3. `patient.controller.ts`**

```ts
const result = await PatientService.getAllPatients(req.query);
sendResponse(res, {
	httpStatusCode: httpStatus.OK,
	success: true,
	message: "Patients fetched successfully",
	meta: result.meta,
	data: result.data,
});
```

Route-এ কিছু বদলাতে হয় না।

## Response shape

```json
{
	"success": true,
	"message": "...",
	"meta": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 },
	"data": [ ... ]
}
```

## মনে রাখার পয়েন্ট

- `include` = সবসময় আসে, `dynamicInclude` = client চাইলে আসে
- `id` নেই এমন model → `tieBreakerFields` দাও
- `fields` দিলে include বাদ পড়ে (Prisma `select` + `include` একসাথে নেয় না)
- `enumFields` → `male` লিখলেও `MALE`; `caseInsensitiveFields` → `cardiology` == `Cardiology`
- `[gte]` কাজ না করলে → `app.ts`-এ query parser চেক করো
- curl-এ test করলে → `curl -g "URL"`
- Code লেখার পর → `pnpm check`

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### ✅ (Done)

1. ✅ **ভুল type-এর value দিলে `500` আসত** — `globalErrorHandler`-এ `PrismaClientValidationError` → `400` (ধাপ ৯)। (untested)

### 🟡 ভালো practice

2. **`fields` দিলে type ভুল বলে।** `getAllDoctors`-এর return type `DoctorWithRelations` (user, specialties সহ), কিন্তু `?fields=name,email` দিলে `data`-তে শুধু ওই দুটো field থাকে। Backend-এ চলবে, তবে frontend-এ এই type ধরে `doctor.user.email` লিখলে runtime-এ `undefined`। Frontend type-এ relation গুলো optional ধরো, বা মনে রাখো।

3. **`searchTerm`-এ বড় শব্দ।** খুব বড় `searchTerm` দিলে ৬টা field-এ `contains` (+ relation join) চলে — DB-তে ভারী। `.slice(0, 100)` দিয়ে লম্বা কেটে দাও:

	```ts
	const term = this.getString("searchTerm")?.trim().slice(0, 100);
	```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

4. **`Infinity` / `1e3`.** `parseScalar` `Number()` ব্যবহার করে, তাই `"1e3"` → `1000`, `"Infinity"` → `Infinity`। `Infinity` Prisma Int-এ গেলে error — ধাপ ৯-এর handler `400` দেবে।
5. **`findMany` আর `count` আলাদা query।** দুটোর মাঝে নতুন doctor যোগ হলে `total` একটু অন্যরকম হতে পারে — pagination-এ এটা সাধারণত মেনে নেওয়া হয়।
6. **Index।** `appointmentFee`, `experience`, `averageRating`-এ sort/filter বেশি হলে Prisma schema-তে `@@index([...])` দিলে বড় data-তে দ্রুত হবে।

## 🔜 পরে যা যোগ হবে

- [ ] Patient, admin module-এ একই QueryBuilder
- [ ] Specialty `getAll`-এ QueryBuilder
- [ ] Frontend-এ pagination + filter UI (`meta` দিয়ে)
