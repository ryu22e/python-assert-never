# 条件指定漏れをチェックできる便利関数`assert_never()`

```{raw} html

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a>
<small>This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.</small>
```

## はじめに

### 自己紹介

* Ryuji Tsutsui@ryu22e
* さくらインターネット株式会社所属
* Python歴は14年くらい（主にDjango）
* Python Boot Camp、Shonan.pyなどコミュニティ活動もしています
* 著書（共著）：
  * 『[Python実践レシピ](https://gihyo.jp/book/2022/978-4-297-12576-9)』（2022年）

### 【PR】『Python実践レシピ』第2版が出ます！

<https://amzn.asia/d/0jaaWbIX>

```{figure} _static/img/qrcode_www.amazon.co.jp.png
:alt: Amazon URLのQRコード

3月16日発売！
```

## `assert_never()`関数の話

Python 3.11で追加されたtypingモジュールの`assert_never()`関数を使うと、
if文やパターンマッチの条件指定漏れを型チェッカーで検出できて便利！という話をします。

### まず、こんな列挙型を定義する

```{revealjs-code-block} python

    """assert_never_example.py"""
    import enum

    class Color(enum.Enum):
        RED = 0
        BLUE = 1
        YELLOW = 2
```

### Colorをパターンマッチで判定するコードを書く

```{revealjs-code-block} python

    # （省略）
    def get_color_name_jp(color: Color) -> str:
        """引数の値に応じて日本語の色名を返す"""
        match color:
            case Color.RED:
                return "赤"
            case Color.BLUE:
                return "青"
            # Color.YELLOWを書き忘れている
            case _:
                return "Unknown"

    print(get_color_name_jp(Color.RED))  # 「赤」
    print(get_color_name_jp(Color.BLUE))  # 「青」
    print(get_color_name_jp(Color.YELLOW))  # 「黄」ではなく「Unknown」
```

### assert_never_example.pyの実行結果

「黄」が出力されない。

```{revealjs-code-block} bash

    % python assert_never_example.py
    赤
    青
    Unknown
```

### テストを書けば気付ける問題だけど……

* テストケースは人間が考えるもの
* うっかりテストケースを書き忘れることもある

### こんな時便利なのが`assert_never()`関数

```{revealjs-code-block} python

    from typing import assert_never  # (1)これを追加

    # （省略）
    def get_color_name_jp(color: Color) -> str:
        match color:
            case Color.RED:
                return "赤"
            case Color.BLUE:
                return "青"
            # Color.YELLOWを書き忘れている
            case _:
                assert_never(color)  # (2)ここを変更

    # （省略）
```

### 型チェッカーが条件指定漏れを検出してくれる

型チェッカーが「引数が`Color.YELLOW`の場合に`assert_never()`関数が呼ばれるよ」と教えてくれる。

```{revealjs-code-block} bash

    % mypy assert_never_example.py
    assert_never_example.py:17: error: Argument 1 to "assert_never" has incompatible type "Literal[Color.YELLOW]"; expected "Never"  [arg-type]
    Found 1 error in 1 file (checked 1 source file)
```

### このコードを実行するとどうなるか？

`assert_never()`関数が`AssertionError`を送出する。

```{revealjs-code-block} bash

    % python assert_never_example.py
    赤
    青
    Traceback (most recent call last):
    File "/***/assert_never_example.py", line 21, in <module>
        print(get_color_name_jp(Color.YELLOW))
            ~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^
    File "/***/assert_never_example.py", line 17, in get_color_name_jp
        assert_never(color)  # ここを変更
        ~~~~~~~~~~~~^^^^^^^
    File "/***/typing.py", line 2582, in assert_never
        raise AssertionError(f"Expected code to be unreachable, but got: {value}")
    AssertionError: Expected code to be unreachable, but got: <Color.YELLOW: 2>
```

### Q. そもそも、条件指定漏れの検出って「型チェック」なの？

* A. はい、型チェックです
* これを説明するため、`assert_never()`関数の仕組みについて見てみましょう

### `assert_never()`関数のソースコード

引数が`Never`型、というのがポイント。

```{revealjs-code-block} python

def assert_never(arg: Never, /) -> Never:
    value = repr(arg)
    if len(value) > _ASSERT_NEVER_REPR_MAX_LENGTH:
        value = value[:_ASSERT_NEVER_REPR_MAX_LENGTH] + '...'
    raise AssertionError(f"Expected code to be unreachable, but got: {value}")
```

### `assert_never()`関数と同じ機能の関数を自作する

`assert_never()`関数と同じく、引数が`Never`型の関数を定義すればよい。

```{revealjs-code-block} python

    from typing import Never

    class UnreachableError(Exception):
        pass

    # ↓引数の型をNeverにする
    def assert_unreachable(arg: Never, /) -> Never:
        raise UnreachableError(arg)
```

### `Never`型って何？

* `Never`型は「ボトム型」
* ボトム型とは、型理論や数理論理学において値を持たない型のこと
* ざっくり説明すると、「どんな値を入れても型チェックでエラーになる型」
* どんな値も入れられる`Any`型の逆バージョンみたいなイメージ

### 以下はすべて型チェックでエラーになる

```{revealjs-code-block} python

    """never_example.py"""
    from typing import Never
    a: Never = 1  # NG
    b: Never = "test"  # NG
    c: Never = True  # NG
    d: Never = None  # NG
```

### 前述のコードを型チェック

```{revealjs-code-block} bash

    % mypy never_example.py
    never_example.py:3: error: Incompatible types in assignment (expression has type "int", variable has type "Never")  [assignment]
    never_example.py:4: error: Incompatible types in assignment (expression has type "str", variable has type "Never")  [assignment]
    never_example.py:5: error: Incompatible types in assignment (expression has type "bool", variable has type "Never")  [assignment]
    never_example.py:6: error: Incompatible types in assignment (expression has type "None", variable has type "Never")  [assignment]
```

### つまり、こういうこと

```{revealjs-code-block} python
    :data-line-numbers: 8-12

    # （省略）
    def get_color_name_jp(color: Color) -> str:
        match color:
            case Color.RED:
                return "赤"
            case Color.BLUE:
                return "青"
            case _:
                # 型チェッカーはここを通るケースがあることを検出する
                # assert_never()関数の引数がNever型なので、
                # 何を渡しても型チェックエラーになる
                assert_never(color)
    # （省略）
```

## 最後に

### まとめ

* `assert_never()`関数は、条件指定漏れを検出してくれる便利関数
* 引数を`Never`型にした関数を定義すれば、`assert_never()`と同じ機能の関数を作れる
* この機能は、`Never`型の特徴を利用した型チェックにより実現している

### ご清聴ありがとうございました

<https://amzn.asia/d/0jaaWbIX>

```{figure} _static/img/qrcode_www.amazon.co.jp.png
:alt: Amazon URLのQRコード

『Python実戦レシピ』第2版 3月16日発売、予約してね！
