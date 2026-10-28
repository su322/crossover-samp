# crossover-samp
# 在 CrossOver 中运行 SA-MP「自由城」——配置说明

记录日期：2026-10-27
适用环境：Apple Silicon Mac + CrossOver 26.3

## 1. 最终效果

- SA-MP 0.3.7-R3 可以正常启动，不再报 `800x600x32` 显示模式错误。
- 游戏以 **1280×832 窗口**运行（等于本机逻辑桌面分辨率），约 **50 FPS**。
- 服务器浏览器、收藏服务器、中文聊天、中文对话框和记分板均正常。
- 正常流程：启动 `samp.exe` → 选中「★Drift Star★自由城1」→「连接」。
- 原版 `samp.dll`、字体、存档、服务器地址和两个 CrossOver 启动项都保留。

## 2. 环境与路径

| 项目           | 值                                           |
| -------------- | -------------------------------------------- |
| macOS          | 15.7                                         |
| 机器           | MacBook Air（Apple M4，16 GB）               |
| CrossOver      | 26.3                                         |
| 容器（bottle） | `samp`，Windows 10（build 19045）            |
| 游戏           | GTA: San Andreas 1.0 + SA-MP 0.3.7-R3 整合包 |
| 服务器         | `★Drift Star★自由城1`，`ds.zycdjz1.com:7777` |
| 显示           | 2560×1664 Retina；逻辑分辨率 1280×832        |
| 游戏窗口       | 1280×832，约 50 FPS（`fpslimit=48`）         |

游戏目录（macOS）：

```
~/Library/Application Support/CrossOver/Bottles/samp/drive_c/Program Files (x86)/侠盗车手：圣安地列斯/
```

由于原路径含中文，容器内额外建了链接：

```
C:\GTA  ->  C:\Program Files (x86)\侠盗车手：圣安地列斯
```

两个 CrossOver 启动项（在 `~/Applications/CrossOver/`）都指向：

```
目标：C:\GTA\samp.exe
工作目录：C:\GTA
```

## 3. 三个问题与修复

### 3.1 启动报 `800x600x32`：自建 d3d9 代理

**现象**：`samp.exe` 启动后报找不到 `800x600x32` 显示模式，游戏退不回主菜单。

**原因**：Wine 自带的 `d3d9` 在枚举显示模式时没有提供游戏启动阶段要求的 `800x600×32` 模式。

**做法**：

1. 在游戏根目录放入一个 32 位 `d3d9.dll` 代理。它转发 `Direct3DCreate9` / `Direct3DCreate9Ex` 到真正的 `C:\windows\system32\d3d9.dll`，并在 `IDirect3D9` 的虚函数表里挂钩两个方法，让模式列表末尾多出一个 `800x600`、32 位（`X8R8G8B8`）的模式。

2. 在容器里设置 DLL 覆盖，让 Wine 优先加载本地代理：

   ```
   [Software\\Wine\\DllOverrides]
   "d3d9"="native,builtin"
   ```

   对应文件：

   ```
   ~/Library/Application Support/CrossOver/Bottles/samp/user.reg
   ```

   也可以在 CrossOver 的 Wine 配置（winecfg）→「函数库」里，新增 `d3d9` 并设为「原生优先于内建」（native, builtin）。

3. 验证方式：临时把 `d3d9.dll` 改名，再启动游戏，`800x600x32` 错误会重新出现；改回后恢复正常，说明代理和覆盖都在起作用。

> 关键点：只放 `d3d9.dll` 而不加 `native,builtin` 覆盖时，Wine 仍会使用内建 d3d9，代理被绕过，错误依旧。

### 3.2 窗口化：III.VC.SA.WindowedMode.asi

整合包自带 ASI Loader（`vorbisFile.dll`），会加载 `scripts\*.asi`。窗口化插件放在：

```
scripts/III.VC.SA.WindowedMode.asi
scripts/III.VC.SA.WindowedMode.cfg
```

`III.VC.SA.WindowedMode.cfg` 当前关键设置：

```ini
[MAIN]
WindowMode = 1      ; 1 = 窗口模式
ConfineCursor = 1
PauseOnFocusLoss = 1
ResumeOnFocus = 1

[WINDOW]
Width = 0           ; 0 = 使用桌面逻辑分辨率
Height = 0
```

