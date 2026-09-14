# Windows 11 26100.9444 的 RDP Wrapper 修复方法

[English](README.md) | [中文](README.zh-CN.md)

本指南说明 Windows 11 更新后 RDP Wrapper 失效的原因，以及如何通过补全 `rdpwrap.ini` 中缺失的 `10.0.26100.9444` 配置来修复。

## 问题

RDP Wrapper 可能显示 **不支持**，或者服务已启动并在 3389 端口监听，但远程桌面登录仍然失败，或者新连接会挤掉现有会话。

## 原因

Windows 版本号可能显示为 `10.0.26200`，但实际的 `termsrv.dll` 内部版本是：

```text
10.0.26100.9444
```

RDP Wrapper 会读取 `termsrv.dll` 的数字私有版本号，因此需要查找：

```ini
[10.0.26100.9444]
```

如果你当前的 `rdpwrap.ini` 中没有这一节，兼容性补丁就不会生效。

> PR [#4062](https://github.com/stascorp/rdpwrap/pull/4062) **不能**修复这个版本。它最高只支持到 `10.0.26100.7623`。

## 适用条件

- Windows 11，且 `termsrv.dll` 版本为 `10.0.26100.9444`
- RDP Wrapper 已安装在 `C:\Program Files\RDP Wrapper`
- 具备管理员权限
- 已备份当前 `rdpwrap.ini`

## 重要提示

- 不要把整个 INI 替换成只包含版本配置的极简文件。
- 必须保留原有的全局 `[PatchCodes]` 节。
- 这个版本**不要**添加 `SLInitHook.x64` 或 `SLInitOffset.x64`。社区反馈显示这两行可能导致 `TermService` 崩溃。
- 确保 INI 文件末尾有空行。最后一行如果没有换行符，可能会被静默忽略。

## 第 1 步 — 检查已安装版本

在 PowerShell 中运行：

```powershell
$v = (Get-Item "C:\Windows\System32\termsrv.dll").VersionInfo
"$($v.FileMajorPart).$($v.FileMinorPart).$($v.FileBuildPart).$($v.FilePrivatePart)"
```

应看到：

```text
10.0.26100.9444
```

## 第 2 步 — 备份当前 INI

```powershell
Copy-Item "C:\Program Files\RDP Wrapper\rdpwrap.ini" `
          "C:\Program Files\RDP Wrapper\rdpwrap.ini.bak-before-9444"
```

## 第 3 步 — 添加缺失的配置节

以管理员身份打开：

```text
C:\Program Files\RDP Wrapper\rdpwrap.ini
```

在文件末尾添加以下两节：

```ini
[10.0.26100.9444]
LocalOnlyPatch.x64=1
LocalOnlyOffset.x64=99801
LocalOnlyCode.x64=jmpshort
SingleUserPatch.x64=1
SingleUserOffset.x64=A637B
SingleUserCode.x64=mov_eax_1_nop_2
DefPolicyPatch.x64=1
DefPolicyOffset.x64=A351D
DefPolicyCode.x64=CDefPolicy_Query_r9d_rdi_jmp

[10.0.26100.9444-SLInit]
bInitialized.x64=12D260
bServerSku.x64=12D264
lMaxUserSessions.x64=12D268
bAppServerAllowed.x64=12D270
bRemoteConnAllowed.x64=12D27C
bMultimonAllowed.x64=12D280
ulMaxDebugSessions.x64=12D288
bFUSEnabled.x64=12D28C
```

最后一行后请按一次 **Enter**，确保文件以空行结束。

## 第 4 步 — 关闭“每用户单会话”

如果你希望同一用户可以同时开多个会话，请把以下两个注册表值都设为 `0`：

```powershell
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" `
                 -Name fSingleSessionPerUser -Value 0 -Type DWord

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" `
                 -Name fSingleSessionPerUser -Value 0 -Type DWord
```

## 第 5 步 — 重启远程桌面服务

在管理员 PowerShell 中运行：

```powershell
Restart-Service -Name TermService -Force
```

这会断开当前已连接的 RDP 会话。

## 第 6 步 — 验证

1. 确认 `TermService` 正在运行：

   ```powershell
   Get-Service -Name TermService
   ```

2. 确认 3389 端口正在监听：

   ```powershell
   netstat -an | findstr ":3389"
   ```

3. 打开 `RDP_CnC.exe`，应显示 **完全支持**。
4. 再测试第二个远程桌面连接。

## 回滚

如需撤销 INI 修改：

```powershell
Copy-Item "C:\Program Files\RDP Wrapper\rdpwrap.ini.bak-before-9444" `
          "C:\Program Files\RDP Wrapper\rdpwrap.ini" -Force

Restart-Service -Name TermService -Force
```

如需恢复单会话策略，请把两个 `fSingleSessionPerUser` 值改回 `1`。

## 参考

- [Issue 4449: 10.0.26100.9444 not supported](https://github.com/stascorp/rdpwrap/issues/4449)
- [Issue 4473: 10.0.26100.9444 despite Fully Supported](https://github.com/stascorp/rdpwrap/issues/4473)
- [sebaxakerhtc/rdpwrap.ini](https://github.com/sebaxakerhtc/rdpwrap.ini)
- [PR 4062](https://github.com/stascorp/rdpwrap/pull/4062)


