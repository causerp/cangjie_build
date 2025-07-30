# 仓颉SDK构建指导书 (Ubuntu 22.04)

## 1 构建概述

### 1.1 构建流程全景图

```mermaid
graph LR
    A[准备环境] --> B[编译核心<br>依赖]
    B --> C[创建<br>工作区]
    C --> D[获取<br>仓颉源码]
    D --> E[编译仓颉<br>编译器和调试器]
    E --> F[编译<br>仓颉运行时]
    F --> G[编译<br>仓颉标准库]
    G --> H[编译<br>STDX扩展库]
    H --> I[编译<br>仓颉工具集]
    I --> J[组织并<br>打包文件]
    J --> K[编译<br>Hello,Cangjie程序]
```

### 1.2 关键注意事项

1. **环境隔离**：所有自定义依赖安装在 `/opt/buildtools`，避免污染系统路径
2. **内存要求**：完整构建需要 ≥8GB 内存，建议添加 4GB 交换空间
3. **网络要求**：首次构建需下载约 3GB 数据，请确保稳定网络连接
4. **windows工具链支持**：需要编译MinGW-w64以及配套工具链
5. **芯片指令集**：请**务必**根据您的计算机芯片架构(x86_64或aarch64)设置对应的环境变量，以及选择合适的系统工具及docker镜像（uname -m)

## 2 环境准备

### 2.1 系统要求

- **操作系统**: Ubuntu 22.04 LTS
- **磁盘空间**: ≥50GB
- **内存**: ≥8GB (物理内存+交换空间)
- **用户权限**: 普通用户 + sudo权限


> 您可以自己搭建相关构建环境，也可以基于我们提供的Docker环境进行仓颉构建，里面已内置所有构建仓颉所需的系统工具和构建工具：
>
> ```bash
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu22_x86_kernel:v1.2
> # 或
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu22_arm_kernel:v1.2
> ```

### 2.2 系统工具安装

- 使用华为镜像源(可选)

  ```bash
  # 1.备份配置文件：
  sudo cp -a /etc/apt/sources.list /etc/apt/sources.list.bak;
  # 2.修改sources.list文件，将http://archive.ubuntu.com和http://security.ubuntu.com替换成http://mirrors.huaweicloud.com，可以参考如下命令：
  sudo sed -i "s@http://.*archive.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  sudo sed -i "s@http://.*security.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  # 3.更新索引
  sudo apt-get update;
  ```

- 安装系统工具

  ```bash
  sudo apt update && sudo apt install -y \
    tar unzip wget curl libcurl4 expat openssl make gcc g++ gettext \
    nfs-common libtool sqlite3 zlib1g-dev libssl-dev cmake ninja-build\
    libcurl4-openssl-dev sudo autoconf build-essential rapidjson-dev \
    texinfo binutils expat libelf-dev libdwarf-dev openssh-client ssh \
    dos2unix libxext-dev libxtst-dev libxt-dev libcups2-dev clang clang-15 libedit-dev\
    libxrender-dev zip bzip2 libopenmpi-dev vim gdb lldb libclang-15-dev libgtest-dev\
    rpm patch libtinfo5 cpio rpm2cpio libncurses5 libncurses5-dev strace net-tools;
  ```

### 2.3 编译MinGW-w64及配套工具链 (关键步骤)

