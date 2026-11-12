shared/commonValidation.ts
```ts
import * as z from "zod";

export const idParamSchema = z.object({ id: z.uuid("Invalid id") });
```

২. specialty.validation.ts (পুরো ফাইল)

```ts
import * as z from "zod";

const nameSchema = z
	.string()
	.trim()
	.min(2, "Name must be at least 2 characters")
	.max(100)
	.regex(/[a-z0-9]/i, "Name must contain letters or numbers");

const createSpecialtySchema = z.object({
	name: nameSchema,
	description: z.string().trim().max(500).optional(),
	icon: z.string().trim().max(255).optional(),
});

const updateSpecialtySchema = z
	.object({
		name: nameSchema.optional(),
		description: z.string().trim().max(500).nullable().optional(),
		icon: z.string().trim().max(255).nullable().optional(),
		isActive: z.boolean().optional(),
	})
	.refine((data) => Object.keys(data).length > 0, {
		error: "Provide at least one field to update",
	});

const getSpecialtiesQuerySchema = z.object({
	page: z.coerce.number().int().min(1).default(1),
	limit: z.coerce.number().int().min(1).max(100).default(10),
	searchTerm: z.string().trim().max(100).optional(),
	isActive: z.stringbool().optional(),
	sortBy: z.enum(["name", "createdAt"]).default("name"),
	sortOrder: z.enum(["asc", "desc"]).default("asc"),
});

export const SpecialtyValidation = {
	createSpecialtySchema,
	updateSpecialtySchema,
	getSpecialtiesQuerySchema,
};
```

specialty.interface.ts

```ts
import type * as z from "zod";
import type { SpecialtyValidation } from "./specialty.validation";

export type TCreateSpecialtyPayload = z.infer<typeof SpecialtyValidation.createSpecialtySchema>;
export type TUpdateSpecialtyPayload = z.infer<typeof SpecialtyValidation.updateSpecialtySchema>;
export type TSpecialtyQuery = z.infer<typeof SpecialtyValidation.getSpecialtiesQuerySchema>;
```
