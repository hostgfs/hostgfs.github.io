---
title: Go+FFmpeg偶发crash修复
date: 2025-06-12 15:00:00 +0800
categories: [后端开发, 音视频]
tags: [Go, FFmpeg, cgo, 静态编译, 崩溃排查]
---

## 背景

2025年6月4日晚上，线上媒体转码服务（media-transcoder）偶现 crash，影响了部分用户体验。crash 调用链为 Go → cgo → C++ → FFmpeg。从 Go 层 crash 堆栈来看，崩溃发生在 cgo 调用的底层 C++ 逻辑。

## 排查过程

### 1. 初步定位
- 通过 `addr2line` 和 `gdb`，结合带调试信息的二进制文件，初步定位到 Go 层的 `DoTranscode` 函数。
- 检查 C++ 层该函数实现，未发现明显的崩溃点，怀疑问题出在更底层的 FFmpeg。

### 2. Native 层堆栈捕获
- 由于 Go 层无法获得更多有用信息，决定在 C++ native 层增加信号捕获，打印崩溃时的堆栈。
- 捕获到的崩溃堆栈如下：

```
=== Crash Backtrace ===
#0  print_backtrace
#1  signal_handler
#2  __restore_rt
#3  gaih_inet.constprop.7
#4  getaddrinfo
#5  tcp_open
#6  ffurl_connect
#7  ffurl_open_whitelist
#8  rtmp_open
#9  ffurl_connect
#10 ffurl_open_whitelist
#11 ffio_open_whitelist
...
#17 DoTranscode
#18 _cgo_Cfunc_DoTranscode
#19 runtime.asmcgocall
```

### 3. 问题分析
- 崩溃点在 FFmpeg 的 `getaddrinfo` 调用。
- 由于 media-transcoder 采用**静态编译**，而 `getaddrinfo` 依赖 glibc 的运行时库（glibc 通过 libnss 依赖本地配置，不能完全静态链接）,编译时就有警告:
  - /usr/bin/ld: /tmp/go-link-2930670155/000006.o: in function `_cgo_77133bf98b3a_C2func_getaddrinfo':
/tmp/go-build/cgo_unix_cgo.cgo2.c:60:(.text+0x37): warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
- 这导致在某些网络操作场景下，静态编译的可执行文件调用 `getaddrinfo` 时可能崩溃。

## 结论

静态链接的 Go + cgo + FFmpeg 项目，在网络相关操作（如 RTMP 拉流、推流）时，因 glibc 的 `getaddrinfo` 不能完全静态链接，可能导致偶发崩溃。

## 修复方案

1. **采用动态链接方式编译 media-transcoder**  
   - 优点：功能最完备，FFmpeg 能力全开。
   - 缺点：可能带来部署兼容性问题（需保证运行环境有合适的动态库）。
   - 适用场景：容器化部署时推荐，理论上环境可控。

2. **使用 musl libc 替代 glibc 编译**  
   - 优点：musl 支持完全静态链接，兼容性更好。
   - 风险：musl 与 glibc 在某些行为上有差异，需充分测试。

3. **去掉相关模块的兜底设计**  
   - 优点：规避问题代码路径。
   - 缺点：可能带来功能不兼容，需确保音频解码（如 AAC LC/HE/HEv2）能被正确处理。

## 总结

本次线上 crash 的根因在于 glibc 静态链接的局限性，尤其是在涉及网络相关的系统调用时。建议在生产环境中优先采用动态链接或 musl 静态链接，并针对音视频转码等高风险路径做好异常捕获和降级处理。

---

**经验教训：**
- 静态编译并非银弹，涉及 cgo 和第三方库时需关注运行时依赖。
- 线上问题排查可结合多层堆栈捕获（Go 层 + native 层），提升定位效率。
- 生产环境建议开启core dump 或完善 native 层异常日志，便于快速定位问题。

---
