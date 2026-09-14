# RDP Wrapper fix for Windows 11 build 26100.9444

[English](README.md) | [中文](README.zh-CN.md)

This guide explains why RDP Wrapper may stop working on Windows 11 after a recent update, and how to fix it by adding the missing `10.0.26100.9444` configuration to `rdpwrap.ini`.

## Problem

RDP Wrapper may report **Not supported**, or the service may start and listen on port 3389 but remote desktop logons still fail or kick the existing session.

## Cause

Your Windows build may report an OS version such as `10.0.26200`, while the actual `termsrv.dll` internal version is:

```text
10.0.26100.9444
```

RDP Wrapper looks up the numeric private part of `termsrv.dll`, so it searches for:

```ini
[10.0.26100.9444]
```

If your installed `rdpwrap.ini` does not contain that section, the compatibility patches are not applied.

> PR [#4062](https://github.com/stascorp/rdpwrap/pull/4062) does **not** fix this build. It only adds support up to `10.0.26100.7623`.

## Requirements

- Windows 11 with `termsrv.dll` version `10.0.26100.9444`
- RDP Wrapper installed in `C:\Program Files\RDP Wrapper`
- Administrator privileges
- A backup of your current `rdpwrap.ini`

## Important notes

- Do **not** replace the whole INI with a minimal file that only contains the build-specific sections.
- Keep the existing global `[PatchCodes]` section intact.
- Do **not** add `SLInitHook.x64` or `SLInitOffset.x64` for this build. Community reports indicate those two lines can make `TermService` crash.
- Make sure the INI file ends with a blank line. An unterminated last line can be silently ignored.

## Step 1 — Check the installed version

Run this in PowerShell:

```powershell
$v = (Get-Item "C:\Windows\System32\termsrv.dll").VersionInfo
"$($v.FileMajorPart).$($v.FileMinorPart).$($v.FileBuildPart).$($v.FilePrivatePart)"
```

You should see:

```text
10.0.26100.9444
```

## Step 2 — Back up the current INI

```powershell
Copy-Item "C:\Program Files\RDP Wrapper\rdpwrap.ini" `
          "C:\Program Files\RDP Wrapper\rdpwrap.ini.bak-before-9444"
```

## Step 3 — Add the missing sections

Open this file as Administrator:

```text
C:\Program Files\RDP Wrapper\rdpwrap.ini
```

Add the following two blocks to the end of the file:

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

Press **Enter** once after the last line so the file ends with a blank line.

## Where these values come from

The offsets and code names in Step 3 are not invented by this guide. They come from a community reverse-engineered configuration for the exact `termsrv.dll` build `10.0.26100.9444`.

- The original values were posted by `glovagnini`, who credited `Heng Shixin` from a Telegram group: [original 9444 config](https://github.com/stascorp/rdpwrap/issues/4449#issuecomment-5594896021)
- The variant used here was posted by `JeeperS1987`; it intentionally omits `SLInitHook.x64` and `SLInitOffset.x64`, because those two lines were reported to crash `TermService` on this build: [working 9444 config](https://github.com/stascorp/rdpwrap/issues/4449#issuecomment-5596861003)

### What the numbers mean

- `LocalOnlyOffset.x64=99801`, `SingleUserOffset.x64=A637B`, and `DefPolicyOffset.x64=A351D` are hexadecimal offsets relative to the base address of the loaded 64-bit `termsrv.dll` module.
- RDP Wrapper adds each offset to the module’s base address and writes the corresponding patch bytes at that location.
- Names such as `jmpshort`, `mov_eax_1_nop_2`, and `CDefPolicy_Query_r9d_rdi_jmp` are symbolic labels for byte patterns already defined in the global `[PatchCodes]` section of `rdpwrap.ini`.
- The `[10.0.26100.9444-SLInit]` block contains offsets to individual fields inside the Terminal Services licensing/session-policy structure that RDP Wrapper’s `New_CSLQuery_Initialize` hook fills in.

Because Microsoft changes `termsrv.dll` with each Windows update, these offsets are build-specific. They are valid only for `termsrv.dll 10.0.26100.9444`, which is why PR 4062 does not help here.
## Step 4 — Disable “single session per user”

If you want concurrent sessions for the same user, set both of these registry values to `0`:

```powershell
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" `
                 -Name fSingleSessionPerUser -Value 0 -Type DWord

Set-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" `
                 -Name fSingleSessionPerUser -Value 0 -Type DWord
```

## Step 5 — Restart Remote Desktop Services

Run this in an elevated PowerShell prompt:

```powershell
Restart-Service -Name TermService -Force
```

This will disconnect any active RDP sessions.

## Step 6 — Verify

1. Check that `TermService` is running:

   ```powershell
   Get-Service -Name TermService
   ```

2. Check that port 3389 is listening:

   ```powershell
   netstat -an | findstr ":3389"
   ```

3. Open `RDP_CnC.exe`. It should now report **Fully supported**.
4. Test a second RDP connection.

## Rollback

To undo the INI change:

```powershell
Copy-Item "C:\Program Files\RDP Wrapper\rdpwrap.ini.bak-before-9444" `
          "C:\Program Files\RDP Wrapper\rdpwrap.ini" -Force

Restart-Service -Name TermService -Force
```

To restore the single-session policy, set both `fSingleSessionPerUser` values back to `1`.

## Related tools

PR 4062 also references a separate tool that can generate these values automatically:

- [`llccd/RDPWrapOffsetFinder`](https://github.com/llccd/RDPWrapOffsetFinder) — automatically finds RDP Wrapper offsets and generates an `rdpwrap.ini` section for a given `termsrv.dll`.
- [`sergiye/rdpWrapper`](https://github.com/sergiye/rdpWrapper) — a separate RDP Wrapper implementation that integrates `RDPWrapOffsetFinder` and `TermWrap`, so it can auto-generate offsets for unsupported Windows builds.

These tools are useful when a new Windows update changes `termsrv.dll` and no community offsets have been published yet.

## References

- [Issue 4449: 10.0.26100.9444 not supported](https://github.com/stascorp/rdpwrap/issues/4449)
- [Issue 4473: 10.0.26100.9444 despite Fully Supported](https://github.com/stascorp/rdpwrap/issues/4473)
- [sebaxakerhtc/rdpwrap.ini](https://github.com/sebaxakerhtc/rdpwrap.ini)
- [PR 4062](https://github.com/stascorp/rdpwrap/pull/4062)


