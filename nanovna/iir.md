---
layout: default
title: IIR study
auther: kamiki18
revision: Nov. 25, 2019.
---

## `scipy.signal.filter()` を使ってのBPF設計


NanoVNA の入力段はダイレクトコンバージョンで 5kHz の IF に落し、そこから 48ksps の ADC でデジタル値にしたあと 5kHz のサイン波/コサイン波と内積して振幅、位相を入手している。
現状の IF 段のフィルタは簡素なものなのだが、使用されている ADC TLV320AIC3204 は miniDSP と言って、さらに 5段の IIR フィルタまたは 25 段の FIR フィルタが内蔵できるようになっている。

その IIR フィルタの設計について。もちろん作るのは 5kHz の BPF になる。

ところでざっと設計したあとで知ったことだが、IF は 5kHz からけっこうバラつく:

![Frequency deviation (IF)](/nanovna/images/freqdev_if.png  "IFの5kHzからの残差")


これは NanoVNA ファームウエア 0.3.1 において、50kHz 〜 1.5GHz まで 1Hz ステップでスキャンした時の IF の 5kHz からのズレをあらわしたヒストグラムである。
＋40Hz 〜 −60Hz くらいは確実に通過させるべき状況であった。

### Elliptic 3rd-order BPF

まず scipy の iirfilter を使って IIR の BPF を手に入れる:

~~~ python

import numpy as np
from scipy import signal
import matplotlib.pyplot as plt

sos = signal.iirfilter(3, [4950.0/24000, 5050.0/24000], rs=70, rp=0.1, btype='band', analog=False, ftype='ellip', output='sos')

~~~

出力形式は miniDSP が扱う second-order sections (sos) にしておく。
ストップバンド減衰幅 (rs) が 70dB だと前後のフィルター込みでも 100dB にちょい足りない感じが残るかもしれないが、
通過帯域リプル (rp) の設定ともども、後述の理由でこのあたりを採用。

~~~ python

print(sos)
[[ 6.72179564e-05  0.00000000e+00 -6.72179564e-05  1.00000000e+00  -1.57669598e+00  9.87339217e-01]
 [ 1.00000000e+00 -1.47527260e+00  1.00000000e+00  1.00000000e+00  -1.57202018e+00  9.93652622e-01]
 [ 1.00000000e+00 -1.67673215e+00  1.00000000e+00  1.00000000e+00  -1.59129508e+00  9.93781560e-01]]

~~~

BIQUAD のフィルタが三つ入ったリストの形で BPF が得られた。
それぞれのフィルタの途中をピックアップしつつ周波数特性を表示してみると:

![filter()の生の結果](/nanovna/images/elliptic3u.png)

最終的なフィルタ(緑の線)はとても立派なものなのだが、初段、青のフィルタの段階で 40dB も叩き落としているのはいただけない。
係数を適当にスケーリングしてこうした:

![rescaled filter](/nanovna/images/elliptic3.png)

5050Hz あたりが 2段通過時点でゲインを持ってしまっているので 30Hz 上に強烈な信号があって飽和すると問題だが、
5kHz の信号を減衰させて増幅とかやって情報落ちするよりは良いだろう。
頂上付近の形状、群遅延の様子:

![頂上付近](/nanovna/images/elliptic3dx.png)

太い線が振幅特性(縦軸左)、細い線が群遅延特性(縦軸右)。
5000Hz 前後で群遅延が平坦になるように、地味に係数を調整してあったりする。
... のだが、通過帯域はともかく群遅延のほうに ± 60Hz もの平坦性はなかった。もうちょっと幅は広く取らないといけない。

### Bessel 3rd-order BPF

Bessel 型 BPF の場合。

~~~ python

sos2 = signal.iirfilter(3, [4850.0/24000, 5150.0/24000], btype='band', analog=False, ftype='bessel', output='sos')

~~~

で、青の線が上記 bessel BPF:

![Beesel BPF and adjusted version](/nanovna/images/adjusted.png)

Bessel 型は LPF の時は群遅延の最大平坦という特徴があるが、HPF や BPF にはそんなもんはない。
頂上は右に傾いでいる。多少補正して

![Adjusted version](/nanovna/images/adjusted2.png)

使う。... のだが、いずれにしても平坦領域不足である。リテイク。

### NanoVNA の 0.3.1 期のソースに含まれていたフィルタ 2 題

NanoVNA のファームウエア 0.3.1 期のソースには miniDSP 用の IIR フィルタが幾つか含まれていた。
なお、どこからも参照されなかったりコメントアウトされていたりでファームバイナリには含まれていない。念のためだが。
ついでにいえば 0.4 になってソース上からも削除されてしまっている。

さて、その

~~~ c

/* bb, aa = signal.bessel(2, (4500.0/24000, 5500.0/24000), 'bandpass') */
const uint8_t adc_filter_config[] = { ... }

~~~

などとなっている部分である。これを TLV320AIC3204 のリファレンスガイドの Biquad section の項 (p26) とデータのバイトオーダーを記したページ (同 p143)
見ながら sos に再現してみると、もちろん直前でコメントアウトされている signal.bessel() 等とは一致しない。
各ステージのゲイン調整と、ステージの入れ換えが行われている。
そのうちの二つ、ベッセル型 2 次とエリプティック型 5 次を各ステージにばらして鑑賞。

~~~ python

b2_sos = np.array( [
   [ 0.02038348, 0.04076695, 0.02038348, 1.00000000, -1.46197867, 0.88831782],
   [ 0.18854105, -0.37708211, 0.18854105, 1.00000000, -1.54453158, 0.89706874] ] )

e5_sos = np.array( [
   [ 0.09142804, -0.15752506, 0.09142804, 1.00000000, -1.54717827, 0.97695541],
   [ 0.09142804, -0.11448097, 0.09142804, 1.00000000, -1.56485701, 0.97178340],
   [ 0.09142804, -0.16294789, 0.09142804, 1.00000000, -1.59058928, 0.97798371],
   [ 0.09142804, -0.12762332, 0.09142804, 1.00000000, -1.61379313, 0.99186194],
   [ 0.09142804, 0.00000000, -0.09142804, 1.00000000, -1.54478621, 0.99124229] ] )

~~~

![Bessel BPF, 2nd-order](/nanovna/images/bessel2_031.png)
![Elliptic BPF, 5th-order](/nanovna/images/elliptic5_031.png)

ベッセル型 2 次のほうは 5200Hz あたりのピークゲインと 5000Hz のターゲットゲインの調整に気くばり感。
でもエリプティック型のほうは 3つ目と 4つ目のステージのゲイン係数調整まだ入ってないかも。

また、ピークが右(高域)にくるステージを早めにもってくるようにしているが、このあたりが良く分からん ...。
0dB を越えるピークができるだけ低くなるよう、右左右左と交互にピークがあるステージをもってくるものじゃないのかなぁ ...。
実際 iirfilter が返してくるデフォルトのフィルタはそういう順序になっているし。

### References {#ref}

* [scipy.signal.iirfilter](https://docs.scipy.org/doc/scipy-1.1.0/reference/generated/scipy.signal.iirfilter.html)
* TLV320AIC3204 Application Reference Guide (SLAA577 - November 2012)
