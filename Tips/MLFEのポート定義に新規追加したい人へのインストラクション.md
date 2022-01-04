# MLFEのポート定義に新規追加したい人へのインストラクション

## はじめに

こんにちは、今回はMLFEのポート定義に新規追加したい人へ、ターミナルサイズの取得を行うポート、実数表現を標準出力するポートを新規で実装することで、新しいポートを実装する方法を解説していきます。それではよろしくお願いいたします。

## 何をするか

ポート定義の新規追加、つまり`READ`や`WRITE`でアクセスするポートの追加をどのように行うか、というものです。

手順は以下のようになります。

入力ポートの定義の場合

1. `port_mlfe.py`の`map_in_port`マップの中に、`port number: port function,`というフォーマットで記述する。
2. `port_mlfe.py`内に関数の実装を行う。引数はポートのデータを保持する`ports`、返却値は32bitの整数値を返す。
3. ポートは入出力合わせて65536個存在し、先頭bitが1の時入力ポートであることが規定されているので`ports`を使うときには`ports.ports[ポート番号+32768]`を使う。
4. ライブラリをインポートするときは関数内で行う。

出力ポート定義の場合

1. `port_mlfe.py`の`map_out_port`マップの中に、`port number: port function`というフォーマットで記述する。
2. `port_mlfe.py`内に関数の実装を行う。引数はポートのデータを保持する`ports`、書き込まれた整数値が入っている`value`で、返却値は無い。
3. ライブラリをインポートするときは関数内で行う。

引数の`ports`はクラス`Ports`のインスタンスです。

```Python
class Ports:
    def __init__(self):
        self.ports = [0 for _ in range(2 ** 16)]
```

ポートのバッファを管理するオブジェクトです。データやオブジェクトの共有に用いたりすることができます。

## ターミナルサイズを取得できるポートを実装しよう

MLFEはポートを用いて標準出力に書き込んでいるのですが、CUIな環境で様々なものを表現するうえでどのような制限のもと出力されているのかを知るのは結構重要です。

というわけで、入力ポート20と21を使って、現在のターミナルの大きさを取得することが出来るポートを実装します。この機能を実装することで、例えば横幅にピッタリなプログレスバーを実装することが出来ます。

Pythonの`shutil`ライブラリの`get_terminal_size`関数を使うことで取得することが出来ます。

```Python
import shutil
shutil.get_terminal_size().columns # 縦の長さ
shutil.get_terminal_size().lines   # 横の幅
```

それでは実装していきましょう。まず`port_mlfe.py`の`map_in_port`マップに以下の記述をします。

```Python
map_in_port = {
    0: None,
    
    10: keyboardHit,
    11: getKeyboardCharctor,
    
    20: getTerminalColumnsSize,
    21: getTerminalLinesSize,
}
```

そして、`getKeyboardEventPosix`の下あたりに以下の関数を記述をします。

```Python
def getTerminalColumnsSize(ports) -> int:
    import shutil
    return shutil.get_terminal_size().columns

def getTerminalLinesSize(ports) -> int:
    import shutil
    return shutil.get_terminal_size().lines
```

あっさりしていますが、実装はこれにて完了です。テストしてみましょう。

```
PGM     START
        READ    GR0, GR1, P_COLMN
        READ    GR0, GR2, P_LINE
        DREG    ='DebugREG'
        RET
P_COLMN DC      20
P_LINE  DC      21
        END
```

```
> python mlfe.py getsize_test.fe
DebugREG
GR    = [0, 95, 36, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
FLAG  = [SF:0, ZF:0, OF:0]
PC,SP = [3, 65535]
```

ちゃんと`GR1`と`GR2`に現在のターミナルのサイズが格納されています。ターミナルのサイズを変えて確認してみてください。

試しに先ほど言ったプログレスバーの実装でもしてみますか。何パーセント進捗しているかを指定して、ターミナル上にプログレスバーを実装してみましょう。

1. 0から100までの値を`GR1`に格納する。
2. 入力ポート20から横幅を取得し、`GR2`に格納します。
3. `GR2`と`GR1`を掛け算して、`GR3`に格納します。
4. `GR3`を100で割ります。
5. `GR3`の数だけ文字を出力します。

こんな感じです。

```
PGM     START
        LD      GR1, PRGRS
        READ    GR0, GR2, P_COLMN
        MULA    GR3, GR1, GR2
        DIVA    GR3, =100
        
LOOP    CPA     GR3, =0
        JZE     LOOPEND
        OUT     ='#', =1
        SUBA    GR3, =1
        JUMP    LOOP
LOOPEND OUT     ='\n', =1
        RET

P_COLMN DC      20
PRGRS   DC      50
        END
```

プログラムが完成しました。実行してみると、大体ターミナルの半分くらいの横幅に`#`が表示されるはずです。ターミナルのサイズを変えてみたり、`PRGRS`の値を変えてみて試してくださいね。

大体入力ポートへの実装法はわかったでしょうか。次は実数表示を行うポートです。

## 分母と分子と表示桁数を指定して実数表示を行うポートを実装しよう

それでは、実数表示を行うポートの実装をします。出力ポート20, 21, 22, 23を使って実装します。

Pythonには`decimal`という十進数表示に関して精密に表現できるライブラリがあります。それを用いることで実装したいと思います。

さて、何故ポートを4つも占有するかですが、以下のようなポートアサインで実装しようと思っているからです。

```
out-20: 分数の分子を指定するポート
   -21: 分数の分母を指定するポート
   -22: 小数の表示桁数を指定するポート
   -23: 出力モードを指定するポート
```

