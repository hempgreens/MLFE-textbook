# MLFEのSVC命令に新しい実装をしたい人へのインストラクション

## はじめに

こんにちは、今回はMLFEのSVC命令に新しい機能の実装を行う方法の解説を行いたいと思います。環境変数を取得する`getenv`、指定した時間プログラムを停止する`sleep`をsvc_mlfe.pyに実装していきながら、SVC命令に新機能の実装を行う手順について解説したいと思います。それでは行きましょう。

## 何をするか

SVC命令への追加、つまり`SVC time`や`SVC printf`のようなものの実装をどのようにおこなうか、ということです。

手順は以下のようになります。

1. `svc_mlfe.py`の`subroutines`マップに`"subroutine":subroutine,`というフォーマットで記述する。
2. `subroutine`という名前の関数を定義する。引数はレジスタ情報を格納したオブジェクト`reg`とメモリ情報を格納した`data`オブジェクト。返却値は無い。
3. ライブラリを参照するときは関数内で`import`をする。

レジスタオブジェクト`reg`の属性は以下の通りです。

```Python
class Register:
    def __init__(self, adr_start):
        self.number_reg = 14
        self.gr = [0 for _ in range(self.number_reg)]
        self.pc = adr_start
        self.sp = 65535
        self.zf = False
        self.sf = False
        self.of = False
    def setGR(self, name: str, value: int) -> None:
        """
        レジスタに値を書き込む。
        name: レジスタ名, value: 値（整数値）
        スタックポインタ、プログラムカウンタ、フラグレジスタに書き込むことは出来ない。
        """
    def getGR(self, name: str):
        """
        レジスタの値を取得する。
        name: レジスタ名
        返却値: 値（整数値）
        """
    def setFR(self, value: int):
        """
        フラグレジスタを書き換える。
        value: 値
        0 < value
            zf = False, sf = False, of = False
        value = 0
            zf = True,  sf = False, of = False
        value < 0
            zf = False, sf = True,  of = False
        out range of signed 32bit number
            zf = False, sf = False, of = True
        """
    def isREG(self, exp):
        """
        expに指定したレジスタが存在するかどうかを判定する。
        exp: レジスタ名
        返却値: レジスタが存在するか否か（真偽値）
        """
```

メモリデータオブジェクト`data`の属性は以下の通りです。

```Python
class Data():
    def __init__(self, data: list):
        self.mem = data
        self.stack = [-1, ]
    def isADDR(self, exp):
        """
        アドレス値として正しいかどうか判定
        exp: アドレス値（整数値）
        返却値: 有効なアドレスか否か（真偽値）
        """
    def get_by_addr(self, p, reg, x=0):
        """
        アドレスを指定してそこのデータを取り出す。
        p: アドレス（整数値）
        reg: レジスタオブジェクト
        x: 指標レジスタ
        
        アクセス先が読み取り不可能だったらエラーを起こして実行を終了する。
        """
```

それでは、各種機能の実装へ参りましょう。

## 環境変数の取得

環境変数の取得をしてみます。Pythonの`os`ライブラリには、`environ`というマップオブジェクトが実装されており、`os.environ[VALUE]`のようなフォーマットで環境変数を取得することが出来ますが、今回はC言語の`stdlib`に含まれる`getenv`のような使い方が出来る命令を実装したいと思います。

```
char *getenv(const char *name)
```

constなchar配列の先頭番地を指定することで、環境文字列の先頭ポインタを取得することが出来ます。Pythonの`os.environ`を使ってこのような動作を再現していきたいと思います。

始めに`const char *name`をどのように実現するかですが、これはシンプルにヌル文字で区切られた連続した領域を文字列とみなし、その先頭アドレスが`GR1`に入っているだろうと期待して読み込みます。

次に取得した`name`を`os.environ`に通して環境変数の取得を行い、その文字列をメモリに割り当てます。

最後に割り当てた文字列の先頭番地を`GR0`に格納します。もしその環境変数が存在しなかったときや何かエラーが発生したときには`GR0`に`NULL`として`0`を入れておきます。

こんな手順で実装していきます。

```Python
def getenv(reg, data):
    import os
    try:
        # GR1に先頭アドレスが入っているはず
        head = reg.getGR("GR1")
        name = ""
        for e in data.mem[head:]:       # 先頭番地から
            if(e[0] == "DATA"):
                if(e[1] == 0):          # 区切り文字まで
                    break
                else:
                    name += chr(e[1])   # 数値を文字に変換
            else:
                # DATA以外にぶつかった
                raise ValueError("Access Not Data")
        else:
            # 区切り文字が見つからなかった
            raise ValueError("Not Found NULL value")
        # メモリの確保はENDのアドレスを押し下げてその位置に挿入されます。
        # GR0に確保したアドレスの先頭番地（確保する前のメモリ配列の長さ-1）を格納
        reg.setGR("GR0", len(data.mem) - 1)
        for e in os.environ[name]:
            # 環境文字列をバラバラにしてメモリに割当
            data.mem.insert(-1, ["DATA", ord(e)])
        else:
            # 最後に区切り文字を挿入
            data.mem.insert(-1, ["DATA", 0])
    except:
        # エラー時にはGR0にヌルを格納
        reg.setGR("GR0", 0)
```

