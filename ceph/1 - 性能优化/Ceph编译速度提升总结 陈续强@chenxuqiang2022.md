## Ceph编译出包速度提升方法

### 问题说明

#### 问题现状

Ceph编译出包速度慢，以Ceph14.2.10为例，在一台96核的机器上，整个编译出包的时间超过30min，其中编译连接耗时超过13min，rpm出包耗时超过20min；过长的编译时间导致难以忍受的开发验证；

#### 问题分析

- Ceph源码量大，文件依赖复杂；
- 编译时间长，除开网络下载速度，使用32线程编译全量编译一次需要15min左右的时间。由于头文件依赖负载，导致某些文件修改会触发大量的编译操作，增量编译时间也较长；
  - 以上两条原因主要是由C++本身的语言特性决定的，这些特性包括：
    - 独立编译，头文件独立解析，头文件嵌套深，尤其是Boost库的头文件；
    - 头文件中含有大量的模板函数等编译时期多态，展开开销大；
- 编译过程中的编译选项优化，如-O2的编译优化 ；
- 网络软件包下载耗时时间长；
- 软件包链接速度慢，主要是跨编译单元的优化，都需要连接器完成，同时它又是单进程处理；
- rpm出包过程中，extract debugInfo时间长，压缩时间长，尤其是部分包，压缩时间超过10min；

### 第三方库编译

* 使用do_cmake.sh时，Boost/rocks_db库的编译默认是单线程的，我们这个通过增加：

  ```sh
  $ do_cmake.sh -DBOOST_J = 64
  ```

  增加编译的线程数量；

### 并发修改 

* 如果是在rpmbuild构建中，则需要配置`$CEPH_SMP_NCPUS`选项，他由两个位置决定：

  1. rpmmacros:

     ```sh
       2 %_topdir /home/chenxuqiang/myramdisk/rpmbuild
       3
       4 %_smp_mflags %( \
       5     [ -z "$RPM_BUILD_NCPUS" ] \\\
       6         && RPM_BUILD_NCPUS="`/usr/bin/nproc 2>/dev/null || \\\
       7                              /usr/bin/getconf _NPROCESSORS_ONLN`"; \\\
       8     if [ "$RPM_BUILD_NCPUS" -gt 16 ]; then \\\
       9         echo "-j16"; \\\
      10     elif [ "$RPM_BUILD_NCPUS" -gt 3 ]; then \\\
      11         echo "-j$RPM_BUILD_NCPUS"; \\\
      12     else \\\
      13         echo "-j3"; \\\
      14     fi )
     ```

     可以看到他会根据CPU的数量来设置，如果大于16个，则采用-j16，但是由于服务器的CPU数量较多，我们可以增加更多的判断：

     ```sh
      8     elif [ "$RPM_BUILD_NCPUS" -gt 3 ]; then \\\
      9         echo "-j$RPM_BUILD_NCPUS"; \\\
      10     else \\\
      11         echo "-j3"; \\\
      12     fi )
     ```

  2. ceph.spec：

     ```sh
     1141 echo "Available memory:"
     1142 free -h
     1143 echo "System limits:"
     1144 ulimit -a
     1145 if test -n "$CEPH_SMP_NCPUS" -a "$CEPH_SMP_NCPUS" -gt 1 ; then
     1146     mem_per_process=2500
     1147     max_mem=$(LANG=C free -m | sed -n "s|^Mem: *\([0-9]*\).*$|\1|p")
     1148     max_jobs="$(($max_mem / $mem_per_process))"
     1149     test "$CEPH_SMP_NCPUS" -gt "$max_jobs" && CEPH_SMP_NCPUS="$max_jobs" && echo "Warning: Reducing build parallelism to -     j$max_jobs because of memory limits"
     1150     test "$CEPH_SMP_NCPUS" -le 0 && CEPH_SMP_NCPUS="1" && echo "Warning: Not using parallel build at all because of memory      limits"
     1151 fi
     ```

     可以看到上面的内存也有限制：最大的线程数量 = free_memory / 2.5G，即每个线程预留2.5G内存。通过内存限制了并发数量，可以通过修改2500来增加并发，但是不能太小，否则出现内存不足OOM的问题导致编译失败。

