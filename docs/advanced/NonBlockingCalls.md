# 非阻塞式的呼叫

在先前的部份我們看到 `take` Effect 讓我們可以在一個集中的地方更好的描述一個非同步的流程。

重新登入的流程範例：

```javascript
function* loginFlow() {
  while (true) {
    yield take('LOGIN')
    // ... 執行登入邏輯
    yield take('LOGOUT')
    // ... 執行登出邏輯
  }
}
```

讓我們來實作並完成這個實際的登入/登出的邏輯。假設我們在遠端伺服器有一個 API，允許我們授權使用者。如果認證成功，伺服器將回傳一個認證的 token ，我們的應用程式使用 DOM storage 將 token 儲存（假設我們的 API 提供其他的 DOM storage 服務）。

當使用者登出時，我們將簡單的刪除先前儲存的認證 token。

### 初步嘗試

到目前為止我們所有需要的 Effect 是為了實現上面的流程，我們可以使用 `take` Effect 在 store 等待指定的 action，使用 `call` Effect 來非同步的呼叫，最後我們可以使用 `put` Effect dispatch action 到 store。

所以讓我們試試看吧：

> 注意：以下的程式碼有一些小問題。請務必將這個部份的解說讀完。

```javascript
import { take, call, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    const token = yield call(authorize, user, password)
    if (token) {
      yield call(Api.storeItem, {token})
      yield take('LOGOUT')
      yield call(Api.clearItem, 'token')
    }
  }
}
```

首先我們建立一個獨立的 Generator `authorize`，它將執行實際的 API 呼叫並在成功時通知 Store。  

`loginflow` 內的 `while (true)` 迴圈實作了一個完整的流程，意思說一旦我們流程到了最後一個步驟（`LOGOUT`），我們透過等待一個新的 `LOGIN_REQUEST` action 來開始一個新的迭代。

`loginFlow` 首先等待一個 `LOGIN_REQUEST` action，然後在 action payload（`user` 和 `password`）取得認證，並使用 `call` 到 `authorize` 的 task。

正如你所注意到的，`call` 不只可以調用一個 function 並回傳 Promise，我們也可以用來調用其他的 Generator function。在上面的範例中，**`loginFlow` 將等待一個 authorize 直到他終止被回傳**（也就是說在執行 api 呼叫後，dispatch action 並回傳 token 到 `loginFlow`）。

如果 API 呼叫成功，`authorize` 將 dispatch 一個 `LOGIN_SUCCESS` action 然後回傳取得的 token。如果結果失敗的話，將調用一個 `LOGIN_ERROR` action。

如果呼叫到 `authorize` 成功的話，`loginFlow` 將在 DOM storage 儲存回傳的 token 並等待一個 `LOGOUT` action。當使用者登出時，我們移除儲存的 token 並等待一個新的使用者登入。

如果在 `authroize` 失敗的情況下，將回傳一個 undefined 的值，它將導致 `loginFlow` 跳過先前的程序，並等待一個新的 `LOGIN_REQUEST` action。

觀察整個邏輯是如何儲存在同一個地方的。一個新的開發者閱讀我們的程式碼不需要在各個程式碼之間做切換就可以了解控制的流程。就像是閱讀同步的程式碼，它很自然的確定了它的步驟。而且我們有 function 可以呼叫其他的 function 並等待它們的結果。

### 但是上面的方法仍然存在一個小問題

假設當 `loginFlow` 再等待以下的呼叫被 reslove：

```javascript
function* loginFlow() {
  while (true) {
    // ...
    try {
      const token = yield call(authorize, user, password)
      // ...
    }
    // ...
  }
}
```

使用者按下 `Logout` 按鈕造成一個 `LOGOUT` action 被 dispatch。

下面的範例解釋了假設的事件序列︰

```
UI                              loginFlow
--------------------------------------------------------
LOGIN_REQUEST...................call authorize.......... waiting to resolve
........................................................
........................................................
LOGOUT.................................................. missed!
........................................................
................................authorize returned...... dispatch a `LOGIN_SUCCESS`!!
........................................................
```

當 `loginFlow` 在 `authorize` 呼叫後被阻塞，最終的呼叫和 response 發生的 `LOGOUT` 將被跳過，因為 `loginFlow` 還沒有執行 `yield take('LOGOUT')` 。

問題在上面 `call` 的程式碼， 它是一個阻塞的 Effect，也就是說 Generator 在呼叫結束之前，不能執行或處理任何東西。但在我們的情況中，我們不只想要 `loginFlow` 執行呼叫認證，也希望能觀察發生在呼叫的之間最終的 `LOGOUT` action。因為 `LOGOUT` 和 `authorize` 是*同時*呼叫的。


