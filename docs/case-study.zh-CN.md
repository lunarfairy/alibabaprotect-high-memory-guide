# Windows 电脑重启后内存占用异常高：一次 AlibabaProtect / AliPaladin 排查实录

## 概述

这篇文档记录了一次真实的 Windows 内存异常排查过程。

现象是：

- 机器为 `16GB` 内存
- 刚重启后，总内存占用接近 `90%` 甚至更高
- 任务管理器里普通用户进程加起来并不大
- `RAMMap` 里 `Nonpaged Pool`、`Paged Pool`、`Page Table` 异常偏高

最终定位到问题根因并不是某个普通前台程序，而是阿里系安全组件：

- `AlibabaProtect`
- `AliPaladin`

其中 `AliPaladin` 是一个文件系统过滤驱动，`AlibabaProtect` 是对应的保护服务。两者一起工作时，一旦异常，可能会把大量内存堆到内核池里，导致“用户进程看起来不大，但整机内存几乎被吃满”的现象。

## 典型症状

如果你遇到以下情况，这个案例值得参考：

- 电脑刚重启就显示内存占用很高
- 任务管理器“用户”页签只显示少量内存
- 普通应用进程加起来远小于总内存占用
- `RAMMap` 中这些项目异常偏高：
  - `Nonpaged Pool`
  - `Paged Pool`
  - `Page Table`
- 某些第三方安全组件、反篡改组件、驱动级保护程序常驻

## 为什么任务管理器看不明白

这是这类问题最容易误导人的地方。

普通应用主要占用的是用户态内存，而 `AlibabaProtect` / `AliPaladin` 这类组件会把一部分占用放到内核层，尤其是：

- `Nonpaged Pool`
- `Paged Pool`
- 驱动映射和页表相关开销

因此你会看到：

- “用户”页签看起来只用了几百 MB 到 1~2GB
- 但总内存占用已经冲到十几 GB

这不是任务管理器显示错误，而是统计口径不同。

## 本次排查的关键信号

### 1. 普通进程占用解释不了总内存

按进程统计后，普通用户进程虽然不少，但不足以解释接近满内存的现象。

### 2. RAMMap 暴露了真正的问题方向

在 `RAMMap -> Use Counts` 中，最值得关注的是：

- `Page Table` 偏高
- `Paged Pool` 偏高
- `Nonpaged Pool` 偏高

这说明问题更像是：

- 驱动
- 文件系统过滤器
- 网络过滤器
- 第三方安全组件

而不是浏览器、聊天软件、编辑器这种普通应用。

### 3. 系统日志出现服务异常

系统日志中，`AlibabaProtect` 反复出现：

- 服务响应超时
- 服务控制请求不响应

这说明它不是一个安静待着的后台，而是一个已经异常运行的保护组件。

### 4. 找到了底层驱动 `AliPaladin`

继续排查注册表和服务配置后，发现除了 `AlibabaProtect` 外，还有一个更底层的驱动：

- 服务名：`AliPaladin`
- 类型：`FILE_SYSTEM_DRIVER`
- 组：`FSFilter Activity Monitor`

这很关键。因为这意味着它不是单纯的用户态程序，而是插在文件系统路径上的过滤驱动。用户态服务出问题时，驱动层往往会把问题放大。

## 根因判断

这次问题的根因可以概括为：

1. `AlibabaProtect` 是阿里系软件带来的安全/反篡改/环境检测组件
2. `AliPaladin` 是其配套的文件系统过滤驱动
3. 两者在当前机器上运行异常
4. 异常造成内核池占用膨胀
5. 最终表现为“重启后内存占用异常高”

更直白一点说：

不是单个前台软件吃掉了 16GB 内存，而是驱动和保护服务把内核内存顶上去了。

## 这套组件大概在做什么

从文件名可以大致推断其职责包括：

- 进程扫描
- 文件监控
- 反注入
- 环境校验
- 风险检测
- 反篡改保护

在安装目录里能看到类似文件：

