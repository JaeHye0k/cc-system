# Next.js App Router i18n 다국어 처리 설정

## 개요

Next.js 15 App Router 환경에서 i18next 기반 다국어 처리를 구현한다.

- 지원 언어: 한국어(ko, 기본), 영어(en)
- URL 기반 라우팅: `/ko/...`, `/en/...`
- 서버 컴포넌트와 클라이언트 컴포넌트 각각에 최적화된 번역 함수 제공

## 필요 패키지

```bash
pnpm add i18next react-i18next i18next-resources-to-backend cookies-next i18next-browser-languagedetector accept-language
```

## 디렉토리 구조

```
app/
├── layout.tsx                    # 최소 루트 레이아웃
└── [lng]/
    ├── layout.tsx                # 언어별 레이아웃 (html lang 설정)
    ├── page.tsx
    └── ...

src/
├── i18n/
│   ├── settings.ts               # 언어 설정 및 옵션
│   ├── server.ts                 # 서버 컴포넌트용 translate 함수
│   ├── client.ts                 # 클라이언트 컴포넌트용 useTranslation hook
│   └── locales/
│       ├── ko/
│       │   └── *.json            # 한국어 번역 파일들
│       └── en/
│           └── *.json            # 영어 번역 파일들
├── types/
│   └── i18n.ts                   # Language 타입 정의
└── constants/
    └── i18nKeys.ts               # 번역 키 상수 관리

middleware.ts                      # 언어 감지 및 리다이렉션
```

## 파일별 구현

### 1. `src/i18n/settings.ts`

언어 설정과 i18next 옵션을 관리하는 핵심 설정 파일.

```typescript
export const fallbackLng = 'ko';
export const languages = [fallbackLng, 'en'] as const;
export const defaultNS = 'translation';
export const cookieName = 'i18next';

export function getOptions(
  lng = fallbackLng,
  ns: string | string[] = defaultNS
) {
  return {
    supportedLngs: languages,
    fallbackLng,
    lng,
    fallbackNS: defaultNS,
    defaultNS,
    ns: Array.isArray(ns) ? ns : [ns],
  };
}
```

### 2. `src/i18n/server.ts` (서버 컴포넌트용)

서버 컴포넌트에서 사용할 translate 함수. 매 요청마다 새 인스턴스 생성.

```typescript
import { createInstance, Namespace, FlatNamespace, KeyPrefix } from 'i18next';
import resourcesToBackend from 'i18next-resources-to-backend';
import { initReactI18next } from 'react-i18next/initReactI18next';
import { getOptions } from './settings';

const initI18next = async (lng: string, ns: string | string[]) => {
  const i18nInstance = createInstance();
  await i18nInstance
    .use(initReactI18next)
    .use(
      resourcesToBackend(
        (language: string, namespace: string) =>
          import(`./locales/${language}/${namespace}.json`)
      )
    )
    .init(getOptions(lng, ns));
  return i18nInstance;
};

export async function translate<
  Ns extends FlatNamespace,
  KPrefix extends KeyPrefix<FlatNamespace> = undefined
>(lng: string, ns?: Ns, options: { keyPrefix?: KPrefix } = {}) {
  const i18nextInstance = await initI18next(lng, ns || 'translation');
  return {
    t: i18nextInstance.getFixedT(
      lng,
      Array.isArray(ns) ? ns[0] : ns,
      options.keyPrefix
    ),
    i18n: i18nextInstance,
  };
}
```

### 3. `src/i18n/client.ts` (클라이언트 컴포넌트용)

클라이언트 컴포넌트용 useTranslation hook. 싱글톤 인스턴스 사용.

```typescript
'use client';

import { useEffect, useState } from 'react';
import i18next, { FlatNamespace, KeyPrefix } from 'i18next';
import {
  initReactI18next,
  useTranslation as useTranslationOrg,
  UseTranslationOptions,
  UseTranslationResponse,
  FallbackNs,
} from 'react-i18next';
import { getCookie, setCookie } from 'cookies-next';
import resourcesToBackend from 'i18next-resources-to-backend';
import LanguageDetector from 'i18next-browser-languagedetector';
import { getOptions, languages, cookieName } from './settings';

const runsOnServerSide = typeof window === 'undefined';

// Initialize i18next for client side
i18next
  .use(initReactI18next)
  .use(LanguageDetector)
  .use(
    resourcesToBackend(
      (language: string, namespace: string) =>
        import(`./locales/${language}/${namespace}.json`)
    )
  )
  .init({
    ...getOptions(),
    lng: undefined, // let detect the language on client side
    detection: {
      order: ['path', 'htmlTag', 'cookie', 'navigator'],
    },
    preload: runsOnServerSide ? languages : [],
  });

export function useTranslation<
  Ns extends FlatNamespace,
  KPrefix extends KeyPrefix<FlatNamespace> = undefined
>(
  lng: string,
  ns?: Ns,
  options?: UseTranslationOptions<KPrefix>
): UseTranslationResponse<FallbackNs<Ns>, KPrefix> {
  const i18nextCookie = getCookie(cookieName);
  const ret = useTranslationOrg(ns, options);
  const { i18n } = ret;

  if (runsOnServerSide && lng && i18n.resolvedLanguage !== lng) {
    i18n.changeLanguage(lng);
  } else {
    // eslint-disable-next-line react-hooks/rules-of-hooks
    const [activeLng, setActiveLng] = useState(i18n.resolvedLanguage);

    // eslint-disable-next-line react-hooks/rules-of-hooks
    useEffect(() => {
      if (activeLng === i18n.resolvedLanguage) return;
      setActiveLng(i18n.resolvedLanguage);
    }, [activeLng, i18n.resolvedLanguage]);

    // eslint-disable-next-line react-hooks/rules-of-hooks
    useEffect(() => {
      if (!lng || i18n.resolvedLanguage === lng) return;
      i18n.changeLanguage(lng);
    }, [lng, i18n]);

    // eslint-disable-next-line react-hooks/rules-of-hooks
    useEffect(() => {
      if (i18nextCookie === lng) return;
      setCookie(cookieName, lng, { path: '/' });
    }, [lng, i18nextCookie]);
  }

  return ret;
}
```

