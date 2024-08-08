## cmake 学习框架

### Q. cmake CMakeLists.txt 文件的基本结构

```bash
# Filename: CMakeLists.txt
# 下方标记 * 的注释为必须的 

# * 指定 Cmake 的最低版本要求 这里指定为 3.10
cmake_minimum_required(VERSION 3.10)

# * 指定项目名称以及项目的版本号 
project(Tutorial VERSION 1.0)

# 设置 C/C++ 项目的标准版本
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)

# * 当前项目是个可执行程序 配置项目的执行程序文件名，以及构建可执行程序依赖的源文件
add_executable(Tutorial tutorial.cxx)

# 构建可执行程序依赖的头文件搜索目录地址 
target_include_directories(Tutorial PUBLIC "${PROJECT_BINARY_DIR}")

# 可以打印 Cmake 构建过程的消息，辅助观测构建过的相关变量
message(STATUS "MySelf [${PROJECT_BINARY_DIR}]")
```

### Q. cmake 项目多个源文件，添加可执行程序源文件时快速扫描源文件列表

```bash
# Filename: CMakeLists.txt

# 通过 aux_source_diretory(path VAR) 命令扫描到自己定义的变量中
# 使用变量的方法是在使用的地方 ${VAR_NAME} , VAR_NAME 就是变量名
# . 代表当前目录
aux_source_diretory(. SRC_FILE_LIST)

```

### Q. 将项目编译为动态链接库 or 静态链接库

```bash
# Filename: CMakeLists.txt

# 将 add_executable 修改为 add_library(LIB_NAME STATIC source_file1 [source_file2] ...)

# 1. 编译为静态链接库
add_library(LIB_NAME STATIC source_file1.cpp)
# or 或者省略掉 STATIC 例: add_library(LIB_NAME source_file1.cpp) 

# 2. 编译为动态链接库
add_library(LIB_NAME SHARED source_file1.cpp)
```

### Q. cmake 中如何实现条件编译，传递参数给代码源文件

```bash
# Filename: CMakeLists.txt

# 首先在修改根目录下的 CMakeLists.txt 文件

# 添加配置文件，生成 C/C++ 配置头文件
# *.h.in 使 cmake 配置文件
# .h 是要生成的头文件,C/C++ 源码 include 使用
# ${PROJECT_BINARY_DIR} cmake 内置变量，build 目录，或构建指定的目录地址
# ${PROJECT_SOURCE_DIR} 项目的根目录目录地址
# 下方示例，在根目录下根据 config.h.in 文件在根目录下生成 config.h 头文件
configure_file (
  "${PROJECT_SOURCE_DIR}/config.h.in"
  "${PROJECT_SOURCE_DIR}/config.h"
)

# cmake option(VAR_NAME "选项描述文字" 默认值) 默认值一般采用 ON/OFF YES/NO
# 选其一种方案即可，大量项目采用 ON/OFF
# 下方示例中，定义了一个 USE_SOCKT 的变量确定开启 socket 链接功能，默认 ON 是开启
option(USE_SOCKT "enable socket connection" ON)


# 在根目录下新建 config.h.in 文件
# configure_file 主要实现如下两个功能:
# 1. 将 <input> 文件里面的内容全部复制到 <output> 文件中
# 2. 根据参数规则，替换 @VAR@ 或 ${VAR} 变量

# 在根目录下新建 config.h.in 文件,根据需要选择两种方法
# 1. cmakedefine 会根据变量的值是否为真（类似 if）来变换为 #define VAR ... 或  #undef VAR
#cmakedefine CMAKEDEFINE_VAR1 @CMAKEDEFINE_VAR1@
#cmakedefine CMAKEDEFINE_VAR2 @CMAKEDEFINE_VAR2@

# 2. define 会直接根据规则来替换
#define DEFINE_VAR1 @DEFINE_VAR1@
#define DEFINE_VAR2 ${DEFINE_VAR2}

# 在代码中就可以使用

#ifdef DEFINE_VAR1

	code...

#enfif

```

