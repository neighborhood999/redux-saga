# Recipes

## Throttling

你可以一個使用一個內建方便的 `throttle` helper 調整被調用的 action 序列。例如，假設當使用者在輸入框輸入文字時，UI 觸發一個 `INPUT_CHANGED` action。

```javascript
import { throttle } from 'redux-saga/effects'

function* handleInput(input) {
  // ...
}

function* watchInput() {
  yield throttle(500, 'INPUT_CHANGED', handleInput)
}
```

透過使用這個 helper，`watchInput` 不會在 500ms 後啟動一個新的 `handleInput` task，但在相同的時間將持續接受最新的 `INPUT_CHANGED` action 到它的底層 `buffer`，所以它將錯過之間所有發生的 `INPUT_CHANGED` action。這是確保 Saga 在每 500ms 時，最多得到一個 `INPUT_CHANGED` action 並可以繼續處理後續的 action。

## Debouncing

為了 debounce 一個 sequence，放置內建的 `delay` helper 在被 fork 的 task：

```javascript

import { delay } from 'redux-saga'

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

`delay` function 使用一個 Promise 實作一個簡單的 debounce。
```
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms))
```

在上面的範例，`handleInput` 在執行邏輯之前等待 500 毫秒。如果使用者在這個期間輸入了一些文字我們將得到更多 `INPUT_CHANGED` action。由於 `handleInput` 將被阻塞在 `delay`，透過 `watchInput` 在執行它的邏輯之前被取消。

上面的範例可以使用 redux-saga 的 `takeLatest` help 重新撰寫：

```javascript

import { delay } from 'redux-saga'

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
  // 在嘗試五次後失敗
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

undo 的行為是尊重使用者，在假設他們在不知道做什麼之前，允許讓 action 順利發生。[redux 文件](http://redux.js.org/docs/recipes/ImplementingUndoHistory.html) 的 [GoodUI](https://goodui.org/#8) 描述一個可靠的方法實作一個 undo，基於修改 reducer 包含 `past`、`present` 和 `future` state。甚至還有一個 library [redux-undo](https://github.com/omnidan/redux-undo) 建立一個 high order reducer 來為 developer 做更多繁重的工作。

然而，這個方法監聽 store 參考，來提供應用程式先前的 state(s)。

使用 redux-saga 的 `delay` 和 `cancel` 我們可以實作一個簡單、 one-time undo，不需要 enhance 我們的 reducer 和 store 先前的 state。

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

  // 樂觀地將 thread 標記為 `archived`
  yield put(actions.updateThread(thread))

  // 允許使用者在五秒執行 undo。
  // 在 5 秒後，'archive' 將會出現 race-condition 的 winner
  const { undo, archive } = yield race({
    undo: take(action => action.type === 'UNDO' && action.undoId === undoId),
    archive: call(delay, 5000)
  })

  // 隱藏 undo UI 元素，race condition 已經有了 winner
  yield put(actions.hideUndo(undoId))

  if (undo) {
    // 還原到先前 state 的 thread
    yield put(actions.updateThread({ id: threadId, archived: false }))
  } else if (archive) {
    // 讓 API 呼叫遠端應用更改
    yield call(updateThreadApi, thread)
  }
}

function* main() {
  while (true) {
    // 等待一個 ARCHIVE_THREAD 發生
    const action = yield take('ARCHIVE_THREAD')
    // 在非阻塞的方法下使用產生的結果來執行 onArchive，當 main saga 得到 cancell 也可以防止 cancelation。
    // 這可以幫助我們在 server 和 client 保持 state 的同步
    yield spawn(onArchive, action)
  }
}
```
