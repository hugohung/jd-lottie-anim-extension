# Lottie 静帧合并动效 · AI 自查表

> 每次生成或修改 merged Lottie JSON 后，按此表逐项自查。
> 后续出现新问题，追加到对应分类下并更新版本号。

**版本**: v1.4 | **日期**: 2026-06-29

---

## 一、图层元数据提取（extract_layer_meta）

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 1.1 | **position 是否完整提取** | 元素位置偏移 | 只取了 `ks.p.k` 的前2分量 | `len(p) >= 2`，保留3分量到 meta |
| 1.2 | **anchor 是否完整提取** | 元素缩放/旋转中心错误 | 只取了 `ks.a.k` 的前2分量 | 保留3分量 `[ax, ay, az]` 到 meta |
| 1.3 | **scale 是否保留3分量** | 缩放异常 | `[sx, sy, 100]` 被截断为 `[sx, sy]` | meta 中 scale 必须为 `[sx, sy, sz]` |
| 1.4 | **rotation 是否提取** | 元素旋转丢失 | `ks.r.k` 被忽略 | meta 中包含 `rotation` 字段 |
| 1.5 | **parent 是否提取** | 子元素位置飞到画布外 | `layer.parent` 被忽略，子元素局部坐标被当作绝对坐标 | meta 中包含 `parent` 字段（None 或 int） |
| 1.6 | **cl 是否提取** | 混合模式/样式丢失 | `layer.cl` 被忽略 | meta 中包含 `cl` 字段 |
| 1.7 | **ty=4 形状层的 shapes 是否提取** | 渲染器卡死（一直"加载中"）| `extract_layer_meta` 忽略了 `shapes` 字段 | meta 中包含 `shapes` 列表（完整保留源文件结构）|
| 1.8 | **ty=0 预合成层的 w/h 是否提取** | 渲染器卡死 | `extract_layer_meta` 忽略了 `w`/`h` 字段 | meta 中包含 `asset_w` / `asset_h`（ty=0 时从图层直接取 w/h）|

---

## 二、静态图层 diff 识别（layer_same）

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 2.1 | **name 是否相同** | 误判为不同图层 | 名称不完全匹配 | 精确字符串比较 |
| 2.2 | **parent 是否相同** | 同名但父子关系不同被判为同一图层 | diff 未比较 parent | `la['parent'] == lb['parent']` |
| 2.3 | **rotation 是否相同** | 同名但旋转不同被判为静态层 | diff 未比较 rotation | `abs(la['rotation'] - lb['rotation']) < 0.01` |
| 2.4 | **position/anchor/scale 容差** | 浮点误差导致误判 | 无容差比较 | 各维度 `abs(diff) < 0.1` |

---

## 三、关键帧生成（build_pos_kfs / build_opacity_kfs）

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 3.1 | **关键帧按 t 升序排列** | 渲染只显示背景/最后一帧，动效不播放 | keyframe 数组未排序 | 生成完 keyframes 后 `kfs.sort(key=lambda k: k['t'])` |
| 3.2 | **禁止 hold keyframe (h:1)** | 动效像幻灯片，无平滑过渡 | h:1 阻止插值 | 所有 keyframe **不设 h 属性**（默认 h:0 平滑插值） |
| 3.3 | **退场 opacity 无条件生成** | Scene B 元素在退场阶段不消失 | `initial_visible=False` 时 exit kf 被跳过 | exit opacity kf 始终生成，不受 initial_visible 影响 |
| 3.4 | **退场 position 起始值** | 退场时元素从错误位置飞出 | exit_start 的 position 用了 entry_x 而非原始 x | exit_start kf 的值始终为原始 `[x, y]` |
| 3.5 | **飞行距离基于视觉边界** | 元素飞出距离过大，飞到画布外 | 用 `CANVAS_W + margin` 绝对偏移 | 基于 `visual_left = pos_x - anchor_x * scale_x` 计算相对偏移 |
| 3.6 | **scale 归一化用于计算** | 飞行距离计算用原始 scale 值 | scale 值是百分比 | 计算前 `scale_x = s[0] / 100.0`, `scale_y = s[1] / 100.0` |
| 3.7 | **center 方向元素交叉溶解（无空窗）** | center 元素在切换时闪烁/短暂消失（几帧空白）| B 入场等 A 完全退场后才开始，产生空窗期 | center 方向的 B 在 A **退场开始时刻**就入场（`cf_start=对方exit_s`），A 同理；两者反向渐变实现交叉溶解 |
| 3.8 | **时间轴 A/B 对称** | A 静置时间远短于 B（如 0.2s vs 3.5s） | `T_A_END`/`T_B_END` 不对称 | T_A_END ≈ T_TOTAL - T_B_END，两画面静置时长接近 |