因此窗口是 1280×832。插件文档说明 `Alt+Enter` / `Ctrl+Enter` 可在窗口和无边框之间切换（本次未专门测试）。

如果不需要窗口化，删掉这两个文件即可恢复游戏自己的全屏模式。

### 3.3 中文聊天与 UI：只改 sa-mp.cfg 字体

**原因**：聊天、对话框、记分板使用 `Arial`，Wine 自带的 Arial 没有中文字形；而 `sampaux3.ttf` 不是聊天字体，它是击杀列表的图标字体（只有 21 个字形），不要用它替换成中文字体。

CrossOver 会自动把 macOS 的系统字体注册进容器。本机已有的 `Arial Unicode MS` 来自：

```
/System/Library/Fonts/Supplemental/Arial Unicode.ttf
```

容器注册表 `system.reg` 中可看到它被映射为 `Arial Unicode MS (TrueType)`，因此不需要往 `C:\windows\Fonts` 安装任何字体。

修改：

```
~/Documents/GTA San Andreas User Files/SAMP/sa-mp.cfg
```

加入（或确认）以下三行：

```ini
fontface="Arial Unicode MS"
fontweight=1
fontsize=1
```

修改后重新进入服务器即可看到中文。SA-MP 退出时会重写该文件，但已验证这三行会保留。

### 3.4 中文路径与启动项

原游戏目录名含中文，因此：

- 建立 `C:\GTA` → 中文目录的链接；
- 两个启动项的 `.lnk` 目标改为 `C:\GTA\samp.exe`，工作目录设为 `C:\GTA`；
- CrossOver 里的两个入口都保留，名称未改。

## 4. 日常启动步骤

1. 打开 CrossOver，或从启动台打开：
   - 「圣安地列斯 联机工具 SA-MP 0.3.7-R3语音插件版」
   - 或「圣安地列斯游戏本体+SAMP-R3带语音插件汉化版」
2. 在 SA-MP 服务器浏览器中找到「★Drift Star★自由城1」。
3. 选中服务器，点「连接」。
4. 进入服务器后中文聊天正常。

服务器地址：`ds.zycdjz1.com:7777`
收藏记录保存在 `SAMP/USERDATA.DAT`，未改动。

## 5. 文件与配置清单

### 新增或修改的文件

| 路径                                                    | 作用                              | 校验值                                                       |
| ------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------ |
| 游戏根目录 `d3d9.dll`                                   | 800×600×32 模式代理，271,360 字节 | SHA-256 `fde5c00e2246f939ae542b5535cd39642dfaeda020a169bf11494b560fb134c2` |
| `scripts/III.VC.SA.WindowedMode.asi`                    | 窗口化插件，3,232,600 字节        | SHA-256 `8b2485e89761044f56bbd4e2fa882ec9667939edf542fde682e016bd7e9373a9` |
| `scripts/III.VC.SA.WindowedMode.cfg`                    | 窗口化设置                        | `WindowMode=1`、`Width=0`、`Height=0`                        |
| `~/Documents/GTA San Andreas User Files/SAMP/sa-mp.cfg` | 中文字体设置                      | 见 3.3                                                       |
| `~/Library/.../Bottles/samp/user.reg`                   | `d3d9=native,builtin` 覆盖        | 见 3.1                                                       |
| `C:\GTA`                                                | 指向中文游戏目录的链接            | —                                                            |
| 两个 CrossOver `.app` / `.lnk`                          | 启动入口，目标 `C:\GTA\samp.exe`  | —                                                            |

### 未改动的原文件

| 文件                                      | SHA-256                                                      |
| ----------------------------------------- | ------------------------------------------------------------ |
| `samp.dll`                                | `9c9b2cc31a4ced6967420b1880c096b5c4e7630e227aa379be4019c21b6fddc1` |
| `samp.exe`                                | `b5e760036bc2cd8c3b6ce6c3fd0d2a3f5ea896a33ffcbc3914d6895d1e46dfa5` |
| `sampaux3.ttf`                            | `3adfe84498e8951e61ece57ad08de21b95d162697517291564ae6ef3a8790c1b` |
| `gtaweap3.ttf`                            | `086dc4351bf1056d80a73168440ffe4261831bde992e5134902b9bae017dcbe9` |
| `vorbisFile.dll`（整合包自带 ASI Loader） | `c33e1bafafd441b42eaa6d322393b9dd700f8a9c13fb2e7780daf49707a46353` |

