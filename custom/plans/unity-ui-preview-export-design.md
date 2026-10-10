# 设计方案：Unity 运行结果的预览与导出

状态：设计（未编码）。前置评估见 `custom/plan/unity-ui-blend-alignment.md`。
目标（用户确认）：让 PhotoCraft 能**预览**和**导出** Unity UI 在运行时合成出来的画面，
从而在设计阶段就看到"Unity 里到底长什么样"。

---

## 0. 范围

**做**：一个可切换的「Unity 兼容合成」模式，作用在合成器上，同时服务于
(a) 画布预览、(b) 导出 RGBA（给 Unity 当 UI 贴图/RenderTexture 用）。默认关闭，关闭时行为与现在**逐位相同**。

**不做**：不改文档默认语义（保持 Photoshop/PSD 兼容），不写进 `.pcraft`，不模拟 Unity 的贴图采样/mip/字体图集，
不做真正的 GPU 端 8bit RT 仿真（作为可选保真维度，见 P3）。

---

## 1. 语义模型：Unity 兼容合成 = 4 个正交维度

| # | 维度 | 取值 | 影响 |
|---|---|---|---|
| 1 | 混合空间 `BlendSpace` | `Document`（现状）/ `Linear` | RGB 结果，**唯一影响预览正确性的维度** |
| 2 | Alpha 表示 `AlphaModel` | `Straight`（UI/Default）/ `Premultiplied`（`One OneMinusSrcAlpha`、预乘字体图集） | 存储与采样表现；对不透明终端的 over 结果数学等价 |
| 3 | 目标 alpha `DestAlpha` | `Standard` / `UnityUiDefault` | 透明 RT 的 α 通道（`As²`） |
| 4 | 量化 `Quantize` | `Float`（现状）/ `Rgba8PerLayer` | 8bit RT 逐层量化 |

**数学结论（设计据此收敛）**：

- RGB 差异**只**来自维度 1。直通 vs 预乘在"over"上等价（`Co = Cs·As + Cd(1−As)`），
  差异只出现在存储表示、透明 RT 的反预乘、以及双线性采样（光晕）上。
- α 通道差异来自维度 3：`UI/Default` 的 `Blend SrcAlpha OneMinusSrcAlpha` 同时作用于 alpha，
  得 `A_out = As² + Ad(1−As)`，而 PhotoCraft 用标准 `αo = αs + αb(1−αs)`。
  仅当合成到透明 RT 时可见（对画布预览无影响）。
  注：部分 Unity 版本改用 `Blend SrcAlpha OneMinusSrcAlpha, One OneMinusSrcAlpha`，
  该分支是否启用**以你工程的 Unity 版本实测为准**（Frame Debugger）。
- 维度 4 只造成 ±1–2/255 级别的漂移，属于"保真"而非"正确"。

⇒ 因此 **预览 = 维度 1（+可选 4）**；**导出 = 维度 1+2+3（+可选 4）**，
其中维度 3 只在"导出为透明 RGBA 供后续 RT 管线消费"时才有意义。

---

## 2. 行为规范（精确算式与基准值）

记 `L(·)` / `E(·)` 为**文档传递曲线**的解码/编码（8/16bit RGB = sRGB 曲线；灰度/双色调 = dot-gain 1.732；
32bit = 恒等）。`as = 图层 alpha × 不透明度`，`ab` = 背景 alpha。

**模式 A `Document`（默认，= 现状 = Unity Gamma 色彩空间）**

```
Co = ((1−as)·ab·Cb + (1−ab)·as·Cs + as·ab·B(Cb,Cs)) / αo       αo = as + ab(1−as)
```

**模式 B `UnityLinear`（= Unity Linear 色彩空间 + sRGB RT）**

```
Co = E( ((1−as)·ab·L(Cb) + (1−ab)·as·L(Cs) + as·ab·B_lin(L(Cb),L(Cs))) / αo )
```

即：**权重不变，B() 与 over 都改在解码后的线性值上做，最后重新编码**。
非 Normal 模式必须先解码再求 B（Photoshop 的 gamma 1.00 行为，也是硬件混合的行为）。

**模式 C `UnityLinearPremultiplied8`（导出用）**：在 B 之上追加
`αo = as² + αb(1−as)`、写 RGBA8 时乘入 alpha、每层合成后按 8bit 量化（`round(E(·)·255)/255`）。

**必须进测试的基准值**（护栏，防止后来者改坏）：

| 用例 | 模式 A | 模式 B |
|---|---|---|
| 50% 黑 over 白 | 128 | **188** |
| 两层 50% 黑 over 白 | 64 | **137** |
| 50% 白 over 黑 | 128 | 188 |
| 任意全不透明层 | X | X（两模式必须相等） |
| 32bit 文档（已是线性） | X | X（模式 B 退化为恒等，**不得二次转换**） |

