# `rust-musl-builder-china`: 用于编译构建 Rust 静态链接的 Docker 容器

本项目在以下开源项目基础上修改，以适合中国的网络环境。

- This repository is a fork of [nasqueron/rust-musl-builder](https://github.com/nasqueron/rust-musl-builder), opt for China networks.
- Original from [ekidd/rust-musl-builder](https://github.com/emk/rust-musl-builder).

## 为什么需要这个项目？

1. crates-io 连接不稳定，经常下载依赖包失败。
2. Ubuntu官方源速度太慢

## 解决什么问题？

这个项目镜像可用于构建编译 Rust 静态链接的库或可执行文件，这些文件可以在任何现代 Linux 系统上运行，而无需依赖。镜像已经预制了`diesel`，`sqlx`，`openssl`等静态库，可以直接使用。编译出的文件可以直接复制到任何现代 Linux 系统上直接运行，而无须安装依赖其他包。

## 使用方法

全量构建, 用于 CI/CD:

```sh
alias rust-builder='docker run --rm -it -v "$(pwd)":/home/rust/src rust-musl-builder-china'
rust-builder cargo build --release

```

增量构建，用于开发环境中:

```sh
alias rust-builder='docker run --rm -it -v cargo-git:/home/rust/.cargo/git -v cargo-registry:/home/rust/.cargo/registry -v target:/home/rust/src/target -v "$(pwd)":/home/rust/src rust-musl-builder-china'
rust-builder sudo chown -R rust:rust /home/rust/.cargo/git /home/rust/.cargo/registry /home/rust/src/target
rust-builder cargo build --release
```

更多例子可以看 [examples/using-diesel](./examples/using-diesel) 和 [examples/using-sqlx](./examples/using-sqlx).

## Rust 编译结果

编译的二进制文件在 `target/x86_64-unknown-linux-musl/release`, 直接拷贝到 x86_64 的Linux 主机上执行即可. 

In particular, you should be able make static release binaries using TravisCI and GitHub, or you can copy your Rust application into an [Alpine Linux container][]. See below for details!

## How it works

`rust-musl-builder-china` 安装了 [musl-libc][], [musl-gcc][], 和 [rustup][] `target`.  包括了以下库的静态版本: 

- 标准 `musl-libc` 库
- OpenSSL, 被很多库依赖使用
- `libpq`,  被 `diesel` 依赖
- `libz`,  被 `libpq` 依赖
- SQLite3, [examples/using-diesel](./examples/using-diesel/).

## 其他

This image also supports the following extra goodies:

- Basic compilation for `armv7` using `musl-libc`. Not all libraries are supported at the moment, however.
- [`mdbook`][mdbook] and `mdbook-graphviz` for building searchable HTML documentation from Markdown files. Build manuals to use alongside your `cargo doc` output!
- [`cargo about`][about] to collect licenses for your dependencies.
- [`cargo deb`][deb] to build Debian packages
- [`cargo deny`][deny] to check your Rust project for known security issues.

## Making OpenSSL work

If your application uses OpenSSL, you will also need to take a few extra steps to make sure that it can find OpenSSL's list of trusted certificates, which is stored in different locations on different Linux distributions. You can do this using [`openssl-probe`](https://crates.io/crates/openssl-probe) as follows:

```rust
fn main() {
    openssl_probe::init_ssl_cert_env_vars();
    //... your code
}
```

## Making Diesel work

In addition to setting up OpenSSL, you'll need to add the following lines to your `Cargo.toml`:

```toml
[dependencies]
diesel = { version = "1", features = ["postgres", "sqlite"] }

# Needed for sqlite.
libsqlite3-sys = { version = "*", features = ["bundled"] }

# Needed for Postgres.
openssl = "*"
```

For PostgreSQL, you'll also need to include `diesel` and `openssl` in your `main.rs` in the following order (in order to avoid linker errors):

```rust
extern crate openssl;
#[macro_use]
extern crate diesel;
```

If this doesn't work, you _might_ be able to fix it by reversing the order. See [this PR](https://github.com/emk/rust-musl-builder/issues/69) for a discussion of the latest issues involved in linking to `diesel`, `pq-sys` and `openssl-sys`.

## Making static releases with Travis CI and GitHub

These instructions are inspired by [rust-cross][].

First, read the [Travis CI: GitHub Releases Uploading][uploading] page, and run `travis setup releases` as instructed.  Then add the following lines to your existing `.travis.yml` file, replacing `myapp` with the name of your package:

```yaml
language: rust
sudo: required
os:
- linux
- osx
rust:
- stable
services:
- docker
before_deploy: "./build-release myapp ${TRAVIS_TAG}-${TRAVIS_OS_NAME}"
deploy:
  provider: releases
  api_key:
    secure: "..."
  file_glob: true
  file: "myapp-${TRAVIS_TAG}-${TRAVIS_OS_NAME}.*"
  skip_cleanup: true
  on:
    rust: stable
    tags: true
```

Next, copy [`build-release`](./examples/build-release) into your project and run `chmod +x build-release`.

Finally, add a `Dockerfile` to perform the actual build:

```Dockerfile
FROM nasqueron/rust-musl-builder

# We need to add the source code to the image because `rust-musl-builder`
# assumes a UID of 1000, but TravisCI has switched to 2000.
ADD --chown=rust:rust . ./

CMD cargo build --release
```

When you push a new tag to your project, `build-release` will automatically build new Linux binaries using `rust-musl-builder`, and new Mac binaries with Cargo, and it will upload both to the GitHub releases page for your repository.

For a working example, see [faradayio/cage][cage].

[rust-cross]: https://github.com/japaric/rust-cross
[uploading]: https://docs.travis-ci.com/user/deployment/releases
[cage]: https://github.com/faradayio/cage

## Making tiny Docker images with Alpine Linux and Rust binaries

Docker now supports [multistage builds][multistage], which make it easy to build your Rust application with `rust-musl-builder` and deploy it using [Alpine Linux][]. For a working example, see [`examples/using-diesel/Dockerfile`](./examples/using-diesel/Dockerfile).

[multistage]: https://docs.docker.com/engine/userguide/eng-image/multistage-build/
[Alpine Linux]: https://alpinelinux.org/

## Adding more C libraries

If you're using Docker crates which require specific C libraries to be installed, you can create a `Dockerfile` based on this one, and use `musl-gcc` to compile the libraries you need. For an example, see [`examples/adding-a-library/Dockerfile`](./examples/adding-a-library/Dockerfile). This usually involves a bit of experimentation for each new library, but it seems to work well for most simple, standalone libraries.

If you need an especially common library, please feel free to submit a pull request adding it to the main `Dockerfile`!  We'd like to support popular Rust crates out of the box.

## Development notes

After modifying the image, run `./test-image` to make sure that everything works.

## Other ways to build portable Rust binaries

If for some reason this image doesn't meet your needs, there's a variety of other people working on similar projects:

- [messense/rust-musl-cross](https://github.com/messense/rust-musl-cross) shows how to build binaries for many different architectures.
- [japaric/rust-cross](https://github.com/japaric/rust-cross) has extensive instructions on how to cross-compile Rust applications.
- [clux/muslrust](https://github.com/clux/muslrust) also supports libcurl.
- [golddranks/rust_musl_docker](https://github.com/golddranks/rust_musl_docker). Another Docker image.

## License

Either the [Apache 2.0 license](./LICENSE-APACHE.txt), or the
[MIT license](./LICENSE-MIT.txt).

[Alpine Linux container]: https://hub.docker.com/_/alpine/
[about]: https://github.com/EmbarkStudios/cargo-about
[deb]: https://github.com/mmstick/cargo-deb
[deny]: https://github.com/EmbarkStudios/cargo-deny
[mdbook]: https://github.com/rust-lang-nursery/mdBook
[musl-libc]: http://www.musl-libc.org/
[musl-gcc]: http://www.musl-libc.org/how.html
[rustup]: https://www.rustup.rs/
