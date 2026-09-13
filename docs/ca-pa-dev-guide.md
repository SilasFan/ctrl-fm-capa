# 在 FM26 皮肤中显示 CA/PA 数值 — 开发者指南

Displaying CA/PA Values in FM26 Skins — A Developer's Guide

> 适用版本 / Verified against: **FM 26.3.2.2329565**（Windows, IL2CPP, Unity 6000.0.52）
> 路径与 PropertyID 随游戏版本变化，大版本更新后请按第 8 节方法重新验证。
> Property IDs and registry contents change between game versions; re-verify per the methods in §8 after major updates.

---

# 中文版

面向 FM26 皮肤开发者，说明如何把球员的**当前能力（CA）与潜力（PA）裸数值**显示在皮肤 UI 中。结论来自对 26.3.2 的 IL2CPP 逆向与实测（已在 CTRL 26 皮肤上落地验证）。

## 1. 一句话结论

CA/PA 数值**可以绑定、可以显示**。核心路径：

```
Player.PlayerCurrentAbility      # CA 裸数值（PropertyID 0x50434142 = "PCAB"）
Player.PlayerPotentialAbility    # PA 裸数值（PropertyID 0x50504142 = "PPAB"）
```

游戏自己的 `TacticTeamSelectionTableFactory` 调试列（`hidden-current-ability` / `hidden-potential-ability`）就在直接绑定它们排序和显示——这是可行性的一手证据。

## 2. 原理：为什么可以用

FM26 皮肤是纯声明式的，UI 通过 `SI.Bindable` 数据绑定框架被动接收引擎推送的值：

1. **注册表层**：`SI.Bindable.Reference.Core.PropertyRegistration.Initialise` 在启动时注册全部 **7805 个属性**（名字 → 4 字节 `PropertyID`）。绑定路径走白名单哈希查找，**未注册的名字一律无效**。
2. **大小写不敏感**：哈希前统一转小写，`binding.player.currentability` 与 `binding.Player.CurrentAbility` 等价。
3. **kind 含义**：`binding="kind=1; path=X"` 中 `Kind` 枚举为 `Value=1 / Reference=2`——数值显示用 `kind=1`。
4. **属性值类型由原生侧运行时决定**，静态文件查不到；数值性靠游戏自身用法判定。`PlayerCurrentAbility`/`PlayerPotentialAbility` 为裸数值（区别于 `tactic.PlayerContextTacticAbility` 那种带 `.MaxValue` 的区间对象、以及受球探知识影响的 `*StarRange` 星级族）。

## 3. 可用路径速查

| 路径 | 类型 | 说明 |
| --- | --- | --- |
| `Player.PlayerCurrentAbility` | 裸数值 | **首选**。上下文须能解析 `Player`（见 §4） |
| `Player.PlayerPotentialAbility` | 裸数值 | **首选** |
| `nonplayer.NonPlayerCurrentAbility` / `NonPlayerPotentialAbility` | 裸数值 | 职员版本 |
| `Player.PerceivedPotentialAbility` | 数值 | 游戏原生 UI 已用 |
| `CurrentAbilityScore` / `PotentialAbilityScore` | 数值（推断） | 已注册未使用，可作备选 |
| `Player.CurrentAbilityStarRange` / `PotentialAbilityStarRange`（含 `.MaxValue`） | 星级区间 | **受球探知识影响**，非裸数值 |
| `focusitem.PlayerCurrentAbility(+IncrementEnabled/DecrementEnabled)` | 数值（可编辑） | 招募焦点编辑面板用法 |

## 4. 前置检查：绑定上下文

路径首段是**上下文名**，不是随便写的。把值绑定加进一个面板前，先确认该面板现有绑定里出现过 `Player.` 前缀（例如 CTRL 的属性面板里有 `binding-mappings="report=Player.PlayerReportForContextTeam"`），即可证明上下文可用。球员报告/属性类面板通常满足。

## 5. 三种实现方式（按易到难）

### 方式 A：表格列（最简单，纯文件，Builder 原生支持）

在皮肤的 `table-views/*.json` 里加列即可，无需 UABEA：

