# Linux-Project

\# 步骤



\## 环境搭建, ubuntu24.04LTS, jazzy

1\. 储存库设置

&nbsp;	1. 打开软件与更新(Software \& Updates)窗口，确保启动了==universe==存储库

&nbsp;	2. 切换aliyun服务器

2\. 安装虚拟机与客户机交互插件

``` bash

sudo apt update

sudo apt-get install open-vm-tools

sudo apt-get install open-vm-tools-desktop

```

3\. 检查编码格式

``` bash

\#检查是否支持 UTF-8

locale

\#若不支持

sudo apt update \&\& sudo apt install locales

sudo locale-gen en\_US en\_US.UTF-8

sudo update-locale LC\_ALL=en\_US.UTF-8

LANG=en\_US.UTF-8

export LANG=en\_US.UTF-8

```

4\. 设置软件源

``` bash

sudo apt install software-properties-common

sudo add-apt-repository universe

sudo apt update \&\& sudo apt install curl -y

sudo curl -sSLhttps://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

```

\- 正常无输出，报错则

&nbsp;	1. 查询 \*raw.githubusercontent.com\* IP地址 ==185.199.111.133==

&nbsp;	2. 安装gedit编辑器

&nbsp;	`sudo apt-get gedit install`

&nbsp;	3. 打开\*host\*文件

&nbsp;	`sudo gedit /etc/hosts`

&nbsp;	4. 在写有IP+域名的代码后加上 对应的IP地址与域名，保存

&nbsp;	`185.199.111.133 raw.githubusercontent.com`

&nbsp;	5. 重复步骤 \*设置软件源\*

``` bash

\#将存储库添加到源列表

echo "deb \[arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archivekeyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release \&\& echo $UBUNTU\_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

```

5\. 安装ROS2

`sudo apt install ros-jazzy-desktop`

6\. ROS2环境搭建

``` bash

source /opt/ros/jazzy/setup.bash

echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc

sudo apt update \&\& sudo apt install python3-colcon-common-extensions

```

7\. 环境测试

``` bash

\#终端1中，执行完毕，会启动一个绘有小乌龟的窗口

ros2 run turtlesim turtlesim\_node

\#终端2中，执行完毕，可以在此终端中通过键盘控制乌龟运动

ros2 run turtlesim turtle\_teleop\_key

```



\## stm32读取MPU6050  

1\. 配置STM32CubeMX

&nbsp;	1. 新建工程 -> 选 STM32F103C8T6

&nbsp;	2. \*Pinout \& Configuration\*

&nbsp;		1. \*System Core\* -> RCC -> HSE -> Crystal/Ceramic Resonator

&nbsp;					 -> Sys -> Debug -> Serial Wire

&nbsp;		2. \*Connectivity\*  -> I2C1 -> I2C -> I2C

&nbsp;				      -> USART1 -> Mode -> Asynchronous

&nbsp;	3. \*Clock Configuration\*

&nbsp;		 HCLK -> 72

&nbsp;	4. \*Project Manager\*

&nbsp;		1. \*Project\* -> Toolchain/IDE -> MDK-ARM \& V5.32

&nbsp;		2. \*Code Generator\* -> 勾选 Copy all the necessary library files \& Generate peripheral initialization as a pair of '.c/.h' files per peripheral

&nbsp;	5. \*Generate Code\*

&nbsp;	> 可以根据自身要求添加如Oled Lcd 等

2\. 编写驱动及主函数

3\. stm32端验证（如有）



\## ROS2通过CH340读取MPU6050

1\. 创建工作空间与功能包

``` bash

mkdir -p ~/mpu6050\_ws/src \&\& cd ~/mpu6050\_ws

ros2 pkg create --build-type ament\_cmake mpu6050\_driver \\

&nbsp;      --dependencies rclcpp sensor\_msgs std\_msgs geometry\_msgs serial

```

2\. 安装serial包

&nbsp;	 1. 虚拟机设置  -> 选项 -> 共享文件夹 -> 启用

&nbsp;	 2. 在主机中创建共享文件夹

&nbsp;    3. 浏览器 \*https://codeload.github.com/wjwwood/serial/zip/main\* 下载到共享文件夹，在虚拟机中解压

&nbsp;    ``` bash

&nbsp;    cd /mnt/hgfs/共享文件夹名

&nbsp;    unzip 文件名.zip -d ~/mpu6050\_ws/src/

&nbsp;    ```

3\. 功能包编写

&nbsp;   1. 编写\*mpu6050\_driver.cpp\* （用于处理mpu6050数据，并发布原始数据的话题）

&nbsp;	2. 修改\*CMakeLists.txt\* （配置文件）

4\. ROS2读取mpu6050

&nbsp;	1. 确认硬件连接

&nbsp;	2. 开启权限并使用串口工具（picocom）显示数据

``` bash

\#权限设置 运行后重启

sudo usermod -a -G dialout $USER 

\#显示工具

sudo apt update \&\& sudo apt install picocom 

\#编译工作空间

cd ~/mpu6050\_ws \&\& colcon build --symlink-install 

\#激活工作空间环境变量并运行你的节点

source install/setup.bash

ros2 run mpu6050\_driver mpu6050\_driver

\#在另一个终端中验证是否成功发布数据

ros2 topic echo /imu/data\_raw

```



\## 静态TF发布

\- TF是一个用于处理坐标系之间变换关系的工具，可以让不同传感器或机器人部件的数据在统一的坐标框架下进行融合和显示

\- 将IMU坐标系（imu\_link）与机器人的基坐标系（base\_link）关联起来，确保Rviz2能够正确显示IMU数据

1\. 创建\*Launch\*启动文件

``` bash 

cd ~/mpu6050\_ws/src/mpu6050\_driver

mkdir -p launch 

nano launch/imu.launch.py 

\#编写mpu6050\_driver和static\_transform\_publisher节点，保存并退出

```

2\. 打开\*CmakeList.txt\*安装\*Launch\*文件

3\. 验证

``` bash

\#编译 激活 运行

cd ~/mpu6050\_ws \&\& colcon build --symlink-install

source install/setup.bash

ros2 launch mpu6050\_driver imu.launch.py

\#另终端验证发布的坐标变换

ros2 run tf2\_ros tf2\_echo base\_link imu\_link

```



\## 数据滤波及姿态解算

\- 解算四元数确定姿态，mpu6050本身不带有四元数

``` bash

sudo apt install ros-jazzy-imu-filter-madgwick 

nano launch/imu.launch.py 

\#编写imu-filter-madgwick节点

```



\## Rviz2 + rviz\_imu\_plugin  IMU数据可视化

\- 使用Rviz2读取/imu/data（处理后的数据） 话题中的数据

``` bash

sudo apt install ros-jazzy-rviz-imu-plugin

\#运行启动文件

ros2 launch mpu6050\_driver imu.launch.py

\#打开Rviz2

rviz2

```

\- Global Options -> Fixed Frame -> base\_link

\- Add -> By Topic -> /imu/data ->  Imu 显示

\- 把 “Box” 打钩，就能看到随板子晃动的小方块

