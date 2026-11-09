# 27 - Footer (Server Component + `"use cache"` দিয়ে সাল)

> Public page-গুলোর নিচে footer: বামে copyright (চলতি সাল সহ), ডানে menu link — header-এর একই `publicNavItems` config থেকে (note 26)।
> পুরোটা server component, browser-এ কোনো JS যায় না।

```
CommonLayout:  <Navbar />  →  <main className="flex-1">  →  <Footer />
                                                             ├─ © <CurrentYear />  ("use cache", দিনে একবার)
                                                             └─ publicNavItems → Link
```

## ফাইল

```
src/components/shared/Footer/Footer.tsx
src/app/(commonLayout)/layout.tsx
```

---

## ধাপ ১: `src/components/shared/Footer/Footer.tsx`

```tsx
import { publicNavItems } from "@/lib/publicNavItems";
import { cacheLife } from "next/cache";
import Link from "next/link";

// cacheComponents চালু থাকায় সরাসরি new Date() লিখলে build error দেয়।
// "use cache" দিয়ে সাল-টা cache করা হলো, দিনে একবার নতুন করে হিসাব হবে।
async function CurrentYear() {
	"use cache";
	cacheLife("days");
	return new Date().getFullYear();
}

export default function Footer() {
	return (
		<footer className="border-t bg-muted/30">
			<div className="container mx-auto flex flex-col items-center gap-4 px-4 py-6 md:flex-row md:justify-between">
				{/* বাম পাশে copyright */}
				<p className="text-sm text-muted-foreground">
					© <CurrentYear /> <span className="font-semibold text-primary">Phdoc</span>. All rights reserved.
				</p>

				{/* ডান পাশে menu, header-এর সেই একই config থেকে */}
				<nav aria-label="Footer navigation">
					<ul className="flex flex-wrap justify-center gap-x-4 gap-y-2">
						{publicNavItems.map((item) => (
							<li key={item.href}>
								<Link
									href={item.href}
									className="text-sm text-muted-foreground transition-colors hover:text-foreground"
								>
									{item.title}
								</Link>
							</li>
						))}
					</ul>
				</nav>
			</div>
		</footer>
	);
}
```

### ভেঙে বোঝা

**১. `"use client"` নেই কেন**
- footer-এ active link দেখানোর দরকার নেই → `usePathname` লাগে না।
- কোনো state বা click handler নেই → পুরোটা server component, browser-এ কোনো JS যায় না।
- header-এর `NavLinks` (note 26) client ছিল শুধু active link-এর জন্য; footer-এ সাধারণ `Link`-ই যথেষ্ট।

**২. `CurrentYear` আর `"use cache"`**

| লেখা | `cacheComponents: true`-তে কী হয় |
|---|---|
| সরাসরি `new Date().getFullYear()` | build error — Next জানে না এটা build-এর সময়ের মান নাকি প্রতি request-এর |
| `"use cache"` + `cacheLife("days")` | মানটা cache হয়, static অংশের ভেতরেই থাকে; দিনে একবার নতুন করে হিসাব |
| (বিকল্প) `await connection()` + `Suspense` | প্রতি request-এ চলে — শুধু সালের জন্য বাড়তি খরচ |

- `"use cache"` function-এর একদম প্রথম লাইনে থাকতে হয়, আর function টা `async` হতে হয়।
- `cacheLife("days")` → Next.js-এর built-in profile: মোটামুটি দিনে একবার নতুন মান।
- async server component হওয়ায় `<CurrentYear />` সরাসরি JSX-এ লেখা যায় (non-async `Footer`-এর ভেতরেও)।

**৩. Layout (responsive)**
- mobile: `flex-col items-center` → copyright আর menu উপর-নিচে, মাঝখানে।
- `md` থেকে: `md:flex-row md:justify-between` → বামে copyright, ডানে menu।
- menu `flex-wrap` + `gap-x-4 gap-y-2` → জায়গা কম হলে পরের লাইনে যায়, ফাঁক ঠিক থাকে।