### 4. `src/types/i18n.ts`

Language 타입 정의.

```typescript
import { languages } from '../i18n/settings';

export type Language = (typeof languages)[number];
```

### 5. `src/constants/i18nKeys.ts`

번역 키를 상수로 관리하여 타입 안정성과 유지보수성을 확보한다.

**장점:**

- 오타로 인한 런타임 에러 방지
- IDE 자동완성 지원
- 키 변경 시 일괄 수정 용이
- 사용되지 않는 키 추적 가능

```typescript
/**
 * i18n 번역 키 상수
 * 타입 안정성과 유지보수성을 위해 모든 번역 키를 상수로 관리
 */

// Header 네임스페이스
export const HEADER_KEYS = {
  PROJECTS: 'projects',
  CONTACT: 'contact',
} as const;

// Footer 네임스페이스
export const FOOTER_KEYS = {
  FIELDS: {
    COMPANY: 'fields.company',
    REPRESENTATIVE: 'fields.representative',
    BUSINESS_NUMBER: 'fields.businessNumber',
    EMAIL: 'fields.email',
    PHONE: 'fields.phone',
    ADDRESS: 'fields.address',
  },
  VALUES: {
    COMPANY: 'values.company',
    REPRESENTATIVE: 'values.representative',
    BUSINESS_NUMBER: 'values.businessNumber',
    EMAIL: 'values.email',
    PHONE: 'values.phone',
    ADDRESS: 'values.address',
  },
  COPYRIGHT: 'copyright',
} as const;

// Main 페이지 네임스페이스
export const MAIN_KEYS = {
  TITLE: 'title',
  ABOUT_US: {
    TITLE: 'aboutUs.title',
    CONTENT: 'aboutUs.content',
    BUTTON: 'aboutUs.button',
  },
  PROJECTS: {
    TITLE: 'projects.title',
    SUBTITLE: 'projects.subtitle',
  },
} as const;

// Pagination 네임스페이스
export const PAGINATION_KEYS = {
  LOADING: 'loading',
} as const;

// NotFound 네임스페이스
export const NOT_FOUND_KEYS = {
  TITLE: 'title',
  DESCRIPTION: 'description',
} as const;
```

**키 구조 설계 원칙:**

1. 네임스페이스별로 상수 객체 분리 (`HEADER_KEYS`, `FOOTER_KEYS` 등)
2. 중첩 구조는 dot notation으로 표현 (`aboutUs.title`)
3. `as const`로 리터럴 타입 보장
4. JSON 파일의 키 구조와 1:1 매핑

### 6. `middleware.ts`

언어 감지 및 리다이렉션 처리.