```bash
# 创建工作目录
export BUILD_ROOT=/opt/buildtools;
sudo mkdir -p $BUILD_ROOT;
sudo chown $USER:$USER $BUILD_ROOT;
cd $BUILD_ROOT;
tmp_cpus=$(grep -w processor /proc/cpuinfo|wc -l);
wget http://gcc.gnu.org/pub/gcc/infrastructure/gmp-6.2.1.tar.bz2 --no-check-certificate;
wget http://gcc.gnu.org/pub/gcc/infrastructure/mpfr-4.1.0.tar.bz2 --no-check-certificate;
wget http://gcc.gnu.org/pub/gcc/infrastructure/mpc-1.2.1.tar.gz --no-check-certificate;
wget http://gcc.gnu.org/pub/gcc/infrastructure/isl-0.24.tar.bz2 --no-check-certificate;
wget https://gcc.gnu.org/pub/gcc/releases/gcc-10.3.0/gcc-10.3.0.tar.gz --no-check-certificate;
wget https://gcc.gnu.org/pub/binutils/releases/binutils-2.41.tar.gz --no-check-certificate;
wget https://github.com/mingw-w64/mingw-w64/archive/refs/tags/v11.0.1.tar.gz --no-check-certificate;
wget https://github.com/libffi/libffi/releases/download/v3.4.2/libffi-3.4.2.tar.gz --no-check-certificate;
wget https://ftp.gnu.org/gnu/texinfo/texinfo-6.8.tar.gz --no-check-certificate;
wget https://github.com/openssl/openssl/archive/refs/tags/openssl-3.0.9.tar.gz -q --no-check-certificate;

tar xf gmp-6.2.1.tar.bz2;
tar xf mpfr-4.1.0.tar.bz2;
tar xf mpc-1.2.1.tar.gz;
tar xf isl-0.24.tar.bz2;
tar xf gcc-10.3.0.tar.gz;
tar xf binutils-2.41.tar.gz;
tar xf v11.0.1.tar.gz;
tar xf libffi-3.4.2.tar.gz;
tar xf texinfo-6.8.tar.gz;
tar xf openssl-3.0.9.tar.gz;

cp gmp-6.2.1.tar.bz2 mpfr-4.1.0.tar.bz2 mpc-1.2.1.tar.gz isl-0.24.tar.bz2 gcc-10.3.0;
export install_dir=$BUILD_ROOT/mingw-w64;
mkdir -p $install_dir;

# prepare build envs
src_root=$(pwd);
tmp_cpus=$(grep -w processor /proc/cpuinfo|wc -l);
CFLAG_BASE="-O2 -pipe -fno-ident";
CXXFLAGS_BASE="-O2 -pipe -fno-ident -std=c++17";
export PATH="$install_dir/bin:$PATH";
export CFLAGS="$CFLAGS_BASE -Wl,-z,now -fPIE -pie -fstack-protector-all -s";
export CXXFLAGS="$CXXFLAGS_BASE -Wl,-z,now -fPIE pie -fstack-protector-all -s";
export LDFLAGS="-pipe -fno-ident";

# build texinfo
cd "$src_root/texinfo-6.8";
mkdir -p build;
cd build;
../configure --prefix="$src_root/texinfo";
make -j $tmp_cpus > build.log;
make install > install.log;
export PATH="$src_root/texinfo/bin:$PATH";

# build binutils
cd "$src_root/binutils-2.41";
mkdir build;
cd build;
../configure --prefix="$install_dir" --disable-shared --enable-static --with-sysroot="$install_dir" --target=x86_64-w64-mingw32 --disable-nls --enable-lto --disable-gdb;

make -j $tmp_cpus > build.log;
make install > install.log;

# build mingw-w64-headers
export CFLAGS="$CFLAG_BASE";
export CXXFLAGS="$CXXFLAG_BASE";
cd "$src_root/mingw-w64-11.0.1";
mkdir header-build;
cd header-build;
../mingw-w64-headers/configure --build=x86_64-pc-linux-gnu --host=x86_64-w64-mingw32 --prefix="$install_dir/x86_64-w64-mingw32" --with-default-msvcrt=msvcrt;
make install;

# build gcc
cd "$src_root/gcc-10.3.0";
sed -i 's/gmp-6.1.0.tar.bz2/gmp-6.2.1.tar.bz2/g' ./contrib/download_prerequisites;
sed -i 's/mpfr-3.1.4.tar.bz2/mpfr-4.1.0.tar.bz2/g' ./contrib/download_prerequisites;
sed -i 's/mpc-1.0.3.tar.gz/mpc-1.2.1.tar.gz/g' ./contrib/download_prerequisites;
sed -i 's/isl-0.18.tar.bz2/isl-0.24.tar.bz2/g' ./contrib/download_prerequisites;
./contrib/download_prerequisites --no-verify;
sed -i 's/$(CFLAGS)/-g -O2 -fstack-protector-strong/g' libstdc++-v3/Makefile.in;
sed -i 's/$(CXXFLAGS)/-g -O2 -fstack-protector-strong/g' libstdc++-v3/Makefile.in;
sed -i 's/-fno-stack-protector/-fstack-protector-strong/g' libgcc/Makefile.in;
mkdir build;
cd build;
../configure --target=x86_64-w64-mingw32 --disable-multilib --prefix="$install_dir" --enable-languages=c,c++ --disable-nls --enable-threads=posix --enable-lib64 --disable-lib32;
make -j $tmp_cpus all-gcc > build.log;
make install-gcc > install.log;

# build mingw-w64-crt
cd "$src_root/mingw-w64-11.0.1";
mkdir crt-build;
cd crt-build;
../mingw-w64-crt/configure --build=x86_64-pc-linux-gnu --host=x86_64-w64-mingw32 --prefix="$install_dir/x86_64-w64-mingw32" --with-default-msvcrt=msvcrt --with-sysroot="$install_dir/x86_64-w64-mingw32" --enable-lib64 --disable-lib32 --enable-wildcard;
make -j $tmp_cpus > build.log;
make install > install.log;

# build winpthreads
cd "$src_root/mingw-w64-11.0.1";
mkdir -p winpthreads-build;
cd winpthreads-build;
../mingw-w64-libraries/winpthreads/configure --build=x86_64-pc-linux-gnu --host=x86_64-w64-mingw32 --prefix="$install_dir/x86_64-w64-mingw32";
make -j $tmp_cpus > build.log;
make install > install.log;

# finish build gcc
cd "$src_root/gcc-10.3.0/build";
make -j $tmp_cpus > build.log;
make install > install.log;
touch "$install_dir/lib/gcc/x86_64-w64-mingw32/10.3.0/libgcc_eh.a";

# building windows-native binutils
cd "$src_root/binutils-2.41";
mkdir -p native-build;
cd native-build;
../configure --prefix="$install_dir/x86_64-w64-mingw32" --disable-shared --enable-static --host=x86_64-w64-mingw32 --target=x86_64-w64-mingw32 --disable-multilib --disable-nls --enable-lto --disable-gdb --enable-64-bit-bfd --enable-plugins --enable-gold --disable-rpath;
make -j $tmp_cpus > build.log;
make install > install.log;

# build openssl
cd "$src_root/openssl-openssl-3.0.9";
mkdir build;
cd build;
../Configure mingw64 --prefix="$install_dir/x86_64-w64-mingw32" --cross-compile-prefix=x86_64-w64-mingw32- --libdir=lib;
make -j $tmp_cpus > build.log;
make install > install.log;
```
### 2.4 设置环境变量

