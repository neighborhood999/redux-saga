# 取消 Task

我們在[非阻塞呼叫](NonBlockingCalls.md)部份已經看過取消的範例。在這個部份我們將回顧更多取消 task 的細節。

一旦 task 被 fork，你可以使用 `yield cancel(task)` 中止它。

我們來看一下它是如何運作的，讓我們考慮一個簡單的範例：可以透過一些 UI 的 command 啟動或暫停背景同步。接收一個 `START_BACKGROUND_SYNC` action，我們 fork 一個背景 task 將定期的從遠端資料庫同步一些資料。

task 將持續執行直到一個 `STOP_BACKGROUND_SYNC` action 被觸發。然後我們取消背景 task 並等待下一次的 `START_BACKGROUND_SYNC` action。   

```javascript
import { take, put, call, fork, cancel, cancelled } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { someApi, actions } from 'somewhere'

function* bgSync() {
  try {
    while (true) {
      yield put(actions.requestStart())
      const result = yield call(someApi)
      yield put(actions.requestSuccess(result))
      yield call(delay, 5000)
    }
  } finally {
    if (yield cancelled())
      yield put(actions.requestFailure('Sync cancelled!'))
  }
}

function* main() {
  while ( yield take(START_BACKGROUND_SYNC) ) {
    // 在背景啟動 task
    const bgSyncTask = yield fork(bgSync)

    // 等待 user 的 stop action
    yield take(STOP_BACKGROUND_SYNC)
    // user 按下暫停，取消背景任務
    // 這將導致被 fork 的 bgSync task 跳到它最後的 finally 區塊
    yield cancel(bgSyncTask)
  }
}
```

在上方的範例中，`bgSyncTask` 的取消將造成 Generator 跳到 finally 的區塊。這裡你可以使用 `yield cancelled()` 來確認 Generator 是否被取消了。

在取消一個執行的 task 同時也取消目前被阻塞的 Effect。

例如，在一個應用程式的生命週期的某個時候，我們有一個 pending 的 call chain（鏈結）：

```javascript
function* main() {
  const task = yield fork(subtask)
  ...
  // 接著
  yield cancel(task)
}

function* subtask() {
  ...
  yield call(subtask2) // 在這個 call 目前被阻塞
  ...
}

function* subtask2() {
  ...
  yield call(someApi) // 在這個 call 目前被阻塞
  ...
}
```

`yield cancel(task)` 在 `subtask` 觸發一個取消，反過來又將在 `subtask2` 觸發一個取消。

現在我們看到取消不斷的往下傳播（相反的，被回傳的值和沒有捕捉的錯誤不斷往上）。你可以看到 caller（調用非同步的操作）和 callee（被調用的操作）之間的*對照*。callee 是負責執行操作。如果它完成了（不管是成功或失敗）結果將會往上到它的 caller，最終到 caller 的調用方。就是這樣，callee 是負責*完成流程*。

現在如果 callee 一直處於等待，而且 caller 決定取消操作，它將觸發一種訊號往下傳播到 callee（以及透過 callee 本身被呼叫的任何深層操作）。所有深層等待的操作將被取消。

如果加入的 task 被取消的話，task 的 joiner（那些被阻塞的 `yield join(task)`）將也會被取消。同樣的，任何那些 joiner 潛在的 caller 將會被取消（因為他們阻塞的操作已經從外面被取消）。

## Testing generators with fork effect

When `fork` is called it starts the task in the background and also returns task object like we have learned previously. When testing this we have to use utility function `createMockTask`. Object returned from this function should be passed to next `next` call after fork test. Mock task can then be passed to `cancel` for example. Here is test for `main` generator which is on top of this page.

```javascript
import { createMockTask } from 'redux-saga/utils';

describe('main', () => {
  const generator = main();

  it('waits for start action', () => {
    const expectedYield = take(START_BACKGROUND_SYNC);
    expect(generator.next().value).to.deep.equal(expectedYield);
  });

  it('forks the service', () => {
    const expectedYield = fork(bgSync);
    expect(generator.next().value).to.deep.equal(expectedYield);
  });

  it('waits for stop action and then cancels the service', () => {
    const mockTask = createMockTask();

    const expectedTakeYield = take(STOP_BACKGROUND_SYNC);
    expect(generator.next(mockTask).value).to.deep.equal(expectedTakeYield);

    const expectedCancelYield = cancel(mockTask);
    expect(generator.next().value).to.deep.equal(expectedCancelYield);
  });
});
```

You can also use mock task's functions `setRunning`, `setResult` and `setError` to set mock task's state. For example `mockTask.setRunning(false)`.

### 注意

很重要的是，記得 `yield cancel(task)` 不會等待被取消的 task 結束（也就是說執行到 final 區塊）。cancel effect 的行為像是 fork，它初始 cancel 後並回傳，一旦被取消後，task 應該執行清除邏輯並回傳。

## 自動取消

除了手動取消之外，也有一些是被觸發而自動取消的情況

1. 在一個 `rafe` effect，除了 winner 之外，所有 race 競爭者都會被取消。

2. 在一個併行的 effect（`yield [...]`）中，一旦其中一個子 effect 被 reject，併行的 effect 會被 reject（像是 `Promise.all` 一樣）。在這個情況，所有其他的子 effect 都會自動的被取消。
