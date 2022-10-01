## Creating a new Spin application from a template

Spin helps you create a new application based on templates:

```console
$ spin templates list
You have no templates installed. Run
spin templates install --git https://github.com/fermyon/spin
to install a starter set.
```

We first need to configure the [templates from the Spin repository](https://github.com/fermyon/spin/tree/main/templates):

```console
$ spin templates install --git https://github.com/fermyon/spin
Copying remote template source
Installing template redis-rust...
Installing template http-rust...
Installing template http-go...
+--------------------------------------------------+
| Name         Description                         |
+==================================================+
| http-go      HTTP request handler using (Tiny)Go |
| http-rust    HTTP request handler using Rust     |
| redis-rust   Redis message handler using Rust    |
| ...                                              |
+--------------------------------------------------+
```

Let's create a new Spin application based on the Rust HTTP template:

```console
$ spin new http-rust spin-hello-world
Project description: A simple Spin HTTP component in Rust
HTTP base: /
HTTP path: /hello
$ tree
├── .cargo
│   └── config.toml
├── .gitignore
├── Cargo.toml
├── spin.toml
└── src
    └── lib.rs
```

This command created all the necessary files we need to build and run our first
Spin application. Here is `spin.toml`, the manifest file for a Spin application:

```toml
spin_version = "1"
description = "A simple Spin HTTP component in Rust"
name = "spin-hello-world"
trigger = { type = "http", base = "/" }
version = "0.1.0"

[[component]]
id = "spin-hello-world"
source = "target/wasm32-wasi/release/spin_hello_world.wasm"
[component.trigger]
route = "/hello"
[component.build]
command = "cargo build --target wasm32-wasi --release"
```

This represents a simple Spin HTTP application (triggered by an HTTP request), with
a single component called `spin-hello-world`. Spin will execute the `spin_hello_world.wasm`
WebAssembly module for HTTP requests on the route `/hello`.
(See the [configuration document](./configuration.md) for a detailed guide on the Spin
application manifest.)

Now let's have a look at the code. Below is the complete source
code for a Spin HTTP component written in Rust — a regular Rust function that
takes an HTTP request as a parameter and returns an HTTP response, and it is
annotated with the `http_component` macro:

```rust
use anyhow::Result;
use spin_sdk::{
    http::{Request, Response},
    http_component,
};

/// A simple Spin HTTP component.
#[http_component]
fn spin_hello_world(req: Request) -> Result<Response> {
    println!("{:?}", req.headers());
    Ok(http::Response::builder()
        .status(200)
        .header("foo", "bar")
        .body(Some("Hello, Fermyon".into()))?)
}
```

We can build this component using the regular Rust toolchain, targeting
`wasm32-wasi`, which will produce the WebAssembly module and place it at
`target/wasm32-wasi/release/spinhelloworld.wasm` as referenced in the
`spin.toml`. For convenience, we can use the `spin build` command, which will
execute the command defined above in `spin.toml` and call the Rust toolchain:

```console
$ spin build
Executing the build command for component spin-hello-world: cargo build --target wasm32-wasi --release
   Compiling spin_hello_world v0.1.0
    Finished release [optimized] target(s) in 0.10s
Successfully ran the build command for the Spin components.
```

If you run into errors, you might want to use `rustup check` to see if your Rust installation is up-to-date.

## Running the application with `spin up`

Now that we configured the application and built our component, we can _spin up_
the application (pun intended):

```bash
$ spin up
Serving HTTP on address http://127.0.0.1:3000
Available Routes:
  spin-hello-world: http://127.0.0.1:3000/hello
```

Optionally, set the RUST_LOG environment variable for detailed logs, before running `spin up`.

```bash
$ export RUST_LOG=spin=trace
```

Spin will instantiate all components from the application manifest, and
will create the router configuration for the HTTP trigger accordingly. The
component can now be invoked by making requests to `http://localhost:3000/hello`
(see route field in the configuration):

Note that Codespaces prompts you because a port was exposed and forwarded automatically

```
$ curl -i localhost:3000/hello
HTTP/1.1 200 OK
foo: bar
content-length: 15

Hello, Fermyon!
```