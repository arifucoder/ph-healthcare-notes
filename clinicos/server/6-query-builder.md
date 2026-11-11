
```text
src/app/shared/query/
├─ query.interface.ts
├─ queryBuilder.ts
└─ queryBuilder.test.ts
src/app/module/specialty/specialty.constant.ts   (নতুন)
specialty.service.ts / controller / validation / interface   (update)
```

১. shared/query/query.interface.ts
```ts
export type TSortOrder = "asc" | "desc";

// value কোন type-এ convert হবে — builder কখনো আন্দাজ করে না
export type TFilterType = "string" | "number" | "boolean" | "enum" | "uuid";

// equals: ঠিক মিল | contains: আংশিক মিল (string) | in: একাধিক value | gte/lte: range
export type TFilterOperator = "equals" | "contains" | "in" | "gte" | "lte";

export type TFilterDef = {
	// Prisma path. to-many relation হলে "some" নিজে লিখবে: "specialties.some.specialty.slug"
	path: string;
	type: TFilterType;
	op?: TFilterOperator; // default "equals"
	values?: readonly string[]; // শুধু enum-এর জন্য
};

export type TListConfig = {
	searchableFields?: readonly string[];
	filters?: Readonly<Record<string, TFilterDef>>; // key = URL-এর query key
	sortableFields: readonly string[];
	defaultSortBy: string;
	defaultSortOrder?: TSortOrder;
	defaultLimit?: number;
	maxLimit?: number;
};

// app start-এ একবার তৈরি হয় — প্রতি request-এ split/Set বানানোর খরচ নেই
export type TCompiledFilter = TFilterDef & {
	key: string;
	parts: readonly string[];
	op: TFilterOperator;
	allowed?: ReadonlySet<string>;
};

export type TCompiledListConfig<TWhere, TOrderBy> = {
	searchPaths: readonly (readonly string[])[];
	filters: readonly TCompiledFilter[];
	sortable: ReadonlyMap<string, readonly string[]>;
	defaultSortBy: string;
	defaultSortOrder: TSortOrder;
	defaultLimit: number;
	maxLimit: number;
	// শুধু type বহন করে (phantom), runtime-এ থাকে না
	readonly __types?: { where: TWhere; orderBy: TOrderBy };
};

export type TListQuery<TWhere, TOrderBy> = {
	where: TWhere;
	args: { where: TWhere; orderBy: TOrderBy[]; skip: number; take: number };
	page: number;
	limit: number;
};
```

