# Pull 未來的 action

到目前為止，我們使用了 `takeEvery` helpler function，為了在每次進來的 action 發起一個任務。這個有點模仿 redux-thunk 的行為：舉例來說，在每次 Component 調用一個 `fetchProducts` Action Creator，Action Creator 將 dispatch 一個 thunk 來執行控制流程。

實際上，`takeEvery` 只是在一個強大的底層 API 建立一個 helper function。在這個部份我們將看到一個新的 Effect：`take`，透過 action 完整控制的觀察過程，讓建立複雜的控制流程變成可能。

## 一個簡單的 logger

讓我們用一個簡單的 Saga 範例來觀察所有被 dispatch 到 action 的 store，並記錄它們到 console。

使用 `takeEvery('*')`（完全匹配的模式 `*`）不論它是什麼類型，我們都可以捕捉到所有被 dispatch 的 action。

```javascript
import { select, takeEvery } from 'redux-saga/effects'

function* watchAndLog() {
  yield takeEvery('*', function* logger(action) {
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  })
}
```

現在讓我們來看一下如何使用 `take` Effect 來實現上方相同的流程。

```javascript
import { select, take } from 'redux-saga/effects'

function* watchAndLog() {
  while (true) {
    const action = yield take('*')
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  }
}
```

`take` 就像 `take` 和 `put` 讓我們看起來很簡單。它建立其他控制的物件，告訴 middleware 等待指定的 action。與 `call` Effect 的情況相同，middleware 會暫停 Generator 直到 Promise resolve。在 `take` 的情況它將暫定 Generator 直到符合的 action 被 dispatch。以上的範例 `watchAndLog` 處於暫停的狀態，直到任何的 action 被 dispatch。

注意我們執行一個無窮的迴圈 `while (true)`。記住這是一個 Generator function，它沒有一個執行到完成（run-to-completion）的行為。我們的 Generator 在每次迭代時被阻塞，來等待一個 action 發生。

我們使用 `take` 撰寫程式碼有一個微妙的影響。在 `takeEvery` 的情況中，當他們被呼叫時，被調用的 task 無法控制，在每次符合的 action 發生時不斷的被調用，它們也無法控制什麼時候該停止觀察。

在 `take` 的情況中，控制的方式剛好相反。不是將 action *push* 到處理的 task，Saga 是透過本身去 *pull* action。看起來像是 Saga 執行一個正常的 function 呼叫 `action = getNextAction()`，當 resolve 時 action 會被 dispatch。

這樣的反轉控制讓我們可以用傳統的 *push* 方法來實現不同的控制流程。

這是一個簡單的範例，假設在我們的 Todo 應用程式，我們想要觀察使用者的 action，當使用者建立三個 todo 時，顯示一個祝賀訊息。

```javascript
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for (let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```

與 `while (true)` 不同，我們執行一個迭代三次的 `for` 迴圈。在執行迴圈後得到三個 `TODO_CREATED` 之後，`watchFirstThreeTodosCreation` 將因為應用程式要顯示祝賀訊息而終止。意思說 Generator 將垃圾回收並不會有其他的觀察發生。

pull 方法另一個好處是：我們可以使用熟悉的同步風格的程式碼，描述我們的控制流程。例如，假設我們想要實作一個登入流程有兩個 action：`LOGIN` 和 `LOGOUT`。使用 `takeEvery`（或 `redux-thunk`） 我將要撰寫兩個獨立的任務（或 thunk）：一個是 `LOGIN` 而另一個是 `LOGOUT`。

結果就是我們的邏輯被分成兩個地方了。為了讓其他人了解我們的程式碼做了哪些事，他必須閱讀這兩個處理登入流程的原始碼，並連結這兩個部份的邏輯。意思說他需要在心中重新排序各個地方程式碼的正確順序，並在腦海重新建立流程的模型。

使用 pull 模型可以在相同的地方撰寫我們的流程，而不是處理重覆相同的 action。

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

`loginFlow` Saga 更清楚地傳達預期 action 的順序。我們知道 `LOGIN` action 後面總是應該跟著一個 `LOGOUT` action，而且 `LOGOUT` 後面也總是跟著一個 `LOGIN` （一個好的 UI 的 action 應該總是執行一個固定的順序，透過隱藏或禁用非預期的 action）。
