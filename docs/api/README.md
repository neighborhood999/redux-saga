# API 參考

* [`Middleware API`](#middleware-api)
  * [`createSagaMiddleware(options)`](#createsagamiddlewareoptions)
  * [`middleware.run(saga, ...args)`](#middlewarerunsaga-args)
* [`Saga Helpers`](#saga-helpers)
  * [`takeEvery(pattern, saga, ...args)`](#takeeverypattern-saga-args)
  * [`takeLatest(pattern, saga, ..args)`](#takelatestpattern-saga-args)
  * [`throttle(ms, pattern, saga, ..args)`](#throttlems-pattern-saga-args)
* [`Effect creators`](#effect-creators)
  * [`take(pattern)`](#takepattern)
  * [`take.maybe(pattern)`](#takemaybepattern)
  * [`take(channel)`](#takechannel)
  * [`take.maybe(channel)`](#takemaybechannel)
  * [`put(action)`](#putaction)
  * [`put.resolve(action)`](#putresolveaction)
  * [`put(channel, action)`](#putchannel-action)
  * [`call(fn, ...args)`](#callfn-args)
  * [`call([context, fn], ...args)`](#callcontext-fn-args)
  * [`call([context, fnName], ...args)`](#callcontext-fnname-args)
  * [`apply(context, fn, args)`](#applycontext-fn-args)
  * [`cps(fn, ...args)`](#cpsfn-args)
  * [`cps([context, fn], ...args)`](#cpscontext-fn-args)
  * [`fork(fn, ...args)`](#forkfn-args)
  * [`fork([context, fn], ...args)`](#forkcontext-fn-args)
  * [`spawn(fn, ...args)`](#spawnfn-args)
  * [`spawn([context, fn], ...args)`](#spawncontext-fn-args)
  * [`join(task)`](#jointask)
  * [`join(...tasks)`](#jointasks)
  * [`cancel(task)`](#canceltask)
  * [`cancel(...tasks)`](#canceltasks)
  * [`cancel()`](#cancel)
  * [`select(selector, ...args)`](#selectselector-args)
  * [`actionChannel(pattern, [buffer])`](#actionchannelpattern-buffer)
  * [`flush(channel)`](#flushchannel)
  * [`cancelled()`](#cancelled)
  * [`setContext(props)`](#setcontextprops)
  * [`getContext(prop)`](#getcontextprop)
* [`Effect combinators`](#effect-combinators)
  * [`race(effects)`](#raceeffects)
  * [`all([...effects]) (aka parallel effects)`](#alleffects---parallel-effects)
  * [`all(effects)`](#alleffects)
* [`Interfaces`](#interfaces)
  * [`Task`](#task)
  * [`Channel`](#channel)
  * [`Buffer`](#buffer)
  * [`SagaMonitor`](#sagamonitor)
* [`External API`](#external-api)
  * [`runSaga(options, saga, ...args)`](#runsagaoptions-saga-args)
* [`Utils`](#utils)
  * [`channel([buffer])`](#channelbuffer)
  * [`eventChannel(subscribe, [buffer], matcher)`](#eventchannelsubscribe-buffer-matcher)
  * [`buffers`](#buffers)
  * [`delay(ms, [val])`](#delayms-val)
  * [`cloneableGenerator(generatorFunc)`](#cloneablegeneratorgeneratorfunc)
  * [`createMockTask()`](#createmocktask)


# Cheatsheets

* [Blocking / Non-blocking](#blocking--nonblocking)

## Middleware API

### `createSagaMiddleware(options)`

建立一個 Redux middleware 並連結 Saga 到 Redux Store。

- `options: Object` - 傳送到 middleware 的選項清單，目前支援的選項：

 - `sagaMonitor` : [SagaMonitor](#sagamonitor) - 如果提供一個 Saga Monitor，middleware 將提供監視事件給 monitor。

 - `emitter` : 將 redux 的 action 傳送給 redux saga（通過 redux middleware）。Emitter 是一個 higher order function，它內建一個 builtin emitter 並回傳另一個 emitter。

      **範例**

      在以下的範例我們建立一個 emitter，它「unpack」action 的陣列，並且 emit 從陣列被提取的 action。

     ```javascript
     createSagaMiddleware({
       emitter: emit => action => {
        if (Array.isArray(action)) {
          action.forEach(emit);
          return
        }
        emit(action);
       }
     });
     ```

  - `logger` : Function -  定義一個自訂的 logger middleware。預設上，middleware 記錄所有錯誤並在 console 中警告。告訴 middleware 傳送錯誤或警告到提供的 logger。被呼叫的 logger 與它的參數 `(level, ...args)`，第一個說明記錄的層級：('info', 'warning' or 'error')。其餘部分對應於以下參數（你可以使用 `args.join(' ') 來連接所有參數成為一個單一的字串`）。

  - `onError` : Function - 如果有提供的話，middleware 將從 Saga 呼叫它與未捕獲錯誤。對於傳送未捕獲例外到追蹤錯誤服務非常有用。

#### 範例

以下我們將建立一個 `configureStore` function，增強 Store 並新增一個 `runSaga` 方法。然後在我們主要的 module 中，使用這個方法來啟動應用程式的 root Saga。

**configureStore.js**
```javascript
import createSagaMiddleware from 'redux-saga'
import reducer from './path/to/reducer'

export default function configureStore(initialState) {
  // 注意：將 middleware 做為最後一個參數傳送給 createStore 要求 redux@>=3.1.0 以上的版本
  const sagaMiddleware = createSagaMiddleware()
  return {
    ...createStore(reducer, initialState, applyMiddleware(/* other middleware, */sagaMiddleware)),
    runSaga: sagaMiddleware.run
  }
}
```

**main.js**
```javascript
import configureStore from './configureStore'
import rootSaga from './sagas'
// ... 其他 imports

const store = configureStore()
store.runSaga(rootSaga)
```

#### 注意

觀察以下在 `sagaMiddleware.run` 方法的更多資訊。

### `middleware.run(saga, ...args)`

只在 `applyMiddleware` 階段**之後**動態執行 Saga。

- `saga: Function`: 一個 Generator function
- `args: Array<any>`: 提供給 `saga` 的參數

這個方法回傳一個 [Task descriptor](#task-descriptor)。

#### 注意

`saga` 必須是一個 function 且回傳一個 [Generator 物件](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator)。middleware 將迭代 Generator 並執行所有被 yield 的 Effect。

`saga` 可能也使用 library 提供的各種 Effect 來啟動其他 saga。迭代處理程序也使用於下面所有的子 saga。

在第一個迭代，middleware 調用 `next()` 方法來取得下一個 Effect，然後 middleware 透過 Effect API 執行被指定的 yield Effect。在這個同時，Generator 將暫停直到 effect 結束執行。收到的執行結果後，middleware 在 Generator 呼叫 `next(result)`，將它取得結果作為參數傳送。或拋出一些錯誤。

相反的，如果執行結果是一個錯誤 (根據每個 Effect creator 的指定)，Generator 的 `throw(error)` 會被呼叫，如果 Generator function yield 指令在 `try/catch` 區塊，透過 runtime 的 Generator 調用 `catch` 區塊，runtime 也會調用任何對應的 finally 區塊。

在 Saga 被取消的情況（也許是手動或是使用提供的 Effect），middleware 將調用 Generator 的 `return()` 方法。這會造成 Generator 跳過目前的 finally 區塊。

## Saga Helpers

> 注意：以下 function 是 helper function 建立在 Effect creator 之下。

### `takeEvery(pattern, saga, ...args)`

每次 dispatch 的 action 符合 `pattern` 時，產生一個 `saga`。

- `pattern: String | Array | Function` - 更多資訊請參考 [`take(pattern)`](#takepattern) 文件

- `saga: Function` - 一個 Generator function

- `args: Array<any>` - 啟動 task 時傳送的參數。`takeEvery` 將傳入的 action 加入到參數列表（意思說，action 將做為最後一個參數提供給 `saga`）

#### 範例

以下範例，我們建立一個簡單的 `fetchUser` task，在每次 dispatch `USER_REQUESTED` action 時，使用 `takeEvery` 來啟動一個新的 `fetchUser` task：

```javascript
import { takeEvery } from `redux-saga/effects`

function* fetchUser(action) {
  ...
}

function* watchFetchUser() {
  yield takeEvery('USER_REQUESTED', fetchUser)
}
```

#### 注意

`takeEvery` 是一個高階 API，使用 `take` 和 `fork` 建立。這裡是如何使用低階 Effects 實作 helper

```javascript
const takeEvery = (pattern, saga, ...args) => fork(function*() {
  while (true) {
    const action = yield take(pattern)
    yield fork(saga, ...args.concat(action))
  }
})
```

`takeEvery` 允許處理併發的 action。在上面的範例中，當一個 `USER_REQUESTED` action 被 dispatch，新的 `fetchUser` task 會被啟動，即使之前的 `fetchUser` 還在等待（例如，使用者快速的在一個 `Load User` 按鈕按了兩次，第二次點擊仍然會 dispatch 一個 `USER_REQUESTED` action，即使第一個觸發的 `fetchUser` 還沒結束）。

`takeEvery` 不會處理多個 task 的 response 排序。這不會保證 task 按照啟動的順序結束，如果要處理 response 的排序，你可能考慮以下的 `takeLatest`。

### `takeLatest(pattern, saga, ...args)`

在每次 dispatch 的 action 和符合 `pattern` 時，產生一個 `saga`，並自動取消先前啟動而且可能在執行的 `saga`。

在每次 action 符合 `pattern`，而且被 dispatch 到 store 時，`takeLatest` 會在背景啟動一個新的 `saga`，如果 `saga` task 在先前被啟動（在最後被 dispatch action 之前實際的 action），task 仍然會被取消。

- `pattern: String | Array | Function` - 更多資訊請參考 [`take(pattern)`](#takepattern) 文件

- `saga: Function` - 一個 Generator function

- `args: Array<any>` - 啟動 task 時傳送的參數。`takeLatest` 將傳入的 action 加入到參數列表（意思說，action 將做為最後一個參數提供給 `saga`）

#### 範例

在以下的範例，我們建立一個簡單的 `fetchUser` task，在每次 dispatch `USER_REQUESTED` action 時，使用 `takeLatest` 來啟動一個新的 `fetchUser` task。由於 `takeLatest` 取消任何先前啟動等待的 task，我們要確保如果使用者快速觸發多個 `USER_REQUESTED` action 時，只會得到最後一個 action。

```javascript
import { takeLatest } from `redux-saga/effects`

function* fetchUser(action) {
  ...
}

function* watchLastFetchUser() {
  yield takeLatest('USER_REQUESTED', fetchUser)
}
```

#### 注意

`takeLatest` 是一個高階 API，使用 `take` 和 `fork` 建立。這裡是如何使用低階 Effects 實作 helper

```javascript
const takeLatest = (pattern, saga, ...args) => fork(function*() {
  let lastTask
  while (true) {
    const action = yield take(pattern)
    if (lastTask) {
      yield cancel(lastTask) // 如果 task 已經結束，cancel 是一個空操作
    }
    lastTask = yield fork(saga, ...args.concat(action))
  }
})
```

### `throttle(ms, pattern, saga, ...args)`

在一個 action 符合 `pattern` 被 dispatch 到 Store 產生一個 `saga`。在產生一個 task 後，它仍然接受傳入的 action 到底層的 `buffer`，最多保留一個（最近傳入的），但在同一時間內，保有 `ms` 毫秒產生新的 task（因此它的名稱為 - `throttle`）。 目的是為了對於在給定時間內正在處理的 task，而忽略傳入的 action。

- `ms: Number` - 時間窗口的長度（以毫秒為單位），在 action 開始處理之後，將忽略其他 action

- `pattern: String | Array | Function` - 更多資訊請參考 [`take(pattern)`](#takepattern) 文件

- `saga: Function` - 一個 Generator function

- `args: Array<any>` - 傳送給被啟動 task 的參數。`throttle` 將會新增傳入 action 到參數 list（意思是，action 將會是最後一個參數，提供給 `saga`）

#### 範例

在以下的範例，我們建立一個簡單的 `fetchAutocomplete` task。當 `FETCH_AUTOCOMPLETE` action 被 dispatch 時，我們使用 `throttle` 來啟動一個新的 `fetchAutocomplete` task。由於 `throttle` 忽略一段時間內連續的 `FETCH_AUTOCOMPLETE`，我們要確保使用者不會向使用者傳送大量的 request。

```javascript
import { call, put, throttle } from `redux-saga/effects`

function* fetchAutocomplete(action) {
  const autocompleteProposals = yield call(Api.fetchAutocomplete, action.text)
  yield put({type: 'FETCHED_AUTOCOMPLETE_PROPOSALS', proposals: autocompleteProposals})
}

function* throttleAutocomplete() {
  yield throttle(1000, 'FETCH_AUTOCOMPLETE', fetchAutocomplete)
}
```

#### 注意

`throttle` 是一個使用內建的 `take`、`fork` 和 `actionChannel` 的 high-level API。這裡如何使用 low-level Effect 實作 helper

```javascript
const throttle = (ms, pattern, task, ...args) => fork(function*() {
  const throttleChannel = yield actionChannel(pattern)

  while (true) {
    const action = yield take(throttleChannel)
    yield fork(task, ...args, action)
    yield call(delay, ms)
  }
})

```

## Effect creators

> 注意：

> - 以下每個 function 回傳一個純 JavaScript 物件而且不執行任何操作。
> - 執行是由 middleware 在上述迭代處理過程中進行的。
> - middleware 檢查每個 Effect 的描述並執行對應的 action。

### `take(pattern)`

建立一個 Effect 描述，指示 middleware 在 Store 等待指定的 action。
Generator 會暫停，直到一個符合 `pattern` 的 action 被 dispatch。

使用以下規則來解釋 `pattern`：

- 如果 `take` 參數為空或者為 `'*'`，所有被 dispatch 的 action 都符合（例如：`take()` 將符合所有 action）。

- 如果是一個 function，`pattern(action)` 為 true 時，action 才符合（例如：`take(action => action.entities)` 將符合所有 `entities` 欄位為 true 的 action）。
> 注意：如果 pattern function 有定義 `toString`，`action.type` 將會針對 `pattern.toString()` 測試。如果你使用 action creator library 像是 redux-act 或 redux-actions 非常有用。

- 如果是一個字串，當 `action.type === pattern` 為 true 時，action 才符合（例如：`take(INCREMENT_ASYNC)`）。

- 如果它是一個陣列，每個在陣列內的元素符合先前提到的規則，它支援 function predicates 和 string 的混合陣列。最常見的使用情況是字串陣列，所以 `action.type` 符合陣列內所有的元素（意思是，`take([INCREMENT, DECREMENT])` 會符合 type `INCREMENT` 或是 `DECREMENT` 的 action。）。

middleware 提供一個特別的 `END` action。如果你 dispatch END action，所有 Saga 被阻塞在 take Effect，不論指定的 pattern 是什麼都會被結束。如果被結束的 Saga 還有其他被 fork 的 task 會繼續執行，它在結束 Task 之前，等待所有子 task 結束。

### `take.maybe(pattern)`

與 `take(pattern)` 一樣，但是不會在 `END` action 自動結束 Saga。相反的，所有 Saga 在取得 `END` 物件時被阻塞再 take Effect。

#### 注意

`take.maybe` 的名稱來自 FP 的 analogy - 它不是回傳一個 `ACTION` type（與自動處理），我們可以有一個 `Maybe(ACTION)` type，所以我們可以處理兩種情況：

- 當它是一個 `Just(ACTION)` 情況（我們有一個 action）
- `NOTHING` 的情況（channel 被關閉*）。意思是，我們需要一些方法來 map `END`

* 當 `dispatch(END)` 發生時，內部所有通過 `stdChannel` 被 `dispatch` 的 action 會被關閉

### `take(channel)`

建立一個 Effect 描述，指示 middleware 從提供的 Channel 等待指定的訊息。如果 channel 已經關閉，Generator 會依照上面的 `take(pattern)` 處理描述立即結束。

### `take.maybe(channel)`

相同於 `take(channel)`，但是當接收到一個 `END` action 不會自動的終止 相反的，當 take 的 Effect 得到 `END` 物件，所有 Saga 都會被 block。[了解更多](#takemaybepattern)

### `put(action)`

建立一個 Effect 描述來說明 middleware 去 dispatch 一個 action 到 Store。
這個 effect 是非阻塞的，任何錯誤都會向下拋出（例如：在一個 reducer）不會冒泡回到 saga。

- `action: Object` - [完整資訊請參考 Redux `dispatch` 文件](http://redux.js.org/docs/api/Store.html#dispatch)

### `put.resolve(action)`

就像 [`put`](#putaction) 一樣，但是 effect 是阻塞的（如果 promise 從 `dispatch` 被回傳，它將等待 resolve）並會從 downstream 冒泡錯誤。

- `action: Object` - [完整資訊請參考 Redux `dispatch` 文件](http://redux.js.org/docs/api/Store.html#dispatch)

### `put(channel, action)`

建立一個 Effect 描述，指示 middleware put 一個 action 到提供的 channel。

- `channel: Channel` - 一個 [`Channel`](#channel) 物件
- `action: Object` - [完整資訊請參考 Redux `dispatch` 文件](http://redux.js.org/docs/api/Store.html#dispatch)

如果 put 的 effect *不是*被緩衝的，而是立即被 takers consume，那麼它是阻塞的。如果在任何 takers 拋出一個錯誤，它將會冒泡回到 saga。

### `call(fn, ...args)`

建立一個 Effect 描述，指示 middleware 呼叫 function `fn` 和 `args` 作為參數。

- `fn: Function` - 一個 Generator function，或者正常的 function 回傳一個 Promise 做為結果。
- `args: Array<any>` - 一個陣列值被作為參數傳送到 `fn`

#### 注意

`fn` 可以是一個*正常*或是一個 Generator function。

middleware 調用 function 並檢查它的結果。

如果結果是一個 Generator 物件，middleware 執行它就像啟動 Generator（在啟動時被傳送到 middleware）一樣。父 Generator 將暫停直到子 Generator 正常結束，在這個情況父 Generator 被恢復並透過子 Generator 回傳值。或者，子 Generator 因為一些錯誤而終止，在這個情況父 Generator 會從內部拋出錯誤。

如果結果是一個 Promise，middleware 將暫停 Generator 直到 Promise 被 resolve，在這個情況恢復 Generator 與 resolve 後的值，或者直到 Promise 被 reject，透過 Generator 從內部拋出錯誤。

如果結果不是一個 Iterator 物件或是 Promise，middleware 將會立即回傳 value 給 saga，
所以它可以同步的恢復執行。

如果目前的 `yield` 有一個 `try/catch` 區塊，當錯誤從 Generator 內部被拋出來時，將會被傳送到 `catch` 區塊。除此之外，Generator 隨著發出的錯誤終止，如果目前的 Generator 被其他 Generator 呼叫，錯誤將會傳播到呼叫的 Generator。

### `call([context, fn], ...args)`

與 `call(fn, ...args)` 一樣，但是支援傳送 `this` context 到 `fn`，這在調用物件方法很有用。

### `call([context, fnName], ...args)`

相同於 `call([context, fn], ...args)`，但是支援傳送一個 `fn` 作為 string。對於調用物件的方法很有用，也就是 `yield call([localStorage, 'getItem'], 'redux-saga')`

### `apply(context, fn, [args])`

`call([context, fn], ...args)` 的別名（Alias）。

### `cps(fn, ...args)`

建立一個 Effect 描述，指示 middleware 去調用 Node 風格的 `fn` function。

- `fn: Function` - 一個 Node 風格 function。例如：一個 function 除了本身的參數，在 `fn` 結束時調用 callback。callback 接受兩個參數，第一個參數被用來報告錯誤，第二個是被用來報告成功的結果。

- `args: Array<any>` - 一個陣列做為參數傳給 `fn`

#### 注意

middleware 將執行呼叫 `fn(...arg, cb)`。`cb` 透過 middleware 傳送給 `fn`。如果 `fn` 正常結束時，它必須呼叫 `cb(null, result)` 來通知 middleware 成功了，如果 `fn` 遇到一些錯誤，它必須呼叫 `cb(error)`，為了通知 middleware 有錯誤發生。

middleware 仍然會暫停，直到 `fn` 終止。

### `cps([context, fn], ...args)`

支援傳送一個 `this` context 到 `fn`（調用物件方法）。

### `fork(fn, ...args)`

建立一個 Effect 描述，指示 middleware 在 `fn` 執行一個*非阻塞呼叫*。

#### 參數

- `fn: Function` - 一個 Generator function，或者正常的 function 回傳一個 Promise 做為結果

- `args: Array<any>` - 一個陣列值被作為參數傳送到 `fn`

回傳一個 [Task](#task) object。

#### 注意

`fork` 類似於 `call`，可以被用來調用普通 function 和 Generator function。但是，呼叫是非阻塞的，middleware 不會在 `fn` 等待結果而暫停 Generator。相反的會調用 `fn`，立即恢復 Generator。

`fork` 和 `race` 是一個中心化的 Effect，管理 Saga 之間的併發。

`yield fork(fn ...args)` 的結果是一個 [Task](#task) 物件。一個物件有一些有用的方法和屬性。

所有被 fork 的 task 會被*附加*到它們的 parents。當 parent 終止它本身的執行時，它將等待所有被 fork 的 task 再回傳前終止。

從 child task 發生的錯誤將會自動得冒泡到它們的 parent。如果任何被 fork 的 task 引發了一個未捕獲的錯誤，paraent task 將終止 child 與錯誤，並且整個 Parent 執行 tree（意思是，被 fork 的 task 以及如果 parent 本身*主要 task*它仍然在執行）將會被取消。

取消 fork 的 Task 將會自動取消所有被 fork 且仍然在執行的 task。如果取消的 task 被阻塞的話，它也會取消目前的 Efffect。

如果一個被 fork task *同步*失敗（意思是，在執行任何非同步操作之前，執行後立即失敗），不會有 Task 被回傳，相反的，parent 將會立即的終止（由於 parent 和 child 兩者平行執行， parent 注意到 child 的失敗將會立即終止）。

如果要建立*分離*的 fork，使用 `spawn`。

### `fork([context, fn], ...args)`

支援被調用有 `this` context 的 fork function。

### `spawn(fn, ...args)`

與 `fork(fn, ...args)` 一樣，但是建立一個*分離*的 task。分離的 task 是獨立於父 task，而且行為像是一個高階的 task。父 task 回傳時將不會等待分離的 task 終止，而且所有事件可能影響父 task 或是完全分離獨立的 task（錯誤、取消）。

### `spawn([context, fn], ...args)`

支援被調用有 `this` context 的 spawn function。

### `join(task)`

建立一個 Effect 描述，指示 middleware 等待先前被 fork task 的結果。

- `task: Task` - 透過先前 `fork` 回傳一個 [Task](#task) 物件

#### 注意

`join`  將解析相同被加入 task 的結果（成功或失敗）。如果被加入的 task 被取消，取消也會傳播到 join effect 執行的 Saga。同樣的，任何那些 joiner 潛在的 caller 將被取消。

### `join(...tasks)`

建立一個 Effect 描述，指示 middleware 等待先前被 fork task 的結果。

- `tasks: Array<Task>` - 透過先前 `fork` 回傳一個 [Task](#task) 物件

#### 注意

它只是在 [join effects](#jointask) wrap task 的陣列，大致相等於 `yield tasks.map(t => join(t))`。

### `cancel(task)`

建立一個 Effect 描述，指示 middleware 取消先前被 fork 的 task。

- `task: Task` - 透過先前 `fork` 回傳一個 [Task](#task) 物件

#### 注意

為了取消執行的 task，middleware 將在底層的 Generator 物件調用 `return`，它將取消目前在 task 的 Effect 並跳到 finally 區塊（如果有定義）。

在內部的 finally 區塊，你可以執行任何清理邏輯或 dispatch 一些 action 來保持 store 的 state 一致（也就是說，當 ajax 請求被取消時，重置 spinner 的 state）。如果發出一個 `yield cancelled()` 取消 Saga，你可以確認內部的 finally 區塊。

取消往下傳播到子 saga。當取消一個 task，middleware 也取消目前的 Effect（task 目前被阻塞）。如果目前 Effect 呼叫其他 Saga，它也將被取消。當取消一個 Saga 時，所有*被附加的 fork*（sagas 使用 `yield fork()` fork）將被取消。這意思取消 task 將影響整個執行 的 tree。

`cancel` 是一個非阻塞的 Effect。例如：Saga 執行取消後，將立即恢復。

對於 function 回傳 Promise 結果，你可以透過附加 `[CANCEL]` 到 Promise 加入你的取消邏輯。

以下範例顯示如何附加取消邏輯到 Promise 結果：

```javascript
import { CANCEL } from 'redux-saga'
import { fork, cancel } from 'redux-saga/effects'

function myApi() {
  const promise = myXhr(...)

  promise[CANCEL] = () => myXhr.abort()
  return promise
}

function* mySaga() {

  const task = yield fork(myApi)

  // ... 之後
  // 在 myAPI 的結果將呼叫 promise[CANCEL]
  yield cancel(task)
}
```

redux-saga 使用 `abort` 方法將自動取消 jqXHR 物件。

### `cancel(...tasks)`

建立一個 Effect 描述來指示 middleware 取消先前被 fork 的 task。

- `tasks: Array<Task>` - 透過先前 `fork` 回傳一個 [Task](#task) 物件

#### 注意

它只是在 [cancel effects](#canceltask) wrap task 陣列，大致上相等於 `yield tasks.map(t => cancel(t))`。

### `cancel()`

建立一個 Effect 描述指示 middleware 去取消一個已經被 yield 的 task（取消本身）。
對於外部（`cancel(task)`）和本身（`cancel()`），它允許在 `finally` 區塊複用 destructor-like 邏輯。

#### 範例

```javascript
function* deleteRecord({ payload }) {
  try {
    const { confirm, deny } = yield call(prompt);
    if (confirm) {
      yield put(actions.deleteRecord.confirmed())
    }
    if (deny) {
      yield cancel()
    }
  } catch(e) {
    // 處理錯誤
  } finally {
    if (yield cancelled()) {
      // 被共用的取消邏輯
      yield put(actions.deleteRecord.cancel(payload))
    }
  }
}
```

### `select(selector, ...args)`

建立一個 Effect 描述，指示 middleware 在目前 Store 的 state 調用提供的 selector（例如：回傳 `selector(getState(), ...args)` 的結果）。
- `selector: Function` - 一個 `(state, ...args) => args` 的 function。它 take 目前 state 和可選的參數並回傳一個目前 Store 的 state 的部份。

- `args: Array<any>` - 可選的參數被傳送到另外的 `getState` selector。

如果 `select` 如果被呼叫時參數為空（例如：`yield select()`），effect 會 resolve 整個 state（與呼叫 `getState()` 相同）。

> 注意，當一個 action 被 dispatch 到 store，middleware 首先轉發 action 到 reducer 並通知 Saga，意思當你查詢 Store 的 State 時，你可以在 action 被處理**之後**取得 State。
> 然而，這個行為只保證如果所有後續的 middleware 同步呼叫 `next(action)`。如果任何後續 middleware 非同步呼叫 `next(action)`（這是不正常的，但是可能發生），Saga 將從**之前**處理的 action 取得 state。因此建議檢查每個後續的 middlewar 的來源，以確保它同步呼叫 `next(action)`，或者確保該 redux-saga 是呼叫 chain 中最後一個 middleware。

例如，假設我們有這個形狀的 state 在我們的應用程式：

```javascript
state = {
  cart: {...}
}
```

例如，一個 function 知道如何從 State 提取 `cart` 資料：

`./selectors`
```javascript
export const getCart = state => state.cart
```

然後我們從內部 Saga 可以使用 `select` Effect 的 selector：

`./sagas.js`
```javascript
import { take, fork, select } from 'redux-saga/effects'
import { getCart } from './selectors'

function* checkout() {
  // 使用被 export 的 selector 查詢 state
  const cart = yield select(getCart)

  // ... 呼叫一些 API 然後 dispatch 一個 success 或 error 的 action
}

export default function* rootSaga() {
  while (true) {
    yield take('CHECKOUT_REQUEST')
    yield fork(checkout)
  }
}
```

`checkout` 透過 `select(getCart)` 直接取得需要的資訊，Saga 只是加上 `getCart` selector。如果我們有許多 Saga（或 React Components）需要去存取部份的 `cart`，他們會耦合到相同的 `getCart` function。如果我們現在改變 state 形狀，我們只需要更新 `getCart`。

### `actionChannel(pattern, [buffer])`

建立一個 Effect 描述，指示 middleware 使用符合 `pattern` 的 action 事件 channel 隊列。或者，你可以提供一個緩衝去控制被隊列的 action 緩衝。

- `pattern:` - 參考 `take(pattern)` API
- `buffer: Buffer` - 一個 [Buffer](#buffer) 物件

#### 範例

以下程式碼建立 channel 去緩衝所有 `USER_REQUEST` action。注意，事件 Saga 或許被阻塞在 `call` effect。所有 action 在阻塞的同時自動被緩衝，這就是 Saga 為什麼可以一次執行一個 API 呼叫。

```javascript
import { actionChannel, call } from 'redux-saga/effects'
import api from '...'

function* takeOneAtMost() {
  const chan = yield actionChannel('USER_REQUEST')
  while (true) {
    const {payload} = yield take(chan)
    yield call(api.getUser, payload)
  }
}
```

### `flush(channel)`

建立一個 Effect 指示 middleware 從 channel 去 flush 所有被緩衝的項目。被 flush 的項目被回傳到 saga，如果需要可以使用它們。

- `channel: Channel` - 一個 [`Channel`](#channel) 物件。

#### 範例

```javascript

function* saga() {
  const chan = yield actionChannel('ACTION')

  try {
    while (true) {
      const action = yield take(chan)
      // ...
    }
  } finally {
    const actions = yield flush(chan)
    // ...
  }

}
```

### `cancelled()`

建立一個 Effect 描述，指示 middleware 去回傳這個 generator 是否已經被取消。通常你使用這個 Effect 在 finally 區塊執行取消特定的程式碼。

#### 範例

```javascript

function* saga() {
  try {
    // ...
  } finally {
    if (yield cancelled()) {
      // 邏輯應該只執行在取消
    }
    // 邏輯應該執行在所有情況（例如：關閉一個 channel）
  }
}
```

### `setContext(props)`

建立一個 effect 指示 middleware 更新它本身的 context。這個 effect 繼承 saga 的 context 而不是替換它。

### `getContext(prop)`

建立一個 effect 指示 middleware 回傳 saga context 特定的 property。

## Effect combinators

### `race(effects)`

建立一個 Effect 描述來指示 middleware 在多個 Effect 之間去執行一個 *Race*（這有點類似於 [`Promise.race([...])`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise/race) 行為）。

`effects: Object` - 物件形式 {label: effect, ...}

#### 範例

以下程式碼在兩個 effect 之間執行 race：

1. 呼叫 `feetchUsers` function 回傳一個 Promise
2. `CANCEL_FETCH` action 可能最終在 Store 被 dispatch

```javascript
import { take, call, race } from `redux-saga/effects`
import fetchUsers from './path/to/fetchUsers'

function* fetchUsersSaga {
  const { response, cancel } = yield race({
    response: call(fetchUsers),
    cancel: take(CANCEL_FETCH)
  })
}
```

如果 `call(fetchUsers)` 首先 resolve（或 reject），`race` 的結果將是一個 `{response: result}` 的物件，`result` 是 `fetchUsers` 被 resolve 的結果。

如果一個 `CANCEL_FETCH` 類型的 action 在 `fetchUsers` 完成前在 Store 被 dispatch，結果將會是一個 `{cancel: action}`，action 是被 dispatch 的 action。

#### 注意

當 resolve `race` 時，middleware 自動的取消所有輸掉的 Effect。

### `all([...effects]) - parallel effects`

建立一個 Effect 描述來指示 middleware 在平行情況下執行多個 Effect，並等待它們完成。它相當於 [`Promise#all`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) API 的標準。

#### 範例

以下範例同時執行兩個阻塞的呼叫：

```javascript
import { fetchCustomers, fetchProducts } from './path/to/api'
import { all, call } from `redux-saga/effects`

function* mySaga() {
  const [customers, products] = yield all([
    call(fetchCustomers),
    call(fetchProducts)
  ])
}
```

### `all(effects)`

相同於 [`all([...effects])`](#alleffects-parallel-effects)，但讓你可以在 directionary 傳送帶有 label effects 的物件，就像 [`race(effects)`](#alleffects)

- `effects: Object` - 一個 directionary {label: effect, ...} 形式的物件

#### 範例

以下的範例併行執行兩個 blocking 呼叫：

```javascript
import { fetchCustomers, fetchProducts } from './path/to/api'
import { all, call } from `redux-saga/effects`

function* mySaga() {
  const { customers, products } = yield all({
    customers: call(fetchCustomers),
    products: call(fetchProducts)
  })
}
```

#### 注意

當同時執行 Effect，middleware 暫停 Generator 直到以下其中一個發生：

- 所有 Effect 成功的被完成：恢復 Generator 與一個包含所有 Effect 的陣列結果。

- 其中一個 Effect 在所有 effect 完成之前被 reject：在 Generator 內拋出 reject 錯誤。

## Interfaces

### Task

Task interface 指定使用 `fork`、`middleware` 或 `runSaga` 執行 Saga 的結果。

<table id="task-descriptor">
  <tr>
    <th>方法</th>
    <th>回傳值</th>
  </tr>
  <tr>
    <td>task.isRunning()</td>
    <td>如果 task 還沒回傳或拋出一個錯誤為 true 時</td>
  </tr>
  <tr>
    <td>task.isCancelled()</td>
    <td>如果 task 已經被取消為 true 時</td>
  </tr>
  <tr>
    <td>task.result()</td>
    <td>task 回傳值。如果 task 仍然在執行 `undefined`</td>
  </tr>
  <tr>
    <td>task.error()</td>
    <td>task 拋出錯誤。如果 task 仍然在執行 `undefined`</td>
  </tr>
  <tr>
    <td>task.done</td>
    <td>
      Promise 結果可能是：
        <ul>
          <li>被 resovle task 的回傳值</li>
          <li>被 reject task 拋出錯誤</li>
        </ul>
      </td>
  </tr>
  <tr>
    <td>task.cancel()</td>
    <td>取消 task（如果 task 仍然在執行）</td>
  </tr>
</table>

### Channel

一個 channel 物件是被用在 task 之間傳送和接收訊息。來自 sender 的訊息被隊列直到一個 receiver 要求一個訊息，被註冊的 receiver 被隊列直到有一個可用訊息。

每個 channel 有一個底層的 buffer（緩衝），定義 buffer 的策略（修正 size、dropping、sliding）。

Channel interface 定義三個方法：`take`、`put` 和 `close`

`Channel.take(callback):` 被用來註冊一個 taker。take 是使用以下規則被 resolve：

- 如果 channel 有被緩衝的訊息，`callback` 將從底層 buffer 調用下一個訊息（使用 `buffer.take()`）  
- 如果 channel 被關閉，而且沒有任何的訊息，`callback` 調用 `END`
- 除此之外，`callback` 將隊列直到一個訊息被 put 到 channel

`Channel.put(message):` 在 buffer 上 put 訊息。put 使用以下規則處理：

- 如果 channel 被關閉，put 不會有 effect
- 如果是等待的 taker，調用舊的 taker 訊息
- 除此之外，在底層緩衝 put 訊息

`Channel.flush(callback):` 使用從 channel 提取所有被緩衝的訊息。使用以下規則來 resolve flush

- 如果 channel 被關閉，而且沒有任何被緩衝的訊息，`callback` 與 `END` 被調用
- 除此之外，`callback` 調用所有被緩衝的訊息

`Channel.close():` 關閉 channel 意思是不再允許更多的 put。所有 pending 的 taker 將會與 `END` 被調用。

### Buffer

在 channel 實現緩衝（Buffer）策略。Buffer interface 定義三個方法： `isEmpty`、`put` 和 `take`

- `isEmpty()`: 如果緩衝沒有任何訊息為 true 時，每當一個新的 taker 被註冊時，channel 呼叫這個方法
- `put(message)`: 放入新的訊息到 buffer。注意，Buffer 可以選擇不儲存訊息（例如：如果超過給定限制的訊息，刪除緩衝區可以丟棄訊息）
- `take()` 取得被緩衝的訊息。注意此方法的行為必須和 `isEmpty` 一致

### SagaMonitor

由 middleware 去 dispatch monitor 事件。實際上 middleware dispatch 五個事件：

- 當一個 effect 被觸發（透過 `yield someEffect`），middleware 調用 `sagaMonitor.effectTriggered`

- 如果 effect 成功被 resolve，middleware 調用 `sagaMonitor.effectResolved`

- 如果 effect 有一個錯誤被 reject，middleware 調用 `sagaMonitor.effectRejected`

- 如果 effect 被取消，middleware 調用 `sagaMonitor.effectCancelled`

- 最後，當一個 Redux action 被 dispatch，middleware 調用 `sagaMonitor.actionDispatched`

以下每個方法的署名：

- `effectTriggered(options)`：具有以下欄位的物件選項

  - `effectId` : Number - 獨立的 ID 被分配到被 yield 的 effect   

  - `parentEffectId` : Number - 內部所有被 yield 的 effect 將直接 `race` 或 `parallel` effect。在高階 effect 情況下，parent 將會包含 Saga

  - `label` : String - 在 `race` effect 的情況下，所有子 effect 將被分配作為 label 對應物件的 key 傳送到 `race`

  - `effect` : Object - 被 yield effect 的本身

- `effectResolved(effectId, result)`

    - `effectId` : Number - 被 yield effect 的 ID

    - `result` : any - effect 成功解決的結果。在 `fork` 或是 `spawn` effect 的情況，結果將會是一個 `Task` 物件。

- `effectRejected(effectId, error)`

    - `effectId` : Number - 被 yield effect 的 ID

    - `error` : any - 引發的錯誤與 effect 的 reject

- `effectCancelled(effectId)`

    - `effectId` : Number - 被 yield effect 的 ID

- `actionDispatched(action)`

    - `action` : Object - 被 dispatch 的 Redux action。如果透過一個 Saga dispatch action，action 將會有一個 `SAGA_ACTION` 屬性被設定為 true（可以從 `redux-saga/utils` import `SAGA_ACTION`）


## 外部 API
------------------------

### `runSaga(options, saga, ...args)`

允許在 Redux middleware 環境外啟動 saga。除了 store action 外，如果你想要連結 Saga 到外部的輸入或輸出這是非常有用的。

`runSaga` 回傳一個 Task 物件，就像從 `fork` effect 回傳一樣。

- `options: Object` - 目前支援的選項：

  - `subscribe(callback): Function` - 接受一個 callback 並回傳一個 `unsubscribe` function。

    - `callback(input): Function` - callback（透過 runSaga 提供）被用來訂閱輸入事件。`subscribe` 必須支援註冊多個訂閱。
      - `input: any` - 由 `subscribe` 傳送參數到 `callback`（參考下面注意事項）。

  - `dispatch(output): Function` - 用來滿足 `put` effect。
    - `output: any` -  由 Saga 提供的參數給 `put` Effect（參考下面注意事項）。

  - `getState(): Function` - 用來滿足 `select` 和 `getState` effect。

  - `sagaMonitor` : [SagaMonitor](#sagamonitor) - 參考 [`createSagaMiddleware(options)`](#createsagamiddlewareoptions) 文件。

  - `logger: Function` - 參考 [`createSagaMiddleware(options)`](#createsagamiddlewareoptions) 文件

  - `onError: Function` - 參考 [`createSagaMiddleware(options)`](#createsagamiddlewareoptions) 文件

- `saga: Function` - 一個 Generator function

- `args: Array<any>` - 提供給 `saga` 的參數

#### 注意

`{subscribe, dispatch}` 被用來完成 `take` 和 `put` Effects，定義 Saga 輸入和輸出的介面（interface）。

`subscribe` 被用來完成 `take(PATTERN)` effects，在每次有一個輸入被 dispatch 時，呼叫 `callback`（例如：如果 Saga 被連結到 DOM 點擊事件，在每次滑鼠點擊時）。
每次 `subscribe` 發出一個輸入到它的 callback，如果 Saga 被阻塞在 `take` effect 而且如果目前傳入的輸入符合 take pattern，Saga 恢復該輸入。

`dispatch` 被用來完成 `put` effects。每次 Saga 發出一個 `yield put(output)`，`dispatch` 被調用到該輸出。

## Utils

### `channel([buffer])`

一個 factory 方法被用來建立 Channel。你也可以選擇傳送 buffer 去控制 channel buffer 的訊息。

預設上，如果沒有提供緩衝，channel 最多將隊列 10 筆傳入的訊息，直到 taker 被註冊。預設緩衝採用 FIFO 策略來傳遞訊息：一個新的 taker 將傳遞在緩衝內最舊的訊息。

### `eventChannel(subscribe, [buffer], [matcher])`

建立 channel 使用 `subscribe` 方法來訂閱事件來源。從事件來源傳入的事件在 channel 將被隊列，直到有 taker 被註冊。

- `subscribe: Function` function 必須回傳一個取消訂閱（unsubscribe）的 function 來結束訂閱。

- `buffer: Buffer` 可選的 Buffer 物件在這個 channel 緩衝訊息。如果沒有提供訊息，在這個 channel 將不會被緩衝

- `matcher: Function` 可選的斷言（predicate） function（`any => Boolean`） 過濾傳入的訊息。只有被 matcher 接受的訊才會被放到 channel。

通知 channel 事件來源被終止，你可以用一個 `END` 通知被提供的訂閱者。

#### 範例

在以下範例我們建立一個事件 channel 將訂閱 `setInterval`：

```javascript
const countdown = (secs) => {
  return eventChannel(emitter => {
      const iv = setInterval(() => {
        console.log('countdown', secs)
        secs -= 1
        if (secs > 0) {
          emitter(secs)
        } else {
          emitter(END)
          clearInterval(iv)
          console.log('countdown terminated')
        }
      }, 1000);
      return () => {
        clearInterval(iv)
        console.log('countdown cancelled')
      }
    }
  )
}
```

### `buffers`

提供一些常見的 buffer

- `buffers.none()`: 如果沒有等待的 taker，新訊息將會遺失。

- `buffers.fixed(limit)`: 新的訊息將會被緩衝直到 `limit` 上限。Overflow 將會引發一個錯誤。省略 `limit` 將會導致只能緩衝 10 筆的訊息。

- `buffers.expanding(initialSize)`: 類似 `fixed`，但是 Overflow 將會造成緩衝動態的擴展。

- `buffers.dropping(limit)`: 相同於 `fixed`，但是 Overflow 將會直接的丟棄訊息。

- `buffers.sliding(limit)`: 相同於 `fixed`，但是 Overflow 將會在尾部新增新訊息，並丟棄在緩衝內最舊的訊息。

### `delay(ms, [val])`

回傳一個 Promise，將在 `ms` 毫秒後 resolve `val`。

### `cloneableGenerator(generatorFunc)`

帶了一個 generator function（function*）並回傳一個 generator function。
所有從這個 function 實例化的 generator 都可以可以被 clone。
僅用於測試。

#### 範例

當你想要測試不同的 saga branch 的時候很有用的，你不需要重複這些動作。

```javascript

function* oddOrEven() {
  // 一些東西在這裡完成了
  yield 1;
  yield 2;
  yield 3;

  const userInput = yield 'enter a number';
  if (userInput % 2 === 0) {
    yield 'even';
  } else {
    yield 'odd'
  }
}

test('my oddOrEven saga', assert => {
  const data = {};
  data.gen = cloneableGenerator(oddOrEven)();

  assert.equal(
    data.gen.next().value,
    1,
    'it should yield 1'
  );

  assert.equal(
    data.gen.next().value,
    2,
    'it should yield 2'
  );

  assert.equal(
    data.gen.next().value,
    3,
    'it should yield 3'
  );

  assert.equal(
    data.gen.next().value,
    'enter a number',
    'it should ask for a number'
  );

  assert.test('even number is given', a => {
    // 在給出一個 number 之前，我們製作一個 clone 的 generator；
    data.clone = data.gen.clone();

    a.equal(
      data.gen.next(2).value,
      'even',
      'it should yield "event"'
    );

    a.equal(
      data.gen.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.test('odd number is given', a => {

    a.equal(
      data.clone.next(1).value,
      'odd',
      'it should yield "odd"'
    );

    a.equal(
      data.clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.end();
});

```
### `createMockTask()`

回傳一個 mock task 的物件。
僅用於測試目的，
[詳細資訊請參考 Task Cancellation 文件。](/docs/advanced/TaskCancellation.md#testing-generators-with-fork-effect)
)

## Cheatsheets

### Blocking / Non-blocking

| Name                 | Blocking                                                    |
| -------------------- | ------------------------------------------------------------|
| takeEvery            | No                                                          |
| takeLatest           | No                                                          |
| throttle             | No                                                          |
| take                 | Yes                                                         |
| take(channel)        | Sometimes (see API reference)                               |
| take.maybe           | Yes                                                         |
| put                  | No                                                          |
| put.resolve          | Yes                                                         |
| put(channel, action) | No                                                          |
| call                 | Yes                                                         |
| apply                | Yes                                                         |
| cps                  | Yes                                                         |
| fork                 | No                                                          |
| spawn                | No                                                          |
| join                 | Yes                                                         |
| cancel               | Yes                                                         |
| select               | No                                                          |
| actionChannel        | No                                                          |
| flush                | Yes                                                         |
| cancelled            | Yes                                                         |
| race                 | Yes                                                         |
| all                  | Blocks if there is a blocking effect in the array or object |
