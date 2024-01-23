# `rust-musl-builder-china`: 用于编译构建 Rust 静态链接的 Docker 容器

本项目在以下开源项目基础上修改，以适合中国的网络环境。

- This repository is a fork of [nasqueron/rust-musl-builder](https://github.com/nasqueron/rust-musl-builder), opt for China networks.
- Original from [ekidd/rust-musl-builder](https://github.com/emk/rust-musl-builder).

## 为什么需要这个项目？

1. crates-io 连接不稳定，经常下载依赖包失败。
2. Ubuntu官方源速度太慢

## 解决什么问题？

这个项目镜像可用于构建编译 Rust 静态链接的库或可执行文件，这些文件可以在任何现代 Linux 系统上运行，而无需依赖。镜像已经预制了`diesel`，`sqlx`，`openssl`等静态库，可以直接使用。编译出的文件可以直接复制到任何现代 Linux 系统上直接运行，而无须安装其他依赖包。

## 使用方法

全量构建, 用于 CI/CD:

```sh
alias rust-builder='docker run --rm -it -v "$(pwd)":/home/rust/src vebodev/rust-musl-builder-china'
rust-builder cargo build --release

```

增量构建，用于开发环境中:

```sh
alias rust-builder='docker run --rm -it -v cargo-git:/home/rust/.cargo/git -v cargo-registry:/home/rust/.cargo/registry -v "$(pwd)":/home/rust/src rust-musl-builder-china'
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

还支持了以下基础包的能力：

- 通过 `musl-libc` 支持 `armv7` 编译。但是还没有支持全部库。
- [`mdbook`][mdbook] 和 `mdbook-graphviz`  用来从 Markdown 生成 HTML 文档。 执行 `cargo doc` 即可!
- [`cargo about`][about] 用于收集依赖库的 License 信息.
- [`cargo deb`][deb] 创建 Debian 包 
- [`cargo deny`][deny] 扫描 Rust 工程中是否有安全漏洞.
- [`cargo watch`][watch] 用于监控文件变化，自动编译.

## 如何在项目中使用 OpenSSL ?

当 Rust 项目使用 OpenSSL时，还需要额外的步骤，以便让 OpenSSL 能够找到系统的证书列表。不同的 Linux 发行版，证书列表的存储位置不同。可以使用 [`openssl-probe`](https://crates.io/crates/openssl-probe) 来解决这个问题:

```rust
fn main() {
    openssl_probe::init_ssl_cert_env_vars();
    //... your code
}
```

## 编译使用 Diesel 项目

除了上述设置 OpenSSL，还需要配置 `Cargo.toml`:

```toml
[dependencies]
diesel = { version = "1", features = ["postgres", "sqlite"] }

# 当使用 SQLite3 时
libsqlite3-sys = { version = "*", features = ["bundled"] }

# 当使用 Postgres 时 
openssl = "*"
```

使用 PostgreSQL 时，还需要在 `main.rs` 中添加 `diesel` 和 `openssl`，并且需要按照以下顺序，以避免链接错误:

```rust
extern crate openssl;
#[macro_use]
extern crate diesel;
```

 如果遇到编译错误，可以尝试调整顺序。参考 [this PR](https://github.com/emk/rust-musl-builder/issues/69).

## 在Travis CI 和 GitHub 编译静态链接的二进制文件

参考自 [rust-cross][].

首先, 阅读 [Travis CI: GitHub Releases Uploading][uploading] 页面，按照指示运行 `travis setup releases`。然后在你的 `.travis.yml` 文件中添加以下行，将 `myapp` 替换为你的包名:

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

然后, 在项目中添加一个 `build-release` 脚本，用于构建 Linux 和 Mac 版本的二进制文件:

最后, 添加一个 `Dockerfile` 用于执行实际的构建:

```Dockerfile
FROM vebodev/rust-musl-builder-china

# We need to add the source code to the image because `rust-musl-builder`
# assumes a UID of 1000, but TravisCI has switched to 2000.
ADD --chown=rust:rust . ./

CMD cargo build --release
```

当你推送一个新的 tag 到你的项目时，`build-release` 会自动使用 `rust-musl-builder` 构建新的 Linux 二进制文件，使用 Cargo 构建 Mac 二进制文件，并且会自动上传到 GitHub releases 页面。

例子在 [faradayio/cage][cage].

[rust-cross]: https://github.com/japaric/rust-cross
[uploading]: https://docs.travis-ci.com/user/deployment/releases
[cage]: https://github.com/faradayio/cage

## 基于 Alpine Linux 构建小空间的 Docker 容器

Docker 支持了 [multistage builds][multistage]，可以很容易的使用 `rust-musl-builder` 构建 Rust 应用，并且使用 [Alpine Linux][] 部署。参考 [`examples/using-diesel/Dockerfile`](./examples/using-diesel/Dockerfile).

[multistage]: https://docs.docker.com/engine/userguide/eng-image/multistage-build/
[Alpine Linux]: https://alpinelinux.org/

## 增加其他 C 语言的库

 如果你的项目还依赖了其他 C 语言的库，可以在 `Dockerfile` 中，增加 `musl-gcc` 编译库的步骤。例子：[`examples/adding-a-library/Dockerfile`](./examples/adding-a-library/Dockerfile). 编译每个库的情况可能略有不同，需要按实际情况调整。

也欢迎提交 PR 把你用到的三方库合并到这个 Dockerfile 中。

## 开发必读

开发修改后，可以执行 `./test-image` 来测试.

## 参考资料

- [nasqueron/rust-musl-builder](https://github.com/nasqueron/rust-musl-builder)
- [ekidd/rust-musl-builder](https://github.com/emk/rust-musl-builder).
- [messense/rust-musl-cross](https://github.com/messense/rust-musl-cross) 
- [japaric/rust-cross](https://github.com/japaric/rust-cross) 
- [clux/muslrust](https://github.com/clux/muslrust) 
- [golddranks/rust_musl_docker](https://github.com/golddranks/rust_musl_docker)

## License

[Apache 2.0 license](./LICENSE-APACHE.txt), 或
[MIT license](./LICENSE-MIT.txt).

[Alpine Linux container]: https://hub.docker.com/_/alpine/
[about]: https://github.com/EmbarkStudios/cargo-about
[deb]: https://github.com/mmstick/cargo-deb
[deny]: https://github.com/EmbarkStudios/cargo-deny
[mdbook]: https://github.com/rust-lang-nursery/mdBook
[musl-libc]: http://www.musl-libc.org/
[musl-gcc]: http://www.musl-libc.org/how.html
[rustup]: https://www.rustup.rs/
