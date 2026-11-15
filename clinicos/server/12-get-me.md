auth.service.ts
```ts
const getMe = async (userId: string) => {
	return prisma.user.findUniqueOrThrow({
		where: { id: userId },
		select: {
			id: true,
			name: true,
			email: true,
			emailVerified: true,
			image: true,
			role: true,
			needPasswordChange: true,
			patientProfile: {
				select: {
					id: true,
					fullName: true,
					phone: true,
					dateOfBirth: true,
					gender: true,
					address: true,
					bloodGroup: true,
					allergies: true,
					emergencyContactName: true,
					emergencyContactPhone: true,
				},
			},
		},
	});
};
```

auth.controller.ts
```ts
const getMe = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const result = await AuthService.getMe(userId);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Profile retrieved successfully",
		data: result,
	});
});
```

auth.route.ts
```ts
router.get("/me", checkAuth(), AuthController.getMe);
```