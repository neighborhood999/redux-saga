# Recipes

## Throttling

You can throttle a sequence of dispatched actions by using a handy built-in `throttle` helper. For example, suppose the UI fires an `INPUT_CHANGED` action while the user is typing in a text field.

```javascript
import { throttle } from 'redux-saga/effects'

function* handleInput(input) {
  // ...
}

function* watchInput() {
  yield throttle(500, 'INPUT_CHANGED', handleInput)
}
```

By using this helper the `watchInput` won't start a new `handleInput` task for 500ms, but in the same time it will still be accepting the latest `INPUT_CHANGED` actions into its underlaying `buffer`, so it'll miss all `INPUT_CHANGED` actions happening in-between. This ensures that the Saga will take at most one `INPUT_CHANGED` action during each period of 500ms and still be able to process trailing action.

## Debouncing

To debounce a sequence, put the built-in `delay` helper in the forked task:

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

The `delay` function 使用 Promise 實作一個簡單的 debounce。
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

The ability to undo respects the user by allowing the action to happen smoothly
first and foremost before assuming they don't know what they are doing. [GoodUI](https://goodui.org/#8)
The [redux documentation](http://redux.js.org/docs/recipes/ImplementingUndoHistory.html) describes a
robust way to implement an undo based on modifying the reducer to contain `past`, `present`,
and `future` state.  There is even a library [redux-undo](https://github.com/omnidan/redux-undo) that
creates a higher order reducer to do most of the heavy lifting for the developer.

However, this method comes with overhead because it stores references to the previous state(s) of the application.

Using redux-saga's `delay` and `race` we can implement a simple, one-time undo without enhancing
our reducer or storing the previous state.

```javascript
import { take, put, call, spawn, race } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { updateThreadApi, actions } from 'somewhere'

function* onArchive(action) {

  const { threadId } = action
  const undoId = `UNDO_ARCHIVE_${threadId}`

  const thread = { id: threadId, archived: true }

  // show undo UI element, and provide a key to communicate
  yield put(actions.showUndo(undoId))

  // optimistically mark the thread as `archived`
  yield put(actions.updateThread(thread))

  // allow the user 5 seconds to perform undo.
  // after 5 seconds, 'archive' will be the winner of the race-condition
  const { undo, archive } = yield race({
    undo: take(action => action.type === 'UNDO' && action.undoId === undoId),
    archive: call(delay, 5000)
  })

  // hide undo UI element, the race condition has an answer
  yield put(actions.hideUndo(undoId))

  if (undo) {
    // revert thread to previous state
    yield put(actions.updateThread({ id: threadId, archived: false }))
  } else if (archive) {
    // make the API call to apply the changes remotely
    yield call(updateThreadApi, thread)
  }
}

function* main() {
  while (true) {
    // wait for an ARCHIVE_THREAD to happen
    const action = yield take('ARCHIVE_THREAD')
    // use spawn to execute onArchive in a non-blocking fashion, which also
    // prevents cancellation when main saga gets cancelled.
    // This helps us in keeping state in sync between server and client
    yield spawn(onArchive, action)
  }
}
```
