---
name: jd-lottie-anim-extension
description: 将两个静帧 Lottie JSON 合并为带切帧动效的 Lottie JSON，自动识别静态图层并生成平滑的入场/退场动画。
version: "8.0.0"
author: honghaoxiang
agent_created: true
trigger:
  - Lottie动效
  - 合并Lottie
  - 切帧动效
  - Lottie自动动效
  - Lottie静帧合并
  - 自动生成Lottie动画
---

# Lottie 静帧合并动效 — AI 执行指南

## 一句话说明

把两张相同背景、不同前景的 Lottie 静帧图，合成一个循环播放的切换动效。

## 快速执行

```bash
python scripts/generate_merged_lottie.py <场景A.json> <场景B.json> [输出目录]
```

**必须遵守的规则：**

1. **Python 路径**：用 `C:\Users\honghaoxiang\.workbuddy\binaries\python\versions\3.13.12\python.exe`
2. **输出目录默认值**：不传第三个参数时，输出到脚本所在目录的上级
3. **运行后必须做两件事**：
   - 启动 HTTP 服务器让用户预览（见下方"预览方法"）
   - 对照自查表逐项检查（`references/LOTTIE_BUG_CHECKLIST.md`）

## 输入要求

| 条件 | 说明 | 不满足时怎么处理 |
|------|------|-----------------|
| 尺寸一致 | 两个 JSON 的 `w` 和 `h` 必须相同 | 脚本以文件A为准，警告用户 |
| 格式合法 | 必须是标准 Lottie JSON（含 v/fr/w/h/layers） | 报错退出 |
| 编码 UTF-8 | 文件编码必须是 UTF-8 | 脚本已处理，无需额外操作 |

**典型输入示例**（见 `examples/` 目录）：
- `scene-a.json` — 初始状态（如电商横幅第一屏）
- `scene-b.json` — 变化状态（如电商横幅第二屏）

## 输出物

运行成功后，输出目录会生成：

```
输出目录/
├── merged_output.json   # 合并后的 Lottie 动效文件 ← 用户最终要的这个
└── preview.html         # 预览页面（含播放/暂停/下载按钮）
```

## 预览方法（重要！）

preview.html **不能直接双击打开**，必须通过 HTTP 服务器访问：

```bash
# 在输出目录启动 HTTP 服务器（端口 8770）
cd 输出目录 && python -m http.server 8770

# 浏览器访问
http://localhost:8770/preview.html
```

**预览功能清单：**
- ▶/⏸ 播放/暂停切换按钮
- 🔄 重播按钮
- ⏩ 0.5x / 1x / 2x 速度切换
- 📥 下载 JSON 按钮（通过 FileSaver.js 确保文件名正确）
- 📊 实时帧数显示

## 工作流程（完整步骤）

当用户说"帮我合并这两个 Lottie"时，按以下顺序执行：

### Step 1: 确认输入文件

```bash
# 检查文件是否存在且是合法 Lottie JSON
python3 -c "
import json, sys
for f in sys.argv[1:]:
    try:
        d = json.load(open(f, 'r', encoding='utf-8'))
        print(f'✅ {f}: v={d.get(\"v\")} fr={d.get(\"fr\")} {d.get(\"w\")}x{d.get(\"h\")} layers={len(d.get(\"layers\",[]))}')
    except Exception as e:
        print(f'❌ {f}: {e}')
" scene-a.json scene-b.json
```

### Step 2: 运行脚本

```bash
python scripts/generate_merged_lottie.py scene-a.json scene-b.json ./output
```

### Step 3: 检查脚本输出

脚本会在终端打印关键信息：
```
FPS=30  v=5.6.10  W=750  H=700
TOTAL=150f (5.0s)
Static (N) — 应包含背景层
Scene A FG (M) / Scene B FG (K) — 前景变化图层
✅ All refIds valid
✅ 循环衔接: 为所有动画属性在 t=TOTAL 处补入首帧锚点 (共 N 处)
```

**异常信号（需要排查）：**
- `Static (0)` 且预期应有背景 → 源文件的背景图层位置/内容不完全匹配
- `Missing refIds` → asset 引用断裂
- 图层数量异常少 → 可能是 ty 类型未覆盖

### Step 4: 启动预览

```bash
cd ./output && python -m http.server 8770
```

浏览器打开 `http://localhost:8770/preview.html`

### Step 5: 自查验证

打开 `references/LOTTIE_BUG_CHECKLIST.md`，逐项核对。重点检查：
1. 版本号是否从源文件读取（不是硬编码）
2. 帧率是否正确
3. 是否有 refId 缺失
4. 循环衔接锚点是否已补入
5. position 是否是 3 分量 [x, y, z]

### Step 6: 交付用户

确认无误后：
1. 告知用户预览地址
2. 说明如何下载 JSON
3. 如需推送到 GitHub，确认后再推送

## 核心技术约定（修改代码前必读）

### 时间轴设计（V8 回归 V6 简单逻辑）

