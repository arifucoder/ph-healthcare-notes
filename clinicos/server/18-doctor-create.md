1. Schema: prisma/schema/clinic.prisma
```prisma
model DoctorProfile {
  id                   String  @id @default(uuid())
  userId               String  @unique
  licenseNumber        String  @unique
  consultationFeeCents Int
  roomNumber           String?
  isPaused             Boolean @default(false)
  slotMinutesDefault   Int     @default(15)
  bio                  String?
  photoUrl             String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  user              User              @relation(fields: [userId], references: [id])
  doctorSpecialties DoctorSpecialty[]

  @@map("doctor_profiles")
}

model DoctorSpecialty {
  doctorProfileId String
  specialtyId     String
  createdAt       DateTime @default(now())

  doctorProfile DoctorProfile @relation(fields: [doctorProfileId], references: [id], onDelete: Cascade)
  specialty     Specialty     @relation(fields: [specialtyId], references: [id])

  @@id([doctorProfileId, specialtyId])
  @@index([specialtyId])
  @@map("doctor_specialties")
}
```

user.validation.ts
```ts
const createDoctorSchema = z.object({
	name: personNameSchema,
	email: emailSchema,
	phone: phoneSchema.optional(),
	specialtyIds: z
		.array(z.uuid("Invalid specialty id"))
		.min(1, "Select at least one specialty")
		.max(5, "A doctor can have at most 5 specialties")
		.transform((ids) => [...new Set(ids)]),
	licenseNumber: z.string().trim().toUpperCase().min(3, "License number is too short").max(50),
	consultationFeeCents: z.number().int("Fee must be in whole cents").min(0).max(10_000_000),
	roomNumber: z.string().trim().max(20).optional(),
	bio: z.string().trim().max(1000).optional(),
});
```

user.interface.ts
```ts
export type TCreateDoctorPayload = z.infer<typeof UserValidation.createDoctorSchema>;
```

user.service.ts
```ts
const doctorProfileSelect = {
	id: true,
	licenseNumber: true,
	consultationFeeCents: true,
	roomNumber: true,
	bio: true,
	photoUrl: true,
	doctorSpecialties: {
		select: { specialty: { select: { id: true, name: true, slug: true } } },
	},
} as const;

const createDoctor = async (payload: TCreateDoctorPayload) => {
    const { name, email, phone, specialtyIds, licenseNumber, consultationFeeCents, roomNumber, bio } = payload;

    const existingUser = await prisma.user.findUnique({
        where: { email },
        select: { id: true },
    });

    if (existingUser) {
        throw new AppError(httpStatus.CONFLICT, "An account with this email already exists");
    }

    const validSpecialtyCount = await prisma.specialty.count({
        where: { id: { in: specialtyIds }, isDeleted: false, isActive: true },
    });

    if (validSpecialtyCount !== specialtyIds.length) {
        throw new AppError(httpStatus.BAD_REQUEST, "One or more specialties are not found or inactive");
    }

    const licenseTaken = await prisma.doctorProfile.findUnique({
        where: { licenseNumber },
        select: { id: true },
    });

    if (licenseTaken) {
        throw new AppError(httpStatus.CONFLICT, "A doctor with this license number already exists");
    }

    const temporaryPassword = randomBytes(9).toString("base64url");
    const hashedPassword = await hashPassword(temporaryPassword);
    const userId = crypto.randomUUID();

    const { user, doctorProfile } = await prisma.$transaction(async (tx) => {
        const createdUser = await tx.user.create({
            data: {
                id: userId,
                name,
                email,
                phone: phone ?? null,
                role: Role.DOCTOR,
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

        const createdProfile = await tx.doctorProfile.create({
            data: {
                userId,
                licenseNumber,
                consultationFeeCents,
                roomNumber: roomNumber ?? null,
                bio: bio ?? null,
                doctorSpecialties: {
                    createMany: {
                        data: specialtyIds.map((specialtyId) => ({ specialtyId })),
                    },
                },
            },
            select: doctorProfileSelect,
        });

        return { user: createdUser, doctorProfile: createdProfile };
    });

    let emailSent = true;

    try {
        await sendEmail({
            to: email,
            subject: `Your ${envVars.CLINIC_NAME} doctor account`,
            previewText: "Your account is ready. Sign in with the temporary password inside.",
            template: "staffWelcome",
            data: {
                name,
                roleLabel: "Doctor",
                email,
                temporaryPassword,
                loginUrl: `${envVars.FRONTEND_URL}/login`,
            },
        });
    } catch (error) {
        console.error(`Failed to send welcome email to user ${user.id}`, error);
        emailSent = false;
    }

    const { doctorSpecialties, ...profile } = doctorProfile;

    return {
        user,
        doctorProfile: {
            ...profile,
            specialties: doctorSpecialties.map(({ specialty }) => specialty),
        },
        emailSent,
    };
};
```

user.controller.ts
```ts
const createDoctor = catchAsync(async (req, res) => {
	const result = await UserService.createDoctor(req.body);

	sendResponse(res, {
		statusCode: httpStatus.CREATED,
		message: result.emailSent
			? "Doctor created. Login details have been sent to their email"
			: "Doctor created, but the welcome email could not be sent. Reset their password to send it again",
		data: result,
	});
});
```

user.route.ts
```ts
router.post(
	"/doctors",
	checkAuth(...ADMINS),
	validateRequest(UserValidation.createDoctorSchema),
	UserController.createDoctor,
);
```