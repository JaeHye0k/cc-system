# client-api-generate.md

이 문서는 프로젝트의 API 통신 및 데이터 관리 계층 구조를 정의하는 가이드라인입니다. 모든 AI 코드 생성은 이 규칙과 예시를 따릅니다.

### 1. 데이터 타입 정의 계층 (`src/types/*.ts`)

**변환 규칙:**

- **Case 변환:** `snake_case` → `camelCase` (예: `viewindex` → `viewIndex`)
- **ID 변환:** `_id` → `id` (중첩 객체 포함 모든 필드 대상)
- **날짜 필드:** `created_at` → `createdAt`, `updated_at` → `updatedAt`
- **복합 타입:** 유니언 타입(`enum` 또는 `string literal union`) 및 인터페이스 유니언 사용

```ts
// 백엔드 DTO (Data Transfer Object)
export interface TicketDTO {
  _id: string;
  title: string;
  start_time: string;
  hosted_by: string;
  viewindex: number;
  created_at: string;
  updated_at: string;
}

// 클라이언트 Model
export interface Ticket {
  id: string;
  title: string;
  startTime: string;
  hostedBy: string;
  viewIndex: number;
  createdAt: string;
  updatedAt: string;
}
```

### 2. 변환 계층: Mapper (`src/mappers/*.ts`)

DTO를 클라이언트 형식으로 매핑합니다. 반드시 `import type`을 사용합니다.

```ts
import type { Ticket, TicketDTO } from '../types/ticket';
import { parseDate } from '../utils/parser';

export const toTicket = (dto: TicketDTO): Ticket => ({
  id: dto._id,
  title: dto.title,
  startTime: dto.start_time,
  hostedBy: dto.hosted_by,
  viewIndex: dto.viewindex,
  createdAt: parseDate(dto.created_at),
  updatedAt: parseDate(dto.updated_at),
});
```

### 3. 호출 계층: Custom Fetch (`src/api/*.ts`)

`src/api/custom-fetch.ts`의 `customFetch`를 사용하여 요청을 수행합니다.

```ts
// src/api/custom-fetch.ts
const BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000';

interface CustomFetchOptions extends RequestInit {
  params?: Record<string, string>; // 쿼리 스트링 처리용
}

export async function customFetch<T>(
  endpoint: string,
  options: CustomFetchOptions = {}
): Promise<T> {
  const { params, headers, ...restOptions } = options;

  // 1. URL 및 쿼리 스트링 설정
  const url = new URL(`${BASE_URL}${endpoint}`);
  if (params) {
    Object.entries(params).forEach(([key, value]) =>
      url.searchParams.append(key, value)
    );
  }

  // 2. 공통 헤더 설정 (JSON 기반 예시)
  const defaultHeaders = {
    'Content-Type': 'application/json',
    // 'Authorization': `Bearer ${token}`, // 필요 시 토큰 추가
  };

  try {
    const response = await fetch(url.toString(), {
      ...restOptions,
      headers: {
        ...defaultHeaders,
        ...headers,
      },
    });

    // 3. 공통 에러 처리 (상태 코드별)
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));

      // 서버 에러 메시지가 있으면 우선 사용, 없으면 상태 코드별 기본 메시지
      const errorMessage = errorData.message || `에러 발생: ${response.status}`;

      if (response.status === 401) {
        console.error('인증 에러: 로그인이 필요합니다.');
        // 예: redirect('/login') 등 처리
      }

      throw new Error(errorMessage);
    }

    // 4. 성공 시 JSON 반환
    return await response.json();
  } catch (error) {
    console.error('Fetch Error:', error);
    throw error; // 에러를 호출부로 전파하여 처리하게 함
  }
}
```

api경로는 `src/constants/api.ts` 에서 관리합니다.

```ts
import { customFetch } from './custom-fetch';
import type { TicketDTO, TicketsResponseDTO } from '../types/ticket';
import { API_PATH } from '../constants/api';

export const fetchTickets = async (
  page: number
): Promise<TicketsResponseDTO> => {
  const url = API_PATH.fetchTickets(page);
  // customFetch는 { status, data, headers }를 반환함
  const response = await customFetch<TicketsResponseDTO>(url, {
    method: 'GET',
  });
  return response.data;
};
```

### 4. 상태 관리 계층: Tanstack-Query (`src/hooks/use*.ts`)

`select` 옵션을 사용하여 데이터를 변환(Mapping)합니다.

```ts
import { useQuery } from '@tanstack/react-query';
import { fetchTickets } from '../api/ticket';
import { toTicket } from '../mappers/ticket';

export const useTickets = (page: number) => {
  return useQuery({
    queryKey: ['tickets', page],
    queryFn: () => fetchTickets(page),
    select: (dto) => ({
      tickets: dto.data.map(toTicket),
      totalCount: dto.total_count,
    }),
  });
};
```

---

이 가이드는 프로젝트의 일관성을 유지하기 위한 핵심 문서입니다. 코드를 생성할 때 위 구조를 엄격히 준수하십시오.
