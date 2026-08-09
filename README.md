# rust-http-controllers

A skill-building project for learning how to build a REST API in Rust from scratch. Covers authentication, authorization, middleware, password security, SQLite database access, and unit testing using best practices.

## What This Project Builds

A fully functional HTTP API with:

- User registration and login (JWT-based)
- Password reset via email token
- Role-based access control (RBAC) with `User` and `Admin` roles
- Admin endpoints for managing users
- Argon2 password hashing
- Axum extractor-based middleware for auth enforcement
- In-memory SQLite database with seed data (no server setup required)
- Unit tests for permissions, password hashing, JWT tokens, error mapping, DB schema, and models

## Prerequisites

- [Rust](https://rustup.rs/) (stable, 1.85+)
- No external services required — SQLite runs in-process

## Getting Started

```bash
git clone https://github.com/your-username/rust-http-controllers
cd rust-http-controllers
cargo run
```

The server starts on `http://localhost:3000`.

Set the log level with the `RUST_LOG` environment variable:

```bash
RUST_LOG=debug cargo run   # verbose
RUST_LOG=info cargo run    # standard
```

## Running Tests

```bash
cargo test                      # run all tests
cargo test test_permissions     # run tests whose name contains "test_permissions"
cargo test -- --nocapture       # show log output during tests
```

Tests are co-located with the code they test inside `#[cfg(test)]` modules. No external services are required — database tests use an in-memory SQLite instance spun up per test.

## Endpoints

### Auth (Public)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/auth/register` | Register a new user |
| `POST` | `/auth/login` | Login and receive a JWT |
| `POST` | `/auth/forgot-password` | Request a password reset email |
| `POST` | `/auth/reset-password` | Redeem a reset token and set a new password |

### Admin (Requires `Admin` role JWT)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/admin/users` | List all users |
| `PATCH` | `/admin/users/:id` | Update a user's details |
| `POST` | `/admin/users/:id/reset-password` | Force-reset a user's password |
| `DELETE` | `/admin/users/:id` | Delete a user |

## Authentication

After a successful login, include the returned JWT in the `Authorization` header on protected requests:

```
Authorization: Bearer <token>
```

Tokens expire after 24 hours.

## Project Structure

```
src/
├── main.rs              # Entry point, routing, app state
├── db.rs                # SQLite setup, schema, seed data
├── models.rs            # Structs for DB rows and HTTP payloads
├── permissions.rs       # Role and Permission enums
├── middleware.rs        # Auth and admin extractors
├── error.rs             # Unified AppError type
├── auth/
│   ├── mod.rs
│   ├── password.rs      # Argon2 hashing and verification
│   └── token.rs         # JWT creation and validation
└── handlers/
    ├── mod.rs
    ├── auth.rs          # Public auth endpoints
    └── admin.rs         # Admin-only endpoints
```

## Guide

See [GUIDE.md](./GUIDE.md) for a detailed, step-by-step walkthrough of the entire implementation — including the rationale behind every design decision, unit test examples for each module, and a best practices checklist. Aimed at junior developers.

## License

MIT
