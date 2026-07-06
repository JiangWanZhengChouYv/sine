# Wine 魔改研究报告 - CrossOver 6 大功能源码定位

## 研究对象
- Wine 源码：sine/ 目录（从 gitlab.winehq.org 克隆的最新主干）
- 参考：CrossOver 26.2.0
- 目标：找到 CrossOver 6 大功能在纯上游 Wine 中的对应位置

---

## 1. NtCreateUserProcess（最核心）

### 代码位置
| 文件 | 位置 | 说明 |
|------|------|------|
| `dlls/ntdll/unix/process.c` | 第 682 行 | NtCreateUserProcess 主实现 |
| `dlls/ntdll/ntsyscalls.h` | - | 系统调用声明 |
| `dlls/ntdll/ntdll.spec` | - | DLL 导出定义 |
| `server/process.c` | - | wineserver 端进程创建 |
| `server/request_handlers.h` | 第 10 行 | new_process 请求处理 |

### 调用链
```
CreateProcess (kernel32)
    ↓
NtCreateUserProcess (ntdll)
    ↓
    ├─ get_pe_file_info()  # 读取 PE 文件信息
    ├─ create_startup_info()  # 创建启动信息
    ├─ socketpair()  # 创建父子进程通信 socket
    ├─ SERVER_START_REQ(new_process)  # 请求 wineserver 创建进程
    └─ exec_wineloader()  # 执行 wine 加载器
        ↓
        ├─ get_alternate_wineloader()  # 备用加载器
        └─ preloader_exec()  # 预加载器执行
```

### 关键发现
- 第 748-749 行：`FIXME("unhandled input attribute %lx\n", ...)`
  - **很多 PS_ATTRIBUTE 属性未实现！**
  - CEF 可能用到某些未实现的属性
- 第 717-718 行：`FIXME( "Unsupported thread flags %#x.\n", ...)`
  - 线程创建标志支持不完整

### 可魔改性：⭐⭐⭐⭐⭐（5星 - 最值得改）
- 直接修改 `NtCreateUserProcess` 实现
- 补充未实现的 `PS_ATTRIBUTE`
- 修复 CEF 依赖的进程创建参数

---

## 2. SuppressAltLoader / Alternate Loader

### 代码位置
| 文件 | 位置 | 说明 |
|------|------|------|
| `dlls/ntdll/unix/loader.c` | 第 412 行 | `get_alternate_wineloader()` |
| `dlls/ntdll/loader.c` | 第 3457 行 | `LdrLoadDll()` DLL 加载 |

### 关键发现
- **Wine 的 "alternate wineloader" 不是 CrossOver 的 "Alternate Loader"**
  - Wine 的 alternate wineloader = 备用架构的 wine 加载器（WOW64 用）
  - CrossOver 的 Alternate Loader = DLL 加载拦截/替换机制
  - **两者完全不是一回事！**

- **DLL 加载入口在 `LdrLoadDll()`**（`dlls/ntdll/loader.c` 第 3457 行）
  - 可以在这里加钩子，针对特定进程名加载不同的 DLL
  - 类似 CrossOver 的 cxcompatdb 做的事情

### 可魔改性：⭐⭐⭐（3星）
- 可以修改 `LdrLoadDll()` 添加进程特定的 DLL 覆盖
- 但需要自己实现"兼容性数据库"逻辑
- CrossOver 的 SuppressAltLoader 是更复杂的机制

---

## 3. apply_in_process_hacks（进程内 hack）

### 代码位置
| 文件 | 位置 | 说明 |
|------|------|------|
| `dlls/ntdll/loader.c` | - | 进程初始化（LdrInitializeThunk） |
| `dlls/ntdll/unix/loader.c` | - | Unix 端加载逻辑 |
| `dlls/kernel32/process.c` | - | 进程初始化 |

### 关键发现
- **Wine 中没有类似 "in_process_hacks" 的机制**
- 但可以在以下位置注入 hack：
  1. `LdrInitializeThunk` - 进程/线程初始化时
  2. `LdrLoadDll` - DLL 加载时
  3. `NtCreateUserProcess` - 进程创建时

- **可能的 hack 方式**：
  - 检测进程名（steamwebhelper.exe / steam.exe）
  - 修改内存中的函数指针
  - Hook 特定 Windows API
  - 修改 PE 导入表

### 可魔改性：⭐⭐⭐（3星）
- 技术上可行，但需要具体知道 hack 什么
- 需要逆向 CrossOver 的 cxcompatdb.so 才能知道具体改了什么
- 工作量较大

---

## 4. cxcompatdb 兼容性数据库

### 代码位置
| 文件 | 位置 | 说明 |
|------|------|------|
| `dlls/apphelp/apphelp.c` | 全文 | apphelp.dll（Windows 兼容性数据库） |

