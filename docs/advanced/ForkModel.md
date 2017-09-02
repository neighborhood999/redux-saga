# redux-saga 的 fork 模型

在 `redux-saga` 你可以使用兩種 Effect 動態的在背景執行 fork task

- `fork` 是用來建立被*附加的 fork*
- `spawn` 是用來建立被*分離的 fork*

## 被附加的 fork （使用 `fork`）

被附加的 fork 透過以下的規則繼續附加在它們的 parent

### 結論

- Saga 只會在這之後終止：
  - body 的說明它終止了自己本身
  - 所有被附加的 fork 被終止

例如我們有以下的情形：

```js
import { delay } from 'redux-saga'
import { fork, call, put } from 'redux-saga/effects'
import api from './somewhere/api' // app specific
import { receiveData } from './somewhere/actions' // app specific

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

`call(fetchAll)` 將在這之後終止：

- `fetchAll` 本身被終止，這意思是這三個 effect 都會被執行。由於 `fork` Effect 不是非阻塞的，task 將被阻塞在 `call(delay, 1000)`

- 兩個被 fork 的 task 終止，意思是在 fetch 所需的資源並放入對應的 `receiveData` action

所以整個 task 將阻塞直到一個 1000 毫秒的 delay 被傳送，*且* `task1` 和 `task2` 完成他們的任務。

比方說，1000 毫秒的 delay 和兩個 task 還沒有完成，然後 `fetchAll` 在終止整個 task 之前，將等待直到所有被 fork 的 task 完成。

細心的讀者可能會注意到 `fetchAll` saga 使用平行 Effect 被改寫

```js
function* fetchAll() {
  yield all([
    call(fetchResource, 'users'),     // task1
    call(fetchResource, 'comments'),  // task2,
    call(delay, 1000)
  ])
}
```

事實上，被附加的 fork 與平行 Effect 共享相同的語意：

- 在平行情況下我們執行 task
- 在所有被 launch 的 task 終止後，paraent 將會終止


這也適用於其他語意（錯誤和取消傳播）。你可以簡單的把它考慮作為一個*動態平行* Effect，了解附加 fork 的行為。

## Error 傳播

按照同樣的比喻，我們來詳細的檢查在平行的 Effect 如何處理錯誤

例如，假設我們有這個 Effect

```js
yield all([
  call(fetchResource, 'users'),
  call(fetchResource, 'comments'),
  call(delay, 1000)
])
```

一旦其中一個子 Effect 失敗，上方的 Effect 就會失敗。此外，未捕獲的錯誤將會造成平行 Effect 取消所有其他 pending 的 Effect。例如，如果 `call(fetchResource, 'users')` 發出了一個未捕獲的錯誤，平行 Effect 將會取消其他兩個 task（如果他們仍然在 pending），並從失敗的呼叫中，終止自己本身的錯誤。

類似於被 attach 的 forks，Saga 會在以下情況馬上終止

- 它主要 body 的說明拋出了一個錯誤

- 一個未捕獲的錯誤透過其中一個被 attach 的 forks 被發出

所以在先前的範例：

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

如果在這樣的情況，例如 `fetchAll` 被阻塞在 `call(delay, 1000)` Effect，並說明 `task1` 失敗，然後整個 `fetchAll` task 將會因此失敗

- 取消所有其他在 pending 的 task。這包含：
  - *main task*（`fetchAll` 的本身）：取消的意思是，取消目前的 `call(delay, 1000)` 的 Effect
  - 其他被 fork 的 task 仍然在 pending。例如在我們範例的 `task2`。

- 在 `main` 的 `catch` 內將會捕獲由 `call(fetchAll)` 發出的錯誤

注意，因為我們使用一個阻塞的 call，所以我們只能從 `main` 內部 catch `call(fetchAll)` 的錯誤，而且我們不能直結從 `fetchAll` 捕捉錯誤。這是首要的原則，**你不能從被 fork 的 task 捕捉錯誤**。在一個被附加的 fork 錯誤將會導致被 fork 的 parent 被終止（就像沒有方法在一個平行 Effect *內*捕捉錯誤，只能從外部透過被阻塞的的平行 Effect）。


## Cancellation

取消 Saga 導致:

- *main task* 這意思是當 Saga 被阻塞時，取消目前的 Effect

- 所有被附加的 fork 仍然繼續執行


**WIP**

## 被分離的 fork（使用 `spawn`）

被分離的 fork 存活在它們本身執行的 context。Parent 不會等待被分離的 fork 終止。從被 spawn 的 task 為捕獲的錯誤不會被冒泡到 parent，而且取消一個 parent 並不會自動的取消被分離的 fork（你需要明確的取消它們）。

簡單來說，被分離的 fork 行為像是使用 `middleware.run` API 直接啟動 root Saga。


**WIP**
