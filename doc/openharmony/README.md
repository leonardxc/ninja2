# Compiling Harmony with Ninja2

## 一、本地编译

1.获取OpenHarmony源码

（1）基于docker编译

```sh
sudo docker pull swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
wget -c https://repo.huaweicloud.com/openharmony/os/4.1-Release/code-v4.1-Release.tar.gz
tar -zxvf code-v4.1-Release.tar.gz
cd OpenHarmony-v4.1-Release/OpenHarmony
sudo docker run -it -v $(pwd):/home/openharmony swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
root@ada29360c83c:/home/openharmony# ./build.sh --product-name rk3568 --ccache
```
https://gitee.com/openharmony/docs
开发环境容器镜像https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/get-code/gettools-acquire.md

（2）搭建环境编译

```sh
# 开发环境
Ubuntu18.04及以上版本，X86_64架构，内存推荐 16GB 及以上。
Ubuntu系统的用户名不能包含中文字符。

# 安装库和工具集
apt-get update
apt-get install binutils binutils-dev git git-lfs gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 libc6-dev
apt-get install lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev ccache libgl1-mesa-dev libxml2-utils xsltproc unzip m4 bc gnutls-bin python3.8 python3-pip ruby genext2fs device-tree-compiler make libffi-dev e2fsprogs pkg-config perl openssl libssl-dev libelf-dev libdwarf-dev u-boot-tools mtd-utils cpio doxygen liblz4-tool openjdk-8-jre gcc g++ texinfo dosfstools mtools default-jre default-jdk libncurses5 apt-utils wget scons python3.8-distutils tar rsync git-core libxml2-dev lib32z-dev grsync xxd libglib2.0-dev libpixman-1-dev kmod jfsutils reiserfsprogs xfsprogs squashfs-tools pcmciautils quota ppp libtinfo-dev libtinfo5 libncurses5-dev libncursesw5 libstdc++6 gcc-arm-none-eabi vim ssh locales libxinerama-dev libxcursor-dev libxrandr-dev libxi-dev

# Python 3.8设置为默认Python版本
apt install python3.8 -y
which python3.8
cd /usr/bin && rm python && ln -s /usr/bin/python3.8 python && python --version

# 安装编译工具
wget -c https://repo.huaweicloud.com/openharmony/os/4.1-Release/code-v4.1-Release.tar.gz
tar -zxvf code-v4.1-Release.tar.gz
cd OpenHarmony-v4.1-Release/OpenHarmony/
python3 -m pip install --user build/hb
vim ~/.bashrc
#将以下命令拷贝到.bashrc文件的最后一行，保存并退出。
#export PATH=~/.local/bin:$PATH
source ~/.bashrc

# 使用原ninja能编译成功
./build.sh --product-name rk3568 --ccache
```

2.编译ninja2

[ninja-cloudbuild/ninja2](https://github.com/ninja-cloudbuild/ninja2)

```sh
git clone https://github.com/ninja-cloudbuild/ninja2.git
cd ninja2
./build.sh build
```

3.使用ninja2编译OpenHarmony

```sh
cd /home/user/OpenHarmony-v4.1-Release/OpenHarmony/
sudo cp prebuilts/build-tools/linux-x86/bin/ninja prebuilts/build-tools/linux-x86/bin/ninja.prev
sudo mv /home/user/ninja2/build/ninja prebuilts/build-tools/linux-x86/bin/
# 开始编译
./build.sh --product-name rk3568 --ccache
```

## 二、分布式编译

`for OpenharmonyOS V4.1`

| 主机          | 编号   | 系统            | 内存  | 物理核数 | 逻辑CPU | 运行环境 |
| ------------- | ---- | ------------------ | --- | ---- | ----- | ---- |
| 源码及ninja2  | lab0 | Ubuntu 22.04.5 LTS | 32G | 12   | 16    | 容器   |
| server        | lab2 | Ubuntu 22.04.5 LTS | 32G | 12   | 16    | 主机   |
| executor      | lab1 | Ubuntu 22.04.2 LTS | 32G | 12   | 16    | 容器   |

# 1.操作步骤

## 1.1 配置 NFS Server

```sh
wget -c https://repo.huaweicloud.com/openharmony/os/4.1-Release/code-v4.1-Release.tar.gz
tar -zxvf code-v4.1-Release.tar.gz
cd OpenHarmony-v4.1-Release/OpenHarmony
# 主要配置OpenHarmony的prebuilts
apt install nfs-kernel-server -y

vim /etc/exports
# /etc/exports 添加
# ===========================
........./OpenHarmony-v4.1-Release/OpenHarmony/prebuilts 
*(rw,sync,no_root_squash,no_subtree_check)
# ===========================

# 2.生效并启动
exportfs -rav

systemctl restart nfs-kernel-server
systemctl enable nfs-kernel-server
systemctl status nfs-kernel-server

#可在其他主机上使用以下命令查看
showmount -e [NFS SERVER IP] 
```

## 1.2 运行 buildbuddy server
### (1) clone
```sh
git clone https://github.com/buildbuddy-io/buildbuddy.git
# git clone git@github.com:buildbuddy-io/buildbuddy.git
```
### (2) 运行server
```sh
cd buildbuddy/
bazel build //enterprise/server:server
bazel run //enterprise/server:server
```


## 1.3 运行 buildbuddy executor
```sh
docker pull swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2

docker volume create --driver local --opt type=nfs --opt o=addr="192.168.1.10",rw --opt device=:/home/lxc/OpenHarmony-v4.1-Release/OpenHarmony/prebuilts --name ohnfs

docker run -it --network=host -v ohnfs:/tmp/prebuilts swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
# 下次进入容器时使用 docker exec -it [docker id] /bin/bash

# 进入容器内
git clone https://github.com/buildbuddy-io/buildbuddy.git
cd buildbuddy/
bazel build //enterprise/server/cmd/executor:executor
bazel run //enterprise/server/cmd/executor:executor
```

## 1.2 编译

### (1) 准备 OHOS 编译环境
```sh
cd OpenHarmony-v4.1-Release/OpenHarmony
docker run -it -v $(pwd):/home/openharmony swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
```

### (2) 编译 ninja2
```sh
git clone https://github.com/ninja-cloudbuild/ninja2.git
cd ninja2
./build.sh build
```

### (3) 配置 ninja2
```sh
cd /home/openharmony
# .cloudbuild.yml文件默认放在项目根目录下
cp /path/to/ninja2/doc/openharmony/.cloudbuild.yml ./
# .ninja2.conf 默认放在项目根目录下，放在$HOME/.ninja2.conf 为当前用户全局有效影响所有项目的ninja2编译
cp /path/to/ninja2/doc/openharmony/.ninja2.conf ./
```
注意.ninja2.conf文件中的IP地址修改为您实际的server IP地址
```
cloudbuild: true
grpc_url: "grpc://[您实际的server IP地址]:1985"
```
cp prebuilts/build-tools/linux-x86/bin/ninja prebuilts/build-tools/linux-x86/bin/ninja-back
rm prebuilts/build-tools/linux-x86/bin/ninja
cp /path/to/ninja2/build/bin/ninja prebuilts/build-tools/linux-x86/bin/

### (5) 开始编译
```sh
./build.sh --product-name rk3568 --ccache=false --ninja-args="-r/home/openharmony"
```
