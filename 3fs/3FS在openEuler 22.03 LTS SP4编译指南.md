# 环境信息
OS: openEuler 22.03 LTS SP4
gcc: 10.3.1
clang: bisheng 4.2 clang-17

# 下载代码
git clone https://github.com/deepseek-ai/3FS.git

# 安装libfuse
wget https://github.com/libfuse/libfuse/releases/download/fuse-3.16.1/fuse-3.16.1.tar.gz  --no-check
tar  -zxvf  fuse-3.16.1.tar.gz
mkdir build
cd build
yum install meson
meson setup ..
ninja
ninja install

# 安装毕昇编译器     
wget https://mirrors.huaweicloud.com/kunpeng/archive/compiler/bisheng_compiler/BiShengCompiler-4.2.0-aarch64-linux.tar.gz --no-check

## 使能环境变量
###   设置安装目录。
#### 创建毕昇编译器安装目录（这里以/opt/compiler为例）
mkdir -p /opt/compiler
#### 将毕昇编译器压缩包拷贝到安装目录下：
cp BiShengCompiler-4.2.0-aarch64-linux.tar.gz /opt/compiler
### 解压压缩包。
cd /opt/compiler
tar -zxvf BiShengCompiler-4.2.0-aarch64-linux.tar.gz
#### 解压完成后在当前目录下出现名为BiShengCompiler-4.2.0-aarch64-linux的目录。

chmod -R 755 ${PATH_TO_BISHENG}/BiShengCompiler-x.x.x-aarch64-linux

### 配置毕昇编译器的环境变量。
export PATH=/opt/compiler/BiShengCompiler-4.2.0-aarch64-linux/bin:$PATH 
export LD_LIBRARY_PATH=/opt/compiler/BiShengCompiler-4.2.0-aarch64-linux/lib:/opt/compiler/BiShengCompiler-4.2.0-aarch64

### 参考   
https://www.hikunpeng.com/document/detail/zh/kunpengdevps/compiler/ug-bisheng/kunpengbisheng_06_0005.html

# 安装foundationdb
##  clients
wget https://github.com/apple/foundationdb/releases/download/7.3.61/foundationdb-clients-7.3.61-1.el9.aarch64.rpm --no-check
rpm  -ivh foundationdb-clients-7.3.61-1.el9.aarch64.rpm
##  server
wget https://github.com/apple/foundationdb/releases/download/7.3.61/foundationdb-server-7.3.61-1.el9.aarch64.rpm --no-check
rpm  -ivh foundationdb-server-7.3.61-1.el9.aarch64.rpm

# 安装rust
## 使用阿里云安装脚本
curl --proto '=https' --tlsv1.2 -sSf https://mirrors.aliyun.com/repo/rust/rustup-init.sh | sh
参考
https://developer.aliyun.com/mirror/rustup?spm=a2c6h.13651102.0.0.3ca81b114Iqa71

# 安装依赖包
yum install cmake libuv-devel lz4-devel xz-devel double-conversion-devel libdwarf-devel libunwind-devel libaio-devel gflags-devel glog-devel gtest-devel gmock-devel lld gperftools-devel openssl-devel gcc gcc-c++ boost-devel libatomic libibverbs-devel rsync gperftools-libs libevent-devel glibc-devel python3-devel

# 下载依赖的第三方组件
cd  3FS
git submodule update --init --recursive

# 打补丁
./patches/apply.sh

# 制作 folly llvm 协程的patch
wget https://github.com/facebook/folly/commit/8f8f6cfae4ff021e9ca098c1ed3d337bc41c1510.patch

cd third_party/folly/
patch -p1 < 8f8f6cfae4ff021e9ca098c1ed3d337bc41c1510.patch

