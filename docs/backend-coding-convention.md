# Backend Coding Convention

> gridi-app/backend NestJS 프로젝트의 코딩 컨벤션 정리 문서

## 1. 프로젝트 스택

| 항목 | 기술 |
|------|------|
| Framework | NestJS 11 |
| Language | TypeScript 5 (ES2023 target) |
| ORM | TypeORM 0.3 |
| Database | MariaDB |
| Queue | BullMQ + Redis |
| Auth | Passport (JWT, Google, GitHub, Notion, Figma OAuth) |
| Validation | class-validator + class-transformer |
| API Docs | Swagger (@nestjs/swagger) |
| Logging | nestjs-pino (Loki 연동) |
| Formatter | Prettier (singleQuote, trailingComma: all) |
| Linter | ESLint (typescript-eslint, no-relative-import-paths) |

---

## 2. 디렉토리 구조

### 모듈 표준 구조

```
backend/src/{module}/
├── {module}.controller.ts        # HTTP 요청 처리, Command 생성 → Service 호출 → Response 빌드
├── {module}.service.ts           # 비즈니스 로직, Command/Query → Result 반환
├── {module}.module.ts            # NestJS 모듈 정의
├── dto/
│   ├── create{Entity}.dto.ts     # Request DTO + Response DTO
│   ├── update{Entity}.dto.ts
│   └── get{Entity}.dto.ts
├── entities/
│   └── {entity}.entity.ts        # TypeORM 엔티티
├── types/
│   └── {module}.types.ts         # Command/Query/Result 인터페이스
├── guards/                       # (선택) 인증/인가 가드
├── strategies/                   # (선택) Passport 전략
└── services/                     # (선택) 도메인별 세분화 서비스
```

### 네이밍 규칙

| 대상 | 규칙 | 예시 |
|------|------|------|
| **파일명** | camelCase | `explorationProjects.service.ts` |
| **디렉토리명** | camelCase | `browserSessions/`, `figmaExport/` |
| **컨트롤러** | `{module}.controller.ts` | `auth.controller.ts` |
| **서비스** | `{module}.service.ts` | `users.service.ts` |
| **DTO** | `{verb}{Entity}.dto.ts` | `createUser.dto.ts` |
| **엔티티** | `{entity}.entity.ts` | `user.entity.ts` |
| **가드** | `{type}Auth.guard.ts` / `{type}Role.guard.ts` | `jwtAuth.guard.ts` |
| **전략** | `{provider}.strategy.ts` | `google.strategy.ts` |
| **타입** | `{module}.types.ts` | `auth.types.ts` |
| **상수** | `{topic}.ts` (camelCase 파일, UPPER_SNAKE_CASE 변수) | `errorMessages.ts` |

> **금지**: kebab-case 파일명/디렉토리명 (`shared-links/` → `sharedLinks/`)

---

## 3. Import 규칙

### 절대경로 필수 (`@/`)

```typescript
// GOOD
import { User } from '@/users/entities/user.entity';
import { COMMON_ERRORS } from '@/constants/errorMessages';

// BAD - 상대경로 금지
import { User } from '../../users/entities/user.entity';
import { User } from './entities/user.entity';
```

- `@/`는 `src/`에 매핑 (tsconfig paths + ESLint 규칙으로 강제)
- 유일한 예외: auto-generated 파일 (`api/generated/index.ts`)

### Import 순서

```typescript
// 1. NestJS / 외부 패키지
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

// 2. 내부 서비스/모듈
import { UsersService } from '@/users/users.service';

// 3. 엔티티
import { User } from '@/users/entities/user.entity';

// 4. 상수/타입
import { COMMON_ERRORS } from '@/constants/errorMessages';
import type { CreateUserCommand } from '@/users/types/users.types';
```

---

## 4. 계층형 아키텍처 (Command/Query/Result 패턴)

### 데이터 흐름

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
  ResponseDto.buildFromCommandResult(result)
  ↓
  HTTP Response (implements generated interface)
