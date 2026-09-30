# Go Service Code Base Structure

Tài liệu này chốt hướng thiết kế ban đầu cho service Go theo mô hình modular monolith, có thể mở rộng dần sang Redis, Kafka và background worker.

## Tech stack đã chốt

### Core

| Thành phần | Lựa chọn | Mục đích |
| --- | --- | --- |
| Language | Go | Ngôn ngữ chính |
| HTTP framework | `github.com/gin-gonic/gin` | Router, middleware, binding request và HTTP response |
| Database | PostgreSQL | Database chính cho user, order, ticket |
| SQL access | `github.com/jmoiron/sqlx` | Mở rộng từ `database/sql`, hỗ trợ `Get`, `Select`, named query và scan vào struct |
| PostgreSQL driver | `github.com/jackc/pgx/v5/stdlib` | Driver PostgreSQL cho `database/sql`/`sqlx` |
| Migration | Tự viết DDL | Team chủ động quản lý schema; không dùng auto migration từ entity |
| Configuration | Environment variables | Tách config khỏi source code |
| Logging | `log/slog` | Logger chuẩn của Go, structured logging |
| Testing | `testing` + `httptest` | Unit test và HTTP handler test |

### Authentication

| Thành phần | Lựa chọn | Mục đích |
| --- | --- | --- |
| Password hashing | `golang.org/x/crypto/bcrypt` | Hash và verify password |
| Token | `github.com/golang-jwt/jwt/v5` | Access token/refresh token nếu dùng JWT |
| Validation | Validator tích hợp với Gin binding | Validate request DTO |

### Thành phần mở rộng theo phase

| Thành phần | Lựa chọn | Dùng khi |
| --- | --- | --- |
| Cache | `github.com/redis/go-redis/v9` | Cần cache, rate limit hoặc lưu session/token |
| Queue | `github.com/hibiken/asynq` | Cần background job dựa trên Redis |
| Event streaming | `github.com/segmentio/kafka-go` hoặc `franz-go` | Cần Kafka và event-driven processing |
| API documentation | `swaggo/swag` hoặc OpenAPI | Cần Swagger/OpenAPI |
| Local hot reload | `air-verse/air` | Chỉ dùng trong development |
| Lint | `golangci-lint` | Static analysis và kiểm tra coding convention |

Không đưa Redis, Kafka hoặc queue vào phase đầu nếu auth/user chưa ổn định. Trước hết nên hoàn thiện boundary và interface để thêm các thành phần này mà không phải sửa business logic.

## Vì sao dùng sqlx thay vì ORM?

`sqlx` không phải ORM. Nó là một lớp tiện ích trên `database/sql`:

- Viết SQL trực tiếp, kiểm soát rõ query và index.
- Scan kết quả vào struct bằng `db` tag.
- Hỗ trợ `Get`, `Select`, `NamedExec`, transaction.
- Không tự sinh migration từ entity.
- Không có lazy loading, relationship magic hoặc query behavior khó đoán như một số ORM.

Ví dụ convention:

```go
type User struct {
    ID    uuid.UUID `db:"id" json:"id"`
    Email string    `db:"email" json:"email"`
}
```

Repository sẽ sở hữu SQL và database mapping. Application service không được biết SQL cụ thể.

```text
handler -> application service -> repository interface -> postgres repository -> sqlx -> PostgreSQL
```

## Migration

Migration được viết thủ công bằng DDL. Có thể dùng một migration runner bên ngoài, nhưng nội dung schema vẫn do developer kiểm soát.

```text
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_create_orders.up.sql
└── 000002_create_orders.down.sql
```

Quy ước:

- Mỗi thay đổi schema tạo một version mới.
- Không sửa migration đã chạy trên môi trường shared/production.
- Migration `up` dùng để apply thay đổi.
- Migration `down` dùng để rollback trong local/test khi phù hợp.
- Luôn review index, constraint, foreign key và backward compatibility.
- Không dùng `AutoMigrate` trong production.

## Cấu trúc tổng thể

