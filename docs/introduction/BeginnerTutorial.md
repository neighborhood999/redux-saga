# 新手教學

## 這個教學的目標

這個教學嘗試（希望）透過平易近人的方式來介紹 redux-saga。

我們使用 Redux repo 中簡單的 Counter 範例作為我們的入門教學。這個應用程式相對是簡單的，但是很適合來說明 redux-saga 的基本概念，才不會讓你迷失在過多的細節裡。

### 初始步驟

在開始之前，請先 clone [教學的 repository](https://github.com/yelouafi/redux-saga-beginner-tutorial)。

> 這個教學最終完成的程式碼位在 `sagas` 的 branch。

然後在 command line 執行：

```sh
$ cd redux-saga-beginner-tutorial
$ npm install
```

要啟動這個應用程式，請執行：

```sh
$ npm start
```

我們先從最簡單的範例：兩個按鈕 `Increment` 和 `Decrement` 的計數器。在這之後，我們將會介紹非同步的呼叫。

如果一切順利的話，你應該會看到兩個按鈕 `Increment` 和 `Decrement`，還有一個在下方顯示的訊息 `Counter: 0`。

> 你在執行應用程式如果遇到一個問題，請在這個[教學 repo](https://github.com/yelouafi/redux-saga-beginner-tutorial/issues) 建立一個 issue。

## Hello Sagas!

我們已經建立了我們的第一個 Saga。根據傳統，我們將撰寫我們 Saga 版本的 'Hello, world'。

建立一個 `saga.js` 的檔案，然後加入以下的程式碼片段：

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!')
}
```

所以這沒什麼好怕的，只是一個正常的 function（除了有一個 `*`）。它所做的就是列印一個歡迎的訊息到 console。

為了執行我們的 Saga，我們需要：

- 建立一個 Saga middleware 與我們要執行的 Saga（目前我們只有一個 `helloSaga`）
- 連結 Saga middleware 到 Redux store

我們將改變我們的 `main.js`：

```javascript
// ...
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

// ...
import { helloSaga } from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(helloSaga)

// 休息不改變（rest unchange）
```

首先我們 import 我們的 Saga 從 `./sagas` module。然後使用 `redux-saga` library 匯出的 `createSagaMiddleware` factory function 建立一個 middleware。

在執行我們的 `helloSaga` 之前，我們必須使用 `applyMiddleware` 連結我們的 middleware 到 Store。然後我們可以使用 `sagaMiddleware.run(helloSaga)` 來啟動我們的 Saga。

到目前為止，我們的 Saga 沒有什麼特別的。它只是 log 一個訊息然後離開。

## 進行非同步的呼叫

現在讓我們加入一些東西來更接近原始的 Counter 範例。為了要說明非同步的呼叫，我們將加入另一個按鈕，在按下一秒後來增加 counter。

首先，我們將提供一個額外的 callback `onIncrementAsync` 到 UI component。

```javascript
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
  <div>
    {' '}
    <button onClick={onIncrementAsync}>Increment after 1 second</button>
    <hr />
    <div>Clicked: {value} times</div>
  </div>
```

接下來我們應該連結 Component 的 `onIncrementAsync` 到 Store action。

我們將根據以下修改 `main.js` module：

```javascript
function render() {
  ReactDOM.render(
    <Counter
      onIncrementAsync={() => action('INCREMENT_ASYNC')}
    />,
    document.getElementById('root')
  )
}
```

注意，這不像 redux-thunk，我們 component dispatch 一個純物件的 action。

現在我們將介紹另一個 Saga 來執行非同步呼叫。我們使用的方式如下：

> 在每個 `INCREMENT_ASYNC` action，我們需要啟動一個 task 來做以下的事：

> - 等待一秒然後 counter 加一

加入以下的程式碼到 `sagas.js` module：

```javascript
import { takeEvery, delay } from 'redux-saga'
import { put } from 'redux-saga/effects'

// 我們工作的 Saga：將執行非同步的 increment task
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// 我們觀察的 Saga：在每個 INCREMENT_ASYNC 產生一個新的 incrementAsync task
export function* watchIncrementAsync() {
  yield* takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

該是花一些時間解釋的時候了。

我們 import 一個 utility funciton `delay`，它回傳一個 [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)，在指定內的毫秒數後 resolve。我們將使用這個 function 來 *block* Generator。

Saga 被實作為 [Generator function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)，*yield* 物件到 redux-saga middleware。被 yield 的物件是一種指令，透過 middleware 被解讀。當一個 Promise 被 yield 到 middleware，middleware 將暫停 Saga，直到 Promise 完成。在上面的範例中，`incrementAsync` Saga 被暫停，直到 Promise 透過 `delay` 回傳 resolve；它將發生在一秒後。

一旦 Promise 被 resolve，middleware 將恢復 Saga，執行程式碼直到下一次的 yield。在這個範例中，下一個 statement 是另一個被 yield 的物件：呼叫 `put({type: 'INCREMENT'})` 的結果，它說明了 middleware dispatch 一個 `INCREMENT` 的 action。

`put` 是我們呼叫一個 *Effect* 的範例。Effect 是一些簡單的 JavaScrpt 物件，包含由 middleware 實現的說明。當一個 middleware 透過 Saga 取得一個被 yield 的 Effect，Saga 會被暫停，直到 Effect 被完成。

所以讓我們總結一下，`incrementAsync` Saga 藉由呼叫 `delay(1000)` sleep 一秒後，然後 dispatch 一個 `INCREMENT` action。

接下來，我們建立另一個 `watchIncrementAsync` Saga。我們使用由 `redux-saga` 提供的 `takeEvery` helper function，來監聽被 dispatch 的 `INCREMENT_ASYNC` action 並每次執行 `incrementAsync`。

現在我們有兩個 Saga，而且我們需要一次啟動他們兩個。如果要這麼做，我們將加入一個 `rootSaga` 負責啟動我們其他的 Saga。在相同的 `sagas.js` 檔案裡，加入以下程式碼：

```javascript
// 單一進入點，一次啟動所有 Saga
export default function* rootSaga() {
  yield [
    helloSaga(),
    watchIncrementAsync()
  ]
}
```

這個 Saga yield 一個陣列，呼叫我們的兩個 saga：`helloSaga` 和 `watchIncrementAsync` 的結果。意思說這兩個 Generators 將會被平行啟動。現在我們只有在 `main.js` 的 root Saga 調用 `sagaMiddleware.run`。

```javascript
// ...
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = ...
sagaMiddleware.run(rootSaga)

// ...
```

## 讓我們的程式碼可以測試

我們需要測試我們的 `incrementAsync` Saga 以確保它所需要執行的 task。

建立另一個 `saga.spec.js` 檔案：

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // 怎麼辦呢？
});
```

由於 `incrementAsync` 是一個 Generator function，當我們在 middleware 外執行它，每次調用 generator 的 `next`，你可以取得以下形狀的物件：

```javascript
gen.next() // => { done: boolean, value: any }
```

`value` 欄位包含被 yield 後的表達式，也就是說在 `yield` 後的表達式結果。`done` 欄位說明如果 generator 是否結束或是還有 `yield` 的表達式。

在 `incrementAsync` 的情況下，generator 連續 yield 兩個值：

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`

