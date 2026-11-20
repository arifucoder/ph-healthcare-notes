packages 
```shell
pnpm add cloudinary multer file-type
pnpm add -D @types/multer
```
১. prisma/schema/clinic.prisma: DoctorProfile-এ একটা field
```prisma
  photoUrl             String?
  photoPublicId        String?
```

```shell
pnpm prisma migrate dev --name add_doctor_photo_public_id
```

২. src/app/config/env.ts

EMAIL_FROM-এর পরে, object-এর ভেতরে:
```
CLOUDINARY_CLOUD_NAME: z.string().trim().min(1).optional(),
CLOUDINARY_API_KEY: z.string().trim().min(1).optional(),
CLOUDINARY_API_SECRET: z.string().trim().min(1).optional(),

//superRefine-এর ভেতরে, শেষ if-এর পরে:
// ৩টাই দাও, নয়তো একটাও না। অর্ধেক দিলে ভুলটা startup-এই ধরা পড়বে
const cloudinaryKeys = ["CLOUDINARY_CLOUD_NAME", "CLOUDINARY_API_KEY", "CLOUDINARY_API_SECRET"] as const;
const setCount = cloudinaryKeys.filter((key) => env[key]).length;
if (setCount > 0 && setCount < cloudinaryKeys.length) {
    ctx.addIssue({
        code: "custom",
        path: ["CLOUDINARY_CLOUD_NAME"],
        message: "Set all three CLOUDINARY_* variables, or none of them",
    });
}
```

৩. নতুন file: src/app/utils/storage.ts
```ts
import { envVars } from "@/app/config/env";
import { AppError } from "@/app/errorHelpers/AppError";
import { v2 as cloudinary } from "cloudinary";
import httpStatus from "http-status";
import { randomUUID } from "node:crypto";

const { CLOUDINARY_CLOUD_NAME, CLOUDINARY_API_KEY, CLOUDINARY_API_SECRET } = envVars;

export const isStorageConfigured = Boolean(CLOUDINARY_CLOUD_NAME && CLOUDINARY_API_KEY && CLOUDINARY_API_SECRET);

if (isStorageConfigured) {
	cloudinary.config({
		cloud_name: CLOUDINARY_CLOUD_NAME,
		api_key: CLOUDINARY_API_KEY,
		api_secret: CLOUDINARY_API_SECRET,
		secure: true,
	});
}

const assertStorageConfigured = () => {
	if (!isStorageConfigured) {
		throw new AppError(httpStatus.SERVICE_UNAVAILABLE, "Photo upload is not configured");
	}
};

export type TUploadedImage = { url: string; publicId: string };

// Buffer → Cloudinary। নাম (public_id) server বানায়, client-এর filename কখনো না
export const uploadImage = (buffer: Buffer, folder: string): Promise<TUploadedImage> => {
	assertStorageConfigured();

	return new Promise((resolve, reject) => {
		const stream = cloudinary.uploader.upload_stream(
			{ folder, public_id: randomUUID(), resource_type: "image", overwrite: false },
			(error, result) => {
				if (error || !result) {
					console.error("Cloudinary upload failed", error);
					reject(new AppError(httpStatus.BAD_GATEWAY, "Photo upload failed, please try again"));
					return;
				}
				resolve({ url: result.secure_url, publicId: result.public_id });
			},
		);
		stream.end(buffer);
	});
};

export const deleteImage = async (publicId: string): Promise<void> => {
	assertStorageConfigured();
	await cloudinary.uploader.destroy(publicId, { resource_type: "image", invalidate: true });
};

// পুরনো ছবি মোছা fail করলে request fail হবে না, শুধু log হবে
export const deleteImageQuietly = async (publicId: string | null): Promise<void> => {
	if (!publicId) return;
	try {
		await deleteImage(publicId);
	} catch (error) {
		console.error(`Failed to delete image ${publicId}`, error);
	}
};
```

৪. নতুন file: src/app/middleware/uploadPhoto.ts
```ts
import multer from "multer";

export const PHOTO_MAX_BYTES = 2 * 1024 * 1024;

// Memory-তে রাখে (disk-এ না) → আগে যাচাই, তারপর Cloudinary
export const uploadPhoto = multer({
	storage: multer.memoryStorage(),
	limits: { fileSize: PHOTO_MAX_BYTES, files: 1 },
}).single("photo");
```

৫. নতুন file: src/app/shared/getValidatedPhoto.ts
```ts
import { AppError } from "@/app/errorHelpers/AppError";
import type { Request } from "express";
import { fileTypeFromBuffer } from "file-type";
import httpStatus from "http-status";

// SVG ইচ্ছা করে বাদ: SVG-র ভেতরে <script> থাকতে পারে
const ALLOWED_PHOTO_TYPES = new Set(["image/jpeg", "image/png", "image/webp"]);

// Client-এর mimetype বা extension-এ ভরসা না, file-এর প্রথম byte-গুলো (magic bytes) দেখে আসল type
export const getValidatedPhoto = async (req: Request): Promise<Buffer> => {
	if (!req.file) {
		throw new AppError(httpStatus.BAD_REQUEST, 'Photo is required (form field "photo")');
	}

	const type = await fileTypeFromBuffer(req.file.buffer);
	if (!type || !ALLOWED_PHOTO_TYPES.has(type.mime)) {
		throw new AppError(httpStatus.BAD_REQUEST, "Only JPEG, PNG or WebP images are allowed");
	}

	return req.file.buffer;
};
```

