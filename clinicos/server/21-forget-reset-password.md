১. apps/api/templates/emails/passwordResetCode.ejs (নতুন file)
```html
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
	Reset your password
</h1>

<p style="margin: 0 0 24px 0">
	Hi <%= name %>, we received a request to reset the password for your <%= clinicName %> account. Enter this code on
	the reset screen and choose a new password.
</p>

<!-- code box -->
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td
			align="center"
			style="padding: 24px 16px; background-color: #eff4ff; border: 1px solid #c7d7fe; border-radius: 12px"
		>
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
				YOUR RESET CODE
			</div>
			<div
				class="code"
				style="
					padding: 10px 0 6px 10px;
					font-family: Arial, Helvetica, sans-serif;
					font-size: 40px;
					line-height: 48px;
					font-weight: bold;
					letter-spacing: 10px;
					color: #0f172a;
				"
			>
				<%= code %>
			</div>
			<div style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; line-height: 20px; color: #334155">
				Expires in <strong><%= expiresInMinutes %> minutes</strong> &middot; at <%= expiresAtLabel %>
			</div>
		</td>
	</tr>
</table>

<p style="margin: 24px 0">
	After you reset your password, you will be signed out on every device and need to log in again with the new password.
</p>

<!-- security note -->
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="border-collapse: separate">
	<tr>
		<td
			style="
				padding: 14px 16px;
				background-color: #fff1f2;
				border-radius: 10px;
				font-family: Arial, Helvetica, sans-serif;
				font-size: 14px;
				line-height: 21px;
				color: #475569;
			"
		>
			<strong style="color: #0f172a">Didn't ask for this?</strong> You can ignore this email &mdash; your password
			stays the same without the code. <%= clinicName %> staff will never ask you for this code by phone or message.
		</td>
	</tr>
</table>
```

২. src/app/shared/renderEmail.ts: TEmailTemplates-এ যোগ
```ts
export type TEmailTemplates = {
	verificationCode: {
		name: string;
		code: string;
		expiresInMinutes: number;
		expiresAtLabel: string;
	};
	passwordResetCode: {
		name: string;
		code: string;
		expiresInMinutes: number;
		expiresAtLabel: string;
	};
	staffWelcome: {
		// ...আগের মতোই
	};
};
```

৩. src/app/lib/auth.ts: দুই জায়গায় বদল
(ক) emailAndPassword:
```ts
	emailAndPassword: {
		enabled: true,
		requireEmailVerification: true,
		revokeSessionsOnPasswordReset: true,

		// Better Auth reset সফল হওয়ার পর এটা চালায়
		async onPasswordReset({ user }) {
			await prisma.user.updateMany({
				where: { id: user.id, needPasswordChange: true },
				data: { needPasswordChange: false },
			});
		},
	},
```

(খ) emailOTP-এর ভেতরে পুরো sendVerificationOTP এটা দিয়ে বদলাও:
```ts
			async sendVerificationOTP({ email, otp, type }) {
				if (type !== "email-verification" && type !== "forget-password") return;

				const user = await prisma.user.findUnique({
					where: { email },
					select: { name: true, emailVerified: true },
				});
				if (!user) return;
				if (type === "email-verification" && user.emailVerified) return;

				const expiresAt = new Date(Date.now() + OTP_EXPIRES_MINUTES * 60_000);
				const data = {
					name: user.name,
					code: otp,
					expiresInMinutes: OTP_EXPIRES_MINUTES,
					expiresAtLabel: formatClinicTime(expiresAt),
				};

				const sending =
					type === "email-verification"
						? sendEmail({
								to: email,
								subject: `${otp} is your ${envVars.CLINIC_NAME} verification code`,
								previewText: `Use this code to verify your email. It expires in ${OTP_EXPIRES_MINUTES} minutes.`,
								template: "verificationCode",
								data,
							})
						: sendEmail({
								to: email,
								subject: `${otp} is your ${envVars.CLINIC_NAME} password reset code`,
								previewText: `Use this code to reset your password. It expires in ${OTP_EXPIRES_MINUTES} minutes.`,
								template: "passwordResetCode",
								data,
							});

				// await নেই: response-এর সময় দেখে যেন বোঝা না যায় email-টা আছে কিনা
				void sending.catch((error: unknown) => {
					console.error(`Failed to send ${type} email`, error);
				});
			},
```