**৪. Menu একই config থেকে**
- `publicNavItems` (note 26) → নতুন page যোগ করলে header, mobile menu, footer তিন জায়গাতেই চলে আসে।
- `<nav aria-label="Footer navigation">` → page-এ তিনটা nav (Main, Mobile, Footer), আলাদা নাম দিয়ে screen reader চেনে।

---

## ধাপ ২: `src/app/(commonLayout)/layout.tsx`

```tsx
import Footer from "@/components/shared/Footer/Footer";
import Navbar from "@/components/shared/Navbar/Navbar";

export default function CommonLayout({ children }: LayoutProps<"/">) {
	return (
		<>
			<Navbar />
			<main className="flex-1">{children}</main>
			<Footer />
		</>
	);
}
```

### Footer সবসময় নিচে থাকে কীভাবে (sticky footer)

| জায়গা | class | কাজ |
|---|---|---|
| root `<html>` | `h-full` | পুরো screen-এর উচ্চতা |
| root `<body>` | `min-h-full flex flex-col` | অন্তত screen-এর সমান উঁচু, ভেতরের জিনিস উপর-নিচে সাজানো |
| `<main>` | `flex-1` | বাকি সব খালি জায়গা নিয়ে নেয় |

- ফলে page-এ content কম থাকলেও (যেমন "No doctors found") footer screen-এর একদম নিচে থাকে; content বেশি হলে স্বাভাবিকভাবে শেষে।
- `QueryProvider` শুধু context দেয়, কোনো HTML element বানায় না → `Navbar`, `main`, `Footer` সরাসরি body-র flex-এর ভেতরে থাকে, তাই কাজ করে।

---

## 🧪 Test (browser-এ)

| কী করবে | ফল |
|---|---|
| যেকোনো public page | নিচে "© 2026 Phdoc. All rights reserved." + menu |
| কম content-এর page (যেমন `/ngos`) | footer screen-এর একদম নিচে, মাঝে ঝুলে থাকে না |
| mobile width | copyright আর menu উপর-নিচে, মাঝখানে |
| desktop | বামে copyright, ডানে menu |
| DevTools → Sources/Network | Footer-এর জন্য আলাদা JS chunk নেই |
| `pnpm build` | `new Date()`-এর জন্য কোনো error নেই |

---

## 🤖 Claude Recommendation

> এগুলো শুধু suggestion — উপরের code বদলানো হয়নি।

### 🟡 ভালো practice

1. **Header, footer আর page-এর চওড়া এক রাখা** — footer-এও header-এর মতো `container` (বড় screen-এ max 1536px), কিন্তু page-গুলো `max-w-7xl` (1280px)। note 26-এর suggestion নিলে এখানেও একই দাও:
   ```tsx
   <div className="mx-auto flex w-full max-w-7xl flex-col items-center gap-4 px-4 py-6 md:flex-row md:justify-between">
   ```

### 🔵 পরে কাজে লাগবে / জেনে রাখো

2. **নতুন বছরের প্রথম কয়েক ঘণ্টা** — server সাধারণত UTC-তে চলে, আর cache দিনে একবার নতুন হয়। তাই ১ জানুয়ারি ঢাকায় কয়েক ঘণ্টা (বড়জোর এক দিন) পুরনো সাল দেখাতে পারে। copyright-এর জন্য এটা কোনো সমস্যা না।

3. **Footer বড় করতে চাইলে** — পরে contact (phone, email), social link, "About/Privacy/Terms" এর মতো আলাদা column যোগ করা যায়। তখনো server component রাখো; শুধু newsletter form-এর মতো interactive অংশ আলাদা client component হবে।

---

## 🔜 পরে যা যোগ হবে

- [ ] contact info আর social link
- [ ] Privacy Policy / Terms page