---

## 3. 架构与落点（严格遵守分层）

```
L0 color     新增 BlendSpace/AlphaModel/BlendOptions + composite_with()（纯函数）
L3 compose   新增 RenderOptions（含曲线来源）+ render_with()/flatten_with()，Ctx 携带快照
L3 gpu       plan 携带同一 RenderOptions；累积 tile 的键含选项
L4 io        导出：flatten_to_surface_with() → 预乘/量化 → PNG
L5 engine    会话状态 + view.* / file.export.* 命令 + prefs 持久化
L6 ui-egui   View 菜单开关/选项对话框；画布缓存键接入；状态提示
```

### 3.1 L0 API 草案

```rust
// photocraft-color::blend
pub enum BlendSpace { Document, Linear }
pub enum AlphaModel { Straight, Premultiplied }
pub struct BlendOptions { space: BlendSpace, alpha: AlphaModel, quantize: Option<Quantize> }
pub struct Curve { pub decode: fn(f32)->f32, pub encode: fn(f32)->f32 }  // 或 Transfer 值类型

pub fn composite_with(curve: Curve, opts: BlendOptions, mode: BlendMode,
                      backdrop: [f32;4], source: [f32;4], opacity: f32) -> [f32;4];

// 现状保持为薄封装（oracle 与 145 处调用零改动）
pub fn composite(mode, b, s, opacity) -> [f32;4] {
    composite_with(Curve::ENCODED_IDENTITY, BlendOptions::DOCUMENT, mode, b, s, opacity)
}
```

`compose::psblend::composite_gamma`（现有的文字 gamma 通路）并入同一实现：
`text_gamma = 1.45` 变成 `BlendSpace::Power(1.45)` 的一个取值，`gamma == 1.0` 的"关闭"哨兵
改为显式枚举分支（这是现在最脆的一处）。

### 3.2 L3 compose API 草案

```rust
pub struct RenderOptions { pub blend: BlendOptions, pub dest_alpha: DestAlpha }
impl RenderOptions {
    pub const DOCUMENT: Self;                    // 现状
    pub const UNITY_GAMMA: Self;                 // == DOCUMENT（用于自证与导出）
    pub const UNITY_LINEAR: Self;
    pub const UNITY_LINEAR_RT8: Self;             // 导出：+ 目标 alpha + 量化
}
pub fn render_with(doc, rect, opts) -> Buffer;
pub fn render_tiled_with(doc, rect, tile, opts) -> Buffer;   // 现有 render_tiled_with 加一参
pub fn flatten_with(doc, opts) -> Buffer;
pub fn flatten_to_surface_with(doc, fmt, bg, opts) -> Surface;
pub fn render_reduced_with(...)                              // 画布降采样路径同样要带
```

要点：
- `Ctx`（`crates/compose/src/lib.rs:440`）增加 `render: RenderOptions` 字段，
  在 `Ctx::for_doc` **快照一次**。这样既避免逐层读进程级 atomic 被并发改而撕裂单次合成
  （`lib.rs:184-189` 的 #1112 是同类问题），也让同一进程能并行渲染两种模式（预览 + 缩略图）。
- 曲线来源用现有 `adjust::Transfer::for_document(mode, depth)`（`adjust.rs:26-38`），
  不自己硬编码 sRGB：32bit → 恒等（模式 B 空操作）、灰度 → 1.732、CMYK → 走现有 `cmyk_space`。
- Lab 文档的 `LAB_MIX`（`psblend.rs:74-109`）与新混合空间的关系必须定死：
  **推荐 Lab 文档下模式 B 不生效并给出提示**（Unity 没有 Lab 概念），并写进文档与测试。

### 3.3 L3 gpu

- `plan.rs` 现有 `F_TEXT_GAMMA` + `p4.w`（`:372-376`、`:708/:894/:1237/:1250`）扩展为
  "本 pass 的混合空间"；`compose.wgsl:284-296` 的 `text_enc/text_dec` 改为按传递曲线参数化，
  `composite`/`composite_g` 收敛为一个带选项的实现。
- **累积 tile 缓存的键必须包含 `RenderOptions`**（现在只按文档/page 标识），否则切模式画面不动。
- GPU 与 CPU 必须共用同一个 `RenderOptions` 结构体与同一组 parity 测试，禁止各写一套。

### 3.4 L5 engine（一切皆命令）

**状态**

- 全局默认：`ColorSettings` 增加 `unityCompat: UnityOptions`（`color_cmds.rs:106`），随 prefs 持久化
  （`prefs.rs:1342/1478/1508` 同款位置）。