```

### 핵심 규칙

| 규칙 | 설명 |
|------|------|
| Service는 DTO를 import하지 않음 | 계층 격리 - Service는 Command/Query interface만 수신 |
| Service는 Entity→Result 변환 담당 | Controller에서 Entity 직접 접근 금지 |
| Response DTO에 static builder 필수 | `buildFromCommandResult()` 또는 `buildFromQueryResult()` |
| Response DTO는 generated interface implements | OpenAPI 정합성 보장 |
| Command/Query/Result는 interface로 정의 | `types/{module}.types.ts`에 위치 |

### Controller 패턴

```typescript
@ApiTags('Users')
@Controller('users')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get('me')
  @ApiOperation({ summary: '내 정보 조회' })
  async getMe(@Request() req: AuthenticatedRequest): Promise<GetUserResponseDto | null> {
    const query: GetUserQuery = { userId: req.user.id };
    const result = await this.usersService.getMe(query);
    if (!result) return null;
    return GetUserResponseDto.buildFromQueryResult(result);
  }
}
```

### Service 패턴

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly repository: Repository<User>,
  ) {}

  async create(command: CreateUserCommand): Promise<UserQueryResult> {
    const { email, password, name } = command;
    // 비즈니스 로직...
    const user = this.repository.create({ id: uuidv4(), email, name });
    const saved = await this.repository.save(user);
    return this.toQueryResult(saved);
  }

  private toQueryResult(user: User): UserQueryResult {
    return { id: user.id, email: user.email, name: user.name, ... };
  }
}
```

### 메소드 파라미터 규칙 (2개 이상 → 객체)

> 파라미터가 2개 이상인 메소드는 반드시 객체로 감싸서 전달한다.

```typescript
// GOOD - 객체로 감싸서 전달
async uploadScreenshot(params: UploadScreenshotParams): Promise<string> {
  const { projectId, nodeId, buffer } = params;
  // ...
}

// BAD - 개별 파라미터 나열 금지
async uploadScreenshot(projectId: string, nodeId: string, buffer: Buffer): Promise<string> {
  // ...
}
```

**적용 범위**:
- public / private 메소드 모두 적용
- Service, Controller 메소드 모두 적용

**예외**:
- 생성자(Constructor) DI 파라미터
- NestJS 데코레이터 기반 Controller 메소드 (`@Request()`, `@Body()`, `@Param()` 등)
- 프레임워크 인터페이스 오버라이드 (`validate()`, `canActivate()` 등)
- 파라미터가 1개인 메소드

**파라미터 객체 정의 위치**:
- public 메소드: `types/{module}.types.ts`에 인터페이스로 정의
- private 메소드: 같은 서비스 파일 상단 또는 `types/{module}.types.ts`에 정의

**네이밍**: `{MethodName}Params` 또는 Command/Query 패턴과 통합

```typescript
// types/users.types.ts
export interface FindByProviderParams {
  provider: AuthProvider;
  providerId: string;
}

// users.service.ts
async findByProvider(params: FindByProviderParams): Promise<User | null> {
  const { provider, providerId } = params;
  return this.repository.findOne({ where: { provider, providerId } });
}
```

### 코드 스타일 원칙

#### `const` 우선, `let` 지양

```typescript
// GOOD
const user = await this.repository.findOne({ where: { id } });
const name = user?.name ?? 'Unknown';

// BAD - 재할당이 불필요한 곳에 let 사용
let user = await this.repository.findOne({ where: { id } });
let name = 'Unknown';
if (user) {
  name = user.name;
}
```

- 변수 선언은 `const`가 기본
- `let`은 루프 카운터, 누적값 등 재할당이 **불가피한** 경우에만 사용

#### `try-catch` 최소화

```typescript
// GOOD - NestJS 예외를 그대로 전파, 글로벌 필터가 처리
async findOne(query: GetProjectQuery): Promise<ProjectQueryResult> {
  const project = await this.repository.findOne({ where: { id: query.projectId } });
  if (!project) throw new NotFoundException(COMMON_ERRORS.PROJECT_NOT_FOUND);
  return this.toQueryResult(project);
}

// BAD - 불필요한 try-catch 래핑
async findOne(query: GetProjectQuery): Promise<ProjectQueryResult> {
  try {
    const project = await this.repository.findOne({ where: { id: query.projectId } });
    if (!project) throw new NotFoundException(COMMON_ERRORS.PROJECT_NOT_FOUND);
    return this.toQueryResult(project);
  } catch (error) {
    throw error; // 의미 없는 재throw
  }
}
```

