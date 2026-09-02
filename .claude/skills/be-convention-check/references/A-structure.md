# Category A — 구조/네이밍/Import

> 출처: `docs/backend-coding-convention.md` §2 디렉토리 구조, §3 Import 규칙

## 검사 규칙

### A1. 파일명/디렉토리명 camelCase
- **금지**: kebab-case (`shared-links/`, `error-messages.ts`)
- **필수**: camelCase (`sharedLinks/`, `errorMessages.ts`)
- 검사: `backend/src` 하위 모든 파일/디렉토리에서 `-`가 포함된 이름 탐색 (확장자 제외)
- 예외: 없음 (generated 제외)

### A2. 표준 파일 네이밍 패턴
- Controller: `{module}.controller.ts`
- Service: `{module}.service.ts`
- Module: `{module}.module.ts`
- DTO: `{verb}{Entity}.dto.ts` (예: `createUser.dto.ts`)
- Entity: `{entity}.entity.ts`
- Guard: `{type}Auth.guard.ts` 또는 `{type}Role.guard.ts`
- Strategy: `{provider}.strategy.ts`
- Types: `{module}.types.ts`
- 검사: 잘못된 패턴(예: `userController.ts`, `user-dto.ts`) 탐색

### A3. 절대경로 import (`@/`) 필수
- **금지**: 상대경로 (`./`, `../`)
- **필수**: `@/` (= `src/`)
- 검사: `from '\.\.?/` 패턴 grep
- 예외: `backend/src/api/generated/**` (자동 생성)

### A4. Import 순서 (권장 — soft check)
1. NestJS / 외부 패키지
2. 내부 서비스/모듈 (`@/...`)
3. 엔티티
4. 상수/타입
- 위반 시 INFO 레벨로만 보고 (자동 수정 X)

## 수정 가이드

- **A1 (kebab-case 디렉토리/파일)**: `git mv`로 rename. import 경로도 함께 갱신.
- **A2 (네이밍 패턴)**: 파일 rename + import 갱신.
- **A3 (상대경로)**: `from '../foo/bar'` → `from '@/foo/bar'`로 변환. 정확한 매핑은 파일의 실제 경로 기준으로 계산.
- **A4 (import 순서)**: 자동 수정 보류 (ESLint 룰로 처리 권장).

## 검사 명령 힌트

```bash
# A1
find backend/src -name "*-*" -not -path "*/generated/*" -not -path "*/node_modules/*"
# A3
grep -rn "from ['\"]\.\.\?/" backend/src --include="*.ts" | grep -v "/generated/"
```
