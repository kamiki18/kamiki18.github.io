---
layout: default
title: I2C readback, etc.
auther: kamiki18
revision: Nov. 25, 2019.
---

## I2C command.


TLV320AIC3204 の miniDSP にフィルタを書き込むにあたり、TLV320AIC3204 のレジスタがどうなってるかを見るための I2C readback が欲しくなった。
リファレンスガイドの Figure 2-55. I<sup>2</sup>C Read (p79) を見ながら書く:

~~~ c

int tlv320aic3204_read(uint8_t d0)
{
  int addr = AIC3204_ADDR;
  uint8_t buf[] = { d0 };
  i2cAcquireBus(&I2CD1);
  i2cMasterTransmitTimeout(&I2CD1, addr, buf, 1, NULL, 0, 1000);
  i2cMasterReceiveTimeout(&I2CD1, addr, buf, 1, 1000);
  i2cReleaseBus(&I2CD1);
  return buf[0];
}

~~~

で、I2C 読み書き用のコマンド `cmd_i2c()` を適当にこしらえておく。

~~~ sh

pause                 #   sweep を止める。
i2c  0 8              #   8ページ目を選択。
i2c  0                #   reg0 を読む。
0x8                   #   無事 8ページ目が選択された。
i2c  1                #   Page8/reg1 を読む。
0x4                   #   Adaptive mode は有効になっている。
i2c  1  5             #   Page8/reg1 に 5 を書き込んで ADC 内のバッファにあったフィルタデータを miniDSP に送り込む。 

~~~

まあ、わりとそれだけだが。ただこれまでまったく参照されていなかった `i2cMasterReceiveTimeout()`
がファームの中に含まれるようになったのでコードサイズは少し増えた。
コンパイルオプションで -O2 でなく -Os を使ってるので世間的にサイズ変化がどうであるかはよくわからない。


### Known BUG というか制限

sweep 中は TLV320AIC3204 にゲイン設定などを発行しているので、ページ選択して、レジスタを読み書き、の間に選択されたページが変わってしまう。
読み書きする時は sweep を止めること。コンソールの i2c コマンドのほうで設定したページを覚えておくとかは、出来なくはないがちょっとやりたくない。


### Reference {#ref}

 * TLV320AIC3204 Application Reference Guide (SLAA577 - November 2012)

