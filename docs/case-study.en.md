# Abnormally High Windows Memory Usage After Reboot: An AlibabaProtect / AliPaladin Case Study

English | [中文](case-study.zh-CN.md)

## Overview

This document records a real-world Windows memory troubleshooting case.

The symptoms were:

- The machine had `16GB` of RAM
- Immediately after reboot, total memory usage was close to `90%` or even higher
- Ordinary user processes in Task Manager did not add up to that amount
- In `RAMMap`, `Nonpaged Pool`, `Paged Pool`, and `Page Table` were abnormally high

The final root cause was not an ordinary foreground program. It was traced to Alibaba-related security/protection components:

- `AlibabaProtect`
- `AliPaladin`

`AliPaladin` is a file-system filter driver, and `AlibabaProtect` is the corresponding protection service. When they work together and enter an abnormal state, they can push a large amount of memory usage into kernel pools. The result is a confusing situation where user processes look small, but the whole machine is almost out of memory.

## Typical Symptoms

This case is worth checking if you see the following:

- Memory usage is already high right after boot
- The "Users" tab in Task Manager shows only a small amount of memory
- Ordinary application processes add up to far less than the total memory usage
- These `RAMMap` categories are abnormally high:
  - `Nonpaged Pool`
  - `Paged Pool`
  - `Page Table`
- Third-party security components, anti-tamper components, or driver-level protection programs are resident

## Why Task Manager Is Misleading Here

This is the most misleading part of this type of issue.

Ordinary applications mainly use user-mode memory. Components such as `AlibabaProtect` / `AliPaladin` can place part of their cost in kernel memory, especially:

- `Nonpaged Pool`
- `Paged Pool`
- Driver mappings and page-table-related overhead

So you may see:

- The "Users" tab appears to use only a few hundred MB to 1-2GB
- But total memory usage has already climbed into the tens of GB

That does not mean Task Manager is wrong. It means the accounting view is different.

## Key Signals in This Investigation

### 1. Ordinary process usage could not explain total memory

After checking process-level memory usage, ordinary user processes were not enough to explain the near-full memory state.

### 2. RAMMap exposed the real direction

In `RAMMap -> Use Counts`, the most important categories were:

- High `Page Table`
- High `Paged Pool`
- High `Nonpaged Pool`

This pointed toward:

- Drivers
- File-system filters
- Network filters
- Third-party security components

It did not look like an ordinary browser, chat app, or editor issue.

### 3. System logs showed service failures

In the system logs, `AlibabaProtect` repeatedly showed:

- Service response timeouts
- Service control requests that did not respond

This indicated that it was not simply an idle background service. It was a protection component already behaving abnormally.

### 4. The lower-level `AliPaladin` driver was found

After checking the registry and service configuration, another lower-level driver was found in addition to `AlibabaProtect`:

- Service name: `AliPaladin`
- Type: `FILE_SYSTEM_DRIVER`
- Group: `FSFilter Activity Monitor`

This was important because it meant the component was not merely a user-mode program. It was a filter driver inserted into the file-system path. When the user-mode service behaves badly, the driver layer can amplify the problem.

## Root-Cause Assessment

The root cause in this case can be summarized as:

1. `AlibabaProtect` is a security, anti-tamper, or environment-check component installed by Alibaba-related software
2. `AliPaladin` is its companion file-system filter driver
3. Both components behaved abnormally on this machine
4. The abnormal behavior inflated kernel pool usage
5. The final symptom was abnormally high memory usage immediately after reboot

Put more directly:

It was not a single foreground application consuming 16GB of RAM. A driver and protection service were pushing kernel memory usage upward.

## What These Components Appear to Do

Based on file names, their responsibilities likely include:

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

These components are not necessarily always problematic. But if a driver conflict, retry loop, abnormal state, or compatibility issue occurs, they can raise kernel pool usage dramatically.

## How We Fixed It

### Step 1: Confirm it was not an ordinary application issue

We first checked process memory and found that ordinary applications could not explain the total usage, so the focus moved to the kernel layer.

### Step 2: Confirm that `AlibabaProtect` was abnormal

Service status and system logs showed:

- `AlibabaProtect` started automatically
- The service timed out for a long period
- Control requests did not respond

### Step 3: Find `AliPaladin`

After checking services and driver entries, `AliPaladin` was found running as a file-system filter driver.

### Step 4: Handle both the service layer and the driver layer

Disabling only `AlibabaProtect` was not enough, because the driver layer could still keep the chain alive.

The effective fix handled both:

- `AlibabaProtect`
- `AliPaladin`

The concrete actions included:

- Set both startup types to `Disabled`
- Remove the corresponding service entries
- Reboot the system so the currently loaded driver and service actually exit memory

### Step 5: Verify after reboot

After rebooting:

- `AlibabaProtect` no longer existed as an installed service
- `AliPaladin` no longer existed as an installed driver service
- `AlibabaProtect.exe` was no longer running
- Kernel pool usage dropped noticeably

## Why Memory Dropped So Much

This is the part that often surprises people the first time they encounter this type of issue.

The removed cost was not a "foreground app using only a few hundred MB." It was a protection chain affecting kernel pools:

- Service layer
- Driver layer
- Filter-driver path
- Page table and mapping overhead

Once the source was removed, the memory drop was not limited to the working set of `AlibabaProtect.exe`. It also removed system-level overhead from the whole chain.

That is why the before-and-after memory difference can be so large.

## Risks and Cautions

### 1. Do not blindly delete drivers or services

First confirm whether this is truly your source of the problem. Not every high-memory case is related to `AlibabaProtect`.

### 2. Do not rely only on Task Manager

Use these tools together:

- Task Manager
- `RAMMap`
- System logs
- Service list
- Driver list

### 3. Changing only the service may not be enough

If the companion driver remains active, it may bring the service back.

### 4. Some services cannot be stopped online

Some services or drivers may show:

- `NOT_STOPPABLE`
- `NOT_PAUSABLE`

That does not mean the command is useless. It means the component is already loaded in the current session and must be disabled and then unloaded by rebooting.

## Recommended Troubleshooting Flow

If you run into a similar issue, check in this order:

1. Check total memory, available memory, and large foreground processes
2. Use `RAMMap` to inspect `Nonpaged Pool` / `Paged Pool` / `Page Table`
3. Check system logs for abnormal service timeouts
4. Check whether matching protection services and drivers exist
5. Handle both the service layer and the driver layer
6. Reboot and verify again

## Useful Command Examples

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

View recent Service Control Manager logs:

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Service Control Manager'} -MaxEvents 100
```

## Final Conclusion

This issue was not simply "16GB is too little," and it was not caused by opening too many ordinary applications.

The more accurate conclusion is:

- The Alibaba-related protection service `AlibabaProtect`
- The companion file-system filter driver `AliPaladin`

were behaving abnormally on this machine, which noticeably raised kernel pool usage and caused memory usage to be abnormally high immediately after reboot.

After removing this chain from the startup path and driver path, memory usage returned to normal and the issue was resolved.

## Who This Is Useful For

This document is especially useful for:

- Windows users whose memory is already exhausted right after boot
- Users who cannot explain memory usage in Task Manager but see abnormal values in `RAMMap`
- Users who installed Alibaba-related desktop clients and later found stubborn background components
- Operators or developers troubleshooting abnormal kernel pool usage

## License

This documentation is shared under `CC BY 4.0`. See [`../LICENSE`](../LICENSE) for details.