---

## 四、图层排序（layers 数组）

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 4.1 | **背景必须在 layers 末尾** | 只看到背景，前景被遮挡 | Lottie 渲染：layers[0] = 最上层 | 背景层追加到数组末尾（最大 ind） |
| 4.2 | **顶层静态（腰封）在数组首** | 前景元素遮盖了腰封/UI框 | 腰封 ind 不是最小 | 静态层中 ind 小的放数组前面 |
| 4.3 | **同场景前景保持原始 ind 顺序** | 前景元素前后遮挡关系错乱 | 合并时打乱了原始 ind 排序 | static_top → Scene B FG(sorted by ind) → Scene A FG(sorted by ind) → static_bottom |
| 4.4 | **ind 唯一且从1开始连续** | Lottie 解析异常 | 合并后 ind 有重复或不连续 | 最终 `for i, l: l['ind'] = i + 1` |

---

## 五、parent 父子关系

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 5.1 | **parent 值是否输出到 layer** | 子元素位置错误 | make_layer 未包含 parent 字段 | 输出 layer 包含 `"parent": <ind>` |
| 5.2 | **parent ind 是否 remap** | 子元素指向错误的父级 / 父级不存在 | 图层排序后 ind 变化，parent 仍引用旧的 ind | 建立 `(source_tag, source_ind) → new_ind` 映射，remap 所有 parent |
| 5.3 | **parent remap 跨场景检查** | 父级在 A 场景，子元素被误认为是独立层 | 不同场景同名图层可能有不同 parent | 每个场景独立维护 `_source_tag`，remap 按 tag 分区 |

---

## 六、资产（assets）合并

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 6.1 | **重复 asset 去重** | 输出文件膨胀 | 两场景中共用素材重复存储 | 基于 `(w, h, base64前100字符)` 签名去重 |
| 6.2 | **refId 正确映射** | 图层引用错误图片 | asset 的 id 未更新到新映射 | `asset_map[(source_tag, orig_refid)] → new_refid`，图层 refId 使用新 id |
| 6.3 | **不同场景同名 refId 隔离** | 场景A的 image_2 和场景B的 image_2 混淆 | 仅用 refId 做 asset key | 使用 `(source_tag, refId)` 作为 asset_map key |

---

## 七、首尾衔接循环

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 7.1 | **frame 0 和最后一帧状态一致（无条件锚点）** | 循环跳跃/闪烁；即使数据值相等也可能因插值异常闪烁 | 仅检查 `v_start == v_last` 不够，lottie 在 OP 跳回 IP 时需要显式锚点 | **无条件**为每个动画属性在 `t=OP` 处补入一个关键帧（值 = t=0 的值），不判断是否相等；输出日志应报告补入的锚点数量 |
| 7.2 | **totalFrames 足够容纳完整动效** | 动效未播完就跳到开头 | op 设置过小 | `op >= entry_end + hold_duration` |
| 7.3 | **center 元素交叉溶解不产生空窗期** | 文案/中心内容短暂消失（"闪一下"）| A 消失后 B 延迟才出现（或反向）| crossfade 窗口对齐 non-center 飞入时间(cf_ab_s=F_B_ENTER_S)，A/B 同时刻反向渐变，零透明度重叠 |

---

