# axum-reject-macro

A derive macro that turns your error enums into axum responses. Annotate each variant with a status code and error message, and the macro implements `IntoResponse` for you.

## Usage

Add the dependency to your `Cargo.toml`:

```toml
[dependencies]
axum-reject-macro = { git = "https://github.com/CelesteLove/axum-reject-macro" }
```

Then derive `HttpError` on any enum. Each variant needs an `#[http_error(...)]` attribute specifying the HTTP `status` code and a JSON `message`:

```rust
use axum_reject_macro::HttpError;

#[derive(Debug, HttpError)]
pub enum AppError {
    #[http_error(status = 404, message = "not_found")]
    NotFound,

    #[http_error(status = 500, message = "internal_error")]
    Internal(String),
}
```

This generates an `IntoResponse` implementation that returns the given status code with a JSON body like `{"error": "not_found"}`.

## With `thiserror`

You can use `HttpError` to handle the response side and `thiserror` to implement `Display`/`Error`. They are a great fit for each other:

```rust
use axum_reject_macro::HttpError;
use thiserror::Error;

#[derive(Debug, Error, HttpError)]
pub enum LoginError {
    #[http_error(status = 401, message = "invalid_credentials")]
    #[error("invalid credentials")]
    InvalidCredentials(#[from] VerificationError),

    #[http_error(status = 500, message = "internal_error")]
    #[error("database error")]
    Database(#[from] sqlx::Error),

    #[http_error(status = 400, message = "bad_request")]
    #[error("bad request: {0}")]
    BadRequest(String),
}
```

Now `LoginError` implements both `std::error::Error` (for logging, `?` propagation, error chains) and `IntoResponse` (for returning structured HTTP errors from handlers).

## How it works

The macro generates a `match` over your enum that pairs each variant with its annotated status code and message. A variant like:

```rust
#[http_error(status = 401, message = "invalid_credentials")]
InvalidCredentials(#[from] VerificationError),
```

produces a match arm equivalent to:

```rust
Self::InvalidCredentials(_) => {
    (StatusCode::from_u16(401).unwrap(), r#"{"error": "invalid_credentials"}"#.to_string())
        .into_response()
}
```

Unit variants work the same way, my macro can handle both.

## License

Licensed under either of

- [Apache License, Version 2.0](LICENSE-APACHE)
- [MIT License](LICENSE-MIT)

at your option.
