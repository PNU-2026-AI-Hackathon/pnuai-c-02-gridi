# Category B — 계층형 아키텍처 (Command/Query/Result 패턴)

> 출처: `docs/backend-coding-convention.md` §4 계층형 아키텍처

## 검사 규칙

### B1. Service는 Request/Response DTO를 import하지 않는다
- 검사: `*.service.ts` 파일에서 `from '@/.*\.dto'` 또는 `RequestDto|ResponseDto` 타입 import 탐색
- **금지**: Service가 DTO 클래스를 직접 import
- **허용**: Command/Query/Result interface (`types/{module}.types.ts`)

### B2. Service는 generated OpenAPI 타입을 import하지 않는다
- 검사: `*.service.ts`에서 `from '@/api/generated'` 또는 `components\['schemas'\]` 사용 탐색
- 이유: 계층 격리 — Service는 Command/Query/Result만 다룸

### B3. Controller에서 Entity → Result 변환 금지
- 검사: `*.controller.ts`에서 Entity 타입을 직접 다루는 코드 (Entity import 후 필드 접근, `new XxxResponseDto(entity)` 등)
- **필수**: Service가 Result 반환 → Controller는 `ResponseDto.buildFrom*Result(result)` 호출만

### B4. Controller는 Command/Query 생성 → service 호출 → ResponseDto.buildFrom*Result 패턴
- 검사: Controller 메서드에서 다음 4단계 누락 여부
  1. `@Body()` DTO 수신
  2. Command/Query 객체 생성 (interface)
  3. `await this.service.method(command)` 호출
  4. `ResponseDto.buildFromCommandResult(result)` 또는 `buildFromQueryResult(result)` 반환
- 위반 예: Controller에서 직접 repository 접근, Service가 ResponseDto를 반환

### B5. Command/Query/Result는 `types/{module}.types.ts`에 interface로 정의
- 검사: Service 파일 내부에 `interface ...Command`/`...Query`/`...Result`가 정의된 경우 (types/로 이동 필요)
- 검사: `class XxxCommand` (interface가 아닌 class)

### B6. Command 핸들러(데이터 변경)는 `@Transactional()` 사용
- 검사: Service의 메서드가 `repository.save/insert/update/delete/remove`를 2회 이상 호출하면서 `@Transactional()` 데코레이터가 없는 경우
- 출처: CLAUDE.md "Command 핸들러는 반드시 @Transactional() 사용"

## 수정 가이드

- **B1**: Service에서 DTO import 제거 → Command/Query interface로 시그니처 교체. Controller에서 DTO → Command 변환.
- **B2**: generated 타입 사용 부분을 Result interface 필드로 옮김.
- **B3**: Controller의 Entity 변환 로직을 Service 내부 `toCommandResult/toQueryResult` private 메서드로 이동.
- **B4**: 누락된 단계 추가. Service 시그니처가 DTO를 받으면 Command interface로 변경.
- **B5**: Service 내부 interface를 `types/{module}.types.ts`로 이동, class는 interface로 변환.
- **B6**: 다중 쓰기 메서드에 `@Transactional()` 추가 (typeorm-transactional).

## 검사 명령 힌트

```bash
# B1
grep -rn "from '@/.*\.dto'" backend/src --include="*.service.ts"
# B2
grep -rn "@/api/generated" backend/src --include="*.service.ts"
# B5
grep -rn "interface .*\(Command\|Query\|Result\)" backend/src --include="*.service.ts"
```