- `try-catch`는 외부 API 호출, 파일 I/O 등 **예측 불가능한 실패를 처리**할 때만 사용
- NestJS 예외(`NotFoundException` 등)는 글로벌 필터가 처리하므로 감싸지 않음
- catch 후 단순 재throw(`throw error`)는 금지

#### Early Return으로 `if-else` 깊이 최소화

```typescript
// GOOD - early return으로 depth 낮게
async update(command: UpdateProjectCommand): Promise<ProjectCommandResult> {
  const project = await this.repository.findOne({ where: { id: command.projectId } });
  if (!project) throw new NotFoundException(COMMON_ERRORS.PROJECT_NOT_FOUND);
  if (!command.name) throw new BadRequestException(ERRORS.NAME_REQUIRED);

  project.name = command.name;
  const saved = await this.repository.save(project);
  return this.toCommandResult(saved);
}

// BAD - if-else 중첩으로 depth 깊어짐
async update(command: UpdateProjectCommand): Promise<ProjectCommandResult> {
  const project = await this.repository.findOne({ where: { id: command.projectId } });
  if (project) {
    if (command.name) {
      project.name = command.name;
      const saved = await this.repository.save(project);
      return this.toCommandResult(saved);
    } else {
      throw new BadRequestException(ERRORS.NAME_REQUIRED);
    }
  } else {
    throw new NotFoundException(COMMON_ERRORS.PROJECT_NOT_FOUND);
  }
}
```

- 실패 조건을 먼저 검사하고 `throw` / `return`으로 빠져나감 (Guard Clause 패턴)
- `else`는 가능한 쓰지 않음 — early return으로 대체
- 중첩 depth는 최대 2단계 이내 유지

---

## 5. DTO 규칙

### 네이밍

| 타입 | 패턴 | 예시 |
|------|------|------|
| Request | `{Verb}{Entity}RequestDto` | `CreateExplorationProjectRequestDto` |
| Response (생성) | `Create{Entity}ResponseDto` | `CreateExplorationProjectResponseDto` |
| Response (수정) | `Update{Entity}ResponseDto` | `UpdateUserResponseDto` |
| Response (단일조회) | `Get{Entity}DetailResponseDto` | `GetExplorationProjectDetailResponseDto` |
| Response (목록조회) | `Get{Entity}ListResponseDto` | `GetExplorationProjectListResponseDto` |
| Response (상태) | `Get{Entity}StatusResponseDto` | `GetBrowserSessionStatusResponseDto` |

### 검증 데코레이터

```typescript
export class SignupRequestDto {
  @ApiProperty({ example: 'user@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'Password123!' })
  @IsString()
  @MinLength(8)
  @MaxLength(100)
  password: string;
}
```

- 모든 필드에 `@ApiProperty({ example: ... })` 필수
- class-validator 데코레이터 사용 (`@IsString`, `@IsEmail`, `@MinLength` 등)
- `@IsEnum()` 금지 → `@IsIn(Object.values(CONST_OBJ))` 사용

### Response DTO Builder 패턴

```typescript
export class GetUserResponseDto implements IGetUserResponseDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  email: string;

  static buildFromQueryResult(result: UserQueryResult): GetUserResponseDto {
    const dto = new GetUserResponseDto();
    dto.id = result.id;
    dto.email = result.email;
    return dto;
  }
}
```

---

## 6. 타입/인터페이스 규칙

### 위치

`types/{module}.types.ts`에 모든 Command/Query/Result 인터페이스 정의

### 네이밍

| 타입 | 패턴 | 예시 |
|------|------|------|
| Command | `{Verb}{Entity}Command` | `CreateExplorationProjectCommand` |
| Query | `Get{Entity}Query` | `GetExplorationProjectQuery` |
| CommandResult | `{Verb}{Entity}CommandResult` | `CreateExplorationProjectCommandResult` |
| QueryResult | `Get{Entity}QueryResult` | `GetExplorationProjectQueryResult` |

### 열거형 (as const 패턴)

```typescript
// GOOD
export const STATUS = {
  ACTIVE: 'active',
  INACTIVE: 'inactive',
} as const;
export type Status = (typeof STATUS)[keyof typeof STATUS];

// BAD - TypeScript enum 금지
enum Status { ACTIVE = 'active', INACTIVE = 'inactive' }
```

- 상수 객체명: `UPPER_SNAKE_CASE` (`AUTH_PROVIDER`, `NODE_STATUS`)
- DB 컬럼: `type: 'varchar'` 사용 (`type: 'enum'` 금지)

