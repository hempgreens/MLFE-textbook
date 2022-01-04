modとpowをマクロで実装してみよう

# MLFEのマクロ命令の拡張をしたい人へのインストラクション

## はじめに

こんにちは、この記事ではMLFEのマクロ命令を拡張したい人へマニュアルよりも詳細な手順を示すものです。マクロとして埋め込みたいものはやはり定数か処理だと思うので、`SWAP`マクロの実装と`IPアドレス`定数の実装をしていく過程で、様々な手順の解説ができればと思います。それでは行きましょう。

## 何をするか

マクロ命令の実装をするうえで必要な手順は、

1. macro_mlfe.pyのMacrosクラスに`MACRO="MACRO"`というフォーマットで記述する。
2. macro_mlfe.pyの`expand_macros`という関数の`macros`マッピング型の中で`"MACRO":MACRO,`というフォーマットで記述する。カンマを忘れずに。
3. macro_mlfe.py内に実装するマクロと同名の関数を実装する。

という手順です。この流れをまずは覚えておいてください。それでは実装に行きます。

テンプレートはこれを使います。使うときは関数名と下の`synerror`の引数を変えてくださいね。

```Python
# number: このマクロが何行目に配置されるか
# list_macro: マクロ宣言の入ったリスト
# isLabeled: ラベルがあるかどうか
def MACRO(number, list_macro, isLabeled):
    adj_label = lambda : 1 if isLabeled else 0
    if(False):
        expanded = []
        if isLabeled:
            expanded[0].insert(0, list_macro[0])
        label = get_labels(expanded)
        return expanded, label
    else:
        synerror("MACRO", list_macro, number)
```

## SWAPを実装する

SWAPマクロというものをご存じでしょうか。`SWAP A B`のように記述することで変数の値を交換する機能を持つ、ソートとかで用いられるものです。今回は変数同士でも変数とレジスタでもレジスタ同士でも交換できるマクロを実装します。

フォーマットは
```
SWAP    A, B
SWAP    r, X
SWAP    r1, r2

; 凡例
;   A, B, X アドレスを指すラベル
;   r, r1, r2 汎用レジスタ 
```

やり方はこうです。下の説明では左側の値を`A`、右側の値を`B`と呼称します。

1. GR0の値をスタックへPUSH（レジスタ退避）
2. Aの値をGR0にロード
3. GR0の値をスタックへPUSH（Aの値をスタックトップへ保持）
4. Bの値をGR0にロード
5. GR0の値をAへストア
6. スタックからGR0へPOP（Aの値をGR0に読み込ませる）
7. GR0の値をBへストア
8. スタックからGR0へPOP（レジスタ回復）

文字列で見てもわかりづらいと思うので、是非紙などに具体的な数値などを使って交換される様子を観察してくださいね。

フォーマットと大体のやり方が決まったらどのような制限を掛けるか考えましょう。正しくないフォーマットの時は同プログラムの`synerror`関数を使って例外をメインプログラムへ投げます。

渡される文字列は`list_macro`引数に入っています。それが正しいか検証するわけですね。まず考えられるダメな場合はそもそも長さが`3`では無いことです。ラベルも入れたら`4`ですね。`adj_label`関数はラベルがあったときに`1`を返す関数です。この実装が好みではないならこのあたりの実装は変えて大丈夫です。

```Python
adj_label = lambda : 1 if isLabeled else 0
if(len(list_macro) == 3 + adj_label()):
    
else:
    synerror("SWAP", list_macro, number)
```

フォーマットが三通りあるのでそれぞれ考えます。

- 両方レジスタの場合

両方レジスタの場合は、`list_macro`の1番目と2番目はどちらも`GR`から始まる汎用レジスタであるはずなので、それを検証します。あとGR0を指定されるのも困るのでそれも検証します。

```Python
adj_label = lambda : 1 if isLabeled else 0
isGR = lambda s: 3 <= len(s) and s[:2]=="GR" and s != "GR0"
if(len(list_macro) == 3 + adj_label()
    and (isGR(list_macro[1+adj_label()]) and isGR(list_macro[2+adj_label()]))
    ):
    
else:
    # 省略
```

まあこの方法だと`GR100`とか指定された時にバグが起こりますが、マクロ側が正確なレジスタの数を取得することは責任の範囲外と考えてこのままで大丈夫です。

- 片方レジスタもう片方ラベルの場合

次にレジスタとアドレスの場合ですが、レジスタの検証は上のと同様。アドレスラベルの検証はマクロ側では難しいので文字列がどうかだけ検証します。

```Python
adj_label = lambda : 1 if isLabeled else 0
isGR = lambda s: 3 <= len(s) and s[:2]=="GR" and s != "GR0"
if(len(list_macro) == 3 + adj_label()
    and (isGR(list_macro[1+adj_label()]) and isGR(list_macro[2+adj_label()]))
    or (isGR(list_macro[1+adj_label()]) and type(list_macro[2+adj_label()]) == str)
    ):
    
else:
    # 省略
```

- 両方ラベルの場合

両方ラベルなので、どちらも文字列かどうか判定します。

