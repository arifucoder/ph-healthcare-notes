.env-এ access token ছোট করো:
```dotenv
ACCESS_TOKEN_EXPIRES_IN=15m
```

1. auth.service.ts

```ts
const refreshTokens = async (refreshToken: string, sessionToken: string) => {
	// ধাপ ১: signature + মেয়াদ + payload-এর shape (ভুল হলে নিজেই 401 throw করে)
	const payload = tokenUtils.verifyRefreshToken(refreshToken);

	// ধাপ ২: DB-তে session
	const session = await prisma.session.findUnique({
		where: { token: sessionToken },
		select: {
			userId: true,
			expiresAt: true,
			user: { select: { role: true, isActive: true, isDeleted: true } },
		},
	});

	// ধাপ ৩: session আছে, মেয়াদ আছে, আর refresh token একই user-এর
	if (!session || session.expiresAt <= new Date() || session.userId !== payload.userId) {
		throw new AppError(httpStatus.UNAUTHORIZED, "Session expired. Please log in again");
	}

	// ধাপ ৪: account-এর এখনকার অবস্থা
	if (!session.user.isActive || session.user.isDeleted) {
		throw new AppError(httpStatus.FORBIDDEN, "Your account is disabled. Contact the clinic");
	}

	// ধাপ ৫: role DB থেকে, পুরনো token থেকে না
	const tokenPayload = { userId: session.userId, role: session.user.role };

	// ধাপ ৬: session-এর মেয়াদ বাড়ানো (token একই থাকে)
	await prisma.session.update({
		where: { token: sessionToken },
		data: { expiresAt: new Date(Date.now() + REFRESH_TOKEN_MAX_AGE) },
	});

	return {
		accessToken: tokenUtils.getAccessToken(tokenPayload),
		refreshToken: tokenUtils.getRefreshToken(tokenPayload),
		sessionToken,
	};
};
```

auth.controller.ts
```ts
const refreshTokens = catchAsync(async (req, res) => {
	const refreshToken = cookieUtils.getCookie(req, COOKIE_NAMES.refreshToken);
	const sessionToken = cookieUtils.getCookie(req, COOKIE_NAMES.session);

	if (!refreshToken || !sessionToken) {
		throw new AppError(httpStatus.UNAUTHORIZED, "You are not logged in");
	}

	const result = await AuthService.refreshTokens(refreshToken, sessionToken);

	cookieUtils.setAccessTokenCookie(res, result.accessToken);
	cookieUtils.setRefreshTokenCookie(res, result.refreshToken);
	cookieUtils.setSessionCookie(res, result.sessionToken);

	sendResponse(res, {
		statusCode: httpStatus.OK,
		message: "Session refreshed successfully",
		data: null,
	});
});
```

auth.route.ts
```ts
router.post("/refresh-token", AuthController.refreshTokens);
```