---
layout: default
title: Rational approximation module test
auther: kamiki18
revision: Nov. 27, 2019.
---

## Si5351の設定のための有理近似のテスト

NanoVNA の出力やダイレクトコンバージョン用 LO のクロックは fractional PLL の Si5351 で作成される。

原発 Fc から、
<dl>
  <dd> F<sub>PLL</sub> = Fc * ( m<sub>p</sub> + n<sub>p</sub> / d<sub>p</sub> ),      &emsp;&emsp;&emsp;&emsp; // PLL の逓倍段 </dd>
  <dd> Fout = F<sub>PLL</sub> / ( m<sub>m</sub> + n<sub>m</sub> / d<sub>m</sub> ) / R    &emsp; // Multisynth の分周段 </dd>
</dl>
のような感じで出力が作成される。この分周の係数決めが `si5351_set_frequency_with_offset()` で行われている。
ここで 1 &le; d &le; 1048575, 0 &le; n &le; 1048575 である。m や R の制約は略。

なかなかの自由度なのだが、いまいち adafruits や arduino の Si5351 ライブラリあたりでは使い切ってない。
0.3.1 の NanoVNA も同様。周波数残差の最悪値が数十ヘルツあるのは確かにやむを得ないのだが、平均的にもそれというのはちょっとどうかと。

### 係数の自由度

`m + n/d` という係数は微細な周波数決定に `n * d` の自由度がある。そして実際、有理近似によってこの自由度を使い切ることができる。

1.5GHz までを 1Hz 毎に表現するとすると 30bit 必要になるが、si5351 での d は 20bit しかなく、全域での表現力は足りない。
しかし逆に言えば  n が 10bit 以上あれば表現力に問題なく、1024/1048575 = 0.1% で、99.9% の周波数では誤差 1 Hz 以下に設定できる。

有理近似で残差を抑えるコードを書いたは良いがランダムテストしかしてなかったので、いつまでたってもワーストケースを発見できないという罠に嵌まっていた。
全数検査に入って初めて気づいたり。


### ワーストケースでの残差

NanoVNA の場合、ワーストケースは 1.456GHz 付近の ofreq (CLK0; ダイレクトコンバージョンの LO 用の出力) で起きる。

26MHz の原発を使って 7倍高調波で 1.456GHz を作るためには Multisynth の分周を 4 に固定 (以下、pll_div = 4) して
<dl>
  <dd> 26 [MHz] * (32 + 0/1048575)/4 * 7       =  1456.000000 [MHz] </dd>
</dl>
という係数を使う。この周波数の隣は明らかに
<dl>
  <dd> 26 [MHz] * (32 + 1/1048575)/4 * 7       =  1456.00004339222 [MHz] </dd>
</dl>
で、ここに 43.4Hz のギャップがある。つまり ofreq = 1456.000022 [MHz] を要求した時に 21.4Hz の誤差が生じるのはやむをえない。

同様に、F<sub>PLL</sub> を 832MHz に固定した状況においてはワーストケースは 520 MHz 付近の ofreq で生じる:
<dl>
   <dd> 832/(8 + 0/1048575) * 5 = 520.000000 [MHz] </dd>
   <dd> 832/(8 + 1/1048575) * 5 = 519.999938 [MHz] </dd>
</dl>
なので 62Hz のギャップがあり、つまり 31Hz の残差は避けがたい。
ただしこちらは PLL 動かしてよいのなら、出力周波数の純度を犠牲にしてではあるが、残差を減らすことはできる。

#### 普通の状況

ofreq が 1456.000MHz 付近にある時、それより 5kHz 低い freq (CLK1; VNA 出力用) は平均的な状況にある。つまり頑張れば残差を抑え込める:

![freq_freq1455-1455.png](/nanovna/images/freq_freq1455-1455.png)
![freq_freq1455-1455.png](/nanovna/images/freqn_freq1455-1455.png)

1455.9949MHz から 1455.9951MHz まで 1Hz きざみで freq をスキャンした時の残差のヒストグラム。
一つ目の青は現ファームのコード、二つ目の緑が有理近似で追い込んだもの。コードはこんな風。

~~~ python

