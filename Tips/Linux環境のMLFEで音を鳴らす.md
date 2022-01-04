# Linux環境のMLFEで音を鳴らす

## はじめに
こんにちは、今回はLinux環境のMLFEの20番台のポートを使って音を鳴らす方法を解説いたします。LinuxというかPosixレベルで互換のある音を鳴らす方法というものを思いつかなかったので標準環境には含めませんでした、なので外伝的に解説したいと思います。

## 環境

```
MLFEバージョン

$ python mlfe.py --version

Pythonバージョン

$ python --version
Python 3.7.3

```

## やり方
と言ってもやることはたいしたことではありません。基本的に決められたデータをもとに音を生成、その音を再生という手順でやっているのですが、音の生成はPythonがありさえすれば出来るように実装してあるので、問題は生成した音の再生です。何がしたいかと言えばつまり`wav`ファイルの再生がプログラムから出来れば良いのです。

修正箇所は`port_mlfe.py`のここです。

```Python
def playSound(ports, value):
    import wave
    import struct
    import math
    import os
    
    ports.ports[10] = value
    
    # 省略
        
        try:
            t = arrange(0, sample * sec)
            # 省略
            file.close()
            if(os.name == "nt"):
                import winsound
                winsound.PlaySound(name, winsound.SND_FILENAME)
            else:
                """
                ここです
                """
            os.remove(name)
        except:
            pass
```

### paplayを使う（Debian系OS向け）

`paplay`とは`Debian`系のLinuxに標準でインストールされている`PulseAudio`を土台にコマンドラインからサクッと音を鳴らせるコマンドです。これを`subprocess`から呼び出します。

```
else:
    import subprocess
    subprocess.run(["paplay", name])
os.remove(name)
```

### afplayを使う（Mac OS向け）

`afplay`とは`MacOS`にインストールされている音楽を再生できるコマンドです。上の`paplay`とほぼ同様に実行できると思います。

```
else:
    import subprocess
    subprocess.run(["afplay", name])
os.remove(name)
```

### 外部モジュールを使う

上の二つでほぼカバーできると思いますが、どうしてもいろんな環境で動かしたいときにはPythonのモジュールをインストールする必要があるでしょう。`wav`を再生できれば何でも良いのですが、今回は`playsound`というモジュールを紹介いたします。

まず`pip`からインストールします。
```
$ pip install playsound
```

そしてこの中でそれを使います。

```
else:
    from playsound import playsound
    playsound(name)
os.remove(name)
```

## おわりに
今回はPosix環境で実行されるMLFEプログラムについて、出力ポート20番台を使った音を鳴らす方法について解説いたしました。これからこのような標準環境に含めるのは憚られる細かいトピックをこのようにまとめていきたいと思います。よろしくお願いいたします。

## 参考

[paplay(1) - Linux man page](https://linux.die.net/man/1/paplay)

[afplay](https://ss64.com/osx/afplay.html)

[playsound 1.3.0](https://pypi.org/project/playsound/)

## 更新日
2021/11/22