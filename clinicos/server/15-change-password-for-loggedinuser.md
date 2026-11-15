auth.validation.ts
```ts
const passwordSchema = z.string().min(8, "Password must be at least 8 characters").max(128);

const changePasswordSchema = z
	.object({
		currentPassword: z.string().min(1, "Current password is required").max(128),
		newPassword: passwordSchema,
	})
	.refine((data) => data.currentPassword !== data.newPassword, {
		path: ["newPassword"],
		message: "New password must be different from the current password",
	});
```

auth.interface.ts
```ts
export type TChangePasswordPayload = z.infer<typeof AuthValidation.changePasswordSchema>;
```

auth.service.ts
```ts
const changePassword = async (
	userId: string,
	sessionToken: string | undefined,
	payload: TChangePasswordPayload,
) => {
	const { currentPassword, newPassword } = payload;

	const account = await prisma.account.findFirst({
		where: { userId, providerId: "credential" },
		select: { id: true, password: true },
	});

	if (!account?.password) {
		throw new AppError(httpStatus.BAD_REQUEST, "Password login is not set up for this account");
	}

	const isMatch = await verifyPassword({ hash: account.password, password: currentPassword });

	// 401 না, 400: 401 দিলে frontend ভাববে session শেষ, আর refresh/logout শুরু করবে
	if (!isMatch) {
		throw new AppError(httpStatus.BAD_REQUEST, "Current password is incorrect");
	}

	const hashedPassword = await hashPassword(newPassword);

	// তিনটা একসাথে: password বদলাল কিন্তু পুরনো session রয়ে গেল, এমন অর্ধেক অবস্থা যেন না হয়
	await prisma.$transaction([
		prisma.account.update({
			where: { id: account.id },
			data: { password: hashedPassword },
		}),
		prisma.user.update({
			where: { id: userId },
			data: { needPasswordChange: false },
		}),
		// এই device-এর session রেখে বাকি সব বাতিল
		prisma.session.deleteMany({
			where: { userId, ...(sessionToken && { token: { not: sessionToken } }) },
		}),
	]);
};
```

auth.controller.ts
```ts
const changePassword = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const sessionToken = cookieUtils.getCookie(req, COOKIE_NAMES.session);

	await AuthService.changePassword(userId, sessionToken, req.body);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Password changed successfully",
		data: null,
	});
});
```

auth.route.ts
```ts
router.post(
	"/change-password",
	checkAuth(),
	authRateLimiter,
	validateRequest(AuthValidation.changePasswordSchema),
	AuthController.changePassword,
);
```