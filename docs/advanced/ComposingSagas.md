# 組合 Saga

雖然使用 `yield*` 是提供組合 Saga 的慣用方式，但是這個方法有一些限制：

- 你可能想獨立測試巢狀化的 generator。這導致一些在測試程式碼的重複以及重複執行的開銷。我們不想要執行一個巢狀化的 generator，但是要確保發出呼叫時它的參數是正確的。

- 更重要的是，`yield*` 只允許順序組合的 task，所以你只能 `yield*` 到 generator 一次。

你可以簡單使用 `yield` 來同時啟動一個或多個子 task。當 yeild 一個 `call` 到一個 generator，Saga 將等待 generator 終止，然後以回傳值恢復執行（或是如果從子 task 傳播錯誤，會拋出例外）。

```javascript
function* fetchPosts() {
  yield put(actions.requestPosts())
  const products = yield call(fetchApi, '/products')
  yield put(actions.receivePosts(products))
}

function* watchFetch() {
  while (yield take(FETCH_POSTS)) {
    yield call(fetchPosts) // 等待 fetchPosts task 終止
  }
}
```

Yield 一個巢狀化的 generator 陣列，將同時啟動所有子 generator，等待它們完成然後回傳所有結果並恢復執行：

```javascript
function* mainSaga(getState) {
  const results = yield all([call(task1), call(task2), ...])
  yield put(showResults(results))
}
```

事實上，yield Saga 與 yield 其他 effect 沒有什麼不同（未來的 action、timeout 等等）。這個意思說你可以使用 effect combinators 來合併所有其他類型的 Saga。

例如，你可能想要使用者在限制的時間內結束遊戲：

```javascript
function* game(getState) {
  let finished
  while (!finished) {
    // 在 60 秒內完成
    const {score, timeout} = yield race({
      score: call(play, getState),
      timeout: call(delay, 60000)
    })

    if (!timeout) {
      finished = true
      yield put(showScore(score))
    }
  }
}
```
