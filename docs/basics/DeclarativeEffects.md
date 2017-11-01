# 宣告 Effects

在 `redux-saga` 中，Saga 都是使用 Generator function 實作的。我們從 Generator yield 純 JavaScript 物件來表達 Saga 的邏輯。我們稱這些物件為 *Effects*。一個 Effect 是一個簡單的物件，它包含了一些由 middleware 解譯的資訊。你可以把 Effect 看作是給 middleware 執行一些操作的說明（也就是說，調用一些非同步的 function，dispatch 一個 action 到 store）。

你可以使用在 library 內提供的 `redux-saga/effects` package 的 function 來建立 Effect。

在這個部份和接下來的部份，我們將介紹一些基礎的 Effect，而且可以看到這些概念讓 Saga 變得更容易測試。

Saga 可以 yield 多種形式的 Effect。最簡單的方式就是 yield 一個 Promise。

例如，假設我們有一個 Saga 觀察一個 `PRODUCTS_REQUESTED` action。在每次發出的 action 符合 takeEvery 的 action 時，它啟動一個 task 來從伺服器取得一些產品。

```javascript
import { takeEvery } from 'redux-saga/effects'
import Api from './path/to/api'

function* watchFetchProducts() {
  yield takeEvery('PRODUCTS_REQUESTED', fetchProducts)
}

function* fetchProducts() {
  const products = yield Api.fetch('/products')
  console.log(products)
}
```

在上面的範例中，我們從 Generator 內直接調用了 `Api.fetch`（在 Generator function 中，任何在 yield 右邊的表達式都會被求值，然後結果被 yield 到 caller）。

`Api.fetch('/products')` 觸發一個 AJAX 請求並回傳一個 Promise，Promise 將被 resolve，被 resolve 的 Promise 會取得 response。簡單且直覺的，但是...

假設我們要測試上面的 generator：

```javascript
const iterator = fetchProducts()
assert.deepEqual(iterator.next().value, ??) // 我們期望都到的是？
```

我們想要檢查 generator yield 後第一個結果的值。在我們這個情況執行 `Api.fetch('/products')` 的結果是一個 Promise。在測試時，執行真正的服務不是一個實際可行的方法，所以我們需要 *mock* `Api.fetch` function，也就是說，我們將有一個替換真實 function 而不實際執行 AJAX 請求，確認我們呼叫 `Api.fetch` 與它的參數 （在這情況中，這裡的參數是 `'/products'`）。

Mock 讓測試更加困難而且不可靠。另一方面，那些只回傳數值的 function 更容易的測試，因此我們可以簡單的使用 `equal()` 來測試結果。這種方式可以撰寫更加可靠的測試。

不相信嗎？我建議你去閱讀 [Eric Elliott 的文章](https://medium.com/javascript-scene/what-every-unit-test-needs-f6cd34d9836d#.4ttnnzpgc)：

> (...)`equal()`，自然的回答這兩個重要的問題，是每個單元測試必須回答的，
但大多數不是這樣：
- 實際的輸出是什麼？
- 期望的輸出是什麼？
>
> 如果你完成一個測試沒有回答這兩個問題，就不是一個真實的單元測試。你的測試只是一個馬虎、不成熟的測試。

實際上我們需要確保 `fetchProducts` task yield 一個呼叫正確的 function 和正確的參數。

不是從 Generator 內直接調用非同步 function，**我們可以只 yield 一個 function 調用的描述**。也就是說我們簡單 yield 一個物件，看起來像是：

```javascript
// Effect -> 呼叫 function Api.fetch 與 `./products` 作為參數
{
  CALL: {
    fn: Api.fetch,
    args: ['./products']
  }
}
```

另一種方式，Generator 將 yield 包含純物件的*說明*，`redux-saga` middleware 將確保執行那些指令並將它們執行的結果給到 Generator。透過這種方式，在測試 Generator 的時候，我們只需要將 yield 後的物件透過簡單的 `deepEqual` 確認 yield 是否為期望的指令。

根據這樣的原因，library 提供了一個不同的執行非同步呼叫方式。

```javascript
import { call } from 'redux-saga/effects'

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // ...
}
```

我們現在使用 `call(fn, ...args)` function。**不同於前面的範例，我們現在不直接執行 fetch 呼叫，相反的，透過 `call` 建立一個 effect 的描述**。就像在 Redux 你可以使用 action creator 來建立一個純物件描述 action，透過 Store 接收後被執行，`call` 建立一個純物件來描述 function 的呼叫。redux-saga middleware 確認執行 function 的呼叫，並在 resolve response 時恢復 generator。

這讓我們在 Redux 外的環境更容易的測試 Generator。因為 `call` 只是一個 function，回傳一個純物件。

```javascript
import { call } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// 期望一個 call 的指令
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)
```

現在我們不需要 mock 任何東西，一個簡單的相等測試就足夠了。

這些*宣告的呼叫*的優點是，我們透過簡單的迭代 Generator，並在 yield 成功後得到的值做 `deepEqual` 測試，就可以測試所有在 Saga 的邏輯。這是真實的好處，你複雜的非同步操作不再是黑盒，不管它多麼複雜，你都可以測試每一個項目的操作邏輯。

`call` 也支援調用物件的方法，使用以下的形式，你可以提供一個 `this` context 到調用的 function：

```javascript
yield call([obj, obj.method], arg1, arg2, ...) // 如同我們使用 obj.method(arg1, arg2 ...)
```

`apply` 是這個調用方法形式的別名：

```javascript
yield apply(obj, obj.method, [arg1, arg2, ...])
```

`call` 和 `apply` 非常適合回傳 Promise 結果的 function。另一個 function `cps` 可以備用來處理 Node 風格的 function（例如：`fn(...args, callback)`，這個 `callback` 是 `(error, result) => ()` 的形式）。`cps` 是代表 Continuation Passing Style。

例如：

```javascript
import { cps } from 'redux-saga/effects'

const content = yield cps(readFile, '/path/to/file')
```

當然你可以測試它，就像測試 `call` 一樣：

```javascript
import { cps } from 'redux-saga/effects'

const iterator = fetchSaga()
assert.deepEqual(iterator.next().value, cps(readFile, '/path/to/file') )
```

`cps` 與 `call` 一樣，支援相同的調用方法形式。
