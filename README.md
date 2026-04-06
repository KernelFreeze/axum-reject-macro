# axum-reject-macro

A derive macro that turns your error enums into axum responses. Annotate each variant with a status code and error message, and the macro implements `IntoResponse` for you.

## Usage

Add the dependency to your `Cargo.toml`:

```toml
[dependencies]
axum-reject-macro = { git = "https://github.com/CelesteLove/axum-reject-macro" }
```

Then derive `HttpError` on any enum. Each variant needs an `#[http_error(...)]` attribute specifying the HTTP `status` (using the `StatusCode` constant name) and a JSON `message`:

```rust
use axum_reject_macro::HttpError;

#[derive(Debug, HttpError)]
pub enum AppError {
    #[http_error(status = NOT_FOUND, message = "not_found")]
    NotFound,

    #[http_error(status = INTERNAL_SERVER_ERROR, message = "internal_error")]
    Internal(String),
}
```

The `status` value must be a valid `axum::http::StatusCode` constant name (e.g. `NOT_FOUND`, `BAD_REQUEST`, `UNAUTHORIZED`). Invalid names will produce a compile-time error.

This generates an `IntoResponse` implementation that returns the given status code with a JSON body like `{"error": "not_found"}`.

## With `thiserror`

You can use `HttpError` to handle the response side and `thiserror` to implement `Display`/`Error`. They are a great fit for each other:

```rust
use axum_reject_macro::HttpError;
use thiserror::Error;

#[derive(Debug, Error, HttpError)]
pub enum LoginError {
    #[http_error(status = UNAUTHORIZED, message = "invalid_credentials")]
    #[error("invalid credentials")]
    InvalidCredentials(#[from] VerificationError),

    #[http_error(status = INTERNAL_SERVER_ERROR, message = "internal_error")]
    #[error("database error")]
    Database(#[from] sqlx::Error),

    #[http_error(status = BAD_REQUEST, message = "bad_request")]
    #[error("bad request: {0}")]
    BadRequest(String),
}
```

Now `LoginError` implements both `std::error::Error` (for logging, `?` propagation, error chains) and `IntoResponse` (for returning structured HTTP errors from handlers).

## How it works

The macro generates a `match` over your enum that pairs each variant with its annotated status code and message. A variant like:

```rust
#[http_error(status = UNAUTHORIZED, message = "invalid_credentials")]
InvalidCredentials(#[from] VerificationError),
```

produces a match arm equivalent to:

```rust
Self::InvalidCredentials(_) => {
    (StatusCode::UNAUTHORIZED, r#"{"error": "invalid_credentials"}"#.to_string())
        .into_response()
}
```

Unit variants work the same way, my macro can handle both.

## License

Licensed under either of

- [Apache License, Version 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)

at your option.
