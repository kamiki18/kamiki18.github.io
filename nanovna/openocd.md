---
layout: default
title: openocd
auther: kamiki18
revision: Nov. 27, 2019.
---

## SWD へ繋ぐ Openocd


### ヘッダ付け

NanoVNA 基板は SWD 用に 5pin の 2.54mmピッチのコネクタが付けられるようになっているが、
黒ABSケース入りNanoVNA-H だと案の定、上下が狭くて何も考えずにヘッダを付けるとケースが閉まらない。
L字? ケースに穴開けめんどくさい。

 >        ケースサイズ 91.5mm x 60.0mm x 17mm        // ケース無しは 85.5 x 54mm x 11mm だそうだから、ケース厚みは余裕込みでちょうど 3mmt かな。
 >        部品面の基板上空  6.5mmH
 >        LCD面の基板上空   4.7mmH
 >        基板version: v3.3, レジストは青。
 >        バッテリコネクタ molex 53261-0271(多分)    // NanoVNA-H の回路図は2.54mmピッチのが指定されているが現物は1.25mm のものだった。
 >                                                   // 多分、というのはつまり molex 51021-0200 持ってきての嵌合テストはしてないので。

普通のピンヘッダは基板上空に 12mm 要るから駄目。そこでピンヘッダを LCD面から挿し、部品面側に飛び出した部分がソケットと嵌合するようにした。
これでも僅かにピンが高くてケースにぶつかるので、すこし押し下げて LCD面側に飛び出たピンをニッパで切り落とすことになった。

![ヘッダ周辺](/nanovna/images/swdpin.jpg)

なお、ピンのはんだ付けは部品面からである。写真で基板の台座になっているのが NanoVNA-H のABSケース。高さの比較用に。


### ケーブル製作


jtagkey-tiny clone で openocd を使って NanoVNA の STM32F072 まで。

ケーブルの回路まわり:

 >
 >                                                     <----------------> 作成したケーブル
 >                                         
 >                          jtagkey clone の中ここまで↓                ↓NanoVNAの中ここから
 >                                                    ↓                ↓
 >                                                    ↓                 -- 1pin/VDD
 >                       VDD --------------------------                  -- 2pin/NRST STM32F072Cの7pin/nRST
 >                       GND ---------------------------------------------- 3pin/GND
 >     2232Lの24pin/AD0  TCK -------|＞-------100ohm ---------------------- 4pin/TCK  STM32F072Cの37pin/PA14/SWCLK
 >     2232Lの22pin/AD2  TDO_in ------＜|-----100ohm ---------------+------ 5pin/TMS  STM32F072Cの34pin/PA13/SWDIO
 >     2232Lの23pin/AD1  TDI -------|＞-------100ohm ----- 470ohm --+
 >            21pin/AD3  TMS -----------------100ohm --
 >            17pin/AD6  nSRST_in ----＜|--+
 >            13pin/AC1  nSRST -----|＞----+--100ohm --
 >            15pin/AC0  nTRST -----|＞-------100ohm --
 >            20pin/AD4  /OE
 >            11pin/AC3  nSRST /OE
 >            12pin/AC2  nTRST /OE

![ケーブル回路の実装](/nanovna/images/swdcable.jpg)


TDI のところに 470Ωの抵抗のリード線を挿して圧着、TDO のところに AWG28 と 0.3mmφスズメッキ線を同時に圧着する。
そのスズメッキ線と抵抗をはんだ付けして主要部は完成。回路図がそうなっているからといって、AWGワイヤの途中ではんだ付けとかは止めましょう。

はんだ付け部よりさらに右、抵抗のリード線が茶色の線にクルクル巻いてるのは特に意味ない、
というか抵抗をねっこまで挿しこんでしまってるから、そこに力が掛かるので害悪かも。

繋いだのは 3 本だけで VDD も NRST も繋いでいないけども、

 * VDD について。NanoVNA の VDD ピンはバッテリ内蔵であるかどうかにかかわらず出力ピンなので、jtagkey 側の VDD を入力に切替える手間を嫌った。
   フールプルーフ的にデフォルトを入力にしてあったのを忘れてハマったのは秘密。
 * NRST について。n*RST を繋ぐ場合、clone の中で VDD にプルアップされるため、おそらく VDD のラインも繋ぐ必要がある。
   NanoVNA回路上プルアップ/ダウンともに無いので繋ぐ時に何をどうするかは検討もせずに繋がなかった。やみくもに繋いでよいかどうかは不明。
   繋いでなくてもリセットは機能した。 

jtagkey の動作として、

 * TCK,TDI は /OE が L の時のみ有効。
 * TDO_in, nSRST_in, Vref_in は常に有効。
 * nSRST 出力を L にしたい時は nSRST を下げたあと nSRST /OE を下げる。