```text
root/
├── cmd/
│   ├── root.go
│   ├── http.go
│   └── kafka.go
│
├── docker/
│   ├── api.Dockerfile
│   └── db/
│       └── initdb/
│           └── postgres-init.sh
│
├── migrations/
│   ├── 000001_create_users.up.sql
│   ├── 000001_create_users.down.sql
│   ├── 000002_create_orders.up.sql
│   └── 000002_create_orders.down.sql
│
├── src/
│   ├── bootstrap/
│   │   ├── http.go
│   │   └── kafka.go
│   │
│   ├── applications/
│   │   ├── commands/
│   │   ├── queries/
│   │   ├── dto/
│   │   ├── listeners/
│   │   └── services/
│   │
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── user.go
│   │   │   └── order.go
│   │   ├── valueobjects/
│   │   │   ├── email.go
│   │   │   ├── money.go
│   │   │   └── timestamp.go
│   │   ├── repositories/
│   │   │   ├── user_repository.go
│   │   │   └── order_repository.go
│   │   ├── events/
│   │   │   ├── order_created.go
│   │   │   └── user_registered.go
│   │   └── errors/
│   │       ├── bad_request.go
│   │       └── internal_server_error.go
│   │
│   ├── infrastructure/
│   │   ├── config/
│   │   ├── constants/
│   │   ├── persistence/
│   │   │   ├── database.go
│   │   │   └── postgres/
│   │   │       ├── user_repository.go
│   │   │       └── order_repository.go
│   │   ├── messaging/
│   │   │   ├── kafka/
│   │   │   └── rabbitmq/
│   │   ├── external/
│   │   │   ├── aws/
│   │   │   └── ...
│   │   └── utils/
│   │
│   └── interfaces/
│       ├── rest/
│       │   ├── handlers/
│       │   ├── middleware/
│       │   └── router.go
│       ├── grpc/
│       └── kafka/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   │   ├── database/
│   │   └── interfaces/
│   └── mocks/
│
├── storage/
│   └── logs/
│
├── .env.example
├── .gitignore
├── Dockerfile
├── Makefile
├── go.mod
├── go.sum
├── main.go
└── README.md
```

## Giải thích từng thư mục và file

### Root files

| File | Công dụng |
| --- | --- |
| `main.go` | Entry point tối thiểu của application. Đọc command/root và gọi bootstrap phù hợp. Không chứa business logic. |
| `go.mod` | Khai báo module path, Go version và dependency trực tiếp. Tương đương một phần vai trò của `package.json`. |
| `go.sum` | Lưu checksum của dependency. File này được Go tự quản lý và nên commit vào Git. |
| `Makefile` | Chuẩn hóa các lệnh `run`, `test`, `lint`, `build`, migration và generate cho team. |
| `Dockerfile` | Build image production của service. Nếu dùng nhiều image riêng thì có thể dùng `docker/api.Dockerfile`. |
| `.env.example` | Danh sách các biến môi trường cần có, không chứa secret thật. |
| `.gitignore` | Loại trừ `.env`, binary, log, coverage và file local khỏi Git. |
| `README.md` | Hướng dẫn chạy project, setup database, test, build và các quy ước cơ bản. |

### `cmd/`

Chứa command entrypoint. Mỗi command có thể build/deploy thành một process riêng.

| File | Công dụng |
| --- | --- |
| `cmd/root.go` | Root command, khai báo flag/config dùng chung và các subcommand. |
| `cmd/http.go` | Command khởi động HTTP API bằng Gin. |
| `cmd/kafka.go` | Command khởi động Kafka consumer/producer process nếu hệ thống cần Kafka. |

HTTP API và Kafka worker nên là hai process/container riêng để scale độc lập. Không bắt buộc start cả hai trong cùng một process.

### `docker/`

Chứa các file phục vụ build và khởi tạo môi trường Docker.

| Path | Công dụng |
| --- | --- |
| `docker/api.Dockerfile` | Multi-stage build cho API binary. Dùng khi muốn tách Dockerfile khỏi root. |
| `docker/db/initdb/postgres-init.sh` | Script khởi tạo PostgreSQL local, tạo extension/database/user nếu cần. Không thay thế migration schema. |

`postgres-init.sh` chỉ nên xử lý initialization của container. Bảng và index vẫn do migration quản lý.

### `migrations/`

Chứa DDL versioned do developer tự viết.

| File | Công dụng |
| --- | --- |
| `*_create_users.up.sql` | Tạo hoặc thay đổi schema theo chiều tiến. |
| `*_create_users.down.sql` | Rollback migration trong local/test khi cần. |

Migration không đặt trong `src` vì đây là database lifecycle, không phải runtime business code.

### `src/bootstrap/`

Composition root của từng transport/process. Nơi wire dependency từ ngoài vào trong.

| File | Công dụng |
| --- | --- |
| `bootstrap/http.go` | Load config, mở PostgreSQL, tạo repository/service/handler, đăng ký Gin route và tạo HTTP server. |
| `bootstrap/kafka.go` | Tạo Kafka client, consumer, listener và dependency cần cho worker. |

Bootstrap được phép biết tất cả layer. Các layer bên trong không được import ngược bootstrap.

### `src/applications/`

Chứa use case của hệ thống. Đây là application layer, điều phối domain và repository.

| Folder | Công dụng |
| --- | --- |
| `commands/` | Use case làm thay đổi state: register user, update user, create order. |
| `queries/` | Use case đọc dữ liệu: get current user, list orders, search ticket. |
| `dto/` | Input/output model của application layer, không nhất thiết trùng entity database. |
| `listeners/` | Xử lý event/message từ Kafka hoặc queue rồi gọi application use case. |
| `services/` | Logic orchestration dùng chung giữa command/query khi chưa cần tách thành nhiều package. |

