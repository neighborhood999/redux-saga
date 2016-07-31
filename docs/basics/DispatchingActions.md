# Dispatch action 到 store

在先前的範例我們更進一步的，在每次儲存後，我們想要 dispatch 一些 action 來通知 Store fetch 已經成功了（目前我們先忽略失敗的情況）。

我們可以傳送 Store 的 `dispatch` function 到 Generator。然後在取得回應後，Generator 可以調用它：

```javascript
// ...

function* fetchProducts(dispatch) {
  const products = yield call(Api.fetch, '/products')
  dispatch({ type: 'PRODUCTS_RECEIVED', products })
}
```

然而，這個解決方式和在 Generator 直接調用 function 一樣，存在一些缺點（我們先前有討論到）。如果我們想要測試 `fetchProducts` 在接收 AJAX 回應後執行 dispatch，我們將需要再次 mock `dispatch` function。

相反的，我們需要相同的宣告解決方式。只要建立一個物件來告訴 middleware 我們需要 dispatch 一些 action，並讓 middleware 執行真實的 dispatch。我們可以用同樣的測試方式來測試 Generator 的 dispatch：透過檢查 yield 後的 Effect 並確認它是不是包含正確的指令。

為了這個目的，library 提供了另一個 `put` function，可以建立 dispatch 的 Effect。

```javascript
import { call, put } from 'redux-saga/effects'
// ...

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // 建立並 yield 一個 dispatch
  yield put({ type: 'PRODUCTS_RECEIVED', products })
}
```

現在我們可以像前面一樣簡單的測試 Generator：

```javascript
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// 執行一個 call 的指令
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// 建立一個假的 response
const products = {}

// 預期一個 dispatch 的指令
assert.deepEqual(
  iterator.next(products).value,
  put({ type: 'PRODUCTS_RECEIVED', products }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_RECEIVED', products })"
)
```

注意現在我們經由 `next` 方法傳送假的 response 到 Generator。在 middleware 環境之外，我們可以完全控制所有 Generator，我們透過簡單的 mock 結果並恢復 Generator 來假設一個真實環境。Mock 資料比 mock function 和 spy call 來的簡單。
