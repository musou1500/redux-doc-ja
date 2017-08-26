# ミドルウェアを理解する

ミドルウェアは，非同期のAPI呼び出しなど，様々なことに利用することができ，その起源を理解することはとても重要です．

ロギングとクラッシュレポーティングを例に挙げて，ミドルウェアに至るまでの過程についてのガイドを行います．

## 問題: ロギング

Reduxの一つの利点は，stateの変更を予測可能で，透過性にすることです．actionがdispatchされると，新しいstateが計算され，保存されます．

stateは自分自身を変更せず，特定のアクションの結果としてのみ変更されます．

アプリケーション内ですべてのアクションを記録することができれば，良いと思いませんか？なにか間違いが怒った時，ログを見返して，どのアクションがstateを破壊したのか見つけ出すことができます．

reduxでどのようにアプローチするのでしょうか．

## 試行 1: 手動でロギングする

もっとも純朴な解決方法は，`store.dispatch(action)`を呼び出す度に，単にアクションと次のstateを記録することです．これは本当の解決ではないですが，問題を理解するための最初のステップです．

> ### Note
>
>  もしもあなたがreact-reduxや，その他のバインディングを使用しているなら，コンポーネントからstoreに直接アクセスしないでしょう．
>
> 次の幾つかのパラグラフでは，単にストアを明示的に渡すと仮定してください

つまり，あなたがtodoを作成する時にこのように呼び出す：

```js
store.dispatch(addTodo('Use Redux'))
```

actionとstateを記録するため，このように変更することができます．

```js
let action = addTodo('Use Redux')

console.log('dispatching', action)
store.dispatch(action)
console.log('next state', store.getState())
```

これは希望した効果を作り出しますが，これを毎回やりたくはないでしょう．

## 試行2: Dispatchをラップする

あなたはロギングを関数に抽出することができます．

```js
function dispatchAndLog(store, action) {
  console.log('dispatching', action)
  store.dispatch(action)
  console.log('next state', store.getState())
}
```

これで`store.dispatch()`の代わりにどこでもこれを使うことができます．

```js
dispatchAndLog(store, addTodo('Use Redux'))
```

これで完了とすることもできますが，特殊な関数を毎回インポートするのは便利ではありません．

## 試行3: Dispatchをモンキーパッチする

storeインスタンス内にある`dispatch`関数を置き換えるのはどうでしょうか．

Reduxのstoreは幾つかのメソッドを持ったプレーンなオブジェクトで，Javascriptで書かれています．なので，`dispatch`の実装をモンキーパッチすることができます．

```js
let next = store.dispatch
store.dispatch = function dispatchAndLog(action) {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}
```

これは私達が欲していたものに近いです！どこでactionをdispatchしても，記録されることが保証されます．モンキーパッチングは正しくないように感じますが，今はこれで生きることができます．

## 問題: クラッシュレポーティング

このような`dispatch`に対する変形を複数回適用したい場合はどうでしょうか．

異なる便利な変形として，プロダクションでJavaScriptのエラーを報告することを思い浮かべています．

グローバルな`window.onerror`イベントは，幾つかの古いブラウザでスタック情報を提供しないため，信頼性のあるものではありません．

エラーが，どんな時でもactionをdispatchした結果として投げられ，これをSentryのようなクラッシュレポーティングサービスに，スタックトレース，エラーを発生させたアクションと，現在のstateと一緒に送信することができれば便利ですよね．この方法では開発環境で再現することも簡単です．

しかし，ロギングとクラッシュレポーティングを別に保つことが重要です．

理想的には，異なるモジュール，潜在的には異なったパッケージ含まれるようにしたいです．

そうでなければ，私たちはそのようなユーティリティのエコシステムを持つことができません．(ヒント: 私たちはゆっくりとミドルウェアが何であるかにたどり着いています)

ロギングとクラッシュレポーティングが別のユーティリティだとしたら，このようになるでしょう．

```js
function patchStoreToAddLogging(store) {
  let next = store.dispatch
  store.dispatch = function dispatchAndLog(action) {
    console.log('dispatching', action)
    let result = next(action)
    console.log('next state', store.getState())
    return result
  }
}

function patchStoreToAddCrashReporting(store) {
  let next = store.dispatch
  store.dispatch = function dispatchAndReportErrors(action) {
    try {
      return next(action)
    } catch (err) {
      console.error('Caught an exception', err)
      Raven.captureException(err, {
      	extra: {
          action,
          state: store.getState()
      	}
      })
      throw err;
    }   
  }
}
```

もしこれらの関数が異なるモジュールとして公開されているとすれば，私たちはstoreをパッチするために，これらをあとで使用することができます:

```js
patchStoreToAddLogging(store)
patchStoreToAddCrashReporting(store)
```

まだこれはよくありません．

## 試行4 モンキーパッチングを隠す

モンキーパッチングはハックです．"あなたの好きなように，何らかのメソッドを置き換える"，どのような種類のAPIがこれでしょうか．

