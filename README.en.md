# AlibabaProtect / AliPaladin Causes Abnormally High Windows Memory Usage After Reboot

English | [中文](README.zh-CN.md)

This is a real-world Windows memory troubleshooting record.

The symptom I ran into was:

- The machine had `16GB` of RAM
- Immediately after reboot, total memory usage was close to `90%`
- Ordinary user processes in Task Manager were not using much memory
- In `RAMMap`, `Nonpaged Pool`, `Paged Pool`, and `Page Table` were clearly high

The final root cause was not an ordinary foreground application. It was traced to Alibaba-related protection components:

- `AlibabaProtect`
- `AliPaladin`

`AliPaladin` is a file-system filter driver, and `AlibabaProtect` is the companion service. When the two behave abnormally, they can push a large amount of usage into kernel pools, creating the pattern where user processes look small but the whole machine is nearly out of memory.

## TL;DR

If you see these signals:

- Memory usage is high immediately after boot
- Task Manager does not show many large processes
- `RAMMap` shows high `Nonpaged Pool` / `Paged Pool` / `Page Table`
- Alibaba-related desktop clients or security/protection components have been installed

Check these components first:

- `AlibabaProtect`
- `AliPaladin`

## Key Symptoms in This Case

### 1. Ordinary processes could not explain total memory usage

Foreground applications and ordinary background processes did not add up to the near-full memory usage reported by the system.

### 2. RAMMap pointed toward kernel memory

The abnormal categories were mainly:

- `Nonpaged Pool`
- `Paged Pool`
- `Page Table`

That made the issue look more like a driver, filter, or third-party protection component problem than a browser or chat-client problem.

### 3. System logs showed service failures

`AlibabaProtect` repeatedly appeared in the system logs with:

- Service response timeouts
- Service control requests that did not respond

### 4. A lower-level driver was present

After checking services and drivers, I also found:

- `AliPaladin`
- Type: `FILE_SYSTEM_DRIVER`
- Group: `FSFilter Activity Monitor`

This means it is not just an ordinary background process. It is a file-system filter driver.

## Root-Cause Assessment

The root cause in this case can be summarized as:

1. `AlibabaProtect` is a security, anti-tamper, or environment-check component installed by Alibaba-related software
2. `AliPaladin` is its companion file-system filter driver
3. Both components behaved abnormally on this machine
4. The abnormal state pushed kernel pool usage noticeably higher
5. The visible symptom was abnormally high memory usage immediately after reboot

## What These Components Appear to Do

Based on file names and driver type, these components commonly perform:

- Process scanning
- File monitoring
- Anti-injection checks
- Environment validation
- Risk detection
- Anti-tamper protection

The installation directory may contain files such as:

- `AntiInject.dll`
- `FileMonitor.dll`
- `ProcessScanner.dll`
- `CloudDetector.dll`
- `ThreatSieveSDK.dll`

These components are not necessarily broken in every environment. But if they enter an abnormal state or run into compatibility issues, kernel pool usage can grow dramatically.

## How I Fixed It

The effective fix was not simply closing one foreground process. Both layers had to be addressed:

- `AlibabaProtect`
- `AliPaladin`

The process was:

1. Confirm that ordinary process memory could not explain total memory usage
2. Use `RAMMap` to confirm the issue pointed toward kernel pools
3. Confirm from system logs that `AlibabaProtect` was abnormal
4. Find the lower-level `AliPaladin` driver
5. Set both components to `Disabled`
6. Remove the corresponding service entries
7. Reboot Windows so the loaded service and driver actually exit

## Why Memory Dropped So Much

The reclaimed memory was not just the working set of `AlibabaProtect.exe`. It came from the system-level overhead of the whole protection chain:

- Service layer
- Driver layer
- Filter-driver path
- Page table and mapping overhead

That is why a service that appears to use only a few hundred MB can still cause several GB of extra machine-wide memory pressure.

## Result After the Fix

After rebooting and checking again:

- The `AlibabaProtect` service no longer existed
- The `AliPaladin` driver service no longer existed
- `AlibabaProtect.exe` was no longer running
- Kernel pool usage dropped noticeably

On this machine, the approximate result was:

- Available memory returned to `8GB+`
- `Nonpaged Pool` dropped from about `1.7GB` to about `785MB`
- `Paged Pool` dropped from about `1.0GB` to about `480MB`

## Useful Commands

View services:

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

View recent service-control logs:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Service Control Manager'} -MaxEvents 100
```

## Notes and Cautions

- Do not blindly delete drivers or services
- First confirm whether the issue really comes from this chain
- Some services cannot be stopped online and require a disable-and-reboot cycle before they fully unload
- When Task Manager is not enough, use `RAMMap` and system logs together

## Full Record

The complete investigation is available here:

- [`docs/case-study.en.md`](docs/case-study.en.md)

Chinese version:

- [`README.zh-CN.md`](README.zh-CN.md)
- [`docs/case-study.zh-CN.md`](docs/case-study.zh-CN.md)

## License

This documentation is shared under `CC BY 4.0`. See `LICENSE` for details.