関数の実装は出来ました。svc_mlfe.pyに記述していきます。

```Python
def getenv(reg, data):
    import os
    try:
        head = reg.getGR("GR1")
        name = ""
        for e in data.mem[head:]:
            if(e[0] == "DATA"):
                if(e[1] == 0):
                    break
                else:
                    name += chr(e[1])
            else:
                raise ValueError("Access Not Data")
        else:
            raise ValueError("Not Found NULL value")
        reg.setGR("GR0", len(data.mem) - 1)
        for e in os.environ[name]:
            data.mem.insert(-1, ["DATA", ord(e)])
        else:
            data.mem.insert(-1, ["DATA", 0])
    except:
        reg.setGR("GR0", 0)

subroutines = {
    "time":time,
    "scanf":scanf,
    "printf":printf,
    "malloc":malloc,
    
    "getenv":getenv,
}
```

実行してみます。以下のようなプログラムを記述しました。

```
PGM     START
        LAD     GR1, NAME
        SVC     getenv
        CPA     GR0, =0
        JZE     ERROR
        LD      GR1, GR0
        LAD     GR2, 's'
        SVC     printf
        RET
ERROR   OUT     ='Not Found Env value\n', =20
        RET
NAME    DC      'LANG\0'
        END
```

```
> python mlfe.py getenv_test.fe
C
```
私の環境では`C`と出力されました。みなさんの環境で出力される文字とは違う可能性があります。

また、`NAME`をでたらめな文字列に変更すると`ERROR`が実行されます。

```
> python mlfe.py getenv_test.fe
Not Found Env value
```

以上で、環境変数の取得するSVC命令についての説明を終了します。おおまかなものはわかったでしょうか。次にスリープ処理機能の実装を行います。

## スリープ処理

それでは、指定時間プログラムを停止する機能の実装をしてみたいと思います。Pythonには`time`ライブラリに`sleep`という関数がありますが、それを使ってミリ秒単位でプログラムの停止が行うことが出来る機能を実装してみましょう。


`windows.h`の中には`Sleep`という関数があります。定義はこんな感じです。

```
void Sleep(unsigned long milisec);
```

このようなものを実装していくのですが、ただこれを実装していくのはつまらないので、途中で[Ctrl+C]を押されたなどで最後まで行かなかったとき残りの時間を返すような機能を作ってみましょう。

Pythonの`time`ライブラリには`time`関数というプログラム内の経過時間を取得することが出来る関数があります。これで経過時間を取得して、設定した停止時間から引き算すれば残り時間を算出できそうですね。

実装してみます。

```Python
def sleep(reg, data):
    import time
    # 始めの経過時間をstartに保持
    start = time.time()
    try:
        # GR1には停止時間（ミリ秒）が入っているはず
        v = reg.getGR("GR1")
        # Pythonのtime.sleep関数は秒単位なので変換
        sleep(v / 1000)
        # GR0に正常終了の0を入れる
        reg.setGR("GR0", 0)
    except:
        # 経過時間をミリ秒に直して整数値へ変換
        t = int((time.time() - start) * 1000)
        # 返却値としてGR0に格納
        reg.setGR("GR0", reg.getGR("GR1") - t)
```

できました。svc_mlfe.pyに記述します。

```Python
def sleep(reg, data):
    import time
    start = time.time()
    try:
        v = reg.getGR("GR1")
        sleep(v / 1000)
        reg.setGR("GR0", 0)
    except:
        t = int((time.time() - start) * 1000)
        reg.setGR("GR0", reg.getGR("GR1") - t)


subroutines = {
    "time":time,
    "scanf":scanf,
    "printf":printf,
    "malloc":malloc,
    
    "getenv":getenv,
    "sleep":sleep,
}
```

実行してみましょう。以下のようなプログラムを書きます。

```
PGM     START
        
        LAD     GR1, 5000       ; 5000(msec)
        LAD     GR2, 1          ; stdout_decimal
        
        SVC     sleep
        
        CPA     GR0, =0
        JNZ     ERROR
        
        WRITE   GR2, GR1
        OUT     ='ミリ秒経過しました。\n', =11
        RET
ERROR   OUT     ='残り', =2
        WRITE   GR2, GR0
        OUT     ='ミリ秒で止められました。\n', =13
        RET
        END
```

5秒間停止するプログラムです。とりあえず実行してみます。

```
> python mlfe.py sleep_test.fe
5000ミリ秒経過しました。
```

途中で[Ctrl+C]を入力してスリープを止めます。

```
> python mlfe.py sleep_test.fe
残り3492ミリ秒で止められました。
```

きちんと出来てますね。これにて解説を終了したいと思います。他にもコマンドライン引数の取得やメモリコピー、文字列の比較など実装出来たら楽しそうですね。ぜひチャレンジしてみてください。

## おわりに

お疲れさまでした。今回はSVC命令の拡張について解説しました。説明がちょっと簡潔すぎてわかりにくいかもしれませんが、色々試してみてください。ありがとうございました。

## 更新日

2021年11月26日