- `AntiInject.dll`
- `FileMonitor.dll`
- `ProcessScanner.dll`
- `CloudDetector.dll`
- `ThreatSieveSDK.dll`

这类组件本身不一定有问题，但一旦出现驱动冲突、循环重试、状态异常或与其他系统组件兼容性不好，就可能把内核池占用抬得很夸张。

## 我们是怎么解决的

### 第一步：确认不是普通应用问题

先统计进程内存，发现普通程序加起来解释不了总占用，于是把重点转向内核层。

### 第二步：确认 `AlibabaProtect` 异常

检查服务状态和系统日志，发现：

- `AlibabaProtect` 自动启动
- 服务长期超时
- 控制请求不响应

### 第三步：找到 `AliPaladin`

检查服务和驱动项后，发现 `AliPaladin` 正在运行，而且是文件系统过滤驱动。

### 第四步：同时处理服务层和驱动层

这里只关掉 `AlibabaProtect` 是不够的，因为驱动层仍可能把它拉起来。

真正有效的做法是同时处理：

- `AlibabaProtect`
- `AliPaladin`

具体动作包括：

- 将两者启动类型改为 `Disabled`
- 删除对应服务项
- 重启系统，让当前已加载的驱动和服务真正从内存中退出

### 第五步：重启后验收

重启后复查：

- `AlibabaProtect` 不再存在为已安装服务
- `AliPaladin` 不再存在为已安装服务
- `AlibabaProtect.exe` 不再运行
- 内核池占用明显下降

## 为什么内存会一下掉这么多

这是很多人第一次遇到这类问题时最困惑的一点。

原因是，我们清掉的不是一个“只有几百 MB 的前台应用”，而是一条影响内核池的保护链：

- 服务层
- 驱动层
- 过滤器链路
- 页表和映射开销

因此一旦问题源头被拔掉，回落的就不只是 `AlibabaProtect.exe` 本身那一点工作集，而是整条链路带来的系统级开销。

这也是为什么前后内存差值会非常大。

## 风险与注意事项

### 1. 不建议盲删驱动或服务

先确认这是不是你真正的问题来源。不是所有高内存都与 `AlibabaProtect` 有关。

### 2. 不要只看任务管理器

请结合这些工具判断：

- 任务管理器
- `RAMMap`
- 系统日志
- 服务列表
- 驱动列表

### 3. 只改服务不一定够

如果配套驱动仍在，服务可能被重新拉起。

### 4. 有些服务在线不可停止

某些服务或驱动会显示：

- `NOT_STOPPABLE`
- `NOT_PAUSABLE`

这时不是命令无效，而是它当前会话里已经加载，必须靠“禁用后重启”来完成最终卸载。

## 推荐排查流程

如果你也遇到了类似问题，可以按这个顺序排查：

1. 看总内存、可用内存、前台大进程
2. 用 `RAMMap` 看 `Nonpaged Pool` / `Paged Pool` / `Page Table`
3. 看系统日志里是否有异常服务超时
4. 查是否存在对应的保护服务和驱动
5. 同时处理服务层和驱动层
6. 重启后复查

## 可用命令示例

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

## 最终结论

这次问题并不是“16GB 内存太小”，也不是“某个普通应用开太多”。

更准确的结论是：

- 阿里系保护服务 `AlibabaProtect`
- 配套文件系统过滤驱动 `AliPaladin`

在这台机器上异常运行，导致内核池占用明显升高，最终表现为重启后内存占用异常偏高。

把这条链从启动链和驱动链里拔掉之后，内存占用恢复正常，问题得到解决。

## 适合分享给谁

这篇文档尤其适合分享给：

- 遇到“刚开机内存就爆了”的 Windows 用户
- 任务管理器看不懂但 `RAMMap` 异常的人
- 装过阿里系桌面客户端后出现顽固后台的人
- 排查内核池异常占用的运维或开发者

## 许可建议

如果准备开源分享，建议采用：

- `MIT License`
- 或 `CC BY 4.0`

如果是写成经验帖或博客，`CC BY 4.0` 会更合适；如果打算把排查脚本也一起开源，`MIT` 会更方便。
