---
title: "python一筋3年がA Tour of Goで詰まったこと"
qiita_id: "e6b19ea26662482e0003"
url: "https://qiita.com/ikuoikuo/items/e6b19ea26662482e0003"
created_at: "2024-12-09T09:53:35+09:00"
updated_at: "2024-12-09T09:53:35+09:00"
tags: ["Python","Go"]
---

<!-- Archived from Qiita: https://qiita.com/ikuoikuo/items/e6b19ea26662482e0003 -->
## この記事 is 何
いつもはPythonを使用して機械学習やデータ分析を行っています。
最近後輩がGoを使いだしたらやめられないと言っていたのでGoのチュートリアルを受けてみることにしました。


https://go-tour-jp.appspot.com/list

というわけで、Pythonとの違いや詰まったところをまとめていこうと思います。

## TL;DR
以下の部分で詰まりました。
- ポインタ
- 型宣言
- 使用しない変数を許さない

## ポインタ
C言語などでポインタは出てくると噂でしたが、Goでも出てくるとは。。。
避けては通れないですね。

ポインタは値のメモリアドレスを示すもので、以下のようにして使用します。

```go
i := 42
p := &i 
fmt.Println(*p) //42
```

pythonでは基本的に変数などは参照渡しですが、Goではポインタを用いることで参照渡しをすることができるらしいです。

こちらの記事がわかりやすかったので、メモしておきます。

https://qiita.com/Sekky0905/items/447efa04a95e3fec217f


## 型宣言
でた。。型宣言。。
Pythonは自動で推論してくれるから楽ですよね。

なんでいちいち書かなきゃいけないんだって思うけど、多くのエンジニアは型がないと気持ち悪いって言いますよね。。

見慣れない表記だったのは、型だけ宣言して初期化しなかったときにゼロ値が与えられることです。

```go
func main() {
	var i int
	var f float64
	var b bool
	var s string
	fmt.Printf("%v %v %v %q\n", i, f, b, s)
    // 0 0 false ""
}
```

## 使用しない変数を許さない
この機能はpythonにも実装してほしいなと思いました。
（すぐコードを散らかしてしまうタイプなので、、、）

Goは使用しない変数があるとビルドできません。
例えば先ほどのコードに変数tを含めておくと`declared and not used`というエラーがでます。

```go
func main() {
	var i int
	var f float64
	var b bool
	var s string
    var t string
	fmt.Printf("%v %v %v %q\n", i, f, b, s)
    // declared and not used: t
}
```

使用していないインポート等も同様にエラーがでるので、コードの可読性が強制的に上がりますね！

## まとめ
A Tour of Goを通して、Pythonでいかに楽してきたかがわかりました。

型宣言などは最近Pythonの関数等で使用している人も増加していたり、参照渡しを行っていると意識できたりと、Pythonの理解度も深まった旅でした。

また、使わないインポートなどはPythonでも整理しようと思います（笑）

