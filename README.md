# Sine (正弦)

专为游戏优化的 macOS Wine 引擎。

基于 Wine 11.12 源码魔改，重命名为 Sine（正弦），专注于提升游戏兼容性和性能。

## ⚠️ 项目状态

**早期开发阶段**，正在进行 Steam 等游戏的兼容性魔改。

## ✨ 特性

- 🎮 游戏优先优化
- 🚀 性能调优
- 🎯 专为 macOS 优化
- 🍷 基于 Wine 上游源码

## 🏗️ 构建

### 依赖

```bash
brew install llvm bison mingw-w64
```

### 编译

```bash
export PATH="/opt/homebrew/opt/llvm/bin:/opt/homebrew/opt/bison/bin:$PATH"
./configure --enable-win64 --disable-win16 --without-x --without-fontconfig
make -j$(sysctl -n hw.ncpu)
```

### 运行

```bash
./tools/wine/sine --version
./tools/wine/sine notepad.exe
```

## 📋 与上游 Wine 的区别

- 程序名：wine → sine
- 版本字符串：wine-x.x → sine-x.x
- 游戏兼容性魔改（开发中）
- macOS 专项优化

## 📜 许可证

本项目基于 [Wine](https://www.winehq.org/) 源码修改，遵循 **GNU Lesser General Public License v2.1 (LGPL-2.1)**。

详见 [LICENSE](LICENSE) 及 Wine 官方许可证说明。

---

*Sine - 让游戏在 Mac 上正弦运行* 🎮🍎
