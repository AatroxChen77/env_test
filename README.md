# Python环境依赖导出实验

这个项目是一个关于Python环境依赖导出方法的实验，测试了不同的conda和pip命令导出环境依赖的效果和差异。

## 项目概述

本项目通过对比不同的环境导出命令，分析了各种方法的优缺点，为Python环境管理提供了实用的参考。特别关注了conda和pip混用环境下的依赖导出问题。

## 实验环境

- **Python版本**: 3.8.20
- **包管理器**: Conda + Pip
- **Conda版本**: 24.11.3
- **操作系统**: Windows 10
- **实验环境名**: envtest/envtest2

## 前言

### conda 和 pip 混用

实际使用中，经常会出现`conda` 和 `pip` 混用的情况。

> **建议**：最好先 `conda install`，不行再使用 `pip install`安装。
> 因为**`conda install`** 能解析跨语言依赖，降低冲突概率。

但这样环境中会同时存在通过两者安装的包，所以导出环境依赖时，要注意导出所有包（`conda`和`pip`的）

### 输出文件格式

输出时，最好使用 `environment.yml` 文件，而不是 `environment.txt`，尤其是在 `conda` 环境管理的上下文中。

- `conda` 官方推荐使用 `.yml`（YAML 格式）来导出和共享环境配置。

> To share an environment and its software packages, you must export your environment's configurations into a **`.yml`** file.
> https://www.anaconda.com/docs/tools/working-with-conda/environments#sharing-an-environment

不过实测其实也没啥区别，但是.yml更规范。

## 构建信息（Build String）详解

### 什么是Build String？

一般导出的依赖文件会带有包的构建信息（build string）。它通常包括：

- Python 版本
- 包的构建版本号
- 其他一些特定的环境和系统信息（如系统架构或特定依赖库的版本）

例如：使用`conda list --export > environment.yml`

```yaml
# 就是`=` 后面的部分
- bleach=6.2.0=py311h06a4308d5_0
- bottleneck=1.4.2=py311hf4808d0_0
- brotli-python=1.0.9=py311h6a678d5_9
```

这里的 `6.2.0=py311h06a4308d5_0` 中：
- `6.2.0` 是包的版本
- `py311` 表示这个包是为 Python 3.11 构建的
- `h06a4308d5_0` 是构建信息，通常与包的构建系统、系统架构、依赖等相关

### 为什么保留Build String可能会导致安装错误？

不同操作系统、不同环境下的构建信息可能不同，保留这个构建信息直接用来安装可能会导致安装失败。

1. **二进制不兼容**
   - Ubuntu 的包编译用的是 Linux 工具链（GCC, GLIBC）
   - Windows 需要 MSVC 编译的二进制
2. **依赖关系不同**：
   - Linux 包依赖 **`libopenblas.so.0`**
   - Windows 需要 **`.dll`** 文件
3. **Conda 的严格匹配**：
   Conda 会尝试精确匹配 **`包名=版本=build_string`**，而 Windows 根本没有这个 build string 的包

一般 `conda` 会根据目标环境**自动选择**合适的构建版本，所以其实绝大部分情况不需要build string。

### 如何删除Build String？

除了手动删除build string，还有两种解决方案：

**正则表达式删除**：
```
(={1}[^=]+)$
```
正则查找然后替换为空即可。

**导出时不带Build String**：
```bash
# 移除平台特定标记（提高跨平台兼容性）
conda env export --no-builds > environment.yml
```

1. **Conda 的智能解析**：
   - 当没有 build string 时，Conda 会根据**当前平台**自动选择兼容的构建
   - Windows 会找到基于 MSVC 编译的版本
   - Linux 会找到基于 GLIBC 的版本
2. **依赖关系自动适配**：
   Conda 会自动处理平台特定的依赖：

   | **包** | **Linux 依赖** | **Windows 依赖** |
   | --- | --- | --- |
   | numpy | libopenblas, glibc | mkl, vc_runtime |
   | scipy | libgfortran | libflang |

### 安装时强制忽略Build String

```bash
conda env create -f environment.yml --force
# 然后手动更新：
conda update --all --no-build-locked
```

## 各种导出指令分析

### 1. conda env export (environment.yml & environment3.yml)

**命令**: `conda env export > environment.yml`

**特点**:
- ✅ 导出格式规范，包含环境名称、channels和dependencies
- ✅ 清晰区分conda安装和pip安装的包
- ✅ 包含完整的依赖信息
- ✅ 可以直接用于环境重建
- ✅ 官方推荐的标准方法

**导出内容**:
- 环境名称和channels配置
- conda安装的包（包括系统依赖）
- pip安装的包（在pip子节点下）

