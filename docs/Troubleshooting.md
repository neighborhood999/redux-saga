# 疑難排解

### 加入 Saga 後應用程式停住了

確認你 `yield` 的 effect 來自 generator function。

考慮以下的範例：

```js
import { take } from 'redux-saga/effects'

function* logActions() {
  while (true) {
    const action = take() // 錯誤
    console.log(action)
  }
}
```

這會讓應用程式進入無窮迴圈，因為 `take` 只建立一個 effect 的描述。除非你 `yield` 到 middleware 去執行，否則上面的 `while` 就像正常的 `while` 迴圈，並凍結你的應用程式。

加入 `yield` 將暫停 generator，並將控制回傳給 Redux Saga middleware 執行 effect。在 `take()` 的情況中，Redux Saga 等待下一個符合的 action，並恢復 generator。

為了修正上面的範例，只要 `yield take()` 回傳的 effect：

```js
import { take } from 'redux-saga/effects'

function* logActions() {
  while (true) {
    const action = yield take() // 正確
    console.log(action)
  }
}
```

### Saga miss 被 dispatch 的 action

確認 Saga 不會被阻塞在某些 effect，當 Saga 等待一個 Effect resolve，它不會 take 被 dispatch 的 action，直到 Effect 被 resolve。

例如，考慮以下的範例：

```javascript
function watchRequestActions() {
  while (true) {
    const {url, params} = yield take('REQUEST')
    yield call(handleRequestAction, url, params) // Saga 將在這阻塞
  }
}

function handleRequestAction(url, params) {
  const response = yield call(someRemoteApi, url, params)
  yield put(someAction(response))
}
```

當 `watchRequestActions` 執行 `yield call(handleRequestAction, url, params)` 時，它會在下一個 `yield take` 之前，等待 `handleRequestAction` 直到它終止回傳。例如，假設我們有一個事件的順序是這樣：

```
UI                     watchRequestActions             handleRequestAction
-----------------------------------------------------------------------------
.......................take('REQUEST').......................................
dispatch(REQUEST)......call(handleRequestAction).......call(someRemoteApi)... Wait server resp.
.............................................................................
.............................................................................
dispatch(REQUEST)............................................................ Action missed!!
.............................................................................
.............................................................................
.......................................................put(someAction).......
.......................take('REQUEST')....................................... saga is resumed
```

根據上述，當一個 Saga 被阻塞在 **blocking call**，它將忽略所有被 dispatch 的 action。

為了避免阻塞的 Saga，你可以使用 **non-blocking call** `fork` 來替代 `call`。

```javascript
function watchRequestActions() {
  while (true) {
    const {url, params} = yield take('REQUEST')
    yield fork(handleRequestAction, url, params) // Saga 將立刻恢復
  }
}
```
