
# 1. 环境搭建
## 1.1 安装 ==Ubuntu24.04LTS==
## 1.2 储存库设置
1. 打开软件与更新(Software & Updates)窗口，确保启动了==universe==存储库
2. 切换aliyun服务器
## 1.3 安装虚拟机与客户机交互插件
``` bash
sudo apt update
sudo apt-get install open-vm-tools
sudo apt-get install open-vm-tools-desktop
```
## 1.4 检查编码格式
``` bash
#检查是否支持 UTF-8
locale
#若不支持
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8
LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```
## 1.5 设置软件源
1. 查询 *raw.githubusercontent.com* IP地址 ==185.199.111.133==
2. 打开*host*文件
``` bash
sudo apt update && sudo apt install gedit -y  
sudo gedit /etc/hosts
#在第4行后添加
#185.199.111.133 raw.githubusercontent.com
#保存并退出
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y

sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```
## 1.6 安装ROS2 jazzy环境
``` bash
sudo apt update
sudo apt install ros-jazzy-desktop
sudo apt install python3-colcon-common-extensions
source /opt/ros/jazzy/setup.bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
```
## 1.7 安装功能包
``` bash
sudo apt update
#显示工具
sudo apt install picocom 
#数据滤波及姿态解算工具
sudo apt install ros-jazzy-imu-filter-madgwick 
#可视化工具
sudo apt install ros-jazzy-rviz-imu-plugin
#编译工具
sudo apt install build-essential cmake
```
 - 安装serial包
	 1. 虚拟机设置  -> 选项 -> 共享文件夹 -> 启用
	 2. 在主机中创建共享文件夹
     3. 浏览器 *https://codeload.github.com/wjwwood/serial/zip/main* 下载到共享文件夹，在虚拟机中解压
``` bash
cd /mnt/hgfs/共享文件夹名
unzip 文件名.zip -d ~/mpu6050_ws/src/
```
## 1.8 环境测试
``` bash
#终端1中，执行完毕，会启动一个绘有小乌龟的窗口
ros2 run turtlesim turtlesim_node
#终端2中，执行完毕，可以在此终端中通过键盘控制乌龟运动
ros2 run turtlesim turtle_teleop_key
```

# 2. 读取MPU6050
## 2.1 硬件连接
![stm32--MPU6050--CH340g]()

## 2.2 stm32读取MPU6050
### 2.2.1 配置STM32CubeMX
1. 新建工程 -> 选 STM32F103C8T6
2. *pinout view* 
	- PB8，PB9 -> GPIO_Output
3. *Pinout & Configuration*
	1. *System Core*
		- RCC -> HSE -> Crystal/Ceramic Resonator
		- Sys -> Debug -> Serial Wire
	2. *Connectivity*  
		- I2C1 -> I2C -> I2C
		- USART1 -> Mode -> Asynchronous
	3. *GPIO*
		- PB8，PB9 
			- GPIO output level -> Low
			- GPIO mode -> Output Push Pull
4. *Clock Configuration*
	- HCLK -> 72
5. *Project Manager*
	1. *Project* 
		- Toolchain/IDE -> MDK-ARM & V5.32
	2. *Code Generator* 
		- 勾选 Copy all the necessary library files 和
	Generate peripheral initialization as a pair of '.c/.h' files per peripheral
6. *Generate Code*
### 2.2.2 驱动及主函数编写

## 2.3 ROS2通过CH340读取MPU6050
1. 创建工作空间与功能包
``` bash
mkdir -p ~/mpu6050_ws/src && cd ~/mpu6050_ws
ros2 pkg create --build-type ament_cmake mpu6050_driver \
       --dependencies rclcpp sensor_msgs std_msgs geometry_msgs serial
```
2. 功能包编写
    1. 编写*mpu6050_driver.cpp* （用于处理mpu6050数据，并发布原始数据的话题）
	2. 修改*CMakeLists.txt* （配置文件）
3. ROS2读取mpu6050
``` bash
#权限设置 运行后重启
sudo usermod -a -G dialout $USER 
#编译工作空间
cd ~/mpu6050_ws && colcon build --symlink-install 
#激活工作空间环境变量并运行你的节点
source install/setup.bash
ros2 run mpu6050_driver mpu6050_driver
#在另一个终端中验证是否成功发布数据
ros2 topic echo /imu/data_raw
```

# 3. 坐标变换及姿态解算
``` bash 
cd ~/mpu6050_ws/src/mpu6050_driver
#创建Launch启动文件
mkdir -p launch 
nano launch/imu.launch.py 
#编写mpu6050_driver,static_transform_publisher,imu-filter-madgwick节点\
#打开CmakeList.txt安装Launch文件
#编译 激活 运行
colcon build --symlink-install
source install/setup.bash
ros2 launch mpu6050_driver imu.launch.py
#另终端验证发布的坐标变换
ros2 run tf2_ros tf2_echo base_link imu_link
#另终端验证姿态解算数据
ros2 topic echo /imu/data
```

# 4. 数据可视化
- 使用Rviz2读取/imu/data（处理后的数据） 话题中的数据
``` bash
colcon build --symlink-install
source install/setup.bash
ros2 launch mpu6050_driver imu.launch.py
#打开Rviz2
rviz2
```
- Global Options -> Fixed Frame -> base_link
- Add -> By Topic -> /imu/data ->  Imu 显示
- 把 “Box” 打钩，能看到随板子晃动的小方块