# Category D — Entity / Type / Enum

> 출처: `docs/backend-coding-convention.md` §6 타입/인터페이스, §7 엔티티

## 검사 규칙

### D1. UUID PK + `@PrimaryColumn`
- 필수: `@PrimaryColumn({ type: 'varchar', length: 120 })` `id: string`
- **금지**: `@PrimaryGeneratedColumn()`, auto-increment, number id
- 서비스에서 `uuidv4()`로 생성

### D2. DB enum 컬럼 금지
- 검사: `@Column({ type: 'enum', ... })` 사용 탐색
- **필수**: `@Column({ type: 'varchar', length: N })` + as const 상수

### D3. `simple-json` 컬럼 금지 (시스템 데이터)
- 검사: `type: 'simple-json'` 사용 탐색
- 예외: 사용자 비정형 메타데이터(태그, 메모) — 주석으로 명시되어 있는지 확인
- **필수**: 모든 설정값은 개별 typed 컬럼

### D4. 문자열 컬럼은 항상 `length` 지정
- 검사: `@Column({ type: 'varchar' ... })` 중 `length` 누락
- 검사: `@Column()` 단독 사용 (string 필드인 경우)

### D5. Nullable 컬럼은 `nullable: true` + TypeScript `| null`
- 검사: `nullable: true`인 컬럼의 TS 타입에 `| null` 누락
- 검사: TS 타입은 `| null`인데 `nullable: true` 누락

### D6. FK 네이밍은 `{entity}Id`
- 예: `userId`, `projectId`
- 검사: `_id`, `Id` 미접미사 (예: `user_fk`)

### D7. 타임스탬프는 `@CreateDateColumn`/`@UpdateDateColumn`
- 검사: `createdAt`/`updatedAt` 필드인데 일반 `@Column` 사용

### D8. TypeScript `enum` 키워드 금지
- 검사: `^enum |^export enum ` 패턴
- **필수**: `as const` 객체 + `typeof` 타입
- 상수 객체명: UPPER_SNAKE_CASE (예: `AUTH_PROVIDER`, `NODE_STATUS`)

### D9. Command/Query/Result interface는 `types/{module}.types.ts`에 위치
- 검사: `*.service.ts`, `*.controller.ts`, `*.dto.ts`에 정의된 `interface .*\(Command\|Query\|Result\)`
- (Category B와 일부 중복 — 여기서는 "위치" 관점에서만 검사)

### D10. 관계는 `@JoinColumn` + `onDelete` 명시
- 검사: `@ManyToOne`, `@OneToMany` 중 `onDelete` 누락 (best effort)

## 수정 가이드

- **D1**: `@PrimaryGeneratedColumn` → `@PrimaryColumn({ type: 'varchar', length: 120 })`. 서비스에서 `id: uuidv4()` 생성 추가. (마이그레이션 영향 → 사용자 보고)
- **D2**: `type: 'enum', enum: STATUS` → `type: 'varchar', length: N`. 상수 객체가 없으면 `constants/`에 생성.
- **D3**: 사용자 보고. 자동 변환은 위험 (스키마 변경 필요).
- **D4**: 적절한 length 추가 (예: 짧은 식별자 50, 일반 텍스트 255). 판단 어려우면 사용자 확인.
- **D5**: 누락된 쪽 추가.
- **D8**: `enum Foo { A='a', B='b' }` → `export const FOO = { A: 'a', B: 'b' } as const; export type Foo = typeof FOO[keyof typeof FOO];` 사용처 import 갱신.
- **D9**: interface를 `types/{module}.types.ts`로 이동.

## 검사 명령 힌트

```bash
# D1
grep -rn "@PrimaryGeneratedColumn" backend/src --include="*.entity.ts"
# D2
grep -rn "type: 'enum'" backend/src --include="*.entity.ts"
# D3
grep -rn "type: 'simple-json'" backend/src --include="*.entity.ts"
# D8
grep -rn "^export enum \|^enum " backend/src --include="*.ts"
```
