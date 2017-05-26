# redux-saga 的 fork 模型

在 `redux-saga` 你可以使用兩種 Effect 的動態執行 fork task 。

- `fork` 被用來建立*被附加的 forks*
- `spawn` 被用來建立*被分離的 forks*

## 被附加的 forks（使用 `fork`）

透過以下規則附加 fork 到 parent：

### 完成

- 一個 Saga 只終止在：
  - 它說明終止自己本身的 body 之後
  - 所有被附加的 fork 都被終止之後

例如，我們有以下的程式碼：

```js
import { delay } from 'redux-saga'
import { fork, call, put } from 'redux-saga/effects'
import api from './somewhere/api' // 特定 app
import { receiveData } from './somewhere/actins' // 特定 app

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield call(delay, 1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  yield call(fetchAll)
}
```

`call(fetchAll)` 將會終止在：

- `fetchAll` body 本身終止之後；意思說，三個所有的 effects 都被執行。由於 `fork` effect 是非同步阻塞，task 將阻塞在 `call(delay, 1000)`

- 兩個被 fork 的 task 終止，例如：在 fetch 之後，請求的 resource 被 put 到對應的 `receiveData` actions

所以整個 task 將被阻塞，直到一個 1000 毫秒的 delay 通過**以及** `tasks1` 和 `task2` 完成他們的商業邏輯。

比方說，1000 毫秒的 delay 和兩個 task 還沒有完成，然後 `fetchAll` 在終止整個 task 之前，將直到等待所有被 fork 的 task 完成。

細心的讀者可能會注意到 `fetchAll` saga 可以使用平行的 Effect 覆寫：

```js
function* fetchAll() {
  yield all([
    call(fetchResource, 'users'),     // task1
    call(fetchResource, 'comments'),  // task2,
    call(delay, 1000)
  ])
}
```

事實上，被附加的 fork 與平行的 Effect 共享相同的語意：

- 我們在 parallel 的狀況下執行 task
- 在所有被發出的 task 終止之後，父 task 將會終止


這個適用於所有其他語意（錯誤和取消傳播）。只考慮它作為一個*動態平行*的 Effect，你可以了解如何附加 fork 行為。

## 錯誤傳播

以此類推，讓我們在 parallel 的 Effect 中來研究如何處理錯誤。

例如，我們有這個 Effect：

```js
yield all([
  call(fetchResource, 'users'),
  call(fetchResource, 'comments'),
  call(delay, 1000)
])
```

如果三個子 Effect 其中一個失敗，會造成其他都失敗。此外，未捕獲的錯誤將造成 parallel 的 Effect 取消所有其他等待的 Effect。例如，如果 `call(fetchResource), 'users')` 發出一個未捕獲的錯誤，parallel 的 Effect 將取消其他兩個 task（如果他們仍再等待），並從失敗的呼叫中，abort 本身具有相同的錯誤。

同樣為附加的 fork，一個 Saga abort 一個 Saga

- 說明主要 body 拋出一個錯誤

- 透過其中一個被附加的 fork，發起一個未捕獲的錯誤

所以在先前範例：

```js
//... imports

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield call(delay, 1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  try {
    yield call(fetchAll)
  } catch (e) {
    // 處理 fetchAll 錯誤
  }
}
```

例如，如果在這個時候 `fetchAll` 被阻塞在 `call(delay, 1000)` Effect，並說明 `task1` 失敗了，整個 `fetchAll` task 將造成失敗。

- 其他尚未完成的 task 被取消。這些包含：  
  - *main task* （`fetchAll` 的 body）: 取消意思是，取消目前的 Effect `call(delay, 1000)`  
  - 其他被 fork 的 task 仍然在等待。例如在我們的 `task2` 範例。

- `call(fetchAll)` 本身發出一個錯誤的話，將會在 `main` 的 `catch` 捕獲錯誤。

注意，因為我們使用阻塞呼叫，所以只能從 `main` 裡面的 `call(fetchAll)` 捕獲錯誤。我們並不能直接從 `fetchAll` 捕獲錯誤。這是一個經驗法則，**你不能從被 fork 的 task 捕獲錯誤**。一個被附加的 fork 失敗時，會造成父 fork 被 abort（就像沒有辦法在一個平行 Effect *內部*捕獲錯誤，只能透過外部阻塞 parallel 的 Effect）。


## 取消

取消一個 Saga 造成：

- *main task* 意思是，在 Saga 被阻塞的地方，取消目前的 Effect

- 所有被附加的 fork 仍然繼續執行


**WIP**

## 分離 forks（使用 `spawn`）

 分離的 fork 存在他們本身執行的 context。父 task 不會等待被分離的 fork 終止。從被 spawn 的 task 未捕獲的錯誤不會被冒泡到父 task。而且取消一個父 task 不會自動的取消被分離的 fork（你需要顯式 cancel 他們）。

簡單來說，分離 fork 行為像是使用 `middleware.run` API 直接啟動 root Saga。


**WIP**
