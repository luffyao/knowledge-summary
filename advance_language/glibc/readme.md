# 一. GLIBC 介绍 (GNU C Library)

是 GNU 项目实现的标准 C 运行时库。

**包括：**

- C 标准库（libc）的 GNU 实现
- Linux 用户态最核心的基础库
- 几乎所有 C/C++ 程序的运行依赖

在大多数 Linux 发行版（如 Ubuntu、Debian、Red Hat Enterprise Linux）中，默认 C 库就是 glibc。

## 有哪些 GLIBC 的替代

- **musl**

## 1.1 GLIBC 作用是什么？

GNU libc 实现：

- **系统调用的隔离层**
- **用户态资源管理**（malloc、pthread、stdio 缓存等都是 glibc 实现）
- **语言层面的标准规范**（ISO C 标准，POSIX 标准）
- **ABI 稳定性**

简单点就是：**glibc 是用户态运行时环境 + 系统调用封装层 + 标准库实现。**

## GLIBC 所在层级

```
应用程序 → glibc → 系统调用 → Linux 内核 → 硬件
```
