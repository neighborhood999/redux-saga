# 使用 Channel

到目前為止，我們使用了 `take` 和 `put` effect 來和 Redux Store 溝通。Channels 概括了這些 Effect 與外部事件來源或 Saga 它們之間的溝通。它們也可以從 Store 指定隊列的 action。

在這個部份，我們將會看到：

- 如何從 Store 使用 `yield actionChannel` Effect 來緩衝指定的 action。

- 如何使用 `eventChannel` factory function 連結 `take` Effect 到外部的事件來源。

- 如何使用通用的 `channel` factory function 建立一個 channel，並在兩個 Saga 之間使用 `take` 和 `put` Effect 做溝通。

## 使用 `actionChannel` Effect

讓我們回顧一下經典的範例：

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

上方的範例說明典型的 *watch-and-fork* 的模式，`watchRequests` saga 使用 `fork` 來避免阻塞，因此不會錯過任何來自 store 的 action。`handleRequest` task 是在每次得到 `REQUEST` action 時被建立，所以如果有許多 action 在一個 race 被觸發，可以同時執行許多 `handleRequest` task。

想像現在我們需要以下的功能：我們想要每次只處理一個 `REQUEST`，意思是如果我們在一個時間點有四個 action，我們想要一個一個處理 `REQUEST` action，處理完第一個 action 後，再接著處理第二個 action...。

所以我們想要的是*隊列*所有還沒被處理的 action，一旦我們處理完目前的 request，我們可以從隊列取得下一個訊息。

library 提供一個 `actionChannel` helper Effect，讓我們可以處理這些東西。讓我們來看如何使用它重新撰寫先前的範例：

```javascript
import { take, actionChannel, call, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 1- 建立一個 channel 給 request action
  const requestChan = yield actionChannel('REQUEST')
  while (true) {
    // 2- 從 channel 取得
    const {payload} = yield take(requestChan)
    // 3- 注意，我們使用一個阻塞的呼叫
    yield call(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

第一件事情是建立一個 action channel，我們使用 `yield actionChannel(pattern)`，這個 pattern 被解讀成我們先前提到的 `take(pattern)` 並使用相同的規則。這兩個形式不同的地方是，如果 Saga 還沒準備好接收它們的話（例如一個被阻塞的 API 呼叫），`actionChannel` **可以緩衝傳入的訊息**。

接下來是 `yield take(requestChan)`，除了使用一個 `pattern` 從 Redux Store 接收指定的 action 之外，`take` 也可以被用在 channel（在上面我們從指定的 Redux Store 建立 channel 物件）。`take` 可以阻塞 Saga，直到在 channel 有一個可用的訊息。如果有一個訊息被儲存在基礎緩衝區，take 也可以立即的恢復。

最重要的是注意到我們如何使用一個阻塞的 `call`。Saga 將停留在阻塞狀態，直到 `call(handleRequest)` 回傳，但如果其他的 `REQUEST` action 被 dispatch，而 Saga 仍然被阻塞時，透過 `requestChan` 被隊列在內部。當 Saga 從 `call(handleRequest)` 恢復並執行下一個 `yield take(requestChan)`，take 將 resolve 被隊列的訊息。

預設上，`actionChannel` 沒有限制緩衝所有傳入的訊息。如果你想要更多的緩衝控制，你可提供一個 Buffer 的參數到 effect creator。library 提供一些普遍的 buffer（none、dropping、sliding），但你也可以提供你自己的 buffer 實作，更多細節請參考 [API 文件](../api#buffers)。

例如，如果我們只想要處理最近的五筆資料，你可以使用：

```javascript
import { buffers } from 'redux-saga'
import { actionChannel } from 'redux-saga/effects'

function* watchRequests() {
  const requestChan = yield actionChannel('REQUEST', buffers.sliding(5))
  ...
}
```

## 使用 `eventChannel` factory 連結外部的事件

像是 `actionChannel`（Effect）、`eventChannel`（一個 factory function，而不是一個 Effect）為 Redux Store 以外的事件來源建立一個 Channel。

這是一個從 interval 建立 Channel 的範例：

```javascript
import { eventChannel, END } from 'redux-saga'