`samp.dll` 没有改名、没有打补丁；代理逻辑全部在独立的 `d3d9.dll` 里。

> 最终版 `d3d9.dll` 已去掉调试日志输出，游戏目录不会再生成 `d3d9proxy.log`。

## 6. 回滚

先退出游戏和 CrossOver，再按需操作：

1. **撤销 800×600 修复**：删除游戏根目录的 `d3d9.dll`，并在 winecfg 的「函数库」中删除 `d3d9` 覆盖（或把 `user.reg` 里的 `"d3d9"="native,builtin"` 删除）。
2. **撤销窗口化**：删除 `scripts/III.VC.SA.WindowedMode.asi` 和 `.cfg`。
3. **撤销中文字体**：删除 `sa-mp.cfg` 中的 `fontface` / `fontweight` / `fontsize` 三行，或改回 `fontface="Arial"`。
4. **撤销快捷方式**：把两个启动项的目标改回中文游戏目录下的 `samp.exe`（或直接删除 `C:\GTA` 链接后重新指定）。

游戏本体、存档、服务器收藏均不受上述操作影响。

## 7. 常见问题

**启动又报 `800x600x32`**

- 检查游戏根目录 `d3d9.dll` 是否存在、哈希是否正确。
- 检查 `d3d9` 覆盖是否还在；CrossOver 更新或修复容器后可能被重置。
- 确认启动的是 `samp` 这个容器。

**中文变成方块或问号**

- 检查 `sa-mp.cfg` 的三行字体设置是否被客户端重写。
- 确认 `system.reg` 中 `Arial Unicode MS` 仍指向 macOS 的 `Arial Unicode.ttf`。
- 不要把 `sampaux3.ttf` 换成中文字体，它不是聊天字体。

**画面全屏、无法切窗口**

- 检查 `scripts/III.VC.SA.WindowedMode.cfg` 的 `WindowMode` 是否为 `1`。
- 删除该 cfg 后重启游戏，会重新生成默认设置。
- 可尝试 `Alt+Enter` / `Ctrl+Enter`。

**窗口太大或太小**

- `Width` / `Height` 设为 `0` 表示使用桌面逻辑分辨率；也可以写死，例如 `1280` / `832`。

**启动项打不开**

- 确认 `C:\GTA` 链接仍存在，且指向中文游戏目录。
- 启动项目标应为 `C:\GTA\samp.exe`，工作目录 `C:\GTA`。

## 附录 A：d3d9 代理原理

1. 游戏加载同目录的 `d3d9.dll`（需要 `d3d9=native,builtin` 覆盖）。
2. 代理在 `DllMain` 中不做任何渲染工作，只把 `Direct3DCreate9` / `Direct3DCreate9Ex` 转发给 `C:\windows\system32\d3d9.dll`。
3. 拿到真正的 `IDirect3D9` 对象后，复制一份虚函数表，替换其中两个函数：
   - `GetAdapterModeCount`：返回值 +1；
   - `EnumAdapterModes`：多出来的那一项返回 `800×600`、`RefreshRate=0`、格式为请求的 32 位格式。
4. 游戏因此能在模式列表里找到 `800x600x32`，正常进入。
5. 除这两个方法外，其余 D3D9 调用全部走原实现，不影响渲染。

## 附录 B：`d3d9proxy.c`（最终无日志版）

