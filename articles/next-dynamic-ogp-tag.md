---
title: "Next.jsでCustomApp+動的なOGP設定をしてハマった点"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "ogp"]
published: false
---

# tl;dr

Next.jsのCustomAppでは、Componentを条件付きレンダーしてはいけない。

# OGPが出ない

以下のようなCustomAppとページがあるとする。

```tsx:pages/_app.tsx
import type { AppProps } from "next/app";

export default function MyApp({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}
```

[公式DocのExample](https://nextjs.org/docs/pages/building-your-application/data-fetching/get-server-side-props) のほぼまま。

```tsx:pages/index.tsx
import type { InferGetServerSidePropsType, GetServerSideProps } from "next";
import Head from "next/head";

type Repo = {
  name: string;
  stargazers_count: number;
};

export const getServerSideProps = (async (context) => {
  const res = await fetch("https://api.github.com/repos/vercel/next.js");
  const repo = (await res.json()) as Repo;
  return { props: { repo } };
}) satisfies GetServerSideProps<{
  repo: Repo;
}>;

export default function Page({
  repo,
}: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return (
    <>
      <Head>
        <meta property="og:title" content={repo.name} />
      </Head>
      <div>{repo.stargazers_count}</div>
    </>
  );
}
```

このとき、`<meta property="og:title" content={repo.name} />` はサーバーサイドでレンダリングされるので、OGPが正しく出る。

![image1](/images/next-dynamic-ogp-tag-01.png)

しかし、CustomAppに以下のような処理を入れていて、OGPが出ずにハマった。

```tsx:pages/_app.tsx
import type { AppProps } from "next/app";
import { useEffect, useState } from "react";

export default function MyApp({ Component, pageProps }: AppProps) {
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    // なにか処理
    const timer = setTimeout(() => {
      setLoaded(true);
    }, 100);

    return () => clearInterval(timer);
  }, []);

  return loaded ? <Component {...pageProps} /> : <div>Loading...</div>;
}
```

Curlしてみた結果、`<meta property="og:title" content={repo.name} />` が初回レンダリングされていないことがわかる。

![image2](/images/next-dynamic-ogp-tag-02.png)

# CustomAppでComponentの条件付きレンダーはダメ

ページ初回読み込み時にクライアント側で非同期処理することが必要で、
その間に表示崩れが起きるため、Loadingを表示するようにしていた。

今考えてみれば当たり前だが、`AppProps.Component`がレンダリングされることで、`getServerSideProps`で取得した`pageProps`が注入されサーバーサイドのレンダリングが行われMetaタグの埋め込みなどが行われるので、Componentのレンダリングを遅延させるとOGPタグの埋め込みが遅延されてしまうのは当然だった。


`AppProps.Component`はPageComponentそのものなので、`Next/Head`などの処理もそこに含まれることを意識すべきだった。

# 対応とまとめ

今回は処理中にLoadingを表示する必要があったので、対応として Loadingのオーバーレイ要素をページ全体の上に表示するようにしておき、そちらをStateによってアンマウントするようにした。
`AppProps.Component`をオーバーレイの裏で常にレンダリングしておく。

画面を見て確認していたが、ブラウザ上ではMetaタグは存在するし、この問題には気づくまでに時間がかかった。
またWeb上のOGP確認ツールなどで確認していたが、それぞれのサイトでクロールの仕組みが違うのか、確認ツールでは出るがSNSなどでは出ないということもあった。
OGPが出ないという問題はビジネス的にもインパクトが大きい場合があるので、死活監視やE2Eテストなども加えて担保しておきたいところだ。
