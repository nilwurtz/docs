---
title: "Wiremock の GraphQL Extension を作って公開して、Wiremock Organization に仲間入りした"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wiremock", "e2e", "graphql"]
published: true
---

# 概要

E2E などで API を Mock するツールとして、[Wiremock](https://wiremock.org/)がある。
Wiremock は Url、QueryParam、RequestBody などの Matching が充実しており、普通の REST API の Mock には十分使える。

ただ、GraphQL の Mock を作成する場合、Wiremock の Matching の機能だけでは GraphQL の Query の Matching がつらいという問題がある。
今回はその問題を解決するために、Wiremock の GraphQL 向け Extension を作成し、Wiremock Organization に仲間入りしたのでその紹介。

つくった --> [Wiremock GraphQL Extension](https://github.com/wiremock/wiremock-graphql-extension)

## GraphQL の Query の Matching がつらい

業務の中で、GraphQL の外部 API を利用することになり、これも Wiremock で Mock しテストを記述していた。
GraphQL の Query は、HTTP の POST で送信するので、Wiremock の RequestBody の Matching を使って Matching することができるので、最初はあまり問題ないように思えていたが、次第につらみが出てきた。

以下のような Query に対して、Mock を作成するとする。

```graphql
query {
  hero {
    name
  }
}
```

これを Wiremock の RequestBody の Matching で記述すると、以下のようになる。
(jsonPath の Matching なども利用できる)

```json
{
  "request": {
    "method": "POST",
    "url": "/graphql",
    "bodyPatterns": [
      {
        "equalToJson": "{\"query\":\"query { hero { name } }\"}"
      }
    ]
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "data": {
        "hero": {
          "name": "Luke Skywalker"
        }
      }
    }
  }
}
```

このように、GraphQL の Query を Matching するために、Query の文字列をそのまま一致させる必要がある。
そのため、スペース一つ、改行一つでも違うと、Matching に失敗してしまうことになり、これにずっと苦しんでいた。

```json
{
  "request": {
    "method": "POST",
    "url": "/graphql",
    "bodyPatterns": [
      {
        // スペースが一つ多いだけで、Matchingしなくなってしまう
        "equalToJson": "{\"query\":\"query { hero {  name } }\"}"
      }
    ]
  },
  "response": {
    "status": 200,
    "jsonBody": {
      "data": {
        "hero": {
          "name": "Luke Skywalker"
        }
      }
    }
  }
}
```

## Wiremock の Extension を作成した

Wiremock には[Extension 機能](https://wiremock.org/docs/extending-wiremock/)があり、Matching の機能を拡張することができる。その他にも、Response の Transform、Request の Filtering などさまざまな機能を拡張することができる。

今回は、GraphQL の Query をセマンティックに検証し、Matching する拡張機能を書いた。
Request Matching Extension は、`RequestMatcherExtension`を継承して以下のように実装する。([公式](https://wiremock.org/docs/extensibility/custom-matching/))

```kotlin
import com.github.tomakehurst.wiremock.extension.Parameters
import com.github.tomakehurst.wiremock.http.Request
import com.github.tomakehurst.wiremock.matching.MatchResult
import com.github.tomakehurst.wiremock.matching.RequestMatcherExtension

class GraphqlBodyMatcher() : RequestMatcherExtension() {
    override fun match(request: Request, parameters: Parameters): MatchResult {
        val isQueryMatch = // QueryのMatching
        val isVariablesMatch = // VariablesのMatching
        return when {
            isQueryMatch && isVariablesMatch -> MatchResult.exactMatch()
            else -> MatchResult.noMatch()
        }
    }

    override fun getName(): String {
        return "graphql-body-matcher"
    }
}
```

このあたりの詳細やクエリ正規化の実装に関して気になる方はソースを見てみてください。
薄い拡張なのですぐ読めます。

### Match する例

以下のクエリは同じクエリとして Matching する。
(空白などの正規化を行った後、セマンティックな検証を行う。)

```graphql
{
  hero {
    name
    friends {
      name
      age
    }
  }
}
```

```graphql
{
  hero {
    friends {
      age
      name
    }
    name
  }
}
```

### Match しない例

```graphql
{
  hero {
    name
    friends {
      name
      age
    }
  }
}
```

```graphql
{
  hero {
    name
    friends {
      name
    }
  }
}
```

## 使い方

[README](https://github.com/wiremock/wiremock-graphql-extension/blob/main/README.md)を見てもらうのが良いが、軽く紹介。

注意すべきなのは、Wiremock の Extension は Wiremock の起動時に読み込まれるので、Extension の jar を読み込む必要があること。そのため、Remote Wiremock を利用する場合は、Extension の jar を Wiremock の Docker イメージに追加し、Stub の設定なども少し変わる。

### ソースコード上で Wiremock を起動する場合

```kotlin
import io.github.nilwurtz.GraphqlBodyMatcher
// run server
WireMockServer(wireMockConfig().extensions(GraphqlBodyMatcher::class.java))
// stub
WireMock.stubFor(
    WireMock.post(WireMock.urlEqualTo("/graphql"))
        .andMatching(GraphqlBodyMatcher.withRequestJson("""{"query": "{ hero { name }}"}"""))
        .willReturn(WireMock.ok())
)
```

### Standalone Wiremock を利用する場合

docker などの場合、jar をマウントする OR COPY して Docker イメージを作成する必要がある。

```shell
docker run -it --rm \
      -p 8080:8080 \
      --name wiremock \
      -v /path/to/wiremock-graphql-extension-0.6.2-jar-with-dependencies.jar:/var/wiremock/extensions/wiremock-graphql-extension-0.6.2-jar-with-dependencies.jar \
      wiremock/wiremock \
      --extensions io.github.nilwurtz.GraphqlBodyMatcher
```

```dockerfile
FROM wiremock/wiremock:latest
COPY ./wiremock-graphql-extension-0.6.2-jar-with-dependencies.jar /var/wiremock/extensions/wiremock-graphql-extension-0.6.2-jar-with-dependencies.jar
CMD ["--extensions", "io.github.nilwurtz.GraphqlBodyMatcher"]
```

standalone の場合、すこしインターフェイスが変わる。
(ここについてはややこしいため統一するなど検討中です。)

```kotlin
import com.github.tomakehurst.wiremock.client.WireMock
import com.github.tomakehurst.wiremock.client.WireMock.*
import io.github.nilwurtz.GraphqlBodyMatcher

fun registerGraphQLWiremock(json: String) {
    WireMock(8080).register(
        post(urlPathEqualTo(endPoint))
            .andMatching(GraphqlBodyMatcher.extensionName, GraphqlBodyMatcher.withRequest(json))
            .willReturn(aResponse().withStatus(200))
    )
}
```

## Wiremock Organization に仲間入りした

この Extension を作成したのは業務で GraphQL の Mock がつらすぎるというところだったが、ちょうど Wiremock 側でも GraphQL など各プロトコルへの対応を強化しているところだったため、メールなどを頂いて Wiremock Organization にリポジトリを移動することになった。

**普段業務で使っているソフトウェアに技術者として関われるのはうれしい**
![I've joined wiremock organization](/images/wiremock-graphql-extension-01.png)

Official の GraphQL 拡張も検討中なようなので、GraphQL ユーザーはぜひチェックしていてほしい。
([issue](https://github.com/wiremock/ecosystem/issues/13))

FB、PR、Issue 等もぜひお願いします！
[https://github.com/wiremock/wiremock-graphql-extension](https://github.com/wiremock/wiremock-graphql-extension)
