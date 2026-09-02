# Category E — 코드 스타일

> 출처: `docs/backend-coding-convention.md` §4 메소드 파라미터 규칙, 코드 스타일 원칙

## 검사 규칙

### E1. 메소드 파라미터 2개 이상 → 객체로 감싸기
- 검사: `*.service.ts`, `*.controller.ts` (controller는 `@Body/@Param/@Query/@Request` 데코레이터 사용 시 예외)에서 파라미터 2개 이상인 메서드
- public/private 모두 적용
- **예외**:
  - 생성자 DI
  - NestJS 데코레이터 기반 Controller 메서드 (`@Body`, `@Param`, `@Query`, `@Request`, `@Headers` 등이 있는 파라미터)
  - 프레임워크 인터페이스 오버라이드 (`validate()`, `canActivate()`, `intercept()` 등)
  - 파라미터 1개
- 네이밍: `{MethodName}Params`

### E2. `const` 우선, `let` 지양
- 검사: 재할당이 없는 `let` 변수 (best-effort: 단일 할당 후 수정 없음)
- 자동 수정 가능한 명백한 케이스만 수정

### E3. `try-catch` 최소화
- **금지 패턴 1**: catch 블록이 `throw error` 또는 `throw err`만 하는 경우 (의미 없는 재throw)
- **금지 패턴 2**: NestJS 예외(`NotFoundException`, `BadRequestException` 등)를 try-catch로 감쌈
- **허용**: 외부 API 호출, 파일 I/O, JSON parse 등 예측 불가능한 실패 처리

### E4. Early return / 중첩 깊이 ≤ 2
- 검사: `if (...) { ... } else { ... }`에서 if 또는 else 블록 안에서 throw/return하는 패턴 (else 제거 가능)
- 검사: depth 3 이상의 중첩 (best-effort)

### E5. 매직 넘버 / 하드코딩 (참고만)
- 검사 안 함 (별도 컨벤션 없음)

## 수정 가이드

- **E1**: `method(a: A, b: B)` → 인터페이스 `MethodNameParams { a: A; b: B }`를 `types/`에 추가 → `method(params: MethodNameParams)` 변경 → 호출처 모두 객체로 변환.
- **E2**: 단일 할당 `let x = ...; (재할당 없음)` → `const x = ...`.
- **E3**: 의미 없는 try-catch 제거. NestJS 예외를 감싼 try-catch는 제거하고 직접 throw로 전파.
- **E4**: `if (cond) { A } else { B }`에서 A 또는 B가 throw/return이면 if 블록을 guard로 변환.

## 검사 명령 힌트

```bash
# E3 — 의미 없는 재throw
grep -rn -A2 "} catch" backend/src --include="*.ts" | grep -B1 "throw error\|throw err\b"
```

## 주의

E1, E4는 false positive가 많으므로 fix agent가 신중하게 판단. 애매하면 INFO 레벨 보고만.