请务必在构建前配置以下环境变量：

```bash
export MINGW_PATH=${BUILD_ROOT}/mingw-w64
export PATH=${MINGW_PATH}/bin:/usr/lib/llvm-15/bin:$PATH; # 用于将mingw、clang-15加入环境变量
# 以下变量仅影响文件名
export CANGJIE_VERSION=1.0.0
export STDX_VERSION=1
```

## 3 源码准备

### 3.1 创建工作区

**<font color="red">\* 下方示例的`/path/to/workspace`需更换为您构建仓颉SDK的工作目录</font>**

```bash
export WORKSPACE=/path/to/workspace
mkdir -p $WORKSPACE;
cd $WORKSPACE;
```

### 3.2 获取仓颉源码

```bash
git clone https://gitcode.com/Cangjie/cangjie_compiler.git;
git clone https://gitcode.com/Cangjie/cangjie_runtime.git;
git clone https://gitcode.com/Cangjie/cangjie_tools.git;
git clone https://gitcode.com/Cangjie/cangjie_stdx.git;
```

## 4 编译流程

### 4.1 编译仓颉编译器和调试器

```bash
# 编译linux x64 host
cd ${WORKSPACE}/cangjie_compiler;
python3 build.py clean;
python3 build.py build -t release --no-tests;

# 编译windows sdk
export CMAKE_PREFIX_PATH=${MINGW_PATH}/x86_64-w64-mingw32;
python3 build.py build -t release \
  --product cjc \
  --no-tests \
  --target windows-x86_64 \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin \
  --build-cjdb;
python3 build.py build -t release \
  --product libs \
  --target windows-x86_64 \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin;

python3 build.py install --host windows-x86_64;
python3 build.py install;
cp -rf output-x86_64-w64-mingw32/* output;
```

验证安装：
```bash
source output/envsetup.sh;
cjc -v;
```

### 4.2 编译仓颉运行时

```bash
cd ${WORKSPACE}/cangjie_runtime/runtime;
python3 build.py clean;
python3 build.py build -t release \
  --target windows-x86_64 \
  --target-toolchain ${MINGW_PATH}/bin \
  -v ${CANGJIE_VERSION};
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/windows_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/windows_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output-x86_64-w64-mingw32;
```

### 4.3 编译仓颉标准库