```json
{ "columnID": "hidden-current-ability",  "widthOverride": 60, "minWidthOverride": 60 },
{ "columnID": "hidden-potential-ability", "widthOverride": 60, "minWidthOverride": 60 }
```

这两个列 ID 由游戏内置 tablefactory 提供并直接绑裸数值。注意区分：`current-ability` 在多数工厂里绑的是**星级区间**，裸数值列是 `hidden-*` 前缀。列 ID 全集（1851 个）见 §7 资料库。

### 方式 B：UXML 覆盖（panels 文件）

在 `panels/<面板名>.uxml` 里给 `si:SIText` 加值绑定：

```xml
<si:SIText class="global-text-primary body-regular-14px-regular"
           text-binding="kind=1; path=Player.PlayerCurrentAbility" />
```

**警告**：若需要静态标签文本（`text="CA"`），当前 FM Skin Builder（0.7.0）的 UXML 编译器存在缺陷——静态 `text` 属性会触发元素丢弃甚至损坏 bundle（实测闪退 `Position out of bounds!`）。标签建议走翻译绑定或已有的 visual-script 路径，值绑定不受影响。

### 方式 C：序列化注入（UABEA / UnityPy，最可控）

直接改 bundle 内面板资产的序列化树，适合 Builder 管不到的场景（例如 Builder 不应用 `_uabea` 补丁时的替代通道）。在 `m_VisualElementAssets` 增加元素、在 `references.RefIds` 增加对应 `SIText/UxmlSerializedData`：

```json
// references.RefIds 新条目（关键部分）
{
  "rid": 1065,
  "type": { "class": "SIText/UxmlSerializedData", "ns": "SI.Bindable", "asm": "SI.Bindable" },
  "data": {
    "uxmlAssetId": <元素 m_Id>,
    "name": "",
    "TextBinding": {
      "m_kind": 1,
      "m_direct": { "Path": { "m_path": "Player.PlayerCurrentAbility" }, "Nullable": 0 },
      "m_visualFunction": { "m_isAssigned": 0, "...": "全空" }
    },
    "...": "其余字段从同资产同类条目克隆"
  }
}
```

元素条目要点：

- `m_ParentId` 指向挂载容器；`m_OrderInDocument` 接在现有序号后（首子垫底渲染则复用首个子节点的 ord，靠数组位置定先后）；
- `m_Classes` 复用现成样式类（如 `global-text-primary body-regular-14px-regular`）；
- `m_SerializedData.rid` 与 RefIds 条目互指，`uxmlAssetId` 必须等于元素 `m_Id`；
- 新 `rid` 取该资产现有最大 rid +1；`m_Id` 取未占用值；
- 静态标签：元素 `m_Text = "CA"` + 引用数据 `text = "CA"`、`text_UxmlAttributeFlags = 1`（游戏原生先例：`PersonPersonalBlock_4x6` 的 `"当前能力"`）。

## 6. 序列化格式硬规则（踩坑换来的）

1. **`m_direct` 必须 `Path + Nullable` 包装**（Windows/mac 皆是），禁止旧扁平 `"m_direct": {"m_path": …}`；
2. **UABEA 转储的数组字段要 `{"Array": [...]}` 包装**，UnityPy 的 typetree 是裸列表——两种形式不可混用；
3. **`rid = -2` 占位条目**：UABEA 形式带 `"data": {}`；UnityPy 写回时需有 `data` 键（可 `null`），且 UnityPy 的 writer 对空类型托管引用缺少 reader 的跳过逻辑（会 KeyError），需要打对称补丁；
4. 结构改动后校验 `references.RefIds` / `rid` / `uxmlAssetId` / `m_ParentId` 相互一致再导入。

## 7. 工具与资料

