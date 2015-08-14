#Opencv2.4.10安装配置
参考文档：http://blog.csdn.net/ax_bx_cx_dx/article/details/37561643
http://zhongcong386.blog.163.com/blog/static/134727804201302341638825/
http://docs.opencv.org/2.4.4/doc/tutorials/introduction/desktop_java/java_dev_intro.html 如何编译opencv使可以支持java
http://docs.opencv.org/trunk/doc/tutorials/introduction/java_eclipse/java_eclipse.html   如何在eclipse搭建编写java opencv程序
http://stackoverflow.com/questions/17636166/cmake-find-packagejni-not-work-in-ubuntu-12-04-amd64 编译时候出错的解决方案

###步骤
1.安装编译工具包

```
$ sudo apt-get install build-essential cmake
```

2.配置ffmpeg库

2.1 安装依赖


> $ sudo apt-get install make automake g++ bzip2 python unzip patch subversion ruby build-essential git-core 
checkinstall yasm texi2html libfaac-dev libmp3lame-dev libopencore-amrnb-dev libopencore-amrwb-dev libsdl1.2-dev
 libtheora-dev libvdpau-dev libvorbis-dev libvpx-dev


 2.2，安装yasm


 - git clone git://github.com/yasm/yasm.git
 - cd yasm
 - ./autogen.sh
 - ./configure
 - make
 - make install
 
2.3 安装x264编码库
 
 - cd /my/path/where/i/keep/compiled/stuff
 - git clone git://git.videolan.org/x264.git
 - cd x264
 - ./configure --enable-static --enable-shared
 - make
 - make install
 - ldconfig

2.4，配置ffmpeg库

 - git clone git://source.ffmpeg.org/ffmpeg.git
 - cd ffmpeg
 - ./configure --enable-static --enable-shared --enable-gpl (--enable-libx264)这边不知道为什么在make的时候总会报错，所以干脆不用libx264,因为没有涉及到编码，所以暂且不管。
 - make
 - make install
 - ldconfig

3,安装opencv2.4.8

官网下载open2.4.8

- $ unzip open-2.4.8.zip
- $ cd open-2.4.8
- $ mkdir build
- $ cd build
- $ cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local ..
- $ make -j8
- $ sudo make install
- $ sudo make install
- $ sudo gedit /etc/ld.so.conf.d/opencv.conf   # 添加下面这句命令到文件中，文件件是空的，不影响 /usr/local/lib
- $ sudo ldconfig
- $ sudo vim /etc/profile  # 添加下面两行到文件的末尾。


```

PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig
export PKG_CONFIG_PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

```

路径配置完毕进行测试：

- $ cd opencv-2.4.5/samples/c
- $ chmod +x build_all.sh
- $ ./build_all.sh
- $ ./facedetect --cascade="/usr/local/share/OpenCV/haarcascades/haarcascade_frontalface_alt.xml" --scale=1.5 lena.jpg
