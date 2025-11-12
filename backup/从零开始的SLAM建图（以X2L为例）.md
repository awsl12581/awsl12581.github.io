[从零开始的SLAM建图（以X2L为例）.md](https://github.com/user-attachments/files/22318013/SLAM.X2L.md)
# 前言
本次的探索是从激光雷达拆开上电开始，到SLAM图建立的过程，中间涉及的专业性知识，比如订阅与发布之类的，可以在B站搜索ROS进行学习



# 正文
## 前提条件
> Ubuntu20.04（不要使用21.04，这是个大坑）
>
> YDLIDAR X2 激光雷达（包括转接器，usb转type-c线，以及雷达本体）
>
> 一个善于思考的大脑和一双手
>

`**资料引用**`

> [https://ydlidar.cn/service_support/download.html](https://ydlidar.cn/service_support/download.html)
>
> 官方资料支持
>
> [https://blog.csdn.net/zhu751191958/category_7380985.html](https://blog.csdn.net/zhu751191958/category_7380985.html)
>
> 感谢这位大佬的整理
>
> [https://blog.csdn.net/r1141207831/article/details/106327172](https://blog.csdn.net/r1141207831/article/details/106327172)
>
> 20.04版本的ROS（包括rviz）安装
>

## 在Ubuntu20.04中安装ROS Noetic（包括可视化的rviz）
在20.04版本下，最最最推荐安装Noetic版本

### <font style="color:rgb(77, 77, 77);">1、添加 sources.list（设置你的电脑可以从 packages.ros.org 接收软件.）</font>
```shell
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'

```

### 2、添加 keys
```shell
sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654

```

### 3、安装,首先，确保你的Debian软件包索引是最新的：
```shell
sudo apt update
```

### 4、安装桌面完整版 : 包含ROS、[rqt](http://wiki.ros.org/rqt)、[rviz](http://wiki.ros.org/rviz)、机器人通用库、2D/3D 模拟器、导航以及2D/3D感知
```shell
sudo apt install ros-noetic-desktop-full
```

### 5、您必须在使用ROS的每个bash终端中获取此脚本的源代码。
```shell
source /opt/ros/noetic/setup.bash
```

### 6、环境配置
```shell
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

<font style="color:rgb(77, 77, 77);">至此已经在Ubuntu20.04的系统中完整安装ROS  Noetic。</font>

### ROS的启动和使用
在一个控制台下使用

```shell
roscore
```

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641300051879-1bfdfa80-06fb-4acd-8b97-eb9c798df51a.png)

<font style="color:rgb(77, 77, 77);">启动整个ROS服务，其子服务在本控制台不关闭的情况下重新打开新的控制台使用</font>

```shell
rosrun rviz rviz
```

打开rviz

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641300128965-5fe6fb59-fd1e-4af8-a858-07d046e7c98d.png)

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641300166742-140d512d-3a27-4ca5-ac88-67eb1506d5f2.png)

至此，我们需要的软件完成

## 在ubuntu20.04下安装X2L的SDK以及驱动包
**<font style="color:#2F54EB;background-color:#FADB14;">注意，在这里，我们的根目录是cd ~目录</font>**

### 安装官方SDK
```git
$ git clone https://github.com/YDLIDAR/YDLidar-SDK.git
$ cd YDLidar-SDK
$ cmake .
$ make
$ sudo make install
```

这里官方说是build，然而并没有这个文件夹，所以我修改了一下

### ROS驱动包安装
**<font style="color:#2F54EB;background-color:#FADB14;">依旧是在cd ~目录</font>**

#### 1) 克隆github的ydlidar_ros_driver软件包：
```git
$ git clone https://github.com/YDLIDAR/ydlidar_ros_driver.git ydlidar_ws/src/ydlidar_ros_driver
```

这里其实是创建了一个目录`~/ydlidar_ws/src/`这也是我们项目的目录（工作空间）

#### 2) 构建ydlidar_ros_driver软件包：
```git
$ cd ydlidar_ws
$ catkin_make
```

#### 3) 软件包环境设置：(这个步骤后面也会设置)
```git
$ source ./devel/setup.sh
```

注意：添加永久工作区环境变量。如果每次启动新的shell时ROS环境变量自动添加到您的bash会话中，将很方便：

```bash
$ echo "source ~/ydlidar_ws/devel/setup.bash" >> ~/.bashrc
$ source ~/.bashrc
```

#### 4) 确认要确认已设置您的软件包路径，请回显该ROS_PACKAGE_PATH变量。
```bash
$ echo $ROS_PACKAGE_PATH
```

`**您应该看到类似以下内容：/home/tony/ydlidar_ws/src:/opt/ros/melodic/share**`



#### 创建串行端口别名[可选]
```bash
$ chmod 0777 src/ydlidar_ros_driver/startup/*
$ sudo sh src/ydlidar_ros_driver/startup/initenv.sh
```

#### 现在，我们运行一下
使用启动文件运行ydlidar_ros_driver，例子如下：

```bash
$ roslaunch ydlidar_ros_driver X2.launch
```

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641300855235-1b629a76-d48c-4584-8a69-63971ff9e4bf.png)

雷达被顺利驱动，我们现在将它展示到RVIZ上。

修改一处地方，文件默认以G4雷达为例，若使用其它型号雷达，需将lidar_view.launch文件中的lidar.launch改为对应的**.launch文件。（如使用X2雷达，需改成X2.launch）

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641300969617-3012525b-c517-4436-a143-b918538ae499.png)

之后

```bash
$ roslaunch ydlidar_ros_driver lidar_view.launch
```

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641301006926-ddc331f5-6274-4abd-8f59-3f29022f369f.png)

注意！！！

1）在运行`roslaunch ydlidar_ros_driver lidar_view.launch` 这句代码时，应该另外打开控制台。

2）请确保在运行`roslaunch ydlidar_ros_driver lidar_view.launch` 之前`roslaunch ydlidar_ros_driver X2.launch` 已经运行，这句话的意思是，这两句指令不能同时运行，一次只能使用一个。

### 绘制SLAM图(使用hector_slam)建图
现在，我们的`~/ydlidar_ws/src/`以其其资源驱动应该已经存在了

#### 在工作空间目录下下载gmapping与laser_scan_matcher以及依赖csm的源码包：
```bash
cd ~/ydlidar_ws/src
git clone https://github.com/tu-darmstadt-ros-pkg/hector_slam
cd ~/ydlidar_ws
catkin_make
```

#### 修改发布内容，这部份涉及专业内容，需要了解ROS的发布
```bash
cd ~/ydlidar_ws/src/hector_slam/hector_slam_launch/launch/
sudo vim All_nodes.launch
```

编写以下代码

```xml
<?xml version="1.0"?>
<launch>
 <include file="$(find ydlidar_ros_driver)/launch/X2.launch" />
 <node pkg="tf" type="static_transform_publisher"
	 name="map_to_odom" args="0.0 0.0 0.0 0 0 0.0 /odom /base_link 40"/>

<node pkg="tf" type="static_transform_publisher" 
	name="base_frame_laser" args="0 0 0 0 0 0 /base_link /laser_frame 40" />
 <!--<node pkg="rviz" type="rviz" name="rviz"args="-d $(find hector_slam_launch)/rviz_cfg/mapping_demo.rviz"/>-->
  <include file="$(find hector_mapping)/launch/mapping_default.launch" />
  <node pkg="rviz" type="rviz" name="rviz" args="-d $(find ydlidar_ros_driver)/launch/lidar.rviz" />
  <include file="$(find hector_geotiff_launch)/launch/geotiff_mapper.launch" />
</launch>
```

`$(find ydlidar_ros_driver)`就是找激光雷达的驱动文件，

` <include file="$(find ydlidar_ros_driver)/launch/X2.launch" />`是用来找驱动文件的，

根据自己的情况来



#### <font style="color:#000000;">修改</font><font style="color:#000000;">mapping</font>
<font style="color:#000000;">进入hector_mapping/launch文件夹下</font>

```bash
cd ~/ydlidar_ws/src/hector_slam/hector_mapping/launch
```

<font style="color:#000000;">备份原文件为mapping_default_backups.launch</font>

```bash
cp mapping_default.launch  ./mapping_default_backups.launch
```

<font style="color:#000000;">打开mapping_default.launch</font>

<font style="color:#000000;">修改为</font>

```xml
<?xml version="1.0"?>
<launch> 
  <arg name="tf_map_scanmatch_transform_frame_name" default="/scanmatcher_frame" /> 
  <arg name="base_frame" default="base_link" /> 
  <arg name="odom_frame" default="base_link" /> 
  <arg name="pub_map_odom_transform" default="true" /> 
  <arg name="scan_subscriber_queue_size" default="5" /> 
  <arg name="scan_topic" default="scan" /> 
  <arg name="map_size" default="2048" /> 
  <node pkg="hector_mapping" type="hector_mapping" name="hector_mapping" output="screen"> 
    <!-- Frame names --> 
    <param name="map_frame" value="map" /> 
    <param name="base_frame" value="$(arg base_frame)" /> 
    <param name="odom_frame" value="$(arg base_frame)" /> 
    <!-- Tf use --> 
    <param name="use_tf_scan_transformation" value="true" /> 
    <param name="use_tf_pose_start_estimate" value="false" /> 
    <param name="pub_map_odom_transform" value="$(arg pub_map_odom_transform)" /> 
    <!-- Map size / start point --> 
    <param name="map_resolution" value="0.050" /> 
    <param name="map_size" value="$(arg map_size)" /> 
    <param name="map_start_x" value="0.5" /> 
    <param name="map_start_y" value="0.5" /> 
    <param name="map_multi_res_levels" value="2" />     
    <!-- Map update parameters --> 
    <param name="update_factor_free" value="0.4" /> 
    <param name="update_factor_occupied" value="0.7" />     
    <param name="map_update_distance_thresh" value="0.2" /> 
    <param name="map_update_angle_thresh" value="0.9" /> 
    <param name="laser_z_min_value" value = "-1.0" /> 
    <param name="laser_z_max_value" value = "1.0" />     
    <!-- Advertising config --> 
    <param name="advertise_map_service" value="true" /> 
    <param name="scan_subscriber_queue_size" value="$(arg scan_subscriber_queue_size)" /> 
    <param name="scan_topic" value="$(arg scan_topic)" /> 
    <!-- Debug parameters --> 
    <!-- 
      <param name="output_timing" value="false" /> 
      <param name="pub_drawings" value="true" /> 
      <param name="pub_debug_output" value="true" /> 
    --> 
    <param name="tf_map_scanmatch_transform_frame_name" value="$(arg tf_map_scanmatch_transform_frame_name)" /> 
  </node> 
  <!--<node pkg="tf" type="static_transform_publisher" name="map_nav_broadcaster" args="0 0 0 0 0 0 map nav 100" />--> 
</launch>
```

#### 重新编译
```bash
cd ~/ydlidar_ws
catkin_make
```

#### 添加环境配置
```bash
source ~/ydlidar_ws/devel/setup.bash
```

#### 运行
确保ROS核心开启

另起控制台

`roslaunch hector_slam_launch All_nodes.launch`

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641302444784-5d74c5f1-6e17-4955-afa4-24445d0380de.png)

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641302462713-3a610345-f6fc-46d0-aa7a-6bc9518be6ce.png)![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641302477490-218a3096-179f-4b65-b090-cd6ff532f403.png)



点击add，选中map

![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641302501832-33243a8d-3b9b-4f5d-b34d-e2bf65061ba3.png)



![](https://cdn.nlark.com/yuque/0/2022/png/21764230/1641302536266-182def50-1a07-422f-b4d0-73ce1cf48297.png)

至此，阶段性数据完成。

