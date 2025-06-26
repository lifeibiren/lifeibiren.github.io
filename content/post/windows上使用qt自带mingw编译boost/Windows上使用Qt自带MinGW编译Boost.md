---
title: "Windows上使用Qt自带MinGW编译Boost"
date: 2025-06-27T00:24:02+08:00
---

# 环境
MinGW: 7.3.0 64-bit
Boost: 1.79.0

# 步骤

1.首先解压 boost 源代码到任意目录中

2.打开 cmd，并 cd 到 boost 源代码所在目录

3.临时设置环境变量 PATH 为 MinGW 的目录

```
SET PATH=C:\Qt\Qt5.14.2\Tools\mingw730_64\bin
```

4.直接执行 bootstrap.bat 发现指定的 TOOLSET 并没有起作用。因此需要稍微修改一下

![20250626223432.png](/post/windows上使用qt自带mingw编译boost/20250626223432.png)

5.执行 bootstrap.bat 并生成 b2.exe
```
.\bootstrap.bat gcc
```

6.使用 b2.exe 编译
```
.\b2.exe gcc
```