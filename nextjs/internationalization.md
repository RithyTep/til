# Internationalization (i18n) in Next.js

<div align="center">

![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![i18n](https://img.shields.io/badge/i18n-Internationalization-blue?style=for-the-badge)

*Build multi-language applications with App Router.*

</div>

## Directory Structure

```
app/
├── [lang]/
│   ├── layout.tsx
│   ├── page.tsx
│   └── about/
│       └── page.tsx
├── dictionaries/
│   ├── en.json
│   └── km.json
└── middleware.ts
```

## Dictionaries

```json
// dictionaries/en.json
{
  "home": {
    "title": "Welcome",
    "description": "This is my website"
  },
  "nav": {
    "home": "Home",
    "about": "About"
  }
}

// dictionaries/km.json
{
  "home": {
    "title": "សូមស្វាគមន៍",
    "description": "នេះជាគេហទំព័ររបស់ខ្ញុំ"
  },
  "nav": {
    "home": "ទំព័រដើម",
    "about": "អំពី"
  }
}
```

## Dictionary Loader

```tsx
// lib/dictionaries.ts
const dictionaries = {
  en: () => import('@/dictionaries/en.json').then(m => m.default),
  km: () => import('@/dictionaries/km.json').then(m => m.default),
};

export const getDictionary = async (locale: 'en' | 'km') => {
  return dictionaries[locale]();
};
```

## Middleware for Locale Detection

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const locales = ['en', 'km'];
const defaultLocale = 'en';

function getLocale(request: NextRequest) {
  // Check cookie first
  const cookieLocale = request.cookies.get('locale')?.value;
  if (cookieLocale && locales.includes(cookieLocale)) {
    return cookieLocale;
  }

  // Check Accept-Language header
  const acceptLanguage = request.headers.get('accept-language');
  // ... parse and match

  return defaultLocale;
}

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Check if pathname has locale
  const pathnameHasLocale = locales.some(
    locale => pathname.startsWith(`/${locale}/`) || pathname === `/${locale}`
  );

  if (pathnameHasLocale) return;

  // Redirect to locale path
  const locale = getLocale(request);
  request.nextUrl.pathname = `/${locale}${pathname}`;
  return NextResponse.redirect(request.nextUrl);
}

export const config = {
  matcher: ['/((?!api|_next|.*\\..*).*)'],
};
```

## Using Translations

```tsx
// app/[lang]/page.tsx
import { getDictionary } from '@/lib/dictionaries';

export default async function Home({
  params: { lang },
}: {
  params: { lang: 'en' | 'km' };
}) {
  const dict = await getDictionary(lang);

  return (
    <div>
      <h1>{dict.home.title}</h1>
      <p>{dict.home.description}</p>
    </div>
  );
}
```

## Language Switcher

```tsx
'use client';

import { usePathname, useRouter } from 'next/navigation';

export function LanguageSwitcher({ currentLang }) {
  const pathname = usePathname();
  const router = useRouter();

  const switchLocale = (newLocale: string) => {
    const newPath = pathname.replace(`/${currentLang}`, `/${newLocale}`);
    document.cookie = `locale=${newLocale}; path=/`;
    router.push(newPath);
  };

  return (
    <select value={currentLang} onChange={e => switchLocale(e.target.value)}>
      <option value="en">English</option>
      <option value="km">ភាសាខ្មែរ</option>
    </select>
  );
}
```

---

*Learned: December 20, 2025*
*Tags: Next.js, i18n, Internationalization, Localization*
