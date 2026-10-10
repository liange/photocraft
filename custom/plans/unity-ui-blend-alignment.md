# 与 Unity UI 对齐图层 Alpha 混合：可行性评估

目标：判断能否把 PhotoCraft 的图层合成空间改成与 Unity UI（uGUI）一致，用于"设计稿 = 运行时画面"的
预览/导出，并给出改动点、工作量与风险。
结论先行：**可行，且是"接一根线"级别的改动**——混合空间的数学已经存在、CPU/GPU 已各有实现并对齐测试过，
缺的是把"当前图层 gamma"这一个参数提升为"文档/会话级混合空间"。真正贵的是"精确复刻 Unity 的 8bit
RT 语义"（预乘、目标 alpha、逐层量化），那部分建议只做导出通路。

---

## 1. PhotoCraft 现状（代码证据）

### 1.1 合成发生在**编码值**空间（= Photoshop 默认 = Unity Gamma 空间）

- U8 像素进入合成时只做 `/255`，不做线性化：`crates/raster/src/lib.rs:285-292`。
  只有 CMYK（按文档 profile）和 Lab（转 sRGB）在 `to_rgba` 里做色彩转换（`:609-630`）。
- 合成公式是 W3C/PDF 直通 alpha 的通用式
  `αo = αs + αb(1−αs)`、`αo·Co = (1−αs)αb·Cb + (1−αb)αs·Cs + αs·αb·B(Cb,Cs)`：
  `crates/color/src/blend.rs:413-439`，Photoshop 变体在 `crates/compose/src/psblend.rs:125-162`
  （Normal 有快路径 `out = ((1−as)·ab·Cb + as·Cs)/ao`）。
- Normal + 不透明背景时该式退化为 `(1−as)·Cb + as·Cs`，即编码值的线性插值——和 Unity Gamma 空间里
  `Blend SrcAlpha OneMinusSrcAlpha` 的结果**逐位一致**。

### 1.2 唯一的图层→背景混合漏斗

```
composite_at            crates/compose/src/lib.rs:1217  blend_into_g(.., text_gamma(layer))
composite_atop          crates/compose/src/lib.rs:1574  composite_gamma(.., gamma)
blend_into              crates/compose/src/lib.rs:1580-1582 → blend_into_g(.., 1.0)
blend_into_g            crates/compose/src/lib.rs:1585-1604
图层效果合并             crates/compose/src/effects.rs:1835-1838
```

`text_gamma(layer)`（`lib.rs:1249-1251`）对文字图层返回 Photoshop 的
"Blend Text Colors Using Gamma"（默认 1.45，进程级 atomic），其他图层返回 1。

### 1.3 "在任意编码空间里混合"的机制**已经存在**

```rust
// crates/compose/src/psblend.rs:196-233
pub fn text_encode(v, gamma) -> f32 { srgb_to_linear(v).powf(1.0 / gamma) }
pub fn text_decode(v, gamma) -> f32 { linear_to_srgb(v.powf(gamma)) }
pub fn composite_gamma(mode, b, s, opacity, gamma) -> [f32;4]  // 权重不变，只在 g() 空间里混合
```

- `gamma → 1` 趋近**线性光**，`gamma ≈ 2.2` 趋近编码值混合（注释见 `psblend.rs:190-194`）。
- 它**不是** text-only：`gamma == 1.0` 只是被当作"关闭"的哨兵值提前返回编码混合
  （`psblend.rs:209-211`，语义见 `:173-179`）。这个哨兵正是"线性光 = gamma 1.00"（Adobe 的语义）
  与现有实现冲突的地方，改的时候要一并理顺。
- GPU 侧同款：`crates/gpu/src/compose.wgsl:284-296` `composite_g`（由 `p4.w` 传 gamma +
  `F_TEXT_GAMMA` 开关选择），`crates/gpu/src/plan.rs:372-376` 定义 flag，`:708/:894/:1237/:1250`
  四处塞 `p.extra[3] = text_gamma(layer)`。
- 已有 CPU/GPU 一致性测试模板：`crates/gpu/tests/parity.rs:1337-1359`
  （对 Normal/Multiply/Screen/Color × gamma 1.8/1.0 逐一比对）。
- 文档级"混合空间"已经有了两个布尔先例：Lab 文档的 Normal 在 CIELAB 里混、32bit 的 HDR 不裁剪
  （`crates/compose/src/psblend.rs:74-109` 的 thread-local + `crates/compose/src/lib.rs:187-189`
  `in_blend_space`）。

### 1.4 32bit 文档本来就是线性

`crates/compose/src/adjust.rs:26-38`：`Transfer::for_document` 对 `F32` 返回 `Gamma(1.0)`；
灰度/双色调走 dot-gain 1.732，其余 sRGB。所以"线性光混合"对 32bit 文档必须是**空操作**，
否则会二次转换。