である。
上のケーブルの回路は openocd に付属する `swd-resistor-hack.cfg` の回路そのものだが、jtagkey の場合、良く見ると SRST に 3-state の OR のループバック経路が存在する。
つまり nSRST を L に引いておき、nSRST /OE に TDI を割り当て、nSRST_in に TDO_in を割り当てれば抵抗なしのケーブルで動くんじゃなかろうか。
`ftdi_layout_signal` で TCK や TDI を指定できるのか知らないけど。

~~~ sh

% cat nanovna.cfg
interface ftdi                                        # interface/ftdi/jtagkey2.cfg からコピって desc,vid 改変。
ftdi_device_desc "Dual RS232"  
ftdi_vid_pid 0x0403 0x6010
ftdi_layout_init 0x0c08 0x0f1b                        # 方向 = 0f1b で AC0-3,AD4,AD3,AD1,AD0 が出力ピン。初期値 = 0c08で AC2,AC3,AD3が H,それ以外が L.
ftdi_layout_signal nTRST -data 0x0100 -noe 0x0400     #  data並び順は AC7-AC0,AD7-AD0. 0x0100ならAC0, /OE は AC2. -data なので反転なし。
ftdi_layout_signal nSRST -data 0x0200 -noe 0x0800     #  AC1 に nSRST, AC3 に /OE.  openocd の論理的にはこれで nSRST を NRST に繋ぐことができるはず。
transport select swd                                  # interface/ftdi/swd-resistor-hack.cfg より。
ftdi_layout_signal SWD_EN -data 0
source [find target/stm32f0x.cfg]                     # 内で adapter_khz 1000 している。
reset_config none                                     
adapter_khz 500                                       #  500 でも 1000 でも 8000 でもたいしてかわらない。

~~~

この cfg は openocd 0.9.0 用。cfg の書き方ころころ変えるのほんとやめてほしい。まあ愚痴はともかく openocd は動くようになった。


### openocdの速度


~~~ sh
# /usr/bin/openocd -f test.cfg  

Open On-Chip Debugger 0.9.0 (2018-01-21-13:43)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : FTDI SWD mode enabled
adapter speed: 1000 kHz                                         #   デフォルト。MCLK/6 まで動くはず。つまり 8000kHz に耐えるはず。
adapter_nsrst_delay: 100
none separate
cortex_m reset_config sysresetreq
none separate
adapter speed: 500 kHz
Info : clock speed 500 kHz
Info : SWD IDCODE 0x0bb11477
Info : stm32f0x.cpu: hardware has 4 breakpoints, 2 watchpoints
Info : accepting 'telnet' connection on tcp/4444
flash
  flash bank bank_id driver_name base_address size_bytes chip_width_bytes
            bus_width_bytes target [driver_options ...]
  flash banks
      ...

~~~

で、他から

~~~ sh

% telnet 127.0.0.1 4444
Escape character is '^]'.
Open On-Chip Debugger
      ...
> program ch.bin 0x08000000 verify reset exit                      #  書き込みコマンド発行。
      ...
wrote 98304 bytes from file ch.bin in 4.243720s (22.622 KiB/s)     #  170kbit/s しかない。500kHz指定しようが要するにほとんど erase の時間ということ。
      ...
verified 96444 bytes in 1.179007s (79.884 KiB/s)                   #  実際、こちらはちゃんと 640kbit/s 出ている(むしろ速くない?)。
** Verified OK **
** Resetting Target **                                             #  NRSTに繋いでなくてもリセットは動作した。

~~~


### ToDo.

dfu があまり機能してなくてしょうがなく作ったようなものだが、
テストだけなら本体に触らなくても出来るようになったかなぁ。

ということで全般的に dfuの信頼感向上を希望。

~~~ sh

void HardFault_Handler(void)
{
    while (true) {}
}

~~~

してないでハンドラから dfu 呼ぶとか。現状だとメニューが上がるまえにコケると手も足も出なくなる。
壊れたファームにあたった時にフェイルバックすらできない。
実際、メモリが不足気味なのに -O2 でコンパイルしてるのを不思議に思って Makefile を書き換え -Os にしたら↑になった。

それと、｜逸般人向けファーム《developer version》でなら USB が活きてたらそっちにフォールトの｜詳細報告《ブルスク》出してから reboot するコード希望。


### References  {#ref}

 * NanoVNA-a1-20170115.pdf ([NanoVNAキット](https://ttrf.tk/kit/nanovna/) より)
 * RM0091 Reference manual STM32F0x1/STM32F0x2/STM32F0x8 (DocID018940 Rev 9)
 * NanoVNA User Guide_20190711.pdf  ([Nano-VNA-H](https://github.com/hugen79/NanoVNA-H) の doc内 )
 * STM32F072x8,STM32F072xB (DS9826 rev 6)
 * SD103AWS datasheet   <!-- https://www.diodes.com/assets/Datasheets/ds30101.pdf -->
 * [NanoVNA-Q](https://github.com/qrp73/NanoVNA-Q)



