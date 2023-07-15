---
title: "docker-composeだけで全てのコンテナ起動を待機する"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["docker", "dockercompose"]
published: false
---

# 概要

`docker-compose up -d` で、本当に全てのコンテナが起動し終わるまで Compose 内で完結するように待機する。

## HealthCheck

docker-compose は`depends_on`を用いて起動順を指定できるが、本当に順番のみを指定するオプションであり、特定の状態になるまで待機 ということを行うには Docker の[HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#healthcheck)を利用する。

HEALTHCHECK はコンテナ内で指定したコマンドを実行してコンテナの健康状態をチェックし、コンテナには**health status**(_starting_,_healthy_,_unhealthy_)が設定される。
docker-compose からでもこの機能を利用して、**db が起動してから API を起動する**ということができる。

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - 5432:5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      retries: 10
      timeout: 10s
  api:
    image: python:3
    command: python -m http.server 8888
    depends_on:
      postgres:
        condition: service_healthy
```

これで一見完璧に見える。

## 落とし穴

問題は API のコンテナ起動コマンドの実行に時間がかかる場合である。（例えば、コンテナの起動時に DB のマイグレーションを流すなど。）
CI などで完全に起動し切る前に `docker-compose up -d` が終了してしまう。

### docker-compose up の流れ

```
postgresが起動する
↓
postgresのCMD(or ENTRYPOINT)命令が実行される
↓
postgresのhealthcheckが成功する
↓
apiが起動する
↓
apiのCMD(or ENTRYPOINT)命令が実行される ← ここでdocker-compose up -dが終了してしまう
↓
apiのhealthcheckが成功する
```

これは API の HealthCheck を加えればいいのでは？と思うかもしれないが、`docker-compose up -d`した場合、最後のコンテナだけは HealthCheck が成功するまで待機することはしてくれない・・・。

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - 5432:5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      retries: 10
      timeout: 10s
  api:
    image: python:3
    # Commandが完了しサーバーが立ち上がるまで若干の時間がかかる想定
    command: /bin/bash -c "sleep 1 && python -m http.server 8888"
    # healthcheckを追加
    healthcheck:
      test: ["CMD-SHELL", "curl http://localhost:8888/ || exit 1"]
      interval: 5s
      retries: 10
      timeout: 10s
    depends_on:
      postgres:
        condition: service_healthy
```

※ `docker-compose up -d` が終了した瞬間には API が _starting_ な様子。

```shell
❯ docker-compose up -d && docker-compose ps
Creating network "compose-health-check_default" with the default driver
Creating compose-health-check_postgres_1 ... done
Creating compose-health-check_api_1      ... done
             Name                            Command                      State                   Ports
----------------------------------------------------------------------------------------------------------------
compose-health-check_api_1        python -m http.server 8888      Up (health: starting)
compose-health-check_postgres_1   docker-entrypoint.sh postgres   Up (healthy)            0.0.0.0:5432->5432/tcp
```

# 解決策

CI 上などでは待機用の Shell スクリプト（API に対して一定時間 Curl し続けるなど）を書いておいて、`docker-compose up -d`の後に実行するというのが一番手っ取り早いと思う。

しかし「Compose だけで完結したい・・・・」という人には、こんなやり方がある。

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - 5432:5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
      interval: 5s
      retries: 10
      timeout: 10s
  api:
    image: python:3
    command: /bin/bash -c "sleep 1 && python -m http.server 8888"
    healthcheck:
      test: ["CMD-SHELL", "curl http://localhost:8888/ || exit 1"]
      interval: 5s
      retries: 10
      timeout: 10s
    depends_on:
      postgres:
        condition: service_healthy
  # HealthCheck用のコンテナを追加
  healthchecker:
    image: busybox
    depends_on:
      api:
        condition: service_healthy
```

### 結果

```shell
❯ docker-compose up -d && docker-compose ps
Creating network "compose-health-check_default" with the default driver
Creating compose-health-check_postgres_1 ... done
Creating compose-health-check_api_1      ... done
Creating compose-health-check_healthchecker_1 ... done
                Name                              Command                  State               Ports
-------------------------------------------------------------------------------------------------------------
compose-health-check_api_1             /bin/bash -c sleep 1 && py ...   Up (healthy)
compose-health-check_healthchecker_1   sh                               Exit 0
compose-health-check_postgres_1        docker-entrypoint.sh postgres    Up (healthy)   0.0.0.0:5432->5432/tcp
```

### この時の docker-compose up の流れ

```
postgresが起動する
↓
postgresのCMD(or ENTRYPOINT)命令が実行される
↓
postgresのhealthcheckが成功する
↓
apiが起動する
↓
apiのCMD(or ENTRYPOINT)命令が実行される
↓
apiのhealthcheckが成功する
↓
healthcheckerが起動する
↓
healthcheckerのCMD(or ENTRYPOINT)命令が実行される ← ここでdocker-compose up -dが終了する
```

若干無駄なコンテナが増えるのが気になる人もいると思うが、全てのコンテナが起動した後に`docker-compose up -d`が終了するので、CI 上などでの利用を想定しているなら有用だと思う。
