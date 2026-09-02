# Category F — 에러 처리 / Swagger / Module

> 출처: `docs/backend-coding-convention.md` §8 에러 처리, §10 Module, §11 Swagger

## 검사 규칙

### F1. 에러 메시지는 `constants/errorMessages.ts`에서 import
- **금지**: `throw new NotFoundException('Project not found')` (하드코딩 문자열)
- **필수**: `throw new NotFoundException(COMMON_ERRORS.PROJECT_NOT_FOUND)`
- 검사: `throw new \w+Exception\(['"]` 패턴 — 따옴표로 시작하는 인자

### F2. NestJS 예외 타입 사용
- 검사: `throw new Error(...)` 대신 NestJS 예외 사용
- 매핑:
  - 유효성 → `BadRequestException`
  - 인증 → `UnauthorizedException`
  - 권한 → `ForbiddenException`
  - 미존재 → `NotFoundException`
  - 충돌 → `ConflictException`

### F3. Controller는 `@ApiTags()` 필수
- 검사: `@Controller(...)` 위에 `@ApiTags(...)`가 없는 클래스

### F4. 모든 엔드포인트는 `@ApiOperation({ summary })` 필수
- 검사: `@Get/@Post/@Put/@Patch/@Delete` 메서드 위에 `@ApiOperation` 없음

### F5. 인증 필요 엔드포인트는 `@ApiBearerAuth()` 필수
- 검사: `@UseGuards(JwtAuthGuard ...)` 사용 시 `@ApiBearerAuth()` 누락

### F6. Module 구조
- 검사: `@Module()` 의 `imports`에 `TypeOrmModule.forFeature([...])` 누락 (해당 모듈에 entity가 있는 경우)
- 검사: 다른 모듈에서 import되는 service가 `exports`에 누락 (best-effort)

## 수정 가이드

- **F1**: 하드코딩된 에러 메시지를 `backend/src/constants/errorMessages.ts`에 상수로 추가 → import 후 교체. 이미 같은 메시지의 상수가 있으면 재사용.
- **F2**: `throw new Error('foo')` → 적절한 NestJS 예외 + errorMessages 상수.
- **F3**: 모듈 이름 기반 `@ApiTags('Module Name')` 추가.
- **F4**: 메서드 의도 기반 한국어 summary 추가 (`@ApiOperation({ summary: '~~ 조회' })`).
- **F5**: `@ApiBearerAuth()` 데코레이터 추가.
- **F6**: 누락된 import/exports 추가.

## 검사 명령 힌트

```bash
# F1
grep -rEn "throw new \w+Exception\(['\"]" backend/src --include="*.ts" | grep -v __tests__
# F2
grep -rn "throw new Error(" backend/src --include="*.ts" | grep -v __tests__
# F3
grep -rL "@ApiTags" backend/src --include="*.controller.ts"
```
