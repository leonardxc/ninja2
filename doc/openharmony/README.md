# Compiling Harmony with Ninja2

## 一、本地编译

1.获取OpenHarmony源码

（1）基于docker编译

[harmony编译](https://gitee.com/cloudbuild888/cloudbuild/blob/master/doc/projects/openharmony.md)

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
cd cd OpenHarmony-v4.1-Release/OpenHarmony/
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
| 源码          | lab0 | Ubuntu 22.04.5 LTS | 32G | 12   | 16    | 容器   |
| remote server | lab2 | Ubuntu 22.04.5 LTS | 32G | 12   | 16    | 主机   |
| executor      | lab1 | Ubuntu 22.04.2 LTS | 32G | 12   | 16    | 容器   |

# 1.操作步骤

## 1.1 运行buildbuddy
### 1.1.1 clone
```sh
git clone https://github.com/buildbuddy-io/buildbuddy.git
# git clone git@github.com:buildbuddy-io/buildbuddy.git
```
### 1.1.2 运行server
```sh
cd buildbuddy/
bazel build //enterprise/server:server
bazel run //enterprise/server:server
```
### 1.1.3 运行executor
```sh
docker pull swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
docker run -it --network=host swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
# 下次进入容器时使用 docker exec -it [docker-id] /bin/bash

wget -c https://repo.huaweicloud.com/openharmony/os/4.1-Release/code-v4.1-Release.tar.gz
tar -zxvf code-v4.1-Release.tar.gz
rm code-v4.1-Release.tar.gz
ln -s /.../Openharmony/prebuilts /tmp

cd buildbuddy/
bazel build //enterprise/server/cmd/executor:executor
bazel run //enterprise/server/cmd/executor:executor
```

## 1.2 编译

**(1) 准备 OHOS 编译环境**
```sh
wget -c https://repo.huaweicloud.com/openharmony/os/4.1-Release/code-v4.1-Release.tar.gz
tar -zxvf code-v4.1-Release.tar.gz
cd OpenHarmony-v4.1-Release/OpenHarmony
docker run -it -v $(pwd):/home/openharmony swr.cn-south-1.myhuaweicloud.com/openharmony-docker/docker_oh_standard:3.2
```

**(2) 编译 ninja2**
```sh
git clone https://github.com/ninja-cloudbuild/ninja2.git
cd ninja2
./build.sh build
```

**(3) 配置 $HOME/.ninja2.conf**
```
cloudbuild: true
grpc_url: "grpc://192.168.1.12:1989"
```

**(4) .cloudbuild.yml**
```
# 作用：配置 RBE（远程构建执行）的命令规则
rules:
  # 仅本地仅执行的命令（如本地文件预处理、依赖检查）
  local_only_rules:
    - "CXX_COMPILER__LumeShaderCompiler"
    - "CXX_EXECUTABLE_LINKER__LumeShaderCompiler" 
    - "CXX_COMPILER__SPIRV-Tools-shared"
    - "CXX_SHARED_LIBRARY_LINKER__SPIRV-Tools-shared"
    - "CXX_COMPILER__SPIRV-Tools-static"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV-Tools-static"
    - "CXX_COMPILER__SPIRV-Tools-opt"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV-Tools-opt"
    - "CXX_COMPILER__SPIRV-Tools-reduce"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV-Tools-reduce"
    - "CXX_COMPILER__SPIRV-Tools-link"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV-Tools-link"
    - "CXX_COMPILER__SPIRV-Tools-lint"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV-Tools-lint"
    - "CXX_COMPILER__SPIRV-Tools-diff"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV-Tools-diff"
    - "CXX_COMPILER__GenericCodeGen"
    - "CXX_STATIC_LIBRARY_LINKER__GenericCodeGen"
    - "CXX_COMPILER__MachineIndependent"
    - "CXX_STATIC_LIBRARY_LINKER__MachineIndependent"
    - "CXX_COMPILER__glslang"
    - "CXX_STATIC_LIBRARY_LINKER__glslang"
    - "CXX_COMPILER__OSDependent"
    - "CXX_STATIC_LIBRARY_LINKER__OSDependent"
    - "CXX_COMPILER__OGLCompiler"
    - "CXX_STATIC_LIBRARY_LINKER__OGLCompiler"
    - "CXX_COMPILER__SPIRV"
    - "CXX_STATIC_LIBRARY_LINKER__SPIRV"
    - "CXX_COMPILER__spirv-cross-glsl"
    - "CXX_STATIC_LIBRARY_LINKER__spirv-cross-glsl"
    - "CXX_COMPILER__spirv-cross-core"
    - "CXX_STATIC_LIBRARY_LINKER__spirv-cross-core"
    - "CXX_COMPILER__LumeAssetCompiler"
    - "CXX_EXECUTABLE_LINKER__LumeAssetCompiler"
    - "clang_x64_rust_bin"
    - "clang_x64_rust_macro"
    - "clang_x64_rust_rlib"
    - "rust_bin"
    - "rust_cdylib"
    - "rust_dylib"
    - "rust_rlib"
    - "rust_staticlib"
    - "solink"
    - "alink"
    - "link"
    - "clang_x64_solink"
    - "clang_x64_alink"
    - "clang_x64_link"    
    - "mingw_x86_64_solink"
    - "mingw_x86_64_alink"
    - "mingw_x86_64_link"
    - "stamp"
    - "copy"
  # 远程执行但不缓存的命令（如每次需重新拉取资源的命令）
  remote_exec_rules:
  # 模糊匹配的命令（如包含特定关键词的编译命令）
  local_only_fuzzy:
    - "build_toolchain_ohos_ohos_clang_arm__rule"
    - "build_toolchain_linux_clang_x64__rule"
    - "build_toolchain_mingw_mingw_x86_64__rule"
    - "asm_defines/asm_defines/defines"
    - "soft_musl_crt/crti"
    - "FREETYPE_MINOR"
    - "compiler/optimizer/code_generator/target"
    - "QuickFixService"
    - "BundleTool"
    - "render_service_base/rs_base_blocklist"
    - "render_service_client/rs_client_blocklist"
    - "externals/libjpeg-turbo/libjpeg"
    - "externals/libjpeg-turbo/simd"
```

**(5) 开始编译**
```sh
cp prebuilts/build-tools/linux-x86/bin/ninja prebuilts/build-tools/linux-x86/bin/ninja-back
rm prebuilts/build-tools/linux-x86/bin/ninja
cp /home/ninja2/build/bin/ninja prebuilts/build-tools/linux-x86/bin/
./build.sh --product-name rk3568 --ccache=false --ninja-args="-r/home/openharmony"
```
