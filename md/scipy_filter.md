## scipy.signal.filter を使う時の留意点。

NanoVNA 0.3.1 以前の tlv320aic3204.c には TLV320AIC3204 の miniDSP 用の BPF の実験コードが入っていた (0.4.0 では削除されている)。

~~~ c
const uint8_t adc_filter_config[] = { ... }
~~~

の部分である。これをアプリケーションガイド (TLV320AIC3204 Application Reference Guide SLAA577) の
Biquad section (p27) とデータのバイトオーダー(同 p143) を見ながら $H(z)$ を再生する。

\\[ H(z) = \frac{0.02 + 0.0408z^{-1} + 0.02z^{-2}}{1 - 1.46z^{-1} + 0.888z^{-2}} \frac{0.189 - 0.377z^{-1} + 0.189z^{-2}}{1 - 1.54z^{-1} + 0.897z^{-2}} \\]

これは scipy の signal./* bb, aa = signal.ellip(2, 0.1, 100, (4500.0/24000, 5500.0/24000), 'bandpass') */


sos = np.array( [
   [ 0.02038348, 0.04076695, 0.02038348, 1.00000000, -1.46197867, 0.88831782],
   [ 0.18854105, -0.37708211, 0.18854105, 1.00000000, -1.54453158, 0.89706874] ] )




scipy によるフィルタの設計。ipython3 から、          # python コメントは space x2 # + space + 空白以外の文字、になる。フォーマットやかましい。

signal.iirfilter の引数
   (17, [50, 200], rs=60, btype='band',analog=True, ftype='cheby2')
    次数
        ナイキスト周波数基準で
    rp = 通過帯域リプル。
    rs = 阻止帯域阻止量。
    analog  Trueでアナログフィルタ。なしでデジタル。b,a 表記の次数順が変わることに注意。

冒頭のおまじない:

import numpy as np
from scipy import signal
import matplotlib.pyplot as plt
from matplotlib import ticker

ただ、ipython はコードは
%run file.ipy
で読む。もろもろの定義を atropos:/home/src/EDA/NanoVNA/calc_iir/nano.ipy と /buster:~/nano.ipy にふくめて置いたが、おまじない定義をも含んでいる。

ax = plot_pre('Bessel BPF')
plot_ba(ax, b, a, "order 5")
plot_post(ax)

ax = plot_pre_delay('delay test/ba')
plot_ba_delay(ax, b, a, "order 5, fail")
plot_post(ax)

ax = plot_pre_delay('delay test/sos')
plot_sos_delay(ax, sos, "order 5, sucess")
plot_post(ax)

のように使う。SOS の累積表示の場合、
ax = plot_pre('Elliptic BPF')

