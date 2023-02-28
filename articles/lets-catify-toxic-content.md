---
title: "毒気のある Web サービスをすべて猫化する"
emoji: "🐱"
type: "idea"
topics: ["userscript", "usercss"]
published: true
---

> 貴様も猫にしてやろうか……!

# 文脈

これは個人的な話にはなるのですが、最近（色々と知見を積んで）ユーザーコンテンツと向き合う必要のある Web サービスを業務で作っている方々への解像度が上がり、ユーザーコンテンツに対して業務として向き合っている人の中には、その人にとって直視したくないコンテンツと業務として向き合う必要が発生している、と言う事へ関心が向くようになりました。

それでこの見たくないコンテンツをなんとかして**技術的に** 解毒できないかと考えていたところ、最近始めた [misskey.io](https://misskey.io/) の謎機能である**猫化を応用することでなんとかならないか？** と思い立ち色々とやってみたところ、そこそこに良い感じの UserScript が出来上がったので、今日はそれを共有したいと思います。

※ なおこの UserScript を適用する範囲は皆様にお任せします。

# すべてのコンテンツを猫にする UserScript

```js
// ==UserScript==
// @name        Catify
// @namespace   https://the.kalaclista.com/
// @grant       none
// @version     0.0.1
// @author      OKAMURA Naoki aka nyarla <nyarla@kalaclista.com>
// @description 2023/2/28 13:58:09
// ==/UserScript==

const options = {
  childList: true,
  subtree: true,
};

const catify = (target) => {
  const catifyChild = (next, node) => {
    if (node === null || typeof node === "undefined") {
      return;
    }

    if (node.nodeType === Node.TEXT_NODE) {
      let catified = document.createTextNode(
        node.textContent.replace(/な/g, "にゃ")
      );
      node.parentNode.replaceChild(catified, node);
      return next;
    }

    if (node.nodeType === Node.ELEMENT_NODE) {
      let child = node.firstChild;
      while (child !== null && typeof child !== "undefined") {
        child = catifyChild(child.nextSibling, child);
      }

      return next;
    }

    return next;
  };

  let child = target.firstChild;
  while (child !== null && typeof child !== "undefined") {
    child = catifyChild(child.nextSibling, child);
  }
};

const target = document.body;
const callback = (mutationList, observer) => {
  for (const mutation of mutationList) {
    if (mutation.type !== "childList") {
      continue;
    }

    observer.disconnect();
    catify(mutation.target);
    observer.observe(target, options);
  }
};

catify(target);

const observer = new MutationObserver(callback);
observer.observe(target, options);
```

# この UserScript がやっている事

基本的にはこの UserScript が適用されるすべての Web サイト・Web サービスで『**な**』を『**にゃ**』に置換しているだけです。

またこの『**な**』を『**にゃ**』にすると言うアイディアは文脈にも書きましたが [misskey](https://github.com/misskey-dev/misskey) を多大に参考にしています。

# この UserScript の技術的な見所

## DOM API で文字列置換をやっている

まず最初に `catify` と言う関数では DOM API で `Node.TEXT_NODE` に対する Node だけに文字列の置換を掛けています。

これは何故かと言うと、例えば HTML 内のテキストの置換を手軽にしようと思うと、

```js
const el = document.body;
el.innerHTML = el.innterHTML.replace(/な/g, "にゃ");
```

に相当する手法が考えられますが、**この手法は DOM Node に対するイベント登録が全て無効化されてしまう** ため、**動的なインターフェースを含む Web アプリケーションで `innerHTML` を使うとその Web アプリが壊れます** 。そのため今回の UserScript では DOM API を使うと言う手法を取ることになりました。

## `MutationObserver` を使っている

次の見所としては DOM の監視をするために `addEventListener` を使うのではなく `MutationObserver` を使っていると言う点があります。

ここでなぜ `MutationObserver` を使っているのかと言えば、今回の UserScript では **動的に生成された物も含め、すべての DOM Node で**『**な**』を『**にゃ**』を書き換えたいと言う要件があったため、これを実現するためには _すべての DOM イベントを監視できる `MutationObserver` を利用するのがベストである_ 、との判断から `MutationObserver` による実装を採用しました。

また今回の実装では `MutationObserver` を使う際に、

```js
observer.disconnect();
catify(mutation.target);
observer.observe(target, options);
```

と言うコードで _文字列置換を行う前に一旦 MutationObserver による DOM 監視を止めている_ のですが、これは何故かと言うと、

1. `MutationObserver` で DOM 書き換えを検出する
2. `MutationObserver` の `callback` で `Node.TEXT_NODE` を書き換える
3. `MutationObserver` の `callback` による DOM 書き換えを `MutationObserver` が検出する
4. → 以下ループ

と言う事態を防ぐためです。

実際このスクリプトを作る際にはこの `MutationObserver` の無限ループに苦しめられていたのですが、結局ところ `disconnect()` してからの `observe()` でなんとかなったので、この手法は他の場面でも利用できるのではないか？と個人的に思っています。

## Twitter などでも問題なく動作する

今回の UserScript では、

1. DOM の書き換えを `MutationObserver` で検出する
2. 純粋な DOM API で `Node.TEXT_NODE` のみを書き換える

と言う流れで Web アプリ中のテキストを置換しています。

そのため React などと干渉しづらく、 Twitter と言った動的な Web アプリケーションでも動作する、と言う事が手元の環境では確認できています。

とは言え Web アプリケーションの中には動作がおかしくなる物もあると思いますし、何よりマシンへの負荷は多少上がると考えられるため、常に DOM が書き換えられている、と言ったタイプの Web アプリケーションには不向きではないか？と言う懸念はあります。

ただ今回の UserScript でそこまでの対応は難しいかなと考えているので、この UserScript を利用する方はその点を念頭に入れて頂けると助かります。

# 以上です

今回の UserScript では『**な**』を『**にゃ**』に書き換えることによって精神的負荷を減らす試みを行なっていますが、これに対し UserStyle で font を愛らしいものに替えるとより効果がアップします（個人の感想です）。

また私の場合だとフォントには [しねまきゃぷしょん](http://www.vector.co.jp/soft/data/writing/se314690.html) を利用（ローカルにインストール）し、下記の様な UserStyle を適用してみたところ、かなりの精神的負荷が下がるような印象でした。

なお今回の UserScript としねまきゃぷしょんを組み合わせた例は下記のスクリーンショットの様な感じです：

[![今回の UserScript と UserStyle を適用したスクリーンショット](https://zo.kalaclista.com/2023/02/28/19-15-40.webp)](https://zo.kalaclista.com/2023/02/28/19-15-40.webp)

また今回の UserScript とフォント変更の組み合わせはかなり殺伐としたサービスのマイルド化に役立つ印象でしたので、もし環境的に利用可能であれば今回の UserScript とフォントの変更は一度やってみる価値はあるかと思います。

---

と言うことでこちらからは以上となります。

今回の UserScript がユーザーコンテンツと向き合う必要のあるエンジニアの方々の助けになれば幸いです。