- 每文档覆盖：`ColorState` 增加 `unity: HashMap<DocId, UnityOptions>`，**对齐 View 菜单的兄弟项**
  （`proofs: HashMap<DocId, ProofView>`、`hdr: HashMap<DocId, HdrPreview>`，`color_cmds.rs:201-213`）。
  理由：Proof Colors / 32-bit Preview 都是"视图"语义且按文档；Unity 预览常需要 A/B 两个文档对比。
  *降级方案*：只做全局（省约 1 小时），代价是不能并存对比。

**命令**（id 一律取自 `crates/ui-egui/src/menu_catalog.rs`，先加目录项再加命令）

| id | 菜单 | 参数 | 说明 |
|---|---|---|---|
| `view.unityPreview` | View | `{on:bool, preset:"gamma"\|"linear"\|"linearRT8"}` | 勾选项，快速开关 |
| `view.unityPreviewOptions` | View | 与 `UnityOptions` 同构 | 由 `filter_dialog` 自动生成对话框（无需手写控件） |
| `file.export.unityUi` | File ▸ Export | `{path\|dir, scope:"canvas"\|"layers", premultiply:bool, quantize8:bool, dilateRGB:bool}` | 导出 |

`view.unityPreviewOptions` 与 `view.thirtyTwoBitPreviewOptions`（`menu_catalog.rs:643`）同构，可直接照搬。

### 3.5 L6 ui-egui（预览必须触发重合成）

- 两处画布入口都要把模式纳入缓存键：`ensure_texture`（`canvas.rs:914`，键为
  `preview_key ^ display_key`）与 `ensure_gpu`（`canvas.rs:1072`，键为 `cache.revision` + `preview_key`）。
  建议把模式项混入 `display_doc` 返回的 `preview_key`（`canvas.rs:763`），一处接入两条路径。
- **不要**把 Unity 预览做成 Proof/LUT：`CanvasDisplay`（`display_color.rs:395-426`）那套是
  **最终像素的逐点变换**，无法表达"不同的合成空间"。反例：模式 A/B 在 50% 黑 over 白 上分别得
  128/188，任何与内容无关的逐点映射都无法把这一对映射正确而同时保持全不透明区域不变。
- 预览开启时给可见提示（画布角标或状态栏文案），避免用户误以为文档被改了。
- 预览**不代表**文档被修改：不产生历史、不置 dirty（照 `adjust_preview` 的做法）。

---

## 4. 关键设计决策汇总

| # | 决策 | 结论 | 理由 |
|---|---|---|---|
| D1 | 预览实现位置 | 重新合成，不走 proof/LUT | 见 §3.5 反例 |
| D2 | 设置归属 | 全局默认 + 每文档覆盖（存 ColorState，不进文档） | 与 View 兄弟项一致；PSD 无法承载 |
| D3 | 选项传递 | per-render 快照进 `Ctx`，不用进程级 atomic | 防撕裂、允许两种模式并行渲染 |
| D4 | 默认值 | 全部默认关闭 = 现状 | 零回归、零性能代价 |
| D5 | 曲线来源 | `adjust::Transfer` | 32bit/灰度/CMYK 才正确 |
| D6 | 预览与导出 | 同一实现，导出多两个维度 | 保证"预览所见 = 导出所得" |
| D7 | 预览后的显示链 | **保留** doc→monitor ICC（可关） | Unity 不做色彩管理；保留则在不同显示器上仍一致，关闭则模拟"无管理"的观感 |
| D8 | 模式 A 与"Unity Gamma" | 同一实现的两个名字（`RenderOptions::UNITY_GAMMA == DOCUMENT`） | 让"现状已对齐"成为可回归的断言，而不是口头结论 |

---

## 5. 分阶段计划（每阶段独立可交付）

| 阶段 | 内容 | 产出 | 预估 |
|---|---|---|---|
| **P0** | 语义固化：§2 的算式与基准值写成测试 + 文档；确认 Unity 版本 UI/Default 的 alpha 混合状态 | `docs/` 语义表、单测 | 0.5 天 |
| **P1** | 维度 1 打通：L0 `composite_with` + L3 `RenderOptions`/`Ctx` 快照 + L3 gpu + L5 状态/命令 + L6 缓存键与提示 | 可切换的画布预览 | 2 天 |
| **P2** | 导出：`file.export.unityUi`（维度 1+2，可选 4）+ 命名/批量 + CLI/automation 参数 | 导出文件与参考帧比对 | 1.5 天 |
| **P3** | 保真：维度 3（透明 RT 的 α）、8bit 逐层量化、Lab/灰度/CMYK/F32 边角、性能优化 | 边角测试全绿 | 2 天 |
| **P4** | 文档与流程：`docs/architecture.md`、`docs/parity.md`、scorecard、devlog；如需，给 corpus 加一张 Photoshop "Blend RGB Using Gamma" 的 oracle | 文档齐全 | 0.5 天 |

