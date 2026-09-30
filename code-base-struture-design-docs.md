root/
├── cmd/
│   ├── http.go
│   ├── kafka.go
│   └── root.go
│
├── docker/
│   ├── api.Dockerfile
│   └── db/initdb/postgres-init.sh
│
├── migrations/
│   ├── 000001_create_users.up.sql
│   ├── 000001_create_users.down.sql
│   ├── 000002_create_orders.up.sql
│   └── 000002_create_orders.down.sql
│
├── src/
│   │
│   └── bootstrap/
│       ├── http.go
│       └── kafka.go
│   │
│   ├── applications/
│   │   ├── commands/
│   │   ├── queries/
│   │   ├── dto/
│   │   ├── listeners/
│   │   └── services/
│   │
│   │
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── user.go
│   │   │   └── order.go
│   │   │
│   │   ├── valueobjects/
│   │   │   ├── email.go
│   │   │   ├── money.go
│   │   │   └── timestamp.go
│   │   │
│   │   ├── repositories/
│   │   │   ├── user_repository.go
│   │   │   └── order_repository.go
│   │   │
│   │   ├── events/
│   │   │   ├── order_created.go
│   │   │   └── user_registered.go
│   │   │
│   │   └── errors/
│   │       ├── bad_request.go
│   │       └── internal_server_error.go
│   │
│   ├── infrastructure/
│   │   ├── config/
│   │   ├── constants/
│   │   │
│   │   ├── persistence/
│   │   │   ├── database.go
│   │   │   └── postgres/
│   │   │       ├── user_repository.go
│   │   │       └── order_repository.go
│   │   │
│   │   ├── messaging/
│   │   │   ├── kafka/
│   │   │   └── rabbitmq/
│   │   │
│   │   ├── external/
│   │   │   ├── aws/
│   │   │   └── ...
│   │   │
│   │   └── utils/
│   │
│   ├── interfaces/
│       ├── rest/
│       │   ├── handlers/
│       │   ├── middleware/
│       │   └── router.go
│       │
│       ├── grpc/
│       ├── kafka/
│
│
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
├── go.mod
├── go.sum
├── main.go
└── Makefile



lifecycle api:


Client
  │
  ▼
┌──────────────────────────────┐
│ interfaces/rest              │
│ router → middleware → handler│
└──────────────┬───────────────┘
               │ DTO / Command
               ▼
┌──────────────────────────────┐
│ applications                 │
│ command / query / service    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ domain                      │
│ entity / value object       │
│ business rule / repository  │
└──────────────┬───────────────┘
               │ Repository Interface
               ▼
┌──────────────────────────────┐
│ infrastructure              │
│ PostgreSQL / Redis / Kafka  │
└──────────────┬───────────────┘
               │
               ▼
            Database

Response đi ngược lại:

Database
   ↓
Repository
   ↓
Application
   ↓
Handler
   ↓
HTTP Response
   ↓
Client


khi bootstrap app:

main.go
   ↓
bootstrap/app.go
   ↓
load config
   ↓
connect PostgreSQL
   ↓
create repositories
   ↓
create application services / handlers
   ↓
register routes
   ↓
start HTTP server



| File                 | Trách nhiệm               |
| -------------------- | ------------------------- |
| `main.go`            | Entry point               |
| `cmd/root.go`        | Root CLI                  |
| `cmd/http.go`        | Command để chạy HTTP      |
| `cmd/kafka.go`       | Command để chạy Kafka     |
| `bootstrap/http.go`  | Wire dependency cho HTTP  |
| `bootstrap/kafka.go` | Wire dependency cho Kafka |
| `applications/`      | Use case                  |
| `domain/`            | Business                  |
| `infrastructure/`    | DB/Kafka/Redis/AWS        |
| `interfaces/`        | HTTP/gRPC/Kafka adapter   |

cmd/http.go và cmd/kafka.go thường là 2 entry command khác nhau, không phải mặc định start cả hai. Bạn muốn HTTP + Kafka cùng process thì có thể thiết kế root để start cả hai, nhưng nếu muốn scale/deploy độc lập thì chạy 2 process/container sẽ hợp lý hơn.















