| 资源 | 说明 |
| --- | --- |
| [fm26-skin-tool](https://github.com/)（本地 `F:\tools\fm26-skin-tool`） | 构建后补丁框架：幂等 apply、自动备份、全对象自检、原位复验；自带 CA/PA 补丁实现 |
| `registry_parsed.json`（7805 条） | 属性注册表全量——绑定路径的理论上限清单 |
| `all_m_paths.txt`（11014 条） | 游戏 UI 实际使用的 `m_path` 全集 |
| `column_ids_union.txt`（1851 个） | 表格列 ID 全集（含每列绑定接线） |
| CTRL 仓库 `docs/数据绑定机制.md` | 绑定系统完整机制说明（中文） |

## 8. 版本更新后如何重新验证

1. Il2CppDumper 导出新版 `dump.cs`，锚点搜 `PlayerCurrentAbility`；
2. `PropertyRegistration.Initialise` 提取新注册表，确认路径仍存在（ID 会变）；
3. 用 `scan_paths.py` 类工具重扫 UI bundle 的 `m_path`，核对游戏自身用法；
4. 表格路线最稳（列 ID 数据驱动，随 tablefactory 资产更新）。

---

# English Version

A guide for FM26 skin developers on displaying raw **Current Ability (CA)** and **Potential Ability (PA)** values in skin UI. Findings come from reverse-engineering 26.3.2 (IL2CPP) and were validated in production on the CTRL 26 skin.

## 1. TL;DR

CA/PA values **are bindable and displayable**. The core paths:

```
Player.PlayerCurrentAbility      # raw CA (PropertyID 0x50434142 = "PCAB")
Player.PlayerPotentialAbility    # raw PA (PropertyID 0x50504142 = "PPAB")
```

The game's own `TacticTeamSelectionTableFactory` debug columns (`hidden-current-ability` / `hidden-potential-ability`) bind exactly these for sorting and display — first-hand proof of feasibility.

## 2. How the binding system works

FM26 skins are purely declarative: the UI receives values pushed by the engine through the `SI.Bindable` framework.

1. **Registry layer**: `SI.Bindable.Reference.Core.PropertyRegistration.Initialise` registers all **7805 properties** at startup (name → 4-byte `PropertyID`). Binding paths are resolved through a whitelist hash lookup — **unregistered names are always invalid**.
2. **Case-insensitive**: names are lowercased before hashing; `binding.player.currentability` equals `binding.Player.CurrentAbility`.
3. **`kind` semantics**: in `binding="kind=1; path=X"`, `Kind` is `Value=1 / Reference=2` — use `kind=1` to display values.
4. **Property value types are decided at runtime by the native side** and are not statically discoverable; numeric-ness is inferred from in-game usage. `PlayerCurrentAbility`/`PlayerPotentialAbility` are raw numbers (unlike `tactic.PlayerContextTacticAbility`, which is a range object with `.MaxValue`, or the scout-knowledge-gated `*StarRange` family).

## 3. Path cheat sheet

| Path | Type | Notes |
| --- | --- | --- |
| `Player.PlayerCurrentAbility` | raw number | **Primary**. Requires a resolvable `Player` context (see §4) |
| `Player.PlayerPotentialAbility` | raw number | **Primary** |
| `nonplayer.NonPlayerCurrentAbility` / `NonPlayerPotentialAbility` | raw number | staff variant |
| `Player.PerceivedPotentialAbility` | number | already used by the stock UI |
| `CurrentAbilityScore` / `PotentialAbilityScore` | number (inferred) | registered but unused; good fallbacks |
| `Player.CurrentAbilityStarRange` / `PotentialAbilityStarRange` (with `.MaxValue`) | star range | **scout-knowledge gated**, not raw |
| `focusitem.PlayerCurrentAbility(+IncrementEnabled/DecrementEnabled)` | editable number | recruitment-focus editor usage |

## 4. Pre-check: the binding context

The first segment of a path is a **context name**. Before adding a value binding to a panel, confirm that `Player.` already appears in that panel's existing bindings (e.g. CTRL's attributes panel contains `binding-mappings="report=Player.PlayerReportForContextTeam"`) — that proves the context resolves. Player-report/attributes panels generally qualify.

## 5. Three implementation routes (easiest first)

### Route A: table columns (simplest, plain files, native Builder support)

Add columns to your skin's `table-views/*.json` — no UABEA needed:

```json
{ "columnID": "hidden-current-ability",  "widthOverride": 60, "minWidthOverride": 60 },
{ "columnID": "hidden-potential-ability", "widthOverride": 60, "minWidthOverride": 60 }
```

These column IDs are provided by built-in tablefactories and bind the raw values directly. Careful: in most factories `current-ability` actually binds the **star range**; the raw-number columns carry the `hidden-*` prefix. Full column ID union (1851) in §7.