# 3FS代码修改
```
From 3c19b60d2ac1a3a9bbc92e83a8fd979e3e076635 Mon Mar 24 00:00:00 2025
From: wangzengliang <wangzengliang2@huawei.com>
Date: Tue, 25 Mar 2025 11:31:30 +0800
Subject: [PATCH] diff

---
 .cargo/config.toml      |  2 +-
 .clangd                 |  2 +-
 CMakeLists.txt          |  6 ++++--
 cmake/CLangFormat.cmake |  6 +++++-
 cmake/CLangTidy.cmake   |  8 ++++----
 setup.py                |  4 ++--
 src/fuse/CMakeLists.txt | 12 +++++++-----
 7 files changed, 24 insertions(+), 16 deletions(-)

diff --git a/.cargo/config.toml b/.cargo/config.toml
index ab8abf6..ba96e8a 100644
--- a/.cargo/config.toml
+++ b/.cargo/config.toml
@@ -1,2 +1,2 @@
 [build]
-rustflags = ["-Clinker=clang++-14", "-Clink-arg=-fuse-ld=lld"]
+rustflags = ["-Clinker=clang++", "-Clink-arg=-fuse-ld=lld"]
diff --git a/.clangd b/.clangd
index ca1f2dc..0a48410 100644
--- a/.clangd
+++ b/.clangd
@@ -1,3 +1,3 @@
 CompileFlags:
-  Add: -fcoroutines-ts
+  Add: -fcoroutines
   Remove: [-fcoroutines, -DNO_INTELLISENSE]
diff --git a/CMakeLists.txt b/CMakeLists.txt
index b997361..d71653e 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -55,11 +55,13 @@ set(CMAKE_POSITION_INDEPENDENT_CODE ON)
 set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
 
 if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
-    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines-ts")
-    set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -latomic")
+    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines")
+    set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -latomic -lm -lstdc++")
     add_link_options(-fuse-ld=lld)
     # Do not build with libc++ (LLVM's implementation of the C++ standard library) in fdb
     set(USE_LIBCXX OFF)
+    add_compile_options(-Wno-unused-but-set-variable)
+    add_compile_options(-Wno-unused-command-line-argument)
 elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
     set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines")
 endif()
diff --git a/cmake/CLangFormat.cmake b/cmake/CLangFormat.cmake
index c74b3dd..4374329 100644
--- a/cmake/CLangFormat.cmake
+++ b/cmake/CLangFormat.cmake
@@ -1,4 +1,8 @@
-set(CLANG_FORMAT "/usr/bin/clang-format-14")
+find_program(CLANG_FORMAT
+NAMES clang-format clang-format-14 clang-format-15
+PATHS /usr/bin /usr/local/bin
+DOC "Path to clang-format executable"
+NO_ERROR)
 if(EXISTS ${CLANG_FORMAT})
     message(STATUS "Found clang-format at ${CLANG_FORMAT}")
diff --git a/cmake/CLangTidy.cmake b/cmake/CLangTidy.cmake
index 6f387b0..708cf5a 100644
--- a/cmake/CLangTidy.cmake
+++ b/cmake/CLangTidy.cmake
@@ -1,7 +1,7 @@
 # clang-tidy generates too many warnings, so just disable it by default.
 option(ENABLE_CLANG_TIDY "Run clang-tidy during build" OFF)
 
-find_program(CLANG_TIDY NAMES clang-tidy-14)
+find_program(CLANG_TIDY NAMES clang-tidy)
 if(CLANG_TIDY)
     if(CMake_SOURCE_DIR STREQUAL CMake_BINARY_DIR)
         message(FATAL_ERROR "CMake_RUN_CLANG_TIDY requires an out-of-source build!")
@@ -42,11 +42,11 @@ if(CLANG_TIDY)
     # Taking a quick route for now. We should deal with it sometime down the line.
     add_custom_target(clang-tidy
         COMMENT "Running clang-tidy"
-        COMMAND run-clang-tidy-14 -header-filter ${HEADER_FILTER} `find ${SOURCE_DIRS} -name "*.cc" -o -name "*.cpp" -not -name "*.actor.cpp" ` -quiet)
+        COMMAND run-clang-tidy -header-filter ${HEADER_FILTER} `find ${SOURCE_DIRS} -name "*.cc" -o -name "*.cpp" -not -name "*.actor.cpp" ` -quiet)
 
     add_custom_target(clang-tidy-fix
         COMMENT "Running clang-tidy -fix"
-        COMMAND run-clang-tidy-14 -header-filter ${HEADER_FILTER} `find ${SOURCE_DIRS} -name "*.cc" -o -name "*.cpp" -not -name "*.actor.cpp" ` -fix -quiet)
+        COMMAND run-clang-tidy -header-filter ${HEADER_FILTER} `find ${SOURCE_DIRS} -name "*.cc" -o -name "*.cpp" -not -name "*.actor.cpp" ` -fix -quiet)
 else()
-    message(WARNING "clang-tidy-14 not found!!!")
+    message(WARNING "clang-tidy not found!!!")
 endif()
\ No newline at end of file
diff --git a/setup.py b/setup.py
index ef264d1..6fc49d0 100644
--- a/setup.py
+++ b/setup.py
@@ -45,8 +45,8 @@ class CMakeBuild(build_ext):
             f"-DCMAKE_LIBRARY_OUTPUT_DIRECTORY={extdir}",
             f"-DPYTHON_EXECUTABLE={sys.executable}",
             f"-DCMAKE_BUILD_TYPE={cfg}",  # not used on MSVC, but no harm
-            "-DCMAKE_CXX_COMPILER=clang++-14",
-            "-DCMAKE_C_COMPILER=clang-14",
+            "-DCMAKE_CXX_COMPILER=clang++",
+            "-DCMAKE_C_COMPILER=clang",
             "-DCMAKE_EXPORT_COMPILE_COMMANDS=ON",
             "-DUSE_RTTI=ON",
             "-DOVERRIDE_CXX_NEW_DELETE=OFF",
diff --git a/src/fuse/CMakeLists.txt b/src/fuse/CMakeLists.txt
index eaaeb9c..89b938c 100644
--- a/src/fuse/CMakeLists.txt
+++ b/src/fuse/CMakeLists.txt
@@ -1,8 +1,10 @@
-if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
-    link_directories(/usr/local/lib/x86_64-linux-gnu/ /usr/lib64 /usr/local/lib64)
-elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
-    link_directories(/usr/local/lib/aarch64-linux-gnu/ /usr/lib64 /usr/local/lib64)
-endif()
+#if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
+#    link_directories(/usr/local/lib/x86_64-linux-gnu/ /usr/lib64 /usr/local/lib64)
+#elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
+#    link_directories(/usr/local/lib/aarch64-linux-gnu/ /usr/lib64 /usr/local/lib64)
+#endif()
+link_directories(/usr/lib64 /usr/local/lib64)
+
 
 target_add_lib(hf3fs_fuse common core-app meta-client storage-client fuse3 client-lib-common)
 target_add_bin(hf3fs_fuse_main hf3fs_fuse.cpp hf3fs_fuse)
-- 
2.33.0
```

