---
title: "Kotlin Maven 用 Gauge テンプレートを作った"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gauge", "kotlin", "maven"]
published: true
---

# つくった

```bash
// kotlin_maven の部分は任意
gauge template kotlin_maven https://github.com/nilwurtz/gauge-kotlin-maven/releases/latest/download/kotlin_maven.zip
gauge init kotlin_maven
```

---

Gauge には `gauge init <name>` でテンプレートを生成する機能があるが、Kotlinは対応していない。
業務で Gauge は利用しており Kotlin + maven のプロジェクトの運用実績はある上、Gauge プロジェクトを作成することは日常茶飯事なので、テンプレートがデフォルトであればよいな〜と思い [discussion](https://github.com/getgauge/gauge/discussions/2325) になげてみた。

すると 「現在は専任のコアチームはなく、カスタムテンプレート機能がおすすめ」 ということだったのでつくった。
https://github.com/nilwurtz/gauge-kotlin-maven

一度 `gauge template <name> <url>` で設定すれば繰り返し使えるので、ぜひ使ってください