# 在 Parallel 中執行 Task

`yield` 語句可以很簡單的表達非同步的控制流程，但是我們也需要做一些同步的事情，我們不能只簡單撰寫：

```javascript
// 錯誤寫法，effect 將按照順序的被執行
const users  = yield call(fetch, '/users'),
      repos = yield call(fetch, '/repos')
```

因為第二個 effect 將直到第一個呼叫 resolve 後才執行，所以我們應該這樣撰寫：

```javascript
import { all, call } from 'redux-saga/effects'

// 正確寫法，在併行情況下 effect 執行
const [users, repos]  = yield all([
  call(fetch, '/users'),
  call(fetch, '/repos')
])
```

當我們 yield 一個 effect 的陣列，generator 是被阻塞的，直到所有 effect 都被 resolve 或者是被 reject（就像 `Promise.all` 的行為）。
