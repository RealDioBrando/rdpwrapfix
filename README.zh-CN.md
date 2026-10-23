# Windows 11 26100.9444 与 26100.9549 的 RDP Wrapper 修复方法

[English](README.md) | [中文](README.zh-CN.md)

本指南说明 Windows 11 更新后 RDP Wrapper 失效的原因，以及如何通过补全 `rdpwrap.ini` 中缺失的 `termsrv.dll` 版本配置来修复。

## 问题

RDP Wrapper 可能显示 **不支持**，或者服务已启动并在 3389 端口监听，但远程桌面登录仍然失败，或者新连接会挤掉现有会话。

## 原因

RDP Wrapper **不是**按 Windows 系统版本号查找配置，而是按 `termsrv.dll` 的**数字私有版本号**查找。

例如：

- Windows 系统版本可能显示为 `10.0.26200`
- `termsrv.dll` 的私有版本可能是 `10.0.26100.9549`
- RDP Wrapper 因此会查找：

```ini
[10.0.26100.9549]
```

如果你当前的 `rdpwrap.ini` 中没有这一节，兼容性补丁就不会生效。

> 在 `10.0.26100.9549` 这个版本上，文件属性里显示的 `FileVersion` 字符串可能仍然是 `10.0.26100.8875`。真正需要看的是数字型的 `FilePrivatePart`。