所以，如果我們連續調用三次 generator 的 next 方法，我們會得到以下的結果：

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

前面兩次調用回傳 yield 表達式的結果。在第三次的調用由於沒有其他的 yield 了，所以 `done` 欄位被設為 true。而且 `incrementAsync` Generator 不會回傳任何東西（沒有 `return` 語句），`value` 欄位被設定為 `undefined`。

所以現在呢，為了測試 `incrementAsync` 內的邏輯，我們只需要對每個回傳 Generator 做簡單的迭代並檢查 yield 後的值。

```javascript
import test from 'tape';

import { incrementAsync } from '../src/sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next(),
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

這裡有個問題是我們要怎麼測試 `delay` 的回傳值？我們不能在 Promise 上只做簡單的相等測試。如果 `delay` 回傳一個*標準*值，這樣會更容易的測試。

好吧，`redux-saga` 提供了一種方式讓上面的語句變得可能。不是在 `incrementAsync` 內直接呼叫 `delay(1000)`，我們將*間接*的呼叫它：

```javascript
// ...
import { put, call } from 'redux-saga/effects'
import { delay } from 'redux-saga'

export function* incrementAsync() {
  // 使用 call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

我們現在做的是 `yield call(delay, 1000)`，而不是直接 `yield delay(1000)`。這有什麼不同呢？

在第一個情況下，yield 表達式 `delay(1000)` 被傳送到 `next` 的 caller 已經被執行了（當執行我們的程式碼時，caller 可能是 middleware。它可能是我們執行測試程式碼的 Generator function 和迭代後回傳的 Generator）。所以當 caller 得到一個 Promise，像是上面的測試程式碼一樣。

在第二個情況下, yield 表達式 `call(delay, 1000)` 被傳送到 `next` 的 caller。`call` 就像 `put`，回傳指示給 middleware 去呼叫給定的 function 與給定的參數的 Effect。事實上，`put` 和 `call` 透過他們本身執行任何 dispatch 或非同步的呼叫，它們只是簡單回傳純 JavaScript 的物件。

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

這裡的情況是：middleware 檢查每個 yield Effect 的類型，然後決定實現哪個 Effect。如果 Effect 的類型是一個 `PUT`，然後它將 dispatch 一個 action 到 Store。如果 Effect 是一個 `CALL`，它將呼叫給定的 function。

這種將建立 Effect 和執行 Effect 分離的作法，使得我們用令人驚訝的簡單方式來測試我們的 Generator：

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    call(delay, 1000),
    'incrementAsync Saga must call delay(1000)'
  )

  assert.deepEqual(
    gen.next().value,
    put({type: 'INCREMENT'}),
    'incrementAsync Saga must dispatch an INCREMENT action'
  )

  assert.deepEqual(
    gen.next(),
    { done: true, value: undefined },
    'incrementAsync Saga must be done'
  )

  assert.end()
});
```

由於 `put` 和 `call` 回傳純物件，我們可以在我們的測試程式重複使用相同的 function。在 `incrementAsync` 測試的邏輯，我們只是迭代 generator 並對他們的值做 `deepEqual` 測試。

為了執行上面的測試，輸入：

```sh
$ npm test
```

測試結果會在 console 上。