function countdown(secs) {
  return eventChannel(emitter => {
      const iv = setInterval(() => {
        secs -= 1
        if (secs > 0) {
          emitter(secs)
        } else {
          // 這裡造成 channel 關閉
          emitter(END)
        }
      }, 1000);
      // 訂閱者必須回傳取消訂閱功能
      return () => {
        clearInterval(iv)
      }
    }
  )
}
```

`eventChannel` 第一個參數是一個 *subscriber* function。訂閱者的規則是初始化外部的來源（上面使用 `setInterval`），透過提供的 `emitter` 將來源路由所有傳入的事件調用到 channel。在上面的範例我們在每秒調用 `emitter`：

> 注意：你需要清除你的事件來源，不是通過事件 channel 傳送 null 或 undefined。雖然可以透過數字傳送，但我們推薦像是 redux action 一樣，組織你的事件 channel 資料。

注意，也可以調用 `emitter(END)`。channel 被關閉時，我們使用 `emitter(END)` 來通知所有 channel consumer，意思是沒有其他的訊息可以可以通過這個 channel。

讓我看一下如何從 Saga 使用這個 channel。（這個範例是來自 repo 的 cancellable-counter ）

```javascript
import { take, put, call } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// 在每秒間隔建立一個事件 Channel
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {    
    while (true) {
      // take(END) 將造成 saga 終止，跳到 finally 區塊
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    console.log('countdown terminated')
  }
}
```

所以 Saga yield 一個 `take(chan)` 造成阻塞，直到一個訊息被 put 在 channel。在我們上面的範例，它對應到我們調用 `emitter(secs)`，注意我們還在在一個 `try/finally` 區塊執行整個 `while (true {...}` 迴圈。當間隔終止時，countdown function 透過調用 `emitter(END)` 關閉 channel。在 channel 的 `take` effect 關閉 channel 終止所有被阻塞的 Saga。在我們的範例，終止 Saga 將造成它跳到 `finally` 區塊（如果有提供的話，否則 Saga 只是簡單的終止）。

訂閱者回傳一個 `unsubscribe` function，這是被用來在事件來源完成之前，透過 channel 取消訂閱。在 Saga 內使用來自事件 channel 的訊息，如果我們想要在事件來源完成之前*提早離開*（例如：Saga 已經被取消），你可以從來源呼叫 `chan.close()` 關閉 channel 並取消訂閱。

例如，我們可以讓我們的 Saga 支援取消：

```javascript
import { take, put, call, cancelled } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// 在每秒間隔建立一個事件 Channel
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {    
    while (true) {
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    if (yield cancelled()) {
      chan.close()
      console.log('countdown cancelled')
    }    
  }
}
```

這裡是另一個例子，你如何使用事件 channel 去傳送 WebSockeet 事件到你的 saga（例如：使用 socket.io library）。
假設你等待伺服器的一個 `ping` 訊息，然後在 delay 後回覆一個 `pong` 訊息。

```javascript
import { take, put, call, apply } from 'redux-saga/effects'
import { eventChannel, delay } from 'redux-saga'
import { createWebSocketConnection } from './socketConnection'

// 這個 function 從一個指定的 socket 建立一個 event channel
// 設定傳入 `ping` events 的 subscription
function createSocketChannel(socket) {
  // `eventChannel` 接收一個 subscriber function
  // subscriber function 接收一個 `emit` 參數，把 message 放到 channel 上 the channel
  return eventChannel(emit => {

    const pingHandler = (event) => {
      // 放入 event payload 到 channel
      // 這可以讓 Saga 從被回傳的 channel 接收 payload
      emit(event.payload)
    }

    // 設定 subscription
    socket.on('ping', pingHandler)

    // subscriber 必須回傳一個 unsubscribe function
    // 當 saga 呼叫 `channel.close` 方法將會被調用
    const unsubscribe = () => {
      socket.off('ping', pingHandler)
    }

    return unsubscribe
  })
}

// 透過調用的 `socket.emit('pong')` 回傳一個 `pong` 訊息
function* pong(socket) {
  yield call(delay, 5000)
  yield apply(socket, socket.emit, ['pong']) // 呼叫 `emit` 作為一個方法並以 `socket` 作為 context
}

export function* watchOnPings() {
  const socket = yield call(createWebSocketConnection)
  const socketChannel = yield call(createSocketChannel, socket)

  while (true) {
    const payload = yield take(socketChannel)
    yield put({ type: INCOMING_PONG_PAYLOAD, payload })
    yield fork(pong, socket)
  }
}
```

> 注意：預設上，訊息在一個 eventChannel 不會被緩衝。你可以提供一個緩衝到 eventChannel factory 來指定 channel 的緩衝策略（例如：`eventChannel(subscriber, buffer)`）。
更多資訊請參考 [API 文件](../api#buffers)。

### 使用 channel 在 Saga 之間溝通

除了 action channel 和事件 channel 之外，你也可以直接建立 channel，預設上可以不用連結任何的來源，你可以在 channel 手動的 `put`。當你想要在 saga 之間使用 channel 溝通是非常方便的。

為了說明，讓我們回顧先前的請求操作範例：

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

我們可以看到 watch-and-fork pattern 允許我們同時操作多個請求，在併行下，沒有工作 task 的數量限制。然後我們使用 `actionChannel` effect 來限制併發一次執行一個 task。

因此，我們要求在同一時間內執行三個 task，當我們取得一個 request，而且執行的 task 小於三個，我們會立即的處理 request，但是如果我們已經有三個 task 執行了，我們將 task 隊列，並等待其中一個 *slots* 完成。

下面的範例使用 channel 解決：

```javascript
import { channel } from 'redux-saga'
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 建立一個 channel 隊列傳入的請求
  const chan = yield call(channel)

  // 建立三個 worker thread
  for (var i = 0; i < 3; i++) {
    yield fork(handleRequest, chan)
  }

  while (true) {
    const {payload} = yield take('REQUEST')
    yield put(chan, payload)
  }
}

function* handleRequest(chan) {
  while (true) {
    const payload = yield take(chan)
    // 處理請求
  }
}
```

在上面的範例中，我們使用 `channel` factory 建立一個 channel。我們取回一個 channel，預設上我們放入所有緩衝的訊息（除非有一個正在等待的 taker，如果有訊息的話立即恢復 taker）。

`watchRequests` saga fork 三個 worker saga。注意，建立的 channel 提供給所有被 fork 的 saga，`watchRequests` 將使用這個 channel 來 *dispatch* 工作到三個 worker saga。在每個 `REQUEST` action，Saga 簡單的在 channel 放入 payload，任何**空閒**的 worker 會接收 payload，也就是說它將透過 channel 被隊列，直到一個 woker Saga 準備接收它。

所有的 worker 執行一個典型的 while 迴圈。在每次迭代 worker 將取得下一次的 request，或者阻塞直到有可用的訊息。注意，這個機制在三個 worker 之間提供一個自動載入平衡。
