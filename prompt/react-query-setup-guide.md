# TanStack React Query 세팅 가이드

Next.js 15 + React 19 환경에서 TanStack React Query를 설정하는 가이드.

---

## 1. 패키지 설치

```bash
# React Query 설치
{package-manager} {install-command} @tanstack/react-query

# DevTools 설치 (개발 환경에서만 사용)
{package-manager} {install-command} @tanstack/react-query-devtools
```

### 변수 설명

- {package-manager}: npm, pnpm, yarn 등, 패키지 매니저.

**중요**: 사용자가 입력한 패키지 매니저에 맞는 패키지 설치 명령어를 사용해라. DevTools는 devDependencies로 설치해라.

---

## 2. QueryClientProvider 생성

`src/contexts/QueryClientProvider.tsx` 파일 생성:

```tsx
'use client';

import {
  QueryClient,
  QueryClientProvider as ReactQueryClientProvider,
} from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // 기본 staleTime: 0 (항상 stale 상태)
      staleTime: 0,
      // 기본 gcTime: 5분 (캐시 유지 시간)
      gcTime: 1000 * 60 * 5,
      // 실패 시 재시도 횟수
      retry: 1,
      // 윈도우 포커스 시 자동 refetch 비활성화
      refetchOnWindowFocus: false,
    },
  },
});

export default function QueryClientProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ReactQueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </ReactQueryClientProvider>
  );
}

// DevTools에서 queryClient 접근용 (디버깅 목적)
if (typeof window !== 'undefined') {
  window.__TANSTACK_QUERY_CLIENT__ = queryClient;
}
```

---

## 3. 루트 레이아웃에 Provider 적용

`app/[lng]/layout.tsx`에서 children을 `QueryClientProvider`로 감싸기:

```tsx
import QueryClientProvider from '@/src/contexts/QueryClientProvider';

export default async function RootLayout({
  children,
  params,
}: Readonly<{
  children: React.ReactNode;
  params: Promise<{ lng: string }>;
}>) {
  const { lng } = await params;
  return (
    <html lang={lng} dir={dir(lng)}>
      <body className='fixed_hidden'>
        <QueryClientProvider>
          <GlobalScroll>{/* 기존 컨텐츠 */}</GlobalScroll>
        </QueryClientProvider>
      </body>
    </html>
  );
}
```

> **중요**: `QueryClientProvider`는 `'use client'` 지시어가 필요하므로 별도 파일로 분리하여 Server Component인 레이아웃에서 import한다.

---

## 4. React Query DevTools 설정

### 4.1 기본 설정 (위 Provider에 포함)

```tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// Provider 내부에 추가
<ReactQueryDevtools initialIsOpen={false} />;
```

---

## 5. TypeScript 타입 확장 (선택)

`window.__TANSTACK_QUERY_CLIENT__` 타입 에러 방지를 위해 `src/types/global.d.ts` 생성:

```typescript
import type { QueryClient } from '@tanstack/react-query';

declare global {
  interface Window {
    __TANSTACK_QUERY_CLIENT__?: QueryClient;
  }
}
```

---

## 6. 사용 예시

### 기본 쿼리

```tsx
import { useQuery } from '@tanstack/react-query';

function Component() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['todos'],
    queryFn: () => fetch('/api/todos').then((res) => res.json()),
  });
}
```

### 뮤테이션

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function Component() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (newTodo) =>
      fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify(newTodo),
      }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
}
```

---

## 참고 문서

- [TanStack Query 공식 문서](https://tanstack.com/query/latest)
- [TanStack Query DevTools](https://tanstack.com/query/latest/docs/framework/react/devtools)
