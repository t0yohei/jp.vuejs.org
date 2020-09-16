# リアクティブの探求

さらに深く見ていきましょう！Vue の最大の特徴の 1 つは、控えめなリアクティブシステムです。モデルはプロキシされた JavaScript オブジェクトです。それらを変更するとビューが更新されます。これは状態管理を非常にシンプルかつ直感的にしますが、よくある問題を避けるためにその仕組みを理解することも重要です。このセクションでは、Vue のリアクティブシステムの低レベルの詳細の一部について掘り下げていきます。

# リアクティブとは何か？

この言葉はここ最近のプログラミングの中でしばしば目にしますが、それについて言及する時どういう意味で使っていますか？リアクティブは宣言的な方法で変更に対応できるようにするプログラミングのパラダイムです。優れているために、標準的な例としてしばしば例に上がるのが Excel スプレッドシートです。

<video width="550" height="400" controls>
  <source src="/images/reactivity-spreadsheet.mp4" type="video/mp4">
  お使いのブラウザはビデオタグに対応していません。
</video>

最初のセルに数字の 2 を入力し、2 番目のセルに数字の 3 を入力して SUM を要求すると、スプレッドシートは結果を提供してくれます。なんの驚きもありません。ただし、最初のセルの数字を更新すると、 SUM もなんと自動的に更新されます。

JavaScript は通常、このように機能しません。JavaScript で同等のものを作成しようとした場合は次のようになります：

```js
var val1 = 2;
var val2 = 3;
var sum = val1 + val2;

// sum
// 5

val1 = 3;

// sum
// 5
```

最初の値を更新しても、合計値は調整されません。

では、 JavaScript を使って以下の要素をどうやって実現するのでしょうか。

- いずれかの値に変化があった時に検出する
- それを変更する関数を追跡する
- 関数をトリガーにして、最終的な値を更新できるようにする

## Vue がこれらの変更を追跡する方法

プレーンな JavaScript オブジェクトを `data` オプションとしてアプリケーションまたはコンポーネントインスタンスに渡すと、Vue はそのすべてのプロパティを渡り歩いて、ゲッターとセッターのハンドラーを使用してそれらをプロキシに変換します。 これは ES6 のみの機能ですが、旧式の `Object.defineProperty` を使用して IE ブラウザをサポートするための Vue 3 のバージョンを提供しています。どちらも表面的には同じ API を提供しますが、プロキシバージョンの方がよりスリムで、より高いパフォーマンスを提供します。

<div class="reactivecontent">
  <iframe height="500" style="width: 100%;" scrolling="no" title="Proxies and Vue's Reactivity Explained Visually" src="https://codepen.io/sdras/embed/zYYzjBg?height=500&theme-id=light&default-tab=result" frameborder="no" allowtransparency="true" allowfullscreen="true">
    See the Pen <a href='https://codepen.io/sdras/pen/zYYzjBg'>Proxies and Vue's Reactivity Explained Visually</a> by Sarah Drasner
    (<a href='https://codepen.io/sdras'>@sdras</a>) on <a href='https://codepen.io'>CodePen</a>.
  </iframe>
</div>

これはかなり素早いので、理解するにはプロキシについての知識がある程度必要です。では、少し詳しく見ていきましょう。プロキシに関する文献はたくさんありますが、実際に知っておく必要があることは、 **プロキシは別のオブジェクトまたは関数を包み、操作を差し込むことができるオブジェクトだといいうことです。**

proxy は次のように使用します: `new Proxy(target, handler)`

```
const dinner = {
  meal: 'tacos'
}

const handler = {
  get(target, prop) {
    return target[prop]
  }
}

const proxy = new Proxy(dinner, handler)
console.log(proxy.meal)

// tacos
```

今のところは、オブジェクトをラップしてそのままそれを返すだけです。かっこいいですが、まだ役に立つ物ではありません。しかし、これを見てください。プロキシでラップしている中で、このオブジェクトに操作を差し込むこともできます。この操作の差し込みはトラップと呼ばれています。

```js
const dinner = {
  meal: 'tacos'
}

const handler = {
  get(target, prop) {
    console.log(‘intercepted!’)
    return target[prop]
  }
}

const proxy = new Proxy(dinner, handler)
console.log(proxy.meal)

// intercepted!
// tacos
```

コンソールログ以外にも、ここでは思い通りの操作がなんでもできます。必要な場合は、実際の値を返さないようにすることさえ可能です。これが、 API の作成の置いてプロキシが強力なものになっている理由です。

さらに、プロキシは別の機能も提供してくれます。`target[prop]` のような値をただ返すだけではなく、これをさらに一歩進めて、 `this` のバインディングを適切に行うことができる `Reflect` と呼ばれる機能を使用することができます。これは次のようになります。

```js{7}
const dinner = {
  meal: "tacos",
};

const handler = {
  get(target, prop, receiver) {
    return Reflect.get(...arguments);
  },
};

const proxy = new Proxy(dinner, handler);
console.log(proxy.meal);

// intercepted!
// tacos
```

前述の通り、何かが変更されたときに最終的な値を更新する API を実装するには、何かが変更されたときに新しい値を設定する必要があります。この処理をハンドラーの `track` という関数で `target` と `key` という引数を渡し行います。