def calc_frequency_fixeddiv(f, pll_div, harmonics):
   coeff = Fraction( f / 26000000.0 * pll_div / harmonics).limit_denominator(1048575)
   return 26000000 * float(coeff) / pll_div * harmonics

~~~

アルゴリズム実体は `Fraction().limit_denominator()` の中で表には出てきてないが、こんな感じ:

$b/a$ (簡単のため $b < a$ であるとする) を連分数展開して
<dl>
  <dd> $b/a = 1/(d_0 + 1/(d_1 + 1/(d_2 + 1/(d_3 + \cdots ))))$ </dd>
</dl>  
を得たとする。これを途中で打ち切ると有理近似数列  $b_0/a_0, b_1/a_1, b_2/a_2, b_3/a_3, \cdots $ になる。
つまり  $b_0/a_0 = 1/d_0, b_1/a_1 = 1/(d_0 + 1/d_1), b_2/a_2 = 1/(d_0 + 1/(d_1 + 1/d_2))), \cdots $ もちろん  $a_0 < a_1 < a_2 < \cdots $ である。また、
<dl>
  <dd>  $  b_{m+1}/a_{m+1}  =  (b_{m-1} + d_{m+1} b_m)/(a_{m-1} + d_{m+1} a_m)  $  </dd>
</dl>
が成り立つ。

分母が $M$ を越えない有理近似が欲しい時、$b_{m-1}/a_{m-1}, b_m/a_m, b_{m+1}/a_{m+1}$ の三つに焦点をあてよう。ここで $a_m < M < a_{m+1}$ である。
有理近似の性質により、$a_m$ より小さい分母で $ b_m/a_m$ に優る近似分数は存在しない。つまり $a_m$ よりも大きい分母の範囲を探せばよい。

有理近似の数列は右にいくほど真値 $b/a$ に近く、また真値の両側から交互に真値に近付いていく。
したがって以下のどちらかが成り立つ。

 1.  $ b_{m-1}/a_{m-1} <  b_{m+1}/a_{m+1} < b/a < b_m/a_m,  $
 2.  $ b_m/a_m < b/a < b_{m+1}/a_{m+1} < b_{m-1}/a_{m-1}.  $

1\. の場合として、$b_{m-1}/a_{m-1}$ と $b_m/a_m$ の間の分数を $(b_{m-1} + k b_m)/(a_{m-1} + k a_m)$ と表現し、
$k$ が $a_{m-1} + k a_m < M$ を満たす整数として、

<dl>
  <dd>  $ a_{m-1} + k a_m < M < a_{m+1}, $  </dd>
  <dd>  $ b_{m-1}/a_{m-1} < (b_{m-1} + k b_m)/(a_{m-1} + k a_m)  < b_{m+1}/a_{m+1}  < b/a <  b_m/a_m. $  </dd> 
</dl>

 $b_{m+1}/a_{m+1}$ と $b_m/a_m$  の間に要求をみたす近似分数はない。可能性があるのは $(b_{m-1} + k b_m)/(a_{m-1} + k a_m)$ であり、
この分数と $b_m/a_m$ のうち、優秀なほうを答えとする。2. の場合も不等号をひっくりかえして以下同様。


元の python の関数に話を戻す。
有理近似の性能を使い切るためには `float()` を使わず有理数のままで追うべき:

~~~ python

   coeff = Fraction( f * pll_div, 26000000 * harmonics).limit_denominator(1048575)

~~~

だが、そのままでは桁数が足りず、ところどころで計算を間違う。
単純にみても (2^32-1) / 6 = 715827882.5 だから pll_div=6 の領域では 716MHz 以上で `Fraction()` の分子が 32bit 非負整数の範囲を越えてしまう。
有理数 `A/B` と `C/D` を比較する時、正確に行うなら `A*D` と `B*C` の比較に直す。`A, B, C, D` が 32bit 越えなら比較は 64bit 越えになる。
ワーニングは出ているので文句言う筋合いではないが、ワーニングが出ずっぱりなので辟易したというか気にしてなかったというか。
いや、こういうライブラリは常識で考えて、無限多倍長計算するものだろう。64bit 程度で綻しないでほしい。