## 八、输出 JSON 完整性

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 8.1 | **非动画属性用 `"a": 0`** | 渲染器行为异常 | 静态属性（anchor/scale/rotation）设了 `"a": 1` | 仅 position 和 opacity 设 `"a": 1` |
| 8.2 | **position/anchor 值必须是3元素 [x,y,z]** | **lottie-web 崩溃：Cannot read properties of undefined (reading 'length')** | `extract_layer_meta` 只取了 `[p[0], p[1]]` 2分量；`build_pos_kfs` 中 `make_kf(0, [x,y])` 只传2元素 | **所有 position 和 anchor 值（无论动画关键帧还是静态）都必须是 `[x, y, z=0]` 三元组**，Lottie 规范要求如此 |
| 8.3 | **opacity keyframe s 值为单元素数组** | 渲染器报错 | s 值为裸数字 `100` | `s` 始终为数组 `[100]` 或 `[0]` |
| 8.4 | **`build_fg_keyframes` 中转时字段完整性** | 渲染器卡死（形状层/预合成层结构不完整） | 该函数创建新字典时只复制了部分字段，`shapes`/`w`/`h` 在中转过程中丢失 | `build_fg_keyframes` 返回的每个元素必须包含 `shapes`（ty=4 时）和 `w`/`h`（ty=0 时），与 `extract_layer_meta` 提取的字段一一对应 |
| 8.5 | **预览 HTML CDN 可用性** | 预览页面空白/卡在"加载中" | 使用 cdnjs.cloudflare.com 在国内企业网被墙/超时，lottie 库无法加载 | 优先 jsdelivr + cdnjs 备用；用 fetch 异步加载 JSON（非内嵌） |
| 8.6 | **预览下载按钮可用性 + 全局作用域** | 点击下载/播放/暂停等所有按钮均无反应 | ① `<a>` 未 appendChild 到 DOM 就 click；② anim/ss/dlJson 被包在 IIFE 闭包内，onclick 访问不到局部变量 | **①** `document.body.appendChild(a)` 后再 `a.click()`；**②** 禁用 IIFE，所有 onclick 调用的函数必须是全局变量（`var anim=null` 在顶层作用域） |
| 8.7 | **预览 JS 不使用 IIFE 闭包** | 所有按钮失效（播放/暂停/下载/速度切换全部不响应） | `anim`/`ss`/`dlJson`/`doPlay`/`doPause`/`doReplay` 定义在 `(function(){...})()` 内部，HTML 的 `onclick="..."` 只能访问全局作用域 | 预览 HTML 的 `<script>` 中直接声明全局变量和全局函数，不用 IIFE 包装 |

---

## 自查流程

**生成完成后，AI 必须按顺序检查：**

1. **运行脚本，查看 stdout 日志** → 确认静态层数、前景层数、图层排序正确
2. **对比源文件与输出** → 抽查至少3个图层（含一个有 parent 的元素）的 position/anchor/scale/rotation/parent/cl
3. **在预览 HTML 中验证** → 观察第0帧（Scene A 静置状态）与源文件第0帧是否一致
4. **验证循环** → 播放2遍，确认首尾衔接无跳跃

---

## 修订记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v1.4 | 2026-06-29 | **新增 7.3（crossfade零空窗）+ 8.7（IIFE闭包导致所有按钮失效）**；修下载按钮(window.open兜底)；加循环衔接自动验证+修正逻辑；non-center opacity与position窗口对齐 |
| v1.3 | 2026-06-26 | **新增 3.7（center元素交叉溶解防闪烁）+ 3.8（时间轴对称）+ 8.5（CDN可用性）+ 8.6（下载按钮DOM修复）**；修复预览空白(CDN)、文案闪烁(空窗期)、下载失效(未挂DOM) |
| v1.2 | 2026-06-18 | **新增 1.7/1.8（shapes/w/h 字段提取）+ 8.4（build_fg_keyframes 中转字段完整性）**，修复 LottieLab 导出文件合并后白屏/卡死问题（根因确认） |
| v1.1 | 2026-06-18 | 修正 8.2：position/anchor 必须是3元素 [x,y,z]，2元素会导致 lottie-web 崩溃（根因确认） |
| v1.0 | 2026-06-18 | 初始版本，收录全部已修复的12类错误 |
