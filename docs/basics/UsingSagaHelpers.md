# 使用 Saga Helpers

`redux-saga` 提供了一些 helper function，當指定的 action 被 dispatch 到 Store 時會產生 tasks。

helper function 建立在頂部較低層的 API 上。在進階部份我們會看到這些 function 可以被實現。

第一個 function `takeEvery` 會是我們很熟悉的，它提供類似於 `redux-thunk` 的行為。

讓我們說明常見的 AJAX 範例。在每次按下 Fetch 的按鈕，我們 dispatch 一個 `FETCH_REQUESTED` 的 aciton。我們透過啟動這個 task，將從伺服器取得一些資料，來處理這個 action。

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

為了每次在 `FETCH_REQUESTED` 啟動上面的 task：

```javascript
import { takeEvery } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeEvery('FETCH_REQUESTED', fetchData)
}
```

在上面的範例中，`takeEvery` 允許多個 `fetchData` 實例被同時啟動。在特定的時間，我們啟動一個新的 `fetchData` task，儘管前面還有一個或多個 `fetchData` task 還沒結束。

如果我們只想要取得最新被觸發請求的 response（例如：總是顯示最新版本的資料），我們可以使用 `takeLatest` helper：

```javascript
import { takeLatest } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeLatest('FETCH_REQUESTED', fetchData)
}
```

與 `takeEvery` 不同，`takeLatest` 在任何時候只允許一個 `fetchData` task 執行，它將啟動最新的 task。如果先前提供的 task 還在執行，當其他的 `fetchData` task 被啟動，先前的 task 會自動的被取消。

如果你有多個 Saga 觀察不同的 action，你可以建立多個觀察者和 `fork`（我們之後將討論關於 `fork`，現在它是一個 Effect 允許我們在背景啟動多個 Saga）。

例如：

```javascript
import { takeEvery } from 'redux-saga'

// FETCH_USERS
function* fetchUsers(action) { ... }

// CREATE_USER
function* createUser(action) { ... }

// 在並行下使用它們
export default function* rootSaga() {
  yield takeEvery('FETCH_USERS', fetchUsers)
  yield takeEvery('CREATE_USER', createUser)
}
```