---

## 7. 엔티티 규칙

### PK

- UUID 문자열 (`@PrimaryColumn({ type: 'varchar', length: 120 })`)
- 서비스에서 `uuidv4()`로 생성
- `@PrimaryGeneratedColumn()` 금지

### 컬럼 네이밍

| 필드 | 패턴 |
|------|------|
| PK | `id: string` |
| FK | `{entity}Id: string` (예: `userId`, `projectId`) |
| 타임스탬프 | `@CreateDateColumn() createdAt` / `@UpdateDateColumn() updatedAt` |
| 문자열 | 항상 `length` 지정 |
| Nullable | 항상 `nullable: true` 명시 + TypeScript `| null` 타입 |

### 컬럼 타입

```typescript
// GOOD - typed 컬럼
@Column({ type: 'varchar', length: 20, default: NODE_STATUS.PENDING })
status: NodeStatus;

// BAD - simple-json 금지 (시스템 데이터용)
@Column({ type: 'simple-json' })
config: Record<string, unknown>;
```

> `simple-json`은 사용자 비정형 메타데이터에만 허용 (태그, 메모 등)

### 관계

```typescript
@ManyToOne(() => ExplorationProject, { onDelete: 'CASCADE' })
@JoinColumn()
project: ExplorationProject;
```

---

## 8. 에러 처리

### 중앙화된 에러 메시지

모든 에러 메시지는 `constants/errorMessages.ts`에서 관리:

```typescript
export const COMMON_ERRORS = {
  PROJECT_NOT_FOUND: 'Project not found',
  USER_NOT_FOUND: 'User not found',
} as const;
```

### NestJS 예외 사용

| 상황 | 예외 타입 |
|------|-----------|
| 유효성 검증 실패 | `BadRequestException` |
| 인증 실패 | `UnauthorizedException` |
| 권한 부족 | `ForbiddenException` |
| 리소스 미존재 | `NotFoundException` |
| 중복/충돌 | `ConflictException` |
| 서버 오류 | `InternalServerErrorException` |

```typescript
// GOOD
throw new NotFoundException(COMMON_ERRORS.PROJECT_NOT_FOUND);

// BAD - 하드코딩 금지
throw new NotFoundException('Project not found');
```

---

## 9. 인증/인가

### Guard 패턴

```typescript
// JWT 인증 (대부분의 엔드포인트)
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()

// OAuth 인증
@UseGuards(GoogleAuthGuard)

// 관리자 권한
@UseGuards(JwtAuthGuard, AdminRoleGuard)
```

### AuthenticatedRequest

```typescript
// common/types/authenticatedRequest.ts
export interface AuthenticatedRequest extends Request {
  user: { id: string; email: string };
}

// Controller에서 사용
async getMe(@Request() req: AuthenticatedRequest): Promise<...> {
  const userId = req.user.id;
}
```

### 리소스 소유권 검증

```typescript
// ProjectAuthorizerService 사용
await this.authorizer.authorizeOwner({ projectId, userId: req.user.id });
await this.authorizer.authorizeViewer({ projectId, userId: req.user.id, shareToken });
```

---

## 10. Module 패턴

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User, UserAuthProvider]),
    SubscriptionModule,
  ],
  controllers: [UsersController, UserAuthProviderController],
  providers: [UsersService, UserAuthProviderService, CryptoService],
  exports: [UsersService, UserAuthProviderService],
})
export class UsersModule {}
```

- `TypeOrmModule.forFeature([...])` 로 엔티티 등록
- `exports`로 다른 모듈에서 사용할 서비스 공개
- 비동기 설정: `registerAsync` + `ConfigService` 주입

---

## 11. Swagger 문서화

```typescript
@ApiTags('Exploration Projects')
@Controller('exploration-projects')
export class ExplorationProjectsController {

