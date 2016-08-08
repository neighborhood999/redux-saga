# 術語表

這是一個在 Redux Saga 核心術語的詞彙表。

### Effect

Effect 是一個純 JavaScript 物件，包含一些透過 saga middleware 被執行的一些說明。

你使用 redux-saga library 提供的 factory function 建立 effect。例如，你使用 `call(myfunc, 'arg1', 'arg2')` 來說明 middleware 調用 `myfunc('arg1', 'arg2')` 並回傳結果到被 yield 的 effect 的 Generator。

### Task

Task 像是一個執行在背景的處理程序。基於 redux-saga 的應用程式，你可以同時執行多個 task，透過 `fork` function 建立 task。

```javascript
import {fork} from "redux-saga/effects"

function* saga() {
  ...
  const task = yield fork(otherSaga, ...args)
  ...
}
```

### 阻塞和非阻塞呼叫

一個阻塞呼叫意思是 Saga yield 一個 Effect，在恢復 yield Generator 下一個指令之前，等待它外部的執行。

一個非阻塞呼叫意思是 Saga 在 yield Effect 後，立即恢復執行。

例如：

```javascript
import {call, cancel, join, take, put} from "redux-saga/effects"

function* saga() {
  yield take(ACTION)              // 阻塞：等待 the action
  yield call(ApiFn, ...args)      // 阻塞：等待 ApiFn（如果 ApiFn 回傳一個 Promise）
  yield call(otherSaga, ...args)  // 阻塞：等待 otherSaga 終止

  yield put(...)                   // 非阻塞：將 dispatch 內部 scheduler

  const task = yield fork(otherSaga, ...args)  // 非阻塞:不等待 otherSaga
  yield cancel(task)                           // 非阻塞：立即恢復
  // 或
  yield join(task)                              // 阻塞：等待 task 終止
}
```

### Watcher 和 Worker

使用兩個獨立的 Saga 組織控制流程。

- watcher：觀察被 dispatch 的 action 並在每個 action fork 一個 worker

- worker：處理 action 並終止

example

```javascript
function* watcher() {
  while (true) {
    const action = yield take(ACTION)
    yield fork(worker, action.payload)
  }
}

function* worker(payload) {
  // ... 做其他事
}
```
