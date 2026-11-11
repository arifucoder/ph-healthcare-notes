apps/api/src/app/middleware/validateRequest.ts

```ts
import type { NextFunction, Request, RequestHandler, Response } from "express";
import type { ZodType } from "zod";

// শুধু req.body validate করে; Express 5-এ req.query read-only, তাই params/query controller-এ parse হবে
export const validateRequest = (schema: ZodType): RequestHandler => {
	return async (req: Request, _res: Response, next: NextFunction) => {
		const result = await schema.safeParseAsync(req.body);

		if (!result.success) {
			// ZodError → globalErrorHandler → handleZodError → 400
			return next(result.error);
		}

		// parsed data দিয়ে replace: unknown key বাদ, trim/transform প্রয়োগ হয়
		req.body = result.data;
		next();
	};
};
```