```bash
cd ${WORKSPACE}/cangjie_runtime/std;
python3 build.py clean;
python3 build.py build -t release \
  --target windows-x86_64 \
  --target-lib=${WORKSPACE}/cangjie_runtime/runtime/output \
  --target-lib=${MINGW_PATH}/x86_64-w64-mingw32/lib \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin;
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output/;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output-x86_64-w64-mingw32/;
```

### 4.4 编译 STDX 扩展库

```bash
cd ${WORKSPACE}/cangjie_stdx;
python3 build.py clean;
python3 build.py build -t release \
	--include=$WORKSPACE/cangjie_compiler/include \
    --target-lib=${MINGW_PATH}/x86_64-w64-mingw32/lib \
	--target windows-x86_64 \
    --target-sysroot ${MINGW_PATH}/ \
    --target-toolchain ${MINGW_PATH}/bin;
python3 build.py install;
export CANGJIE_STDX_PATH=${WORKSPACE}/cangjie_stdx/target/windows_x86_64_cjnative/static/stdx;
```

### 4.5 编译工具集

#### (1) cjpm

```bash
cd ${WORKSPACE}/cangjie_tools/cjpm/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

#### (2) cjfmt

```bash
cd ${WORKSPACE}/cangjie_tools/cjfmt/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

#### (3) HyperLangExtension

```bash
cd ${WORKSPACE}/cangjie_tools/hyperlangExtension/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

#### (4) LSP Server

```bash
cd ${WORKSPACE}/cangjie_tools/cangjie-language-server/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

## 5 打包发布

### 5.1 组织文件

```bash
# 清空历史构建
mkdir -p $WORKSPACE/software;
rm -rf $WORKSPACE/software/*;
cd $WORKSPACE/software;

# 拷贝cangjie目录
cp -R $WORKSPACE/cangjie_compiler/output-x86_64-w64-mingw32 cangjie;

# 删除 ast-support.a
rm -rf cangjie/lib/windows_x86_64_cjnative/libcangjie-ast-support.a

# 组织文件
cp $WORKSPACE/cangjie_tools/cjpm/dist/cjpm.exe cangjie/tools/bin;
mkdir -p cangjie/tools/config;
cp $WORKSPACE/cangjie_tools/cjfmt/build/build/bin/cjfmt.exe cangjie/tools/bin;
cp $WORKSPACE/cangjie_tools/cjfmt/config/*.toml cangjie/tools/config;
cp $WORKSPACE/cangjie_tools/hyperlangExtension/target/bin/main.exe cangjie/tools/bin/hle.exe;
cp -r $WORKSPACE/cangjie_tools/hyperlangExtension/src/dtsparser cangjie/tools;
rm -rf cangjie/tools/dtsparser/*.cj;
cp $WORKSPACE/cangjie_tools/cangjie-language-server/output/bin/LSPServer.exe cangjie/tools/bin;

# 打包和设置权限
chmod -R 750 cangjie
zip -qr cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip cangjie;
chmod 550 cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip;
```

### 5.2 组织并打包 STDX

```bash
cd $WORKSPACE/software;
# 拷贝stdx目录
cp -r $WORKSPACE/cangjie_stdx/target/windows_x86_64_cjnative ./;
chmod -R 750 windows_x86_64_cjnative;
zip -qr cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip windows_x86_64_cjnative;
chmod 550 cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip;
```

## 6 编译Hello, Cangjie程序

检查生成的文件：

```bash
ls -lh $WORKSPACE/software
# 应包含：
# - cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip
# - cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip
```

验证hello cangjie程序（请在windows执行以下命令）：

1. 安装仓颉SDK

   - 若使用 Windows 命令提示符（CMD）环境，请执行：

     ```bash
     path\to\cangjie\envsetup.bat
     ```

   - 若使用 PowerShell 环境，请执行：

     ```bash
     . path\to\cangjie\envsetup.ps1
     ```

   - 若使用 MSYS shell、bash 等环境，请执行：

     ```bash
     source path/to/cangjie/envsetup.sh
     ```

2. 创建并运行cj文件：

   ```bash
   echo "main() { println(\"Hello, Cangjie\") }" > hello.cj
   cjc hello.cj -o hello.exe && ./hello.exe
   # 应打印：
   # Hello, Cangjie
   ```

🎉 恭喜您成功构建并运行了了仓颉SDK并运行了Hello, Cangjie程序！
   
