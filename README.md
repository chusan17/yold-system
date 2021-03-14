# YOLD - Drone Detector -
You Only Look Drone with Jetson Nano

![ian-usher-JPAfSd_acI8-unsplash](https://user-images.githubusercontent.com/79794586/109420853-e1704580-7a17-11eb-8e66-47ff49b930a5.jpg)

## Introduction

Welcome to You Only Look Drone (YOLD) repo!
<br>
YOLD is a drone detector based on `YOLOv4 tiny`.
<br>
100+ drone pictures are used for re-traning `YOLOv4 tiny` to create a new object detection network which detects only a drone.
<br>

Furthermore, re-trained `YOLOv4 tiny` is put on Jetson Nano 2GB to perform Real-Time Drone Detection with webcam.
<br>

## Configuration

In this repo, [Colaboratory file](https://github.com/chusan17/yold-system/blob/main/train_YOLOv4_tiny.ipynb) is stored.
<br>
This file shows how to re-train `YOLOv4 tiny` using your original dataset in Colaboratory.
<br>
YOLO needs unique dataset format, so you need to create dataset for YOLO with [this labeling tool](https://github.com/tzutalin/labelImg).
<br>

**Inference is performed only in Jetson which means no necessary Internet connection during Real-Time detection.**
<br>
※ Notification to your LINE account introduced below needs WiFi for API call.
<br>

### Hardwares
- Jetson Nano 2GB
- USB-C power supply (5V 3A)
- Webcam (Logicool C525n)
- WiFi<br>

## Re-train YOLOv4 tiny with your original dataset

Open your Colaboratory and enable GPU mode, and let's get `YOLOv4 (darknet)`.
```
!git clone https://github.com/AlexeyAB/darknet
```
Rewrite `makefile` to use GPU and OpenCV.
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
It takes lots of time...
<br>
Once finished, Download pre-trained weights from YOLOv4 repo.
```
!wget https://github.com/AlexeyAB/darknet/releases/download/darknet_yolo_v4_pre/yolov4-tiny.conv.29
```
Mount your own Google Drive on Colaboratory to read datasets you made.
<br>
Actually lots of files described below are needed for training YOLO.
<br>
>All of files are stored in this repo except your own datasets and train/test.txt, please download them and put them into same directory.
<br>

- `backup`         empty directory where generated weights are stored
- `obj`            directory for training dataset
- `test`           directory for test dataset
- `train.txt`      describe PATH for training images
- `test.txt`       describe PATH for test images
- `obj.data.txt`
- `obj.names`
- `yolov4-tiny-custom.cfg`
- `yolov4-tiny.conv.29`

If you have prepared all of files, let's train YOLOv4 tiny with your datasets.
```
!./darknet detector train /path/to/obj.data.txt /path/to/yolov4-tiny-custom.cfg /path/to/yolov4-tiny.conv.29 -dont_show -map
```
Once training finished, `yolov4-tiny-custom_best.weights` would be generated in `backup`.
<br>
This weight and `obj.data.txt` `obj.names` `yolov4-tiny-custom.cfg` should be put into Jetson Nano 2GB to do inference at local environment.
<br>
## Setting up Jetson Nano 2GB

You follow below videos to set up Jetson Nano, almost all of things are needed for Real-Time detection will be installed on your Jetson.
<br>
If not enough, Download things you need from your Jetson terminal separately.
<br>

- [Jetson AI Fundamentals - S1E1 - First Time Setup with JetPack](https://youtu.be/uvU8AXY1170)
- [Jetson AI Fundamentals - S3E1 - Hello AI World Setup](https://youtu.be/QXIwdsyK7Rw)
<br>

All procedures in these video have been done, open up your terminal, move to directory where you want to install `YOLOv4 (darknet)`, then get YOLO from Github repo.
```
$ git clone https://github.com/AlexeyAB/darknet
```
Change `makefile` according to [this article](https://qiita.com/tayutayufk/items/3d715184e0a7cefa5e9a), and then do `make` it.
```
$ make
```
That's it. Ready for YOLD!
<br>
Let's activate Real-Time Drone Detection with Jetson Nano and Webcam.
```
$ ./darknet detector demo /path/to/obj.data.txt /path/to/yolov4-tiny-custom.cfg /path/to/yolov4-tiny-custom_best.weights -thresh 0.1 >darknet.log
```
Webcam would be activated without extra setting.
<br>
`thresh 0.1` means that results with 10% or more confidence will be displayed as being detected.
<br>
And `>darknet.log` is log of inference results which is used for LINE notification.
<br>
## LINE notification with YOLD

Let's install `swatch` on your Jetson, open your terminal,
```
$ sudo apt-get install swatch
```
Make `swatch.conf` by `nano` or `vim`.
>※ If Jetson doesn't have nano or vim, install them from terminal.<br>
※ Access rights should be granted if necessary.
```
$ vim swatch.conf
```
Write below script on `swatch.conf`.
```txt:swatch.conf
watchfor /drone/
	exec /path/to/LINEnotify.sh
	threshold track_by=/drone/,type=both,count=3,seconds=10
```
Make `LINEnotify.sh` file for API call. 
```
$ vim LINEnotify.sh
```
```LINEnotify.sh
curl -g -X POST -H "Authorization: Bearer ACCESS_TOKEN" -F "message=WRITE YOUR MESSAGE HERE" https://notify-api.line.me/api/notify

echo
exit 0
```
Issued token should be described in `ACCESS_TOKEN`.
<br>
If you want to know how to issue token, please refer to [this article](https://qiita.com/iitenkida7/items/576a8226ba6584864d95).
<br>

Run swatchdog during Real-Time Drone Detection,
```
$ swatchdog --config-file=/path/to/swatch.conf --tail-file=/path/to/darknet.log
```
You can get notification from Jetson via LINE notify!
<br>
## Conclusion

We tried running YOLOv4 tiny trained by original dataset with Jetson Nano 2GB and send a message to your phone when a dorne is detected.
<br>
With this configuration, about `5 FPS` comes out.
<br>
Even if you flutter a book next to a flying toy drone, it will not be falsely detected. Even though it is `tiny` with about only 100 pics, it is quite excellent.
<br>

![スクリーンショット 2021-03-10 233930](https://user-images.githubusercontent.com/79794586/110646185-02941b80-81fa-11eb-9633-ba24db5e7744.png)
<br>

Now it's safe even if a drone breaks into the room!
<br>
Thanks!
<br>
## Reference
- [JETSON AI COURSES AND CERTIFICATION](https://developer.nvidia.com/ja-jp/embedded/learn/jetson-ai-certification-programs)
- [LINE Notify](https://notify-bot.line.me/ja/)
- [GitHub - dusty-nv/jetson-inference: Hello AI World guide to deploying deep-learning inference networks and deep vision primitives with TensorRT and NVIDIA Jetson.](https://github.com/dusty-nv/jetson-inference)
- [GitHub - AlexeyAB/darknet: YOLOv4v / Scaled-YOLOv4 - Neural Networks for Object Detection (Windows and Linux version of Darknet )](https://github.com/AlexeyAB/darknet)
- [Train YOLOv4-tiny on Custom Data - Lightning Fast Object Detection](https://blog.roboflow.com/train-yolov4-tiny-on-custom-data-lighting-fast-detection/)

<br>
