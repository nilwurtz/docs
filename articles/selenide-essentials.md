---
title: "Selenideを利用するときに心がけること"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Selenide", "E2E", "Test"]
published: True
---

# 概要

Testライブラリの[Selenide](https://selenide.org/)はUIテスト用の協力なライブラリで、業務でもよく利用している。

その機能を利用し、安定するテストを作るために 気をつけるべきこと がいくつかあるのでまとめておく。

※ Selenide:6.17.0 での記述です。

## 要素を検証するときは、必ずSelenideのメソッドを利用する

UIの検証する際、非同期通信の結果を画面に表示することがほとんどである。
そのため、要素が表示されるまで待つ必要がある。

Selenideのアサーションメソッドは要素が見つからない場合、タイムアウトに達するまで自動的にリトライする。
この機能はSelenideを利用する上でかなり協力な機能なので、ちゃんと使うようにする。

> Assertions also play role of explicit waits. They wait for condition (e.g. size(2), empty, texts("a", "b", "c")) to be satisfied until timeout reached (the value of Configuration.collectionsTimeout that is set to 6000 ms by default).

引用: https://selenide.org/documentation.html

### 実際のコード
```kotlin
fun assertElement() {
    // この場合、要素が見つからない場合、タイムアウトまでリトライする
    $(byText("Login")).shouldBe(visible);
    // この場合、要素が消えるまで待機する
    $(".loading").should(disappear);
}
```

別のアサートライブラリなどと組み合わせると、このWait機能が働かないため、不安定なテストとなる。
```kotlin
fun assertElement() {
    // この場合、要素が見つからない場合、タイムアウトまでリトライせずすぐに失敗する
    assertThat($("ul li")).hasSize(2)
}
```

**他にも・・・**

Selenideの機能として、アサーションに失敗した場合、スクリーンショットを自動的に撮影する機能がある。
これも、Selenideのアサーションメソッドを利用しないと働かないため、注意が必要である。

### アサーションを拡張する

一方で、Selenide標準のアサーションメソッドでは検証したいことが検証できないこともある。
例えば、HTML要素のAttributeを検証したい場合、Selenideのアサーションメソッドでは完全一致になってしまう。

部分一致で検証したい場合、Conditionを返す関数を作成しこれを利用することで実現できる。

```kotlin
import com.codeborne.selenide.Condition
import com.codeborne.selenide.Driver
import org.openqa.selenium.WebElement

fun attributeContains(attr: String, value: String): Condition {
    return object : Condition("attributeContains") {
        override fun apply(driver: Driver, element: WebElement): Boolean =
            element.getAttribute(attr).contains(value)
    }
}

fun assertElement() {
    // 待機しつつ、部分一致で検証する。検証に失敗した場合、スクリーンショットが撮影される
    $("input").shouldHave(attributeContains("value", "awesome"));
}
```

## WebdriverのSetup

Selenideには4.6.0以降、SeleniumManagerが組み込まれており、6.17.0からはデフォルトで利用される。
（それまでは[bonigarcia/webdrivermanager](https://github.com/bonigarcia/webdrivermanager)が利用されていた。）
これは、ブラウザ種類やバージョンを検知、WebDriverを自動的にダウンロード、キャッシュしてくれるツールである。

参考: https://selenide.org/2023/08/02/selenide-6.17.0/

このおかげで、基本的にはWebDriverのセットアップを意識する必要はないが、Headlessや各種Optionの設定、RemoteDriverの利用などカスタムしたい場合がある。その場合、`ChromeDriverFactory`を継承したクラスを作成し、`selenide.browser`に設定することでカスタムできる。


```kotlin
package com.example

import com.codeborne.selenide.Browser
import com.codeborne.selenide.Config
import com.codeborne.selenide.webdriver.ChromeDriverFactory
import org.openqa.selenium.Proxy
import org.openqa.selenium.chrome.ChromeOptions
import java.io.File

class CustomChromeDriverFactory : ChromeDriverFactory() {
    override fun createCapabilities(
        config: Config,
        browser: Browser,
        proxy: Proxy?,
        browserDownloadsFolder: File?
    ): ChromeOptions {
        return super.createCapabilities(config, browser, proxy, browserDownloadsFolder).apply {
            addArguments("--headless")
        }
    }
}
```

```
mvn test -Dselenide.browser=com.example.CustomChromeDriverFactory
```
(system property、CLI引数などですべて設定することもできるが、Classを書いておくと管理しやすい)