### 关键发现
- **Wine 的 apphelp.dll 几乎全是 stub！**
  ```c
  BOOL WINAPI ApphelpCheckInstallShieldPackage(...) {
      FIXME("stub: ...");  // 占位符，什么都不做
      return TRUE;
  }
  ```
  - `SdbCreateDatabase` - stub
  - `SdbInitDatabase` - stub
  - `ShimFlushCache` - stub
  - 等等...全部都是 stub！

- **CrossOver 的 cxcompatdb.so = 自己实现的兼容性数据库**
  - 不依赖 apphelp.dll
  - 直接在 ntdll 层面集成
  - 包含程序识别规则 + 针对性修复

### 可魔改性：⭐⭐（2星）
- 可以完善 apphelp.dll，但工作量巨大
- 或者自己做一个简化版的兼容性数据库
- 最简单的方式：在 Wine 源码里硬编码针对 Steam 的特殊处理

---

## 5. Apple GPTK / D3DMetal

### 代码位置
| 文件 | 位置 | 说明 |
|------|------|------|
| `dlls/d3d11/` | - | D3D11 实现（Wine 内置） |
| `dlls/dxgi/` | - | DXGI 实现 |
| `dlls/d3d10core/` | - | D3D10 核心 |
| `dlls/wined3d/` | - | Wine Direct3D 核心（OpenGL 后端） |

### 关键发现
- **Wine 内置的 D3D 实现 = WineD3D（OpenGL 后端）**
  - 性能一般
  - 兼容性不如 DXVK 或 D3DMetal

- **DXVK 的集成方式** = DLL 覆盖
  - 替换 d3d11.dll、dxgi.dll 等
  - 通过 `WINEDLLOVERRIDES` 环境变量指定
  - Whisky 已经集成了 DXVK ✅

- **Apple GPTK / D3DMetal**
  - Apple 官方的 D3D→Metal 翻译层
  - CrossOver 专有授权
  - 不开源，无法集成
  - 但 GPTK 是 Apple 出的，理论上可以单独下载集成（需研究）

### 可魔改性：⭐（1星）
- D3DMetal 是专有技术，无法直接用
- GPTK 可能可以单独集成，但需要研究
- WineD3D 可以改，但工作量巨大
- DXVK 已经集成了，效果比 WineD3D 好

---

## 6. DXVK 集成

### 代码位置
| 文件 | 位置 | 说明 |
|------|------|------|
| `dlls/d3d11/d3d11_main.c` | - | Wine 内置 D3D11 |
| `dlls/dxgi/` | - | Wine 内置 DXGI |
| `dlls/d3d9/` | - | Wine 内置 D3D9 |

### 关键发现
- **DXVK 的工作原理** = 原生 DLL 替换
  - DXVK 提供自己编译的 d3d9.dll / d3d11.dll / dxgi.dll
  - 这些是 Windows DLL（PE 格式），在 Wine 中运行
  - 通过 Vulkan 渲染（macOS 上通过 MoltenVK）

- **Whisky 已经集成了 DXVK** ✅
  - 在 DXVK 目录中
  - 通过 WINEDLLOVERRIDES 加载
  - 工作正常

### 可魔改性：无需修改（已集成）

---

## 综合评估

### 优先级排序（从高到低）

| 优先级 | 功能 | 难度 | 预期效果 | 建议 |
|--------|------|------|---------|------|
| 1️⃣ 最高 | NtCreateUserProcess | 中高 | ⭐⭐⭐⭐⭐ | 最值得改，CEF 多进程核心 |
| 2️⃣ 高 | 进程名检测 + 特殊处理 | 中 | ⭐⭐⭐⭐ | 类似 cxcompatdb，硬编码 Steam 规则 |
| 3️⃣ 中 | LdrLoadDll DLL 覆盖增强 | 中 | ⭐⭐⭐ | 针对 CEF 进程替换特定 DLL |
| 4️⃣ 低 | 完善 apphelp.dll | 极高 | ⭐⭐ | 工作量太大，不如硬编码 |
| 5️⃣ 最低 | D3DMetal / GPTK | 极高 | ⭐ | 专有技术，无法集成 |
| 6️⃣ 已完成 | DXVK 集成 | - | - | 已集成 |

### 推荐魔改路线

**第一阶段（最有价值）**：
1. 在 `NtCreateUserProcess` 中添加 CEF 兼容修复
2. 添加进程名检测（steamwebhelper.exe / steam.exe）
3. 针对 Steam 进程做特殊处理

**第二阶段（进阶）**：
1. 完善未实现的 `PS_ATTRIBUTE`
2. 在 `LdrLoadDll` 中添加 DLL 覆盖钩子
3. 修复 CEF 依赖的其他系统调用

**第三阶段（长期）**：
1. 建立简化的兼容性数据库
2. 支持更多程序的针对性修复

---

## 研究文件位置
- 本报告：`sine/crossover-reference/Wine魔改研究报告.md`
- CrossOver 参考文件：`sine/crossover-reference/`
- Wine 源码：`sine/`

## 研究时间
2026-07-06
