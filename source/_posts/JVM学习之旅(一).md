---
title: JVM学习之旅(一)
date: 2020-03-22 18:41:33
tags: [Java]
---


# 写在前面的话

蛋疼了这么久，终于又开始了新一年的学习了，目前也在搞蛋疼的2020年KPI的规划，有句港句，我觉得挺傻逼的。大公司KPI毛病一如既往...

本章主要做的事情就是在我的电脑上把jvm的编译调试环境搭起来了，主要还是借鉴各种网上的经验吧...

# 走进JAVA

## 自己编译JDK

- 首先下载并解压源码

可以到某度网盘上去下载，更快 


- 编译

设定好对应的环境变量

```
export LANG=C
# JDK安装路径
export ALT_BOOTDIR=/Library/Java/JavaVirtualMachines/open_jdk8/Contents/Home
# Mac平台，C编译器不再是GCC，是clang
export CC=clang
# 跳过clang的一些严格的语法检查，不然会将N多的警告作为Error
export COMPILER_WARNINGS_FATAL=false
# 链接时使用的参数
export LFLAGS='-Xlinker -lstdc++'
# 是否使用clang
export USE_CLANG=true
# 使用64位数据模型
export LP64=1
# 告诉编译平台是64位，不然会按32位来编译
export ARCH_DATA_MODEL=64
# 允许自动下载依赖
export ALLOW_DOWNLOADS=true
# 并行编译的线程数，编译时间长，为了不影响其他工作，我选择为2
export HOTSPOT_BUILD_JOBS=2
# 是否跳过与先前版本的比较
export SKIP_COMPARE_IMAGES=true
# 是否使用预编译头文件，加快编译速度
export USE_PRECOMPILED_HEADER=true
# 是否使用增量编译
export INCREMENTAL_BUILD=true
# 编译内容
export BUILD_LANGTOOLS=true
export BUILD_JAXP=true
export BUILD_JAXWS=true
export BUILD_CORBA=true
export BUILD_HOTSPOT=true
export BUILD_JDK=true
# 编译版本
export SKIP_DEBUG_BUILD=true
export SKIP_FASTDEBUG_BUILD=false
export DEBUG_NAME=debug
# 避开javaws和浏览器Java插件之类的部分的build
export BUILD_DEPLOY=false
export BUILD_INSTALL=false
unset JAVA_HOME
```

如果在./configure时有检查报错，则去common/autoconf/generated-configure.sh下移除掉对应的检测(一般都是过度检测)

最后`bash ./configure`和`make all`二连开始编译jdk

编译完成后，需要将编译出来的jdk放入环境变量中;

由于现在基本都使用的idea，就直接在idea中增加自己编译出来的jdk，并且让项目开始使用新jdk

为了表明是自己新的jdk，在jdk/java.c中的JAVAMAIN方法中新增加了自己的printf方法，最终打印如下

![jvm结果](/imgs/learnjvm_1.png)

到这里，JVM的学习之旅算是正式开始了