# 一個常見的抽象：Effect

概括來說，從一個 Saga 內部觸發 Side Effect 總是由 yield 一些宣告的 Effect 來完成的（你也可以直接 yield Promise，但是這會造成測試上的困難，如同我們在前面章節所看到的）。

Saga 實際上所做的事是將所有 Effect 組合在一起，來實現控制流程。最簡單的例子就是透過一個接一個的 yield 來序列化 yield Effect。你也可以使用熟悉的控制流程操作符（`if`、`where`、`for`）來實現更多的更複雜的控流程。

我們看到使用像是 `call` 和 `put` 的 Effect，與高階 API 像是 `takeEvery` 做組合，讓我們可以達到像是 `redux-thunk` 一樣的效果，但是額外的好處是我們可以更容易的測試。

但是 `redux-saga` 比 `redux-thunk` 提供了更多的優勢。在進階部分你將遇到一些更強大的 Effect，當你在表達複雜的控制流程同時，也可以保持相同可測試的好處。
