# ClickHouse 源码编译

此文参考 https://www.cnblogs.com/xianghuaqiang/p/14381077.html  
记录一下clickhouse编译的过程，以便未来可能需要对clickhouse进行源码改造。

## 编译过程

1. 首先，使用yum安装依赖组件

```
yum -y install gcc

yum -y install gcc-c++

yum -y install zlib

yum -y install zlib-devel

yum -y install libtool

yum -y install -y libstdc++-static

yum -y install -y readline-devel

yum -y install -y libicu-devel
```

2. 安装 cmake 3.3 以上的版本

- 下载 [cmake](https://cmake.org/download/) 3.3 以上的版本
- 解压 cmake

```
tar zxvf cmake-3.19.0-rc1.tar.gz.

```
- 编译并安装

```
./bootstrap
make && make install
```

3. 安装 gcc

- 下载gcc 源码，版本9.0以上

```
wget https://ftp.gnu.org/gnu/gcc/gcc-9.1.0/gcc-9.1.0.tar.gz
```

- 解压gcc，切换目录

```
tar zxvf gcc-9.1.0.tar.gz
cd gcc-9.1.0
```

- 执行脚本download_prerequisites，下载所依赖库

```
./contrib/download_prerequisites
```
或者手动将这些包下载到gcc源码的目录中，然后解压，并创建如下软连接。

```
tar zxvf mpc-1.0.3.tar.gz
tar jxvf gmp-6.1.0.tar.bz2
tar jxvf isl-0.18.tar.bz2
tar jxvf mpfr-3.1.4.tar.bz2
ln -sf mpc-1.0.3 mpc
ln -sf gmp-6.1.0 gmp
ln -sf isl-0.18 isl
ln -sf mpfr-3.1.4 mpfr
```

- 编译前配置

```
./configure --enable-languages=c,c++ --disable-multilib --disable-checking
```

- 扩大交换分区,gcc 的编译和安装需要很大的存储空间

```
SWAP=/tmp/swap
dd if=/dev/zero of=$SWAP bs=1M count=500
mkswap $SWAP
swapon $SWAP
```

- 编译

```
make -j 16
```

- 安装

```
make install
```

- 完成上述过程后，gcc会安装到 /usr/local/bin。进入这个目录，创建如下软链接

```
ln -sf /usr/local/bin/gcc /usr/local/bin/gcc-9
ln -sf /usr/local/bin/g++ /usr/local/bin/g++-9
ln -sf /usr/local/bin/gcc /usr/local/bin/cc
ln -sf /usr/local/bin/g++ /usr/local/bin/c++
```

- 将动态库 libstdc++.so 复制到 /usr/lib64 目录中。对于gcc-9.1 需要复制的是 libstdc++.so.6.0.26

```
cp ./x86_64-pc-linux-gnu/libstdc++-v3/src/.libs/ libstdc++.so.6.0.26 /usr/lib64
```

- 在/usr/lib64中创建指向libstdc++.so.6.0.26的软链接

```
cd /usr/lib64
rm -f libstdc++.so.6
ln -s libstdc++.so.6.0.26 libstdc++.so.6
```

4. 安装 clickhouse

- 声明如下环境变量

```
export CC=gcc-9
export CXX=g++-9
```

- 下载 ClickHouse源码

- 解压源码，并进入解压后的目录

```
tar ClickHouse-20.9.2.20-stable.tar.gz
cd ClickHouse-20.9.2.20-stable
```

- 编译并安装

```
mkdir build
cd build
cmake ..
ninja
```

- 安装后，创建用户clickhouse，用户组clickhouse

```
groupadd -g 993 clickhouse
useradd -g clickhouse --no-create-home -d /nonexistent --shell /bin/false clickhouse
```
