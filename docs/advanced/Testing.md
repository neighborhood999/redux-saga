# 測試 Sagas

**有效的回傳純 JavaScript 物件**

這些物件描述 effect 和 redux-saga 負責執行。

這讓測試非常容易，因為你只需要透過 saga 描述的 effect 比較被 yield 的物件就可以了。

## 基本範例

```javascript
console.log(put({ type: MY_CRAZY_ACTION }));

/*
{
  @@redux-saga/IO': true,
  PUT: {
    channel: null,
    action: {
      type: 'MY_CRAZY_ACTION'
    }
  }
}
 */
```

測試一個 saga，等待使用者的 action 和 dispatch

```javascript
const CHOOSE_COLOR = 'CHOOSE_COLOR';
const CHANGE_UI = 'CHANGE_UI';

const chooseColor = (color) => ({
  type: CHOOSE_COLOR,
  payload: {
    color,
  },
});

const changeUI = (color) => ({
  type: CHANGE_UI,
  payload: {
    color,
  },
});


function* changeColorSaga() {
  const action = yield take(CHOOSE_COLOR);
  yield put(changeUI(action.payload.color));
}

test('change color saga', assert => {
  const gen = changeColorSaga();

  assert.deepEqual(
    gen.next().value,
    take(CHOOSE_COLOR),
    'it should wait for a user to choose a color'
  );

  const color = 'red';
  assert.deepEqual(
    gen.next(chooseColor(color)).value,
    put(changeUI(color)),
    'it should dispatch an action to change the ui'
  );

  assert.deepEqual(
    gen.next().done,
    true,
    'it should be done'
  );

  assert.end();
});
```

另一個很大的好處是你也可以測試你的文件！它們描述所有應該發生的事。

## 分支 Saga

有時候你的 saga 會有不同的結果。為了測試不同的 branch 且不要重複這些步驟，你可以使用 **cloneableGenerator** utility
```javascript
const CHOOSE_NUMBER = 'CHOOSE_NUMBER';
const CHANGE_UI = 'CHANGE_UI';
const DO_STUFF = 'DO_STUFF';

const chooseNumber = (number) => ({
  type: CHOOSE_NUMBER,
  payload: {
    number,
  },
});

const changeUI = (color) => ({
  type: CHANGE_UI,
  payload: {
    color,
  },
});

const doStuff = () => ({
  type: DO_STUFF,
});


function* doStuffThenChangeColor() {
  yield put(doStuff());
  yield put(doStuff());
  const action = yield take(CHOOSE_NUMBER);
  if (action.payload.number % 2 === 0) {
    yield put(changeUI('red'));
  } else {
    yield put(changeUI('blue'));
  }
}

import { put, take } from 'redux-saga/effects';
import { cloneableGenerator } from 'redux-saga/utils';

test('doStuffThenChangeColor', assert => {
  const data = {};
  data.gen = cloneableGenerator(doStuffThenChangeColor)();

  assert.deepEqual(
    data.gen.next().value,
    put(doStuff()),
    'it should do stuff'
  );

  assert.deepEqual(
    data.gen.next().value,
    put(doStuff()),
    'it should do stuff'
  );

  assert.deepEqual(
    data.gen.next().value,
    take(CHOOSE_NUMBER),
    'should wait for the user to give a number'
  );

  assert.test('user choose an even number', a => {
    // 在傳送資料前 clone generator
    data.clone = data.gen.clone();
    a.deepEqual(
      data.gen.next(chooseNumber(2)).value,
      put(changeUI('red')),
      'should change the color to red'
    );

    a.equal(
      data.gen.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.test('user choose an odd number', a => {
    a.deepEqual(
      data.clone.next(chooseNumber(3)).value,
      put(changeUI('blue')),
      'should change the color to blue'
    );

    a.equal(
      data.clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });
});
```

也可以參考 [Task cancellation](TaskCancellation.md) 對於 effects 的測試：

也可以參考 Repository 的範例：

https://github.com/redux-saga/redux-saga/blob/master/examples/counter/test/sagas.js

https://github.com/redux-saga/redux-saga/blob/master/examples/shopping-cart/test/sagas.js