代わりに，これの本質を見つけ出しましょう．以前，私達の関数は`store.dispatch`に置き換えられました．代わりに，新しい`dispatch`関数を返すのはどうでしょうか．

```js
function logger(store) {
  let next = store.dispatch
  
  // Previously:
  // store.dispatch = function dispatchAndLog(action) {
  return function dispatchAndLog(action) {
  	console.log('dispatching', action)
    let result = next(action)
    console.log('next state', store.getState())
    return result
  }
}
```

実際のモンキーパッチングを適用する，redux内部のヘルパーを，実装の詳細として提供することができます．

```js
function applyMiddlewareByMonkeypatching(store, middlewares) {
  middlewares = middlewares.slice()
  middlewares.reverse()
  
  // Transform dispatch function with each middleware.
  middlewares.forEach(middleware => {
    store.dispatch = middleware(store)
  })
}
```

複数のミドルウェアを適用するために，このように使用することができます．

```js
applyMiddlewareByMonkeypatching(store, [logger, crashReporter])
```

しかし，これはまだモンキーパッチングです．

ライブラリの内部にこれを隠しているという事実は変わりません．

## 試行5 モンキーパッチングを消す

なぜ`dispatch`を上書きするのでしょうか．もちろん，これをあとで呼び出すことができるからです．しかし，他の理由が存在します．

全てのミドルウェアがその前にラップした`store.dispatch`にアクセス(または呼び出し)できるからです．

```js
function logger(state) {
  let next = store.dispatch
  
  return function dispatchAndLog(action) {
    console.log('dispatching', action)
    let result = next(action)
    console.log('next state', store.getState())
    return result
  }
}
```

これはミドルウェア連鎖の本質です．



もし，`applyMiddlewareByMonkeypatching`がはじめのミドルウェアを処理した後すぐに`store.dispatch`をすぐに代入しないなら，`store.dispatch`は元の`dispatch`関数を指したままでしょう．そうすると，2番目のミドルウェアは元の`dispatch`関数と結び付けられるでしょう．

しかし，連鎖することができる別の方法があります．ミドルウェアは，`store`インスタンスからそれを読み取る代わりに，`next()`dispatch関数を引数として受け入れることができます．

```js
function logger(store) {
  return function wrapDispatchToAddLogging(next) {
  	return function dispatchAndLog(action) {
      console.log('dispatching', action)
      let result = next(action)
      console.log('next state', store.getState())
      return result
  	}
  }
}
```

これは [“we need to go deeper”](http://knowyourmeme.com/memes/we-need-to-go-deeper) の類のものなので，意味がわかるには時間が掛かるでしょう．この関数のカスケードには威圧感を感じます．ES6のアロー関数はこのカリー化を見やすくします．

```js
const logger = store => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}

const crashReporter = store => next => action => {
  try {
    return next(action)
  } catch (err) {
    console.error('Caught an exception!', err)
    Raven.captureException(err, {
      extra: {
        action,
        state: store.getState()
      }
    })
    throw err
  }
}

```

**これはまさにreduxミドルウェアの外観です．**

ミドルウェアは`next()`dispatch関数を受け取り，dispatch関数を返します．dispatch関数は，次にミドルウェアの`next()`として機能します．

`getState()`等の幾つかのstoreのメソッドにアクセスできることは便利であるため，`store`はトップレベルの引数として利用可能です．



## 試行 6 ミドルウェアを適用する

`applyMiddlewareByMonkeypatching()`の代わりに，最初に，最終的に，完全にラップされた`dispatch()`関数を取得し，それを使ってストアのコピーを返す，`applyMiddleware`を書くことができます．

```js
// Warning: Naïve implementation!
// That's *not* Redux API.
function applyMiddleware(store, middlewares) {
  middlewares = middlewares.slice()
  middlewares.reverse()
  let dispatch = store.dispatch
  middlewares.forEach(middleware => {
    dispatch = middleware(store)(dispatch)
  })
  
  return Object.assign({}, store, { dispatch })
}
```

Reduxで提供されている`applyMiddleware`の実装は似ていますが，3つの重要な側面で異なっています．

* ミドルウェアにstore APIのサブセットのみを公開している: `dispatch(action)`と`getState()`
* It does a bit of trickery to make sure that if you call `store.dispatch(action)` from your middleware instead of `next(action)`, the action will actually travel the whole middleware chain again, including the current middleware. This is useful for asynchronous middleware, as we have seen [previously](http://redux.js.org/docs/advanced/AsyncActions.html).
* To ensure that you may only apply middleware once, it operates on `createStore()` rather than on `store` itself. シグネーチャーは`(store, middlewares) => store`の代わりに，`(...middlewares) => (createStore) => createStore` となります．



Because it is cumbersome to apply functions to `createStore()` before using it, `createStore()`accepts an optional last argument to specify such functions.

 