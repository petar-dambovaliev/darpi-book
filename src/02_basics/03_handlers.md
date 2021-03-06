# Handlers

The `handler` macro has 3 optional arguments in a json like format.
Trailing commas are not supported as of now.
```rust
#[handler({
    container: Container,
    middleware: {
        request: [],
        response: []
    },
    jobs: {
        request: [],
        response: []
    }
})]
```

A request handler is an async function that accepts zero or more arguments.
The arguments can be provided in several ways, by macro attributes.
Handlers bound by a `GET` method do not have access to a request body.
The return type must `impl darpi::response::Responder`. For convenience, it is implemented for all common types.

## possible arguments

- dependency injection container
  - `#[inject] my_arg: Arc<dyn SomeTrait>`
- request
  - `#[request] r: darpi::Request<darpi::Body>` consumes the entire request therefore prevents other attributes from being used except `#[path]` 
  - `#[request_parts] rp: &darpi::RequestParts`
  - `#[query] q: MyStruct` where `MyStruct` has to implement
    `serde::Deserialize`. Furthermore, if wrapped in an `Option` it is not mandatory.
  - `#[path] path: MyStruct` where `MyStruct` has to implement
    `serde::Deserialize`. `/user/{user_id}/article/{article_id}` will deserialize both ids into `MyStruct`.
  - `#[body] data: impl FromRequestBody<T: serde:Deserialize>` if handler is not linked to a `GET` request
- middleware
  - `#[middleware::request(i) my_arg: T]` where `i` is a literal index of the middleware linked to the handler
  - `#[middleware::response(i) my_arg: T]` does not exist (obviously), because response middleware runs after the handler returns


While i recommend using the macros. If you really want to, you could implement the handler trait.

```rust
use darpi_middleware::body_size_limit;
use darpi::{app, handler, response::Responder, Method, Path, Json, Query};
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize, Debug, Path, Query)]
pub struct Name {
    name: String,
}

#[handler({
    middleware: {
        // roundtrip returns Result<String, Error>
        // later we can access it via #[middleware::request(0)]
        request: [roundtrip("blah")]
    }
})]
async fn do_something(
  #[request_parts] _rp: &RequestParts,
  // the request query is deserialized into Name
  // if deseriliazation fails, it will result in an error response
  // to make it optional wrap it in an Option<Name>
  #[query] query: Name,
  // the request path is deserialized into Name
  #[path] path: Name,
  // the request body is deserialized into the struct Name
  // it is important to mention that the wrapper around Name
  // should implement darpi::request::FromRequestBody
  // Common formats like Json, Xml and Yaml are supported out
  // of the box but users can implement their own
  #[body] payload: Json<Name>,
  // we can access the T from Ok(T) in the middleware result
  #[middleware::request(0)] m_str: String, // returning a String works because darpi has implemented
  // the Responder trait for common types
) -> String {
  format!(
    "query: {:#?} path: {} body: {} middleware: {}",
    query, path.name, payload.name, m_str
  )
}
```

An example returning a Response object.
For more information visit

[response](https://docs.rs/http/0.2.3/http/response/struct.Response.html)

[status codes](https://docs.rs/http/0.2.3/http/status/struct.StatusCode.html#associatedconstant.SEE_OTHER)

[body](https://docs.rs/hyper/0.14.5/hyper/struct.Body.html)

```rust
use darpi::{handler, Body, Response, StatusCode};

#[handler]
async fn my_handler() -> Response<Body> {
    Response::builder()
        .status(StatusCode::OK)
        .body(Default::default())
        .unwrap()
}
```
