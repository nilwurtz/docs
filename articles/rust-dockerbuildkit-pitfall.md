---
title: "Rust のアプリで --mount=type=cache + multi-stage build してハマった"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Docker", "rust"]
published: true
---

# 要約

Multi-stage build の Dockerfile において、Buildkit の`--mount=type=cache`で成果物のできるディレクトリをキャッシュする

↓

後の`COPY`で成果物が変わったと見なされず、レイヤーキャッシュが使われてしまい、Build してもアプリが変わらない

※ Rust だけ起こるというわけではないです

## 環境

```
Docker version 20.10.12, build e91ed57
github.com/moby/buildkit 8142d66b5ebde79846b869fba30d9d30633e74aa # v0.8.1
```

## 症状

- rust アプリを Skaffold で開発したい
- Dockerfile は Cargo-chef を利用した multi-stage build で作っていた
- `cargo build` が遅いので、Buildkit の `--mount=type=cache,target=/app/target` した

→ 何回ビルドしてもアプリが変わらない・・・😭

:::details Dockerfile

```dockerfile
FROM rust:1.59.0-slim-buster AS chef
RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release

FROM debian:buster-slim
WORKDIR /app
COPY --from=builder /app/target/release/rest api
CMD [ "./api" ]
```

:::

レイヤーキャッシュが使われている様子

```log
Watching for changes...
Generating tags...
 - docker.io/library/api -> docker.io/library/api:20220326_233739
Checking cache...
 - docker.io/library/api: Not found. Building
Starting build...
Found [minikube] context, using local docker daemon.
Building [docker.io/library/api]...
Target platforms: [linux/amd64]
[+] Building 10.0s (17/17) FINISHED
 => [internal] load build definition from Dockerfile                           0.0s
 => => transferring dockerfile: 727B                                           0.0s
 => [internal] load .dockerignore                                              0.0s
 => => transferring context: 34B                                               0.0s
 => [internal] load metadata for docker.io/library/debian:buster-slim          1.4s
 => [internal] load metadata for docker.io/library/rust:1.59.0-slim-buster     1.5s
 => [internal] load build context                                              0.1s
 => => transferring context: 128.70kB                                          0.0s
 => [stage-3 1/3] FROM docker.io/library/debian:buster-slim@sha256:bbf8ca5a94  0.0s
 => [chef 1/3] FROM docker.io/library/rust:1.59.0-slim-buster@sha256:77c9eda3  0.0s
 => CACHED [chef 2/3] RUN cargo install cargo-chef                             0.0s
 => CACHED [chef 3/3] WORKDIR /app                                             0.0s
 => [planner 1/2] COPY . .                                                     0.5s
 => [planner 2/2] RUN cargo chef prepare --recipe-path recipe.json             0.2s
 => CACHED [builder 1/3] COPY --from=planner /app/recipe.json recipe.json      0.0s
 => [builder 2/3] COPY . .                                                     1.2s
 => [builder 3/3] RUN --mount=type=cache,target=/usr/local/cargo/registry      6.4s
--- ↑↑ cacheを利用したBuildは走っている ---
 => CACHED [stage-3 2/3] WORKDIR /app                                          0.0s
 => CACHED [stage-3 3/3] COPY --from=builder /app/target/release/rest api  0.0s
--- ↑↑ レイヤーキャッシュが利用されている ---
 => exporting to image                                                         0.0s
 => => exporting layers                                                        0.0s
 => => writing image sha256:9e0aa125deb2d754244339b65dd60e2cb0213484bb722ff77  0.0s
 => => naming to docker.io/library/api:20220326_233739                     0.0s
Build [docker.io/library/api] succeeded
```

Docker の`COPY`命令は対象のハッシュ値を見て、レイヤーキャッシュを利用するか判断する。([documentation](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache))
そのため、成果物が変わっていればレイヤーキャッシュは利用されないはず。

`cargo build`は先に流れているように見えるが、`COPY`時は Cache の`/app/target`を見ているような挙動をしている・・・。

## 解決

成果物を Cache 対象ディレクトリから移動させた

```dockerfile
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release && mv target/release/rest /tmp/rest

FROM debian:buster-slim
WORKDIR /app
COPY --from=builder /tmp/rest api
```

## 🙁

Rust 以外でも全然起こりうる現象なので気をつけましょう。