```c
#include <windows.h>
#include <d3d9.h>

static HMODULE g_real_d3d9;
static IDirect3D9 * (WINAPI *g_real_create9)(UINT);
static HRESULT (WINAPI *g_real_create9ex)(UINT, IDirect3D9Ex **);
static IDirect3D9Vtbl *g_orig_vtbl;
static IDirect3D9Vtbl *g_new_vtbl;
static int g_patched;

static UINT __stdcall hook_GetAdapterModeCount(IDirect3D9 *self, UINT Adapter, D3DFORMAT Format)
{
    UINT n = g_orig_vtbl->GetAdapterModeCount(self, Adapter, Format);
    return n + 1;
}

static HRESULT __stdcall hook_EnumAdapterModes(IDirect3D9 *self, UINT Adapter, D3DFORMAT Format,
                                               UINT Mode, D3DDISPLAYMODE *pMode)
{
    UINT n = g_orig_vtbl->GetAdapterModeCount(self, Adapter, Format);
    if (Mode < n)
        return g_orig_vtbl->EnumAdapterModes(self, Adapter, Format, Mode, pMode);
    if (Mode == n) {
        if (!pMode) return D3DERR_INVALIDCALL;
        pMode->Width = 800;
        pMode->Height = 600;
        pMode->RefreshRate = 0;
        pMode->Format = (Format == D3DFMT_UNKNOWN) ? D3DFMT_X8R8G8B8 : Format;
        return S_OK;
    }
    return D3DERR_INVALIDCALL;
}

static void patch_vtable(IDirect3D9 *obj)
{
    if (!obj) return;
    if (!g_patched) {
        g_orig_vtbl = obj->lpVtbl;
        g_new_vtbl = (IDirect3D9Vtbl *)VirtualAlloc(NULL, sizeof(IDirect3D9Vtbl),
                                                    MEM_COMMIT | MEM_RESERVE,
                                                    PAGE_READWRITE);
        if (!g_new_vtbl) return;
        memcpy(g_new_vtbl, g_orig_vtbl, sizeof(IDirect3D9Vtbl));
        g_new_vtbl->GetAdapterModeCount = hook_GetAdapterModeCount;
        g_new_vtbl->EnumAdapterModes = hook_EnumAdapterModes;
        g_patched = 1;
    }
    obj->lpVtbl = g_new_vtbl;
}

static void load_real(void)
{
    if (g_real_d3d9) return;
    g_real_d3d9 = LoadLibraryA("C:\\windows\\system32\\d3d9.dll");
    if (!g_real_d3d9) return;
    g_real_create9 = (void *)GetProcAddress(g_real_d3d9, "Direct3DCreate9");
    g_real_create9ex = (void *)GetProcAddress(g_real_d3d9, "Direct3DCreate9Ex");
}

IDirect3D9 * WINAPI Direct3DCreate9(UINT SDKVersion)
{
    IDirect3D9 *obj;
    load_real();
    if (!g_real_create9) return NULL;
    obj = g_real_create9(SDKVersion);
    patch_vtable(obj);
    return obj;
}

HRESULT WINAPI Direct3DCreate9Ex(UINT SDKVersion, IDirect3D9Ex **ppD3D)
{
    HRESULT hr;
    load_real();
    if (!g_real_create9ex) return E_FAIL;
    hr = g_real_create9ex(SDKVersion, ppD3D);
    if (SUCCEEDED(hr) && ppD3D && *ppD3D)
        patch_vtable((IDirect3D9 *)*ppD3D);
    return hr;
}

BOOL WINAPI DllMain(HINSTANCE hinst, DWORD reason, LPVOID reserved)
{
    if (reason == DLL_PROCESS_ATTACH)
        DisableThreadLibraryCalls(hinst);
    return TRUE;
}
```

## 附录 C：编译方法

代理导出两个函数，需要 `d3d9.def`：

```def
LIBRARY d3d9
EXPORTS
    Direct3DCreate9
    Direct3DCreate9Ex
```

使用 Homebrew 的 zig（本机为 0.17.0）交叉编译为 32 位 Windows DLL：

```bash
brew install zig   # 如果还没有
zig cc -target x86-windows-gnu -shared -g -O2 \
    -o d3d9.dll d3d9proxy.c d3d9.def
```

验证导出表：

```bash
llvm-objdump -p d3d9.dll | sed -n '/Export Table/,/^$/p'
```

应看到：

```
Direct3DCreate9
Direct3DCreate9Ex
```

把生成的 `d3d9.dll` 放到游戏根目录，并确保 `d3d9=native,builtin` 覆盖存在即可。`-g` 只是保留调试信息，不影响运行；如不需要可去掉。