**最佳实践**:
```bash
conda env export --no-builds > environment.yml
```

### 2. conda list --export (environment1.yml)

**命令**: `conda list --export > environment1.yml`

**特点**:
- ✅ 包含所有包（conda + pip安装的）
- ✅ 通过 **`pip`** 安装的包在输出中会被标记为 **`<pypi>`**（在 **`Channel`** 列）
- ❌ 格式不够规范，只是简单的包列表
- ❌ 不区分包的安装来源
- ❌ 缺少环境配置信息

**发现**: 导出的包与`conda list`显示完全一致，包括setuptools、pip和wheel等基础包。

**示例输出**:
```
# Name                    Version                   Build  Channel
_ipyw_jlab_nb_ext_conf    0.1.0            py39haa95532_0    defaults
absl-py                   2.1.0                    pypi_0    pypi # 通过pip安装的
alabaster                 0.7.12             pyhd3eb1b0_0    https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
```

### 3. conda list --explicit (environment2.yml)

**命令**: `conda list --explicit > environment2.yml`

**特点**:
- ✅ 提供包的完整下载URL
- ❌ 只包含conda安装的包
- ❌ 不包含pip安装的包
- ❌ 导出不完整

**发现**: 只导出通过conda install安装的包，pip install的包不会包含在内。

### 4. pip freeze (pip_requirements.txt)

**命令**: `pip freeze > pip_requirements.txt`

**特点**:
- ✅ 格式简洁，适合pip环境
- ❌ 只包含pip安装的包
- ❌ 缺少conda安装的包
- ❌ 不包含setuptools、pip、wheel等基础包

**发现**: 
- pip freeze导出的包比pip list显示的少
- 缺少setuptools、pip、wheel等基础包
- 包含一些本地路径的包引用

## 实验结论

### 最佳实践推荐

**推荐使用**: `conda env export --no-builds > environment.yml`

**原因**:
1. **完整性**: 包含所有包（conda + pip安装的）
2. **规范性**: 格式标准，包含环境配置信息
3. **可读性**: 清晰区分包的安装来源
4. **实用性**: 可以直接用于环境重建
5. **兼容性**: 支持跨平台迁移

### 重要发现

1. **pip freeze不完整**: 比pip list显示的包少，缺少基础包
2. **conda list --explicit局限性**: 只包含conda安装的包
3. **conda env export最全面**: 包含所有包且格式规范
4. **包来源区分**: conda env export能清楚区分conda和pip安装的包
5. **Build String问题**: 跨平台迁移时可能导致安装失败

## 文件说明

- `environment.yml`: 使用 `conda env export` 导出（含 build string 与 `pip` 段，末尾含 `prefix`，跨机使用前建议删除）
- `environment_nob.yml`: 使用 `conda env export --no-builds` 导出（不含 build string，含 `pip` 段；已去除/注释 `prefix` 行）
- `environment3.yml`: 使用 `conda env export` 的示例（含 build string 与 `pip` 段，含 `prefix`）
- `environment1.yml`: 使用 `conda list --export` 导出（纯包清单，包含 conda 与 pip 安装的包，pip 包在 Channel 标注为 `pypi`）
- `environment2.yml`: 使用 `conda list --explicit` 导出（仅含 conda 安装包的完整 URL，缺少 pip 包）
- `environment_nop.yml`: 空文件（占位/示例）
- `pip_requirements.txt`: 使用 `pip freeze` 导出（仅含 pip 包，可能包含本地路径引用，缺少 setuptools/pip/wheel）

## 使用建议

1. **环境备份**: 使用 `conda env export --no-builds > environment.yml`
2. **环境重建**: 使用 `conda env create -f environment.yml`
3. **纯pip环境**: 使用 `pip freeze > requirements.txt`
4. **混合环境**: 优先使用conda env export --no-builds
5. **跨平台迁移**: 务必使用--no-builds参数

## 注意事项

- 不同操作系统导出的环境文件可能不兼容
- 建议在相同操作系统间迁移环境
- 定期更新环境文件以保持依赖的最新状态
- 在生产环境中使用固定版本号避免兼容性问题
- 跨平台迁移时务必使用`--no-builds`参数
- 如果安装失败，检查是否包含build string，可以使用`--no-build-lock`参数强制忽略

## 其他命令

### conda export（新命令）

**`conda export`** 是新出的命令（所以对conda版要求有要求，建议还是用conda env export），功能更多一些

https://docs.conda.io/projects/conda/en/stable/commands/export.html

所以也可以直接省去`env`：

```bash
conda export > environment.yml
```

但为了兼容性，建议继续使用`conda env export`。 