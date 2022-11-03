---
title: "[Rust] 複数のEnumを扱う条件分岐"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---

Rust には複数の条件分岐がありどれも便利だが、最適な書き方を探すのが難しい。
今回は、複数の Enum を扱う条件分岐において、どんな書き方ができるかを考えてみた

## 題材

今回の題材として、2つの `anyhow::Result` の両方が OK なら A, それ以外(どちらか Err)なら B という処理を行う。
両方の処理で、変数の中身を取得する前提。

```rust
fn return_ok() -> anyhow::Result<String> {
    Ok("awesome string".into())
}

fn return_err() -> anyhow::Result<String> {
    Err(anyhow::anyhow!("awesome error"))
}
```

## if

素直 だが 取り出すのは若干面倒。
インデントは深くならないのが良い。`is_ok()` の条件は後述のデストラクトするパターンよりは読みにくい。

```rust
    let o1 = return_ok();
    let e1 = return_err();
    if o1.is_ok() && e1.is_ok() {
        println!("Both result is ok :) {} {}", o1.unwrap(), e1.unwrap())
    } else {
        println!(
            "Either one is an error :/ {} {}",
            o1.unwrap_or_else(|e| e.to_string()),
            e1.unwrap_or_else(|e| e.to_string())
        )
    }
```

## match

若干 unwrap がめんどくさいが、両方 `Ok` であるという条件がデストラクトにより明示的になりよさそう
インデントは深くなる。

```rust
    match (return_ok(), return_err()) {
        (Ok(s1), Ok(s2)) => {
            println!("Both result is ok :) {} {}", s1, s2)
        }
        (x, y) => {
            println!(
                "Either one is an error :/ {} {}",
                x.unwrap_or_else(|e| e.to_string()),
                y.unwrap_or_else(|e| e.to_string())
            )
        }
    };
```

Match arm は補完ができるし、よりわかりやすくなるのでこういうのもそんなに悪くない
ただ変数が3つになったりすると厳しそう。

```rust
    match (return_ok(), return_ok()) {
        (Ok(s1), Ok(s2)) => {
            println!("Both result is ok :) {} {}", s1, s2)
        }
        (Ok(_), Err(_)) => todo!(),
        (Err(_), Ok(_)) => todo!(),
        (Err(_), Err(_)) => todo!(),
    };
```

### match with guard

Matchにはガードがあり、これでもいい感じにかけるかと思っていたが、Enumなら普通に Match 書いたほうがわかりやすい。
unwrapも面倒。
これから条件が増えていくとかなら良さそう。

```rust
    match (return_ok(), return_err()) {
        (x, y) if x.is_ok() && y.is_ok() => {
            println!("Both result is ok :) {} {}", x.unwrap(), y.unwrap())
        }
        (x, y) => {
            println!(
                "Either one is an error :/ {} {}",
                x.unwrap_or_else(|e| e.to_string()),
                y.unwrap_or_else(|e| e.to_string())
            )
        }
    };
```

## if let

if let だと、インデントが少なく、かつ明示的。
ただ else で変数を扱うなら `as_ref()` する必要がある。

else で変数の中身がいらないなら一番スッキリ。

```rust
    let r1 = return_ok();
    let r2 = return_err();
    // as_ref しないと r1 r2 の move が起き、else で見れなくなる
    if let (Ok(o1), Ok(o2)) = (r1.as_ref(), r2.as_ref()) {
        // o1 o2 は &String
        println!("Both result is ok :) {} {}", o1, o2)
    } else {
        println!(
            "Either one is an error :/ {}, {}",
            r1.unwrap_or_else(|e| e.to_string()),
            r2.unwrap_or_else(|e| e.to_string())
        )
    }
```