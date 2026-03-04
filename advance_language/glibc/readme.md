# GLIBC 介绍（GNU C Library）

## 是什么

**GLIBC**（GNU C Library）是 GNU 项目实现的 **标准 C 运行时库**，也是 Linux 用户态最核心的基础库之一。

- 提供 **C 标准库（libc）** 的完整实现（ISO C89/C99/C11/C17 等）
- 绝大多数 Linux 发行版（Ubuntu、Debian、RHEL、Fedora 等）的 **默认 C 库**
- 几乎一切 C/C++ 用户态程序（包括其他语言运行时）都依赖它：系统调用、内存分配、线程、I/O、字符串与数学函数等，最终大多由 glibc 提供或封装

可以简单理解为：**glibc = 用户态运行时环境 + 系统调用封装层 + C/POSIX 标准库实现。**

---

## 在系统中的位置

```
应用程序 / 其他库
       ↓
   glibc (libc.so, libm.so, libpthread 等)
       ↓
  系统调用接口 (syscall)
       ↓
  Linux 内核 → 硬件
```

glibc 夹在「应用程序与各类库」和「内核」之间：上层通过它使用标准 API，它再把请求转化为系统调用或内核提供的机制（如 futex、mmap）。

---

## 主要职责与特性

### 1. 系统调用的隔离与封装

- 应用程序不直接写 `syscall`，而是调用 **glibc 提供的包装函数**（如 `open`、`read`、`mmap`、`clone`）
- glibc 负责：参数与调用约定、错误号与 `errno`、取消点与信号处理等细节
- 不同架构（x86_64、aarch64 等）的差异被封装在 glibc 内部，应用只需面对统一 API

### 2. C 标准库实现

实现 ISO C 标准规定的头文件与函数，例如：

| 领域     | 头文件/模块   | 典型内容                     |
|----------|----------------|------------------------------|
|  I/O     | stdio.h        | printf、scanf、FILE、缓冲    |
| 内存/工具| stdlib.h       | malloc/free、atoi、exit      |
| 字符串   | string.h       | strcpy、memcpy、strlen       |
| 数学     | math.h (libm)  | sin、sqrt、浮点运算          |
| 时间     | time.h         | time、clock、strftime        |
| 字符分类 | ctype.h        | isalpha、toupper             |
| 其他     | assert、setjmp、stdarg 等 | 断言、非局部跳转、可变参数 |

这些是「可移植 C 程序」的基础，与平台无关的代码都依赖它们。

### 3. POSIX 与扩展 API

除纯 C 标准外，glibc 还实现大量 **POSIX** 及常见扩展：

- **线程**：**NPTL**（Native POSIX Thread Library），提供 `pthread_create`、mutex、条件变量、线程局部存储等，是 Linux 上多线程的默认实现
- **实时与时间**：`clock_nanosleep`、timer、POSIX 信号量、消息队列等
- **文件与 I/O**：`openat`、`readv`、`epoll` 封装、`sendfile` 等
- **进程与信号**：`fork`、`exec`、信号处理、`sigaction` 等
- **网络与 NSS**：DNS/名称解析（Name Service Switch）、`getaddrinfo` 等依赖 glibc 的 NSS 与解析器

这些构成了「Linux 上 C/C++ 系统编程」的日常 API。

### 4. 用户态资源管理

很多「资源」虽然最终由内核管理，但接口与策略在 glibc 里实现：

- **堆内存**：**malloc / free / realloc**（ptmalloc 实现），见 [1.2.1 malloc 机制](./1.2.1%20malloc.md)
- **线程与同步**：线程创建、栈、TLS、mutex/cond 的 futex 封装
- **stdio 缓冲**：FILE 的缓冲、行缓冲/全缓冲、`setvbuf` 等
- **动态加载**：`dlopen`、`dlsym`（与动态链接器 `ld.so` 配合，后者常随 glibc 一起发布）

### 5. 国际化与本地化（i18n）

- **locale**：`setlocale`、`LC_*`、多字节与宽字符转换
- **gettext**：消息翻译与多语言支持的基础设施

### 6. ABI 稳定性与符号版本

- glibc 在升级时尽量保持 **ABI 兼容**：老二进制在不重新编译的情况下仍能运行
- 通过 **符号版本（symbol versioning）** 区分同一符号在不同版本中的实现，避免新老程序冲突
- 这对发行版和第三方二进制软件至关重要

---

## 常见「库文件」与 glibc 的关系

| 库名        | 说明 |
|-------------|------|
| libc.so.6   | 主 C 库：C 标准 + POSIX 核心 + 系统调用封装 + NPTL（线程）等 |
| libm.so.6   | 数学库（math.h），常单独链接 `-lm` |
| libpthread  | 现代 glibc 中线程已并入 libc，保留 libpthread 多为兼容性，链接 `-lpthread` 会带出线程符号 |
| librt       | 实时扩展（部分接口已并入 libc） |
| libdl       | 动态加载（`dlopen`/`dlsym`），部分已并入 libc |
| ld-linux-*  | 动态链接器，负责加载可执行文件与依赖库，通常随 glibc 包提供 |

---

## GLIBC 的替代与对比

在嵌入式、安全敏感或追求精简的场景下，会使用其他 C 库：

| 替代库     | 典型特点 |
|------------|----------|
| **musl**   | 体积小、静态链接友好、代码清晰、对 POSIX 的「可选」扩展较少；常用于 Alpine、静态二进制、容器 |
| **uClibc-ng** | 面向嵌入式，可裁剪、资源占用少 |
| **Bionic** | Android 专用，与 Linux 内核接口配合，非 GNU 生态 |
| **diet libc** | 极简，适合小体积可执行文件 |

与 glibc 相比：glibc 功能最全、与 Linux 发行版和各类软件兼容性最好，但代码体量大、历史包袱多；musl 等更精简，但在某些扩展或二进制兼容性上可能与「为 glibc 构建」的软件有差异。

---

## 本目录内容

- **[1.2.1 malloc 机制](./1.2.1%20malloc.md)**：glibc 堆分配器（ptmalloc）的实现要点、与其它 allocator 的对比、现代分配器设计小结。
- **[1.2.2 线程管理](./1.2.2%20thread_mgm.md)**：glibc/NPTL 线程管理（待完善）。
- **[1.2.3 程序启动与对象初始化流程](./1.2.3%20object_start_flow.md)**：从 _start 到 main 的启动链、.preinit_array / _init / .init_array 等初始化顺序与作用，小白向说明。
- **[1.2.4 线程本地存储（TLS）](./1.2.4%20thread_local_storage.md)**：TLS 是什么、为什么需要、在 glibc 中的实现与用法（__thread、pthread_key），小白向说明。

---

## 参考

- [glibc 官网](https://www.gnu.org/software/libc/)
- [glibc 手册](https://www.gnu.org/software/libc/manual/)
- [Linux man pages](https://man7.org/linux/man-pages/)（多数描述的是 glibc 提供的接口行为）
- [glibc 主页](https://sourceware.org/glibc/wiki/HomePage)
