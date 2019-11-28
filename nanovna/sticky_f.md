---
layout: default
title: Sticky ofreq.
auther: kamiki18
revision: Nov. 25, 2019.
---

## 受信フィルタ特性の表示


適当に固定した信号源を CH1 への入力として、その前後を sweep すれば受信フィルタの特性が得られる。

この特性には

  1.  SA612AD の入力側にある HPF(約1kHz),
  1.  SA612AD と TLV320AIC3204 の間にある LPF(約10kHz) と HPF(約160Hz),
  1.  TLV320AIC3204 の中にある Decimation Filter A (約 18kHz; リファレンスガイド p28), 
  1.  TLV320AIC3204 の miniDSP の BIQUAD もしくは FIR (リファレンスガイド p23), 
  1.  TLV320AIC3204 の miniDSP 後段の 1st IIR,
  1.  ファームウエアの dsp_process() による 5kHz信号と 1ms 区間で内積,

が含まれる。ただし 4 番目のフィルタはファームウエア 0.4 には今のところ含まれていない。
<!-- それと、TLV320AIC3204 の初段に DC カット用の HPF として 4Hz くらいのがあったような気がしたけど今データシート検索しても見当たらなかった。 -->
他の定数の計算もちょっと適当。とりあえず参考程度に。

このあたりスペクラムアナライザとして RBW を広げるのに問題になるのでその時にまた。
狭くするほうは miniDSP などより Si5351 の制限で決まりそうで、出来ることは少ない。
上の定数から見ると ADC に届くのは 1k〜10kHz というところだから、スペクラムアナライザとして RBW 上限は 9kHz くらいか。

さて、4 番目のフィルタを適当に設計し、miniDSP に送り込んだあと、受信フィルタの総合特性を見たいとする。
外部の信号源を用意するのも手間なので、NanoVNA 内部の Si5351 自身に肩代りしてもらうことにした。

つまり ofreq を固定したまま freq を sweep させる。逆でも可? 利害得失は良く分からない。


### 表示例

![Frequency deviation (IF)](/nanovna/images/prb_r1.png  "フィルタなし")
![Frequency deviation (IF)](/nanovna/images/prb_r2.png  "エリプティックフィルタ例")

一つ目は標準状態。二つ目が[このへん](iir.html)で作った Elliptic 3rd-order BPF を入れたもの。

 * デフォルトでも 18kHz の崖 (Decimation Filter A) は良く分かる。
 * 同じく、1kHz の HPF も見えている。イメージ混信のせいで左右対称になっている、その対称軸付近がペコンと凹んでいる。
 * 10kHz あたりにあるはずの LPF は、分かるような分からないような。
 * 実は将来も Elliptic BPF を常用するつもりがないが、こういうメリハリのついた特性でないと見ていて楽しくない。
 * 表示されたラインがとても noisy なのは ref 信号がないせいである。そもそも TLV320AIC3204 の IN2 (ref 信号) での、freq と ofreq をかけ算した信号を表示している。

ちなみに

![Frequency deviation (IF)](/nanovna/images/prb_r2_sc.png  "スミスチャートの表示")

[IF の微妙な偏差](fraction.html)で信号の遅延状態がころころ変わるため、フィルタを入れたまま使おうとすると smithchart やキャリブレーションが発狂する。
現状では miniDSP に IIR 書き込んでの動作確認以上の意味はない。

ファームウエア内で数 ms の平均処理をするのは FIR と同義で、群遅延は平坦なのでこういった暴れは起きない。
かといって miniDSP も FIR で使うのは 25-tap しかなくてファーム内の 1ms 分の処理 (96-tap FIR) にも劣り、まず論外。
どうしたものやら。

     