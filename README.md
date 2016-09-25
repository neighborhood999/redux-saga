# redux-saga

[![Join the chat at https://gitter.im/yelouafi/redux-saga](https://badges.gitter.im/yelouafi/redux-saga.svg)](https://gitter.im/yelouafi/redux-saga?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![npm version](https://img.shields.io/npm/v/redux-saga.svg?style=flat-square)](https://www.npmjs.com/package/redux-saga)

`redux-saga` 是一個針對在 React/Redux 應用程式中，可以更容易建立 side effect 的 library（例如：非同步的事件像是資料的 fetch 和存取瀏覽器的快取）。

想法上，redux-saga 像是一個獨立的 thread 在你的應用程式，專門負責 side effect。`redux-saga` 是 redux 的 middleware，意思說從主要應用程式標準的 redux action 可以啟動、暫停和取消 thread，它可以存取整個 redux 應用程式的 state 和 dispatch redux 的 action。

使用 ES6 的 Generators 功能讓非同步的流程可以更容易閱讀、撰寫和測試，*如果你還不熟悉的話，[這裡有一些介紹的連結](https://yelouafi.github.io/redux-saga/docs/ExternalResources.html)*。透過這樣的方式，這些非同步的流程看起來就像標準 JavaScript 同步程式碼（像是 `async`/`await`，但是 generators 還有一些更棒而且我們需要的功能）。

你可能已經使用 `redux-thunk` 來處理你資料的 fetch。不同於 redux thunk，你不會再出現 callback hell 了，你可以簡單測試非同步的流程並保持你的 action 是 pure 的。

# 入門

## 安裝

```sh
$ npm install --save redux-saga
```

或者，你可以直接在 HTML 頁面 `<script>` 標籤使用提供的 UMD build，請參考[這個章節](#using-umd-build-in-the-browser)。

## 使用範例

假設我們有一個 UI，當按下按鈕時，從遠端伺服器取得一些使用者的資料（為了簡單表示，我們只是顯示觸發 action 的程式）。

```javascript
class UserComponent extends React.Component {
  ...
  onSomeButtonClicked() {
    const { userId, dispatch } = this.props
    dispatch({type: 'USER_FETCH_REQUESTED', payload: {userId}})
  }
  ...
}
```

Component dispatch 一個原生 action 物件到 Store。我們將建立一個 Saga 來觀察所有 `USER_FETCH_REQUESTED` action 並觸發呼叫一個 API 取得使用者資料。

#### `sagas.js`

```javascript
import { takeEvery, takeLatest } from 'redux-saga'
import { call, put } from 'redux-saga/effects'
import Api from '...'

// 工作的 Saga：當 action 是 USER_FETCH_REQUESTED 時被觸發
function* fetchUser(action) {
   try {
      const user = yield call(Api.fetchUser, action.payload.userId);
      yield put({type: "USER_FETCH_SUCCEEDED", user: user});
   } catch (e) {
      yield put({type: "USER_FETCH_FAILED", message: e.message});
   }
}

/*
  在每次 dispatch `USER_FETCH_REQUESTED` action 時，啟動 fetchUser。
  允許併發取得使用者。
*/
function* mySaga() {
  yield* takeEvery("USER_FETCH_REQUESTED", fetchUser);
}

/*
  另外你也可以使用 takeLatest。

  但不允許併發取得使用者。當一個 fetch 已經在 pending 時，如果取得 dispatch「USER_FETCH_REQUESTED」，
  正在等待的 fetch 會被取消，只執行最新的發出的 USER_FETCH_REQUESTED。
*/
function* mySaga() {
  yield* takeLatest("USER_FETCH_REQUESTED", fetchUser);
}

export default mySaga;
```

為了要執行 Saga，我們將使用 `redux-saga` middleware 來連結 Redux Store。

#### `main.js`

```javascript
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

import reducer from './reducers'
import mySaga from './sagas'

// 建立 saga middleware
const sagaMiddleware = createSagaMiddleware()
// 將 saga middleware mount 在 Store 上
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)

// 然後執行 saga
sagaMiddleware.run(mySaga)

// render 應用程式
```

# 文件

- [介紹](http://yelouafi.github.io/redux-saga/docs/introduction/index.html)
- [基本概念](http://yelouafi.github.io/redux-saga/docs/basics/index.html)
- [進階概念](http://yelouafi.github.io/redux-saga/docs/advanced/index.html)
- [Recipes](http://yelouafi.github.io/redux-saga/docs/recipes/index.html)
- [外部資源](http://yelouafi.github.io/redux-saga/docs/ExternalResources.html)
- [疑難排解](http://yelouafi.github.io/redux-saga/docs/Troubleshooting.html)
- [術語表](http://yelouafi.github.io/redux-saga/docs/Glossary.html)
- [API 參考](http://yelouafi.github.io/redux-saga/docs/api/index.html)

# Translation

- [Chinese](https://github.com/superRaytin/redux-saga-in-chinese)
- [Chinese Traditional](https://github.com/neighborhood999/redux-saga)

# 在瀏覽器使用 umd build 版本

在 `dist/` 資料夾也有一個 `redux-saga` 的 **umd** build 可以使用。當使用 umd build 的 `redux-saga`，`ReduxSaga` 作為在 window 的全域變數。

umd 版本在你不使用 Webpack 或 Browserify 相當的有用。你可以從 [unpkg](unpkg.com) 直接存取。

以下的 build 都是可用的：

- [https://unpkg.com/redux-saga/dist/redux-saga.js](https://unpkg.com/redux-saga/dist/redux-saga.js)  
- [https://unpkg.com/redux-saga/dist/redux-saga.min.js](https://unpkg.com/redux-saga/dist/redux-saga.min.js)

**重要！**如果你的目標瀏覽器不支援 *ES2015 generators*，你必須提供一個有效的 polyfill，像是 [`babel` 所提供的](https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.25/browser-polyfill.min.js)。polyfill 必須被 import 在 **redux-saga** 之前：

```javascript
import 'babel-polyfill'
// 接著
import sagaMiddleware from 'redux-saga'
```

# 從原始碼中來建立範例

```sh
$ git clone https://github.com/yelouafi/redux-saga.git
$ cd redux-saga
$ npm install
$ npm test
```

以下的範例都是從 Redux repos 所移植（到目前為止）過來的：

### Counter 範例

這裡有三個 counter 的範例。

#### counter-vanilla

這個範例使用原生的 JavaScript 和 UMD build。所有的原始碼都在 `index.html` 內。

如果要啟動範例，只要在你的瀏覽器打開 `index.html`。

> 重要：你的瀏覽器必須支援 Generators。最新版本的 Chrome 和 Firefox、Edge 已經支援。

#### counter

這個範例使用 `webpack` 和高階的 `takeEvery` API。

```sh
$ npm run counter

# generators 的測試 sample
$ npm run test-counter
```

#### cancellable-counter

這個範例使用底層的 API 來證明 task 被取消。

```sh
$ npm run cancellable-counter
```

### Shopping Cart 範例

```sh
$ npm run shop

# generators 的測試 sample
$ npm run test-shop
```

### async 範例

```sh
$ npm run async

# generators 的測試 sample
$ npm run test-async
```

### real-world 範例（使用 webpack 和 hot reloading）

```sh
$ npm run real-world

# 抱歉，還沒有測試。
```
