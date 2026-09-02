# gridi 프로젝트 Claude Code 지침

## 언어 설정

> **필수**: 모든 응답은 **한국어**로 작성합니다. (코드, 커밋 메시지, 기술 용어 제외)

## 필수 규칙

### 백로그 구현 시 workflow-orchestrator 스킬 필수 사용

> **⚠️ 중요**: 백로그 구현 요청 시 반드시 `/workflow-orchestrator` 스킬을 사용해야 합니다.

백로그 구현 요청 예시:
- "BL-XXX 구현해줘"
- "백로그 XXX 개발해줘"
- "{기능명} 구현해줘"
- "이 유저스토리 구현해줘"
- "다음 백로그로 넘어갑시다"

위와 같은 요청을 받으면:

```
1. 즉시 Skill 도구로 workflow-orchestrator 호출
2. 스킬의 Phase 0 ~ Phase 5 워크플로우 순차 실행
3. CI 검증 통과 확인 후 완료
```

### 단일 작업 요청은 개별 스킬 사용

백로그 전체가 아닌 단일 작업 요청 시에는 해당 스킬 직접 사용:

| 요청 유형 | 사용 스킬 |
|----------|----------|
| "유저스토리 작성해줘" | user-story-generator |
| "와이어프레임 만들어줘" | wireframer |
| "테크스펙 작성해줘" | tech-lead |
| "디자인 만들어줘" | product-designer |
| "API 스펙 작성해줘" | be-spec |
| "백엔드 구현해줘" | be-main |
| "프론트엔드 구현해줘" | fe-main |
| "테스트 케이스 작성해줘" | qa-tc |
| "E2E 테스트 작성해줘" | qa-e2e |

## 프로젝트 컨벤션

### 도메인 구성

| 용도 | 도메인 |
|------|--------|
| 메인 도메인 | gridi.ai |
| API 서버 | api.gridi.ai |
| 프론트엔드 앱 | app.gridi.ai |
| 랜딩 페이지 | www.gridi.ai |
| 한국 도메인 (리다이렉트) | gridi.kr → gridi.ai |
| 한국 앱 (리다이렉트) | app.gridi.kr → app.gridi.ai |

### 인프라 관리

> 모든 인프라 변경(AWS 리소스, DNS, 도메인 설정 등)은 Terraform 코드(IaC)로 관리하며,
> 콘솔에서 직접 수정하지 않는다. **인프라 코드와 배포 스크립트는 비공개 저장소에서 관리되며 본 저장소에 포함되지 않는다.**


### UUID Primary Key

모든 엔티티의 PK는 UUID 사용 (Auto-increment 금지)

### TypeORM `simple-json` 컬럼 타입 금지

> **⚠️ 필수**: `simple-json` 컬럼 타입으로 시스템 데이터(config, settings, options 등)를 관리하지 않습니다.

- JSON 컬럼은 마이그레이션으로 스키마 변경을 추적할 수 없어 데이터 정합성 위험
- 타입 안전성이 없고 런타임에서만 검증 가능
- **모든 설정값은 개별 typed 컬럼으로 정의**: `maxDepth: int`, `language: varchar(10)` 등
- 예외: 사용자가 자유롭게 입력하는 비정형 메타데이터(태그, 메모 등)만 `simple-json` 허용

### TypeScript Enum 금지 - as const 사용

> **⚠️ 필수**: TypeScript `enum`은 사용하지 않습니다.

모든 열거형은 `as const` 객체 + `typeof` 타입으로 정의합니다:

```typescript
export const STATUS = {
  ACTIVE: 'active',
  INACTIVE: 'inactive',
} as const;
export type Status = (typeof STATUS)[keyof typeof STATUS];
```

규칙:
- 상수 객체명: UPPER_SNAKE_CASE (예: `AUTH_PROVIDER`, `PROJECT_STATUS`)
- DB 컬럼: `type: 'varchar'` 사용 (`type: 'enum'` 금지)
- DTO 검증: `@IsIn(Object.values(CONST_OBJ))` 사용 (`@IsEnum` 금지)
- 에러 메시지: `backend/src/constants/errorMessages.ts`에 통합 관리

### 파일 네이밍 컨벤션

> **⚠️ 필수**: 파일명과 디렉토리명은 **camelCase**를 사용합니다. kebab-case 금지.

- 예: `errorMessages.ts` (O), `error-messages.ts` (X)
- 예: `browserSessions.service.ts` (O), `browser-sessions.service.ts` (X)
- 예: `browserSessions/` (O), `browser-sessions/` (X)

### 메소드 파라미터 규칙 (2개 이상 → 객체)

> **⚠️ 필수**: 파라미터가 2개 이상인 메소드는 반드시 객체로 감싸서 전달합니다.