w0, h0 = signal.freqz(sos[0][:3], sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(np.array([sos[0], sos[1]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3], sos[4]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
ax.plot(24000/3.1416 * w0, 20 * np.log10(abs(h0)), label='stage1')
ax.plot(24000/3.1416 * w1, 20 * np.log10(abs(h1)), label='stage1+2')
ax.plot(24000/3.1416 * w2, 20 * np.log10(abs(h2)), label='stage1+2+3')
ax.plot(24000/3.1416 * w3, 20 * np.log10(abs(h3)), label='stage1+2+3+4')
ax.plot(24000/3.1416 * w4, 20 * np.log10(abs(h4)), label='total')

plot_post(ax)

SOS の累積を補正した表示の場合、
ax = plot_pre('Elliptic BPF')

w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w0, h0 = signal.freqz(sos[0][:3]/0.000452587, sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(np.array([sos[0]/[0.00595, 0.00595, 0.00595, 1.0, 1.0, 1.0], sos[1]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(np.array([sos[0]/[0.04615, 0.04615, 0.04615, 1.0, 1.0, 1.0], sos[1], sos[2]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(np.array([sos[0]/[0.24887, 0.24887, 0.24887, 1.0, 1.0, 1.0], sos[1], sos[2], sos[3]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3], sos[4]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
# 各段の係数 0.000452587, 0.076065, 1.64821326, 6.6227

ax.plot(24000/3.1416 * w0, 20 * np.log10(abs(h0)), label='stage1')
ax.plot(24000/3.1416 * w1, 20 * np.log10(abs(h1)), label='stage1+2')
ax.plot(24000/3.1416 * w2, 20 * np.log10(abs(h2)), label='stage1+2+3')
ax.plot(24000/3.1416 * w3, 20 * np.log10(abs(h3)), label='stage1+2+3+4')
ax.plot(24000/3.1416 * w4, 20 * np.log10(abs(h4)), label='total')

plot_post(ax)

SOS の累積の補正用に縦軸を絶対表示の場合、
ax = plot_pre('Elliptic BPF')

w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w0, h0 = signal.freqz(sos[0][:3]/0.000452587, sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(np.array([sos[0]/[0.00595, 0.00595, 0.00595, 1.0, 1.0, 1.0], sos[1]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(np.array([sos[0]/[0.04615, 0.04615, 0.04615, 1.0, 1.0, 1.0], sos[1], sos[2]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(np.array([sos[0]/[0.24887, 0.24887, 0.24887, 1.0, 1.0, 1.0], sos[1], sos[2], sos[3]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3], sos[4]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
# 各段の係数 0.000452587, 0.076065, 1.64821326, 6.6227

ax.plot(24000/3.1416 * w0, (abs(h0)), label='stage1')
ax.plot(24000/3.1416 * w1, (abs(h1)), label='stage1+2')
ax.plot(24000/3.1416 * w2, (abs(h2)), label='stage1+2+3')
ax.plot(24000/3.1416 * w3, (abs(h3)), label='stage1+2+3+4')
ax.plot(24000/3.1416 * w4, (abs(h4)), label='total')

plot_post(ax)

メモ。補正係数の入手:

In[20]:  w9,h9 = signal.sosfreqz(np.array(sos[0]), 3.14/24000 * 5000)
In[21]:  abs(h9[0])
Out[21]: 0.007167726796942493
だから、
ax = plot_pre('Elliptic BPF')

w0, h0 = signal.freqz(sos[0][:3], sos[0][3:], 3.14/24000 * 5000)
w1, h1 = signal.sosfreqz(np.array([sos[0], sos[1]]), 3.14/24000 * 5000)
w2, h2 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2]]), 3.14/24000 * 5000)
w3, h3 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3]]), 3.14/24000 * 5000)

a0  = abs(h0[0])
s0  = np.array([ sos[0]/[a0, a0, a0, 1.0, 1.0, 1.0] ]);
a1  = abs(h1[0])
s1  = np.array([ sos[0]/[a1, a1, a1, 1.0, 1.0, 1.0], sos[1] ]);
a2  = abs(h2[0])
s2  = np.array([ sos[0]/[a2, a2, a2, 1.0, 1.0, 1.0], sos[1], sos[2] ]);
a3  = abs(h3[0])
s3  = np.array([ sos[0]/[a3, a3, a3, 1.0, 1.0, 1.0], sos[1], sos[2], sos[3] ]);

w0, h0 = signal.sosfreqz(s0, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(s1, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(s2, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(s3, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))

ax.plot(24000/3.1416 * w0, 20 * np.log10(abs(h0)), label='stage1')
ax.plot(24000/3.1416 * w1, 20 * np.log10(abs(h1)), label='stage1+2')
ax.plot(24000/3.1416 * w2, 20 * np.log10(abs(h2)), label='stage1+2+3')
ax.plot(24000/3.1416 * w3, 20 * np.log10(abs(h3)), label='stage1+2+3+4')
ax.plot(24000/3.1416 * w4, 20 * np.log10(abs(h4)), label='total')

plot_post(ax)

群遅延速度比較。
sos2 = signal.iirfilter(2, [4800.0/24000, 5200.0/24000], rs=60, rp=0.3, btype='band', analog=False, ftype='bessel', output='sos')
sos3 = signal.iirfilter(3, [4800.0/24000, 5200.0/24000], rs=60, rp=0.3, btype='band', analog=False, ftype='bessel', output='sos')
sos4 = signal.iirfilter(4, [4800.0/24000, 5200.0/24000], rs=60, rp=0.3, btype='band', analog=False, ftype='bessel', output='sos')
sos5 = signal.iirfilter(5, [4800.0/24000, 5200.0/24000], rs=60, rp=0.3, btype='band', analog=False, ftype='bessel', output='sos')

ax = plot_pre_delay('delay test')
plot_sos_delay(ax, sos1, "order 2")
plot_sos_delay(ax, sos2, "order 2")
plot_sos_delay(ax, sos3, "order 3")
plot_sos_delay(ax, sos4, "order 4")
plot_post(ax)

ax = plot_pre('freq test')
plot_sos(ax, sos1, "order 2")
plot_sos(ax, sos2, "order 2")
plot_sos(ax, sos3, "order 3")
plot_sos(ax, sos4, "order 4")
plot_post(ax)


ベッセルの傾斜を消し、平坦性を増すテスト。

sos1 = signal.bessel(3, (4930.0/24000, 5070.0/24000), 'bandpass', norm='phase', output='sos' ) 
sos2 = signal.butter(2, (4850.0/24000, 5150.0/24000), 'bandpass',               output='sos' ) 
sos3 = signal.bessel(5, (4930.0/24000, 5070.0/24000), 'bandpass', norm='phase', output='sos' ) 
sos4 = signal.butter(5, (4930.0/24000, 5070.0/24000), 'bandpass',               output='sos' ) 

3段ベッセル + 2段バタワース。バタワースはやや広い。

sos0 = np.array( [ sos1[0], sos1[1], sos1[2], sos2[0], sos2[1] ] )

ax = plot_pre_delay('delay test')
plot_sos_delay(ax, sos0, "order 1")
plot_sos_delay(ax, sos2, "order 2")
plot_sos_delay(ax, sos3, "order 3")
plot_sos_delay(ax, sos4, "order 4")
plot_post(ax)

2段バタワース + 2段バタワース。  234ms@5006 - 246ms@4962Hz.

sos1 = signal.bessel(4, (4930.0/24000, 5070.0/24000), 'bandpass', norm='phase', output='sos' ) 
sos2 = signal.butter(2, (4840.0/24000, 5070.0/24000), 'bandpass',               output='sos' ) 
sos4 = signal.butter(2, (4930.0/24000, 5103.0/24000), 'bandpass',               output='sos' )
sos0 = np.array( [ sos2[0], sos2[1], sos4[0], sos4[1]  ] )
ax = plot_pre_delay('delay test')
plot_sos_delay(ax, sos1, "order  70")
plot_sos_delay(ax, sos2, "order 100")
plot_sos_delay(ax, sos3, "order 200")
plot_sos_delay(ax, sos4, "order 300")
plot_sos_delay(ax, sos5, "order 400")
plot_post(ax)

ax = plot_pre('freq. test')
plot_sos(ax, sos1, "right")
plot_sos(ax, sos3, "3")
plot_sos(ax, sos13, "1+3")
plot_post(ax)

ax = plot_pre_delay('delay test')
plot_sos_delay(ax, sos1, "right")
plot_sos_delay(ax, sos3, "3")
plot_sos_delay(ax, sos13, "1+3")
plot_post(ax)


1次BPF x 2 による最大平坦のテスト。5kHz中心、4965-5039@-0.5dB, 2.38ms.
良く似たベッセル BPF sos0 との比較。
sos は Nanovna のもの。中心が 23Hz ずれてたのでその補正含み。

sos  = signal.bessel(2, (4523.0/24000, 5523.0/24000), 'bandpass',   output='sos' ) 
sos0 = signal.bessel(2, (4890.0/24000,  5120.0/24000), 'bandpass',  output='sos' ) 
sos1 = signal.butter(1, (4960.0/24000,  5160.0/24000), 'bandpass',  output='sos' ) 
sos3 = signal.butter(1, (4844.48/24000, 5044.48/24000), 'bandpass',  output='sos' ) 

sos13 = np.array([ sos1[0] * [1.33, 1.33, 1.33, 1, 1, 1] , sos3[0] ])

ax,ax2 = plot_pre2('freq/delay test')
 
# plot_sos2(ax, ax2, sos, "nanovna 0.3.1")
plot_sos2(ax, ax2, sos0, "bessel")
plot_sos2(ax, ax2, sos13, "adjusted")

plot_post(ax)

良く似たベッセル BPF sos0 との比較、その2。 成分の構造。
sos00 = np.array([ sos0[0] / [0.0129, 0.0129, 0.0129, 1, 1, 1]/[3, 3, 3, 1, 1, 1], sos0[1] * [0.0129, 0.0129, 0.0129, 1, 1, 1]*[3, 3, 3, 1, 1, 1] ]) 

ax,ax2 = plot_pre2('freq/delay test')

plot_sos2(ax, ax2, sos00, "bessel")
plot_sos2(ax, ax2, np.array([ sos00[0] ]), "b0")
plot_sos2(ax, ax2, np.array([ sos00[1] ]), "b1")
plot_sos2(ax, ax2, sos13, "flat")
plot_sos2(ax, ax2, np.array([ sos13[0] ]), "f0")
plot_sos2(ax, ax2, np.array([ sos13[1] ]), "f1")

plot_post(ax)

elliptic で行った平坦性の改善。
対数から倍率への変換:  multi = 10^(ndb/20)

sos0  = signal.iirfilter(3, [4950.0/24000,  5050.0/24000], rp=0.00001, rs=80, btype='band',analog=False, ftype='ellip', output='sos')
sos1  = signal.iirfilter(1, [4873.0/24000,  5187.0/24000], btype='band',analog=False, ftype='butter', output='sos')
ax,ax2 = plot_pre2('elliptic BPF test')
plot_sos2(ax, ax2, sos0, "original")
plot_sos2(ax, ax2, np.array([ sos0[0]*[97.16, 97.16, 97.16, 1, 1, 1] ]), "1")
plot_sos2(ax, ax2, np.array([ sos0[1]/[16.78,16.78,16.78, 1, 1, 1] ]),   "2")
# plot_sos2(ax, ax2, np.array([ (sos0[1]/[16.78,16.78,16.78, 1, 1, 1] - [0, 0, 0, 0.2, 0, 0]) / [2.23, 2.23, 2.23, 1, 1, 1] ]), "2p")
plot_sos2(ax, ax2, np.array([ sos0[2]/[5.7876,5.7876,5.7876,1, 1, 1] ]), "3")
plot_sos2(ax, ax2, sos1, "new 1")
plot_sos2(ax, ax2, np.array([ sos1[0]/[100, 100, 100, 1, 1, 1], sos0[1], sos0[2] ]), "adjusted")
plot_post(ax)


besselベースと elliptic ベースの比較。
sos2  = signal.iirfilter(2, [4991.0/24000,  5009.0/24000], rp=0.00001, rs=80, btype='band',analog=False, ftype='ellip', output='sos')

ax,ax2 = plot_pre2('bessel and elliptic test')
plot_sos2(ax, ax2, sos13, "bessel")
plot_sos2(ax, ax2, np.array([ sos1[0]/[100, 100, 100, 1, 1, 1], sos0[1], sos0[2] ]), "elliptic")
plot_sos2(ax, ax2, sos2, "ellitic2")
plot_post(ax)





定義群:
# SOS の内情を表記。
def plot_sos(sos, title):

# SOS の一覧を表記。soss = [ sos1, sos2, ... ] として作成。
def plot_soss(soss, title):

# 周波数表示、前処理。
def plot_pre(title):

# プロット表示。
def plot_post(ax):

# SOS のプロッティング
def plot_sos(ax, sos, text):

# BA のプロッティング
def plot_ba(ax, b, a, text):

# 群遅延表示の前処理。
def plot_pre_delay(ax, b, a, text):

# BA の群遅延プロッティング
def plot_ba_delay(ax, b, a, text):

def sosgroup_delay(sos, worN):

# SOS の群遅延プロッティング
def plot_sos_delay(ax, sos, text):


b, a = signal.iirfilter(3, [0.1, 0.2], rs=30, btype='band',analog=False, ftype='cheby2')     # ナイキスト周波数基準。Fs=48kHzなら[0.1,0.2]*24kHz = 2.4kHz..4.8kHzの意。

# アナログの場合。
print(a)
[  1.00000000e+00   1.84808567e+03   1.87770708e+06   1.34306663e+09       # 最上位ベキ 1*z^-34 + ...
   7.47252211e+11   3.40596303e+14   1.31228104e+17   4.36193852e+19
   1.26821938e+22   3.25658751e+24   7.43394550e+26   1.51548630e+29
   2.76611765e+31   4.52825982e+33   6.64872357e+35   8.75806584e+37
   1.03344249e+40   1.09264159e+42   1.03344249e+44   8.75806584e+45
   6.64872357e+47   4.52825982e+49   2.76611765e+51   1.51548630e+53
   7.43394550e+54   3.25658751e+56   1.26821938e+58   4.36193852e+59
   1.31228104e+61   3.40596303e+62   7.47252211e+63   1.34306663e+65
   1.87770708e+66   1.84808567e+67   1.00000000e+68]                       # から定数項まで計35個。正規化されてない?

print(b)
[  2.55000128e+00   0.00000000e+00   3.16200158e+06   0.00000000e+00       # see. scipy.signal.freqs
   1.28367064e+12   0.00000000e+00   2.53281427e+17   0.00000000e+00       # H = b/a.
   2.82453186e+22   0.00000000e+00   1.89928175e+27   0.00000000e+00
   7.83298299e+31   0.00000000e+00   1.93887804e+36   0.00000000e+00
   2.70391048e+40   0.00000000e+00   1.93887804e+44   0.00000000e+00
   7.83298299e+47   0.00000000e+00   1.89928175e+51   0.00000000e+00
   2.82453186e+54   0.00000000e+00   2.53281427e+57   0.00000000e+00
   1.28367064e+60   0.00000000e+00   3.16200158e+62   0.00000000e+00
   2.55000128e+64   0.00000000e+00]

# デジタル。
In [3]: print(a)
[  1.          -5.13456274  11.4910984  -14.27182035  10.36566399          # デジタルは定数項から。つまり正規化済み。
  -4.17799802   0.73404825]                                                # 計7つ。つまり 6次の項まである。3つに因数分解できるはずなので、order は 3. 

In [4]: print(b)
[  1.33123329e-02  -4.64822023e-02   5.35211199e-02  -2.36474537e-17       # H = b/a
  -5.35211199e-02   4.64822023e-02  -1.33123329e-02]

w, h = signal.freqz(b,a, 1000)                          # デジタルフィルタ用。サンプル不指定なら 512個。
# w, h = signal.freqs(b, a, 1000)                       # 極周辺で 1000サンプル採取した列 h(w) を作る。アナログフィルタ用。

fig = plt.figure()
ax = fig.add_subplot(111)
ax.set_xscale('log')
ax.set_title('Bessel Type II bandpass frequency response')
ax.set_xlabel('Frequency [Hz]')
ax.set_ylabel('Amplitude [dB]')
ax.axis((2000, 10000, -80, 20))
ax.grid(which='both', axis='both')
w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
ax.plot(24000/3.1416 * w, 20 * np.log10(abs(h)), label='total')
ax.legend() 
plt.show()

# scipy オリジナル。
sos = signal.ellip(5, 0.1, 100, (4800.0/24000, 5200.0/24000), 'bandpass', output='sos')

# 各ステージ設定。
w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w0, h0 = signal.freqz(sos[0][:3], sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.freqz(sos[1][:3], sos[1][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.freqz(sos[2][:3], sos[2][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.freqz(sos[3][:3], sos[3][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.freqz(sos[4][:3], sos[4][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
# デフォルト(ROM) 設定。
w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w0, h0 = signal.freqz(sos[0][:3], sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(np.array([sos[0], sos[1]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3], sos[4]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
# scipy データの並び順は最大フラット、5kHz増幅率を 1 に。 
w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w0, h0 = signal.freqz(sos[0][:3]/0.000452587, sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(np.array([sos[0]/[0.00595, 0.00595, 0.00595, 1.0, 1.0, 1.0], sos[1]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(np.array([sos[0]/[0.04615, 0.04615, 0.04615, 1.0, 1.0, 1.0], sos[1], sos[2]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(np.array([sos[0]/[0.24887, 0.24887, 0.24887, 1.0, 1.0, 1.0], sos[1], sos[2], sos[3]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3], sos[4]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
# 各段の係数 0.000452587, 0.076065, 1.64821326, 6.6227

ax.plot(24000/3.1416 * w,  20 * np.log10(abs(h)), label='total')
ax.plot(24000/3.1416 * w0, 20 * np.log10(abs(h0)), label='stage1')
ax.plot(24000/3.1416 * w1, 20 * np.log10(abs(h1)), label='stage1+2')
ax.plot(24000/3.1416 * w2, 20 * np.log10(abs(h2)), label='stage1+2+3')
ax.plot(24000/3.1416 * w3, 20 * np.log10(abs(h3)), label='stage1+2+3+4')
ax.plot(24000/3.1416 * w4, 20 * np.log10(abs(h4)), label='total')
ax.legend() 
plt.show()

縦軸絶対スケール表示
fig = plt.figure()
ax = fig.add_subplot(111)
ax.set_xscale('log')
ax.set_title('Elliptic Type BPF')
ax.set_xlabel('Frequency [Hz]')
ax.set_ylabel('Amplitude [V]')
ax.axis((3000, 7000, -1, 100))
ax.set_xticks(np.linspace(4000, 6000, 11), minor=True)
ax.grid(which='both', axis='both')

ax.plot(24000/3.1416 * w0, (abs(h0)), label='stage1')
ax.plot(24000/3.1416 * w1, (abs(h1)), label='stage1+2')
ax.plot(24000/3.1416 * w2, (abs(h2)), label='stage1+2+3')
ax.plot(24000/3.1416 * w3, (abs(h3)), label='stage1+2+3+4')
ax.plot(24000/3.1416 * w4, (abs(h4)), label='total')
ax.legend() 
plt.show()


w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 3000, endpoint=False))
w0, h0 = signal.freqz(sos[0][:3], sos[0][3:], np.linspace(0.2618, 1.31, 3000, endpoint=False))
w1, h1 = signal.sosfreqz(np.array([sos[0], sos[1]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w2, h2 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w3, h3 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))
w4, h4 = signal.sosfreqz(np.array([sos[0], sos[1], sos[2], sos[3], sos[4]]), np.linspace(0.2618, 1.31, 3000, endpoint=False))



# 二次で因数分解した表記(Second-order sections representation):
sos = signal.iirfilter(3, [0.1, 0.2], rs=30, btype='band',analog=False, ftype='cheby2', output='sos')

print(sos)

[[ 0.01331233  0.         -0.01331233  1.         -1.66309702  0.84355724]      # 一つ目 b0,b1,b2, a0,a1,a2
 [ 1.         -1.57995815  1.          1.         -1.6695989   0.92365809]      # 
 [ 1.         -1.91170649  1.          1.         -1.80186683  0.94210393]]     # 三つ目


sos = signal.iirfilter(3, [0.2042, 0.2125], rs=50, btype='band',analog=False, ftype='bessel', output='sos')

print(sos)

[[ 0.01331233  0.         -0.01331233  1.         -1.66309702  0.84355724]      # 一つ目 b0,b1,b2, a0,a1,a2
 [ 1.         -1.57995815  1.          1.         -1.6695989   0.92365809]      # 
 [ 1.         -1.91170649  1.          1.         -1.80186683  0.94210393]]     # 三つ目

# sos表記では frez の代わりに scipy.signal.sosfreqz を使う。... んだが scipy 0.19.0 以上で、strech は 0.18.0 だわよ。
# つうか buster の 1.1.0 でも動かないんですが。... て atropos でやっとった。
# ba から sos へは tf2sos つうのがある。逆は sos2tf.
# ba から filter を実行するのは scipy.signal.lfilter,
# sos から filter を実行するのは scipy.signal.sosfilt.
# 群遅延は scipy.signal.group_delay.  引数に ba を要求する。   <- 微分なのでステップを細かくすること。

# DF-design の sos データを python になおすのに /home/src/EDA/NanoVNA/calc_iir/df2py.pl を使った。
% perl df2py.pl < data
で python 化される。


w, h = signal.sosfreqz(sos, worN=1500)

w, h = signal.freqz(b,a, np.linspace(0.001, 6.28, 3000, endpoint=False))      # w=0 点を除く。厳密に h=0 になるため、対数表記できない。

b, a = signal.sos2tf(df2)
w, gd = signal.group_delay((bb, aa), np.linspace(0.001, 6.28, 3000, endpoint=False))
plt.title('Digital filter group delay')
plt.plot(24000/3.1416 * w, gd)
plt.ylabel('Group delay [samples]')
plt.xlabel('Frequency [rad/sample]')
plt.show()

# この周波数はナイキスト基準。4900/24000 = 0.2042, 5100/24000 = 0.2125
sos = signal.iirfilter(3, [0.2042, 0.2125], rs=60, btype='band',analog=False, ftype='bessel', output='sos')
b,a = signal.iirfilter(4, [0.2042, 0.2125], rs=40, btype='band',analog=False, ftype='bessel' )         // 245@5000斜,247@4956,-80dB@4.1k,6.1k
b,a = signal.iirfilter(5, [0.2042, 0.2125], rs=60, btype='band',analog=False, ftype='bessel' )
b,a = signal.iirfilter(3, [0.20625,0.21047], rp=0.2, rs=40, btype='band',analog=False, ftype='ellip' )  // 257@5000凸,473
b,a = signal.iirfilter(3, [0.20625,0.21047], rp=0.2, rs=80, btype='band',analog=False, ftype='ellip' )  // 276@5000凸,458@4945,-80dB@4.39k,5.75k
b,a = signal.iirfilter(3, [0.20625,0.21047], rp=0.2, rs=200, btype='band',analog=False, ftype='ellip' )  // 309@5000凸,531@4948
b,a = signal.iirfilter(3, [0.2042, 0.2125], rp=0.2, rs=80, btype='band',analog=False, ftype='ellip' )  // 138@5000凸,231@4885,-80dB@3.7k,6.5k
b,a = signal.iirfilter(3, [0.2042, 0.2125], rp=0.2, rs=60, btype='band',analog=False, ftype='ellip' )  // 136        133      -60dB@4.4k,5.7k
b,a = signal.iirfilter(3, [0.2042, 0.2125], rp=0.2, rs=80, btype='band',analog=False, ftype='cheby2' )
b,a = signal.iirfilter(3, [0.2042, 0.2125], rp=0.4, rs=80, btype='band',analog=False, ftype='cheby1' ) //  158@5000凸,272@4898,-60dB@4.1k,6k.
b,a = signal.iirfilter(3, [0.2042, 0.2125], rs=60, btype='band',analog=False, ftype='butter' )
b,a = signal.iirfilter(2, [0.2042, 0.2125], rs=60, btype='band',analog=False, ftype='butter' ) // gd=108@5000凹,132@4932.-40dB@4k,6k.

b,a = signal.iirfilter(2, [0.2042, 0.2125], rp=0.2, rs=50, btype='band',analog=False, ftype='ellip' )
b,a = signal.iirfilter(3, [0.2042, 0.2125], rp=0.2, rs=50, btype='band',analog=False, ftype='ellip' )

# この周波数は角周波数基準。 3.1416/24000 * 2000 = 0.2618, 3.1416/24000 * 10000 = 1.309
w, gd = signal.group_delay((b, a), np.linspace(0.2618, 1.31, 10000, endpoint=False))
plt.title('Digital filter group delay')
plt.plot(24000/3.1416 * w, gd)
plt.ylabel('Group delay [samples]')
plt.xlabel('Frequency [rad/sample]')
plt.show()


fig = plt.figure()
ax = fig.add_subplot(111)
ax.set_xscale('log')
ax.set_title('Bessel Type II bandpass frequency response')
ax.set_xlabel('Frequency [Hz]')
ax.set_ylabel('Amplitude [dB]')
ax.axis((2000, 10000, -110, 30))
ax.grid(which='both', axis='both')

ax.plot(24000/3.1416 * w0, 20 * np.log10(abs(h0)))
ax.plot(24000/3.1416 * w1, 20 * np.log10(abs(h1)))


# b,a = signal.iirfilter(2, [0.2042, 0.2125], rp=0.5, rs=60, btype='band',analog=False, ftype='ellip')
# w, h = signal.freqz(b,a, np.linspace(0.2618, 1.31, 10000, endpoint=False))
# w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 10000, endpoint=False))


ax.plot(24000/3.1416 * w, 20 * np.log10(abs(h)))
plt.show()

sos 形式だと多段接続がいく。
np.append(sos1, sos2, axis=0)

多段接続後の group delay は加算で OK.
plt.plot(24000/3.1416 * w, gd1 + gd2)

つまり、
b,a = signal.iirfilter(5, [0.2042, 0.2125], rp=0.2, rs=100, btype='band',analog=False, ftype='ellip' )
について、
sos = signal.iirfilter(5, [0.2042, 0.2125], rp=0.2, rs=100, btype='band',analog=False, ftype='ellip', output='sos' )
として
sos[0] 〜 sos[4] に分解、それぞれについて、

sos012 = np.array( [sos[0], sos[1], sos[2]] );
sos34 = np.array( [sos[3], sos[4] ] );
個別の周波数特性は
w, h = signal.sosfreqz(sos012, np.linspace(0.2618, 1.31, 10000, endpoint=False))

で、group delay は、

b1, a1 = signal.sos2tf(sos012)
b2, a2 = signal.sos2tf(sos34)
w1, gd1 = signal.group_delay((b1, a1), np.linspace(0.2618, 1.31, 10000, endpoint=False))
w2, gd2 = signal.group_delay((b2, a2), np.linspace(0.2618, 1.31, 10000, endpoint=False))
plt.title('Digital filter group delay')
plt.plot(24000/3.1416 * w1, gd1 + gd2)
plt.ylabel('Group delay [samples]')
plt.xlabel('Frequency [rad/sample]')
plt.show()

で出来た。286samples@5kHz凸, 701samples@4896Hz, -100dB@4.6k,5.4k.
          ↑5.958ms.

現在の仕様。
# sos = signal.iirfilter(2, [0.1875, 0.22917], rs=90, btype='band',analog=False, ftype='bessel', output='sos')  26sample@5k,27sample@4.8k
# b,a = signal.iirfilter(2, [0.1875, 0.22917], rs=20, btype='band',analog=False, ftype='bessel')
b,a = signal.bessel(2, (4500.0/24000, 5500.0/24000), 'bandpass')  -15dB@3.9k,6.2k. 26sample@5k,27sample@4.8k

b,a = signal.bessel(2, (4900.0/24000, 5100.0/24000), 'bandpass', norm='delay')    76@5k,76.7@4.97k


グラフの重ね合わせ。

c1,c2,c3,c4 = "blue","green","red","black"      # 各プロットの色
l1,l2,l3,l4 = "sin","cos","abs(sin)","sin**2"   # 各ラベル

*.plot(t, y1, color=c1, label=l1)
*.plot(t, y1, color=c2, label=l2)
    ...
とする。
ループは
l = ['Alice', 'Bob', 'Charlie']
for name in l:
    if name == 'Bob':
        print('!!BREAK!!')
        break
    print(name)

ん。
sos 表記の分解前後の group_delay が一致することの検証。

b,a = signal.iirfilter(3, [0.2042, 0.2125], rp=0.4, rs=rs_val[0], btype='band',analog=False, ftype='ellip' )
sos = signal.iirfilter(3, [0.2042, 0.2125], rp=0.4, rs=rs_val[0], btype='band',analog=False, ftype='ellip', output='sos' )
sos1 = np.array( [sos[0], sos[1] ] )
sos2 = np.array( [sos[2]] )
b1, a1 = signal.sos2tf(sos1)
b2, a2 = signal.sos2tf(sos2)
w, gd  = signal.group_delay((b, a), np.linspace(0.2618, 1.31, 10000, endpoint=False))
w, gd1 = signal.group_delay((b1, a1), np.linspace(0.2618, 1.31, 10000, endpoint=False))
w, gd2 = signal.group_delay((b2, a2), np.linspace(0.2618, 1.31, 10000, endpoint=False))

plt.title('Digital filter group delay')
plt.ylabel('Group delay [samples]')
plt.xlabel('Frequency [rad/sample]')
plt.plot(24000/3.1416 * w, gd1 + gd2, color="red", label="full width")
plt.plot(24000/3.1416 * w, gd1 + gd2, color="blue", label="sum" )

plt.show()

OK. というわけで、

def sosgroup_delay(sos, worN):
    wd = 0.
    for row in sos:
        w, row_gd = signal.group_delay((row[:3], row[3:]), worN)
        wd += row_gd
    return w,wd

となる。したがって group_delay の一覧:

import numpy as np
from scipy import signal
import matplotlib.pyplot as plt

rs_list = [ [ 40, "40",  'blue',   'ellip'  ],
            [ 60, "60",  'green',  'cheby1' ],
            [ 80, "80",  'gold',   'cheby2' ],
            [100, "100", 'orange', 'bessel' ],
            [120, "120", 'red',    'butter' ]
          ]

plt.title('Digital filter group delay')
plt.ylabel('Group delay [samples]')
plt.xlabel('Frequency [Hz]')

for rs_val in rs_list:
      # sos = signal.iirfilter(5, [4990/24000, 5010/24000], rp=0.2, rs=rs_val[0], btype='band',analog=False, ftype='ellip', output='sos' )
      # sos = signal.iirfilter(4, [4990/24000, 5010/24000], rs=rs_val[0], btype='band',analog=False, ftype='bessel', output='sos' )
      # sos = signal.iirfilter(4, [4990/24000, 5010/24000], rp=0.2, rs=80, btype='band',analog=False, ftype=rs_val[3], output='sos' )
    sos = signal.iirfilter(4, [4990/24000, 5010/24000], rp=0.2, rs=rs_val[0], btype='band',analog=False, ftype='cheby2', output='sos' )
    w, gd = sosgroup_delay(sos, np.linspace(0.2618, 1.31, 10000, endpoint=False))
    plt.plot(24000/3.1416 * w, gd, color=rs_val[2], label=rs_val[1])
plt.show()

# ついでに周波数特性のほう:

fig = plt.figure()
ax = fig.add_subplot(111)
ax.set_xscale('log')
ax.set_title('signal.bessel(2, (4500.0/24000, 5500.0/24000), \'bandpass\')' )
ax.set_xlabel('Frequency [Hz]')
ax.set_ylabel('Amplitude [dB]')

for rs_val in rs_list:
    # sos = signal.iirfilter(4, [4990/24000, 5010/24000], rs=rs_val[0], btype='band',analog=False, ftype='bessel', output='sos' )
    # sos = signal.iirfilter(4, [4990/24000, 5010/24000], rp=0.2, rs=80, btype='band',analog=False, ftype=rs_val[3], output='sos' )
    sos = signal.iirfilter(4, [4990/24000, 5010/24000], rp=0.2, rs=rs_val[0], btype='band',analog=False, ftype='cheby2', output='sos' )
    w, h = signal.sosfreqz(sos, np.linspace(0.2618, 1.31, 10000, endpoint=False))
    ax.plot(24000/3.1416 * w, 20 * np.log10(abs(h)), color=rs_val[2], label=rs_val[1] )

ax.axis((2000, 10000, -200, 10))
ax.grid(which='both', axis='both')
plt.show()

b,a = signal.bessel(2, (4900.0/24000, 5100.0/24000), 'bandpass', norm='delay')    76@5k,76.7@4.97k


# データシートの 400Hz@44100Hz HPF 向け。
w, h = signal.sosfreqz(sos, np.linspace(2*3.1416/44100*10, 2*3.1416/44100*23000, 10000, endpoint=False))
fig = plt.figure()
ax = fig.add_subplot(111)
ax.plot(44100/(2*3.1416) * w, 20 * np.log10(abs(h)))
ax.set_xscale('log')
ax.set_title('Butterworth HPF, Cutoff = 400Hz@(Fs=44100Hz)')
ax.set_xlabel('Frequency [Hz]')
ax.set_ylabel('Amplitude [dB]')
ax.axis((10, 23000, -21, 3))
ax.set_yticks(np.linspace(-21, 3, 9), minor=False)
ax.grid(which='both', axis='both')
plt.show()

# 5kHz BPF 向け。
# w, h = signal.freqz(bb,aa, np.linspace(3.1416/24000*2000, 3.1416/24000*10000, 10000, endpoint=False))
w, h = signal.sosfreqz(sos, np.linspace(3.1416/24000*2000, 3.1416/24000*10000, 10000, endpoint=False))
fig = plt.figure()
ax = fig.add_subplot(111)
ax.plot(24000/3.1416 * w, 20 * np.log10(abs(h)))
ax.set_xscale('log')
ax.set_title('Chebyshev Type II bandpass frequency response')
ax.set_xlabel('Frequency [Hz]')
ax.set_ylabel('Amplitude [dB]')
ax.axis((2000, 10000, -110, 10))
ax.grid(which='both', axis='both')
plt.show()

b, a = signal.sos2tf(sos)
w, gd = signal.group_delay((b, a), np.linspace(0.2618, 1.31, 10000, endpoint=False))
plt.title('Digital filter group delay')
plt.plot(24000/3.1416 * w, gd)
plt.ylabel('Group delay [samples]')
plt.xlabel('Frequency [rad/sample]')
plt.show()



ヒストグラム作成。

> print(x)
[0, 1, 1, 1, 1, 2, 2, 2, 3, 3, 4, 4, 4, 4, 6, 6, 6, 6.1, 6.4]

に対して、

> plt.hist(x, bins=7, range=(-0.5, 6.5), ec='black' )
> plt.show()

整数配列に対しては bin,range は必須。左端を min(), 右端を max() で取って内分するので各棒グラフ中心とx軸数値の中央が合わない。

ヒストグラムそのものの用意が出来ている場合は棒グラフになる。今回はこっち。

mclk ヒストグラム。idx/100 の周波数偏差(Frequency deviation)の列 hist_mclk より、

plt.bar( np.array(idx) * 0.01, hist_mclk, ec='black', align="center",  linewidth=0.7, width=0.01, color="#FF5555")
plt.title('Frequency deviation (MCLK)')
plt.show()

半透明重ね合わせ

fig = plt.figure()
ax = fig.add_subplot(111)
ax.set_title('Frequency deviation (freq,ofreq)')
ax.set_xlabel('Difference [Hz]')

plt.bar( np.array(idx) , hist_freq, ec='black', align="center",  linewidth=0.7, color="red", alpha=0.5 , label='freq' )
plt.bar( np.array(idx) , hist_ofreq, ec='black', align="center",  linewidth=0.7, color="blue", alpha=0.5, label='ofreq' )

ax.legend()
plt.show()



