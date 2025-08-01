xhost +
docker run -d \
  --gpus all \
  --network=host \
  -e DISPLAY=:1 \
  -v /tmp/.X11-unix/:/tmp/.X11-unix:rw \
  -v ~/projects:/root/projects \
  --name ligo_dev \
  tiryoh/ros-desktop-vnc:noetic \
  bash

 # 1. 停止所有可能在运行的旧容器
docker stop ligo_dev ligo_dev_backup

# 2. 删除所有旧容器
docker rm ligo_dev ligo_dev_backup

# 3. (关键一步) 删除主机上的共享文件夹，确保代码也是全新的
rm -rf ~/projects

# 4. 重新创建干净的共享文件夹
mkdir -p ~/projects

docker ps -a
docker exec -it ligo_dev bash

apt-get update
apt-get install -y git nano wget \
                   libeigen3-dev libpcl-dev libboost-all-dev libopencv-dev \
                   libmetis-dev libfmt-dev \
                   libgoogle-glog-dev libgflags-dev libatlas-base-dev libsuitesparse-dev
apt-get install -y ros-noetic-pcl-ros
# --- 编译 GTSAM ---
cd /root/
git clone https://github.com/borglab/gtsam.git -b 4.2.0
cd gtsam && mkdir build && cd build
cmake .. && make -j$(nproc) && make install

# --- 编译 Sophus ---
cd /root/
git clone https://github.com/strasdat/Sophus.git
cd Sophus && git checkout 1.22.10
mkdir build && cd build
cmake .. -DBUILD_TESTS=OFF && make -j$(nproc) && make install

# --- 编译 Ceres ---
cd /root/
git clone https://github.com/ceres-solver/ceres-solver.git -b 1.14.0
cd ceres-solver && mkdir build && cd build
cmake .. -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF && make -j$(nproc) && make install

# 1. 确保所有编译依赖都已安装
apt-get update && apt-get install -y libgoogle-glog-dev libgflags-dev libatlas-base-dev libsuitesparse-dev

# 2. 从GitHub克隆Ceres 2.1.0版本
cd /root/
git clone https://github.com/ceres-solver/ceres-solver.git -b 2.1.0
cd ceres-solver && mkdir build && cd build
cmake .. -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF && make -j$(nproc) && make install

# --- 编译 Livox-SDK ---
# (我们将把它直接克隆到工作区里，然后编译)


# 创建工作区
mkdir -p /root/projects/ligo_ws/src
cd /root/projects/ligo_ws/src

# 克隆所有需要的包
git clone https://github.com/HKUST-Aerial-Robotics/gnss_comm.git
git clone https://github.com/Joanna-HE/LIGO.git
git clone https://github.com/Livox-SDK/livox_ros_driver.git
git clone https://github.com/Livox-SDK/Livox-SDK.git


cd /root/projects/ligo_ws/src/Livox-SDK/build
cmake .. && make -j$(nproc) && make install

# 回到工作区根目录
cd /root/projects/ligo_ws/

激活ros1
. /opt/ros/noetic/setup.bash
catkin_make
rm -rf build/ devel/


echo "source /opt/ros/noetic/setup.bash" >> /root/.bashrc
echo "source /root/projects/ligo_ws/devel/setup.bash" >> /root/.bashrc
source /root/.bashrc


# (推荐) 先更新一下软件包列表
apt-get update

# 安装libdw的开发包
apt-get install -y libdw-dev

ldconfig

docker exec -it ligo_dev bash
cd /root/projects/ligo_ws

xhost +
roslaunch ligo mapping_velody16.launch
roslaunch ligo mapping_avia.launch

source /root/.bashrc
cd /root/projects/
rosbag play bridge.bag --clock
rosbag play deg1.bag --clock
rosbag play out.bag --clock
# 下载并准备好数据包后，按顺序播放
# rosbag play TST_M8N_... .bag
# rosbag play 2019-04-28-....bag --clock
cd /root/projects/
rosbag play TST_M8N_2024-03-06-20-37-17.bag
rosbag play 2019-04-28-20-58-02.bag --clock
rosbag play bridge.bag --clock
