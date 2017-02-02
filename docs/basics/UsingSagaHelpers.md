# 使用 Saga Helpers

當指定的 action 被 dispatch 到 Store 時，`redux-saga` 提供了一些 helper effects function wrap 內部的 fucnction 產生 task。

helper function 建立在低階的 API 上。我們在進階部份會看到這些 function 可以被實現。

第一個 function `takeEvery` 是我們很熟悉的，它提供類似於 `redux-thunk` 的行為。

讓我們說明一些常見的 AJAX 範例。在每次按下 Fetch 按鈕時，我們 dispatch 一個 `FETCH_REQUESTED` 的 aciton。透過啟動這個 task，將從伺服器取得一些資料處理這個 action。

首先我們建立 task 將執行非同步的 action：

```javascript
import { call, put } from 'redux-saga/effects'

export function* fetchData(action) {
   try {
      const data = yield call(Api.fetchUser, action.payload.url)
      yield put({type: "FETCH_SUCCEEDED", data})
   } catch (error) {
      yield put({type: "FETCH_FAILED", error})
   }
}
```

為了每次在得到 `FETCH_REQUESTED` action 時，啟動上面的 task：

```javascript
import { takeEvery } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeEvery('FETCH_REQUESTED', fetchData)
}
```

在以上的範例中，`takeEvery` 允許多個 `fetchData` 實例被同時啟動。在特定的時間，我們啟動一個新的 `fetchData` task，儘管前面還有一個或多個 `fetchData` task 還沒結束。

如果我們只想要取得最新被觸發請求的 response（例如：顯示最新版本的資料），我們可以使用 `takeLatest` helper：

```javascript
import { takeLatest } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeLatest('FETCH_REQUESTED', fetchData)
}
```

與 `takeEvery` 不同的是，`takeLatest` 在任何時候只允許一個 `fetchData` task 執行，它將啟動最新的 task。如果先前提供的 task 還在執行，當其他的 `fetchData` task 被啟動，先前的 task 會自動被取消。

如果你有多個 Saga 觀察不同的 action，你可以建立多個 watcher 和 `fork`（我們之後將討論關於 `fork`，現在它是一個 Effect 允許我們在背景啟動多個 Saga）。

例如：

```javascript
import { takeEvery } from 'redux-saga'

// FETCH_USERS
function* fetchUsers(action) { ... }

// CREATE_USER
function* createUser(action) { ... }

// 在 parallel 使用它們
export default function* rootSaga() {
  yield takeEvery('FETCH_USERS', fetchUsers)
  yield takeEvery('CREATE_USER', createUser)
}
```
