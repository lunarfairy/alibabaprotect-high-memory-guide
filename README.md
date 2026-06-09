# AlibabaProtect / AliPaladin Windows High Memory Guide

[中文](README.zh-CN.md) | [English](README.en.md)

A real-world Windows memory troubleshooting guide for cases where memory usage is abnormally high right after reboot, while Task Manager does not show enough user processes to explain it.

一份真实的 Windows 内存排障记录，适用于“刚重启内存就异常偏高，但任务管理器里普通用户进程并不大”的场景。

## Quick Links / 快速入口

- [English quick guide](README.en.md)
- [中文快速说明](README.zh-CN.md)
- [English full case study](docs/case-study.en.md)
- [中文完整案例](docs/case-study.zh-CN.md)

## What This Case Found / 本案例发现

| English | 中文 |
| --- | --- |
| The machine had `16GB` of RAM. | 机器内存为 `16GB`。 |
| Memory usage was close to `90%` immediately after reboot. | 刚重启后，总内存占用接近 `90%`。 |
| Normal user processes did not account for the total memory usage. | 任务管理器里的普通用户进程无法解释总内存占用。 |
| `RAMMap` showed high `Nonpaged Pool`, `Paged Pool`, and `Page Table` usage. | `RAMMap` 显示 `Nonpaged Pool`、`Paged Pool`、`Page Table` 明显偏高。 |
| The root cause was traced to `AlibabaProtect` and the `AliPaladin` file-system filter driver. | 最终定位到 `AlibabaProtect` 和文件系统过滤驱动 `AliPaladin`。 |

## Typical Signals / 典型信号

| English | 中文 |
| --- | --- |
| Memory is already high right after boot. | 刚开机内存就很高。 |
| Task Manager does not show many large user processes. | 任务管理器里大进程不多。 |
| `RAMMap` points to kernel memory rather than ordinary applications. | `RAMMap` 指向内核层占用，而不是普通应用。 |
| System logs show repeated service timeout or control-request failures. | 系统日志里反复出现服务超时或控制请求无响应。 |
| Alibaba-related protection components are installed. | 系统中安装过阿里系保护组件。 |

## Useful Read-Only Commands / 可用只读命令

View Alibaba-related services:

```powershell
Get-Service | Where-Object { $_.Name -match 'Ali|Alibaba' -or $_.DisplayName -match 'Ali|Alibaba' }
```

Query service configuration:

```powershell
sc.exe qc AlibabaProtect
sc.exe qc AliPaladin
```

Query service status:

```powershell
sc.exe query AlibabaProtect
sc.exe query AliPaladin
```

Check kernel pool counters:

```powershell
Get-Counter '\Memory\Pool Nonpaged Bytes','\Memory\Pool Paged Bytes'
```

Read recent Service Control Manager logs:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Service Control Manager'} -MaxEvents 100
```

## Safety Note / 注意事项

Do not blindly delete services or drivers. First confirm that the memory pattern, system logs, and installed components actually match this case. Some drivers cannot be stopped online; they may require disabling and rebooting before they fully unload.

不建议盲删服务或驱动。请先确认内存模式、系统日志和已安装组件确实符合本案例。某些驱动在线不可停止，可能必须先禁用再重启，才能真正卸载。

## License / 许可证

This documentation is shared under `CC BY 4.0`. See [LICENSE](LICENSE) for details.

本文档以 `CC BY 4.0` 方式共享，详情见 [LICENSE](LICENSE)。