```Python
adj_label = lambda : 1 if isLabeled else 0
isGR = lambda s: 3 <= len(s) and s[:2]=="GR" and s != "GR0"
if(len(list_macro) == 3 + adj_label()
    and (isGR(list_macro[1+adj_label()]) and isGR(list_macro[2+adj_label()]))
    or (isGR(list_macro[1+adj_label()]) and type(list_macro[2+adj_label()]) == str)
    or (type(list_macro[1+adj_label()]) == str and type(list_macro[2+adj_label()]) == str)
    ):
    
else:
    # 省略
```

フォーマットが正しいかどうかはこんな所でしょうね。

次に実装部です。フォーマットが三通りありますが、実は変えなければいけないのは値をAやBへ格納する時です。相手がアドレスだったら`ST`命令を使って、相手がレジスタだったら`LD`命令を使います。手順の`5`と`7`ですね。

とりあえずトークンを取り出して、書けるところ書いちゃいましょう。

```Python
# 省略
if(省略):
    _a = list_macro[1+adj_label()]
    _b = list_macro[2+adj_label()]
    
    expanded = [
        ["PUSH", 0, "GR0"],
        ["LD", "GR0", _a],
        ["PUSH", 0, "GR0"],
        ["LD", "GR0", _b],
        [],
        ["POP", "GR0"],
        [],
        ["POP", "GR0"],
    ]
    
else:
    # 省略
```

条件分岐します。`_a`と`_b`を検証して、空の配列の所を埋めます。

```Python
# 省略
if(省略):
    _a = list_macro[1+adj_label()]
    _b = list_macro[2+adj_label()]
    
    expanded = [
        ["PUSH", 0, "GR0"],
        ["LD", "GR0", _a],
        ["PUSH", 0, "GR0"],
        ["LD", "GR0", _b],
        [],
        ["POP", "GR0"],
        [],
        ["POP", "GR0"],
    ]
    
    if(isGR(_a) and isGR(_b)):
        expanded[4] = ["LD", _a, "GR0"]
        expanded[6] = ["LD", _b, "GR0"]
    elif((isGR(_a))):
        expanded[4] = ["LD", _a, "GR0"]
        expanded[6] = ["ST", "GR0", _b]
    else:
        expanded[4] = ["ST", "GR0", _a]
        expanded[6] = ["ST", "GR0", _b]
    
else:
    # 省略
```

最後にマクロ内でラベルがあるかどうか確かめ、マクロ命令で貼られていたラベルを貼りなおしてメイン関数に展開後とラベルのリストを返して終わりです。この例ではありませんが、データを埋め込んだ処理の時など、`DC`命令部を実行しないように避けるために使ったりします。`ABS`マクロあたりを見てみてください。

```Python
# 省略
if(省略):
    _a = list_macro[1+adj_label()]
    _b = list_macro[2+adj_label()]
    
    expanded = [
        ["PUSH", 0, "GR0"],
        ["LD", "GR0", _a],
        ["PUSH", 0, "GR0"],
        ["LD", "GR0", _b],
        [],
        ["POP", "GR0"],
        [],
        ["POP", "GR0"],
    ]
    
    if(isGR(_a) and isGR(_b)):
        expanded[4] = ["LD", _a, "GR0"]
        expanded[6] = ["LD", _b, "GR0"]
    elif((isGR(_a))):
        expanded[4] = ["LD", _a, "GR0"]
        expanded[6] = ["ST", "GR0", _b]
    else:
        expanded[4] = ["ST", "GR0", _a]
        expanded[6] = ["ST", "GR0", _b]
    
    label = get_labels(expanded)
    if isLabeled:
        expanded[0].insert(0, list_macro[0])
    return expanded, label
else:
    # 省略
```

さて実行してみましょう。

以下のプログラムを保存し、実行してみます。

```
PGM     START
        
        ; アドレス同士の交換
        SWAP    A, B
        DMEM    ='DebugMEM', A, B
        OUT     ='\n', =1
        
        ; レジスタ同士の交換
        LAD     GR1, 100
        LAD     GR2, 200
        SWAP    GR1, GR2
        DREG    ='DebugREG'
        OUT     ='\n', =1
        
        ; レジスタとアドレスの交換
        SWAP    GR1, A
        OUT     A, =1, =1
        WRITE   GR0, GR1, =1
        OUT     ='\n', =1
        
        RET
A       DC      10
B       DC      20
        END
```

```
DebugMEM Start:115 End:116
  [00115   20  ]
  [00116   10  ]

DebugREG
GR    = [0, 200, 100, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
FLAG  = [SF:0, ZF:0, OF:0]
PC,SP = [41, 65535]

20020
```

最初の`DMEM`命令で反対になっているのがわかりますね。次の`DREG`でレジスタ同士の値が交換されています。最後の`GR1`と`A`の交換、`GR1`には200が入っていて、`A`には20が入っているはずなので、`20020`と出力されています。

上手くできましたね、以上でだいたい処理を埋め込む方法がわかったでしょうか、気になったら自分で何か機能を実装してみると面白いと思います。`MOD`とか`POW`とか実装出来たら楽しそうですね。