こういう感じです。それでは実装していきます。

まず、`map_out_port`マップに以下のように記述します。

```Python
map_out_port = {
    0: printChar,
    1: printDecimal,
    2: printHex,
    3: printBin,
    4: printUnsignedDecinal,
    
    10: playSound,
    11: timePlayMiliSec,
    12: frequency,
    13: sampleFrequency,
    
    20: setNumerator,
    21: setDenominator,
    22: setQuantize,
    23: printRealNumber,
}
```

次に、関数を定義していきます。まずは出力ポート20,21,22から

```Python
def setNumerator(ports, value):
    ports.ports[20] = value
def setDenominator(ports, value):
    ports.ports[21] = value
def setQuantize(ports, value):
    # signedな値をunsignedに直しています。
    ports.ports[22] = value if 0 <= value else value + (2 ** 32)
```

`ports`を使って値を保持しておきます。次に出力ポート23の`printRealNumber`を実装します。

```Python
def printRealNumber(ports, value):
    import sys
    from decimal import Decimal
    if(value != 0):
        mode = value
        n = ports.ports[20]
        d = ports.ports[21]
        q = ports.ports[22]
        sys.stdout.write(
            str(
                Decimal(n / d)
                .quantize(Decimal("1." + "0"*q)))
            )
    
```

`Decimal`関数は引数を実数表現として保持します。`quantize`関数は、引数として指定した`Decimal`オブジェクトと同等の精度にします。最後に`str`に通して文字列として実数表現を取得してそれを`stdout.write`で出力します。

以上の関数を`sampleFrequency`関数の下あたりに記述します。

実行してみましょう。以下のようなソースを用意します。

```
PGM     START
        LD      GR1, NUMRTR
        LD      GR2, DENNTR
        LD      GR3, QUANTZ
        LD      GR4, MODE
        WRITE   GR0, GR1, P_NRTR
        WRITE   GR0, GR2, P_DNTR
        WRITE   GR0, GR3, P_QTZ
        WRITE   GR0, GR4, P_PRTRN
        RET

P_NRTR  DC      20
P_DNTR  DC      21
P_QTZ   DC      22
P_PRTRN DC      23

NUMRTR  DC      10
DENNTR  DC      3
QUANTZ  DC      5
MODE    DC      1
        END
```

```
> python mlfe.py print_realnumber.fe
3.33333
```

出力できています。しかし、`printRealNumber`の`value`が0かそうでないかぐらいしか活用されていないのはもったいない気がするので、以下のようなモードを定義します。

```text
0: デフォルト・何もしない
1: 実数の出力
2: 実数の出力, 設定した数値のクリア
3: 設定した数値のクリア
```

あと、分母の設定が0だと実行されないようにしましょう。

```Python
def printRealNumber(ports, value):
    import sys
    from decimal import Decimal
    if(value != 0):
        mode = value
        
        n = ports.ports[20]
        d = ports.ports[21]
        q = ports.ports[22]
        
        if(d == 0):
            return
        
        exp = str(Decimal(n / d).quantize(Decimal("1." + "0"*
            q)))
        
        if(mode == 1):
            sys.stdout.write(exp)
        elif(mode == 2):
            sys.stdout.write(exp)
            ports.ports[20] = 0
            ports.ports[21] = 0
            ports.ports[22] = 0
        elif(mode == 3):
            ports.ports[20] = 0
            ports.ports[21] = 0
            ports.ports[22] = 0
        else:
            sys.stdout.write(exp)
```

それでは実行してみましょう。以下のようなプログラムを実行してみます。

```
PGM     START
        LD      GR1, NUMRTR
        LD      GR2, DENNTR
        LD      GR3, QUANTZ
        
        LD      GR4, ONE            ; 出力モード
        
        WRITE   GR0, GR1, P_NRTR    ; 分子10
        WRITE   GR0, GR2, P_DNTR    ; 分母3
        WRITE   GR0, GR3, P_QTZ     ; 打ち切り桁5
        
        WRITE   GR0, GR4, P_PRTRN   ; 出力
        OUT     ='\n', =1
        
        WRITE   GR0, GR4, P_PRTRN   ; データが保持されているから出力される
        OUT     ='\n', =1
        
        LD      GR4, TWO            ; 出力+クリアモード
        
        WRITE   GR0, GR4, P_PRTRN   ; 出力し、データをクリア
        OUT     ='\n', =1
        
        WRITE   GR0, GR4, P_PRTRN   ; dが0なので出力されない
        OUT     ='\n', =1
        
        RET

P_NRTR  DC      20
P_DNTR  DC      21
P_QTZ   DC      22
P_PRTRN DC      23

NUMRTR  DC      10
DENNTR  DC      3
QUANTZ  DC      5

ONE     DC      1
TWO     DC      2
THREE   DC      3
        END
```

```
> python mlfe.py print_realnumber.fe
3.33333
3.33333
3.33333

```

上手く動いていますね。OKです。これにて解説を終了したいと思います。他にも何かセンサーの値を取得したり、マウスカーソルを動かしたりしたらおもしろそうですね。興味があったら実装してみてください。

## おわりに

お疲れさまでした。今回はポート定義の新規追加について解説しました。ちょっとイメージが湧きずらかったかもしれませんが、ポートに直接デバイスがくっついているようなものを想像していただけると良いと思います。SVC命令などに比べると自由度が低いですがこれはこれで上手くやると面白いのが出来ると思います。色々作ってみてください。ありがとうございました。

## 更新日
2021/11/27