### 去除不必要的编译模块

 * Dashboard等不需要的模块可以不参与编译，通过修改CMakeList.txt完成

### rpmbuild加速 - 解决压缩时间长的问题

* 修改rpmbuild的压缩算法等级和并发度：

  - 修改`/usr/lib/rpm/redhat/macros`中的

    ```
    - 181 %binary_payload	w2.xzdio
    + 181 %binary_payload	w2T12.xzdio
    ```

  - 修改之后，查看是否生效：

    ```
    $ rpmbuild --showrc | grep "binary_payload"
    -14: _binary_payload	w2T12.xzdio
    ```

  - 之后重新rpmbuild，重新出包，测试发现原来耗时比较长的WROTE流程，当前只需要4min左右就可以完成，rpmbuild的CPU使用率达到了1200%；性能提升比较明显；

### Ccache - 解决头文件依赖的问题

#### [Ccache](https://ccache.dev/)编译加速

- 使能：

  下载ccache-4.6.1源码包:[ccache-4.6.1.tar.gz](https://github.com/ccache/ccache/releases/download/v4.6.1/ccache-4.6.1.tar.gz), 解压之后

  ```sh
  $ mkdir build && cd build && cmake ..
  ```

  ```
  $ make -j32 && make install
  ```

  安装完成之后，则Ceph源码的CMake文件会自动找到该组件，并开启该功能；

  如果要手动关闭和开启，则可以再do_cmake.sh时使用: 

  ```
  $ ./do_cmake.sh -DWITH_CCACHE={ON/OFF}
  ```

  如果要在rpmbuild的时候开启：

  ```sh
  $ vim ceph.spec
  找到${CMAKE} ..所在行，增加-DWITH_CCACHE=ON：
  ```

  ```diff
     1162 CMAKE=cmake
     1163 %endif
     1164 ${CMAKE} .. \
   + 1165     -DWITH_CCACHE=ON          \
  ```

- Ccache的一些使用方法

  - 修改缓存目录：

    ```sh
    $ export CCACHE_DIR=/SSD/ccache
    ```

    要修改默认的缓存目录：

    ```sh
    $ vim /home/${user}/.ccache/ccache.conf
    ccache_dir = /ssd/ccache	#通过将缓存目录设置在快速设备，可以提高性能；
    ```

  - 设置缓存大小：

    ```sh
    $ vim /home/${user}/.ccache/ccache.conf
    max_size = 5.0G		# 避免缓存淘汰，设置稍微大一点
    ```

  - 显示统计数据：

    ```sh
    $ ccache -s			# 可以检视hit和miss，查看性能变化
    ```

  - 清理缓存：

    ```sh
    $ ccache -C
    ```

  - 显示所有配置

    ```sh
    $ ccache -p
    ```

- Ccache中的一些和性能相关的配置：


### distcc分布式编译 - 解决并发编译速度不够的问题

#### [distcc](https://www.distcc.org/)分布式编译加速

- 编译安装：需要liberary.a --> 其源码在GCC源码中，编译出来，拷贝到/usr/lib64目录下面；

- 编译

  ```sh
  $ ./configure && make -j32 && make install
  ```

  编译出来之后，需要调用：

  ```sh
  $ update-distcc-symlinks
  ```

  生成gcc，cc，g++等编译器的distcc版本，即命名伪装，会在/usr/lib/distcc下面生成多个编译器的软连接，他们都是distcc的伪装：

  ```sh
   $ ll -h /usr/lib/distcc
   root root 22 May 18 14:59 aarch64-linux-gnu-g++ -> ../../local/bin/distcc
   root root 22 May 18 14:59 aarch64-linux-gnu-gcc -> ../../local/bin/distcc
   root root 22 May 18 14:59 c++ -> ../../local/bin/distcc                  
   root root 22 May 18 14:59 c89 -> ../../local/bin/distcc                  
   root root 22 May 18 14:59 c99 -> ../../local/bin/distcc                  
   root root 22 May 18 14:59 cc -> ../../local/bin/distcc                   
   root root 22 May 18 14:59 g++ -> ../../local/bin/distcc                  
   root root 22 May 18 14:59 gcc -> ../../local/bin/distcc                  
  ```

- 运行

  1. 首先在不同的hosts上启动distccd服务端：

     ```sh
     $ distccd --daemon --user nobody --allow 172.17.0.0/24 --listen 172.17.0.2 --log-stderr --log-level=debug --no-detach --jobs 12
     ```

     可以设置监听IP，端口，日志位置，日志级别，运行模式，并发数量；更多options查看[distccd(1)](https://linux.die.net/man/1/distccd)

  2. 服务端启动之后，则可以配置客户端distcc的环境变量：

     ```sh
     $ export DISTCC_HOSTS="server1,cpp,lzo server2,cpp,lzo server3,cpp,lzo"											// 服务端Host列表
     $ export DISTCC_VERBOSE=1					// verbose
     $ export DISTCC_LOG="/var/log/distcc.log"	// 日志位置
     ```

     当然也可以卸载~/.bashrc中，保证每个终端自动生效；更多配置查看[distcc(1)](https://linux.die.net/man/1/distcc)

  3. 如果使用CMake编译，则需要替换`${CMAKE_C_COMPILER}`和`${CMAKE_CXX_COMPILER}`两个变量为distcc伪装的`cc`和`c++`，如ceph编译中：

     ```sh
     $ ./do_cmake.sh -DCMAKE_C_COMPILER=/usr/lib/distcc/cc -DCMAKE_CXX_COMPILER=/usr/lib/distcc/c++
     ```

     如果是Makefile，则：

     ```sh
     $ make -jxxx CC="distcc, gcc" CXX="distcc, g++"
     ```

  4. 然后进入build目录，之后有两种模式可以选择：

     base模式

     ```sh
     $ make -jxxx    // 其中xxx可以超过本机节点的CPU核数
     ```

     这种模式网络交互小，但是没有include分析，因此只会提升编译的并发度。

     [pump](https://linux.die.net/man/1/pump)模式：

     ```sh
     $ pump make -jxxxx // 配置DISTCC_HOSTS时需要配置,cpp,lzo
     ```

     该模式会将头文件一起推送到远程节点，进行分析编译。效果可能更好。

### 使用tmpFS加速 - 解决磁盘读写瓶颈的问题

#### 挂载tmpfs

1. 挂载一个tmpfs，这里大小指定为80G，由于整个编译过程中会产生50G的临时数据 + ccache的缓存空间，我们设置为80G

   ```sh
   $ sudo mount -t tmpfs -o size=80G myramdisk /home/chenxuqiang/myramdisk
   ```

   ```sh
   $ df -h
   myramdisk                   80G   42G   39G  53% /home/chenxuqiang/myramdisk
   ```

2. 在tmpfs下面创建对应的rpmbuild路径：

   ```sh
   $ mkdir ${tmpfs-dir}/rpmbuild/{BUILD, RPMS, SRPMS, SOURCES, SPECS}
   ```

3. 设置ccache的CCACHE_DIR：

   ```sh
   $ vim /home/${user}/.ccache/ccache.conf
   cache_dir = ${tmpfs-dir}/ccache
   ```

4. 拷贝ceph.spec & source.tar

   ```sh
   $ cp ceph.spec ${tmpfs-dir}/rpmbuild/SPECS
   $ cp ceph.tar.gz ${tmpfs-dir}/rpmbuld/SOURCES
   ```

5. 修改rpmbuild的macros：

   ```sh
   $ vim ~/.rpmmacros
   %_topdir ${tmpfs-dir}/rpmbuild
   ```

6. 开始构建

   ```sh
   $ rpmbuild -bb ${tmpfs-dir}/rpmbuild/SPECS/ceph.spec
   ```

### 源码修改 - 使用新的C++特性减少多态开销（长期工作）

* 减少boost库的使用；
* 使用C++20 的Modules减少共同头文件传递的开销；
* 头文件依赖顺序修改；
* 前置类型声明代替头文件include；

### 效果

通过上述手段，编译出包时间，可以从30min以上降到10min以内。