### Route B: UXML override (panels files)

Add a value binding to a `si:SIText` in `panels/<PanelName>.uxml`:

```xml
<si:SIText class="global-text-primary body-regular-14px-regular"
           text-binding="kind=1; path=Player.PlayerCurrentAbility" />
```

**Warning**: if you need static label text (`text="CA"`), the current FM Skin Builder (0.7.0) UXML compiler is buggy — the static `text` attribute triggers element drops and even corrupt bundles (observed crash: `Position out of bounds!`). Prefer translation bindings or existing visual-script paths for labels; value bindings are unaffected.

### Route C: serialized injection (UABEA / UnityPy, most control)

Edit the panel asset's serialized tree inside the bundle directly — useful wherever the Builder can't reach (e.g. as the replacement channel when it doesn't apply `_uabea` patches). Add an element in `m_VisualElementAssets` and a matching `SIText/UxmlSerializedData` entry in `references.RefIds`:

```json
// new references.RefIds entry (key parts)
{
  "rid": 1065,
  "type": { "class": "SIText/UxmlSerializedData", "ns": "SI.Bindable", "asm": "SI.Bindable" },
  "data": {
    "uxmlAssetId": <element m_Id>,
    "name": "",
    "TextBinding": {
      "m_kind": 1,
      "m_direct": { "Path": { "m_path": "Player.PlayerCurrentAbility" }, "Nullable": 0 },
      "m_visualFunction": { "m_isAssigned": 0, "...": "all empty" }
    },
    "...": "clone remaining fields from a sibling entry of the same type"
  }
}
```

Element entry essentials:

- `m_ParentId` points at the mount container; `m_OrderInDocument` follows existing numbering (to paint behind as the first child, reuse the current first child's ord — array position decides precedence);
- reuse existing USS classes in `m_Classes` (e.g. `global-text-primary body-regular-14px-regular`);
- `m_SerializedData.rid` and the RefIds entry reference each other; `uxmlAssetId` must equal the element's `m_Id`;
- pick a fresh `rid` (max existing +1) and an unused `m_Id`;
- static labels: element `m_Text = "CA"` plus ref data `text = "CA"`, `text_UxmlAttributeFlags = 1` (native precedent: `PersonPersonalBlock_4x6`'s `"当前能力"`).

## 6. Hard serialization rules (paid for in crashes)

1. **`m_direct` must use the `Path + Nullable` wrapper** (Windows and mac); the legacy flat `"m_direct": {"m_path": …}` is invalid;
2. **UABEA dumps wrap array fields as `{"Array": [...]}`; UnityPy typetrees use plain lists** — never mix the two forms;
3. **the `rid = -2` placeholder**: UABEA form carries `"data": {}`; UnityPy round-trips require the `data` key (null is fine), and UnityPy's writer lacks the reader's skip for null-type managed references (KeyError) — a symmetric monkey-patch is required;
4. after structural edits, verify `references.RefIds` / `rid` / `uxmlAssetId` / `m_ParentId` consistency before importing.

## 7. Tooling and data

| Resource | Description |
| --- | --- |
| fm26-skin-tool (local `F:\tools\fm26-skin-tool`) | post-build patch framework: idempotent apply, auto-backup, full-object verification, in-place re-verification; ships a working CA/PA patch |
| `registry_parsed.json` (7805 entries) | full property registry — the theoretical ceiling of bindable paths |
| `all_m_paths.txt` (11014) | every `m_path` actually used by the stock UI |
| `column_ids_union.txt` (1851) | full column-ID union (with per-column binding wiring) |
| CTRL repo `docs/数据绑定机制.md` | complete binding-system write-up (Chinese) |

## 8. Re-verifying after a game update

1. Il2CppDumper on the new build, anchor-search `PlayerCurrentAbility` in `dump.cs`;
2. extract the new registry from `PropertyRegistration.Initialise` and confirm the paths survive (IDs will change);
3. re-scan UI bundles' `m_path` (e.g. with a `scan_paths.py`-style tool) and check the game's own usage;
4. the table-column route is the most stable (column IDs are data-driven and ship with the tablefactory assets).