### 1.5 文档里那句"已有 per-document 线性光混合"是错的

`docs/architecture.md:321` 原文声称存在该选项；`crates/doc/src` 无该字段，engine 无该命令，
全仓库只有 `BlendMode::LinearLight`（一个混合模式）——已修正该行。

---

## 2. Unity 侧实际做了什么

`UI/Default`（UI-Default.shader，取自带 MIT 许可的内置 shader 镜像）：

```
ZWrite Off, ZTest [unity_GUIZTestMode], Cull Off
Blend SrcAlpha OneMinusSrcAlpha          // 未分离 alpha 通道
frag: color = (tex2D(_MainTex, uv) + _TextureSampleAdd) * IN.color;   // 直通 alpha，不预乘
```

- **色彩空间**决定混合发生在哪个空间：
  - Project Settings ▸ Player ▸ Color Space = **Gamma**：LDR UNORM 目标，硬件对**编码值**混合
    → 与 Photoshop/PhotoCraft 现状一致。
  - = **Linear**：目标是 sRGB 格式，dest 先解码到线性、混合、再编码写回
    → **线性光混合**。
- **目标 alpha 的坑**：`Blend SrcAlpha OneMinusSrcAlpha` 同时作用于 alpha 通道，于是
  `A_out = As² + Ad(1−As)`（不是 `As + Ad(1−As)`）。渲染到**透明 RT**（UI→RenderTexture、模糊背景、
  后期链路）时，RGB 实际上是"预乘色 + 平方 alpha"。部分 Unity 版本的 UI/Default 改用了
  `Blend SrcAlpha OneMinusSrcAlpha, One OneMinusSrcAlpha`——**需要用你的 Unity 版本实测确认**
  （Frame Debugger / 该版本的 UI-Default.shader）。
- 逐层在 8bit RT 上混合会**每一步量化**；PhotoCraft 全程 f32、只在最后取整。
- 自定义 UI shader（预乘字体图集、粒子、第三方 UI 包）常用 `Blend One OneMinusSrcAlpha`。

---

## 3. 差异矩阵（PhotoCraft 现状 vs Unity）

| 场景 | Unity Gamma 空间 | Unity Linear 空间 |
|---|---|---|
| 不透明背景 + Normal + 直通 alpha | **已一致**（同一算式） | **不一致**（线性光 vs 编码值） |
| 半透明层叠加（如两层 50% 黑压白） | 一致：128 | Unity ≈188，PhotoCraft = 128 |
| 非 Normal 模式 | 已一致（27 种模式都有 oracle） | `B()` 也要在线性光里求值，未实现 |
| 合成到透明 RT / 目标 alpha | 不一致：Unity `As²`，PhotoCraft 走 W3C `(1−αb)αs·Cs` 项 + 标准 αo | 同上 |
| 预乘 shader（`One OneMinusSrcAlpha`） | 不一致（公式不同） | 不一致 |
| 多层堆叠的 8bit 逐层量化 | 有 ±1–2/255 漂移 | 同 |
| 双线性采样造成的透明边缘光晕 | PhotoCraft 不存在（不重采样）；Unity 取决于"Alpha Is Transparency"/预乘 | 同 |

即：**如果目标是 Unity 的 Gamma 色彩空间，PhotoCraft 现在就是对齐的**，只剩上表后四行这些边角；
"Unity UI 伽马合成问题"若指 Linear 工程下与设计稿不符，则需要下面方案 A。

---

## 4. 方案与工作量

### A. 会话级"混合空间"（推荐 MVP，2–3 人日）

把"当前图层 gamma"从 `text_gamma(layer)` 提升为渲染上下文里的一个**混合空间**值。

1. **compose**（~0.5–1 天）：新增 `BlendSpace { Document, Gamma(f32), Linear }`；
   在 `Ctx::for_doc`（`lib.rs:464`）里快照，替换 `blend_into_g`/`composite_atop`/`effects.rs:1835`
   的 gamma 入参；`composite_gamma` 的 `== 1.0` 哨兵改成显式枚举分支；encode/decode 改用
   `adjust::Transfer`（`adjust.rs:26-60`）以照顾灰度 dot-gain 与 F32 空操作；Lab 文档的
   `LAB_MIX` 优先级要明确。
2. **gpu**（~0.5 天）：`plan.rs` 四处改成"本 pass 的混合空间"，WGSL 的 `text_enc/text_dec` 接受
   传递曲线（或对 F32/灰度走同一 `Transfer` 表）；`F_TEXT_GAMMA` → 语义更一般的 flag。
3. **engine**（~0.5 天）：`ColorSettings` 加字段（`color_cmds.rs:106`）、`edit.colorSettings`
   参数与校验（`:829-896`、参数说明 `:977`）、`psblend` setter 与 prefs 应用（`prefs.rs:1342/
   1478/1508`）。