  @Post()
  @ApiOperation({ summary: '새 탐색 프로젝트 생성' })
  @ApiResponse({ status: 201, description: '탐색 프로젝트 생성' })
  async create(...): Promise<CreateExplorationProjectResponseDto> { ... }
}
```

- 모든 Controller에 `@ApiTags()` 필수
- 모든 엔드포인트에 `@ApiOperation({ summary })` 필수
- 인증 필요 시 `@ApiBearerAuth()` 추가

---

## 12. Contract-First API 개발 (OpenAPI Generated 타입)

### 개요

이 프로젝트는 **Contract-First** 방식으로 API를 개발한다. OpenAPI 스펙(`docs/openapi.yaml`)이 단일 진실 원천(Single Source of Truth)이며, 백엔드와 프론트엔드 모두 이 스펙에서 자동 생성된 타입/클라이언트를 사용한다.

```
docs/openapi.yaml (API 계약)
  ├── npm run api:generate:be → backend/src/api/generated/schema.d.ts     (openapi-typescript)
  └── npm run api:generate:fe → frontend/src/api/generated/              (openapi-generator-cli)
                                  ├── api/        (API 클래스)
                                  └── models/     (DTO 타입)
```

### 코드 생성 명령어

```bash
npm run api:generate       # BE + FE 동시 생성
npm run api:generate:be    # 백엔드만 (schema.d.ts)
npm run api:generate:fe    # 프론트엔드만 (api/ + models/)
```

> **⚠️ generated 파일은 절대 수동 편집 금지** — 스펙 변경 후 재생성만 가능

### 백엔드: Generated 타입 활용

#### 타입 import 및 별칭

```typescript
import type { components } from '@/api/generated/schema';

// OpenAPI 스키마에서 타입 별칭 생성
type ILoginResponseDto = components['schemas']['LoginResponseDto'];
type IGetUserDetailResponseDto = components['schemas']['GetAdminUserDetailResponseDto'];
```

#### Response DTO에서 implements 필수

```typescript
export class LoginResponseDto implements ILoginResponseDto {
  accessToken: string;
  refreshToken: string;
  user: { id: string; email: string; /* ... */ };

  static buildFromCommandResult(result: AuthCommandResult): LoginResponseDto {
    const response = new LoginResponseDto();
    response.accessToken = result.accessToken;
    response.refreshToken = result.refreshToken;
    response.user = {
      id: result.user.id,
      email: result.user.email,
      createdAt: result.user.createdAt.toISOString(), // Date → ISO String
      // ...
    };
    return response;
  }
}
```

`implements`를 사용하면 OpenAPI 스펙과 Response DTO의 필드가 불일치할 때 **컴파일 타임에 에러**가 발생하여 정합성을 보장한다.

#### 핵심 규칙

| 규칙 | 설명 |
|------|------|
| **Response DTO는 `implements` 필수** | `components['schemas'][...]` 타입을 implements |
| **Request DTO는 `implements` 선택** | class-validator 데코레이터가 주 검증이므로 선택적 |
| **Service는 generated 타입 import 금지** | 계층 격리 — Service는 Command/Query/Result만 사용 |
| **Date → ISO String 변환** | Entity의 `Date` 객체는 `.toISOString()`으로 직렬화 |
| **타입 별칭 네이밍** | `I{SchemaName}` 접두사 (예: `ILoginResponseDto`) |

### 프론트엔드: Generated API 클라이언트 활용

#### 클라이언트 구조

```
frontend/src/api/
├── generated/                # 자동 생성 (수동 편집 금지)
│   ├── api/*.ts             # API 메서드 클래스 (AuthApi, UsersApi 등)
│   └── models/*.ts          # DTO 타입 (Request/Response)
└── client.ts                # API 인스턴스화 + 인터셉터 설정
```

#### client.ts: API 인스턴스 export

```typescript
// frontend/src/api/client.ts
import { AuthApi, UsersApi, ExplorationProjectsApi } from '@/api/generated/api';

export const authApiClient = new AuthApi(configuration, API_BASE_URL, apiAxios);
export const usersApiClient = new UsersApi(configuration, API_BASE_URL, apiAxios);
export const explorationProjectsApiClient = new ExplorationProjectsApi(/* ... */);

// 모델 타입도 re-export
export * from '@/api/generated/models';
```

#### 컴포넌트에서 사용

```typescript
// GOOD — Generated 클라이언트 사용
import { explorationProjectsApiClient } from '@/api/client';
const response = await explorationProjectsApiClient.getExplorationProject({ id });
const project = response.data;