```typescript
// GOOD
async uploadScreenshot(params: UploadScreenshotParams): Promise<string> {
  const { projectId, nodeId, buffer } = params;
}

// BAD
async uploadScreenshot(projectId: string, nodeId: string, buffer: Buffer): Promise<string> {}
```

- public/private 메소드 모두 적용
- 파라미터 객체는 `types/{module}.types.ts`에 인터페이스로 정의
- 네이밍: `{MethodName}Params` 또는 Command/Query 패턴과 통합
- 예외: 생성자 DI, NestJS 데코레이터 기반 Controller 메소드, 프레임워크 인터페이스 오버라이드, 파라미터 1개

### 코드 스타일 원칙

> **⚠️ 필수**: 아래 3가지 원칙을 따릅니다.

**`const` 우선, `let` 지양**
- 변수 선언은 `const`가 기본
- `let`은 루프 카운터, 누적값 등 재할당이 불가피한 경우에만 사용

**`try-catch` 최소화**
- 외부 API 호출, 파일 I/O 등 예측 불가능한 실패를 처리할 때만 사용
- NestJS 예외(`NotFoundException` 등)는 글로벌 필터가 처리하므로 감싸지 않음
- catch 후 단순 재throw(`throw error`)는 금지

**Early Return으로 `if-else` 깊이 최소화**
- 실패 조건을 먼저 검사하고 `throw`/`return`으로 빠져나감 (Guard Clause)
- `else`는 가능한 쓰지 않음 — early return으로 대체
- 중첩 depth는 최대 2단계 이내 유지

### Import 절대경로 필수 (`@/`)

> **⚠️ 필수**: 모든 import는 `@/` 절대경로를 사용합니다. 상대경로(`./`, `../`) 금지.

- `@/`는 `src/`에 매핑됨 (프론트엔드, 백엔드 동일)
- ESLint `no-relative-import-paths` 규칙으로 강제
- 유일한 예외: `frontend/src/components/providers/intl-provider.tsx`의 messages JSON import (`src/` 외부)

```typescript
// GOOD
import { User } from '@/users/entities/user.entity';
import { AppService } from '@/app.service';

// BAD
import { User } from '../../users/entities/user.entity';
import { AppService } from './app.service';
```

### 백엔드 계층형 아키텍처 (Command/Query/Result 패턴)

> **⚠️ 필수**: 모든 백엔드 API는 아래 패턴을 따릅니다.

```
Controller                         Service                        Repository
──────────────────────────        ──────────────────────────      ──────────────
@Body() dto (class-validator)
  ↓
  Command 생성 (interface)
  ↓
  service.method(command)     ──→  const { ... } = command
                                     ↓                            .find() / .save()
                                   Entity 처리
                                     ↓
  ← CommandResult (interface)  ←── return { ... }
  ↓
  Response.buildFromCommandResult(result)
  ↓
  HTTP Response (implements generated interface)
```

핵심 규칙:
- **Service는 Request DTO, Response DTO를 절대 import하지 않음** (계층 격리)
- **Service는 Command/Query interface를 파라미터로 받고, CommandResult/QueryResult를 반환**
- **Controller에서 Entity→Result 변환 금지** — 반드시 Service에서 처리
- **Response 클래스**는 `static buildFromCommandResult(result)` 또는 `static buildFromQueryResult(result)` 사용
- **Response 클래스**는 generated interface를 `implements` (OpenAPI 정합성 보장)
- **Command/Query/Result는 interface**로 `types/{module}.types.ts`에 정의
- **Command 핸들러(데이터 변경 메서드)는 반드시 `@Transactional()` 사용** — 여러 DB 연산을 포함하는 서비스 메서드는 `@Transactional()` 데코레이터로 원자성 보장. 특히 생성(create + 연관 레코드), 삭제(연관 데이터 정리), 상태 변경(read-modify-write) 시 필수

파일 구조:
```
backend/src/{module}/
├── {module}.controller.ts          # Command 생성 → service 호출 → Response.buildFrom*
├── {module}.service.ts             # Command 수신 → Result 반환 (DTO import 금지)
├── dto/
│   ├── createXxx.dto.ts            # Request DTO + Response DTO 클래스
│   └── getXxx.dto.ts
├── types/
│   └── {module}.types.ts           # Command/Query/Result interfaces
└── entities/
    └── {entity}.entity.ts
```

### Request/Response DTO 네이밍 규칙

> **⚠️ 필수**: 모든 DTO는 아래 네이밍 패턴을 따릅니다.

- **Request**: `{Verb}{Entity}RequestDto` (예: `CreateExplorationProjectRequestDto`)
- **Response**: `{Verb}{Entity}{Qualifier}ResponseDto`
  - 생성: `Create{Entity}ResponseDto`
  - 수정: `Update{Entity}ResponseDto`
  - 단일 조회: `Get{Entity}DetailResponseDto`
  - 목록 조회: `Get{Entity}ListResponseDto`
  - 상태 조회: `Get{Entity}StatusResponseDto`
  - 기타 행위: `Start{Entity}ResponseDto`, `Stop{Entity}ResponseDto` 등