# folly代码修改
```
From b8b1c25032058a30d87dbdb18a0c50f38169e68a Mon Mar 24 00:00:00 2025
From: wangzengliang <wangzengliang2@huawei.com>
Date: Tue, 25 Mar 2025 11:40:05 +0800
Subject: [PATCH] m

---
 folly/experimental/StampedPtr.h | 12 ++++++++++++
 1 file changed, 12 insertions(+)

diff --git a/folly/experimental/StampedPtr.h b/folly/experimental/StampedPtr.h
index 668649e2c..dec29be83 100644
--- a/folly/experimental/StampedPtr.h
+++ b/folly/experimental/StampedPtr.h
@@ -79,6 +79,17 @@ struct StampedPtr {
 
   void setStamp(uint16_t stamp) { raw = pack(unpackPtr(raw), stamp)
 }
 
+#if defined(__aarch64__) || defined(__ARM_ARCH)
+  static uint64_t* unpackPtr(uint64_t raw) {
+    uint64_t shifted_value

+      asm (
+        "lsr %0, %1, #16"
+        : "=r" (shifted_value)
+        : "r" (raw), "r" (kInternalStampBits)
+      )

+      return reinterpret_cast<uint64_t*>(static_cast<intptr_t>(shifted_value))

+  }
+#else
   static T* unpackPtr(uint64_t raw) {
     // Canonical form means we need to extend bit 47 of the pointer to
     // bits 48..63 (unless the operating system never hands those pointers
@@ -88,6 +99,7 @@ struct StampedPtr {
     auto extended = static_cast<int64_t>(raw) >> kInternalStampBits

     return reinterpret_cast<T*>(static_cast<intptr_t>(extended))

   }
+#endif
 
   static uint16_t unpackStamp(uint64_t raw) {
     return static_cast<uint16_t>(raw)

-- 
2.33.0
```

# 编译3FS
cmake -S . -B build -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
cmake --build build -j 32

# 部署3FS
参考官方文档
https://github.com/deepseek-ai/3FS/blob/main/deploy/README.md
https://mp.weixin.qq.com/s/Ri9-sjsaoQ3LUE2-dTAN_g