4. **ui-egui**（~0.5 天）：Color Settings 对话框由 `filter_dialog` 从参数串**自动生成**
   （`filter_dialog.rs:295-320`），无需新手写控件；但必须让设置进入画布缓存键/触发重合成
   （`canvas.rs:42-46`、`:737-751`、`display_color.rs:415`）；可再加 View 菜单里的预览开关
   （UI_COMMANDS 属纯视图状态）。
5. **验证**（~0.5 天）：`gpu/tests/parity.rs` 照 `:1337` 扩成文档级；compose 单测；
   engine 测试必须放独立 test binary（进程级设置的并发约束，见 `psblend.rs:173-179`）；
   `cargo xtask layers|wasm|parity|perf --quick`，动了数字再跑 scorecard。

### B. 每文档选项（+1–2 天）

额外要动 `photocraft-doc` 字段 → `format/src/manifest.rs` + `convert.rs`（编译期强制）→ `io`/PSD
（PSD 无对应字段，往返会丢，必须在文档里写清）→ 每个 render 调用点传参 → GPU plan/缓存键。
收益是"同一份文件在不同机器上表现一致"，代价明显更大。

### C. 精确复刻 Unity（+3–5 天，建议只做导出）

预乘累积 + `As²` 目标 alpha + 逐层 8bit 量化 + RT 的 sRGB 语义。这些**不能**塞进编辑模型
（编码值混合的文档语义是对 PSD/Photoshop 兼容性负责的），应做成"导出 ▸ Unity UI RGBA"通路：
一次 flatten，按 Unity 的算子写 RGBA8，可选做 RGB 膨胀（对齐"Alpha Is Transparency"）。
若还要保真采样行为，可在导出时提供"预乘 PNG"。

---

## 5. 风险清单

1. **哨兵冲突**：`composite_gamma` 的 `gamma == 1.0` 现在是"关闭"，而公式极限是线性光。必须显式化。
2. **传递曲线**：`psblend::text_encode` 硬编码 sRGB 曲线；灰度（dot-gain 1.732）、F32（已是线性）、
   CMYK 都不能照搬。F32 上必须是空操作。
3. **Lab/HDR 交互**：`LAB_MIX` 与新的混合空间谁优先，要写进文档并测。
4. **混合模式函数**：非 Normal 的 27 种模式在线性光模式下应在线性光里求值（Photoshop gamma 1.00
   的行为），可分离/不可分离（Hue/Sat/Color/Luminosity）都要覆盖；Dissolve、knockout、
   `advanced.rs` 的预乘中间量、pass-through 组都要过一遍。
5. **性能**：失去 Normal 快路径，每像素每层多两次 `pow`（sRGB 曲线还有分支）——24MP×10 层量级。
   用 `cargo xtask perf` 量，必要时给 `Transfer::Gamma(1.0)` 走 `x^2.2` 快路。
6. **缓存失效**：画布纹理键（`canvas.rs:737-751`）、代理/缩略图（按 revision）、GPU 累积 tile 都要
   在设置变化时失效，否则切设置画面不动（现有 `blendTextGamma` 大概也有这个隐患，值得顺手验证）。
7. **进程级 atomic + rayon**：现在是每层读一次 atomic，渲染中途被改会撕裂单次合成
   （`lib.rs:184-189` 的 #1112 就是同类问题）。放进 `Ctx` 快照可以顺带修掉。
8. **其它入口**：CLI/automation/web 也要拿到这个设置（prefs 已覆盖），否则 headless 渲染与 GUI 不一致；
   新参数要过 `panic_hunt`。
9. **验证手段**：corpus 里没有"Photoshop 打开 Blend RGB Using Gamma"的 oracle；要真正确认对齐，
   最好造一个小 Unity 工程（或等价参考：sRGB RT 上做 src-alpha over 再编码）逐层比对，
   必要时用 RenderDoc 抓帧确认混合空间。

---

## 6. 结论

- "改不同图层的 alpha 混合逻辑"在 PhotoCraft 里**是局部、低风险**的：图层→背景只有 `blend_into_g`
  一个漏斗，"在任意编码空间混合"的数学与 GPU 实现都已存在且有 parity 测试。
- 对齐目标决定要不要动代码：**Unity Gamma = 现状已对齐**；**Unity Linear = 需要方案 A**。
- 精确复刻 Unity 的 RT/预乘/量化语义不建议动编辑模型，放导出通路。
- 前置阻塞项：确认目标工程的 Color Space 与该版本 UI/Default 的混合状态（是否分离 alpha），
  以及一份可反复比对的 Unity 参考帧。

## 参考

- Unity 手册：Color Space（Gamma/Linear）与 sRGB 渲染目标上的混合语义。
- UI-Default.shader 源码（MIT，内置 shader 镜像）：
  https://raw.githubusercontent.com/TwoTailsGames/Unity-Built-in-Shaders/master/DefaultResourcesExtra/UI/UI-Default.shader