- **Command**: `{Verb}{Entity}Command` (예: `CreateExplorationProjectCommand`)
- **Query**: `Get{Entity}Query` (예: `GetExplorationProjectQuery`)
- **CommandResult**: `{Verb}{Entity}CommandResult`
- **QueryResult**: `Get{Entity}QueryResult`

### 프론트엔드: Generated API 클라이언트 필수 사용

> **⚠️ 필수**: 프론트엔드에서 API 호출 시 반드시 generated API 클라이언트만 사용합니다.

- API 클라이언트: `frontend/src/api/client.ts`에서 export
- 직접 `axios`, `fetch`, `apiAxios` 호출 금지
- OpenAPI 스펙 변경 시: `npm run api:generate` → generated 클라이언트 자동 업데이트

```typescript
// GOOD
import { explorationProjectsApiClient } from '@/api/client';
const { data } = await explorationProjectsApiClient.getExplorationProject({ id });

// BAD
import { apiAxios } from '@/api/client';
const { data } = await apiAxios.get(`/exploration-projects/${id}`);
```

### 환경변수 관리 — 4-Layer Pipeline

> 환경변수는 `시크릿 저장소 → 배포 스크립트 → 컨테이너(pass-through) → NestJS Config` 파이프라인으로 주입한다.
> `process.env` 직접 접근을 금지하고 `@nestjs/config`의 `registerAs` 네임스페이스 config를 통해서만 읽는다.
> Config에 fallback이 있으면 `configService.get()`, 없으면 `getOrThrow()`를 사용한다.
> fallback은 로컬 개발 편의용으로 NestJS config에만 허용하며, 배포 계층에서는 금지한다.
> **구체적인 파라미터 경로와 배포 스크립트는 비공개 저장소에서 관리된다.**


### CI 검증 필수

코드 변경 후 반드시 CI 파이프라인 통과 확인

### 단계적 배포 설계 — Expand & Contract (main 향 PR 필수)

> **⚠️ 필수**: main 브랜치로 머지하는 모든 PR은 단계적(zero-downtime) 배포가 가능하도록 설계합니다.

**원칙: breaking change를 만들지 않는다**

기존 변경사항이 breaking인지 아닌지를 판단하는 것이 아니라, **변경사항의 의도를 파악하고 breaking을 제거하는 방향으로 코드와 마이그레이션을 설계**합니다.

**Expand & Contract 패턴 (DB 스키마 변경 시 필수):**

```
Phase 1 (Expand) — PR #1:
  - 새 테이블/컬럼 추가
  - 기존 데이터를 새 구조로 복사
  - 코드에서 dual-write (기존 + 새 구조에 동시 기록)
  - 읽기는 새 구조에서
  → 이전 코드와 공존 가능, 일반 Blue/Green 배포

Phase 2 (Contract) — PR #2 (Phase 1 배포 확인 후):
  - 기존 테이블/컬럼 DROP
  - dual-write 코드 제거
  → 이전 코드는 이미 없으므로 안전
```

**예시 — 테이블 교체 (user_connection → UserAuthProvider):**
```
PR #1: UserAuthProvider 테이블 생성 + 데이터 이관 + dual-write
  → 기존 코드: user_connection 읽기/쓰기 (정상 동작)
  → 새 코드: UserAuthProvider 읽기 + 양쪽 쓰기 (정상 동작)

PR #2: user_connection DROP + User.provider 컬럼 DROP + dual-write 제거
  → 새 코드만 존재하므로 안전
```

**적용 대상:**
- DB 컬럼/테이블 삭제 또는 이름 변경
- API 엔드포인트 경로 변경 또는 응답 필드 삭제
- 환경변수 이름 변경 (SSM 키 경로 변경)

**정말 분리가 불가능한 경우에만** 사용자에게 보고:
- 배포 절차 (점검 모드, 마이그레이션 순서)
- 예상 다운타임
- 롤백 계획

### Figma 가이드라인

디자인 작업 시 `docs/figma-guidelines.md` 규칙 준수

### 다국어(i18n) 필수 적용

프론트엔드 UI 작업 시 반드시 다국어 지원을 함께 구현:

1. **번역 파일 업데이트**: `frontend/messages/ko.json`, `frontend/messages/en.json`에 새 키 추가
2. **useTranslations 훅 사용**: 하드코딩된 텍스트 대신 `t("key")` 형태로 작성
3. **네임스페이스 구분**: auth, nav, project, workflow 등 기능별로 번역 키 그룹화

```tsx
// 예시
import { useTranslations } from "next-intl";

function MyComponent() {
  const t = useTranslations("nav");
  return <span>{t("dashboard")}</span>;
}
```
