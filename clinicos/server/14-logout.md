auth.service.ts
```ts
const logout = async (sessionToken: string | undefined) => {
	if (!sessionToken) return;
    
	await prisma.session.deleteMany({ where: { token: sessionToken } });
};
```

auth.controller.ts
```ts
const logout = catchAsync(async (req, res) => {
	const sessionToken = cookieUtils.getCookie(req, COOKIE_NAMES.session);

	await AuthService.logout(sessionToken);


	cookieUtils.clearAuthCookies(res);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Logged out successfully",
		data: null,
	});
});
```

auth.route.ts
```ts
router.post("/logout", AuthController.logout);
```