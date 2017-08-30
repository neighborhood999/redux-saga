# Recipes

## Throttling

你可以透過一個內建的 `throttle` helper 來 throttle 被 dispatch 的 action。例如，假設當使用者在 text 欄位打字時，在 UI 觸發一個 `INPUT_CHANGED` action。

```javascript
import { throttle } from 'redux-saga/effects'

function* handleInput(input) {
  // ...
}

function* watchInput() {
  yield throttle(500, 'INPUT_CHANGED', handleInput)
}
```

透過使用 `throttle` helper，`watchInput` 不會在 500ms 啟動一個新的 `handleInput` task，但在相同時間內，它仍然接受最新的 `INPUT_CHANGED` 到底層的 `buffer`，所以它會忽略所有 `INPUT_CHANGED` action。這確保 Saga 在 500ms 這段時間，最多接受一個 `INPUT_CHANGED` action，並且可以繼續處理 trailing action。

## Debouncing

為了 debounce 一個序列，把內建的 `delay` helper 放到被 fork 的 task：

```javascript

import { delay } from 'redux-saga'
import { call, cancel, fork, take } from 'redux-saga/effects'

function* handleInput(input) {
  // debounce by 500ms
  yield call(delay, 500)
  ...
}

function* watchInput() {
  let task
  while (true) {
    const { input } = yield take('INPUT_CHANGED')
    if (task) {
      yield cancel(task)
    }
    task = yield fork(handleInput, input)
  }
}
```

`delay` function 使用 Promise 實作一個簡單的 debounce。

```
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms))
```

在上面的範例，`handleInput` 在執行邏輯之前等待 500 毫秒。如果使用者在這個期間輸入了一些文字我們將得到更多 `INPUT_CHANGED` action。由於 `handleInput` 將被阻塞在 `delay`，透過 `watchInput` 在執行它的邏輯之前被取消。

上面的範例可以使用 redux-saga 的 `takeLatest` help 重新撰寫：

```javascript

import { delay } from 'redux-saga'
import { call, takeLatest } from 'redux-saga/effects'

function* handleInput({ input }) {
  // debounce by 500ms
  yield call(delay, 500)
  ...
}

function* watchInput() {
  // 將取消目前執行的 handleInput task
  yield takeLatest('INPUT_CHANGED', handleInput);
}
```

## 嘗試 XHR 呼叫

為了嘗試指定次數的 XHR 呼叫，使用一個 for 迴圈和 delay：

```javascript

import { delay } from 'redux-saga'
import { call, put, take } from 'redux-saga/effects'

function* updateApi(data) {
  for(let i = 0; i < 5; i++) {
    try {
      const apiResponse = yield call(apiRequest, { data });
      return apiResponse;
    } catch(err) {
      if(i < 4) {
        yield call(delay, 2000);
      }
    }
  }
  // 嘗試 5 秒後失敗
  throw new Error('API request failed');
}

export default function* updateResource() {
  while (true) {
    const { data } = yield take('UPDATE_START');
    try {
      const apiResponse = yield call(updateApi, data);
      yield put({
        type: 'UPDATE_SUCCESS',
        payload: apiResponse.body,
      });
    } catch (error) {
      yield put({
        type: 'UPDATE_ERROR',
        error
      });
    }
  }
}

```

在上面的範例，`apiRequest` 將重新嘗試五次，在這之間每次延遲兩秒。After the 5th failure, 在第五次失敗後，透過父 saga 將取得例外，我們將 dispatch `UPDATE_ERROR` action。

如果你不想要限制重新嘗試，你可以將 `for` 回圈替換成 `while (true)`。將 `take` 替換成 `takeLatest`，所以只嘗試最後一次的請求。在錯誤處理加入一個 `UPDATE_RETRY` action ，我們可以通知使用者更新沒有成功，但是它會重新嘗試。

```javascript
import { delay } from 'redux-saga'

function* updateApi(data) {
  while (true) {
    try {
      const apiResponse = yield call(apiRequest, { data });
      return apiResponse;
    } catch(error) {
      yield put({
        type: 'UPDATE_RETRY',
        error
      })
      yield call(delay, 2000);
    }
  }
}

function* updateResource({ data }) {
  const apiResponse = yield call(updateApi, data);
  yield put({
    type: 'UPDATE_SUCCESS',
    payload: apiResponse.body,
  });
}

export function* watchUpdateResource() {
  yield takeLatest('UPDATE_START', updateResource);
}

```

## Undo

Undo 的能力尊重使用者，假設使用者不知道他們在做什麼之前，允許 action 可以首先順利的發生。 [GoodUI](https://goodui.org/#8)
[redux 文件](http://redux.js.org/docs/recipes/ImplementingUndoHistory.html) 描述一個
穩定的方式來實作一個基於修改 reducer 包含 `past`、`present`、`future` state 的 undo。這裡甚至有一個 [redux-undo](https://github.com/omnidan/redux-undo) library，它建立一個 higher order reducer 來為 developer 做繁重的工作。

然而，這個方法附帶了一些開銷，因為它的 store 參考到先前應用程式的 state（s）。

使用 redux-saga 的 `delay` 和 `race` 我們可以實作一個簡單的、一次性的 undo，不需要 enhance 我們的 reducer 或 store 先前的 state。

```javascript
import { take, put, call, spawn, race } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { updateThreadApi, actions } from 'somewhere'

function* onArchive(action) {

  const { threadId } = action
  const undoId = `UNDO_ARCHIVE_${threadId}`

  const thread = { id: threadId, archived: true }

  // 顯示 undo UI 元素，並提供一個 key 來溝通
  yield put(actions.showUndo(undoId))

  // 樂觀地將 thread 標記作為 `archived`
  yield put(actions.updateThread(thread))

  // 允許使用者五秒後執行 undo。
  // 在五秒後，`archive` 會是 race-condition 的 winner
  const { undo, archive } = yield race({
    undo: take(action => action.type === 'UNDO' && action.undoId === undoId),
    archive: call(delay, 5000)
  })

  // race condition 有了結果，隱藏 undo UI 元素
  yield put(actions.hideUndo(undoId))

  if (undo) {
    // revert thread 到先前的 state
    yield put(actions.updateThread({ id: threadId, archived: false }))
  } else if (archive) {
    // 讓 API 呼叫遠端的修改
    yield call(updateThreadApi, thread)
  }
}

function* main() {
  while (true) {
    // 等待一個 ARCHIVE_THREAD 發生
    const action = yield take('ARCHIVE_THREAD')
    // 以非阻塞的方式使用 spawn 來執行 onArchive，
    // 當主要 saga 被取消時，阻止取消。
    // 這可以幫助我們保持 server 和 client 端 state 的同步
    yield spawn(onArchive, action)
  }
}
```