২. shared/query/queryBuilder.ts
```ts
import { AppError } from "@/app/errorHelpers/AppError";
import type { TMeta } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import * as z from "zod";
import type {
	TCompiledFilter,
	TCompiledListConfig,
	TFilterDef,
	TListConfig,
	TListQuery,
	TSortOrder,
} from "./query.interface";

type AnyObject = Record<string, unknown>;

const MAX_IN_VALUES = 50;

// সব list endpoint-এর common query — একবারই define, প্রতি request-এ শুধু parse
const baseQuerySchema = z.object({
	page: z.coerce.number().int().min(1).default(1),
	limit: z.coerce.number().int().min(1).optional(),
	sortBy: z.string().trim().optional(),
	sortOrder: z.enum(["asc", "desc"]).optional(),
	searchTerm: z.string().trim().max(100).optional(),
});

const uuidSchema = z.uuid();

const badRequest = (message: string) => new AppError(httpStatus.BAD_REQUEST, message);

// ["specialties", "some", "specialty", "slug"] + leaf
//   -> { specialties: { some: { specialty: { slug: leaf } } } }
const nest = (parts: readonly string[], leaf: unknown): AnyObject => {
	const lastKey = parts.at(-1) ?? "";
	return parts.slice(0, -1).reduceRight<AnyObject>((node, key) => ({ [key]: node }), { [lastKey]: leaf });
};

const isPlainObject = (value: unknown): value is AnyObject =>
	typeof value === "object" && value !== null && !Array.isArray(value);

const splitPath = (path: string): string[] => {
	const parts = path.split(".");
	if (parts.some((part) => part.trim() === "")) throw new Error(`Invalid path "${path}"`);
	return parts;
};

// ---------- config: app start-এ একবার ----------

export const defineListConfig = <TWhere, TOrderBy>(config: TListConfig): TCompiledListConfig<TWhere, TOrderBy> => {
	const sortable = new Map(config.sortableFields.map((field) => [field, splitPath(field)] as const));

	// ভুল config হলে server start-এই crash — production-এ request আসার আগেই ধরা পড়ে
	if (!sortable.has(config.defaultSortBy)) {
		throw new Error(`defaultSortBy "${config.defaultSortBy}" must be in sortableFields`);
	}

	const filters = Object.entries(config.filters ?? {}).map(([key, def]): TCompiledFilter => {
		const op = def.op ?? "equals";

		if (def.type === "enum" && !def.values?.length) {
			throw new Error(`Filter "${key}": enum type needs "values"`);
		}
		if (op === "contains" && def.type !== "string") {
			throw new Error(`Filter "${key}": "contains" works only with string type`);
		}

		return {
			...def,
			key,
			op,
			parts: splitPath(def.path),
			...(def.values ? { allowed: new Set(def.values) } : {}),
		};
	});

	return {
		searchPaths: (config.searchableFields ?? []).map(splitPath),
		filters,
		sortable,
		defaultSortBy: config.defaultSortBy,
		defaultSortOrder: config.defaultSortOrder ?? "desc",
		defaultLimit: config.defaultLimit ?? 10,
		maxLimit: config.maxLimit ?? 100,
	};
};

// ---------- value parsing: config-এর type অনুযায়ী, আন্দাজ নয় ----------

const parseScalar = (raw: string, filter: TFilterDef & { key: string; allowed?: ReadonlySet<string> }): unknown => {
	const value = raw.trim();

	switch (filter.type) {
		case "string":
			return value; // "01712..." string-ই থাকে, শুরুর 0 হারায় না

		case "number": {
			const num = Number(value);
			if (value === "" || !Number.isFinite(num)) throw badRequest(`"${filter.key}" must be a number`);
			return num;
		}

		case "boolean":
			if (value === "true") return true;
			if (value === "false") return false;
			throw badRequest(`"${filter.key}" must be true or false`);

		case "enum": {
			const upper = value.toUpperCase();
			if (!filter.allowed?.has(upper)) {
				throw badRequest(`"${filter.key}" must be one of: ${[...(filter.allowed ?? [])].join(", ")}`);
			}
			return upper;
		}

		case "uuid":
			if (!uuidSchema.safeParse(value).success) throw badRequest(`"${filter.key}" must be a valid id`);
			return value;

		default: {
			// নতুন TFilterType যোগ করে এখানে case না লিখলে TypeScript compile error দেবে
			const unhandled: never = filter.type;
			throw new Error(`Unhandled filter type: ${String(unhandled)}`);
		}
	}
};

const buildFilterCondition = (raw: unknown, filter: TCompiledFilter): AnyObject | undefined => {
	if (raw === undefined || raw === "") return undefined;

	// ?status=A,B অথবা ?status=A&status=B — দুটোই চলবে, শুধু "in"-এ
	if (filter.op === "in") {
		const list = (Array.isArray(raw) ? raw : [raw])
			.flatMap((item) => (typeof item === "string" ? item.split(",") : []))
			.filter((item) => item.trim() !== "");

		if (list.length === 0) return undefined;
		if (list.length > MAX_IN_VALUES) throw badRequest(`"${filter.key}" accepts at most ${MAX_IN_VALUES} values`);

		return nest(filter.parts, { in: list.map((item) => parseScalar(item, filter)) });
	}

	if (typeof raw !== "string") throw badRequest(`"${filter.key}" must be a single value`);

	const value = parseScalar(raw, filter);

	switch (filter.op) {
		case "contains":
			return nest(filter.parts, { contains: value, mode: "insensitive" });
		case "gte":
			return nest(filter.parts, { gte: value });
		case "lte":
			return nest(filter.parts, { lte: value });
		default:
			return nest(filter.parts, value);
	}
};

// ---------- main ----------

export const buildListQuery = <TWhere, TOrderBy>(
	rawQuery: unknown,
	config: TCompiledListConfig<TWhere, TOrderBy>,
	baseWhere?: TWhere, // code থেকে জোর করা condition: { isDeleted: false }, { patientId } ...
): TListQuery<TWhere, TOrderBy> => {
	const query: AnyObject = isPlainObject(rawQuery) ? rawQuery : {};
	const base = baseQuerySchema.parse(query); // ভুল হলে ZodError → 400

	const conditions: unknown[] = [];
	if (baseWhere) conditions.push(baseWhere);

	// search: যেকোনো একটা field-এ মিললেই
	if (base.searchTerm && config.searchPaths.length > 0) {
		const leaf = { contains: base.searchTerm, mode: "insensitive" };
		conditions.push({ OR: config.searchPaths.map((parts) => nest(parts, leaf)) });
	}

	// filter: শুধু config-এর whitelist; অচেনা query key (যেমন ?_t=123) উপেক্ষা
	for (const filter of config.filters) {
		const condition = buildFilterCondition(query[filter.key], filter);
		if (condition) conditions.push(condition);
	}

	// sort: whitelist-এ না থাকলে 400; id tie-breaker দিলে page বদলালে row duplicate/miss হয় না
	const sortBy = base.sortBy || config.defaultSortBy;
	const sortParts = config.sortable.get(sortBy);
	if (!sortParts) {
		throw badRequest(`Cannot sort by "${sortBy}". Allowed: ${[...config.sortable.keys()].join(", ")}`);
	}
	const sortOrder: TSortOrder = base.sortOrder ?? config.defaultSortOrder;
	const orderBy = [nest(sortParts, sortOrder)];
	if (sortBy !== "id") orderBy.push({ id: sortOrder });

	// pagination: limit সীমার বেশি চাইলে সর্বোচ্চটা দিই
	const page = base.page;
	const limit = Math.min(base.limit ?? config.defaultLimit, config.maxLimit);

	// condition একটা হলে AND দরকার নেই — Prisma-কে ছোট query দিই
	const whereObject = conditions.length === 0 ? {} : conditions.length === 1 ? conditions[0] : { AND: conditions };

	// type boundary: runtime-এ object আমরা config থেকে বানিয়েছি, Prisma-র type এখানে একবারই বসাই
	// oxlint-disable-next-line typescript/no-unsafe-type-assertion
	const where = whereObject as TWhere;
	// oxlint-disable-next-line typescript/no-unsafe-type-assertion
	const typedOrderBy = orderBy as TOrderBy[];

	return {
		where,
		args: { where, orderBy: typedOrderBy, skip: (page - 1) * limit, take: limit },
		page,
		limit,
	};
};

export const buildMeta = (total: number, page: number, limit: number): TMeta => ({
	page,
	limit,
	total,
	totalPages: Math.ceil(total / limit),
});
```

