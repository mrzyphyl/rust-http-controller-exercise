# Rust HTTP Controllers — A Junior Developer's Complete Guide

## What Is This Project?

This is a skill-building exercise where you will build a **REST API in Rust** from scratch. It covers authentication, authorization, middleware, password security, and database access — the core pillars of almost every real-world backend service.

You will not use a database server like PostgreSQL or MySQL. Instead, you will use **SQLite** — a file-based (or in-memory) database — so you can run everything locally with zero infrastructure setup. This keeps the focus on learning Rust patterns, not DevOps.

---

## Table of Contents

1. [Core Principles and Rationale](#1-core-principles-and-rationale)
2. [Project Architecture Overview](#2-project-architecture-overview)
3. [Dependency Choices and Why](#3-dependency-choices-and-why)
4. [Step 1 — Project Structure](#step-1--project-structure)
5. [Step 2 — Cargo.toml Dependencies](#step-2--cargotoml-dependencies)
6. [Step 3 — The Database Layer (SQLite Seed)](#step-3--the-database-layer-sqlite-seed)
7. [Step 4 — The Permissions Enum and Roles](#step-4--the-permissions-enum-and-roles)
8. [Step 5 — Models and Request/Response Types](#step-5--models-and-requestresponse-types)
9. [Step 6 — Password Hashing](#step-6--password-hashing)
10. [Step 7 — JWT and Authentication Tokens](#step-7--jwt-and-authentication-tokens)
11. [Step 8 — Middleware](#step-8--middleware)
12. [Step 9 — Auth Endpoints](#step-9--auth-endpoints)
13. [Step 10 — Admin Endpoints](#step-10--admin-endpoints)
14. [Step 11 — Error Handling](#step-11--error-handling)
15. [Step 12 — Wiring It All Together in main.rs](#step-12--wiring-it-all-together-in-mainrs)
16. [Step 13 — Unit Tests](#step-13--unit-tests)
17. [Best Practices Checklist](#best-practices-checklist)
18. [How to Test Your Endpoints](#how-to-test-your-endpoints)
19. [Glossary](#glossary)

---

## 1. Core Principles and Rationale

Before writing a single line of code, you need to understand **why** the design looks the way it does. Skipping this section is the single biggest mistake junior developers make — they copy patterns without understanding them and then can't adapt when something goes wrong.

### Separation of Concerns

Every piece of code should have **one job**. Your database logic should not be tangled up with your HTTP routing logic. Your password hashing should not be inside your endpoint handler. When each module does one thing, you can:

- Test each part independently
- Replace one part without touching the others
- Read the code without holding the entire system in your head

This project enforces separation via Rust's **module system**. Each concern lives in its own file.

### Defense in Depth

Security is not a single lock on a door — it is many layers. In this API:

- Passwords are **never stored in plain text** — they are hashed with a slow, intentionally expensive algorithm (bcrypt or argon2)
- Tokens are **short-lived JWTs** — even if stolen, they expire
- Routes are protected by **middleware** that checks the token before the handler ever runs
- **Role-based access control (RBAC)** means even authenticated users cannot call admin endpoints unless they have the right role

Each layer is independent. If one fails, the others still hold.

### The Principle of Least Privilege

A user should only be able to do what they need to do. A regular user cannot delete another user. An admin can. This is enforced by the `Permission` enum and the middleware that checks it. Never give a caller more power than they asked for or need.

### Explicit Over Implicit

Rust's type system forces you to be explicit. If a function can fail, it returns a `Result`. If a value might not exist, it is wrapped in `Option`. You will not have null pointer exceptions or silent failures. Lean into this — it is one of Rust's greatest strengths. When you see `unwrap()` in tutorial code, that is a shortcut for demonstration only. In production, always handle errors properly.

### Passwords and Tokens Are Not the Same Thing

A **password** is a secret the user knows. It should be hashed and stored. It is used to prove identity once.

A **token** (JWT) is a short-lived credential the server issues after verifying the password. The client sends the token on every subsequent request. The server does not need to hit the database on every request — it can verify the token's signature mathematically. This is why JWT-based auth scales well.

---

## 2. Project Architecture Overview

```
rust-http-controllers/
├── Cargo.toml
├── src/
│   ├── main.rs              # Entry point — wires everything together
│   ├── db.rs                # SQLite connection, schema creation, and seed data
│   ├── models.rs            # Rust structs that map to DB rows and HTTP payloads
│   ├── permissions.rs       # The Permission enum and Role enum
│   ├── auth/
│   │   ├── mod.rs           # Re-exports for the auth module
│   │   ├── password.rs      # Password hashing and verification
│   │   └── token.rs         # JWT creation and validation
│   ├── middleware.rs        # Axum extractors that enforce auth and roles
│   ├── handlers/
│   │   ├── mod.rs           # Re-exports for all handlers
│   │   ├── auth.rs          # login, register, forgot_password endpoints
│   │   └── admin.rs         # admin-only endpoints
│   └── error.rs             # Unified error type and HTTP error responses
```

Each file has a single responsibility. You can read any one file in isolation and understand what it does.

---

## 3. Dependency Choices and Why

Here are the crates you will add and the reason for each one. Understanding your dependencies is critical — you should never add a crate you cannot explain.

| Crate | Purpose | Why This One |
|---|---|---|
| `axum` | HTTP framework | Built on `tokio` and `hyper`. Ergonomic, composable, widely used in production. Uses Rust's type system for routing and extractors. |
| `tokio` | Async runtime | `axum` requires it. The de facto standard async runtime in Rust. Use the `full` feature for convenience. |
| `serde` + `serde_json` | Serialization | How Rust structs become JSON and vice versa. Essential for any HTTP API. |
| `rusqlite` | SQLite driver | Simple, well-maintained, works in-process. No server needed. Use the `bundled` feature so the SQLite C library is compiled in — no system install required. |
| `bcrypt` or `argon2` | Password hashing | Passwords must never be stored plain. These algorithms are intentionally slow (configurable cost factor) to defeat brute-force attacks. `argon2` is the more modern choice and won the Password Hashing Competition. |
| `jsonwebtoken` | JWT creation/validation | Industry-standard way to issue and verify authentication tokens. |
| `uuid` | Unique IDs | Every user needs a unique identifier that is not predictable from sequence. UUIDs are random and collision-resistant. |
| `chrono` | Date and time | For token expiry timestamps and `created_at` / `updated_at` fields. |
| `thiserror` | Error derivation | Makes defining your own error types clean and concise. |
| `tracing` + `tracing-subscriber` | Structured logging | Better than `println!`. Gives you log levels, spans, and structured output. Essential for debugging in production. |

---

## Step 1 — Project Structure

Create the directories and empty files first. This forces you to think about structure before implementation.

```bash
mkdir src/auth
mkdir src/handlers
touch src/db.rs
touch src/models.rs
touch src/permissions.rs
touch src/middleware.rs
touch src/error.rs
touch src/auth/mod.rs
touch src/auth/password.rs
touch src/auth/token.rs
touch src/handlers/mod.rs
touch src/handlers/auth.rs
touch src/handlers/admin.rs
```

On Windows with PowerShell:

```powershell
New-Item -ItemType Directory -Path src/auth, src/handlers
New-Item -ItemType File -Path src/db.rs, src/models.rs, src/permissions.rs, src/middleware.rs, src/error.rs
New-Item -ItemType File -Path src/auth/mod.rs, src/auth/password.rs, src/auth/token.rs
New-Item -ItemType File -Path src/handlers/mod.rs, src/handlers/auth.rs, src/handlers/admin.rs
```

---

## Step 2 — Cargo.toml Dependencies

Replace the empty `[dependencies]` section with:

```toml
[dependencies]
axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rusqlite = { version = "0.31", features = ["bundled"] }
argon2 = "0.5"
jsonwebtoken = "9"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
rand = "0.8"
```

**Why `rand`?** You will need it to generate a secure random token for password reset emails, since those reset tokens are not JWTs — they are single-use secrets.

**Why `chrono` with the `serde` feature?** So `DateTime` types can be serialized directly into JSON responses without manual conversion.

Run `cargo build` after editing this. Rust will fetch and compile all dependencies. The first build will take a few minutes — subsequent builds are incremental and fast.

---

## Step 3 — The Database Layer (SQLite Seed)

**File: `src/db.rs`**

### What This File Does

- Opens (or creates) an in-memory SQLite database using `rusqlite`
- Creates the `users` and `password_reset_tokens` tables
- Seeds the database with a few sample users so you do not have to register before you can test admin endpoints

### The Schema

Think carefully about the schema before writing code. Here is what you need:

**`users` table:**

| Column | Type | Notes |
|---|---|---|
| `id` | TEXT (UUID) | Primary key. UUID string. |
| `username` | TEXT | Unique. The login name. |
| `email` | TEXT | Unique. Used for password reset. |
| `password_hash` | TEXT | Never the plain password. |
| `role` | TEXT | `"user"` or `"admin"`. Stored as string, mapped to enum in Rust. |
| `created_at` | TEXT | ISO 8601 datetime string. |
| `updated_at` | TEXT | ISO 8601 datetime string. |

**`password_reset_tokens` table:**

| Column | Type | Notes |
|---|---|---|
| `id` | TEXT (UUID) | Primary key. |
| `user_id` | TEXT | Foreign key to `users.id`. |
| `token` | TEXT | Random hex string. Single-use. |
| `expires_at` | TEXT | ISO 8601. Token is invalid after this. |
| `used` | INTEGER | 0 = unused, 1 = used. SQLite has no boolean. |

### Key Concept: In-Memory vs File-Based SQLite

- **In-memory (`":memory:"`)**: The database lives in RAM. It is destroyed when the process exits. Perfect for this exercise — no cleanup needed, always starts fresh.
- **File-based (`"./data.db"`)**: Persists to disk. You can inspect it with a SQLite browser tool.

For this skill project, use in-memory. Pass the connection around using Axum's `State` extractor (explained in Step 12).

### Key Concept: Connection Sharing in Async Code

`rusqlite::Connection` is **not** `Send` + `Sync` by default, meaning you cannot share it across async tasks safely on its own. You have two options:

1. Wrap it in `Arc<Mutex<Connection>>` — a reference-counted, mutually exclusive lock. Only one task accesses the DB at a time.
2. Use `rusqlite`'s `r2d2` integration for a connection pool.

For this exercise, **`Arc<Mutex<Connection>>`** is the right choice. It is simpler and sufficient for a single-user skill project. In production, you would use a proper async driver like `sqlx` with a connection pool.

### Seed Data

Create at least:
- 1 admin user (role: `"admin"`)
- 2 regular users (role: `"user"`)

Hash the passwords when seeding. Do not seed plain text passwords — this reinforces the habit.

---

## Step 4 — The Permissions Enum and Roles

**File: `src/permissions.rs`**

### Why an Enum, Not Just a String?

When roles are plain strings like `"admin"` or `"user"`, it is easy to typo them (`"Admin"`, `"ADMIN"`, `"admim"`). The compiler cannot catch string mistakes. An **enum** turns a runtime error into a **compile-time error**. If you misspell `Role::Admim`, the compiler tells you immediately.

### The `Role` Enum

```rust
#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
pub enum Role {
    User,
    Admin,
}
```

You also need a way to convert between the database string and the enum. Implement `From<String> for Role` (or `TryFrom`) and `From<Role> for String`.

### The `Permission` Enum

A role is what you **are**. A permission is what you are **allowed to do**. This distinction matters as systems grow. For now:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Permission {
    ReadOwnProfile,
    UpdateOwnProfile,
    ReadAllUsers,
    UpdateAnyUser,
    DeleteAnyUser,
    ResetAnyUserPassword,
}
```

Then write a function that maps a `Role` to a `Vec<Permission>`:

```rust
pub fn permissions_for_role(role: &Role) -> Vec<Permission> {
    match role {
        Role::User => vec![
            Permission::ReadOwnProfile,
            Permission::UpdateOwnProfile,
        ],
        Role::Admin => vec![
            Permission::ReadOwnProfile,
            Permission::UpdateOwnProfile,
            Permission::ReadAllUsers,
            Permission::UpdateAnyUser,
            Permission::DeleteAnyUser,
            Permission::ResetAnyUserPassword,
        ],
    }
}
```

And a helper:

```rust
pub fn has_permission(role: &Role, permission: &Permission) -> bool {
    permissions_for_role(role).contains(permission)
}
```

### Why Not Just Check `role == Role::Admin` Everywhere?

You could. But hardcoding role checks throughout your handlers is fragile. If you add a `Moderator` role later, you have to find every `role == Role::Admin` check and decide if a Moderator should also have that permission. With a permission system, you just update `permissions_for_role`. The handlers do not change.

---

## Step 5 — Models and Request/Response Types

**File: `src/models.rs`**

This file contains all the Rust structs that represent data. Think of it as the contract between your application layers.

### User (Database Row)

```rust
#[derive(Debug, Clone, serde::Serialize)]
pub struct User {
    pub id: String,
    pub username: String,
    pub email: String,
    pub password_hash: String,  // Never serialize this in responses!
    pub role: Role,
    pub created_at: String,
    pub updated_at: String,
}
```

### UserResponse (What You Send Back to the Client)

**Never send `password_hash` to the client.** Create a separate response type:

```rust
#[derive(Debug, serde::Serialize)]
pub struct UserResponse {
    pub id: String,
    pub username: String,
    pub email: String,
    pub role: String,
    pub created_at: String,
}
```

This is a critical security principle: **your internal model and your API response type are different structs**. The compiler enforces that you cannot accidentally leak the hash.

### Request Payloads (What the Client Sends to You)

```rust
#[derive(Debug, serde::Deserialize)]
pub struct RegisterRequest {
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(Debug, serde::Deserialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

#[derive(Debug, serde::Deserialize)]
pub struct ForgotPasswordRequest {
    pub email: String,
}

#[derive(Debug, serde::Deserialize)]
pub struct ResetPasswordRequest {
    pub token: String,
    pub new_password: String,
}

#[derive(Debug, serde::Deserialize)]
pub struct AdminUpdateUserRequest {
    pub username: Option<String>,
    pub email: Option<String>,
    pub role: Option<String>,
}

#[derive(Debug, serde::Deserialize)]
pub struct AdminResetPasswordRequest {
    pub new_password: String,
}
```

Notice that `AdminUpdateUserRequest` uses `Option<String>` for all fields. This is the **partial update pattern** — the client only sends the fields they want to change. If a field is `None`, you leave it unchanged in the database.

### The Claims Struct (JWT Payload)

```rust
#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct Claims {
    pub sub: String,   // Subject — the user's UUID
    pub role: String,  // The user's role as a string
    pub exp: usize,    // Expiry timestamp (Unix epoch seconds)
    pub iat: usize,    // Issued-at timestamp
}
```

`sub`, `exp`, and `iat` are standard JWT fields (called "claims"). The `jsonwebtoken` crate validates `exp` automatically.

---

## Step 6 — Password Hashing

**File: `src/auth/password.rs`**

### Why You Cannot Just Use SHA-256 or MD5

MD5 and SHA-256 are **fast** hashing algorithms — designed for checksums and data integrity, not password security. An attacker with a modern GPU can compute **billions of SHA-256 hashes per second**. With a leaked database, they can crack most passwords in hours.

Password hashing algorithms like **Argon2** are different:
- They are **intentionally slow** (configurable)
- They use a lot of **memory** (so you cannot run millions in parallel on a GPU)
- They use a **salt** — a random value mixed into the hash — so two users with the same password get different hashes. This defeats rainbow table attacks.

### The Two Functions You Need

```rust
// Takes a plain-text password, returns a hash string safe to store in DB
pub fn hash_password(password: &str) -> Result<String, AppError>

// Takes a plain-text password and the stored hash, returns true if they match
pub fn verify_password(password: &str, hash: &str) -> Result<bool, AppError>
```

With `argon2`, the hash string it produces already contains the salt, the algorithm parameters, and the hash itself — all in one string. You store that entire string. When verifying, you pass the same string back and `argon2` extracts the salt from it automatically.

**You never store the salt separately.** Argon2's output format encodes everything needed for verification.

### Security Note: Constant-Time Comparison

When verifying a password, the comparison must be **constant-time** — it must take the same amount of time whether the password is correct or not. If incorrect passwords fail faster, an attacker can measure response times and learn information about the stored hash (a **timing attack**). The `argon2` crate handles this for you internally, but be aware of the concept.

---

## Step 7 — JWT and Authentication Tokens

**File: `src/auth/token.rs`**

### What Is a JWT?

A JSON Web Token (JWT) is a compact, URL-safe string with three parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyLXV1aWQiLCJyb2xlIjoidXNlciIsImV4cCI6MTcwMDAwMDAwMH0.SIGNATURE
   ^-- Header (algorithm)      ^-- Payload (claims, base64)                                        ^-- HMAC signature
```

The **header** says which algorithm was used to sign it (e.g., HS256 — HMAC-SHA256).  
The **payload** contains the claims (user ID, role, expiry).  
The **signature** is a cryptographic proof that the token was issued by your server and has not been tampered with.

The server signs the token with a **secret key**. Anyone can decode the payload (it is just base64), but they cannot forge a valid signature without the secret key.

**Important:** JWTs are not encrypted — do not put sensitive data in them. They are signed, not secret. The role and user ID in the payload can be read by anyone who has the token.

### The Two Functions You Need

```rust
// Creates a signed JWT for a user. Expires after N hours.
pub fn create_token(user_id: &str, role: &str, secret: &str) -> Result<String, AppError>

// Validates the JWT signature and expiry. Returns the Claims if valid.
pub fn validate_token(token: &str, secret: &str) -> Result<Claims, AppError>
```

### The Secret Key

The signing secret should be a long, random string (at least 32 characters). For this exercise, define it as a constant in your config or pass it through Axum's `State`. **Never hardcode secrets in production** — use environment variables. For this skill project, a constant is acceptable.

### Token Expiry

Set tokens to expire after 24 hours (or less). In `Claims`, `exp` is a Unix timestamp (seconds since January 1, 1970). Compute it with `chrono`:

```rust
let expiry = chrono::Utc::now()
    .checked_add_signed(chrono::Duration::hours(24))
    .unwrap()
    .timestamp() as usize;
```

### Password Reset Tokens vs JWTs

Password reset tokens are **not JWTs**. They are:
- A random 32-byte hex string (generated with `rand`)
- Stored in the `password_reset_tokens` table
- Single-use (marked `used = 1` after redemption)
- Short-lived (expire after 15–60 minutes)

Why not JWTs? Because you need to be able to **invalidate** them after use. JWTs are stateless — once issued, you cannot "cancel" them without a blocklist. Reset tokens are stateful — stored in the DB, deleted or marked used after redemption.

---

## Step 8 — Middleware

**File: `src/middleware.rs`**

### What Is Middleware?

Middleware is code that runs **before your handler**. In Axum, middleware is implemented as **extractors** — types that implement `FromRequestParts`. When Axum calls your handler, it first tries to extract all the parameters. If an extractor fails (e.g., the token is missing or invalid), Axum returns an error response without ever calling the handler.

This is the correct pattern in Axum. You do not write a function that wraps every route — you write a type that expresses a requirement, and the type system enforces it.

### The `AuthenticatedUser` Extractor

Create a struct:

```rust
pub struct AuthenticatedUser {
    pub user_id: String,
    pub role: Role,
}
```

Implement `FromRequestParts<AppState>` for it. In the implementation:

1. Extract the `Authorization` header from the request
2. Check it starts with `"Bearer "`
3. Extract the token string after `"Bearer "`
4. Call `validate_token()` from your token module
5. If validation succeeds, return `AuthenticatedUser { user_id: claims.sub, role: ... }`
6. If anything fails, return a `401 Unauthorized` response

Any handler that has `AuthenticatedUser` as a parameter will automatically require a valid JWT. Handlers that do not have it (login, register) are public.

### The `AdminUser` Extractor

Create a second extractor:

```rust
pub struct AdminUser(pub AuthenticatedUser);
```

Its `FromRequestParts` implementation:
1. Extract an `AuthenticatedUser` (which already validates the JWT)
2. Check that `role == Role::Admin`
3. If yes, return `AdminUser(...)`
4. If no, return `403 Forbidden`

Now admin-only handlers just take `AdminUser` as a parameter. The compiler enforces the permission check.

### Why Extractors Are Better Than Manual Checks

Without extractors, every handler would start with:

```rust
async fn some_handler(...) {
    let token = get_token_from_header(&req)?;
    let claims = validate_token(&token)?;
    if claims.role != "admin" {
        return Err(forbidden());
    }
    // ... actual handler logic
}
```

This is repetitive, easy to forget, and cannot be enforced by the compiler. With extractors, the requirement is in the **function signature**:

```rust
async fn some_handler(admin: AdminUser, ...) { ... }
```

You cannot call this handler without an admin JWT. The compiler guarantees it.

---

## Step 9 — Auth Endpoints

**File: `src/handlers/auth.rs`**

### POST `/auth/register`

**Request body:** `RegisterRequest`  
**Response:** `201 Created` with `UserResponse`

Steps:
1. Deserialize the request body with Axum's `Json` extractor
2. Validate input — username and email should not be empty, password should be at least 8 characters
3. Check if the username or email already exists in the DB (return `409 Conflict` if so)
4. Hash the password with `hash_password()`
5. Generate a UUID for the new user
6. Insert the user into the `users` table with `role = "user"` (users cannot self-register as admin)
7. Return a `UserResponse` (not the full `User` — never return the hash)

### POST `/auth/login`

**Request body:** `LoginRequest`  
**Response:** `200 OK` with `{ "token": "..." }`

Steps:
1. Look up the user by username in the DB
2. If not found, return `401 Unauthorized` — **do not say "user not found"**. Always return the same error ("Invalid credentials") whether the username is wrong or the password is wrong. This prevents **user enumeration** — an attacker should not be able to determine which usernames exist.
3. Call `verify_password()` with the submitted password and the stored hash
4. If the password does not match, return `401 Unauthorized`
5. Call `create_token()` to generate a JWT
6. Return the token in the response body

### POST `/auth/forgot-password`

**Request body:** `ForgotPasswordRequest` (just the email)  
**Response:** `200 OK` — always, even if the email does not exist

**Why always 200?** Same reason as above — you do not want to leak whether an email is registered. An attacker could otherwise enumerate registered emails.

Steps:
1. Look up the user by email
2. If found:
   a. Generate a random 32-byte token using `rand` and encode it as a hex string
   b. Set expiry to 30 minutes from now
   c. Insert into `password_reset_tokens`
   d. "Send" the email — for this exercise, just `tracing::info!("Reset token for {}: {}", email, token)` to the log. In production you would call an email API (SendGrid, AWS SES, etc.)
3. Return `200 OK` with a generic message like `"If that email is registered, a reset link has been sent"`

### POST `/auth/reset-password`

**Request body:** `ResetPasswordRequest` (token + new_password)  
**Response:** `200 OK`

Steps:
1. Look up the token in `password_reset_tokens`
2. If not found, return `400 Bad Request`
3. Check `used == 0` — if already used, return `400`
4. Check `expires_at` — if expired, return `400`
5. Hash the new password
6. Update the user's `password_hash` in the `users` table
7. Mark the token as `used = 1` in `password_reset_tokens`
8. Return `200 OK`

---

## Step 10 — Admin Endpoints

**File: `src/handlers/admin.rs`**

All of these endpoints require `AdminUser` as an extractor parameter.

### GET `/admin/users`

**Response:** `200 OK` with a list of `UserResponse`

Steps:
1. Query all users from the DB
2. Map each `User` to a `UserResponse` (strip the hash)
3. Return the list

### PATCH `/admin/users/:id`

**Request body:** `AdminUpdateUserRequest`  
**Response:** `200 OK` with updated `UserResponse`

Steps:
1. Extract the `:id` path parameter
2. Look up the user — if not found, return `404`
3. Apply only the `Some(...)` fields from the request (partial update)
4. Update `updated_at` to now
5. Run the UPDATE query
6. Return the updated user as `UserResponse`

### POST `/admin/users/:id/reset-password`

**Request body:** `AdminResetPasswordRequest` (new_password)  
**Response:** `200 OK`

Steps:
1. Look up the user — if not found, return `404`
2. Hash the new password
3. Update `password_hash` and `updated_at` in the DB
4. Return `200 OK`

### DELETE `/admin/users/:id`

**Response:** `204 No Content`

Steps:
1. Look up the user — if not found, return `404`
2. Delete from `users` table (cascade will handle reset tokens if you set up FK constraints)
3. Return `204 No Content` — the standard HTTP response for a successful delete with no body

**Guard:** Prevent admins from deleting themselves. Check if the `:id` matches the `AdminUser.user_id` from the extractor. If so, return `400 Bad Request` with message `"Cannot delete your own account"`.

---

## Step 11 — Error Handling

**File: `src/error.rs`**

### Why a Unified Error Type?

Every function in your application can fail in different ways: database errors, token validation errors, hashing errors, validation errors. If each returns a different error type, your handlers become messy with conversion code everywhere.

A single `AppError` enum unifies them:

```rust
#[derive(thiserror::Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] rusqlite::Error),

    #[error("Authentication failed")]
    Unauthorized,

    #[error("Forbidden")]
    Forbidden,

    #[error("Not found")]
    NotFound,

    #[error("Conflict: {0}")]
    Conflict(String),

    #[error("Bad request: {0}")]
    BadRequest(String),

    #[error("Internal error: {0}")]
    Internal(String),
}
```

### Converting `AppError` to HTTP Responses

Axum requires that errors returned from handlers implement `IntoResponse`. Implement it for `AppError`:

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match &self {
            AppError::Unauthorized  => (StatusCode::UNAUTHORIZED,  self.to_string()),
            AppError::Forbidden     => (StatusCode::FORBIDDEN,     self.to_string()),
            AppError::NotFound      => (StatusCode::NOT_FOUND,     self.to_string()),
            AppError::Conflict(m)   => (StatusCode::CONFLICT,      m.clone()),
            AppError::BadRequest(m) => (StatusCode::BAD_REQUEST,   m.clone()),
            AppError::Database(_)   => (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".to_string()),
            AppError::Internal(_)   => (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".to_string()),
        };

        let body = serde_json::json!({ "error": message });
        (status, Json(body)).into_response()
    }
}
```

Notice that `Database` and `Internal` both return a generic `"Internal server error"` to the client — you never expose internal error details. The real error is only visible in your logs (via `tracing`).

---

## Step 12 — Wiring It All Together in main.rs

**File: `src/main.rs`**

This is where everything is assembled.

### App State

Define a struct to hold shared state — the database connection and the JWT secret:

```rust
#[derive(Clone)]
pub struct AppState {
    pub db: Arc<Mutex<Connection>>,
    pub jwt_secret: String,
}
```

The `Clone` derive is required because Axum clones the state for each request.

### Route Registration

Axum uses a builder pattern for routes:

```rust
let app = Router::new()
    // Public routes
    .route("/auth/register",        post(auth::register))
    .route("/auth/login",           post(auth::login))
    .route("/auth/forgot-password", post(auth::forgot_password))
    .route("/auth/reset-password",  post(auth::reset_password))
    // Protected routes (require valid JWT — enforced by AuthenticatedUser extractor)
    // Admin routes (require admin role — enforced by AdminUser extractor)
    .route("/admin/users",                    get(admin::list_users))
    .route("/admin/users/:id",                patch(admin::update_user))
    .route("/admin/users/:id",                delete(admin::delete_user))
    .route("/admin/users/:id/reset-password", post(admin::reset_user_password))
    .with_state(state);
```

### Starting the Server

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
tracing::info!("Listening on http://0.0.0.0:3000");
axum::serve(listener, app).await.unwrap();
```

### Initializing Tracing

At the very top of `main`:

```rust
tracing_subscriber::fmt()
    .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
    .init();
```

This means you can control log verbosity with the `RUST_LOG` environment variable:

```bash
RUST_LOG=debug cargo run    # shows everything
RUST_LOG=info cargo run     # shows info and above
```

---

## Step 13 — Unit Tests

### Why Write Unit Tests?

Unit tests verify that individual pieces of your code work correctly **in isolation**, without running a real HTTP server or a real database. They run fast (milliseconds), give precise error messages when something breaks, and act as living documentation — a test named `test_user_role_does_not_have_admin_permissions` tells the next developer exactly what the code is supposed to do.

In Rust, unit tests live in the **same file** as the code they test, inside a `#[cfg(test)]` module. The `#[cfg(test)]` attribute tells the compiler to only include this code when running `cargo test` — it is stripped out of your release binary entirely.

```rust
// At the bottom of any .rs file:
#[cfg(test)]
mod tests {
    use super::*;  // import everything from the parent module

    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

### How to Run Tests

```bash
cargo test                         # run all tests
cargo test test_permissions        # run tests whose name contains "test_permissions"
cargo test -- --nocapture          # show println! output during tests
```

### What to Test and What Not to Test

**Test:**
- Pure logic: permission mappings, role conversions, input validation rules
- Password hashing and verification
- JWT creation and validation
- Error type conversions
- Edge cases: expired tokens, already-used reset tokens, empty inputs

**Do not unit test:**
- HTTP routing (use integration tests for that)
- Database queries in isolation (they need a real connection — use a test helper that sets up an in-memory DB)
- Axum extractors directly (they require a full request context)

For database-touching code, use a **test helper function** that creates a fresh in-memory SQLite database with the schema applied. Each test gets its own clean database, so tests do not interfere with each other.

---

### Tests for `src/permissions.rs`

These tests verify your RBAC logic. They are pure logic with no I/O — fast and simple.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // --- Role enum conversion ---

    #[test]
    fn test_role_from_string_user() {
        let role = Role::from("user".to_string());
        assert_eq!(role, Role::User);
    }

    #[test]
    fn test_role_from_string_admin() {
        let role = Role::from("admin".to_string());
        assert_eq!(role, Role::Admin);
    }

    #[test]
    fn test_role_to_string_user() {
        let s = String::from(Role::User);
        assert_eq!(s, "user");
    }

    #[test]
    fn test_role_to_string_admin() {
        let s = String::from(Role::Admin);
        assert_eq!(s, "admin");
    }

    // --- User role permissions ---

    #[test]
    fn test_user_role_has_read_own_profile() {
        assert!(has_permission(&Role::User, &Permission::ReadOwnProfile));
    }

    #[test]
    fn test_user_role_has_update_own_profile() {
        assert!(has_permission(&Role::User, &Permission::UpdateOwnProfile));
    }

    #[test]
    fn test_user_role_cannot_read_all_users() {
        assert!(!has_permission(&Role::User, &Permission::ReadAllUsers));
    }

    #[test]
    fn test_user_role_cannot_delete_any_user() {
        assert!(!has_permission(&Role::User, &Permission::DeleteAnyUser));
    }

    #[test]
    fn test_user_role_cannot_update_any_user() {
        assert!(!has_permission(&Role::User, &Permission::UpdateAnyUser));
    }

    #[test]
    fn test_user_role_cannot_reset_any_password() {
        assert!(!has_permission(&Role::User, &Permission::ResetAnyUserPassword));
    }

    // --- Admin role permissions ---

    #[test]
    fn test_admin_role_has_all_permissions() {
        let all_permissions = vec![
            Permission::ReadOwnProfile,
            Permission::UpdateOwnProfile,
            Permission::ReadAllUsers,
            Permission::UpdateAnyUser,
            Permission::DeleteAnyUser,
            Permission::ResetAnyUserPassword,
        ];
        for permission in &all_permissions {
            assert!(
                has_permission(&Role::Admin, permission),
                "Admin should have permission {:?}",
                permission
            );
        }
    }

    #[test]
    fn test_user_permissions_are_subset_of_admin() {
        // Everything a User can do, an Admin can also do
        let user_permissions = permissions_for_role(&Role::User);
        for permission in &user_permissions {
            assert!(
                has_permission(&Role::Admin, permission),
                "Admin should have User permission {:?}",
                permission
            );
        }
    }
}
```

**What these tests cover:**
- `Role` round-trips correctly through string conversion
- Each role has exactly the permissions it should have
- The subset relationship between User and Admin permissions is maintained
- The `assert!(..., "message")` pattern gives a clear failure message if a test fails

---

### Tests for `src/auth/password.rs`

Password tests verify that hashing is non-deterministic (two hashes of the same password differ) and that verification works correctly.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_hash_password_returns_ok() {
        let result = hash_password("my_secure_password");
        assert!(result.is_ok(), "hash_password should not return an error");
    }

    #[test]
    fn test_hashed_password_is_not_plain_text() {
        let password = "my_secure_password";
        let hash = hash_password(password).unwrap();
        // The hash must never equal the original password
        assert_ne!(hash, password);
    }

    #[test]
    fn test_same_password_produces_different_hashes() {
        // Argon2 includes a random salt — identical inputs produce different outputs
        let password = "my_secure_password";
        let hash1 = hash_password(password).unwrap();
        let hash2 = hash_password(password).unwrap();
        assert_ne!(
            hash1, hash2,
            "Two hashes of the same password must differ (different salts)"
        );
    }

    #[test]
    fn test_verify_correct_password_returns_true() {
        let password = "my_secure_password";
        let hash = hash_password(password).unwrap();
        let result = verify_password(password, &hash).unwrap();
        assert!(result, "Correct password must verify as true");
    }

    #[test]
    fn test_verify_wrong_password_returns_false() {
        let hash = hash_password("correct_password").unwrap();
        let result = verify_password("wrong_password", &hash).unwrap();
        assert!(!result, "Wrong password must verify as false");
    }

    #[test]
    fn test_verify_empty_password_returns_false() {
        let hash = hash_password("correct_password").unwrap();
        let result = verify_password("", &hash).unwrap();
        assert!(!result, "Empty password must not match a non-empty password hash");
    }

    #[test]
    fn test_hash_empty_password_is_still_a_hash() {
        // Even empty passwords should be hashable — the caller decides
        // whether to reject them at the validation layer, not the hashing layer
        let result = hash_password("");
        assert!(result.is_ok(), "hash_password with empty string should not panic");
    }
}
```

**Key concepts demonstrated:**
- `is_ok()` / `is_err()` — check `Result` variants without unwrapping
- `unwrap()` is acceptable in tests — if it panics, the test fails with a clear message
- Testing the negative case (wrong password, empty password) is as important as the happy path
- The "different hashes for same password" test verifies that salting is working

---

### Tests for `src/auth/token.rs`

Token tests verify that JWTs can be created and validated, and that invalid tokens are rejected.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    const TEST_SECRET: &str = "test_secret_key_that_is_long_enough_for_hs256";

    #[test]
    fn test_create_token_returns_a_string() {
        let result = create_token("user-uuid-123", "user", TEST_SECRET);
        assert!(result.is_ok());
        let token = result.unwrap();
        assert!(!token.is_empty());
    }

    #[test]
    fn test_token_has_three_parts_separated_by_dots() {
        // All JWTs have the format: header.payload.signature
        let token = create_token("user-uuid-123", "user", TEST_SECRET).unwrap();
        let parts: Vec<&str> = token.split('.').collect();
        assert_eq!(parts.len(), 3, "JWT must have exactly 3 dot-separated parts");
    }

    #[test]
    fn test_validate_valid_token_returns_claims() {
        let token = create_token("user-uuid-123", "admin", TEST_SECRET).unwrap();
        let result = validate_token(&token, TEST_SECRET);
        assert!(result.is_ok(), "A freshly created token must be valid");
    }

    #[test]
    fn test_validate_token_has_correct_user_id() {
        let user_id = "user-uuid-abc";
        let token = create_token(user_id, "user", TEST_SECRET).unwrap();
        let claims = validate_token(&token, TEST_SECRET).unwrap();
        assert_eq!(claims.sub, user_id);
    }

    #[test]
    fn test_validate_token_has_correct_role() {
        let token = create_token("some-id", "admin", TEST_SECRET).unwrap();
        let claims = validate_token(&token, TEST_SECRET).unwrap();
        assert_eq!(claims.role, "admin");
    }

    #[test]
    fn test_validate_token_with_wrong_secret_fails() {
        let token = create_token("user-uuid-123", "user", TEST_SECRET).unwrap();
        let result = validate_token(&token, "a_completely_different_secret");
        assert!(
            result.is_err(),
            "A token signed with a different secret must be rejected"
        );
    }

    #[test]
    fn test_validate_garbage_string_fails() {
        let result = validate_token("this.is.not.a.jwt", TEST_SECRET);
        assert!(result.is_err(), "A garbage string must not validate as a JWT");
    }

    #[test]
    fn test_validate_empty_string_fails() {
        let result = validate_token("", TEST_SECRET);
        assert!(result.is_err());
    }

    #[test]
    fn test_validate_tampered_token_fails() {
        let token = create_token("user-uuid-123", "user", TEST_SECRET).unwrap();
        // Flip the last character of the signature to simulate tampering
        let mut tampered = token.clone();
        let last = tampered.pop().unwrap();
        let replacement = if last == 'a' { 'b' } else { 'a' };
        tampered.push(replacement);

        let result = validate_token(&tampered, TEST_SECRET);
        assert!(
            result.is_err(),
            "A tampered token signature must be rejected"
        );
    }
}
```

**Key concepts demonstrated:**
- Using a `const TEST_SECRET` so all token tests use a consistent, known secret
- Testing the JWT structure directly (3 dot-separated parts) as a sanity check
- Testing that changing the secret invalidates the token — proves the signing works
- Testing tampered tokens — proves the signature verification is working, not just parsing
- Negative tests (wrong secret, garbage input, empty string) are critical for security code

---

### Tests for `src/error.rs`

Test that your `AppError` variants produce the correct HTTP status codes.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::http::StatusCode;
    use axum::response::IntoResponse;

    fn status_of(err: AppError) -> StatusCode {
        err.into_response().status()
    }

    #[test]
    fn test_unauthorized_maps_to_401() {
        assert_eq!(status_of(AppError::Unauthorized), StatusCode::UNAUTHORIZED);
    }

    #[test]
    fn test_forbidden_maps_to_403() {
        assert_eq!(status_of(AppError::Forbidden), StatusCode::FORBIDDEN);
    }

    #[test]
    fn test_not_found_maps_to_404() {
        assert_eq!(status_of(AppError::NotFound), StatusCode::NOT_FOUND);
    }

    #[test]
    fn test_conflict_maps_to_409() {
        assert_eq!(
            status_of(AppError::Conflict("username taken".to_string())),
            StatusCode::CONFLICT
        );
    }

    #[test]
    fn test_bad_request_maps_to_400() {
        assert_eq!(
            status_of(AppError::BadRequest("invalid input".to_string())),
            StatusCode::BAD_REQUEST
        );
    }

    #[test]
    fn test_internal_maps_to_500() {
        assert_eq!(
            status_of(AppError::Internal("something broke".to_string())),
            StatusCode::INTERNAL_SERVER_ERROR
        );
    }

    #[test]
    fn test_database_error_maps_to_500() {
        // rusqlite::Error can be constructed from its ffi code
        // Use Internal as a proxy since Database wraps rusqlite::Error
        // which is harder to construct in tests — use Internal instead
        assert_eq!(
            status_of(AppError::Internal("db error".to_string())),
            StatusCode::INTERNAL_SERVER_ERROR
        );
    }
}
```

**Why test error mappings?** If you ever refactor `AppError::into_response` and accidentally swap two status codes, these tests catch it immediately. Status codes matter — returning 403 where you meant 401 will confuse every client consuming your API.

---

### Tests for `src/db.rs`

Database tests use a fresh in-memory database per test. Define a test helper:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Helper: creates a fresh in-memory DB with schema and seed data applied
    fn test_db() -> Connection {
        let conn = Connection::open_in_memory().unwrap();
        apply_schema(&conn).unwrap();   // your function that runs CREATE TABLE statements
        seed_data(&conn).unwrap();      // your function that inserts sample users
        conn
    }

    #[test]
    fn test_schema_creates_users_table() {
        let conn = test_db();
        // Query sqlite_master to confirm the table exists
        let count: i64 = conn
            .query_row(
                "SELECT COUNT(*) FROM sqlite_master WHERE type='table' AND name='users'",
                [],
                |row| row.get(0),
            )
            .unwrap();
        assert_eq!(count, 1, "users table must exist after schema creation");
    }

    #[test]
    fn test_schema_creates_password_reset_tokens_table() {
        let conn = test_db();
        let count: i64 = conn
            .query_row(
                "SELECT COUNT(*) FROM sqlite_master WHERE type='table' AND name='password_reset_tokens'",
                [],
                |row| row.get(0),
            )
            .unwrap();
        assert_eq!(count, 1);
    }

    #[test]
    fn test_seed_inserts_at_least_one_admin() {
        let conn = test_db();
        let count: i64 = conn
            .query_row(
                "SELECT COUNT(*) FROM users WHERE role = 'admin'",
                [],
                |row| row.get(0),
            )
            .unwrap();
        assert!(count >= 1, "Seed data must include at least one admin user");
    }

    #[test]
    fn test_seed_inserts_at_least_two_regular_users() {
        let conn = test_db();
        let count: i64 = conn
            .query_row(
                "SELECT COUNT(*) FROM users WHERE role = 'user'",
                [],
                |row| row.get(0),
            )
            .unwrap();
        assert!(count >= 2, "Seed data must include at least two regular users");
    }

    #[test]
    fn test_seeded_passwords_are_not_plain_text() {
        let conn = test_db();
        // Argon2 hashes always start with "$argon2"
        // bcrypt hashes always start with "$2b$"
        // Neither looks like a plain English password
        let hashes: Vec<String> = {
            let mut stmt = conn
                .prepare("SELECT password_hash FROM users")
                .unwrap();
            stmt.query_map([], |row| row.get(0))
                .unwrap()
                .map(|r| r.unwrap())
                .collect()
        };
        for hash in &hashes {
            assert!(
                hash.starts_with("$argon2") || hash.starts_with("$2b$"),
                "password_hash '{}' does not look like a valid hash — was it stored as plain text?",
                hash
            );
        }
    }

    #[test]
    fn test_user_ids_are_unique() {
        let conn = test_db();
        let total: i64 = conn
            .query_row("SELECT COUNT(*) FROM users", [], |row| row.get(0))
            .unwrap();
        let unique: i64 = conn
            .query_row("SELECT COUNT(DISTINCT id) FROM users", [], |row| row.get(0))
            .unwrap();
        assert_eq!(total, unique, "All user IDs must be unique");
    }
}
```

**Key concepts demonstrated:**
- The `test_db()` helper is the most important pattern in database testing — isolated state per test
- Testing schema existence separately from data existence (if schema fails, you know why)
- Asserting the hash format as a proxy for "password was hashed" — a lightweight but effective check
- Uniqueness assertions catch bugs where UUID generation might accidentally repeat

---

### Tests for `src/models.rs`

Test that your model conversions work correctly, particularly `User` → `UserResponse`.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn sample_user() -> User {
        User {
            id: "test-uuid".to_string(),
            username: "alice".to_string(),
            email: "alice@example.com".to_string(),
            password_hash: "$argon2id$v=19$very_secret_hash".to_string(),
            role: Role::User,
            created_at: "2024-01-01T00:00:00Z".to_string(),
            updated_at: "2024-01-01T00:00:00Z".to_string(),
        }
    }

    #[test]
    fn test_user_response_does_not_contain_password_hash() {
        let user = sample_user();
        let response = UserResponse::from(user); // implement From<User> for UserResponse
        // Serialize to JSON and confirm the hash is absent
        let json = serde_json::to_string(&response).unwrap();
        assert!(
            !json.contains("password_hash"),
            "UserResponse JSON must not contain the field 'password_hash'"
        );
        assert!(
            !json.contains("$argon2"),
            "UserResponse JSON must not contain the hash value"
        );
    }

    #[test]
    fn test_user_response_preserves_id() {
        let user = sample_user();
        let response = UserResponse::from(user);
        assert_eq!(response.id, "test-uuid");
    }

    #[test]
    fn test_user_response_preserves_username() {
        let user = sample_user();
        let response = UserResponse::from(user);
        assert_eq!(response.username, "alice");
    }

    #[test]
    fn test_user_response_preserves_email() {
        let user = sample_user();
        let response = UserResponse::from(user);
        assert_eq!(response.email, "alice@example.com");
    }
}
```

**Why test `UserResponse`?** Because the `password_hash` leak is a catastrophic security bug. A test that explicitly checks the serialized JSON for the hash is a permanent safety net. Even if someone refactors `UserResponse` and accidentally adds the hash back, this test will catch it.

---

### Understanding Test Output

When you run `cargo test`, you will see output like:

```
running 32 tests
test permissions::tests::test_user_role_has_read_own_profile ... ok
test permissions::tests::test_user_role_cannot_delete_any_user ... ok
test auth::password::tests::test_same_password_produces_different_hashes ... ok
...
test result: ok. 32 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

If a test fails, you see the file and line number, the assertion that failed, and the actual vs expected values. This is much more useful than a runtime crash in production.

### A Note on Test-Driven Development (TDD)

Some developers write tests **before** the implementation. The workflow is:

1. Write a test that describes the behaviour you want
2. Run `cargo test` — it fails (the code does not exist yet)
3. Write the minimum code to make the test pass
4. Refactor — clean up the code while keeping the tests green

This is called **Red → Green → Refactor**. You do not have to follow TDD strictly, but writing the test first forces you to think about the API of your code before you write it. Many experienced developers find it leads to better-designed code.

---

## Best Practices Checklist

Go through this before considering the project done.

### Security
- [ ] Passwords are hashed with Argon2 (never plain text, never SHA-256/MD5)
- [ ] JWTs expire (24 hours or less)
- [ ] Login returns the same error message whether the username or password is wrong
- [ ] Forgot-password always returns 200 regardless of whether the email exists
- [ ] Password reset tokens are single-use and time-limited
- [ ] `UserResponse` never includes `password_hash`
- [ ] Admins cannot delete their own account
- [ ] Users cannot self-register as admin

### Rust Code Quality
- [ ] No bare `unwrap()` in handler or middleware code — use `?` and `AppError`
- [ ] Database access is behind `Arc<Mutex<>>` — no shared mutable state without a lock
- [ ] `Role` and `Permission` are enums, not strings
- [ ] Request and response types are separate structs (not the same struct as the DB model)
- [ ] Error types implement `thiserror::Error` and `IntoResponse`

### API Design
- [ ] `DELETE` returns `204 No Content` (no body)
- [ ] Successful `POST /register` returns `201 Created` (not 200)
- [ ] `PATCH` supports partial updates (all fields `Option<T>`)
- [ ] Consistent JSON error format: `{ "error": "..." }`

### Observability
- [ ] `tracing::info!` on server start
- [ ] `tracing::info!` for simulated password reset emails
- [ ] `tracing::error!` on database errors (before returning generic 500 to client)

---

## How to Test Your Endpoints

Use `curl` from the terminal or install a GUI tool like [Insomnia](https://insomnia.rest/) or [Bruno](https://www.usebruno.com/).

### Register a user
```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "email": "alice@example.com", "password": "hunter2abc"}'
```

### Login
```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "hunter2abc"}'
# Save the token from the response
```

### Call an admin endpoint
```bash
curl http://localhost:3000/admin/users \
  -H "Authorization: Bearer <token-from-login>"
```

### Trigger password reset (watch the logs for the token)
```bash
curl -X POST http://localhost:3000/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@example.com"}'
```

### Redeem reset token
```bash
curl -X POST http://localhost:3000/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{"token": "<token-from-log>", "new_password": "newpassword123"}'
```

---

## Glossary

**Argon2** — A modern password hashing algorithm that is intentionally slow and memory-hard to resist GPU-based brute-force attacks.

**bcrypt** — An older password hashing algorithm. Still secure, but Argon2 is preferred for new systems.

**Claim** — A piece of information encoded in a JWT payload, such as the user's ID or role.

**Constant-time comparison** — A comparison operation that always takes the same amount of time regardless of where the inputs differ, preventing timing attacks.

**Extractor (Axum)** — A type that Axum automatically constructs from an incoming request. If construction fails, Axum rejects the request before the handler runs.

**JWT (JSON Web Token)** — A signed, compact token that encodes claims. Used to authenticate requests without hitting the database on every call.

**HMAC** — Hash-based Message Authentication Code. The signing mechanism used by HS256 JWTs.

**Middleware** — Code that runs before (or after) a handler. In Axum, implemented as extractors.

**Partial update** — An update operation where only the supplied fields are changed. Implemented with `Option<T>` fields.

**RBAC (Role-Based Access Control)** — A permission model where users are assigned roles, and roles are granted permissions.

**Salt** — A random value mixed into a password before hashing. Prevents two identical passwords from producing the same hash.

**SQLite** — A file-based (or in-memory) relational database that runs in-process. No server required.

**Timing attack** — An attack where the attacker measures how long an operation takes to infer information about secret values.

**User enumeration** — A vulnerability where an API reveals whether a username or email is registered. Prevented by returning identical errors for "wrong username" and "wrong password".

**UUID** — Universally Unique Identifier. A 128-bit random value used as a primary key. Harder to guess than sequential integers.