> PR [#4062](https://github.com/stascorp/rdpwrap/pull/4062) **不能**修复这些版本。它最高只支持到 `10.0.26100.7623`。

## 适用条件

- Windows 11，且 `termsrv.dll` 私有版本为 `10.0.26100.9549` 或 `10.0.26100.9444`
- RDP Wrapper 已安装在 `C:\Program Files\RDP Wrapper`
- 具备管理员权限
- 已备份当前 `rdpwrap.ini`

## 重要提示

- 不要把整个 INI 替换成只包含版本配置的极简文件。
- 必须保留原有的全局 `[PatchCodes]` 节。
- 对于 `9549` 版本，请使用下面的完整配置，包括 `SLInitHook.x64`。
- 如果应用 `9549` 配置后 `TermService` 崩溃，请只删除 `SLInitHook.x64`、`SLInitOffset.x64` 和 `SLInitFunc.x64` 这三行，然后重启服务。
- 对于 `9444` 版本，请不要添加 `SLInitHook.x64` 和 `SLInitOffset.x64`；社区反馈显示这两行可能导致该旧版本的 `TermService` 崩溃。
- 确保 INI 文件末尾有空行。最后一行如果没有换行符，可能会被静默忽略。

## 第 1 步 — 检查已安装版本

在 PowerShell 中运行：

```powershell
$v = (Get-Item "C:\Windows\System32\termsrv.dll").VersionInfo
"{0}.{1}.{2}.{3}" -f $v.FileMajorPart, $v.FileMinorPart, $v.FileBuildPart, $v.FilePrivatePart
```

当前版本应显示：

```text
10.0.26100.9549
```

检查对应的配置节是否已经存在：

```powershell
Select-String -LiteralPath "C:\Program Files\RDP Wrapper\rdpwrap.ini" `
              -Pattern '^\[10\.0\.26100\.9549\]$'
```

如果没有输出，说明该配置节缺失。

## 第 2 步 — 备份当前 INI

```powershell
Copy-Item "C:\Program Files\RDP Wrapper\rdpwrap.ini" `
          "C:\Program Files\RDP Wrapper\rdpwrap.ini.bak-before-9549"
```

## 第 3 步 — 添加 9549 的缺失配置节

以管理员身份打开：

```text
C:\Program Files\RDP Wrapper\rdpwrap.ini
```

在文件末尾添加以下两节：

```ini
[10.0.26100.9549]
SingleUserPatch.x64=1
SingleUserOffset.x64=A96FB
SingleUserCode.x64=mov_eax_1_nop_2
DefPolicyPatch.x64=1
DefPolicyOffset.x64=A689D
DefPolicyCode.x64=CDefPolicy_Query_r9d_rdi_jmp
LocalOnlyPatch.x64=1
LocalOnlyOffset.x64=9CB81
LocalOnlyCode.x64=jmpshort
SLInitHook.x64=1
SLInitOffset.x64=BD8B8
SLInitFunc.x64=New_CSLQuery_Initialize

[10.0.26100.9549-SLInit]
bServerSku.x64=131314
bRemoteConnAllowed.x64=13132C
bFUSEnabled.x64=13133C
bAppServerAllowed.x64=131320
bMultimonAllowed.x64=131330
lMaxUserSessions.x64=131318
ulMaxDebugSessions.x64=131338
bInitialized.x64=131310
```

最后一行后请按一次 **Enter**，确保文件以空行结束。

## 这些数值的来源

第 3 步中的偏移量和代码名称并非随意编写，而是来自针对 `termsrv.dll 10.0.26100.9549` 这个精确版本的社区逆向工程配置。

- 该配置由 `AlexHDman` 发布在 [issue #4485](https://github.com/stascorp/rdpwrap/issues/4485) 中。
- 相同数值也由 [`llccd/RDPWrapOffsetFinder`](https://github.com/llccd/RDPWrapOffsetFinder) 对以下 SHA-256 的 `termsrv.dll` 文件独立生成：

```text
E40261E4EE89F4DC9B4A3EA23B53631AAC2868D533A3930B92A9C821A2BF9456
```

### 数值的含义

- `SingleUserOffset.x64=A96FB`、`DefPolicyOffset.x64=A689D`、`LocalOnlyOffset.x64=9CB81` 是相对于已加载的 64 位 `termsrv.dll` 模块基址的十六进制偏移量。
- RDP Wrapper 会把每个偏移量加上模块基址，然后在对应位置写入补丁字节。
- `jmpshort`、`mov_eax_1_nop_2`、`CDefPolicy_Query_r9d_rdi_jmp` 这类名称不是原始字节，而是 `rdpwrap.ini` 全局 `[PatchCodes]` 节中已经定义的字节模式的符号名。
- `SLInitOffset.x64=BD8B8` 是远程桌面服务授权初始化函数的入口点，RDP Wrapper 会在该位置安装钩子。
- `[10.0.26100.9549-SLInit]` 中的数值是授权 / 会话策略结构中各字段的偏移量，RDP Wrapper 的 `New_CSLQuery_Initialize` 钩子会填充这些字段。

由于 Microsoft 每次更新都会修改 `termsrv.dll`，这些偏移量是版本相关的。它们只适用于 `termsrv.dll 10.0.26100.9549`，这也是 PR 4062 无法解决此问题的原因。

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
Copy-Item "C:\Program Files\RDP Wrapper\rdpwrap.ini.bak-before-9549" `
          "C:\Program Files\RDP Wrapper\rdpwrap.ini" -Force

Restart-Service -Name TermService -Force
```

如需恢复单会话策略，请把两个 `fSingleSessionPerUser` 值改回 `1`。

## 旧版本：10.0.26100.9444

如果你的 `termsrv.dll` 私有版本仍然是 `10.0.26100.9444`，请改用以下配置：

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

对于这个旧版本，**不要**添加 `SLInitHook.x64` 或 `SLInitOffset.x64`。

## 相关工具

当 Windows 更新修改了 `termsrv.dll`，而社区还没有发布新的偏移量时，以下工具可以自动生成新的 INI 配置节：

- [`llccd/RDPWrapOffsetFinder`](https://github.com/llccd/RDPWrapOffsetFinder) — 自动查找 RDP Wrapper 偏移量，并为指定的 `termsrv.dll` 生成 `rdpwrap.ini` 配置节。
- [`sergiye/rdpWrapper`](https://github.com/sergiye/rdpWrapper) — 另一个 RDP Wrapper 实现，集成了 `RDPWrapOffsetFinder` 和 `TermWrap`，可以为暂不支持的 Windows 版本自动生成偏移量。

## 参考

- [Issue 4485: 10.0.26100.9549 not supported](https://github.com/stascorp/rdpwrap/issues/4485)
- [Issue 4449: 10.0.26100.9444 not supported](https://github.com/stascorp/rdpwrap/issues/4449)
- [Issue 4473: 10.0.26100.9444 despite Fully Supported](https://github.com/stascorp/rdpwrap/issues/4473)
- [sebaxakerhtc/rdpwrap.ini](https://github.com/sebaxakerhtc/rdpwrap.ini)
- [PR 4062](https://github.com/stascorp/rdpwrap/pull/4062)
