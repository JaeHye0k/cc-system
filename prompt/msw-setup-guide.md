# MSW (Mock Service Worker) 설정 가이드

Next.js App Router 환경에서 MSW 핸들러를 생성하기 위한 가이드.

---

## 1. 설치

```bash
{package-manager} {install-command} msw @faker-js/faker
npx msw init public --save
```

### 변수 설명

- {package-manager}: npm, pnpm, yarn 등, 패키지 매니저.

**중요**: 사용자가 입력한 패키지 매니저에 맞는 패키지 설치 명령어를 사용해 devDependencies로 설치해라.

---

## 2. MSW 초기화 설정

### SSR/CSR 초기화 분리

| 환경 | 초기화 방법          | 파일                   |
| ---- | -------------------- | ---------------------- |
| SSR  | `instrumentation.ts` | `src/mocks/server.ts`  |
| CSR  | useEffect            | `src/mocks/browser.ts` |

### instrumentation.ts (SSR)

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    if (process.env.NODE_ENV === 'development') {
      const { server } = await import('@/src/mocks/server');
      server.listen({ onUnhandledRequest: 'bypass' });
      console.log('[MSW] Server started');
    }
  }
}
```

**`instrumentation.ts` 사용 이유**: Next.js 서버 부트스트랩 시 가장 먼저 실행되어, 서버 컴포넌트의 fetch보다 MSW가 먼저 활성화됨.

**`NEXT_RUNTIME === 'nodejs'` 체크 이유**: 빌드 타임 최적화. Next.js가 Node.js/Edge 두 번들을 생성하는데, `msw/node`는 Node.js 전용 API 사용. 조건문으로 Edge 번들에서 import가 제거됨 (Dead Code Elimination).

### server.ts (Node.js)

```typescript
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

### browser.ts (브라우저)

```typescript
import { setupWorker } from 'msw/browser';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);
```

### index.ts (클라이언트 초기화)

```typescript
export async function initClientMsw() {
  const { worker } = await import('./browser');
  await worker.start();
  console.log('[MSW] Client started');
}
```

### msw-component.tsx (클라이언트 컴포넌트)

```typescript
'use client';

import { useEffect, useState } from 'react';

export const MSWComponent = ({ children }: { children: React.ReactNode }) => {
  const [mswReady, setMswReady] = useState(false);

  useEffect(() => {
    const init = async () => {
      const { initClientMsw } = await import('@/src/mocks/index');
      await initClientMsw();
      setMswReady(true);
    };
    if (!mswReady) init();
  }, [mswReady]);

  return <>{children}</>;
};
```

### package.json 스크립트

```json
{
  "scripts": {
    "dev:msw": "NEXT_PUBLIC_MSW_ACTIVE=1 next dev --turbopack"
  }
}
```

---

## 프로젝트 구조

```
├── instrumentation.ts              # SSR용 MSW 서버 초기화
├── public/mockServiceWorker.js     # 브라우저용 Service Worker
└── src/
    ├── mocks/
    │   ├── handlers.ts             # MSW 핸들러 정의
    │   ├── browser.ts              # 브라우저용 Worker 설정
    │   ├── server.ts               # Node.js(SSR)용 서버 설정
    │   └── index.ts                # 초기화 함수
    ├── components/common/msw/
    │   └── msw-component.tsx       # 클라이언트 MSW 초기화 컴포넌트
    ├── types/[entity].ts           # API 응답 타입 (DTO, 클라이언트 타입)
    └── constants/api.ts            # API 경로 상수
```

---

## 핸들러 작성 규칙

### 기본 구조

```typescript
import { http, HttpResponse } from 'msw';
import { faker } from '@faker-js/faker';
import { API_PATH } from '@/src/constants/api';
import type { [Entity]DTO } from '@/src/types/[entity]';

const baseUrl = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000';

// Faker 시드 설정 (일관된 데이터 생성)
faker.seed(123);

const createMock[Entity] = (id: string, index: number): [Entity]DTO => ({
  // mock 데이터 생성
});

export const handlers = [
  // 목록 API
  http.get(`${baseUrl}${API_PATH.[ENTITY]_LIST}`, ({ request }) => {
    const url = new URL(request.url);
    const page = parseInt(url.searchParams.get('page') || '1', 10);
    const perPage = parseInt(url.searchParams.get('perPage') || '10', 10);

    const data = Array.from({ length: perPage }, (_, i) =>
      createMock[Entity](`[entity]-${(page - 1) * perPage + i}`, (page - 1) * perPage + i)
    );

    return HttpResponse.json({ result: true, totalCount: 100, data });
  }),

    // 상세 API
    http.get(`${baseUrl}${API_PATH.[ENTITY]_DETAIL}/:id`, ({ params }) => {
        const { id } = params;
        const entity = createMock[Entity](id as string, 0);

        return HttpResponse.json({
            result: true,
            data: entity,
        });
    }),
];
```

### 타입 정의 구조 (types/[entity].ts)

```typescript
// DTO (백엔드 형식: snake_case, _id)
export interface [Entity]sResponseDTO {
  result: boolean;
  totalCount: number;
  data: [Entity]DTO[];
}

export interface [Entity]ResponseDTO {
  result: boolean;
  data: [Entity]DTO;
}

export interface [Entity]DTO {
  _id: string;
  created_at: string;
  updated_at: string;
}

// 클라이언트 형식 (camelCase, id)
export interface [Entity]sResponse {
  result: boolean;
  totalCount: number;
  data: [Entity][];
}

export interface [Entity]Response {
  result: boolean;
  data: [Entity];
}

export interface [Entity] {
  id: string;
  createdAt: string;
  updatedAt: string;
}
```

### API 경로 상수 (constants/api.ts)

```typescript
export const API_PATH = {
  [ENTITY]_LIST: '/[entity]/list',
  [ENTITY]_DETAIL: '/[entity]/show',
};
```

---

## Faker.js 사용법

```typescript
import { faker } from '@faker-js/faker';

faker.seed(123); // 일관된 데이터 생성

// 날짜
const createdDate = faker.date.past({ years: 2 });
const updatedDate = faker.date.between({ from: createdDate, to: new Date() });

// 텍스트
faker.lorem.sentence();
faker.lorem.paragraph(3);

// 숫자
faker.number.int({ min: 1, max: 100 });

// 배열 선택
faker.helpers.arrayElement(['옵션1', '옵션2', '옵션3']);

// 이미지 URL
`https://picsum.photos/seed/${faker.number.int({ min: 1, max: 1000 })}/800/600`;
```

### 한국어 데이터

```typescript
const koreanCompanies = [
  '삼성전자',
  'LG전자',
  '현대자동차',
  '네이버',
  '카카오',
];
const client = faker.helpers.arrayElement(koreanCompanies);
```

---

## middleware 설정

```typescript
// middleware.ts
export const config = {
  matcher: ['/((?!mockServiceWorker.js).*)'],
};
```

- Next.js 16+ 일경우 `middleware.ts` 대신 `proxy.ts` 라는 이름으로 사용하라.

---

## 주의사항

1. 타입 import 시 `type` 키워드 사용
2. 경로 alias 사용: `@/src/...`. 기존 tsconfig.json 파일의 경로 alias를 수정하지 않는다.
3. baseUrl: `process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000'`
4. DTO: snake_case, `_id` / 클라이언트: camelCase, `id`
5. faker.js 필수 사용, `faker.seed(123)` 설정
6. 날짜 일관성: `created_at < updated_at`
