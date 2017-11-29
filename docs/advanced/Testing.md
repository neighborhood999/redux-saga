# Saga 測試的範例

這裡有兩個主要測試 Saga 的方式：一步一步測試 saga generator function，或者是執行整個 saga 並斷言 side effects。

## 測試 Saga Generator Function

假設我們有以下的 action：

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
```

我們想要測試 saga：

```javascript
function* changeColorSaga() {
  const action = yield take(CHOOSE_COLOR);
  yield put(changeUI(action.payload.color));
}
```

Since Sagas always yield an Effect, and these effects have simple factory functions (e.g. put, take etc.) a test may
inspect the yielded effect and compare it to an expected effect. To get the first yielded value from a saga,
call its `next().value`:

```javascript
  const gen = changeColorSaga();

  assert.deepEqual(
    gen.next().value,
    take(CHOOSE_COLOR),
    'it should wait for a user to choose a color'
  );
```

A value must then be returned to assign to the `action` constant, which is used for the argument to the `put` effect:

```javascript
  const color = 'red';
  assert.deepEqual(
    gen.next(chooseColor(color)).value,
    put(changeUI(color)),
    'it should dispatch an action to change the ui'
  );
```

由於這裡沒有更多其他的 `yield`，下一次呼叫 `next()` 時，generator 將會完成：

```javascript
  assert.deepEqual(
    gen.next().done,
    true,
    'it should be done'
  );
```

### Branching Saga

有時候你的 saga 可能會有不同的結果。為了要測試不同的 branch 而不重複所有流程，你可以使用 **cloneableGenerator** utility function

這時候我們新增兩個 action，`CHOOSE_NUMBER` 和 `DO_STUFF`，還有 action 相關的 creator：

```javascript
const CHOOSE_NUMBER = 'CHOOSE_NUMBER';
const DO_STUFF = 'DO_STUFF';

const chooseNumber = (number) => ({
  type: CHOOSE_NUMBER,
  payload: {
    number,
  },
});

const doStuff = () => ({
  type: DO_STUFF,
});
```

Now the saga under test will put two `DO_STUFF` actions before waiting for a `CHOOSE_NUMBER` action and then putting
either `changeUI('red')` or `changeUI('blue')`, depending on whether the number is even or odd.

```javascript
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
```

測試如下：

```javascript
import { put, take } from 'redux-saga/effects';
import { cloneableGenerator } from 'redux-saga/utils';

test('doStuffThenChangeColor', assert => {
  const gen = cloneableGenerator(doStuffThenChangeColor)();
  gen.next(); // DO_STUFF
  gen.next(); // DO_STUFF
  gen.next(); // CHOOSE_NUMBER

  assert.test('user choose an even number', a => {
    // cloning the generator before sending data
    const clone = gen.clone();
    a.deepEqual(
      clone.next(chooseNumber(2)).value,
      put(changeUI('red')),
      'should change the color to red'
    );

    a.equal(
      clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.test('user choose an odd number', a => {
    const clone = gen.clone();
    a.deepEqual(
      clone.next(chooseNumber(3)).value,
      put(changeUI('blue')),
      'should change the color to blue'
    );

    a.equal(
      clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });
});
```

參考：[Task cancellation](TaskCancellation.md) 來測試 fork effects

## 測試完整的 Saga

Although it may be useful to test each step of a saga, in practise this makes for brittle tests. Instead, it may be
preferable to run the whole saga and assert that the expected effects have occurred.

Suppose we have a simple saga which calls an HTTP API:

```javascript
function* callApi(url) {
  const someValue = yield select(somethingFromState);
  try {
    const result = yield call(myApi, url, someValue);
    yield put(success(result.json()));
    return result.status;
  } catch (e) {
    yield put(error(e));
    return -1;
  }
}
```

We can run the saga with mocked values:

```javascript
const dispatched = [];

const saga = runSaga({
  dispatch: (action) => dispatched.push(action);
  getState: () => ({ value: 'test' });
}, callApi, 'http://url');
```

A test could then be written to assert the dispatched actions and mock calls:

```javascript
import sinon from 'sinon';
import * as api from './api';

test('callApi', async (assert) => {
  const dispatched = [];
  sinon.stub(api, 'myApi').callsFake(() => ({
    json: () => ({
      some: 'value'
    })
  }));
  const url = 'http://url';
  const result = await runSaga({
    dispatch: (action) => dispatched.push(action),
    getState: () => ({ state: 'test' }),
  }, callApi, url).done;

  assert.true(myApi.calledWith(url, somethingFromState({ state: 'test' })));
  assert.deepEqual(dispatched, [success({ some: 'value' })]);
});
```

參考 Repository 範例：

https://github.com/redux-saga/redux-saga/blob/master/examples/counter/test/sagas.js

https://github.com/redux-saga/redux-saga/blob/master/examples/shopping-cart/test/sagas.js
