# Player attributes CA/PA values (UABEA)

- **Patch folder:** `player-attributes-ca-pa`
- **Bundle:** `ui-tiles_assets_all`
- **Dump format:** UABEA Next JSON（`references.version = 2`；元素内数组字段均需 `{"Array": [...]}` 包装；`rid = -2` 占位条目须带 `"data": {}`）
- **Stock baseline:** `win/orig/` — 2026-09-13 15:45 构建的全量编译资产（47 元素 / 63 references，含完整 CTRL 布局；`panels/PlayerAttributesLargeBlock.uxml` 编译产物）
- **Working imports:** `win/*.json`（54 元素 / 70 references）
- **mac:** 未制作（个人 Windows 用途；如需发布请按 `UABEA-Notes.md` 平台规则从 mac 基线重做，IDs/CAB 不同）

## ⚠️ 应用方式（重要变更：Builder 0.7.0 不应用 _uabea 补丁）

2026-09-13 实测确认：**FM Skin Builder 0.7.0 不会自动应用 `_uabea/` 下的 patch.json**（后端无相关代码；同为 ui-tiles 的 `match-dugout_tile` 补丁在自构建里也从未生效）。README 里"某些改动必须手工在 UABEA 做"即指此事。

**实际可用流程（已验证）**：`F:\tools\fm26-skin-tool`（工程化的补丁工具，`python apply.py`，幂等）：

1. **每次 Builder Build + Apply 之后**运行一次——Builder 会用未打补丁的编译结果覆盖 `ui-tiles_assets_all.bundle`；
2. 工具自动：读游戏 bundle → 定位锚点（信息列容器 `-1201313256`，ID 失效时回退结构搜索）→ 注入 6 元素/6 引用 → 自检（全部对象可读）→ 带时间戳备份 → 原位部署 → 部署后复验；
3. 已打补丁时直接跳过（幂等）；`--restore` 可回滚备份。

`panels/PlayerAttributesLargeBlock.uxml` **保持存在**（已恢复）——Builder 需要它产出完整 CTRL 布局；它与本补丁不再冲突（Builder 根本不处理 `_uabea`）。

本目录的 `patch.json` / 转储保留作文档与手工 UABEA 备选路径：UABEA 打开 `ui-tiles_assets_all`，导入 `win/PlayerAttributesLargeBlock-CAB-…-7199728689344018492.json`。

> UnityPy 直写的技术注记：writer 对空类型托管引用（`rid=-2`、`cls=""`）缺少与 reader 对称的跳过逻辑，脚本内已猴子补丁修复；`rid=-2` 条目写回时需补 `"data"` 键。

## Asset

| Filename | Path ID | Role |
| -------- | ------- | ---- |
| `PlayerAttributesLargeBlock-CAB-651892cf441cf6c29e5bd20791e24f29-7199728689344018492.json` | `7199728689344018492` | 球员属性大块（完整布局根资产） |

## Target change（v3，与 16:01 编译结构对齐）

在信息列行容器（`m_Id = -1201313256`，`column-direction-normal`）下追加**两个同级条目列**（匹配该编译"平铺列"习惯，克隆源 `m_Id = -1025722924`）：CA 列与 PA 列，各含标签 + 数值两个 SIText：

| m_Id | m_ParentId | 类型 | m_Classes | m_Text | rid |
| ---- | ---------- | ---- | --------- | ------ | -- |
| `719000301` | `-1201313256` | VisualElement | 同克隆源列（`column-direction-normal, margin-bottom-global-gap-large, align-items-start`） | | 动态分配 |
| `719000302` | `719000301` | SIText | `global-text-secondary, body-small-12px-regular, margin-bottom-global-gap-small` | `"CA"` | 动态 |
| `719000303` | `719000301` | SIText | `global-text-primary, body-regular-14px-regular` | | 动态 |
| `719000304` | `-1201313256` | VisualElement | 同上 | | 动态 |
| `719000305` | `719000304` | SIText | 同标签 | `"PA"` | 动态 |
| `719000306` | `719000304` | SIText | 同数值 | | 动态 |

数值绑定（`m_kind = 1`，`m_direct` 用 **Path + Nullable** 包装）：`Player.PlayerCurrentAbility` / `Player.PlayerPotentialAbility`——注册表裸数值属性（`F:\fm-research\FINDINGS.md` G1/G2，大小写不敏感）。标签为静态文本（`text` + `text_UxmlAttributeFlags = 1`，先例：`PersonPersonalBlock_4x6`）。rid 由脚本按当前资产实际占用动态分配。

## 历史教训（四次迭代）

1. **v0（panels/ UXML，14:51）**：静态 `text` 属性触发 Builder 编译器缺陷→丢元素、损 bundle、闪退；`backups/originals` 被覆盖，只能 Steam 校验恢复。
2. **v1（UABEA 补丁，15:45）**：未生效——dump 格式 bug（元素内数组未包 `Array`、`rid=-2` 缺 `data`）+ panels 编译覆盖同资产。
3. **v2（UABEA 补丁，16:01）**：格式已修仍未生效——根因实锤：**Builder 0.7.0 根本不应用 `_uabea` 补丁**（dugout 补丁同样未生效 + 后端无代码）。
4. **v3（当前）**：UnityPy 直写游戏 bundle（含 writer 猴子补丁），部署验证通过。

## Import checklist

1. 运行 `F:\fm-research\patch_game_bundle.py`（或 Builder 重建后重跑）；
2. 冒烟测试：球员属性界面，信息列身高/声望/个性下方出现 `CA`/`PA` 条目；
3. 数值为空 → 改绑 `CurrentAbilityScore` / `PotentialAbilityScore`（脚本内两处 `path=`）。
