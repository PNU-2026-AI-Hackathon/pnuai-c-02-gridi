---
name: be-convention-check
description: >
  gridi-app 백엔드 코딩 컨벤션(docs/backend-coding-convention.md) 위반 사항을 카테고리별로
  병렬 검사하고 수정하는 스킬. 6개 카테고리(구조/네이밍, 계층형 아키텍처, DTO/Contract-First,
  Entity/Type, 코드 스타일, 에러/Swagger)를 각각 독립된 에이전트로 동시 실행한다.
  "백엔드 컨벤션 검사", "BE convention 위반 점검", "백엔드 컨벤션 전수조사" 요청 시 사용.
---

# Backend Convention Check (gridi-app)

`docs/backend-coding-convention.md`의 컨벤션을 카테고리별로 병렬 검사하고, 사용자 승인 후 병렬 수정한다.

## 핵심 원칙

- **Single Source of Truth**: `docs/backend-coding-convention.md`. 이 문서에 명시되지 않은 규칙은 검사하지 않는다.
- **병렬 처리**: 6개 카테고리는 서로 독립적이므로 detect/fix 모두 한 메시지에서 동시 dispatch.
- **검사 → 보고 → 승인 → 수정**: 검사 결과를 통합 리포트로 보여주고, 사용자가 명시 승인할 때까지 수정하지 않는다.
- **수정 범위 제한**: 검사된 위반 외 리팩토링/개선 금지. 컨벤션 문서에 명시된 규칙만 강제.

## 검사 대상

`backend/src/**/*.ts` (단, 다음 제외)
- `backend/src/api/generated/**` (자동 생성)
- `backend/src/migrations/**` (TypeORM 마이그레이션은 별도 컨벤션)
- `**/*.spec.ts`, `**/__tests__/**` (테스트 코드는 별도)

## 6개 카테고리

각 카테고리별 detect agent와 fix agent의 역할/검사 항목은 `references/` 디렉토리의 개별 문서 참조.

| # | 카테고리 | 검사 항목 요약 | 참조 문서 |
|---|---------|--------------|---------|
| A | 구조/네이밍/Import | 디렉토리 camelCase, 파일명 패턴, `@/` 절대경로 | `references/A-structure.md` |
| B | 계층형 아키텍처 | Controller→Command→Service→Result→Response, Service의 DTO/generated import 금지, Controller의 Entity 변환 금지 | `references/B-architecture.md` |
| C | DTO/Contract-First | DTO 네이밍, `@ApiProperty` 필수, Response DTO `implements` generated, builder 패턴 | `references/C-dto.md` |
| D | Entity/Type | UUID PK + `@PrimaryColumn`, `varchar` (enum 금지), `simple-json` 금지, length 명시, `as const` enum, types/ 위치 | `references/D-entity-type.md` |
| E | 코드 스타일 | 파라미터 2개+ → 객체, `const` 우선, `try-catch` 최소화, early return, depth ≤ 2 | `references/E-style.md` |
| F | 에러/Swagger/Module | `errorMessages.ts` 중앙화, 하드코딩 금지, `@ApiTags`/`@ApiOperation` 필수 | `references/F-errors-swagger.md` |

## 워크플로우

### Phase 1 — 검사 (Detect, 병렬)

한 메시지 안에서 6개의 Agent 호출을 동시에 dispatch한다. 각 에이전트는:

1. 해당 카테고리의 reference 문서를 읽음
2. `backend/src/` 전 영역에서 위반 사례를 grep/glob/read로 수집
3. **수정하지 않음** — 위반 목록만 반환

각 detect agent의 출력 형식:

```
## Category {X} — {이름}

### {위반 규칙명}
- `{file}:{line}` — {짧은 설명}
- `{file}:{line}` — {짧은 설명}

### {다른 위반 규칙명}
...

(위반 없음일 경우 "✅ 위반 없음")
```

dispatch 예시 (한 메시지에 6개 Agent 호출):

```
Agent A: subagent_type=general-purpose, prompt="be-convention-check Phase 1 — Category A 검사. references/A-structure.md 를 읽고 backend/src/ (generated/migrations/spec 제외) 에서 위반 사항 수집. 수정 금지. 출력 형식은 SKILL.md 참조."
Agent B: ... (Category B)
... (C, D, E, F)
```

검사 완료 후 **6개 결과를 통합 리포트로 사용자에게 제시**:

```
# 백엔드 컨벤션 검사 결과

## 요약
- Category A: {n}건
- Category B: {n}건
...
- 총 {N}건

## 상세
{각 카테고리별 위반 목록}

수정을 진행할까요? (카테고리 단위로 선택 가능)
```

### Phase 2 — 수정 (Fix, 병렬, 승인 후)

사용자가 승인한 카테고리에 대해서만 fix agent를 병렬 dispatch.

각 fix agent는:

1. Phase 1에서 자기 카테고리가 찾은 위반 목록을 입력으로 받음
2. reference 문서의 "수정 가이드" 섹션에 따라 수정
3. **본인 카테고리에 한정**해서만 수정 (다른 카테고리 위반을 목격해도 무시)
4. 수정한 파일 목록과 변경 요약을 반환

dispatch는 한 메시지에 승인된 카테고리 수만큼 Agent 호출을 묶는다.

### Phase 3 — 검증

수정 완료 후:

1. `cd backend && npx tsc --noEmit` 으로 타입 체크
2. 타입 에러 발생 시 해당 fix agent에게 재의뢰 (또는 사용자에게 보고)
3. 최종 변경 파일 목록 + 카테고리별 처리 건수 보고

## 주의사항

- **fix agent 간 파일 충돌**: 같은 파일을 여러 카테고리가 수정할 수 있다. fix는 카테고리별 순차로 진행하거나, 충돌 가능성이 있는 카테고리 쌍(B/C, D/E)은 순차로 묶는다. 안전을 위해 **검사는 병렬, 수정은 카테고리별 순차**가 기본. 사용자가 명시적으로 "병렬 수정" 요청 시에만 동시 수정.
- **수정은 항상 사용자 승인 후**: Phase 1 결과를 보여주기 전에 자동 수정 금지.
- **컨벤션 문서가 변경되면 references/ 도 함께 갱신**한다.
