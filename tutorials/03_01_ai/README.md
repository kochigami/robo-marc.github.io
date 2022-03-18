---
layout: page
---
# 物体検出システム構築チュートリアル

NEDO特別講座 画像処理・AI技術活用コースのチュートリアルとなります。

## 実行環境

* OS: Ubuntu 18.04 LTS
* ROS: Melodic

## 画像認識AI実行環境構築

以下の手順を参照して画像認識AIの実行環境を構築する
* [setup-object-detection-lib.md](https://github.com/robo-marc/robo-marc.github.io/blob/master/tutorials/tutorial202102/sec/setup-object-detection-lib.md)

## ROS環境構築
### ROSのインストール
```
# リポジトリ登録
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654

# aptをアップデート
sudo apt update
sudo apt upgrade

# ROSをインストール
sudo apt install ros-melodic-desktop-full

# rosdepのインストール
sudo apt install python-rosdep
sudo rosdep init
rosdep update

# 環境変数設定
echo "source /opt/ros/melodic/setup.bash" >> ~/.bashrc
source ~/.bashrc

# パッケージインストーラーをインストール
sudo apt install python-rosinstall python-catkin-tools

# ワークスペースを作成
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/
catkin init

# ビルド
catkin build
source devel/setup.bash
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
```

## 物体検出システムを構築する

### ノード構成
* uvc_camera_node : カメラから画像を取得する
* web_video_server_node : システム内を流れるImage topicを表示する
* preprocess_node : 画像の前処理を行う
* object_detector_yolo_node : YOLOv4-tinyで物体検出をする
* annotation_node : 物体検出の結果を画像に描画する

### メッセージ定義
* ObjectDetectionResult : 物体検出結果のメッセージ。
物体検出結果はカスタムメッセージを自分で定義する

### uvc_camera_nodeとweb_video_serverをインストールする
```
sudo apt install ros-melodic-web-video-server
sudo apt install ros-melodic-uvc-camera
```

### カスタムメッセージを定義する
```
cd ~/catkin_ws/src
catkin_create_pkg object_detector_msg rospy roscpp std_msgs
cd object_detector_msg
mkdir msg
```

object_detector_msg/msgディレクトリに以下のファイルを格納します。
* DetectedObject.msg
* ObjectDetectionResult.msg

CMakeLists.txtを編集する
* object_detector_msg/CMakeLists.txt
```
find_package(catkin REQUIRED COMPONENTS
  roscpp
  rospy
  std_msgs
  message_generation
)
...
add_message_files(
  FILES
  DetectedObject.msg
  ObjectDetectionResult.msg
)
...
generate_messages(
  DEPENDENCIES
  std_msgs
)
...
catkin_package(
#  INCLUDE_DIRS include
#  LIBRARIES object_detector_msg
  CATKIN_DEPENDS roscpp rospy std_msgs
#  DEPENDS system_lib
)
```

package.xmlを編集する
* object_detector_msg/package.xml
```
# 以下の二つを追記する
<build_depend>message_generation</build_depend>
<exec_depend>message_runtime</exec_depend>
```

ビルドする
```
catkin build
. ~/catkin_ws/devel/setup.bash
rosmsg show object_detector_msg/DetectedObject
rosmsg show object_detector_msg/ObjectDetectionResult
```

### 物体検出パッケージを作成する
```
cd ~/catkin_ws/src
catkin_create_pkg object_detector rospy roscpp std_msgs
```

YOLOの実装であるpytorch-YOLOv4をlibに配置する
```
cd ~/catkin_ws/src/object_detector
mkdir lib
cd lib
git clone https://github.com/Tianxiaomo/pytorch-YOLOv4
# 重みをダウンロード
mkdir weight
cd weight
wget https://github.com/AlexeyAB/darknet/releases/download/darknet_yolo_v4_pre/yolov4-tiny.weights
```

ノードとlaunchファイルを配置する
* object_detector/script/object_detector_yolo.py
* object_detector/script/preprocess.py
* object_detector/script/annotation.py
* object_detector/launch/object_detector_yolo.launch

CMakeList.txtを編集する
* object_detector/CMakeLists.txt
```
find_package(catkin REQUIRED COMPONENTS
  roscpp
  rospy
  std_msgs
  object_detector_msg
)
```

package.xmlを編集する
* object_detector/package.xml
```
<buildtool_depend>object_detector_msg</buildtool_depend>
<exec_depend>object_detector_msg</exec_depend>
```

実行する
```
# PYTHONPATHにYOLOの実装へのパスを追加する
export PYTHONPATH=~/catkin_ws/src/object_detector/lib/pytorch-YOLOv4:$PYTHONPATH

# 各ノードに実行権限を設定する
cd ~/catkin_ws/src/object_detector/scripts
chmod +x object_detector_yolo.py
chmod +x preprocess.py
chmod +x annotation.py

# rospkgをインストールする
pip3 install rospkg

catkin build
source ~/catkin_ws/devel/setup.bash
roslaunch object_detector object_detector_yolo.launch
```
```localhost:8080```にブラウザでアクセスすることで、物体検出の結果を確認することができます。

## 物体検出モデルをSSDに入れ替える
ノードとlaunchファイルを配置配置する
* object_detector/scripts/object_detector_ssd.py
* object_detector/launch/object_detector_ssd.launch

SSDの実装であるSSDをlibに配置する
```
cd ~/catkin_ws/src/object_detector/lib
git clone https://github.com/lufficc/SSD
cd SSD
mkdir weight
cd weight
wget https://github.com/lufficc/SSD/releases/download/1.2/mobilenet_v2_ssd320_voc0712_v2.pth
```

実行する
```
# PYTHONPATHにSSDの実装へのパスを追加する
export PYTHONPATH=~/catkin_ws/src/object_detector/lib/SSD:$PYTHONPATH

# launchファイルを実行する
roslaunch object_detector object_detector_ssd.launch
```

YOLOの時と同様に```localhost:8080```にブラウザでアクセスすることで、物体検出の結果を確認することができます。

## 物体追跡機能を追加する

物体追跡のライブラリをインストールする
```
pip3 install norfair
```

ノードとlaunchファイルを配置する
* object_detector/scripts/object_tracking.py
* object_detector/launch/object_tracking.launch

実行する
```
# PYTHONPATHにYOLOの実装へのパスを追加する
export PYTHONPATH=~/catkin_ws/src/object_detector/lib/pytorch-YOLOv4:$PYTHONPATH

# launchファイルを実行
roslaunch object_detector object_tracking.launch
```

```localhost:8080```にブラウザでアクセスすることで、物体検出の結果を確認することができます。
