# FASTLIO2

# Prerequisites


## 1
``` 
ROS Humble
PCL >= 1.8
Eigen >= 3.3.4
```


## 2
``` bash
mkdir -p fast_lio2/src
cd fast_lio2/src  # cd into a ros2 workspace folder
#fast_lio2
git clone https://github.com/Ericsii/FAST_LIO.git --recursive
#liovx_driver
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
cd livox_ros_driver2
./build.sh humble
#liovx_sdk
git clone https://github.com/Livox-SDK/Livox-SDK2.git
#雷达格式转换
git clone https://github.com/your-repository/rs_converter.git
#lidar_sdk
git clone https://github.com/RoboSense-LiDAR/rslidar_sdk.git
cd rslidar_sdk
git submodule init
git submodule update
#lidar_msg
git clone https://github.com/RoboSense-LiDAR/rslidar_msg.git
#imu_driver

```

## build
``` bash
colcon build --symlink-install --parallel-workers 3 --cmake-args -DROS_EDITION=ROS2 -DDISTRO_ROS=humble

source install/setup.bash
```


# imu标定


## Prerequisites


### 
``` bash
#
git clone https://github.com/DarrenCai/imu_utils_ros2.git
cd ~/<your_imu_utils_ros2_ws>
colcon build 
source install/setup.bash
```


### ceres-solver
``` bash
# ceres
git clone https://github.com/ceres-solver/ceres-solver.git
# CMake
sudo apt-get install cmake
# google-glog + gflags
sudo apt-get install libgoogle-glog-dev libgflags-dev
# Use ATLAS for BLAS & LAPACK
sudo apt-get install libatlas-base-dev
# Eigen3
sudo apt-get install libeigen3-dev
# SuiteSparse (optional)
sudo apt-get install libsuitesparse-dev

cd ~/ceres-solver  
git submodule update --init --recursive

mkdir build
cd build
cmake .. -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF
make -j2
sudo make install
colcon build
```


## 内参


### record
``` bash
#录制一个imu静止时候的数据包
#2h （1h应该也行）
cd ~/<your_imu_utils_ros2_ws>
ros2 bag record <your_imu_topic> 
```


### change launch_xml
``` bash
<param name="imu_topic" value="/djiros/imu"/>
<param name="max_time_min" value="120"/>
```


### 打开3个终端，分别运行utils，数据回放，lidar
``` bash
ros2 launch imu_utils <your_launch_xml>

ros2 bag play <path_to_your_rosbag2> -r 200

ros2 launch rslidar_sdk start.py
```


### 内参参数
``` bash
cd ~/<your imu_utils_ros2 ws>/data
sudo gedit /<your_launch_xml_name>_imu_param.yaml
#以下参数是imu的内参，例：
avg-axis:
    gyr_n: 6.3514356009505279e-03
    gyr_w: 2.3928120640880331e-04
avg-axis:
    acc_n: 3.6027150818967985e-02
    acc_w: 2.0791260493261765e-03
```


## 外参


### Prerequisites
``` bash
git clone https://github.com/morte2025/LiDAR_IMU_Init_ROS2.git
cd ~/fast_lio2
source install/setup.bash
cd ~
cd ~/<your_LiDAR_IMU_Init_ROS2_ws>
colcon build --symlink-install --parallel-workers 3 --cmake-args -DROS_EDITION=ROS2 -DDISTRO_ROS=humble
source install/setup.bash
```


#### ceres-solver
```
外参标定需要的ceres-solver版本与上面内参所需要的不一致，根据报错换版本
```


### Change


#### robosence.launch.py
``` python
#修改后为
node = Node(
    package= "lidar_imu_init",
    
os.path.join(get_package_share_directory("lidar_imu_init"),"config","robosense.yaml")
```


#### robosense.yaml
```
按照rslidar_sdk中的要求修改对应参数

从imu_utils_ros2得到的内参数据修改对应imu内参
```