shared/query/queryBuilder.test.ts

```ts
import { describe, expect, it } from "vitest";
import { buildListQuery, buildMeta, defineListConfig } from "./queryBuilder";

const config = defineListConfig({
	searchableFields: ["name", "user.email"],
	filters: {
		isActive: { path: "isActive", type: "boolean" },
		phone: { path: "phone", type: "string" },
		gender: { path: "gender", type: "enum", values: ["MALE", "FEMALE", "OTHER"] },
		status: { path: "status", type: "enum", values: ["CONFIRMED", "CHECKED_IN"], op: "in" },
		specialty: { path: "specialties.some.specialty.slug", type: "string" },
		minFee: { path: "feeCents", type: "number", op: "gte" },
		maxFee: { path: "feeCents", type: "number", op: "lte" },
	},
	sortableFields: ["name", "createdAt", "user.name"],
	defaultSortBy: "createdAt",
});

describe("defineListConfig", () => {
	it("throws at startup when defaultSortBy is not sortable", () => {
		expect(() => defineListConfig({ sortableFields: ["name"], defaultSortBy: "createdAt" })).toThrow();
	});
	it("throws when enum filter has no values", () => {
		expect(() =>
			defineListConfig({ filters: { g: { path: "g", type: "enum" } }, sortableFields: ["id"], defaultSortBy: "id" }),
		).toThrow();
	});
});

describe("buildListQuery", () => {
	it("uses defaults and baseWhere", () => {
		const q = buildListQuery({}, config, { isDeleted: false });
		expect(q.where).toEqual({ isDeleted: false });
		expect(q.args.orderBy).toEqual([{ createdAt: "desc" }, { id: "desc" }]);
		expect(q.args).toMatchObject({ skip: 0, take: 10 });
	});

	it("keeps phone as string (leading zero safe)", () => {
		const q = buildListQuery({ phone: "01712345678" }, config);
		expect(q.where).toEqual({ phone: "01712345678" });
	});

	it("parses booleans strictly", () => {
		expect(buildListQuery({ isActive: "false" }, config).where).toEqual({ isActive: false });
		expect(() => buildListQuery({ isActive: "yes" }, config)).toThrow(/true or false/);
	});

	it("uppercases and validates enums", () => {
		expect(buildListQuery({ gender: "male" }, config).where).toEqual({ gender: "MALE" });
		expect(() => buildListQuery({ gender: "xyz" }, config)).toThrow(/must be one of/);
	});

	it("supports in with comma list and repeated keys", () => {
		const expected = { status: { in: ["CONFIRMED", "CHECKED_IN"] } };
		expect(buildListQuery({ status: "confirmed,checked_in" }, config).where).toEqual(expected);
		expect(buildListQuery({ status: ["confirmed", "checked_in"] }, config).where).toEqual(expected);
	});

	it("builds to-many relation path with some", () => {
		expect(buildListQuery({ specialty: "cardiology" }, config).where).toEqual({
			specialties: { some: { specialty: { slug: "cardiology" } } },
		});
	});

	it("combines range, search and base with AND", () => {
		const q = buildListQuery({ minFee: "500", maxFee: "1000", searchTerm: " rahim " }, config, { isDeleted: false });
		expect(q.where).toEqual({
			AND: [
				{ isDeleted: false },
				{
					OR: [
						{ name: { contains: "rahim", mode: "insensitive" } },
						{ user: { email: { contains: "rahim", mode: "insensitive" } } },
					],
				},
				{ feeCents: { gte: 500 } },
				{ feeCents: { lte: 1000 } },
			],
		});
	});

	it("rejects non-number, unknown sort and repeated single value", () => {
		expect(() => buildListQuery({ minFee: "abc" }, config)).toThrow(/number/);
		expect(() => buildListQuery({ sortBy: "password" }, config)).toThrow(/Cannot sort/);
		expect(() => buildListQuery({ phone: ["1", "2"] }, config)).toThrow(/single value/);
	});

	it("nests relation sort and clamps limit", () => {
		const q = buildListQuery({ sortBy: "user.name", sortOrder: "asc", page: "3", limit: "500" }, config);
		expect(q.args.orderBy).toEqual([{ user: { name: "asc" } }, { id: "asc" }]);
		expect(q.args).toMatchObject({ skip: 200, take: 100 });
	});

	it("rejects bad page", () => {
		expect(() => buildListQuery({ page: "abc" }, config)).toThrow();
		expect(() => buildListQuery({ page: "0" }, config)).toThrow();
	});

	it("ignores unknown keys", () => {
		expect(buildListQuery({ _t: "123", hack: "x" }, config).where).toEqual({});
	});
});

describe("buildMeta", () => {
	it("computes totalPages", () => {
		expect(buildMeta(25, 1, 10)).toEqual({ page: 1, limit: 10, total: 25, totalPages: 3 });
		expect(buildMeta(0, 1, 10).totalPages).toBe(0);
	});
});
```

Vitest এখনো install করা নেই (তোমার package.json-এ দেখলাম)। Setup করতে:

pnpm add -D vitest চালাও।
package.json-এ script যোগ করো: "test": "vitest run"।
vitest.config.ts বানাও:

```ts
   import { fileURLToPath } from "node:url";
   import { defineConfig } from "vitest/config";

   export default defineConfig({
   	resolve: { alias: { "@": fileURLToPath(new URL("./src", import.meta.url)) } },
   });
```
tsconfig.json-এর include-এ "vitest.config.ts" যোগ করো।