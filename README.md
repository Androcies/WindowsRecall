# WindowsRecall
An easy way to return all applications to the PC main screen.

**Launch EXE → Move off-primary-monitor windows → Exit.**

Run `WindowRecall.exe`. It silently recalls qualifying application windows from secondary monitors to the primary display on the **current virtual desktop**. There is no setup, UI, console, tray icon, service, configuration file, or resident shortcut listener. Ordinary applications require no administrator privileges. The EXE can be copied anywhere by itself; the Microsoft C/C++ runtime is statically linked. Ideal for use at Windows shutdown if you want to ensure all apps are on main screen on the next system bootup.

## Build with Visual Studio 2026

Install the **Desktop development with C++** workload, MSVC **v145**, and a Windows 11 SDK. Open `WindowRecall.sln`, select **Release | x64**, then **Build Solution**. The result is `bin\x64\Release\WindowRecall.exe`.

Alternatively, run `Build.ps1` from PowerShell. It finds Visual Studio 2026, builds Release x64, and also copies the executable beside this README:

```powershell
.\Build.ps1
```

From a Visual Studio 2026 developer shell, the equivalent command is:

```powershell
msbuild .\WindowRecall.sln /m /p:Configuration=Release /p:Platform=x64
```

Both Debug and Release use the Windows subsystem. Only Windows system libraries are required; there are no NuGet packages or third-party runtime dependencies.

## Set shortcut key such as: Ctrl + the physical backtick/tilde key to run the executable

In Microsoft PowerToys:

1. Open **Keyboard Manager → Remap a shortcut → Add shortcut remapping**.
2. Record **Ctrl + the physical backtick/tilde key** (usually below Esc). **Do not press Shift**. On a US layout, this is Ctrl + backtick; the key may be labelled `OEM_3` or a backtick in the editor.
3. Choose **Start App** as the action. Set **App** to the full executable path, for example `D:\AppsToMainscreen\WindowRecall.exe`.
4. Leave arguments and the target-app restriction empty. Use **Normal** elevation and **Do nothing** if already running. Save the mapping.

PowerToys must remain running to dispatch the shortcut. WindowRecall itself exits after each invocation. See Microsoft's [Keyboard Manager instructions](https://learn.microsoft.com/en-us/windows/powertoys/keyboard-manager#remap-a-shortcut-to-start-an-app).

For keyboard macro software, bind **Ctrl + that physical key** to launch the EXE with no arguments. If the software supports scan-code bindings, the US physical backtick key is **scan code 0x29** (often `SC029`), with Ctrl and no Shift. Prefer recording the physical key on your own keyboard: virtual-key names depend on the active keyboard layout. Do not configure a typed `~` character, which normally implies Shift.

## Placement and exclusions

- Discovers the primary display using `MONITORINFOF_PRIMARY`; supports negative monitor coordinates and portrait displays.
- Uses Windows' monitor assignment (largest window intersection; saved pre-minimize position for minimized windows). **Any window assigned to the primary monitor is skipped before all write operations**, even if partly offscreen.
- Preserves physical outer dimensions where possible, translating the offset from the original usable area and clamping/shrinking into the primary usable area. It deliberately preserves pixel dimensions rather than scaling every window's logical size.
- Uses Per-Monitor V2 awareness and explicit screen/workspace conversion, including top/left taskbar or appbar offsets. A second queued position request accounts for applications applying their own DPI-change suggested bounds.
- Keeps maximized windows maximized, relocates their maximized frame, and updates their restored bounds. Keeps minimized windows minimized, updates their restored bounds, and preserves the minimized-from-maximized flag.
- Requests no activation and no owner Z-order changes. Ordinary moves use `SWP_NOZORDER`; maximized moves also restore the original Z-order anchor because `SetWindowPlacement` can raise an already-maximized window even with a non-activating show command. Foreign-thread placement is queued asynchronously so a hung application cannot hold up relocation. The process exits after dispatching requests; receiving applications finish them on their own UI threads.
- Accepts normal File Explorer windows (`CabinetWClass` / `ExploreWClass`) without process-wide exclusion of `explorer.exe`.
- Excludes hidden/cloaked windows, inactive virtual desktops, desktop/taskbar components, tool/no-activate windows, borderless overlays and transient surfaces, detected fullscreen surfaces (including minimized borderless fullscreen windows), recognized VR processes, and processes loading recognized OpenVR/OpenXR/Oculus runtime modules. Normal always-on-top apps remain eligible and retain their Z order. Desktop membership or process-inspection failures cause that window to be skipped. If the virtual-desktop service is unavailable, the entire operation exits silently without moving anything.

The filters are deliberately conservative: an ordinary borderless non-resizable app or an app loading a VR runtime can be skipped. Win32 cannot universally identify every game or VR application; unusual runtimes or custom fullscreen implementations may evade detection. Applications can also enforce minimum sizes, override placement, or change state during the operation. Windows security can block elevated/protected applications. These constraints cannot be overridden reliably by a small unprivileged utility; failed requests are skipped without dialogs or retries that wait on an application.

Exit code **0** means enumeration/dispatch completed (individual windows may have been skipped); **1** means startup, monitor discovery, desktop-service initialization, or enumeration failed. No log files are created.