৬. নতুন file: src/app/errorHelpers/handleMulterError.ts
```ts
import httpStatus from "http-status";
import type { MulterError } from "multer";
import type { TGenericErrorResponse } from "./error.interface";

const messages: Partial<Record<MulterError["code"], string>> = {
	LIMIT_FILE_SIZE: "Photo must be 2 MB or smaller",
	LIMIT_FILE_COUNT: "Upload only one photo",
	LIMIT_UNEXPECTED_FILE: 'Send the photo in the "photo" form field',
};

export const handleMulterError = (err: MulterError): TGenericErrorResponse => ({
	statusCode: httpStatus.BAD_REQUEST,
	message: messages[err.code] ?? err.message,
	errorSources: [],
});
```

৭. src/app/middleware/globalErrorHandler.ts

Import যোগ করো:

```ts
import multer from "multer";
import { handleMulterError } from "../errorHelpers/handleMulterError";

//if chain-এ AppError-এর আগে একটা branch বসাও:


} else if (err instanceof multer.MulterError) {
    errorResponse = handleMulterError(err);
} else if (err instanceof AppError) {
```

৮. src/app/middleware/rateLimiter.ts

নিচে যোগ করো:

```ts
export const uploadRateLimiter = rateLimit({
	windowMs: 15 * 60 * 1000,
	limit: 20,
	standardHeaders: "draft-7",
	legacyHeaders: false,
});
```

৯. src/app/module/doctor/doctor.service.ts

Import যোগ করো:
```ts
import { deleteImageQuietly, uploadImage } from "@/app/utils/storage";

// ---------- Photo ----------

const DOCTOR_PHOTO_FOLDER = "clinicos/doctors";

const findMyDoctorIdOrThrow = async (userId: string) => {
    const doctor = await prisma.doctorProfile.findFirst({
        where: { userId, ...notDeleted },
        select: { id: true },
    });

    if (!doctor) {
        throw new AppError(httpStatus.NOT_FOUND, "Doctor profile not found");
    }

    return doctor.id;
};

const findPhotoPublicIdOrThrow = async (id: string) => {
    const doctor = await prisma.doctorProfile.findFirst({
        where: { id, ...notDeleted },
        select: { photoPublicId: true },
    });

    if (!doctor) {
        throw new AppError(httpStatus.NOT_FOUND, "Doctor not found");
    }

    return doctor.photoPublicId;
};

// ক্রম: নতুন ছবি upload → DB update → পুরনো ছবি মোছা
const setDoctorPhoto = async (id: string, photo: Buffer) => {
    const oldPublicId = await findPhotoPublicIdOrThrow(id);

    const uploaded = await uploadImage(photo, DOCTOR_PHOTO_FOLDER);

    // DB update fail করলে নতুন ছবিটা অনাথ হয়ে থাকবে, তাই সেটা মুছে error ফেরত
    const updated = await prisma.doctorProfile
        .update({
            where: { id },
            data: { photoUrl: uploaded.url, photoPublicId: uploaded.publicId },
            select: doctorSelect,
        })
        .catch(async (error: unknown) => {
            await deleteImageQuietly(uploaded.publicId);
            throw error;
        });

    await deleteImageQuietly(oldPublicId);

    return toDoctorResponse(updated);
};

const removeDoctorPhoto = async (id: string) => {
    const oldPublicId = await findPhotoPublicIdOrThrow(id);

    const updated = await prisma.doctorProfile.update({
        where: { id },
        data: { photoUrl: null, photoPublicId: null },
        select: doctorSelect,
    });

    await deleteImageQuietly(oldPublicId);

    return toDoctorResponse(updated);
};

// Doctor নিজে: id session থেকে, client থেকে না (IDOR-safe)
const setMyPhoto = async (userId: string, photo: Buffer) => setDoctorPhoto(await findMyDoctorIdOrThrow(userId), photo);

const removeMyPhoto = async (userId: string) => removeDoctorPhoto(await findMyDoctorIdOrThrow(userId));
```

১০. doctor.controller.ts আর doctor.route.ts

Controller: import যোগ করো, তারপর ৪টা handler:

```ts
import { getValidatedPhoto } from "@/app/shared/getValidatedPhoto";

const uploadMyPhoto = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const photo = await getValidatedPhoto(req);
	const result = await DoctorService.setMyPhoto(userId, photo);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Photo updated successfully", data: result });
});

const removeMyPhoto = catchAsync(async (req, res) => {
	const { userId } = getAuthUser(req);
	const result = await DoctorService.removeMyPhoto(userId);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Photo removed successfully", data: result });
});

const uploadDoctorPhoto = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const photo = await getValidatedPhoto(req);
	const result = await DoctorService.setDoctorPhoto(id, photo);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Photo updated successfully", data: result });
});

const removeDoctorPhoto = catchAsync(async (req, res) => {
	const { id } = idParamSchema.parse(req.params);
	const result = await DoctorService.removeDoctorPhoto(id);

	sendResponse(res, { statusCode: httpStatus.OK, message: "Photo removed successfully", data: result });
});
```

Route (doctor.route.ts):
```ts
import { uploadRateLimiter } from "@/app/middleware/rateLimiter";
import { uploadPhoto } from "@/app/middleware/uploadPhoto";

// PATCH /me-এর নিচে, router.use(checkAuth(...ADMINS))-এর আগে
router.put("/me/photo", checkAuth(Role.DOCTOR), uploadRateLimiter, uploadPhoto, DoctorController.uploadMyPhoto);
router.delete("/me/photo", checkAuth(Role.DOCTOR), DoctorController.removeMyPhoto);

router.use(checkAuth(...ADMINS));

// ... আগের route-গুলো ...

router.put("/:id/photo", uploadRateLimiter, uploadPhoto, DoctorController.uploadDoctorPhoto);
router.delete("/:id/photo", DoctorController.removeDoctorPhoto);
```