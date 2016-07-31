# 錯誤處理

在這個部份我們將可以看到在先前的範例該如何處理失敗的情況。讓我假設我們的 API function fetch 因為某些原因失敗所以被 reject 並回傳一個 Promise。

我們想要在我們的 Saga 內處理一些錯誤，透過 dispatch 一個 `PRODUCTS_REQUEST_FAILED` action 到 Store。

我們在 Saga 內使用我們熟悉的 `try/catch` 捕捉錯誤語法。

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

// ...

function* fetchProducts() {
  try {
    const products = yield call(Api.fetch, '/products')
    yield put({ type: 'PRODUCTS_RECEIVED', products })
  }
  catch(error) {
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
  }
}
```

為了測試失敗的情況，我們將使用 Generator 的 `throw` 方法：

```javascript
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// 預期一個 call 的指令
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// 建立一個假的錯誤
const error = {}

// 預期 dispatch 一個指令
assert.deepEqual(
  iterator.throw(error).value,
  put({ type: 'PRODUCTS_REQUEST_FAILED', error }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_REQUEST_FAILED', error })"
)
```

在這個情況中，我們用 `throw` 方法傳送一個假的錯誤。因為這會將 Generator 目前的流程中斷，並執行 catch 區塊。

當然，你不一定要強迫在 `try`/`catch` 區塊中處理你的 API 錯誤。你也可以讓 API 服務回傳正常的值與一些錯誤的旗標。例如，你可以捕捉 Promise 的 reject，並將它們映射到一個物件的錯誤欄位。

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

function fetchProductsApi() {
  return Api.fetch('/products')
    .then(response => ({ response }))
    .catch(error => ({ error }))
}

function* fetchProducts() {
  const { response, error } = yield call(fetchProductsApi)
  if (response)
    yield put({ type: 'PRODUCTS_RECEIVED', products: response })
  else
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
}
```
