# 一歩進んだ型ヒントの活用（LT版）

2026/02/21 Ryuji Tsutsui

PyCon mini Shizuoka 2026 LT

```{raw} html

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a>
<small>This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.</small>
```

## はじめに

### 自己紹介

* Ryuji Tsutsui@ryu22e
* さくらインターネット株式会社所属
* Python歴は14年くらい（主にDjango）
* 神奈川県横浜市在住
* 著書（共著）：
  * 『[Python実践レシピ](https://gihyo.jp/book/2022/978-4-297-12576-9)』（2022年）

### 【PR】3月16日『Python実践レシピ』第2版が出ます！

最新のPython 3.14に対応、書き下ろし（uv、Ruff、HTTPXなど）多数！

```{figure} _static/img/amazon.png
:alt: Amazonのページ

3月16日発売！（QRコードは次のスライド）
```

```{revealjs-break}
```

<https://amzn.asia/d/0jaaWbIX>

```{figure} _static/img/qrcode_www.amazon.co.jp.png
:alt: Amazon URLのQRコード

3月16日発売！（予約受付中）
```

## `assert_never()`関数の話

Python 3.11でtypingモジュールに追加された`assert_never()`関数を使うと、
if文やパターンマッチの条件指定漏れを型チェッカーで検出できて便利！という話をします（『Python実践レシピ』第2版の内容が元ネタです）。

### まず、こんな列挙型を定義する

```{revealjs-code-block} python

    """assert_never_example.py"""
    import enum

    class Color(enum.Enum):
        """色の種類"""
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
                return "不明"

    print(get_color_name_jp(Color.RED))  # 「赤」
    print(get_color_name_jp(Color.BLUE))  # 「青」
    print(get_color_name_jp(Color.YELLOW))  # 「黄」ではなく「不明」
```

### assert_never_example.pyの実行結果

「黄」が出力されない。

```{revealjs-code-block} bash

    % python assert_never_example.py
    赤
    青
    不明
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
* これを説明するため、`assert_never()`関数の定義を見てみましょう

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
* どんな値を入れてもエラーにならない`Any`型の逆バージョンみたいなイメージ

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

### 条件指定漏れチェックの流れ

`Never`型の特徴を利用して、条件指定漏れチェックを行っている。

```{revealjs-code-block} python

    # （省略）
    def get_color_name_jp(color: Color) -> str:
        match color:
            case Color.RED:
                return "赤"
            case Color.BLUE:
                return "青"
            case _:
                # (1)型チェッカーは引数がColor.YELLOWならここを通ることを
                # 検出
                # (2)引数に指定されているNever型は何を渡しても型エラーに
                # なるので、必ず「assert_never()が呼ばれる=型エラー」になる
                assert_never(color)
    # （省略）
```

## 最後に

### まとめ

* `assert_never()`関数は、条件指定漏れを検出してくれる便利関数
* この機能は、引数に指定された`Never`型の特徴を利用した型チェックにより実現している

### ご清聴ありがとうございました

<https://amzn.asia/d/0jaaWbIX>

```{figure} _static/img/qrcode_www.amazon.co.jp.png
:alt: Amazon URLのQRコード

『Python実践レシピ』第2版 3月16日発売、予約してね！
