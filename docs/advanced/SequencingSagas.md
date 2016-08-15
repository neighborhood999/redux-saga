# 使用 yield* 對 Saga 做排序

你可以使用內建的 `yield*` 操作符，透過保持順序的方式來組合多個 Saga，可以讓你用一個簡單的程式風格來排序你的 *marcro-tasks*。

```javascript
function* playLevelOne() { ... }

function* playLevelTwo() { ... }

function* playLevelThree() { ... }

function* game() {
  const score1 = yield* playLevelOne()
  yield put(showScore(score1))

  const score2 = yield* playLevelTwo()
  yield put(showScore(score2))

  const score3 = yield* playLevelThree()
  yield put(showScore(score3))
}
```

注意，使用 `yield*` 將導致 JavaScript runtime *擴散*到整個序列。由此產生的迭代器（來自 `game()`）將 yield 所有被巢狀化的迭代器裡的值。一個更強大的替代功能是使用更通用的 middleware 組合機制。