৪. auth.validation.ts
```ts
const otpSchema = z.string().regex(/^\d{6}$/, "Code must be exactly 6 digits");

const verifyEmailSchema = z.object({
	email: emailSchema,
	otp: otpSchema,
});

// ...resendOtpSchema, loginSchema, changePasswordSchema আগের মতো

const forgetPasswordSchema = z.object({
	email: emailSchema,
});

const resetPasswordSchema = z.object({
	email: emailSchema,
	otp: otpSchema,
	newPassword: passwordSchema,
});

export const AuthValidation = {
	// ...আগেরগুলো
	forgetPasswordSchema,
	resetPasswordSchema,
};
```

৫. auth.interface.ts
```ts
export type TForgetPasswordPayload = z.infer<typeof AuthValidation.forgetPasswordSchema>;
export type TResetPasswordPayload = z.infer<typeof AuthValidation.resetPasswordSchema>;
```

৬. auth.service.ts

import-এ দুটো type যোগ করো (TForgetPasswordPayload, TResetPasswordPayload), তারপর:
```ts
type TResetCheckUser = { isActive: boolean; isDeleted: boolean; emailVerified: boolean } | null;

const canResetPassword = (user: TResetCheckUser) =>
	user !== null && user.isActive && !user.isDeleted && user.emailVerified;

const forgetPassword = async (payload: TForgetPasswordPayload) => {
	const { email } = payload;

	const user = await prisma.user.findUnique({
		where: { email },
		select: { isActive: true, isDeleted: true, emailVerified: true },
	});

	// account না থাকলে বা অযোগ্য হলে চুপচাপ ফেরত — response একই থাকে
	if (!canResetPassword(user)) return;

	await auth.api.requestPasswordResetEmailOTP({
		body: { email },
	});
};

const resetPassword = async (payload: TResetPasswordPayload) => {
	const { email, otp, newPassword } = payload;

	const user = await prisma.user.findUnique({
		where: { email },
		select: { isActive: true, isDeleted: true, emailVerified: true },
	});

	// ভুল OTP-র ঠিক একই status + message, যাতে বোঝা না যায় account আছে কিনা
	if (!canResetPassword(user)) {
		throw new AppError(httpStatus.BAD_REQUEST, "Invalid verification code");
	}

	await auth.api.resetPasswordEmailOTP({
		body: { email, otp, password: newPassword },
	});
};

export const AuthService = {
	// ...আগেরগুলো
	forgetPassword,
	resetPassword,
};
```

৭. auth.controller.ts
```ts
const forgetPassword = catchAsync(async (req, res) => {
	await AuthService.forgetPassword(req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "If an account exists for this email, a reset code has been sent",
		data: null,
	});
});

const resetPassword = catchAsync(async (req, res) => {
	await AuthService.resetPassword(req.body);

	// session আগেই DB থেকে মুছে গেছে; এই browser-এর cookie-ও পরিষ্কার করি
	cookieUtils.clearAuthCookies(res);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Password reset successfully. Please log in with your new password",
		data: null,
	});
});

export const AuthController = {
	// ...আগেরগুলো
	forgetPassword,
	resetPassword,
};
```

৮. auth.route.ts
```ts
router.post(
	"/forget-password",
	authRateLimiter,
	validateRequest(AuthValidation.forgetPasswordSchema),
	AuthController.forgetPassword,
);
router.post(
	"/reset-password",
	authRateLimiter,
	validateRequest(AuthValidation.resetPasswordSchema),
	AuthController.resetPassword,
);
```