To express non-blocking calls, the library provides another Effect: [`fork`](https://redux-saga.js.org/docs/api/index.html#forkfn-args). When we fork a *task*, the task is started in the background and the caller can continue its flow without waiting for the forked task to terminate.

如果要表達非阻塞的呼叫，library 提供另一個 Effect：[`fork`](http://yelouafi.github.io/redux-saga/docs/api/index.html#forkfn-args)。當我們 fork 一個 *task*，task 在背景被啟動而且 caller 可以持續它的流程，不用等待被 fork 的 task 結束。

所以為了讓 `loginFlow` 跳過一個併發的 `LOGOUT`，我們不應該使用 `call` 和 `authroize` task，而是使用 `fork`。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'

function* loginFlow() {
  while (true) {
    ...
    try {
      // 非阻塞呼叫，這裡會回傳什麼值？
      const ?? = yield fork(authorize, user, password)
      ...
    }
    ...
  }
}
```

現在的問題是，自從我們的 `authroize` action 在背景被啟動後，我們不能取得 token 的結果（因為我們沒有等待它）。所以我們需要移動儲存 token 的操作到 `authorize` 的 task。

```javascript
import { fork, call, take, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    yield fork(authorize, user, password)
    yield take(['LOGOUT', 'LOGIN_ERROR'])
    yield call(Api.clearItem, 'token')
  }
}
```

我們也做了 `yield take(['LOGOUT', 'LOGIN_ERROR'])`。意思我說們觀察兩個併發的 action：

- 如果在使用登出之前 `authorize` task 成功，它將 dispatch 一個 `LOGIN_SUCCESS` action 並結束 task。我們的 `loginFlow` saga 將只會等待一個未來的 `LOGOUT` action（因為 `LOGIN_ERROR` 永遠不會發生）。

- 如果在使用登出之前 `authorize` task 失敗，它將 dispatch 一個 `LOGIN_ERROR` action 並結束 task。所以 `loginFlow` 在 `LOGOUT` 之前接收一個 `LOGIN_ERROR`，然後它將進入另外一個 `while` 迭代並等待下一個 `LOGIN_REQUEST` 的 action。

- 如果使用者在 `authroize` 終止前登出了，那麼 `loginFlow` 將接收一個 `LOGOUT` action 並等待下一個 `LOGIN_REQUEST`。

注意呼叫 `Api.clearItem` 應該是冪等的。如果呼叫 `authorize` 沒有儲存 token 也不會有任何影響。`loginFlow` 確保在等待下一個登入之前，沒有 token 被儲存。

如果我們在一個 API 呼叫期間接收一個 `LOGOUT` ，我們必須**取消** `authorize` 的程序，否則我們將會有兩個併發的 task 並行前進： `authorize` task 將持續執行並在成功（或失敗）時 dispatch 一個 `LOGIN_SUCCESS`（或是 `LOGIN_ERROR` action），這會導致 state 不一致。

為了取消一個被 fork 的 task，我們使用一個專屬的 Effec [`cancel`](https://redux-saga.js.org/docs/api/index.html#canceltask)

```javascript
import { take, put, call, fork, cancel } from 'redux-saga/effects'

// ...

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    // fork 回傳一個 Task 物件
    const task = yield fork(authorize, user, password)
    const action = yield take(['LOGOUT', 'LOGIN_ERROR'])
    if (action.type === 'LOGOUT')
      yield cancel(task)
    yield call(Api.clearItem, 'token')
  }
}
```

`yield fork` 結果在一個 [Task 物件](https://redux-saga.js.org/docs/api/index.html#task)。我們將回傳的物件分配到 local 常數 `task`。之後如果我們接收一個 `LOGOUT` action，我們傳送 task 到 `cancel` Effect。如果 task 持續執行，它將被中止。如果 task 已經完成，不會發生任何事情，取消操作的結果將是一個空操作（no-op）。最後，如果 task 完成後有錯誤，我們不會做任何事情，因為我們知道 task 已經完成。

我們*幾乎*要完成了（併發不是這麼簡單的；你需要認真以待）。

假設當我們接收一個 `LOGIN_REQUEST` action，我們的 reducer 設定一些 `isLoginPending` 的 flag 為 true，並在 UI 顯示一些訊息或 spinner。如果我們在 API 呼叫期間取得一個 `LOGOUT` 並簡單透過 *kill* 方式來中止 task（就是 task 被停止），然後我們可能又以不一致的 state 結束了。我們有一個 `isLoginPending` 設為 true，而且我們的 reducer 正在等待一個結果的 action（`LOGIN_SUCCESS` 或 `LOGIN_ERROR`）。

幸運的是，`cancel` Effect 不會殘酷的 kill 我們的 `authroize` task，相反的它會給予一個機會執行清除的邏輯。在 `finally` 區塊可以取消 task 或處理任何的取消邏輯（以及任何其他類型的完成）。因為最後區塊執行在完成的任何類型（正常的回傳、錯誤、或強制取消），如果你想要特殊的處理取消的方式，這裡有一個 Effect 被 `cancel`：

```javascript
import { take, call, put, cancelled } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  } finally {
    if (yield cancelled()) {
      // ... 在這裡放置特殊的取消操作程式碼
    }
  }
}
```

你可能已經注意到我們還沒完成任何關於 `isLoginPending` state 的事情。關於這一點，至少有兩個可能的解決方式︰

- dispatch 一個專門的 `RESET_LOGIN_PENDING` action
- 更簡單的，讓 reducer 清除在 `LOGOUT` action 接收到的 `isLoginPending`。