### 分别运行lidar，imu，外参文件，lidar格式转换包
``` bash
ros2 launch rslidar_sdk start.py
ros2 run imu_driver imu_driver_node
ros2 launch lidar_imu_init robosence.launch.py
#按照提示给雷达一定的激励
#详情见https://github.com/morte2025/LiDAR_IMU_Init_ROS2
```


# Change


## FAST_LIO


### velodyne.yaml
```
#按照rslidar_sdk中yaml修改对应参数，填入内外参
```


### mapping.launch.py
``` python
declare_config_file_cmd = DeclareLaunchArgument(
    'config_file', default_value='velodyne.yaml',
    description='Config file'
```


## rslidar_sdk


### cmakelist.txt
``` txt
#=======================================
# Custom Point Type (XYZI,XYZIRT, XYZIF, XYZIRTF)
#=======================================
set(POINT_TYPE XYZIRT)
```


### cogfig.yaml
``` yaml
common:
  msg_source: 1                         # 0: not use Lidar
                                        # 1: packet message comes from online Lidar
                                        # 2: packet message comes from ROS or ROS2
                                        # 3: packet message comes from Pcap file
  send_packet_ros: false                 # true: Send packets through ROS or ROS2(Used to record packet)
  send_point_cloud_ros: true            # true: Send point cloud through ROS or ROS2
lidar:
  - driver:
      lidar_type: RS16             #  LiDAR type - RS16, RS32, RSBP, RSAIRY, RSHELIOS, RSHELIOS_16P, RS128, RS80, RS48, RSP128, RSP80, RSP48, 
                                   #               RSM1, RSM1_JUMBO, RSM2, RSM3, RSE1, RSMX.
                                   
      msop_port: 4002              #  Msop port of lidar
      difop_port: 8892             #  Difop port of lidar
      imu_port: 0                  #  IMU port of lidar(only for RSAIRY, RSE1), 0 means no imu.
                                   #  If you want to use IMU, please first set ENABLE_IMU_DATA_PARSE to ON in CMakeLists.txt 
      user_layer_bytes: 0          #  Bytes of user layer. thers is no user layer if it is 0         
      tail_layer_bytes: 0          #  Bytes of tail layer. thers is no tail layer if it is 0


      min_distance: 0.2            #  Minimum distance of point cloud
      max_distance: 150            #  Maximum distance of point cloud
      use_lidar_clock: true        #  true--Use the lidar clock as the message timestamp
                                   #  false-- Use the system clock as the timestamp
      dense_points: false          #  true: discard NAN points; false: reserve NAN points
      
      ts_first_point: true         #  true: time-stamp point cloud with the first point; false: with the last point;   
                                   #  these parameters are used from mechanical lidar

      start_angle: 0               #  Start angle of point cloud
      end_angle: 360               #  End angle of point cloud

                                   #  When msg_source is 3, the following parameters will be used
      pcap_repeat: true            #  true: The pcap bag will repeat play   
      pcap_rate: 1.0               #  Rate to read the pcap file
      pcap_path: /home/robosense/lidar.pcap #The path of pcap file

    ros:
      ros_frame_id: rslidar                           #Frame id of packet message and point cloud message
      ros_recv_packet_topic: /rslidar_packets          #Topic used to receive lidar packets from ROS
      ros_send_packet_topic: /rslidar_packets          #Topic used to send lidar packets through ROS
      ros_send_imu_data_topic: /rslidar_imu_data         #Topic used to send imu data through ROS
      ros_send_point_cloud_topic: /rslidar_points      #Topic used to send point cloud through ROS
      ros_queue_length: 100        
```


# RUN
``` bash
source install/setup.bash
#rslidar
ros2 launch rslidar_sdk start.py
#rs_to_velodyne
ros2 launch rs_converter rs_converter.launch.py
#imu
ros2 run imu_driver imu_driver_node
#fast_lio2
ros2 launch fast_lio mapping.launch.py config_file:=velodyne.yaml
```

---
# 附
## Record
``` bash 
ros2 bag record -o ./<data_name>/<your_senor>_$(date +%Y%m%d_%H%M%S) <your_topic> --compression-mode file
```
