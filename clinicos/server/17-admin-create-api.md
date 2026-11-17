# ClinicOS — Admin তৈরি ও Welcome Email

Admin আর doctor create করার সময় create করার পাশাপাশি আমরা password email-এ পাঠাবো, এজন্য আমরা একটা email template add করবো।

## ১. Email template

**File:** `templates/emails/staffWelcome.ejs`

```ejs
<h1
	class="h1"
	style="
		margin: 0 0 12px 0;
		font-family: Arial, Helvetica, sans-serif;
		font-size: 26px;
		line-height: 32px;
		font-weight: bold;
		color: #0f172a;
	"
>
	Welcome to <%= clinicName %>
</h1>

<p style="margin: 0 0 24px 0">
	Hi <%= name %>, an <strong style="color: #0f172a"><%= roleLabel %></strong> account has been created for you. Use
	the details below to sign in. You will be asked to set your own password the first time.
</p>

<!-- credentials box -->
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td style="padding: 20px 24px; background-color: #eff4ff; border: 1px solid #c7d7fe; border-radius: 12px">
			<div
				style="
					font-family: Arial, Helvetica, sans-serif;
					font-size: 12px;
					line-height: 16px;
					font-weight: bold;
					letter-spacing: 2px;
					color: #2563eb;
				"
			>
				EMAIL
			</div>
			<div
				style="
					padding: 4px 0 16px 0;
					font-family: Arial, Helvetica, sans-serif;
					font-size: 16px;
					line-height: 22px;
					color: #0f172a;
				"
			>
				<%= email %>
			</div>
			<div
				style="
					font-family: Arial, Helvetica, sans-serif;
					font-size: 12px;
					line-height: 16px;
					font-weight: bold;
					letter-spacing: 2px;
					color: #2563eb;
				"
			>
				TEMPORARY PASSWORD
			</div>
			<div
				style="
					padding-top: 4px;
					font-family: 'Courier New', Courier, monospace;
					font-size: 22px;
					line-height: 30px;
					font-weight: bold;
					letter-spacing: 2px;
					color: #0f172a;
				"
			>
				<%= temporaryPassword %>
			</div>
		</td>
	</tr>
</table>

<!-- button -->
<table role="presentation" cellpadding="0" cellspacing="0" border="0" style="margin: 24px 0">
	<tr>
		<td style="background-color: #2563eb; border-radius: 8px">
			<a
				href="<%= loginUrl %>"
				style="
					display: inline-block;
					padding: 12px 24px;
					font-family: Arial, Helvetica, sans-serif;
					font-size: 15px;
					font-weight: bold;
					color: #ffffff;
					text-decoration: none;
				"
				>Sign in</a
			>
		</td>
	</tr>
</table>

<!-- security note -->
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td
			style="
				padding: 14px 16px;
				background-color: #f8fafc;
				border-radius: 10px;
				font-family: Arial, Helvetica, sans-serif;
				font-size: 14px;
				line-height: 21px;
				color: #475569;
			"
		>
			<strong style="color: #0f172a">Keep this private.</strong> This password works only until you change it. If
			you did not expect this account, contact <%= clinicName %>.
		</td>
	</tr>
</table>
```

## ২. Template-এর type

**File:** `shared/renderEmail.ts`

```ts
export type TEmailTemplates = {
	verificationCode: {
		name: string;
		code: string;
		expiresInMinutes: number;
		expiresAtLabel: string;
	};
	staffWelcome: {
		name: string;
		roleLabel: string;
		email: string;
		temporaryPassword: string;
		loginUrl: string;
	};
};
```

## ৩. User module

### ৩.১ Validation

**File:** `user.validation.ts`

```ts
import { emailSchema, personNameSchema, phoneSchema } from "@/app/shared/commonValidation";
import * as z from "zod";

const createAdminSchema = z.object({
	name: personNameSchema,
	email: emailSchema,
	phone: phoneSchema.optional(),
});

export const UserValidation = {
	createAdminSchema,
};
```

