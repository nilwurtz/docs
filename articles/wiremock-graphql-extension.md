---
title: "Wiremock の GraphQL Extension を作って公開して、Wiremock Organization に仲間入りした"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wiremock", "e2e", "graphql"]
published: true
---

# 概要

E2EなどでAPIをMockするツールとして、[Wiremock](https://wiremock.org/)がある。
WiremockはUrl、QueryParam、RequestBodyなどのMatchingが充実しており、普通のREST APIのMockには十分使える。

ただ、GraphQLのMockを作成する場合、WiremockのMatchingの機能だけではGraphQLのQueryのMatchingがつらいという問題がある。
今回はその問題を解決するために、WiremockのGraphQL向けExtensionを作成し、Wiremock Organizationに仲間入りしたのでその紹介。


つくった --> [Wiremock GraphQL Extension](https://github.com/wiremock/wiremock-graphql-extension)


## GraphQL の Query の Matching がつらい
業務の中で、GraphQLの外部APIを利用することになり、これもWiremockでMockしテストを記述していた。
GraphQLのQueryは、HTTPのPOSTで送信するので、WiremockのRequestBodyのMatchingを使ってMatchingすることができるので、最初はあまり問題ないように思えていたが、次第につらみが出てきた。

以下のようなQueryに対して、Mockを作成するとする。
```graphql
query { hero { name } }
```

これをWiremockのRequestBodyのMatchingで記述すると、以下のようになる。
(jsonPathのMatchingなども利用できる)
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

このように、GraphQLのQueryをMatchingするために、Queryの文字列をそのまま一致させる必要がある。
そのため、スペース一つ、改行一つでも違うと、Matchingに失敗してしまうことになり、これにずっと苦しんでいた。

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

## WiremockのExtensionを作成した

Wiremockには[Extension機能](https://wiremock.org/docs/extending-wiremock/)があり、Matchingの機能を拡張することができる。その他にも、ResponseのTransform、RequestのFilteringなどさまざまな機能を拡張することができる。

今回は、GraphQLのQueryをセマンティックに検証し、Matchingする拡張機能を書いた。
Request Matching Extensionは、`RequestMatcherExtension`を継承して以下のように実装する。([公式](https://wiremock.org/docs/extensibility/custom-matching/))
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

### Matchする例
以下のクエリは同じクエリとしてMatchingする。
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

### Matchしない例
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

注意すべきなのは、WiremockのExtensionはWiremockの起動時に読み込まれるので、Extensionのjarを読み込む必要があること。そのため、Remote Wiremockを利用する場合は、ExtensionのjarをWiremockのDockerイメージに追加し、Stubの設定なども少し変わる。

### ソースコード上でWiremockを起動する場合

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

### Standalone Wiremockを利用する場合
dockerなどの場合、jarをマウントする OR COPYしてDockerイメージを作成する必要がある。
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

standaloneの場合、すこしインターフェイスが変わる。
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

## Wiremock Organizationに仲間入りした

このExtensionを作成したのは業務でGraphQLのMockがつらすぎるというところだったが、ちょうどWiremock側でもGraphQLなど各プロトコルへの対応を強化しているところだったため、メールなどを頂いてWiremock Organizationにリポジトリを移動することになった。

**普段業務で使っているソフトウェアに技術者として関われるのはうれしい**
![I've joined wiremock organization](/images/wiremock-graphql-extension-01.png)


現在ではつぎのメジャーリリースであるWiremock 4でOfficialのGraphQL拡張をリリース予定とのことなので、GraphQLユーザーはぜひチェックしていてほしい。
([issue](https://github.com/wiremock/ecosystem/issues/13))


FB、PR、Issue等もぜひお願いします！
[https://github.com/wiremock/wiremock-graphql-extension](https://github.com/wiremock/wiremock-graphql-extension)