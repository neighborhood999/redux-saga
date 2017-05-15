# 連結 Saga 到外部的輸入和輸出

我們已經看過 `take` Effect 透過等待的 action 被 dispatch 到 Store 然後 resolve。透過 dispatch action 作為參數提供給 `put` Effect 被 resolve。

當一個 Saga 被啟動時（不管是初始啟動或延遲動態啟動），middleware 自動連結 `take/put` 到 store。這兩個 Effect 可以看作 Saga 的輸入和輸出（Input/Output）排序。

提供一種方式在 Redux middleware 外部環境下執行一個 Saga，並連結到自訂的輸入和輸出。

```javascript
import { runSaga } from 'redux-saga'

function* saga() { ... }

const myIO = {
  subscribe: ..., // 這個將被用來 resolve take Effects
  dispatch: ...,  // 這個將被用來 resolve put Effects
  getState: ...,  // 這個將被用來 resolve select Effects
}

runSaga(
  saga(),
  myIO
)
```

如果要了解更多資訊，請參考 [API 文件](https://redux-saga.github.io/redux-saga/docs/api/index.html#runsagaiterator-options)。