```typescript
import { NextRequest, NextResponse } from 'next/server';
import acceptLanguage from 'accept-language';
import { languages, cookieName, fallbackLng } from './src/i18n/settings';

acceptLanguage.languages([...languages]);

export const config = {
  // 정적 파일, API 등 제외
  matcher: [
    '/((?!api|_next/static|_next/image|assets|mockServiceWorker.js|sw.js|site.webmanifest|.*\\.png$|.*\\.svg$|.*\\.ico$|.*\\.jpg$|.*\\.mp4$|.*\\.woff$|.*\\.woff2$|.*\\.pdf$).*)',
  ],
};

export function middleware(req: NextRequest) {
  const pathname = req.nextUrl.pathname;

  let lng;
  // 쿠키 확인
  if (req.cookies.has(cookieName))
    lng = acceptLanguage.get(req.cookies.get(cookieName)?.value);
  // 요청 헤더 확인
  if (!lng) lng = acceptLanguage.get(req.headers.get('Accept-Language'));
  // 기본 언어
  if (!lng) lng = fallbackLng;

  // 언어가 이미 경로에 포함되어 있을 경우
  const localeInPath = languages.find((loc) =>
    req.nextUrl.pathname.startsWith(`/${loc}`)
  );

  // 경로에 언어가 포함되어있지 않는 경우, 언어를 포함한 경로로 리다이렉트
  if (!localeInPath && !req.nextUrl.pathname.startsWith('/_next')) {
    return NextResponse.redirect(
      new URL(`/${lng}${req.nextUrl.pathname}${req.nextUrl.search}`, req.url)
    );
  }

  // referer가 존재한다면 referer로부터 언어 감지를 시도하고 그에 따라 쿠키에 저장
  if (req.headers.has('referer')) {
    const refererUrl = new URL(req.headers.get('referer')!);
    const lngInReferer = languages.find((loc) =>
      refererUrl.pathname.startsWith(`/${loc}`)
    );
    const response = NextResponse.next();
    if (lngInReferer) response.cookies.set(cookieName, lngInReferer);
    return response;
  }

  return NextResponse.next();
}
```

### 7. `app/layout.tsx` (루트 레이아웃)

최소한의 구조만 유지. html/body 태그는 `[lng]/layout.tsx`에서 처리.

```tsx
export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return children;
}
```

### 8. `app/[lng]/layout.tsx` (언어별 레이아웃)

언어 코드를 받아 html lang 속성 설정 및 공통 레이아웃 처리.

```tsx
import { dir } from 'i18next';
import type { Language } from '@/src/types/i18n';

export default async function LngLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ lng: string }>;
}) {
  const { lng } = await params;

  return (
    <html lang={lng} dir={dir(lng)}>
      <body>
        {/* Header, Footer 등 공통 레이아웃 */}
        {children}
      </body>
    </html>
  );
}
```

## 사용법

### 서버 컴포넌트

```tsx
import { translate } from '@/src/i18n/server';
import { MAIN_KEYS } from '@/src/constants/i18nKeys';

export default async function Page({
  params,
}: {
  params: Promise<{ lng: string }>;
}) {
  const { lng } = await params;
  const { t } = await translate(lng, 'main');

  return (
    <div>
      <h1>{t(MAIN_KEYS.TITLE)}</h1>
      <p>{t(MAIN_KEYS.ABOUT_US.CONTENT)}</p>
    </div>
  );
}
```

### 클라이언트 컴포넌트

```tsx
'use client';

import { useTranslation } from '@/src/i18n/client';
import { HEADER_KEYS } from '@/src/constants/i18nKeys';

export default function Header({ lng }: { lng: string }) {
  const { t } = useTranslation(lng, 'header');

  return (
    <nav>
      <a href='/projects'>{t(HEADER_KEYS.PROJECTS)}</a>
      <a href='/contact'>{t(HEADER_KEYS.CONTACT)}</a>
    </nav>
  );
}
```

## 번역 파일 구조 예시

### `src/i18n/locales/ko/main.json`

```json
{
  "title": "메인 페이지",
  "aboutUs": {
    "title": "회사 소개",
    "content": "회사 소개 내용",
    "button": "더 알아보기"
  },
  "projects": {
    "title": "프로젝트",
    "subtitle": "프로젝트 설명"
  }
}
```

### `src/i18n/locales/en/main.json`

```json
{
  "title": "Main Page",
  "aboutUs": {
    "title": "About Us",
    "content": "About us content",
    "button": "Learn More"
  },
  "projects": {
    "title": "Projects",
    "subtitle": "Project description"
  }
}
```

### 상수와 JSON 키 매핑

| 상수                          | JSON 키               | 설명                   |
| ----------------------------- | --------------------- | ---------------------- |
| `MAIN_KEYS.TITLE`             | `"title"`             | 단일 키                |
| `MAIN_KEYS.ABOUT_US.TITLE`    | `"aboutUs.title"`     | 중첩 키 (dot notation) |
| `MAIN_KEYS.PROJECTS.SUBTITLE` | `"projects.subtitle"` | 중첩 키                |

## 언어 감지 흐름

1. 사용자가 `/` 접속
2. middleware에서 언어 감지 (쿠키 → Accept-Language → fallback)
3. `/ko` 또는 `/en`으로 리다이렉트
4. `app/[lng]/layout.tsx`에서 `params.lng` 수신
5. 각 페이지/컴포넌트에서 `lng` prop으로 번역 함수 사용

## 언어 감지 우선순위

1. URL 경로의 언어 파라미터 (`/ko`, `/en`)
2. 쿠키에 저장된 언어 설정 (`i18next`)
3. 브라우저 `Accept-Language` 헤더
4. 기본 언어 (ko)

## 참고 자료

- [Next.js i18n with app directory](https://www.locize.com/blog/next-app-dir-i18n)
- [i18next Documentation](https://www.i18next.com/)
- [react-i18next Documentation](https://react.i18next.com/)
