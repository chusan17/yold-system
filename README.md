# YOLD - Drone Detector -
You Only Look Drone with Jetson Nano

![ian-usher-JPAfSd_acI8-unsplash](https://user-images.githubusercontent.com/79794586/109420853-e1704580-7a17-11eb-8e66-47ff49b930a5.jpg)

## Introduction

Welcome to yold-system repo.
YOLD is Drone detector based on YOLOv4 tiny (You Only Look Once).
100+ drone pictures are used for re-traning `YOLOv4 tiny` to create a new object detection network which detects only Drone.

And retrained `YOLOv4 tiny` would be installed on Jetson Nano 2GB to perform Real-Time Drone detection with webcam.

## Configuration

In this repo, Colaboratory file is restored.
this file shows how to re-train `YOLOv4 tiny` using your original dataset in Colaboratory.
YOLO needs unique dataset format, you need to create dataset for YOLO with [this labeling tool](https://github.com/tzutalin/labelImg).

**Inference should be performed inside Jetson which means no necessary Internet connection during Real-Time detection.**
But Notification to your LINE account introduced below needs WiFi for API call.

### Hardwares
- Jetson Nano 2GB
- USB-C power supply (5V 3A)
- Webcam (Logicool C525n)
- WiFi

## Re-train YOLOv4 tiny with your original dataset

open Colaboratory and change to GPU mode.
And get `YOLOv4 (darknet)`.
```
!git clone https://github.com/AlexeyAB/darknet
```
rewrite makefile to use GPU and OpenCV.
```
%cd darknet
!sed -i 's/OPENCV=0/OPENCV=1/' Makefile
!sed -i 's/GPU=0/GPU=1/' Makefile
!sed -i 's/CUDNN=0/CUDNN=1/' Makefile
!sed -i 's/CUDNN_HALF=0/CUDNN_HALF=1/' Makefile
```
do `make` it.
```
!make
```
これには少し時間がかかります。
できたら次は、学習済みの weight を落とします。

```
!wget https://github.com/AlexeyAB/darknet/releases/download/darknet_yolo_v4_pre/yolov4-tiny.conv.29
```
そして、自分の drive をマウントして、データセットを読み込めるようにしましょう。
実は、YOLO の学習にはデータセット以外にも`obj.data.txt`や`yolov4-tiny-custom.cfg`などの設定ファイルが必要です。
用意しましょう。

まず`obj.data.txt`を作って以下を書き込みます。

```txt:obj.data.txt
classes = 1
train = /train.txt
valid = /test.txt
names = /obj.names.txt
backup = /backup
```
今回はドローンを検知するだけなので`classes`は`1`にします。
`obj.names.txt`にはラベル名を記載します。今回はテキストファイルに`drone`とだけ記載すればOKです。
`backup`フォルダには学習済みの重みが記録されます。

`yolov4-tiny-custom.cfg`はニューラルネットのアーキテクチャーを設定するファイルです。
[こちら](https://github.com/alexeyab/darknet#how-to-train-to-detect-your-custom-objects)を参考に内容を書き換えましょう。

`train.txt`と`test.txt`は各データセットの画像へのPATHを記載したテキストファイルです。
仮に、親フォルダ名が`folder`、trainデータセットのフォルダ名が`obj`の場合、以下のコードで生成できます。

```python

import os

image_files = []
os.chdir(os.path.join("folder", "obj"))
for filename in os.listdir(os.getcwd()):
    if filename.endswith(".jpg"):
        image_files.append("folder/obj/" + filename)
os.chdir("..")
with open("train.txt", "w") as outfile:
    for image in image_files:
        outfile.write(image)
        outfile.write("\n")
    outfile.close()
os.chdir("..")
```
Colaboratoryで実行すると、`train.txt`が生成されたはずです。
`test.txt`も同様に作ります。
最終的に以下のファイル/フォルダが同じ場所にあればOKです。

- backup（学習済みweightが保存されるフォルダ）
- obj（トレーニングデータセットが格納されているフォルダ）
- test（テストデータセットが格納されているフォルダ）
- train.txt
- test.txt
- obj.data.txt
- obj.names.txt
- yolov4-tiny-custom.cfg

準備OKですので、オリジナルデータセットで darknet を訓練しましょう！

```
!./darknet detector train obj.data.txt yolov4-tiny-custom.cfg yolov4-tiny.conv.29 -dont_show -map
```
約100枚のドローン画像を用いた学習で約1時間半かかりました。
学習が終わると、`backup`フォルダに`yolov4-tiny-custom_best.weights`が生成されます。
こいつと`obj.data.txt` `obj.names.txt` `yolov4-tiny-custom.cfg`を Jetson Nano に移植して推論していきます！

## Jetson Nano の環境構築

基本的には Colaboratory と同じ環境を Jetson Nano にも作ってあげます。
初期設定などは以下の動画を参考に進めていけばOKです。
これで必要なモノがだいたいインストールされるので、この後は足りないものをちょくちょくダウンロードしていく感じです。

Jetson AI Fundamentals - S1E1 - First Time Setup with JetPack
https://youtu.be/uvU8AXY1170

Jetson AI Fundamentals - S3E1 - Hello AI World Setup
https://youtu.be/QXIwdsyK7Rw

Jetson のターミナルを起動し、YOLOv4 (darknet) を入れたいディレクトリに移動して`git clone`です。

```
$ git clone https://github.com/AlexeyAB/darknet
```
次に Colaboratory 同様に makeflie をいじるのですが、少し方法が違います。
[こちらの記事](https://qiita.com/tayutayufk/items/3d715184e0a7cefa5e9a)を参考に変更しましょう。

できたら`make`コマンドでビルドします。

```
$ make
```
これで準備できました。
Jetson Nano + WEBカメラで YOLOv4 のリアルタイム推論を発動させましょう。

```
$ ./darknet detector demo obj.data.txt yolov4-tiny-custom.cfg yolov4-tiny-custom_best.weights -thresh 0.1 >darknet.log
```
デフォルトで接続されたWEBカメラを使ってくれるはずです。
`- thresh`で検知の閾値を設定します。推論結果として一種の確度(%)みたいなものを出力するのですが、`thresh 0.1`なら`10%`以上の確度を持ってドローンと分類したものだけを表示してくれます。

`>darknet.log`は次に説明するアラート通知に使うためにログを吐き出しています。

## LINE通知の仕組みを作る

今回は`swatch`で`darknet.log`を監視して、「drone」というワードが出てきたら .shファイルを実行させ`LINE Notify`の API を叩きます。

まず`swatch`をインストールします。

```
$ sudo apt-get install swatch
```
次に`nano`や`vim`で`swatch.conf`ファイルを作ります。
>※ nano や vim が入っていない場合はインストールします。
※ アクセス権限も適宜変更してください。

```
$ vim swatch.conf
```
```txt:swatch.conf
watchfor /drone/
	exec /LINEnotify.sh
	threshold track_by=/drone/,type=both,count=3,seconds=10
```
`type=both,count=3,seconds=10`とすると、
「time interval が10秒に設定され、time intervalの間に 3 回ドローンを検知した時点で`LINEnotify.sh`を実行。その後はtime intervalが切れるまで無視される。」
という設定になります。
検知する度にLINEがくるのは嫌なので。。

実行ファイルには以下のように記述します。

```sh:LINEnotify.sh
curl -g -X POST -H "Authorization: Bearer ACCESS_TOKEN" -F "message=ここにメッセージを書く" https://notify-api.line.me/api/notify

echo
exit 0
```
`ACCESS_TOKEN`に発行されたトークンを記入します。
トークンの発行方法は[こちら](https://qiita.com/iitenkida7/items/576a8226ba6584864d95)を参照ください。

これで完成です！
リアルタイムでドローン検知しているときに以下のように実行すると、

```
$ swatchdog --config-file=/swatch.conf --tail-file=/darknet.log
```
こんな感じでLINEのメッセージが届きます。
![LINEnotify_Trim.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/386486/4d5b14b1-7468-34b2-95fd-480d990de7c9.png)

## おわりに

Jetson Nano 2GB でオリジナルデータセットを学習した YOLOv4 tiny を動かしてみました。
この構成でだいたい`5 FPS`は出てます。
飛行しているトイドローンの横で本をひらひらさせても誤検知しません。
100枚程度で tiny なのに、なかなか優秀です。

![example02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/386486/fdaefa86-edca-848f-08cb-9a1d6a603124.png)

これで部屋にドローンが侵入しても安心ですね！

## 参考文献
- [JETSON AI COURSES AND CERTIFICATION](https://developer.nvidia.com/ja-jp/embedded/learn/jetson-ai-certification-programs)
- [LINE Notify](https://notify-bot.line.me/ja/)
- [GitHub - dusty-nv/jetson-inference: Hello AI World guide to deploying deep-learning inference networks and deep vision primitives with TensorRT and NVIDIA Jetson.](https://github.com/dusty-nv/jetson-inference)
- [GitHub - AlexeyAB/darknet: YOLOv4v / Scaled-YOLOv4 - Neural Networks for Object Detection (Windows and Linux version of Darknet )](https://github.com/AlexeyAB/darknet)
- [Train YOLOv4-tiny on Custom Data - Lightning Fast Object Detection](https://blog.roboflow.com/train-yolov4-tiny-on-custom-data-lighting-fast-detection/)

<br>