### ワーストケースと対策

上の平均的な状況を見たあとだと、ofreq のワーストケースがワーストケースであると納得いく形に思える:

![freq_ofreq1455-1455.png](/nanovna/images/freq_ofreq1455-1455.png)
![freq_ofreq1455-1455.png](/nanovna/images/freqn_ofreq1455-1455.png)

平均的には激的に改善するが、残差の最大の改善幅は、まあたいしたことない。

余談だが、freq, ofreq とも現ファームは上のグラフでは右にずれるケースがないようだが、
全周波数でヒストグラム作ってみれば数は少ないながら高い周波数になることもある。

#### IF もブレる。

さて、freq, ofreq のブレは 数百メガヘルツの数十ヘルツで、原発 26MHz が 1ppm 前後の精度だと思えば、無視しようと思えば思えないこともない。
シグナルジェネレータやスペクトラムアナライザとして使う時はもちろん気になるが ... それは置いておいて、ofreq - freq = IF となる IF も数十ヘルツずれてくる。
こちらは 5kHz の数十ヘルツなのでとても気になる:

![freq_if1455-1455.png](/nanovna/images/freq_if1455-1455.png)
![freqn_if1455-1455.png](/nanovna/images/freqn_if1455-1455.png)

改善後は freq の残差がほぼゼロになっているのだから、ofreq と同じ形になるのは当然。
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

![freqn_if1455-1455.png](/nanovna/images/freqn3_if1455-1455.png)

のように見える。かわりに MCLK が 8MHz から大幅にずれる。5kHz にとっての 60Hz は 8MHz にとっての 96kHz だから。

![freqn_if1455-1455.png](/nanovna/images/freqn3_mclk1455-1455.png)
![freqn_if1455-1455.png](/nanovna/images/freqdev_mclk.png)

うむ、酷いな(笑)。ちなみに二つ目の赤のグラフは 50k〜1.5GHzまでスキャンした時の現ファームの MCLK 残差のヒストグラム。本来 0.3Hz しかずれてなかったことが分かる。

#### 代替策: freq を動かす

ofreq が飛ぶ領域でも freq のほうは精密に設定できるので、IF がなるべく厳密に 5kHz になるよう ofreq に合わせて freq をずらして設定するという方針もあり得る。
ただこれは別の問題を含む。これは freq を 1455.99498 〜 1455.99502MHz の間をスキャンするかわり、freq を 1455.995MHz に固定してサンプルするのと同義である。
つまりスキャンしていない。

freq = 1040MHz 付近では上の ofreq と同様の理由によりどうしても 30Hz ステップになるが、1456MHz で諦めるのは早いだろう。
キャリブレーションの時に、中割りの関係で 1455.99498MHz でキャリブレーションするはめになった時 1455.995MHz で代替するのはありかもしれないが。


### 所感

正直 MCLK は動かしたくない。

 - MCLK を補助出力として外に出したい思惑、
 - MCLK を入力クロックとして使っている TLV320AIC3204 の PLL のロック待ちという時間ロス、
 - 常識的には高純度が要請される ADC のクロック品質の問題、

などなど。二つ目なぞ、ロックを知る方法がない上に、データシートとしてもクロックの初期化トータルに 10ms かかるみたいな記述しかない。
この手の codec は入力クロックは固定で使うものだからかなー。10ms毎に MCLK が微妙に変動するというのは想定していないだろう。
そもそも NanoVNA 自体も、Si5351 で作った MCLK を STM32F072 の MCLK ピンに入れているところからみて MCLK を動かすことは想定していなかったはず。

ついでだが、STM32F072 が Si5351 で作った MCLK で動いておらず、内蔵の HSI クロックで動いているのはファームソースを読んで初めて知った。
あちこちに出回っている NanoVNA のブロック図は誤解を招きすぎるだろう。


### Reference

scipy と違って Fraction は python にデフォルトで付属するパッケージ、っぽかった。あらためてインストールしたおぼえがない。

* [fractions](https://docs.python.org/ja/3.7/library/fractions.html)
* Si5351A/B/C-B datasheet (Rev. 1.1)
* Manualy Generating in Si5351 Register MAP (AN619)











