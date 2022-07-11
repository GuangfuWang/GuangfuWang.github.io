# Jetson Nano 2GB OpenCV安装教程
## 安装前准备
> 这一部分参考了[博客](https://zhuanlan.zhihu.com/p/392751819)

由于jetson nano为Arm架构，而opencv官方库对其支持不是很完善，因此可以选择自行编译安装Opencv库，目前（2022-07-11）Opencv库最新版为4.6.0版本，我们使用这个最新版本的Opencv进行编译。
在开始编译之前，我们需要安装好Opencv依赖的库，可使用的命令为：

```shell
sudo apt-get install -y build-essential pkg-config cmake \
    libopenblas-dev libeigen3-dev libtbb-dev \
    libavcodec-dev libavformat-dev \
    libgstreamer-plugins-base1.0-dev libgstreamer1.0-dev \
    libswscale-dev libgtk-3-dev libpng-dev libjpeg-dev \
    libcanberra-gtk-module libcanberra-gtk3-module
```
其中这些依赖的作用为：
| 项目 | 库 | 说明 |
| :----: |:-----:|:------:|
|编译系统|build-essential cmake pkg-config|安装了GCC cmake等，Jetson安装的Ubuntu或者Xubuntu其实已经有了GCC之类的，所以Build-Essential其实不会安装啥东西，pkg-config是未来安装包管理器|
|图像库|libpng-dev libipeg-dev|提供各类图像格式的编解码|
|加速计算|libopenblas-dev lineigen3-dev libtbb-dev|这三项是为了SIMD计算及矩阵运算的高效|
|UI库及解码库|libavcodec-dev libavformat-dev libswscale-dev libgstreamer-plugins-base1.0-dev libgstreamer1.0-dev libgtk-3-dev libcanberra-gtk-module libcanberra-gtk3-module|这块主要是FFMPEG库、GSTREAM库、GTK库|

## 下载OpenCV

```shell
sudo mkdir opencv4.6
git clone https://github.com/opencv/opencv.git
sudo unzip opencv-*.zip
cd opencv-*.zip
sudo mkdir build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_VERBOSE_MAKEFILE=ON \
	-DCMAKE_INSTALL_PREFIX=/usr/local \
	-DOPENCV_EXTRA_MODULES_PATH=<path/to/opencv_contrib-4.3.0/modules> -DOPENCV_ENABLE_NONFREE=ON\
	-DBUILD_opencv_python3=ON -DWITH_1394=OFF \
	-DWITH_IPP=ON -DWITH_TBB=ON -DWITH_OPENMP=ON -DWITH_PTHREADS_PF=ON \
	-DBUILD_TESTS=OFF -DBUILD_PERF_TESTS=OFF -DOPENCV_GENERATE_PKGCONFIG=ON \
	-DWITH_CUDA=ON -DENABLE_FAST_MATH=ON -DCUDA_FAST_MATH=ON -D WITH_CUBLAS=ON \
	-DPYTHON3_LIBRARY=$(python3 -c "from distutils.sysconfig import get_config_var;from os.path import dirname,join ; print(join(dirname(get_config_var('LIBPC')),get_config_var('LDLIBRARY')))") \
	-DPYTHON3_NUMPY_INCLUDE_DIRS=$(python3 -c "import numpy; print(numpy.get_include())") \
	-DPYTHON3_PACKAGES_PATH=$(python3 -c "from distutils.sysconfig import get_python_lib; print(get_python_lib())")
```
这些构建指令的含义为：
CMAKE_BUILD_TYPE=Release：不启用调试信息，并进行速度优化。
CMAKE_VERBOSE_MAKEFILE=ON：可开启以便于发现编译中出现的问题。
CMAKE_INSTALL_PREFIX=/usr/local：安装路径。
BUILD_SHARED_LIBS=ON：成共享库（.so），如果置为 OFF 则会生成静态库（.a）
OPENCV_EXTRA_MODULES_PATH=<opencv-contrib 目录>
OPENCV_ENABLE_NONFREE=ON：生成一些受到专利保护的算法
ENABLE_CXX11=ON：支持 C++11 以上的语法和 STL 库。
BUILD_TESTS=OFF，BUILD_PERF_TESTS=OFF：关闭生成后期的 TEST ，缩短生成时间。
OPENCV_GENERATE_PKGCONFIG=ON：建议开启，便于 C++ 程序引用。
WITH_CUDA=ON，ENABLE_FAST_MATH=ON，CUDA_FAST_MATH=ON，WITH_CUBLAS=ON：如果系统正确安装了 CUDA 并希望 OpenCV 启用 CUDA 支持，这四个选项都要打开。
WITH_IPP=ON，WITH_TBB=ON，WITH_OPENMP=ON，WITH_PTHREADS_PF=ON：这四个选项控制 OpenCV 如何进行并发运算，默认都是 ON，但如果有需要生成一个绝对单线程运行的 OpenCV ，请将这几个选项均置为 OFF 。
构建完成后，执行编译及安装流程
```shell
sudo make -j4
sudo make install 
```
## 后处理
由于Opencv安装后并没有安装Python的支持，因此需要手动进行设置。设置方式为受限确定cv2.*.so安装到了哪里，可使用命令：
```shell
find /usr/local/lib/ -type f -name "cv2*.so"
```
在笔者的Jetson Nano2GB上面，该库在
```shell
/usr/local/lib/python3.8/site-packages/cv2/python/cv2.cpython-aarch64-linux-gnu.so
```
下一步则可以在site-packages中创建软连接，应用命令：
```shell
cd <path/to/lib/python3.*/site-packages/cv2/python/cv2.cpython-aarch64-linux.gnu.so>
sudo ln -s cv2.cpython-aarch64-linux.gnu.so cv2.so
```
软连接创建后，测试是否能够使用opencv:
```shell
python3
>>>import cv2; print(cv2.__version__)
```
如果上述命令可以正常运行，则可以打印出Opencv的版本，如果仅在site-packages中可以打印出版本，离开了该文件夹后无法正常使用，那么这是由于该文件夹不再sys.path中，这可以通过将<path/to/site-packages>加入环境变量$PYTHONPATH中，应用命令:
```shell
echo 'export PYTHONPATH="<path/to/site-packages>:$PYTHONPATH"'>> ~/.bashrc
source ~/.bashrc
```
再次测试应当可以正常运行了。