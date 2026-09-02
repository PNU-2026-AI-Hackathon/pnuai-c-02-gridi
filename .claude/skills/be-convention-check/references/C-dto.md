# Category C — DTO / Contract-First

> 출처: `docs/backend-coding-convention.md` §5 DTO 규칙, §12 Contract-First

## 검사 규칙

### C1. DTO 네이밍 패턴
| 타입 | 패턴 |
|------|------|
| Request | `{Verb}{Entity}RequestDto` |
| Response 생성 | `Create{Entity}ResponseDto` |
| Response 수정 | `Update{Entity}ResponseDto` |
| Response 단일조회 | `Get{Entity}DetailResponseDto` |
| Response 목록조회 | `Get{Entity}ListResponseDto` |
| Response 상태 | `Get{Entity}StatusResponseDto` |

- 검사: `backend/src/**/dto/*.dto.ts`에서 `class .*Dto`를 모두 수집 후 패턴 매칭
- 위반 예: `UserDto`, `CreateUserDtoRequest`, `UserResponse`

### C2. Response DTO는 generated interface를 `implements`
- 필수: `class XxxResponseDto implements IXxxResponseDto`
- 검사: `*ResponseDto` 클래스 중 `implements`가 없는 경우
- generated 타입 별칭 패턴: `type IXxxResponseDto = components['schemas']['XxxResponseDto']`

### C3. Response DTO는 static builder 메서드 필수
- 필수: `static buildFromCommandResult(result)` 또는 `static buildFromQueryResult(result)`
- 검사: `*ResponseDto` 클래스에 위 두 메서드가 모두 없는 경우

### C4. Request DTO 검증
- 모든 필드에 `@ApiProperty({ example: ... })`
- class-validator 데코레이터(`@IsString`, `@IsEmail`, ...)
- **금지**: `@IsEnum()` → `@IsIn(Object.values(CONST_OBJ))` 사용
- 검사: Request DTO 클래스의 필드 중 `@ApiProperty` 없음 / `@IsEnum` 사용

### C5. generated 파일 수동 편집 금지
- `backend/src/api/generated/**`에 git 변경이 있으면 경고
- (참고만 — fix 대상 아님)

### C6. Date → ISO String 변환
- Response DTO의 builder 내에서 Entity의 `Date` 필드를 그대로 할당하지 않고 `.toISOString()` 호출
- 검사: builder 메서드 안에서 `dto.xxxAt = result.xxxAt` (toISOString 없음) 패턴

## 수정 가이드

- **C1**: 클래스 rename + 모든 import/사용처 갱신.
- **C2**: `import type { components } from '@/api/generated/schema'` 추가 → 타입 별칭 정의 → `implements` 추가. 컴파일 에러 발생 시 필드 정합성 수정.
- **C3**: `static buildFromQueryResult(result: XxxQueryResult): XxxResponseDto { ... }` 추가. Controller에서 직접 인스턴스화하던 코드도 함께 변경.
- **C4**: `@IsEnum(X)` → `@IsIn(Object.values(X))`. 누락된 `@ApiProperty` 추가.
- **C6**: `dto.createdAt = result.createdAt.toISOString()` 형태로 수정.

## 검사 명령 힌트

```bash
# C2
grep -rn "class .*ResponseDto" backend/src --include="*.dto.ts" | grep -v "implements"
# C3
grep -L "buildFromCommandResult\|buildFromQueryResult" backend/src/**/dto/*.dto.ts
# C4
grep -rn "@IsEnum" backend/src --include="*.dto.ts"
```