Không đặt Gin `Context`, SQL query hoặc Kafka implementation trực tiếp trong application service.

### `src/domain/`

Chứa business rule thuần, hạn chế dependency vào Gin, PostgreSQL, Kafka hay Redis.

| Folder | Công dụng |
| --- | --- |
| `entities/` | Object có identity và lifecycle, ví dụ `User`, `Order`. |
| `valueobjects/` | Giá trị có validation/invariant riêng, ví dụ `Email`, `Money`, `Timestamp`. |
| `repositories/` | Interface repository mà application cần. Implementation nằm ở infrastructure. |
| `events/` | Domain event, ví dụ `UserRegistered`, `OrderCreated`. |
| `errors/` | Domain/application error có thể map sang HTTP hoặc message error. |

Ví dụ dependency direction:

```text
domain <- applications <- interfaces
domain <- infrastructure
```

`domain` không được import `interfaces` hoặc `infrastructure`.

### `src/infrastructure/`

Chứa implementation phụ thuộc external technology.

| Folder/File | Công dụng |
| --- | --- |
| `config/` | Parse environment variables thành typed config. |
| `constants/` | Constant dùng xuyên infrastructure; constant business nên đặt gần domain. |
| `persistence/database.go` | Mở, cấu hình và đóng `*sql.DB`; thiết lập pool, timeout, ping. |
| `persistence/postgres/` | Implementation repository dùng `sqlx` và PostgreSQL. |
| `messaging/kafka/` | Kafka producer, consumer và adapter. |
| `messaging/rabbitmq/` | RabbitMQ adapter nếu sau này cần; không tạo nếu chưa dùng. |
| `external/aws/` | Adapter gọi S3, SES, SQS hoặc AWS service khác. |
| `utils/` | Chỉ chứa helper infrastructure thực sự dùng chung. Tránh biến thành nơi chứa code không có chủ sở hữu. |

### `src/interfaces/`

Chứa adapter nhận request từ bên ngoài và chuyển vào application layer.

| Folder/File | Công dụng |
| --- | --- |
| `rest/router.go` | Tạo Gin engine, đăng ký route group và middleware. |
| `rest/handlers/` | Parse request, bind/validate DTO, gọi use case và format HTTP response. |
| `rest/middleware/` | Authentication, authorization, request ID, recovery, logging, CORS, rate limit. |
| `grpc/` | gRPC server adapter nếu cần public/internal gRPC API. |
| `kafka/` | Adapter cho message transport; nhận message rồi gọi application listener. |

Handler không truy cập database trực tiếp. Handler chỉ nên làm transport concern.

### `tests/`

| Folder | Công dụng |
| --- | --- |
| `unit/` | Test domain/application không cần database thật. |
| `integration/database/` | Test repository với PostgreSQL thật hoặc container test database. |
| `integration/interfaces/` | Test API qua Gin router và HTTP request. |
| `mocks/` | Mock interface cho unit test. Có thể generate bằng mockery khi số lượng interface tăng. |

### `storage/logs/`

Chỉ nên dùng cho local hoặc môi trường cần ghi file. Production ưu tiên ghi structured log ra stdout để platform thu thập.

## Request lifecycle

```text
Client
  |
  v
Gin router
  |
  v
Middleware
  |
  v
REST handler
  |  bind/validate DTO
  v
Application command/query
  |  orchestrate use case
  v
Domain entity/rule
  |
  v
Repository interface
  |
  v
Postgres repository
  |  sqlx + SQL
  v
PostgreSQL
```

Response đi ngược lại theo chiều:

```text
PostgreSQL -> repository -> application -> handler -> HTTP response -> client
```

## Bootstrap lifecycle

```text
main.go
  -> cmd/root.go
  -> cmd/http.go
  -> load config
  -> open PostgreSQL connection pool
  -> create repositories
  -> create application services/use cases
  -> create Gin handlers
  -> register middleware and routes
  -> start HTTP server
```

## Initial auth/user scope

Phase đầu chỉ cần:

```text
POST /auth/register
POST /auth/login
POST /auth/refresh
POST /auth/logout
GET  /auth/me
PATCH /users/me
```

Các phần nên để phase sau:

- Admin user management.
- Role/permission matrix.
- Email verification.
- Forgot/reset password.
- Redis token revocation.
- Login rate limit.
- Audit log.

## Quy tắc dependency

- Handler chỉ phụ thuộc application interface.
- Application chỉ phụ thuộc domain và repository interface.
- Domain không phụ thuộc Gin, `sqlx`, PostgreSQL, Redis hoặc Kafka.
- PostgreSQL implementation chỉ nằm trong infrastructure.
- SQL không đặt trong handler/service.
- DTO HTTP không dùng trực tiếp làm domain entity.
- Không tạo package `common` hoặc `utils` để gom mọi thứ không rõ ownership.
- Chỉ thêm Kafka/Redis abstraction khi có use case thực tế.
