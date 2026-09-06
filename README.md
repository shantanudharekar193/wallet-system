# Wallet System

A secure digital wallet backend built with Spring Boot. It supports user registration, JWT authentication, wallet top-ups, peer-to-peer transfers, idempotent money operations, and concurrency-safe balance updates using database-level locking.

## Features

- JWT-based authentication and role-based authorization (`USER`, `ADMIN`)
- One wallet per user, created automatically at registration
- Idempotent add/transfer operations via `Idempotency-Key` header (enforced with a unique DB constraint)
- Pessimistic row locking for wallet updates plus optimistic locking via `@Version`
- Atomic transfers with ordered wallet locking to prevent deadlocks
- Global exception handling with consistent error responses
- Request payload validation using Jakarta Bean Validation
- H2 file database for local persistence
- JaCoCo coverage gate (80%+) and SonarQube integration

## Tech Stack

- Java 17
- Spring Boot 4
- Spring Security + JWT (jjwt)
- Spring Data JPA
- H2 Database
- Maven
- JaCoCo + SonarQube

## Project Structure

```
src/main/java/com/shantanu/wallet/
├── config/          # Security and JWT filter
├── controller/      # REST APIs (auth, wallet, admin)
├── dto/             # Request/response objects
├── entity/          # JPA entities and enums
├── exception/       # Custom exceptions + global handler
├── repository/      # Spring Data repositories
└── service/         # Business logic
```

## Prerequisites

- JDK 17+
- Maven 3.9+ (or use the included Maven wrapper)

## Run Locally

```bash
# Windows
.\mvnw.cmd spring-boot:run

# Linux/macOS
./mvnw spring-boot:run
```

The application starts on `http://localhost:8080`.

H2 console: `http://localhost:8080/h2-console`
- JDBC URL: `jdbc:h2:file:./data/walletdb`
- Username: `sa`
- Password: (empty)

## API Request Flow

### 1. Register

```http
POST /auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

Creates a user with role `USER` and an empty wallet.

### 2. Login

```http
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

Returns:

```json
{ "token": "<JWT>" }
```

Use the token in subsequent requests:

```http
Authorization: Bearer <JWT>
```

### 3. Add Money

```http
POST /wallet/add
Authorization: Bearer <JWT>
Idempotency-Key: unique-key-123
Content-Type: application/json

{ "amount": 100.00 }
```

### 4. Transfer Money

```http
POST /wallet/transfer
Authorization: Bearer <JWT>
Idempotency-Key: unique-key-456
Content-Type: application/json

{
  "toUserId": 2,
  "amount": 50.00
}
```

### 5. View Wallet

```http
GET /wallet
Authorization: Bearer <JWT>
```

### 6. View Transaction History

```http
GET /wallet/transactions?page=0&size=10
Authorization: Bearer <JWT>
```

### Admin APIs

Requires a user with role `ADMIN`.

```http
GET /admin/wallets?page=0&size=20
GET /admin/transactions?page=0&size=20
Authorization: Bearer <ADMIN_JWT>
```

## Security Implementation

- Passwords are hashed with BCrypt before storage.
- JWT is validated on every protected request by `JwtAuthenticationFilter`.
- Wallet identity is derived from the authenticated user's email, never from request body fields.
- `/admin/**` endpoints require `ROLE_ADMIN`.
- All other business endpoints require authentication.

## Idempotency Handling

- Clients must send an `Idempotency-Key` header for add/transfer operations.
- The key is stored in the `transactions.idempotency_key` column with a unique constraint.
- On duplicate requests, the service returns the existing transaction instead of reprocessing.
- Concurrent duplicate inserts are handled via `DataIntegrityViolationException` fallback lookup.

## Concurrency Strategy

- **Pessimistic locking**: wallet rows are locked with `PESSIMISTIC_WRITE` during add/transfer.
- **Ordered locking**: transfer locks wallets in ascending ID order to avoid deadlocks.
- **Optimistic locking**: `@Version` on `Wallet` detects concurrent updates and returns HTTP 409.
- In-memory locks/synchronized blocks are not used.

## Transaction Boundaries

- `addMoney` and `transfer` run inside `@Transactional` service methods.
- Wallet balance updates and transaction record creation happen in the same transaction.
- Transfer debits sender and credits receiver atomically; failure rolls back both changes.

## Testing

```bash
# Run tests
.\mvnw.cmd test

# Run tests with coverage report
.\mvnw.cmd verify
```

Coverage report: `target/site/jacoco/index.html`

## SonarQube Analysis

1. Start SonarQube locally or use SonarCloud.
2. Run:

```bash
.\mvnw.cmd clean verify sonar:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=<YOUR_SONAR_TOKEN>
```

Configuration is in `sonar-project.properties` and `pom.xml`.

## Design Decisions & Assumptions

- Each user owns exactly one wallet created at registration time.
- Admin users are not seeded automatically; create them directly in the database or via a test/setup script.
- Transfer target is specified by receiver `userId` (not wallet ID) to align with the assignment entity model.
- Failed transactions are modeled in the schema but current flows persist only successful operations.
- Amount precision uses `BigDecimal` with scale 2 at the entity level.

## Deployment (Optional)

The app can be deployed to Render/Railway/Heroku by:

1. Setting `PORT` from the platform environment.
2. Using an H2 file path or switching to a managed DB if required by the platform.
3. Setting `JWT_SECRET` as an environment variable (recommended for production).

Example Render start command:

```bash
./mvnw -DskipTests spring-boot:run
```

## License

This project is created for an assignment submission.
