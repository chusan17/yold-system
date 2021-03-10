# YOLD - Drone Detector -
You Only Look Drone with Jetson Nano

![ian-usher-JPAfSd_acI8-unsplash](https://user-images.githubusercontent.com/79794586/109420853-e1704580-7a17-11eb-8e66-47ff49b930a5.jpg)

## Introduction

Welcome to yold-system repo.<br>
YOLD is Drone detector based on YOLOv4 tiny (You Only Look Once).<br>
100+ drone pictures are used for re-traning `YOLOv4 tiny` to create a new object detection network which detects only Drone.<br>

And retrained `YOLOv4 tiny` would be installed on Jetson Nano 2GB to perform Real-Time Drone detection with webcam.<br>

## Configuration

In this repo, Colaboratory file is stored.<br>
this file shows how to re-train `YOLOv4 tiny` using your original dataset in Colaboratory.<br>
YOLO needs unique dataset format, you need to create dataset for YOLO with [this labeling tool](https://github.com/tzutalin/labelImg).<br>

**Inference is performed in only Jetson which means no necessary Internet connection during Real-Time detection.**<br>
※ Notification to your LINE account introduced below needs WiFi for API call.<br>

### Hardwares
- Jetson Nano 2GB
- USB-C power supply (5V 3A)
- Webcam (Logicool C525n)
- WiFi<br>

## Re-train YOLOv4 tiny with your original dataset

Open your Colaboratory and change to GPU mode, and let's get `YOLOv4 (darknet)`.
```
!git clone https://github.com/AlexeyAB/darknet
```
Rewrite makefile to use GPU and OpenCV.
```
%cd darknet
!sed -i 's/OPENCV=0/OPENCV=1/' Makefile
!sed -i 's/GPU=0/GPU=1/' Makefile
!sed -i 's/CUDNN=0/CUDNN=1/' Makefile
!sed -i 's/CUDNN_HALF=0/CUDNN_HALF=1/' Makefile
```
Do build it.
```
!make
```
It takes lots of time.<br>
Once finished, download pre-trained weights from YOLOv4 repo.
```
!wget https://github.com/AlexeyAB/darknet/releases/download/darknet_yolo_v4_pre/yolov4-tiny.conv.29
```
And mount your own google drive on Colaboratory to read datasets you made.<br>
Actually lots of files described below are needed for training YOLO.<br>
All of files are stored in this repo, please download and put them into same folder.<br>
- backup         :folder where generated weights are stored
- obj            :folder for training dataset
- test           :folder for test dataset
- train.txt      :describe PATH for training images
- test.txt       :describe PATH for test images
- obj.data.txt
- obj.names.txt
- yolov4-tiny-custom.cfg

If you have prepared all of files, let's train `YOLOv4 tiny` with your datasets.
```
!./darknet detector train obj.data.txt yolov4-tiny-custom.cfg yolov4-tiny.conv.29 -dont_show -map
```
Once training finished, `yolov4-tiny-custom_best.weights` would be generated in `backup` folder.<br>
That weights and `obj.data.txt` `obj.names.txt` `yolov4-tiny-custom.cfg` are put into Jetson Nano 2GB to do inference at local environment.
<br>
## Setting up Jetson Nano 2GB

If you follow below videos to set up Jetson Nano, almost all of things are needed for Real-Time detection will be done installed on your Jetson.
<br>
After that, if not enough, download from your jetson terminal separately.
<br>
Jetson AI Fundamentals - S1E1 - First Time Setup with JetPack
<br>
https://youtu.be/uvU8AXY1170
<br>
Jetson AI Fundamentals - S3E1 - Hello AI World Setup
<br>
https://youtu.be/QXIwdsyK7Rw
<br>
All procedures in these video have been done, open up your terminal, move to directory where you want to install `YOLOv4 (darknet)`, get YOLO from github repo.
```
$ git clone https://github.com/AlexeyAB/darknet
```
Change makefile according to [this article](https://qiita.com/tayutayufk/items/3d715184e0a7cefa5e9a), and then do `make` it.
```
$ make
```
Ready for YOLD!
Let's 
Jetson Nano + WEBカメラで YOLOv4 のリアルタイム推論を発動させましょう。
```
$ ./darknet detector demo obj.data.txt yolov4-tiny-custom.cfg yolov4-tiny-custom_best.weights -thresh 0.1 >darknet.log
```
デフォルトで接続されたWEBカメラを使ってくれるはずです。
`- thresh`で検知の閾値を設定します。推論結果として一種の確度(%)みたいなものを出力するのですが、`thresh 0.1`なら`10%`以上の確度を持ってドローンと分類したものだけを表示してくれます。

`>darknet.log`は次に説明するアラート通知に使うためにログを吐き出しています。
<br>
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
<br>
## おわりに

Jetson Nano 2GB でオリジナルデータセットを学習した YOLOv4 tiny を動かしてみました。
この構成でだいたい`5 FPS`は出てます。
飛行しているトイドローンの横で本をひらひらさせても誤検知しません。
100枚程度で tiny なのに、なかなか優秀です。

これで部屋にドローンが侵入しても安心ですね！
<br>
## Reference
- [JETSON AI COURSES AND CERTIFICATION](https://developer.nvidia.com/ja-jp/embedded/learn/jetson-ai-certification-programs)
- [LINE Notify](https://notify-bot.line.me/ja/)
- [GitHub - dusty-nv/jetson-inference: Hello AI World guide to deploying deep-learning inference networks and deep vision primitives with TensorRT and NVIDIA Jetson.](https://github.com/dusty-nv/jetson-inference)
- [GitHub - AlexeyAB/darknet: YOLOv4v / Scaled-YOLOv4 - Neural Networks for Object Detection (Windows and Linux version of Darknet )](https://github.com/AlexeyAB/darknet)
- [Train YOLOv4-tiny on Custom Data - Lightning Fast Object Detection](https://blog.roboflow.com/train-yolov4-tiny-on-custom-data-lighting-fast-detection/)

<br>