// BAD — 직접 axios 호출 금지
import { apiAxios } from '@/api/client';
const { data } = await apiAxios.get(`/exploration-projects/${id}`);
```

#### lib/api.ts 래퍼 (선택)

반복적인 `response.data` 추출을 간소화하기 위해 래퍼 함수를 사용할 수 있다:

```typescript
// frontend/src/lib/api.ts
export const explorationProjectsApi = {
  getAll: async () => {
    const response = await explorationProjectsApiClient.getExplorationProjects();
    return response.data;
  },
};
```

### 개발 워크플로우

```
1. docs/openapi.yaml 수정 (새 엔드포인트/스키마 추가)
2. npm run api:generate (BE + FE 코드 재생성)
3. 백엔드: Response DTO 작성 (implements generated 타입)
4. 백엔드: Controller/Service 구현
5. 프론트엔드: generated API 클라이언트로 호출
6. CI에서 타입 정합성 검증 (tsc --noEmit)
```

> **OpenAPI 스펙을 먼저 수정하고, 코드를 나중에 맞추는 것이 원칙이다.**

---

## 부록: 발견된 위반 사항

### V1. kebab-case 디렉토리명 (camelCase 위반)

| 현재 | 변경 필요 |
|------|-----------|
| `exploration-projects/` | `explorationProjects/` |
| `shared-links/` | `sharedLinks/` |
| `figma-export/` | `figmaExport/` |
| `browser-sessions/` | `browserSessions/` |

### V2. kebab-case 파일명 (camelCase 위반)

| 현재 | 변경 필요 |
|------|-----------|
| `shared-links.service.ts` | `sharedLinks.service.ts` |
| `shared-links.controller.ts` | `sharedLinks.controller.ts` |
| `shared-links.module.ts` | `sharedLinks.module.ts` |
| `shared-link.entity.ts` | `sharedLink.entity.ts` |
| `data-source.ts` | `dataSource.ts` |

### V3. `simple-json` 시스템 데이터 사용

`explorationNode.entity.ts:90-94`:
```typescript
@Column({ type: 'simple-json', nullable: true })
policies: Record<string, unknown> | null;

