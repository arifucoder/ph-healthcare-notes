# 07 - Complete Patient Registration Process Using Transaction

> Module: 38-01 (২য় অংশ)

**লক্ষ্য:** Patient register করলে দুই জায়গায় data যাবে:
1. `user` table → login এর জন্য (Better Auth)
2. `patient` table → patient এর নিজস্ব info (photo, contact, address ...)

```
POST /api/v1/auth/register
      ↓
auth.api.signUpEmail()  →  user table
      ↓
prisma.patient.create() →  patient table (userId দিয়ে user এর সাথে যুক্ত)
```

---

## ধাপ ১: Patient Model — `prisma/schema/patient.prisma`

```prisma
model Patient {
  id String @id @default(uuid(7))

  name          String
  email         String    @unique
  profilePhoto  String?
  contactNumber String?
  address       String?
  isDeleted     Boolean   @default(false)
  deletedAt     DateTime?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  //relations

  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade, onUpdate: Cascade)

  @@index([email], name: "idx_patient_email")
  @@index([isDeleted], name: "idx_patient_isDeleted")
  @@map("patient")
}
```

**Field গুলো বুঝি:**
- `uuid(7)` → সময় অনুযায়ী sort হওয়া UUID (নতুন গুলো পরে আসে), index এর জন্য ভালো
- `userId String @unique` → একজন user এর **একটাই** patient profile (**One-to-One** relation)
- `@relation(fields: [userId], references: [id])` → `Patient.userId` যুক্ত থাকবে `User.id` এর সাথে

**`onDelete` / `onUpdate: Cascade` মানে:**
| | কী হয় |
|---|---|
| `onDelete: Cascade` | **User delete** হলে তার **patient data ও auto delete** হবে |
| `onUpdate: Cascade` | User এর `id` বদলালে patient এর `userId` ও auto update হবে |

---

## ধাপ ২: User model এ Patient relation যোগ — `auth.prisma`

Prisma তে relation **দুই দিকেই** লিখতে হয়। তাই `User` model এ patient field যোগ করো:

```prisma
model User {
  // ... আগের সব field
  sessions           Session[]
  accounts           Account[]
  patient            Patient?   // ✅ নতুন — একজন user এর patient profile থাকতেও পারে, নাও পারে

  @@unique([email])
  @@map("user")
}
```

> ✏️ এই ধাপটা নোটে ছিল না, কিন্তু এটা না দিলে Prisma error দেবে:
> `The relation field 'user' on model 'Patient' is missing an opposite relation field on the model 'User'`
> (`Patient?` → optional, কারণ Doctor / Admin user এর patient profile থাকবে না)

---

## ধাপ ৩: Migrate + Generate

```shell
pnpm migrate && pnpm generate
```

---

## ধাপ ৪: auth.service.ts — `registerPatient` update

উপরে `prisma` import যোগ করো:

```ts
import { auth } from "@/app/lib/auth";
import { prisma } from "@/app/lib/prisma";
import { UserStatus } from "@/generated/prisma/enums";
```

তারপর `registerPatient`:

```ts
const registerPatient = async (payload: IRegisterPatientPayload) => {
	const { name, email, password } = payload;

	const data = await auth.api.signUpEmail({
		body: {
			name,
			email,
			password,
			//default values
			// needPasswordChange: false,
			// role: Role.PATIENT
		},
	});

	if (!data.user) {
		throw new Error("Failed to register patient");
	}

	try {
		const patient = await prisma.$transaction(async (tx) => {
			const patientTx = await tx.patient.create({
				data: {
					userId: data.user.id,
					name,
					email,
				},
			});

			return patientTx;
		});

		return {
			...data,
			patient,
		};
	} catch (error) {
		console.error("Failed to create patient:", error);

		// Rollback: patient না হলে user টাও মুছে দাও
		await prisma.user.delete({
			where: { id: data.user.id },
		});

		throw error;
	}
};
```

**কী হচ্ছে:**
1. `signUpEmail` → `user` table এ user তৈরি
2. User তৈরি না হলে → error
3. `try` এর ভেতরে `prisma.$transaction` → `patient` table এ data insert, `userId: data.user.id` দিয়ে user এর সাথে যুক্ত
4. সফল হলে → `{ ...data, patient }` return (user + token + patient)
5. **Patient create fail হলে** → `catch` এ গিয়ে **user টা delete** (manual rollback), তারপর error আবার `throw` করে যাতে `catchAsync` error response পাঠায়

**কেন user delete করতে হয়?**
`signUpEmail` transaction এর **বাইরে** চলে (Better Auth নিজে DB তে লেখে), তাই transaction fail হলে user নিজে থেকে rollback হয় না। Delete না করলে user থেকে যাবে কিন্তু patient থাকবে না — আর সে আবার register করতে গেলে "email already exists" পেয়ে আটকে যাবে।

> `onDelete: Cascade` থাকায় user delete করলে তার `session`, `account` ও মুছে যায়।

**Transaction কী?**
Transaction এর ভেতরের সব query **হয় একসাথে সফল হবে, নাহয় একটাও হবে না** (কোনোটা fail হলে আগেরগুলো rollback হয়)। `tx` দিয়ে query লিখতে হয় (`prisma` না), তাহলেই সেটা transaction এর অংশ হয়। এখন ভেতরে একটা query, পরে আরও query (যেমন medical history) যোগ হলে transaction সত্যিই কাজে লাগবে।

---

## ধাপ ৫: Postman দিয়ে Test

```
POST http://localhost:5000/api/v1/auth/register
```

```json
{
	"name": "Rahim",
	"email": "rahim@example.com",
	"password": "12345678"
}
```

✅ Check করো: `pnpm studio` → `user` আর `patient` দুই table এই data এসেছে, আর `patient.userId` = `user.id`

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🔴 গুরুত্বপূর্ণ

**১. ✅ (Done) Patient create fail হলে user rollback**
`catch` এ `prisma.user.delete` যোগ করা হয়েছে (ধাপ ৪ দেখো)।

### 🟡 ভালো practice

**২. `@@index([email])` অপ্রয়োজনীয়**
`email` এ `@unique` আছে — unique নিজেই index বানায়। আলাদা index দিলে DB তে একই কাজের দুইটা index থাকে।

**৩. ✅ (Done) Destructure করা variable use করো**
শুরুতেই `const { name, email } = payload` করা হয়েছে, কিন্তু নিচে আবার `payload.name`, `payload.email`। একই রকম রাখতে `name`, `email` লিখলেই হয়।

**৪. Data duplication খেয়াল রাখো**
`name` আর `email` দুই table এই আছে (`user` + `patient`)। পরে user তার email/name বদলালে **দুই জায়গাতেই update** করতে হবে, না হলে data মিলবে না।

**৫. ✅ (Done) `needsPasswordChange` বানান**
Comment এ `needsPasswordChange`, কিন্তু field এর নাম `needPasswordChange` (05 নোটেও বলা আছে)।

### 🔵 পরে কাজে লাগবে

**৬. Response থেকে অপ্রয়োজনীয় data বাদ দাও**
`{ ...data, patient }` এ user, token, patient সব যাচ্ছে — একই name/email দুইবার। Frontend এর যা দরকার শুধু সেটুকু পাঠালে response ছোট আর পরিষ্কার হবে।

**৭. Duplicate email এর সুন্দর error**
একই email দিয়ে আবার register করলে এখন `500` আসবে। Global error handler বানানোর সময় এটাকে `409 Conflict` + "Email already exists" করে দিও।
