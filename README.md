## NanoVNA とかいろいろ。

ただいま準備中。

何もないのもさびしいので、とりあえず
issue で検索かけてないので他人任せに出来る分がどんくらいあるのか評価できてないけど
ざっと触って思い付いた ToDo 一覧。順不同。


- [] ハードウエアの話。SWD のピンヘッダ付け。  
  ケース付きの NanoVNA-H なので空間制約があった。

    *  裏面(部品面)の空隙 6.5mmH      //  電池の前にコネクタしこむことはできそうだ。
    *  表面(LCD面)の空隙  4.7mmH      //  QRコードの上(部品面的にはバッテリコネクタの反対側)と、電池の前の反対面にはコネクタ可。

なので、表面からピンヘッダ(全長は 12mmH) を差し込んで、LCD面側のピンはすこし折った。


- [] 周波数偏差の抑え込みとシグナルジェネレータモード。  
   現状はワーストで残差 40Hz くらい。いや Si5351 が泣くから。普通に残差 0.001Hz くらい行くから。

- [] SG モード向け、出力制御。  
   もうUSB のコンソールからはイケるっぽい? 

- [] 狭帯域スペアナモード  
   10kHz 下のイメージを除く。101点の探査で STEP < 10kHz となる場合にのみ意味がある。

- [] 超狭帯域スペアナモード  
   サンプル長を 1ms より伸ばし、弁別能力を上げる。

- [] 広帯域スペアナモード  
   CH0 出力を省略してがんばって 303点溜め込む。1 回のサンプリングで BW  = 20kHz を越えられないため、
   せめてスカスカにならないよう正弦波との内積でなく自身で二乗和してパワースペクトル採取のつもりで。

- [] 画面ローテーション  
   180度回転は LCD へのコマンド一発? 問題はポートレートモードだけど、メモリ足りるか?

- [] 文字ばっかりのテスターモード  
   どっかで上がってたような。

- [] オートスリープ  
  ミキサが寝ないので意味あるかどうかは実測次第。デープスリープすると復帰できんし。

- [] LCDの照明制御  
  コマンド一発のような気もする。

- [] エクスポネンシャル平均によるノイズ除去。   
  誰かやってるだろう。つうか本家本元にさすがにあるんでないかと想像する。

- [] x3インターリーブモード  
  LCD は 303点表示できそうなんだし、1/3 ずつ周波数ずらして 303点分の表示 ... はメモリ足んないのかなぁ。

- [] キャリブレーションデータの複数統合。    
  1500MHz までと広くなったが、高域のオマケ領域にキャリブレーションサンプル点を食われてサンプル数が足りない。
  50kHz - 300MHz を 1 にセーブ、300 MHz - 1500 MHz を 2 にセーブして、50kHz から 1500MHz をスキャンする時はキャリブレーションデータ 1,2 を両方使うようにする。

- [] ADCゲインキャリブレーション  

- [] UI開発用のシミュレータ。  
  UI のブラッシュアップ用に、UI 部分だけ PC 上で動かせるように。基本 32x32 の bitblt のようだから大きな問題はないはず。

- [] Si5351 から ADC 出力までの周波数偏位速度テストコード  

- [] ADC のフィルタまわりの特性測定テストコード  

- [] LCD上、背景とかぶる文字列の位置の移動/オンオフ  
  スミスチャートにかぶると逃げ場がない。

- [] 下限 32kHzへの拡張。  
  ぱっと見で 7kHz まではそのままいけそうな。32kHz 水晶の調査に使えるんでないかなぁ。

- [] ジョグダイヤルのヘルプ出力。   
  何をトリガにしてヘルプだすんだ、とかそういうことはそのうち。

- [] MCU がどんな周波数で動いてるかのレポートを info に追加。  
  マニュアル付属のブロック図が間違っていて STM32 は Si5351 からでなく自前の HSI クロックで動いているので。

- [] スペアナモードでアンテナつけて自分自身のノイズ源探し。  

- [] ハードウエアの話。補助出力ピン追加。  
  Si5351 の output2 を 8MHz にきっちり固定した上でだが (現状は 8MHz から僅かにブレることがある)、外部機器の同期用の出力を取り出す。

- [] ハードウエアの話。UART の追加 (Bluetooth の追加)。  
  0.5mm のハンダ付けしなくてもジョグダイヤルとヘッダの TCK から USART2 が引き出せる。あとはジョグダイヤルの GPIO と共存が出来れば。		  

- [] ハードウエアの話。STM32F303 への換装。  
  USB DP のプルアップが最大の焦点だが、これも 0.5mm ピッチからの線引き出しは不要。USB コネクタ近くにスルホール、ヘッダに VDD がある。