@Column({ type: 'simple-json', nullable: true })
uiElements: Record<string, unknown>[] | null;
```

### V4. `@PrimaryGeneratedColumn` 사용 (UUID PrimaryColumn 위반)

`sharedLinkAccessLog.entity.ts:14`:
```typescript
@PrimaryGeneratedColumn('uuid')  // 금지 패턴
id: string;
```

### V5. `api/generated/index.ts` 상대경로 import

`api/generated/index.ts:10`:
```typescript
export type { components, operations, paths } from './schema';
```

> auto-generated 파일이므로 예외 처리 가능

### V6. 메소드 파라미터 2개 이상 개별 전달 (객체 미사용)

**17개 서비스 파일, 총 약 40+ 메소드에서 위반 발견**

주요 위반 (파라미터 3개 이상 - 우선 리팩토링 대상):

| 파일 | 메소드 | 파라미터 수 |
|------|--------|-------------|
| `auth.service.ts` | `validateOAuthUser(email, name, provider, providerId, avatarUrl?)` | 5 |
| `auth.service.ts` | `storeFigmaTokenOnLogin(userId, figmaAccessToken, figmaRefreshToken?, expiresIn?)` | 4 |
| `users.service.ts` | `createOAuthUser(email, name, provider, providerId, avatarUrl?)` | 5 |
| `userConnection.service.ts` | `upsertConnection(userId, provider, providerId, providerEmail, displayName, avatarUrl)` | 6 |
| `screenshotUploader.service.ts` | `uploadScreenshot(projectId, nodeId, buffer)` 외 4개 | 3 |
| `screenshotUploader.service.ts` | `uploadExportFile(projectId, filename, content, contentType)` | 4 |
| `s3Storage.service.ts` | `upload(key, body, contentType)` | 3 |
| `s3Storage.service.ts` | `getSignedUrl(s3DirectUrl, expiresIn, responseContentDisposition?)` | 3 |
| `branchAnalyzer.service.ts` | `analyzeNewScreen(parentNode, featureName, description, ...)` | 7 |
| `interactiveCrawler.service.ts` | `runActionLoop(page, node, goal, explorationId, projectId, baseUrl)` | 6 |
| `llmConfig.service.ts` | `getFeatureConfig(feature, projectProvider?, projectModel?)` | 3 |
| `llmConfig.service.ts` | `updateFeatureConfig(feature, provider, model)` | 3 |
| `projectAuthorizer.service.ts` | `authorizeByShareToken(projectId, userId, shareToken)` | 3 |
| `projectAuthorizer.service.ts` | `logAccess(sharedLinkId, userId, accessType)` | 3 |

파라미터 2개 위반:

| 파일 | 메소드 |
|------|--------|
| `auth.service.ts` | `validateUser(email, password)` |
| `users.service.ts` | `findByProvider(provider, providerId)` |
| `userConnection.service.ts` | `buildConnectUrl(userId, provider)` |
| `userConnection.service.ts` | `findByUserAndProvider(userId, provider)` |
| `userConnection.service.ts` | `findByProviderAndProviderId(provider, providerId)` |
| `payment.service.ts` | `getConfigValue(key, fallback)` |
| `subscription.service.ts` | `isFeatureEnabled(userId, feature)` |
| `explorationState.service.ts` | `emitSSE(explorationId, event)` |
| `figmaClient.service.ts` | `buildAuthHeaders(token, tokenType)` |
| `figmaClient.service.ts` | `verifyToken(token, tokenType)` |
| `sketchFileStorage.service.ts` | `getDownloadUrl(s3DirectUrl, filename)` |
| `puppeteer.service.ts` | `connect(cdpWsUrl, poolSize)` |

### V7. `let` → `const` 전환 가능 케이스

`let` 사용 총 약 45건 중, 정당한 사용(루프 카운터, 누적값, regex match)을 제외한 리팩토링 대상:

| 파일 | 라인 | 변수 | 개선 방향 |
|------|------|------|-----------|
| `llmConfig.service.ts` | 56 | `let config = await ...findOne()` | `const`로 변경 후 분기 처리 분리 |
| `explorationState.service.ts` | 135 | `let subject = this.sseSubjects.get()` | `const` + nullish 처리로 대체 |
| `exploration.service.ts` | 143 | `let session: SessionStatusResult` | try-catch 구조 개선으로 `const` 전환 |
| `figmaExport.service.ts` | 99 | `let figmaUserInfo: FigmaUserInfo` | try-catch 구조 개선으로 `const` 전환 |
| `branchAnalyzer.service.ts` | 255, 269 | `let siblingSection`, `let sharedEdgeSection` | 함수 추출 또는 조건부 할당으로 `const` 전환 |
| `screenAnalyzer.service.ts` | 786-862 | `let serviceSection`, `let metaSection` 등 7건 | 각 섹션 빌더를 함수로 추출하여 `const` 전환 |

> 루프 카운터(`scrollCount`, `successCount`), 누적값(`yOffset`, `remaining`), regex match(`let match`) 등은 `let`이 정당함

### V8. `if-else` → early return 전환 대상

| 파일 | 라인 | 개선 방향 |
|------|------|-----------|
| `steelClient.service.ts` | 26 | early return으로 `else` 제거 |
| `browserSessions.service.ts` | 70 | guard clause로 전환 |
| `anthropicProvider.service.ts` | 58 | early return |
| `puppeteer.service.ts` | 79, 131, 485 | guard clause로 전환 |
| `llmConfig.service.ts` | 68 | early return |
| `htmlPreprocessor.service.ts` | 521 | early return |
| `exploration.service.ts` | 421 | early return |
| `screenGenerator.service.ts` | 385, 418 | early return |
| `htmlToFigmaExtractor.service.ts` | 123, 175 | early return |
| `sketchBuilder.service.ts` | 480 | early return |
| `auth.service.ts` | 108 | early return |

### V9. 불필요한 `try-catch` 제거 검토 대상

`try-catch` 총 약 47건 중, 외부 I/O(S3, Puppeteer, LLM API, Figma API, Steel API)는 정당한 사용.

검토 대상:

| 파일 | 라인 | 사유 |
|------|------|------|
| `llmConfig.service.ts` | 104 | DB 조회 실패 시 catch — TypeORM 에러는 글로벌 필터 위임 가능 |
| `exploration.service.ts` | 149 | 세션 상태 조회 — 내부 서비스 호출이므로 불필요할 수 있음 |
| `exploration.service.ts` | 305 | 노드 처리 중 catch — 개별 에러 핸들링 필요 여부 검토 |

> S3 업로드, Puppeteer 조작, 외부 API 호출 등은 `try-catch` 유지 (예측 불가능한 실패)
