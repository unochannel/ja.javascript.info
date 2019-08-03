# 式をパースする

算術式は2つの数字とそれらの間の演算子で構成されます。:

- `1 + 2`
- `1.2 * 3.4`
- `-3 / -6`
- `-2 - 2`

演算子は `"+"`, `"-"`, `"*"` または `"/"` のいずれかです。

先頭や末尾、間に余分なスペースがあるかもしれません。

式を取り、3つのアイテムを持つ配列を返す `parse(expr)` を作成してください。

1. 最初の数値
2. 演算子
3. 2番目の数値

例:

```js
let [a, op, b] = parse("1.2 * 3.4");

alert(a); // 1.2
alert(op); // *
alert(b); // 3.4
```