### ৩.২ Interface

**File:** `user.interface.ts`

```ts
import type * as z from "zod";
import type { UserValidation } from "./user.validation";

export type TCreateAdminPayload = z.infer<typeof UserValidation.createAdminSchema>;
```

### ৩.৩ Service

**File:** `user.service.ts`

```ts
import { envVars } from "@/app/config/env";
import { AppError } from "@/app/errorHelpers/AppError";
import { prisma } from "@/app/lib/prisma";
import { sendEmail } from "@/app/shared/sendEmail";
import { Role } from "@/generated/prisma/enums";
import { hashPassword } from "better-auth/crypto";
import httpStatus from "http-status";
import { randomBytes } from "node:crypto";
import type { TCreateAdminPayload } from "./user.interface";

const userSelect = {
	id: true,
	name: true,
	email: true,
	phone: true,
	role: true,
	isActive: true,
	needPasswordChange: true,
	createdAt: true,
} as const;

const createAdmin = async (payload: TCreateAdminPayload) => {
	const { name, email, phone } = payload;

	const existingUser = await prisma.user.findUnique({
		where: { email },
		select: { id: true },
	});

	if (existingUser) {
		throw new AppError(httpStatus.CONFLICT, "An account with this email already exists");
	}

	const temporaryPassword = randomBytes(9).toString("base64url");
	const hashedPassword = await hashPassword(temporaryPassword);
	const userId = crypto.randomUUID();

	const user = await prisma.$transaction(async (tx) => {
		const createdUser = await tx.user.create({
			data: {
				id: userId,
				name,
				email,
				phone: phone ?? null,
				role: Role.ADMIN,
				emailVerified: true,
				needPasswordChange: true,
			},
			select: userSelect,
		});

		await tx.account.create({
			data: {
				id: crypto.randomUUID(),
				userId,
				accountId: userId,
				providerId: "credential",
				password: hashedPassword,
			},
		});

		return createdUser;
	});

	let emailSent = true;

	try {
		await sendEmail({
			to: email,
			subject: `Your ${envVars.CLINIC_NAME} admin account`,
			previewText: "Your account is ready. Sign in with the temporary password inside.",
			template: "staffWelcome",
			data: {
				name,
				roleLabel: "Admin",
				email,
				temporaryPassword,
				loginUrl: `${envVars.FRONTEND_URL}/login`,
			},
		});
	} catch (error) {
		console.error(`Failed to send welcome email to user ${user.id}`, error);
		emailSent = false;
	}

	return { user, emailSent };
};

export const UserService = {
	createAdmin,
};
```

### ৩.৪ Controller

**File:** `user.controller.ts`

```ts
import { catchAsync } from "@/app/shared/catchAsync";
import { sendResponse } from "@/app/shared/sendResponse";
import httpStatus from "http-status";
import { UserService } from "./user.service";

const createAdmin = catchAsync(async (req, res) => {
	const result = await UserService.createAdmin(req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: result.emailSent
			? "Admin created. Login details have been sent to their email"
			: "Admin created, but the welcome email could not be sent. Reset their password to send it again",
		data: result,
	});
});

export const UserController = {
	createAdmin,
};
```

### ৩.৫ Route

**File:** `user.route.ts`

```ts
import { checkAuth } from "@/app/middleware/checkAuth";
import { validateRequest } from "@/app/middleware/validateRequest";
import { Role } from "@/generated/prisma/enums";
import { Router } from "express";
import { UserController } from "./user.controller";
import { UserValidation } from "./user.validation";

const router = Router();

router.post(
	"/admins",
	checkAuth(Role.SUPER_ADMIN),
	validateRequest(UserValidation.createAdminSchema),
	UserController.createAdmin,
);

export const UserRoutes = router;
```

### ৩.৬ Route register করা

**File:** `routes/index.ts`

```ts
import { UserRoutes } from "@/app/module/user/user.route";

// moduleRoutes-এর ভেতরে
{ path: "/users", route: UserRoutes },
```