合计约 6.5 天；若只要"预览（Linear）+ 仅 Normal 模式"，P1 的子集约 1.5 天即可见到效果。

---

## 6. 测试与验收矩阵

| 层 | 测试 | 验收 |
|---|---|---|
| L0 | `composite_with` 各维度组合的关键值（§2 表） | 与手算值 1e-6 内一致 |
| L0 | 默认选项与旧 `composite` 逐位相同 | 现有 `blend.rs` 测试零改动通过 |
| L3 CPU | 27 种混合模式 × {A,B} × {U8,U16,F32} × {RGB,Gray,Lab} | 有限值、无 NaN；全不透明区域两模式相等 |
| L3 GPU | 扩展 `gpu/tests/parity.rs:1337`（现只覆盖文字 gamma）为文档级 `blend_space_*` | 预乘差 ≤ 2/255 |
| L5 端到端 | headless：同一文档在 A/B 三种模式下 flatten | `flatten(DOCUMENT) == flatten(UNITY_GAMMA)`；B 的差值与 §2 基准一致 |
| L5 排除回归 | `cargo xtask test-corpus` / `cargo xtask perf --quick` | oracle 与性能都不退（默认模式下零变化） |
| 导出 | RGBA8 导出与 Unity 参考帧逐像素比对 | 报告最大/均值差；预乘导出断言 `RGB ≤ A` |
| 健壮性 | 新命令过 `panic_hunt`（非法参数/空文档/超大尺寸 → `Err`） | 无 panic |

**参考帧是这套设计的关键前置物**：corpus 里没有"Photoshop 打开 Blend RGB Using Gamma"的 oracle，
且 Unity 的最终结果受 RT 格式/版本影响。建议固定一个小 Unity 工程（Unity 版本、Color Space、
RT 格式、UI/Default 是否分离 alpha 全部记录），对同一份 PSD/PNG 出图，作为回归基准。

---

## 7. 性能预算

- 模式 B 失去 Normal 快路径：每像素每层多 2 次 `pow`（sRGB 曲线还带分支）。
  24MP × 10 层量级需实测，纳入 `cargo xtask perf` 的场景之一。
- 可选优化：`Curve::Power(2.2)` 快路（比 sRGB 曲线便宜）、只在开启模式时走慢路。
- **默认关闭 ⇒ 主路径零成本**，这是本设计对性能预算的承诺，必须写进 scorecard 行。

## 8. 风险、回滚、兼容

| 风险 | 缓解 |
|---|---|
| 哨兵 `gamma == 1.0` 语义冲突（线性光 vs 关闭） | 改成显式枚举，P1 一并处理 |
| 32bit 文档被二次线性化 | 曲线统一走 `Transfer`，基准表里加 F32 恒等断言 |
| Lab/HDR 与 `LAB_MIX` 优先级 | P1 定死策略并测试（推荐 Lab 下不生效 + 提示） |
| 混合模式函数在线性值上求值引入新分支（Hue/Sat/Color/Luminosity 尤其） | P3 全覆盖，parity 测试兜底 |
| 缓存未失效导致切模式画面不动 | P1 就把模式纳入两处画布键，并有"切换后像素必须变化"的测试 |
| Unity 版本差异（是否分离 alpha） | 做成模式参数，P0 实测确认默认值 |
| 用户误以为文档被改 | 预览不置 dirty + 可见提示，测试断言 revision 不变 |

**回滚**：整个能力由一个默认关闭的选项驱动，且不写进文档格式 ⇒ 出问题只需忽略该开关；
不需要数据迁移，不影响 PSD/`.pcraft` 往返。

---

## 9. 明确不做（写入文档以防误解）

- 不把 Unity 语义设为编辑默认（Photoshop 兼容性由 `Document` 模式负责）。
- 不模拟 Unity 的贴图导入设置、mipmap、图集打包、字体 SDF 行为。
- 不做"读取 Unity 工程自动同步设置"（跨工具耦合，另一个项目）。

---

## 10. 待确认

1. **（你第 2 点没填）Unity 工程 Color Space 是 Gamma 还是 Linear？**
   它决定默认预设：Gamma ⇒ 默认 = 模式 A，P1 只剩"自证 + 导出"；Linear ⇒ 默认 = 模式 B，P1 全量做。
2. 你工程的 `UI/Default` 是否分离 alpha（决定维度 3 的默认值）。
3. 导出粒度：整张画布 flatten，还是按图层各出一张（多张 UI 元素 PNG）？后者要定命名与批量规则。
4. 是否需要"把 Unity 截图导回 PhotoCraft 做差分"的工作流（有参考帧后很便宜，且能自动守护本功能）。