```
时间轴（秒）→ 帧数（@30fps为例）
─────────────────────────────────────
0.0 ──── 1.0s   场景 A 静置          (0-30f)
1.0 ──── 1.5s   A 退场 + B 入场       (30-45f) ← 同一窗口！
1.5 ──── 3.5s   场景 B 静置          (45-105f)
3.5 ──── 4.0s   B 退场 + A 入场       (105-120f) ← 同一窗口！
4.0 ──── 5.0s   场景 A 静置(循环)    (120-150f)
─────────────────────────────────────
总时长 5.0s，无缝循环
```

**关键原则：A 退场和 B 入场在同一窗口**，center 元素自然交叉溶解，不需要特殊处理。

### 绝对禁止做的事

| 禁止项 | 原因 | 正确做法 |
|--------|------|----------|
| 关键帧设 `h:1` | 导致渲染器不插值 | 不设置 h 属性（默认 h=0） |
| position 用 2 元素 `[x,y]` | lottie-web 崩溃 | 始终 3 元素 `[x,y,z=0]` |
| scale 用 2 元素 `[sx,sy]` | 同上 | 始终 3 元素 `[sx,sy,sz]` |
| IIFE 包裹 JS 函数 | onclick 访问不到闭包变量 | 所有函数声明为全局变量 |
| 循环衔接只做相等性检查 | 即使值相等也可能插值异常 | 无条件在 t=OP 补入首帧锚点 |
| 硬编码版本号/帧率/尺寸 | 不同源文件参数不同 | 全部从源文件动态读取 |
| 依赖图层名(`nm`)做匹配 | AE/Lottielab 导出后 nm 常为空 | 用 7 维度变换属性匹配 |

### 图层排序规则

Lottie 的 layers 数组：**index 0 = 最上层，末尾 = 最底层**

```
输出顺序：
[0..top]     静态顶层（腰封等覆盖层，ind 小的优先）
[top+1..mid]  Scene B 前景（B 盖住 A）
[mid+1..bot]  Scene A 前景
[bot+1..end]  静态底层（背景，ind 大的在后）
```

## 自定义配置

如需调整动效参数，编辑 `scripts/generate_merged_lottie.py` 中的以下区域：

```python
# === 时间轴（秒）===
T_TOTAL  = 5.0   # 总时长
T_A_HOLD = 1.0   # A 静置结束
T_SWITCH1 = 1.5  # A→B 切换结束
T_B_HOLD = 3.5   # B 静置结束
T_SWITCH2 = 4.0  # B→A 切换结束

# === 淡入淡出帧数 ===
FADE = 8         # opacity 过渡帧数

# === 缓动曲线 ===
EASE_OUT     = {"x": [0.333], "y": [0.0]}
EASE_IN      = {"x": [0.667], "y": [1.0]}
EASE_SNAPPY_I = {"x": [0.667], "y": [0.667]}  # 弹性缓入
EASE_SNAPPY_O = {"x": [0.333], "y": [0.333]}  # 弹性缓出
```

## 目录结构

```
jd-lottie-anim-extension/
├── SKILL.md                        # 本文件（AI 指令）
├── README.md                       # 人读文档（GitHub 展示用）
├── LICENSE                         # MIT 协议
├── scripts/
│   └── generate_merged_lottie.py   # 主脚本
├── references/
│   ├── .gitkeep
│   └── LOTTIE_BUG_CHECKLIST.md    # 自查表（生成后逐项核对）
└── examples/
    ├── .gitkeep
    ├── scene-a.json                # 示例：初始状态
    ├── scene-b.json                # 示例：变化状态
    ├── expected-output.json        # 示例：预期输出
    └── preview.html                # 示例：预览页面
```

## 版本历史

| 版本 | 日期 | 主要变更 |
|------|------|----------|
| V8.0 | 2026-06-29 | 预览优化：播放/暂停合并为toggle按钮；下载改用FileSaver.js修复文件名乱码；回归V6简单时间轴消除闪烁；重构skill为标准目录结构 |
| V7.x | 2026-06-24~26 | 7维度静态识别、center交叉溶解、CDN修复、IIFE闭包问题、循环衔接无条件锚点 |
| V6 | 2026-06-18 | position/scale 3分量修复、视觉边界飞行距离、弹性缓动曲线 |
| V1-V5 | 2026-06-17~18 | 基础功能、parent保留、anchor/rotation/cl修复 |

## 排错速查

| 症状 | 最可能原因 | 解决方法 |
|------|-----------|---------|
| 预览白屏 | CDN 加载失败 | 检查网络；预览页有双CDN兜底 |
| 按钮无效 | JS 函数在 IIFE 内 | 确认所有函数是全局声明 |
| 闪烁 | 循环衔接缺锚点 | 确认 t=OP 处有无条件首帧锚点 |
| 元素位置偏移 | anchor 丢失或2元素position | 检查源文件anchor是否保留、position是否3元素 |
| 下载文件名乱码 | blob URL 的 a.download 失效 | 使用 FileSaver.js 的 saveAs() |
| 背景消失 | 背景被识别为前景 | 检查两源文件的背景层位置/内容是否完全匹配 |
