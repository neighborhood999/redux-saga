## 在多個 Effect 之間啟動一個 race

有時候我同時啟動多個 task，但是我們不想要等待它們，我們只需要得到一個 *winner*：第一個被 resolve 的 task（或是 reject）。`race` Effect 提供一種在多個 Effect 之間的 race 觸發方式。

以下的範例顯示一個觸發遠端 fetch 請求的 task，並限制在一秒後 response。

```javascript
import { race, take, put } from 'redux-saga/effects'
import { delay } from 'redux-saga'

function* fetchPostsWithTimeout() {
  const {posts, timeout} = yield race({
    posts: call(fetchApi, '/posts'),
    timeout: call(delay, 1000)
  })

  if (posts)
    put({type: 'POSTS_RECEIVED', posts})
  else
    put({type: 'TIMEOUT_ERROR'})
}
```

`race` 另一個有用的功能是會自動取消失敗的 Effect。例如，假設我們有兩個 UI 按鈕：

- 第一個在背景啟動一個 task 執行一個無窮迴圈 `while (true)`（例如：在每幾秒從伺服器同步一些資料）。

- 一旦背景 task 被啟動了，我們啟用第二個按鈕來取消 task。


```javascript
import { race, take, put } from 'redux-saga/effects'

function* backgroundTask() {
  while (true) { ... }
}

function* watchStartBackgroundTask() {
  while (true) {
    yield take('START_BACKGROUND_TASK')
    yield race({
      task: call(backgroundTask),
      cancel: take('CANCEL_TASK')
    })
  }
}
```

在這個情況一個 `CANCEL_TASK` action 被 dispatch，透過內部拋出一個取消錯誤，`race` Effect 將自動取消 `backgroundTask`。