## IPアドレスマクロを実装する
次に定数宣言のマクロを実装してみたいと思います。定数系のマクロは標準では無かったので解説したいと思います。といっても先ほどよりも簡単だと思います。それでは`IPv4`のアドレスを32bit数値として埋め込む方法へ参りましょう。

皆さんはIPアドレスが32bitだということはご存じでしょうか、よく目にする形式だと`'142.251.42.164'`こんな感じですが（これはGoogleのIPです。）実は32bitなのです。というわけでこれを数値へ変換します。

やりかたは

1. コーテーション`'`を外します。
2. ドット`'.'`で文字列を`split`する。
3. 文字列それぞれを数値へ変換する。
4. 数値を2進数の8文字0埋めの文字列に変換する
5. 文字列のリストに`join`をかける
6. それを2進数として解釈する
7. 出た数値を32bitとして扱う

という感じです。実装します。


IPマクロのフォーマットはこんな感じです。
```
IP      '120.0.0.1'
```
例ではアドレス部分にはローカルホストのアドレスを入れていますが。IPアドレスの判定には正規表現を使いたいと思います。

以下のパターンを使います。[IBMのサイト](https://www.ibm.com/docs/ja/dsm?topic=SS42VS_DSM/c_LSX_guide_commonregex.html)を参考にしました。

```
\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}
```

```Python
def IP(number, list_macro, isLabeled):
    import re
    adj_label = lambda : 1 if isLabeled else 0
    if(re.search("\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}", list_macro[1+adj_label()]) is not None):
        
        expanded = []
        
        if isLabeled:
            expanded[0].insert(0, list_macro[0])
        label = get_labels(expanded)
        return expanded, label
    else:
        synerror("IP", list_macro, number)
```

展開する中身を作ります。まずコーテーションを外し`split`をしそれぞれを数値変換します。

```Python
_ip = list_macro[1+adj_label()]

splitted = _ip.replace("\'", "").split(".")
n = [int(e) for e in splitted]
```

2進数8文字0埋め表示をして、`join`します。

```Python
_ip = list_macro[1+adj_label()]

splitted = _ip.replace("\'", "").split(".")
n = [int(e) for e in splitted]

b = [f"{e:08b}" for e in n]
bs = "".join(b)
```

2進数として解釈し、32bitとして扱う、つまり`−2,147,483,648`から`2,147,483,647`の間で表現します。

```Python
_ip = list_macro[1+adj_label()]

splitted = _ip.replace("\'", "").split(".")
n = [int(e) for e in splitted]

b = [f"{e:08b}" for e in n]
bs = "".join(b)

i = int(bs, 2)
result = i if 0 <= i <= (2 ** 32 // 2 - 1) else i + 2 ** 32
```

これはちょっと冗長なので、もうちょっと短く書きますね。

```Python
_ip = list_macro[1+adj_label()]

i = int("".join([f"{e2:08b}" for e2 in [int(e1) for e1 in _ip.replace("\'", "").split(".")]]), 2)

result = i if 0 <= i <= (2 ** 32 // 2 - 1) else i + 2 ** 32
```

よし、OKです。上手く動くか試してくださいね。

```Python
_ip = list_macro[1+adj_label()]
i = int("".join([f"{e2:08b}" for e2 in [int(e1) for e1 in _ip.replace("\'", "").split(".")]]), 2)
result = i if 0 <= i <= (2 ** 32 // 2 - 1) else i + 2 ** 32

expanded = [
    ["DC", result],
]
```

```Python
def IP(number, list_macro, isLabeled):
    import re
    adj_label = lambda : 1 if isLabeled else 0
    if(re.search("\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}", list_macro[1+adj_label()]) is not None):
        
        _ip = list_macro[1+adj_label()]
        i = int("".join([f"{e2:08b}" for e2 in [int(e1) for e1 in _ip.replace("\'", "").split(".")]]), 2)
        result = i if 0 <= i <= (2 ** 32 // 2 - 1) else i + 2 ** 32

        expanded = [
            ["DC", result],
        ]
        
        if isLabeled:
            expanded[0].insert(0, list_macro[0])
        label = get_labels(expanded)
        return expanded, label
    else:
        synerror("IP", list_macro, number)
```

こんな感じですね。動かしてみましょう。

```
PGM     START
        RET
LOCAL   IP      '120.0.0.1'
        END
```

```
> python mlfe.py ip_test.fe -d
    0 START
    1 RET
    2 DATA    2013265921
    3 END
```

出来ていますね。これでIPアドレスを扱わないといけないときにも安心です。

このように処理の埋め込みだけではなく定数の埋め込みにも使うことが出来ます。よくつかう定数`ZERO`とか`ONE`とか、`TRUE`、`FALSE`とかIPv6とか保持出来たら使いやすそうですね。気になったら実装してみてください。

## おわりに

お疲れさまでした。今回はマニュアルだけではわかりにくいかもしれないマクロの拡張について解説してみました。何となく道筋が見えてきたなら幸いです。どんどんMLFEを使ってあげてください。
ありがとうございました。

## 更新日
2021/11/23

