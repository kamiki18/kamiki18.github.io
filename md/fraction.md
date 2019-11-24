---
layout: default
title: Rational approximation module test
---

## Fraction().limit_denominator でのごにょごにょ。

NanoVNA の出力やダイレクトコンバージョン用 LO のクロックは fractional PLL の si5351 で作成される。

原発 Fc から、
<dl>
  <dd> F<sub>PLL</sub> = Fc * ( m<sub>p</sub> + n<sub>p</sub> / d<sub>p</sub> ),  </dd>
  <dd>  Fout = F<sub>PLL</sub> / ( m<sub>m</sub> + n<sub>m</sub> / d_<sub>m</sub> ) / R  </dd>
</dl>
のような感じで出力が作成される。この分周の係数決めが si5351_set_frequency_with_offset() で行われている。
adafruits や arduino の Si5351 ライブラリの出来も似たりよったりなのだが、
周波数偏差の最悪値が数十ヘルツあることに甘えて平均的にも数十ヘルツの誤差があるコードになっているような。

### ワーストケースでの偏差

NanoVNA の場合、ワーストケースは 1.456GHz 付近の ofreq (CLK0; ダイレクトコンバージョンの LO 用の出力) で起きる。

26MHz の原発を使って 7倍高調波で 1.456GHz を作るためには Multisynth の分周を 4 に固定して
<dl>
  <dd> 26[MHz] * (32 + 0/1048575)/4 * 7       =  1456.000000 [MHz] </dd>
</dl>
という係数を使う。この周波数の隣は明らかに
<dl>
  <dd> 26[MHz] * (32 + 1/1048575)/4 * 7       =  1456.00004339222 [MHz] </dd>
</dl>
で、ここに 43.4 [Hz] のギャップがある。つまり最悪 21.7 [Hz] の偏差はやむをえない。

同様に F<sub>PLL</sub> を 832MHz に固定した状況において、ワーストケースは 520 MHz 付近の ofreq で生じる:
<dl>
   <dd> 832/(8 + 0/1048575) * 5 = 520.000000 [MHz] </dd>
   <dd> 832/(8 + 1/1048575) * 5 = 519.999938 [MHz] </dd>
</dl>
なので 62Hz のギャップがあり、つまり 31 [Hz] の偏差は避けがたい。
ただしこちらは PLL 動かしてよいのなら、出力周波数の純度を犠牲にしてではあるが、偏差を減らすことはできる。

ofreq が 1456.005 MHz 付近にある時、それより 5kHz 低い freq (CLK1; VNA 出力用) の状況は平均的:

![freq_freq1455-1455.png](/images/freq_freq1455-1455.png)
![freq_freq1455-1455.png](/images/freqn_freq1455-1455.png)

1455.9949MHz から 1455.9951MHz まで 1Hz きざみで freq をスキャンした時の偏差のヒストグラム。
青は現コード、緑が有理近似で追い込んだもの。コードはこんな風。

~~~ python

def calc_frequency_fixeddiv(f, pll_div, harmonics):
   coeff = Fraction( f / 26000000.0 * pll_div / harmonics).limit_denominator(1048575)
   return 26000000 * float(coeff) / pll_div * harmonics

~~~

float() を使わず有理数のままで追うべきだが、そのままでは桁数が足りず、計算を間違う。
ワーニングは出ているので文句言う筋合いではないが、ワーニングが出ずっぱりなので辟易したとも言う。

上の平均的な状況を見たあとだと、ofreq のワーストケースがワーストケースであると納得いく形に思える:

![freq_ofreq1455-1455.png](/images/freq_ofreq1455-1455.png)
![freq_ofreq1455-1455.png](/images/freqn_ofreq1455-1455.png)

### 係数の自由度

m + n/d という係数は微細な周波数決定に n * d の自由度がある。そして実際、有理近似によってこの自由度を使い切ることができる。

1.5 GHz までを 1 Hz 毎に表現するとすると 30bit 必要になるが、si5351 での d は 20bit しかなく、全域での表現力は足りない。
しかし逆に言えば  n が 10bit 以上あれば表現力に問題なく、1024/1048575 = 0.1% で、99.9% の周波数では誤差 1 Hz 以下に設定できる。

ランダムテストしかしてなかったので、いつまでたってもワーストケースを発見できないという罠に嵌まっていた。
全数検査に入って初めて気づいたり。


### IF のブレ

さて、freq, ofreq のブレは 数百メガヘルツの数十ヘルツで、原発 26MHz が 1ppm 前後の精度だと思えば、無視しようと思えば思えないこともない。
シグナルジェネレータやスペクトラムアナライザとして使う時はもちろん気になるが ... それは置いておいて、ofreq - freq = IF となる IF も数十ヘルツずれてくる:

![freq_if1455-1455.png](/images/freq_if1455-1455.png)
![freqn_if1455-1455.png](/images/freqn_if1455-1455.png)

どんだけ改善しても 20Hz も残るのはフィルタを書く上で大問題だったので MCLK (CLK2; TLV320AIC3204 の MCLK に繋がる出力)  を動かしてみた。つまり

~~~ python

def freq_pll2(f, pll_div, opll_div, harmonics):
    if harmonics == 3 :
        oharmonics = 5
    if harmonics == 5 :
        oharmonics = 7
    coeff = Fraction( f / 26000000.0 * pll_div / harmonics).limit_denominator(1048575)
    ocoeff = Fraction( (f + 5000.0) * opll_div / 26000000.0 / oharmonics).limit_denominator(1048575)
    freq = 26000000 / pll_div * float(coeff)  * harmonics
    ofreq = 26000000.0 / opll_div * float(ocoeff)  * oharmonics
    pll_clk = float(ocoeff) * 26000000.0;
    mcoeff = Fraction(pll_clk / ((ofreq - freq)/5000.0 * 8000000.0)).limit_denominator(1048575)
    mclk = pll_clk / float(mcoeff)
    return freq, ofreq, mclk

~~~

で、IF が ADC から 5kHz に見えるよう、8MHz であるはずの MCLK を  8000000 * (5000 / IF_clk) に設定する。すると ADC からは IF が:

![freqn_if1455-1455.png](/images/freqn3_if1455-1455.png)

のように見える。かわりに MCLK が 8MHz から大幅にずれる。5KHz にとっての 60Hz は 8MHz にとっての 96kHz だから。

![freqn_if1455-1455.png](/images/freqn3_mclk1455-1455.png)
![freqn_if1455-1455.png](/images/freqdev_mclk.png)

うむ、酷いな(笑)。ちなみに赤のグラフは 50k〜1.5GHzまでスキャンした時の現ファームの MCLK 偏位のヒストグラム。本来 0.3Hz しかずれてなかったことが分かる。


### Reference

scipy と違って Fraction は python にデフォルトで付属するパッケージ、っぽかった。あらためてインストールしたおぼえがない。

* [fractions](https://docs.python.org/ja/3.7/library/fractions.html)





