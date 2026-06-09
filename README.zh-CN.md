# AlibabaProtect / AliPaladin 导致 Windows 重启后内存异常偏高

[English](README.en.md) | 中文

一次真实的 Windows 内存排障记录。

我遇到的问题是：

- 机器内存为 `16GB`
- 刚重启后，总内存占用接近 `90%`
- 任务管理器里普通用户进程并不大
- `RAMMap` 里 `Nonpaged Pool`、`Paged Pool`、`Page Table` 明显偏高

最终定位到根因并不是普通前台软件，而是阿里系保护组件：

- `AlibabaProtect`
- `AliPaladin`

其中 `AliPaladin` 是一个文件系统过滤驱动，`AlibabaProtect` 是配套服务。两者异常运行时，会把大量占用堆到内核池，导致“用户进程看起来不大，但整机内存几乎被吃满”的现象。

## TL;DR

如果你遇到这些现象：

- 刚开机内存就很高
- 任务管理器里大进程不多
- `RAMMap` 的 `Nonpaged Pool` / `Paged Pool` / `Page Table` 偏高
- 装过阿里系客户端或带安全保护组件的软件

可以重点检查：

- `AlibabaProtect`
- `AliPaladin`

## 这次问题的关键表现

### 1. 普通进程解释不了总内存

前台程序和普通后台程序加起来，不足以解释接近满内存的状态。

### 2. RAMMap 把方向指向了内核层

异常项主要在：

- `Nonpaged Pool`
- `Paged Pool`
- `Page Table`

这说明问题更像是驱动、过滤器、第三方保护组件，而不是浏览器或聊天软件。

### 3. 系统日志显示服务异常

`AlibabaProtect` 在系统日志里反复出现：

- 服务响应超时
- 服务控制请求不响应

### 4. 找到了更底层的驱动

继续查服务和驱动后，发现系统里还有：

- `AliPaladin`
- 类型：`FILE_SYSTEM_DRIVER`
- 组：`FSFilter Activity Monitor`

这说明它不是普通后台，而是文件系统过滤驱动。

## 根因判断

这次问题的根因可以概括为：

1. `AlibabaProtect` 是阿里系软件带来的安全/反篡改/环境检测组件
2. `AliPaladin` 是其配套的文件系统过滤驱动
3. 两者在当前机器上运行异常
4. 异常把内核池占用明显抬高
5. 最终表现为“重启后内存异常偏高”

## 这套组件大概在做什么

从文件名和驱动类型看，它们通常会做：

- 进程扫描
- 文件监控
- 反注入
- 环境校验
- 风险检测
- 反篡改保护

目录里能看到类似文件：

- `AntiInject.dll`
- `FileMonitor.dll`
- `ProcessScanner.dll`
- `CloudDetector.dll`
- `ThreatSieveSDK.dll`

它们本身不一定总是有问题，但一旦状态异常、兼容性出问题，内核池占用就可能涨得很夸张。

## 我是怎么解决的

真正有效的动作不是只关掉一个前台进程，而是同时处理两层：

- `AlibabaProtect`
- `AliPaladin`

处理思路是：

1. 先确认普通进程占用无法解释总内存
2. 用 `RAMMap` 确认是内核池方向的问题
3. 从系统日志确认 `AlibabaProtect` 异常
4. 找到更底层的 `AliPaladin`
5. 将两者都改为 `Disabled`
6. 删除对应服务项
7. 重启系统，让已加载的服务和驱动真正退出

## 为什么内存会一下掉很多

因为真正回收掉的，不只是 `AlibabaProtect.exe` 本身那一点工作集，而是整条链带来的系统级开销：

- 服务层
- 驱动层
- 过滤器链路
- 页表和映射开销

这是为什么一个看起来只有几百 MB 的异常服务，最后能让整机多吃掉几 GB 内存。

## 本次处理后的结果

复查结果是：

- `AlibabaProtect` 服务不再存在
- `AliPaladin` 驱动服务不再存在
- `AlibabaProtect.exe` 不再运行
- 内核池占用明显回落

在这台机器上的实际变化大致是：

- 可用内存回升到 `8GB+`
- `Nonpaged Pool` 从约 `1.7GB` 降到约 `785MB`
- `Paged Pool` 从约 `1.0GB` 降到约 `480MB`

## 可用命令

查看服务：

```powershell
Get-Service | Where-Object { $_.Name -match 'Ali|Alibaba' -or $_.DisplayName -match 'Ali|Alibaba' }
```

查询服务配置：

```powershell
sc.exe qc AlibabaProtect
sc.exe qc AliPaladin
```

查询服务状态：

```powershell
sc.exe query AlibabaProtect
sc.exe query AliPaladin
```

查看内核池：

```powershell
Get-Counter '\Memory\Pool Nonpaged Bytes','\Memory\Pool Paged Bytes'
```

查看最近服务控制日志：

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Service Control Manager'} -MaxEvents 100
```

## 注意事项

- 不建议盲删驱动或服务
- 先确认问题是否真的来自这条链
- 某些服务在线不可停止，必须靠“禁用后重启”完成最终卸载
- 任务管理器不够看时，优先结合 `RAMMap` 和系统日志判断

## 完整记录

更完整的排查过程见：

- [`docs/case-study.zh-CN.md`](docs/case-study.zh-CN.md)

英文版本：

- [`README.en.md`](README.en.md)
- [`docs/case-study.en.md`](docs/case-study.en.md)

## License

本文档以 `CC BY 4.0` 方式共享，详情见 `LICENSE`。
