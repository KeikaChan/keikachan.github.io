# 録画データをキレイに変換したい
## これをやったモチベーション
プリチャンの録画データを毎回向きを直すために変換を行っているのだが、最適な変換方法を知りたい。  
現状では h.264 で 2pass エンコーディングで変換しているけど、 h.265 の方が高画質を維持しつつファイル容量を抑えられるんじゃないかって思って比較。

※動画の傾きのみであれば、ファイルの向きデータが入ってる**ヘッダ部分を修正するだけでも無劣化で回転可能**だが、**対応していないソフトウェアがそれなりにある**ため、ffmpegで変換しています。

## 想定される結果
h.265 の方がきれいなんじゃない？  
公式では h.264 と比較して2倍の圧縮性能が出ているから、同じビットレート上では h.265 の方がきれいなはず。

## 使った動画データ
まとめて release に置いてあります。  
[https://github.com/KeikaChan/keikachan.github.io/releases/tag/cc1](https://github.com/KeikaChan/keikachan.github.io/releases/tag/cc1)

## 比較環境
- コーデックは h.264 と h.265
- どちらも録画データ→回転しつつ変換
- 使用ソフトは[ffmpegベースで自作したもの](https://github.com/KeikaChan/VideoRotator)を使用
- それ以外のビットレートやフレームレートなどのパラメータは同様。
- 良し悪しは SSIM で評価 (後述)

## SSIMについて
SSIMは画像について、**元画像と比較して変換画像がどれくらい劣化したか**を示す指標です。  
SSIMはPSNRよりもより人の視覚特性に近い評価手法で、0〜1の範囲で値が出ます。  
その値が大きければ大きいほど劣化が少ないことになります。
なので今回は動画から適当にフレームを抜き出して比較します。  

以下基準

|PSNR|SSIM|主観評価|
|---|---|---|
|40～∞［dB］|0.98以上|元画像と圧縮画像の区別がつかない|
|30～40［dB］|0.90～0.98|拡大すれば劣化がわかるレベル|
|30以下［dB］|0.90以下|明らかに劣化がわかる|

※ [電子化文書の画像圧縮ガイドライン](https://www.jiima.or.jp/pdf/5_JIIMA_guideline.pdf) より引用


## 結果
### ファイル容量とか

|元ファイル|h.264|h.265|
|---|---|---|
|1352MB|1351MB|<span style="color: red; ">**846.9MB**</span>|


元ファイルと h.264 は同じコーデックで同じビットレートの変換なのでほぼ同じ容量になるのは予想できましたが、 h.265 では元データと比較して**6割**ほどまで容量を落とすことができました！！  
さすが後続規格  

で、肝心の綺麗さです。
### SSIMとか
#### 前提
全フレームやるのはデータをまとめるのが超面倒なので、適当に秒数指定して取ってきたフレームを用いて計算することにしました。  
計算とかはこんな感じのシェルスクリプト組んでサクッとやりました。


```test.sh
#!/bin/bash
echo "extract images"
./ffmpeg -ss $1 -i ./190110.mp4 -vframes 1 -f image2 ./origin_$1_orig.png
convert -rotate 270 ./origin_$1_orig.png ./origin_$1.png
rm ./origin_$1_orig.png
./ffmpeg -ss $1 -i ./190110-rotate_h264.mp4 -vframes 1 -f image2 ./h264_$1.png
./ffmpeg -ss $1 -i ./190110-rotate_h265.mp4 -vframes 1 -f image2 ./h265_$1.png
echo "calc SSIM h264"
./ffmpeg -i ./origin_$1.png -i ./h264_$1.png -filter_complex ssim -an -f null -
echo "calc SSIM h265"
./ffmpeg -i ./origin_$1.png -i ./h265_$1.png -filter_complex ssim -an -f null -
```

切り出した秒数は 1　/ 100 / 280 でやりました。特に深い意味はないです。  


|1秒|100秒|280秒|
|---|---|---|
|<img src="https://user-images.githubusercontent.com/46399635/50964641-27ee4780-1513-11e9-99ed-4e4ae7753a9e.jpg" width="150">|<img src="https://user-images.githubusercontent.com/46399635/50964643-27ee4780-1513-11e9-872c-7165b8dc0e3a.jpg" width="150">|<img src="https://user-images.githubusercontent.com/46399635/50964645-2886de00-1513-11e9-8e10-f4a426ac8d4e.jpg" width="150">|


うん、かわいい。

#### 結果

|秒数|SSIM h.264|SSIM h.265|
|---|---|---|
|1秒|<span style="color: red; ">**All:0.989390 (19.742779)**</span>|All:0.969887 (15.212454)|
|100秒|<span style="color: red; ">**All:0.968287 (14.987674)**</span>|All:0.956347 (13.599840)|
|280秒|<span style="color: red; ">**All:0.981853 (17.412039)**</span>|All:0.954132 (13.384881)|

<span style="color: red; ">**え、 h.264 の方が良いじゃん🤔🤔🤔**</span>

全部の出力画像を見てみたい方はこちらからどうぞ↓ 
※結構重いです  
[https://github.com/KeikaChan/keikachan.github.io/issues/2](https://github.com/KeikaChan/keikachan.github.io/issues/2)

## 考察的な何か
h.264よりh.265の方が 0.01 〜 0.03 ぐらい低いのが結構気になる。  
SSIM で考えると h.265 は**スクショ取って拡大するとギリギリわかるレベルだと思う**。  
理論的には h.265 方が良くなるはずなんだけど、まだまだソフトウェアの性能が足りないのかな...  
しかし、元データと比較して6割程容量を減らすことのできる h.265 は驚異的。  
HDDの余裕がない人は h.265 にするのも**現実的な選択肢としてはあり**だと思います。  

他に良い変換方法があったら知りたい...

## 最後に
動画変換中にちょこっと思いついて試してみた無いようなので全然データが足りないですが、動画変換する際の参考になれば幸いです〜〜
でわでわ。
