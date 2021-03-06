# WeakMap と WeakSet 

チャプター <info:garbage-collection> で学んだ通り、JavaScriptエンジンは、それが到達可能な(そして潜在的に利用される可能性がある)間、メモリ上に値を保持しています。

例:
```js
let john = { name: "John" };

// オブジェクトへはアクセス可能です。john がその参照を持っています

// 参照を上書きします
john = null;

*!*
// オブジェクトはメモリから削除されるでしょう
*/!*
```

通常、オブジェクトまたは配列の要素、もしくは別のデータ構造のプロパティは到達可能と考えられ、そのデータ構造がメモリにいる間は保持され続けます。

例えば、あるオブジェクトを配列に入れた場合、その配列が生きている間は、他の参照がなくてもそのオブジェクトは生きていることになります。

例:

```js
let john = { name: "John" };

let array = [ john ];

john = null; // 参照を上書きします

*!*
// john は配列内に格納されているので、ガベージコレクションされません。
// array[0] としてそれを取得することが可能です
*/!*
```

また、通常の Map のキーとしてオブジェクトを使うと、Map が存在している間はそのオブジェクトも存在します。これはメモリを占め、ガベージコレクションされないかもしれません。

例:
```js
let john = { name: "John" };

let map = new Map();
map.set(john, "...");

john = null; // 参照を上書きします

*!*
// john は map の中に保持されています
// map.keys() で取得することができます
*/!*
```

`WeakMap` はこの点で根本的に異なります。これらはキーオブジェクトのガベージコレクションを妨げることはありません。

例でその意味するところを見ていきましょう。

## WeakMap

`Map` との最初の違いは、WeakMap のキーはプリミティブな値ではなくオブジェクトでなければならないことです:

```js run
let weakMap = new WeakMap();

let obj = {};

weakMap.set(obj, "ok"); // 正常に動作します (オブジェクトのキー)

*!*
weakMap.set("test", "Whoops"); // エラー, "test" はプリミティブだからです
*/!*
```

いま、オブジェクトをキーとして使用し、そのオブジェクトへの参照が他にない場合、自動的にメモリ(と map)から削除されます。

```js
let john = { name: "John" };

let weakMap = new WeakMap();
weakMap.set(john, "...");

john = null; // 参照を上書きします

// john はメモリから削除されます!
```

上の例を通常の `Map` の場合と比べて見てください。`WeakMap` のキーとしてのみ `john` が存在する場合、自動的に削除されます。

`WeakMap` は繰り返しと、メソッド `keys()`, `values()`, `entries()` をサポートしません。そのため、すべてのキーや値を取得する方法はありません。

`WeakMap` は次のメソッドのみを持っています:

- `weakMap.get(key)`
- `weakMap.set(key, value)`
- `weakMap.delete(key, value)`
- `weakMap.has(key)`

なぜこのような制限があるのでしょうか？これは技術的な理由です。もしオブジェクトがすべての他の参照を失った場合(上のコードの `john` のように)、自動的に削除されます。しかし、技術的には *いつクリーンアップが発生するか* は正確には指定されていません。

それはJavaScriptエンジンが決定します。エンジンはすぐにメモリのクリーンアップを実行するか、待ってより多くの削除が発生した後にクリーンアップするかを選択できます。従って、技術的には`WeakMap` の現在の要素数はわかりません。エンジンがクリーンアップしている/していない、または部分的にそれをしているかもしれません。このような理由から、`WeakMap` 全体にアクセスするメソッドはサポートされていません。

さて、どこでこのようなものが必要なのでしょう？

## ユースケース: additional data

The main area of application for `WeakMap` is an *additional data storage*.

If we're working with an object that "belongs" to another code, maybe even a third-party library, and would like to store some data associated with it, that should only exist while the object is alive - then `WeakMap` is exactly what's needed.

We put the data to a `WeakMap`, using the object as the key, and when the object is garbage collected, that data will automatically disappear as well.

```js
weakMap.put(john, "secret documents");
// もし john がなくなった場合、秘密のドキュメントは破壊されるでしょう
```

Let's look at an example.

For instance, we have code that keeps a visit count for users. The information is stored in a map: a user object is the key and the visit count is the value. When a user leaves (its object gets garbage collected), we don't want to store their visit count anymore.

Here's an example of a counting function with `Map`:

```js
// 📁 visitsCount.js
let visitsCountMap = new Map(); // map: user => visits count

// increase the visits count
function countUser(user) {
  let count = visitsCountMap.get(user) || 0;
  visitsCountMap.set(user, count + 1);
}
```

