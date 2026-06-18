# Lottie 静帧合并动效 · AI 自查表

> 每次生成或修改 merged Lottie JSON 后，按此表逐项自查。
> 后续出现新问题，追加到对应分类下并更新版本号。

**版本**: v1.0 | **日期**: 2026-06-18

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
| 7.1 | **frame 0 和最后一帧状态一致** | 循环跳跃/闪烁 | Scene A 入场未完成，最后一帧和第一帧不同 | 最后一帧 Scene A 所有元素已到达静置位置，opacity=100 |
| 7.2 | **totalFrames 足够容纳完整动效** | 动效未播完就跳到开头 | op 设置过小 | `op >= entry_end + hold_duration` |

---

## 八、输出 JSON 完整性

| # | 检查项 | 症状 | 根因 | 检查方式 |
|---|--------|------|------|----------|
| 8.1 | **非动画属性用 `"a": 0`** | 渲染器行为异常 | 静态属性（anchor/scale/rotation）设了 `"a": 1` | 仅 position 和 opacity 设 `"a": 1` |
| 8.2 | **position keyframe s 值为2D** | 渲染器报错 | s 值带了3分量 `[x, y, 0]` | 动画 keyframe 的 `s` 统一为 `[x, y]`（2D）；静态属性 `"a": 0` 的 `k` 为 `[x, y, 0]` |
| 8.3 | **opacity keyframe s 值为单元素数组** | 渲染器报错 | s 值为裸数字 `100` | `s` 始终为数组 `[100]` 或 `[0]` |

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
| v1.0 | 2026-06-18 | 初始版本，收录全部已修复的12类错误 |
