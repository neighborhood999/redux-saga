# 常見的併發模式

在基礎的部份，我們看到如何使用 `takeEvery` 和 `takeLatest` helper function 來管理 Effect 之間的併發。

在這個部份我將看到使用低階的 Effect 如何實現那些 helper。

## `takeEvery`

```javascript
import {fork, take} from "redux-saga/effects"

const takeEvery = (pattern, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(pattern)
    yield fork(saga, ...args.concat(action))
  }
})
```

允許多個 `saga` task 同時被 fork 。

## `takeLatest`

```javascript
import {cancel, fork, take} from "redux-saga/effects"

const takeLatest = (pattern, saga, ...args) => fork(function*() {
  let lastTask
  while (true) {
    const action = yield take(pattern)
    if (lastTask) {
      yield cancel(lastTask) // 如果 task 已經被終止的話，cancel 是一個空的操作
    }
    lastTask = yield fork(saga, ...args.concat(action))
  }
})
```

`takeLatest` 不允許多個 saga task 被同時觸發。一旦接收一個新的 dispatch action，它取消任何先前被 fork 的 task（如果 task 持續執行的話）。

在處理 AJAX 請求時，我希望只能在最後一個請求的時候得到 response，`takeLatest` 會非常的有用。