And here's another part of the code, maybe another file using it:

```js
// 📁 main.js
let john = { name: "John" };

countUser(john); // count his visits
countUser(john);

// later john leaves us
john = null;
```

Now `john` object should be garbage collected, but remains in memory, as it's a key in `visitsCountMap`.

We need to clean `visitsCountMap` when we remove users, otherwise it will grow in memory indefinitely. Such cleaning can become a tedious task in complex architectures.

We can avoid it by switching to `WeakMap` instead:

```js
// 📁 visitsCount.js
let visitsCountMap = new WeakMap(); // weakmap: user => visits count

// increase the visits count
function countUser(user) {
  let count = visitsCountMap.get(user) || 0;
  visitsCountMap.set(user, count + 1);
}
```

Now we don't have to clean `visitsCountMap`. After `john` object becomes unreachable by all means except as a key of `WeakMap`, it gets removed from memory, along with the information by that key from `WeakMap`.

## Use case: caching

Another common example is caching: when a function result should be remembered ("cached"), so that future calls on the same object reuse it.

We can use `Map` to store results, like this:

```js run
// 📁 cache.js
let cache = new Map();

// calculate and remember the result
function process(obj) {
  if (!cache.has(obj)) {
    let result = /* calculations of the result for */ obj;

    cache.set(obj, result);
  }

  return cache.get(obj);
}

*!*
// Now we use process() in another file:
*/!*

// 📁 main.js
let obj = {/* let's say we have an object */};

let result1 = process(obj); // calculated

// ...later, from another place of the code...
let result2 = process(obj); // remembered result taken from cache

// ...later, when the object is not needed any more:
obj = null;

alert(cache.size); // 1 (Ouch! The object is still in cache, taking memory!)
```

For multiple calls of `process(obj)` with the same object, it only calculates the result the first time, and then just takes it from `cache`. The downside is that we need to clean `cache` when the object is not needed any more.

If we replace `Map` with `WeakMap`, then this problem disappears: the cached result will be removed from memory automatically after the object gets garbage collected.

```js run
// 📁 cache.js
*!*
let cache = new WeakMap();
*/!*

// calculate and remember the result
function process(obj) {
  if (!cache.has(obj)) {
    let result = /* calculate the result for */ obj;

    cache.set(obj, result);
  }

  return cache.get(obj);
}

// 📁 main.js
let obj = {/* some object */};

let result1 = process(obj);
let result2 = process(obj);

// ...later, when the object is not needed any more:
obj = null;

// Can't get cache.size, as it's a WeakMap,
// but it's 0 or soon be 0
// When obj gets garbage collected, cached data will be removed as well
```

## WeakSet

`WeakSet` behaves similarly:

- It is analogous to `Set`, but we may only add objects to `WeakSet` (not primitives).
- An object exists in the set while it is reachable from somewhere else.
- Like `Set`, it supports `add`, `has` and `delete`, but not `size`, `keys()` and no iterations.

Being "weak", it also serves as an additional storage. But not for an arbitrary data, but rather for "yes/no" facts. A membership in `WeakSet` may mean something about the object.

For instance, we can add users to `WeakSet` to keep track of those who visited our site:

```js run
let visitedSet = new WeakSet();

let john = { name: "John" };
let pete = { name: "Pete" };
let mary = { name: "Mary" };

visitedSet.add(john); // John visited us
visitedSet.add(pete); // Then Pete
visitedSet.add(john); // John again

// visitedSet has 2 users now

// check if John visited?
alert(visitedSet.has(john)); // true

// check if Mary visited?
alert(visitedSet.has(mary)); // false

john = null;

// visitedSet will be cleaned automatically
```

The most notable limitation of `WeakMap` and `WeakSet` is the absence of iterations, and inability to get all current content. That may appear inconvenient, but does not prevent `WeakMap/WeakSet` from doing their main job -- be an "additional" storage of data for objects which are stored/managed at another place.

## Summary

`WeakMap` is `Map`-like collection that allows only objects as keys and removes them together with associated value once they become inaccessible by other means.

`WeakSet` is `Set`-like collection that stores only objects and removes them once they become inaccessible by other means.

Both of them do not support methods and properties that refer to all keys or their count. Only individual operations are allowed.

`WeakMap` and `WeakSet` are used as "secondary" data structures in addition to the "main" object storage. Once the object is removed from the main storage, if it is only found as the key of `WeakMap` or in a `WeakSet`, it will be cleaned up automatically.
