---
name: stage-to-story-chain
description: |
  VLOG / 短视频 / MV / 宣传片 / 电影 / 短剧 / 漫剧 / 故事短片的“先概要、先资产、再站位、再分镜”总控链路 Skill。适用于含 >=1 个角色、需要从剧本或制作计划提取故事板概要、场景调度表、人物 / 场景 / 关键道具 / 可选光影参考图，再生成舞台定位图、黑白 2x4 分镜和 Panel 视频提示词的前期流程。严格串行：双 MD -> 视觉资产 -> 舞台定位图 -> 黑白 2x4 分镜 -> video_prompt_list.md -> question 确认后视频。舞台定位图只允许使用当前段落场景参考图作为 reference image，不得传入人物、道具、光影或黑白分镜图。
trigger-words: [定位分镜链, 舞台定位到分镜, 先资产再站位, 先参考资产再舞台定位图, 舞台定位图转分镜, 先站位再分镜, 先定站位再画分镜, 先角色站位图再故事板, 故事板概要模板源, 人物三庭五眼, 三庭五眼结构, 人物面部结构, 人物参考图信息提取, 多人场景定位, 多人分镜位置漂移, 先blocking再storyboard, blocking to storyboard, stage-position-map zero-to-story]
allowed-tools: [hub_read, hub_write, hub_edit, hub_analyse_media, hub_canvas_get_node, hub_canvas_search_nodes, hub_canvas_write_text_node, hub_canvas_write_table_node, hub_canvas_write_media_node, question]
author: yzsp-ai
repository: https://github.com/yzsp-ai
---


# 定位分镜链 — Story Brief -> Blocking Sheet -> Visual Assets -> Stage Map -> Black-and-white 2x4 Storyboard

本 Skill 是影视前期总控链路，不直接生图 / 生视频；它规定阶段顺序、输入真值、参考图边界、画布交付物和 QC。具体图片 / 视频仍由主流程按模型路由派单。

## 0. 链路总览

```text
STEP A：前置真值与资产
1. 生成 / 更新：故事板概要 — <集数 + 段落编号 + 场景名>
2. 生成 / 更新：场景调度表 — <集数 + 段落编号 + 场景名>
3. 基于故事板概要生成允许范围内视觉资产
4. 基于双 MD + 当前段落场景参考图生成舞台定位图

STEP B：分镜与视频前期
5. 基于双 MD + 全部视觉资产 + 本段舞台定位图生成唯一黑白 2x4 分镜图
6. 整理 / 更新 video_prompt_list.md
7. 通过 question 弹窗确认后进入视频生成
```

任何故事板 / 黑白宫格 / Panel 视频提示词任务，必须先完成 STEP A。不得跳过双 MD、视觉资产或本段舞台定位图直接进入 STEP B。

### 0.1 链路异常自愈与状态恢复

任何阶段因工具调用超时、网络波动或模型中断导致链路失联时，总控 Agent 在重启任务时必须优先调用 `hub_read` 扫描当前段落 `<集数 + 段落编号 + 场景名>` 下的所有已完成节点、QC 记录与最近一次 `question` 弹窗授权记录。Agent 必须自动定位到“上一个成功通过 QC 的环节”，恢复至中断前状态点，并只补做缺失或失败的下一动作；严禁因找不到短期对话上下文就从头跑起，避免重复生成污染 ACTIVE 资产池。

### 0.2 契约门禁与审计链

每个阶段必须执行 Entry Guard / Exit Guard：进入前校验前置节点、角色真值、空间表、参考图与授权状态；离开前校验本阶段产物数量、核心字段、QC 状态与 lineage 元数据。失败时返回结构化错误码，例如 `[E01: Missing Script/Assets]`、`[E05: Spatial Data Corruption]`，禁止继续写入下游。所有 MD、table、image、video 产物元数据必须嵌入审计链：`[Generated_By: <Tool_ID> | Source_Script_Version: <Hash> | Preceding_QC_Status: PASS | Timestamp: <ISO_8601>]`，用于后续精准回溯。

## 1. 全局硬规则

0. **源头剧本依赖**：链路启动前，总控 Agent 必须在工作区画布或上下文中死锁原始剧本节点 `script_source_node`。STEP A / STEP B 的所有中间产物、双 MD 与增量修改，其 `<集数 + 段落编号 + 场景名>` 必须与源头剧本节点建立刚性父子级引用追踪链，确保多轮重做时可回溯剧本真值。
1. **动作授权**：故事板概要、场景调度表、视觉资产、舞台定位图、黑白分镜、Panel 视频提示词、视频生成、修复、重做、追加、跳步恢复，都是独立制作动作。每次动作开始前必须调用 `question` 工具；一次弹窗只授权当前动作，动作完成即失效。唯一例外是 B7：视频前单次事务弹窗可同时完成增强选择、确认 `video_prompt_list.md` 基础内容并授权同轮视频生成。
2. **段落隔离**：所有产物按 `<集数 + 段落编号 + 场景名>` 隔离。禁止串用上一段文档、舞台定位图或参考图。
3. **双 MD 真值源**：图片 / 视频派单前必须重新读取当前段落两份 markdown 节点：`故事板概要` 与 `场景调度表`。禁止只凭对话历史、旧摘要、文件名、nodeId 或“已继承”派单。
4. **空间字段真值源**：`stage_position_xy` 与 `facing_target_xy` 必须保留 `(x, y)`；`screen_position` 只服务画面构图；`camera_facing` 只服务镜头朝向；自然语言方位不能替代坐标。
5. **资产范围固定**：只制作六类视觉参考资产：人物参考图、场景参考图、关键道具参考图、可选光影参考图、舞台定位图、黑白分镜图。服装图、妆造图、表情包、姿势库、材质拆解、气氛小样不得单独新增，必要信息合并进上述资产。
6. **参考图功能隔离**：同一张图不能同时作为人物参考和场景参考。参考图必须解析成真实画布 image 节点或实际 `image_path`；只写名称、nodeId、文件名或清单文字不算已加载。
7. **模型边界**：美术资产参考图使用 `image2 / 1k / 16:9 / high`；舞台定位图使用 `gpt-image-2`；黑白 2x4 分镜使用 `image2 / 4k / 16:9 / low`，每格保持 21:9 宽银幕构图。用户要求改模型或模型不可用时，必须通过 `question` 弹窗确认，不得静默替换。
8. **空间锁文本块不可改写**：`Segment Visual & Tone Lock`、`Segment Avoid`、`Scene Scheduling Sheet Truth Extract`、`Panel Character Spatial Matrix`、`Panel Spatial Lock` 是总控派发给 image agent 的前置语义约束，不是可压缩说明。总控最终 `task_description` 必须完整包裹这些字段块；若出现 `隐藏空间锁`、`空间继承`、`已继承`、`详见上文`、`同上`、`字段化空间锁已传入`、`完整字段已传入`、`按调度表执行` 等占位 / 概括短语，立即判定 FAIL。
9. **图像无文字 ≠ task_description 无字段**：`图内不得出现坐标 / 字段 / 文字` 只约束最终画面，不允许删掉 `task_description` 中的坐标字段。坐标字段必须留在总控派发的数据包里作为幕后空间锁。
10. **重做清理机制**：若任一阶段触发 QC FAIL、修改上游双 MD 或用户要求局部重做，总控 Agent 在重新生成前，必须物理将画布旧节点、旧参考图和旧 `image_path` 深度隔离并打上 `[DEPRECATED_OLD_VERSION]` 标签。下游抓取上下文及相对位置（如 `image_paths` 倒数第二位）时，必须过滤并拒收任何废弃资产，杜绝历史上下文数据中毒。
11. **输入源强校验**：所有从原始剧本或用户补充模板提取的物理坐标、真值字段或特征描述，在写入 MD 节点前必须先进行语义清洗与格式校验。严禁包含控制字符、非法转义符、HTML 残留标签、乱码编码片段，或任何导致坐标解析为 `NaN` 的格式冲突。若解析源存在逻辑冲突或数据格式异常，必须在 A1 / A2 生成阶段立即通过 `question` 弹窗校准，严禁携带脏数据进入 STEP B 派单环节。
12. **全局视觉真值库**：跨集 / 跨段落创作必须维护 `Master_Character_Identity.md` / Global Visual Truth Database。每个核心角色必须有唯一身份哈希 `Unique Identity Hash` 与 `Identity Token`；所有涉及该角色的故事板概要、人物参考图、黑白分镜和视频任务包必须携带同一身份令牌，禁止从当前段落临时重写角色外貌。

## 2. question 弹窗状态机

- 每个制作动作开始前弹窗，一次问全本动作所有选择，推荐项放第一位。
- 当前动作完成后停止在结果汇报处，并写明链路锚点，例如：`当前状态：STEP A4 舞台定位图已完成。链路已暂停，等待 question 弹窗授权进入 STEP B 黑白分镜。`
- 禁止在无新授权时反复弹窗同一步骤或自动进入下一步。
- 只有 `question` 工具真实失败时，才可退化为纯文本编号选项，并说明工具失败。

通用动作选项：

1. 只执行当前阶段的下一项制作动作（推荐）
2. 只生成 / 更新故事板概要与场景调度表
3. 只生成允许范围内视觉资产
4. 生成本段舞台定位图
5. 进入黑白分镜图阶段
6. 进入 Panel 视频提示词 / 视频生成阶段
7. 暂停链路

## 3. STEP A — 双 MD 与视觉资产

### A1. 故事板概要

每一集每一段必须有独立 markdown 节点：

```text
故事板概要 — <集数 + 段落编号 + 场景名>
```

制作前必须读取用户上传剧本、制作计划或影视化优化模板。缺少模板源时，用 `question` 弹窗让用户选择补充模板、从剧本即时提取或暂停链路。

故事板概要必须包含：用户原始需求、模板源摘录、已确认制作标准、本段剧情范围与结尾衔接、情绪目标、动作节拍、8 格动作概要、人物 / 场景 / 关键道具信息提取表、人物面部三庭五眼结构表、角色身份令牌表（`character_id`、`Unique Identity Hash`、`Identity Token`、对应 `Master_Character_Identity.md` 条目）、Medium Lock、Avoid、下一步任务清单。

人物面部三庭五眼结构表每个出场人物至少包含：人设 / 对应比例特征、面部骨相、三庭比例、五眼比例、眉弓、眼距、鼻梁、颧骨、下颌、脸型、真实皮肤细节与瑕疵、服装与阶级设定、电影级摄影参数基准。禁止只写“好看 / 清秀 / 英气 / 古风脸 / 高颜值”。

### A2. 场景调度表 / Blocking Sheet

每一集每一段必须有独立 markdown 节点：

```text
场景调度表 — <集数 + 段落编号 + 场景名>
```

场景调度表只记录会影响空间调度的角色名、状态、坐标、视线、相对距离和运动轨迹。提取场景空间结构时必须强制检索并物理透传前置连续段落的全局大环境上下文（Global Environment Context，如烛光影调基调、全局天气、时间变量），确保跨段落影调连续；人物面部三庭五眼、皮肤瑕疵、骨相、服装材质等生物资产细节保存在 A1，不在调度表重复粘贴。若二者冲突，以故事板概要为上游真值源，先修复场景调度表并记录冲突修复项。

场景调度表必须包含：故事板概要引用记录、冲突修复记录、Global Environment Context、场景空间结构、角色站位表、道具坐标表、Eyeline Map、Panel 动作关键帧列表、Panel Character Spatial Matrix、镜头组使用规则、视频提示词继承规则、人物参考图清单、场景参考图清单、关键道具参考图清单、可选光影参考图清单，并同步生成 / 存储独立基线文档 `stage_blocking_sheet.md`。该基线文档是后续舞台定位图、黑白分镜和视频提示词的空间真值，不得被单张视觉地图替代。

角色站位表字段：`角色`、`Identity Token`、`stage_position_xy`、`facing_target_xy`、`screen_position`、`camera_facing`、`eyeline`、`distance_relation`、`movement_path`、`state_action`、`previous_emotion_state`、`current_emotion_state`、`next_emotion_target`。情感字段必须形成 Character State Machine：后一 Panel 的表演只能从前序情绪逻辑过渡，禁止随机跳变。

场景调度表中的核心变量必须强制采用以下结构化真值输出，严禁使用模糊自然语言：`Panel Character Spatial Matrix` 必须使用 `[角色名: X, Y, Z 相对坐标]` 标定当前段落所有核心角色的物理空间站位；`Eyeline Map` 必须使用 `[发射源角色 -> 接收目标对象: 视线角度/朝向向量]` 标定每一格 Panel 的视线物理交汇点，作为下游 B6 的刚性校验基准。

### A3. 视觉资产

生成舞台定位图前，允许范围内视觉资产必须先完成并落画布。

- **人物参考图**：每个主要角色独立生成一张 16:9 多视角统筹画布，使用 `image2 / 1k / 16:9 / high`。画面包含正面精细特写、正侧面 / 四分之三侧面、全身或半身服装阶级展示；必须强制中性无表情（Neutral Expression）与扁平无方向无戏剧性强光照（Flat Lighting），严禁自带戏剧化情绪、环境高光或强阴影。task_description 必须携带 `[画面构图与排版]`、`[面部骨相与三庭五眼]`、`[真实皮肤细节与瑕疵]`、`[服装与阶级设定]`、`[电影级摄影参数基准]`。
- **场景参考图**：从故事板概要的场景信息提取表生成，包含时代、门第、场景功能、空间结构、光影气氛、本段动作关系。
- **关键道具参考图**：仅限本段明确操作、推动冲突或承担叙事信息的道具；普通陈设并入场景参考图。
- **可选光影参考图**：仅当用户或剧情明确需要光影锁定时生成。

若人物参考图退化为普通单张肖像、网感美女 / 帅哥图、缺少侧面 / 服装阶级展示，或未体现 A1 三庭五眼关键字段，判定 FAIL。

### A4. 舞台定位图

舞台定位图只允许把当前段落场景参考图作为真实 reference image。人物参考图、关键道具参考图、可选光影参考图、黑白分镜图不得传入舞台定位图；它们只服务后续黑白分镜或视频阶段。

舞台定位图派单前必须：

1. 重新读取本段最新故事板概要、场景调度表与 `stage_blocking_sheet.md`。
2. 只从当前段落场景调度表提取场景参考图信息，并解析为真实画布 image 节点或实际 `image_path`。
3. 将剧情范围、场景名、主要冲突、结尾衔接、角色坐标、道具坐标、`facing_target_xy`、朝向箭头、Eyeline Map、当前段落场景参考图路径写入 task_description。
4. 人物和道具只以 `stage_blocking_sheet.md` 的文字坐标、朝向和 Eyeline 写入 task_description，禁止传入人物 / 道具 / 光影参考图片。
5. 若场景参考图是透视室内图，必须要求模型降维映射为 2D 鸟瞰平面，只提取墙壁、门窗和家具相对几何边界。

舞台定位图必须包含：90 度垂直俯视图、x / y 坐标网格、角色矩形色块、每个角色色块旁的简写角色标签文本、道具圆形色块、紫白色短粗朝向箭头、坐标指向文字。生成 table_node 时必须执行空间像素防撞校验：严禁任何两个独立资产（角色或道具）共享完全相同的绝对坐标 `(x, y)`；若同一物理平面产生重叠，必须通过 Z-index 层级标记或最小空间距离隔离阈值处理，否则判定 FAIL。

舞台定位图必须由 `stage_blocking_sheet.md` 派生 / 更新，不得由视觉地图反推调度真值。完成时必须在画布同步产出双重交付物节点，严禁只留单种介质：1. 调用 `gpt-image-2` 生成视觉定位图节点（media_node）；2. 同步生成物理结构化坐标位置表节点（table_node），将角色与道具的几何坐标、朝向、Eyeline 和相对距离锁死在表格看板中，供下游精准对齐。

失败处理：场景参考图无法解析、参考图数量超过模型上限、错误加载人物 / 道具 / 光影 / 黑白分镜图、角色标签缺失或重叠不可读、同坐标重叠未做 Z-index / 距离隔离，或缺少 media_node / table_node 任一交付物，均判定 FAIL，回到对应阶段修复。

### A5. STEP A 退出条件

必须同时具备：当前段落故事板概要、当前段落场景调度表、独立 `stage_blocking_sheet.md`、全部允许范围内视觉资产、当前段落舞台定位图 media_node、同步坐标表 table_node、舞台定位图真实加载参考图记录、可供 STEP B 复制的故事板字段 / 空间字段 / 参考图清单。缺任一项，不进入 STEP B。

## 4. STEP B — 黑白 2x4 分镜

### B1. 输入继承与参考图映射

进入 STEP B 前必须重新读取：当前段落故事板概要、当前段落场景调度表、`stage_blocking_sheet.md`、全部允许范围内视觉资产节点 / 路径、当前段落舞台定位图 media_node / table_node。B1 不得只读取舞台定位图视觉节点；table_node 的结构化坐标表和 `stage_blocking_sheet.md` 文本是几何真值。进入黑白分镜前必须执行【物理真值一致性校验】：比对当前段落 `table_node` 的元数据版本号 / 生成时间戳 / 坐标字段内容，与 `stage_blocking_sheet.md` 文本中的核心空间真值是否完全一致。若二者版本或内容不一致（例如文本已修改但表格节点未重绘，或 table_node 仍指向旧坐标缓存），严禁调用 image agent，必须强制回退至 A4 阶段重绘空间真值表格，直至双重真值逻辑统一。

黑白分镜图使用 `@STAGE_MAP_VISUAL` 作为 Layout / 结构轻度参考锁，人物参考图作为角色轮廓参考，场景参考图作为环境参考，关键道具参考图作为道具参考，可选光影参考图作为光影参考。`@STAGE_MAP_VISUAL` 只作幕后二维坐标映射和空间核验；最终画面不得渲染网格、文字、箭头、坐标、色块或地图边框。

进入最终 task_description 前，必须建立刚性占位符命名空间，禁止再使用 `@参考图N`、`@资产图N`、`@分镜图` 等旧称：

```text
@STAGE_MAP_VISUAL = 当前段落舞台定位图 media_node，仅 Layout / 结构轻度参考锁
@IMAGE_ASSET_N = 彩色视觉资产图，按物理 image_paths 顺序编号，N 从 1 开始
@STORYBOARD_GRID = 黑白 2x4 分镜图，仅在视频阶段出现，固定为 video-stage image_paths 倒数第二位
```

黑白分镜阶段的 `Reference Mention Map` 与真实 `Reference image paths` 必须绝对对齐：`@STAGE_MAP_VISUAL` 对应 `image_paths[0]`；随后 `@IMAGE_ASSET_1..N` 依次对应 `image_paths[1..N]`。同时必须把舞台定位图 table_node 的坐标、朝向、Eyeline、相对距离和 `stage_blocking_sheet.md` 的场景几何文本冷数据写入 `[Scene Scheduling Sheet Truth Extract]` 与 `[Panel Spatial Lock]`，以 table_node / markdown 为真值覆盖视觉地图误差。禁止悬空指针、孤儿图片、编号跳号、重复编号或语义错配。

### B2. 黑白 2x4 分镜生成与物理参数强控

总控 Agent 必须调用专属生图工具生成单张全景 2x4、共 8 个面板的整体宫格黑白分镜大图。向工具派发任务包时，模型与物理参数必须死锁：`image2 / 4k / 16:9 / low`。提示词中必须强制死锁：每一个独立 Panel 子网格矩形格子的画面比例必须完美呈现电影级 21:9 宽荧幕视效（Cinema Cinemascope Scope 21:9 ratio per grid），严禁产生网格拉伸、压扁或特征断裂。生成提示词必须将 A2 场景调度表中的【Panel 动作关键帧文本】与【电影摄影参数】交叉语义拼接，作为每格核心画面主体真值，确保分镜动作与剧本角色行为绝对对齐。

- 单张横版 2x4，4 列 x 2 行 = 8 个面板；不是 8 张独立图片。
- 每格只表达一个动作关键帧。
- 角色可渲染为简笔线框人偶、火柴人或高对比度黑白速写 / 草稿风格人偶；重点是形体动作、站位关系与构图。
- 不复制人物参考图中的完整面部细节、复杂服装图案、材质或颜色。
- 图内不得出现可读文字、编号、标签、对白气泡、UI 标记、坐标、箭头、网格或地图边框。
- 多视角人物参考图只作为三庭五眼比例与服装阶级真值源；分镜模型必须忽略其拼合排版，禁止模仿三视图 / 多视角设定集布局。

### B3. Task Packet 必填文本块

黑白分镜 task_description 必须包含以下四个完整文本块：

1. **[Segment Visual & Tone Lock / 视听氛围锁]**：从故事板概要提取情绪目标、视觉节奏、声音 / 光影制作标准。
2. **[Segment Avoid / 绝对禁止项]**：完整搬运故事板概要中的 Avoid / 禁止项。
3. **[Scene Scheduling Sheet Truth Extract / 场景调度表真值核心摘录]**：只摘录会直接影响画面的真值核心，包括场景空间结构、角色站位表、道具坐标表、Eyeline Map、Panel 动作关键帧列表、参考图清单。不得逐字搬运整份调度表，不重复粘贴 Panel Character Spatial Matrix 原表。
4. **[Panel Spatial Lock]**：按 8 个 Panel 结构化展开，每个 Panel 覆盖本段调度表全部角色。

禁止用“已继承 / 详见上文 / 字段化空间锁已完整传入 / 按调度表执行”替代上述字段块。

### B4. Panel Spatial Lock 格式

```text
Panel Spatial Lock:
Panel <编号>:
- action_keyframe: <该格只表达一个动作关键帧>
- stage_geometry: <从 stage_blocking_sheet.md 与舞台定位图 table_node 继承的场景空间结构；@STAGE_MAP_VISUAL 不在每格重复书写>
- characters:
  - <角色 A>:
      identity_token: <Unique Identity Hash / Identity Token>
      stage_position_xy: (<xA>, <yA>)
      facing_target_xy: (<target_xA>, <target_yA>)
      screen_position: <画面构图层描述>
      camera_facing: <镜头朝向层描述>
      eyeline: <从 Eyeline Map 继承>
      distance_relation: <从 Blocking Sheet 继承>
      movement_path: <从 Blocking Sheet 继承的该格位移轨迹>
      state_action: <从 Blocking Sheet 继承的该格动作关键帧>
      previous_emotion_state: <上一 Panel 或入场前情绪>
      current_emotion_state: <本 Panel 情绪表演>
      next_emotion_target: <下一 Panel 合理过渡目标>
      visibility_state: <visible / background / foreground_partial / occluded / offscreen_but_spatially_locked>
- props:
  - <道具名>:
      stage_position_xy: (<x>, <y>)
      state: <可见 / 被操作 / 被遮挡 / 背景存在>
- camera_spatial_status:
    camera_position_xy: (<x>, <y>)
    camera_target_xy: (<x>, <y>)
    focal_length_mm: <焦段数值>
- hidden_coordinate_rule: 坐标只作为幕后空间锁，画面中不得出现坐标、文字、箭头、地图网格、色块或舞台定位图边框
- camera_movement_rule: 必须严格分离并明确人机运动权。若纯机位运动导致画面位置变化，标记 (camera moved, subject NOT moved)；若人机协同运动，必须精准解构各自运动轨迹（如: camera tracks along X-axis, subject moves along Y-axis）。且跨 Panel 必须死锁镜头轨迹向量链（Camera Vector Chain）：除非 continuity_note 明确写明切换机位(Cut)，否则 Panel X 的 `camera_position_xy`、`camera_target_xy` 与焦段终点状态必须 100% 物理对齐并作为 Panel X+1 的摄影机起点状态，严禁方向无序漂移。
- continuity_note: <说明本格是角色移动、镜头移动、还是情绪/意识转场>
```

展开要求：每个 Panel 必须列出本段角色站位表中的全部角色；每个角色必须保留身份令牌、9 项空间与动向字段，以及 3 项情感状态机字段：`identity_token`、`stage_position_xy`、`facing_target_xy`、`screen_position`、`camera_facing`、`eyeline`、`distance_relation`、`movement_path`、`state_action`、`previous_emotion_state`、`current_emotion_state`、`next_emotion_target`、`visibility_state`。若原始 `Panel Character Spatial Matrix` 是行内压缩格式，必须在派单前展开成上述块状格式；禁止让 image agent 自行展开。

### B5. Task Packet 预审与黑白分镜 QC

黑白 2x4 分镜不得直接从素材和调度表跳到图像生成。必须先完成可审查的总控派发任务包（Task Packet），其中 `task_description` 完整携带空间锁文本块，`image_paths` 真实挂载参考图片。预审失败时禁止调用 image agent。

Task Packet 必须同时满足：

1. `Reference Mention Map` 与 `Reference image paths` 完全对齐。
2. 包含 `[Segment Visual & Tone Lock]`、`[Segment Avoid]`、`[Scene Scheduling Sheet Truth Extract]`、`[Panel Spatial Lock]`。
3. 包含 8 个 Panel，且每个 Panel 覆盖本段全部角色。
4. 每个角色都保留身份令牌、9 项空间与动向字段、3 项情感状态机字段：`identity_token`、`stage_position_xy`、`facing_target_xy`、`screen_position`、`camera_facing`、`eyeline`、`distance_relation`、`movement_path`、`state_action`、`previous_emotion_state`、`current_emotion_state`、`next_emotion_target`、`visibility_state`。
5. 每个 Panel 都包含 `camera_spatial_status`，且保留 `camera_position_xy`、`camera_target_xy`、`focal_length_mm` 三项摄影机数学变量。
6. 不包含任何占位 / 概括短语。
7. 明确说明舞台定位图只作 Layout / 结构轻度参考锁和幕后二维坐标映射。

若 task_description 接近工具字符上限，优先压缩非空间叙事句，禁止压缩或删除 `[Segment Avoid]` 与 `[Panel Spatial Lock]`。底层 image agent 返回的物理 `prompt` 不作为字段完整性判定依据；判定依据是总控派发数据包结构完整，以及生成画面的空间关系、视线关系、动作关系、情绪氛围和黑白风格是否通过 QC。

QC 失败项：整体 2x4 大图分辨率不足、单格未保持 21:9 电影宽银幕构图、不是唯一成品、全角色矩阵不全、坐标 / 镜头层 / 可见性缺失、`camera_spatial_status` 或摄影机三项数学变量缺失、空间关系与舞台定位图 table_node 或 `stage_blocking_sheet.md` 不一致、Eyeline 错位、氛围未体现视听氛围锁、踩中 Avoid、画面出现坐标 / 文字 / 标签 / 箭头 / 网格 / 色块 / 地图边框。任一 FAIL 必须废弃输出，回到 Task Packet 阶段修复；无效图片不得进入 `video_prompt_list.md` 或后续视频阶段。

## 5. Panel 视频提示词与视频生成

### B6. video_prompt_list.md 与视频参考图映射

只生成 / 更新一个主文档 `video_prompt_list.md`。每一格 Panel 的视频任务数据包必须采用三流结构：1. **[Visual Block]** 建立 `@IMAGE_ASSET_N` / `@STORYBOARD_GRID` / 可选光影资产与底层 `image_paths` 的绝对物理映射；总控 Agent 必须无条件从 A1 中提取对应的【人物面部三庭五眼结构表】描述词，将其强行注入到视频生成提示词的 [Visual Block] 中，作为人物外貌的刚性特征约束；2. **[Audio & Underscore Prompt Hook]** 依据 A1 动作节拍与情绪目标，同步输出环境音效（SFX）与电影级氛围乐（Cinematic Underscore / ACE Studio）提示词锚点，并必须包含 Audio-Visual Sync Anchor / Rhythmic Mapping：把动作关键帧的物理位移时间点（如 3s 转头、5s 重击）与 SFX Frequency、低频鼓点、呼吸/衣料/脚步声锚点逐项对齐，重击动作必须绑定低频或冲击音峰值；3. **[Camera & Continuity Block]** 物理携带并解构 B4 的摄影机核心控制流，必须同时包含全局机位运动描述 `camera_movement_rule`，以及 B4 对应 Panel 的摄影机高精数学变量 `camera_spatial_status`（包含 `camera_position_xy`, `camera_target_xy`, `focal_length_mm`），以刚性离散参数强控视频大模型的机位空间，严禁仅使用描述性文本而遗漏数学坐标真值。

[Camera & Continuity Block] 必须严格采用以下 JSON-like 键值格式，严禁压缩为单行字符串或把坐标与焦段线性拼接：

```text
{
  "camera_movement_rule": "<规则文本>",
  "continuity_note": "<转场备注>",
  "camera_spatial_status": {
    "camera_position_xy": [x, y],
    "camera_target_xy": [x, y],
    "focal_length_mm": <数值>
  }
}
```

- `@IMAGE_ASSET_N`：人物、场景、关键道具等彩色资产图。其 N 的自然数编号必须与视频阶段物理 `Reference image paths`（从 `image_paths[0]` 开始）的排列顺序绝对一一对应，严禁语义与索引错位。
- `@STORYBOARD_GRID`：固定放置在视频阶段 `image_paths` 倒数第二位，代表当前段落黑白 2x4 分镜图，只提供 Panel 取景、站位与运动链。
- 可选光影资产：固定放置在 `image_paths` 最后一位；若无可选光影参考图，则不传。

视频任务包必须包含 `Video Reference Mention Map / 视频参考图引用映射` 与真实 `Reference image paths`。二者必须与 B8 合并 / 未合并后的物理顺序刚性对齐。禁止悬空调用、孤儿图片、舞台定位图入参、裸 `(x, y)` 坐标、字幕、删减用户已确认内容。

### B6.1 资产引用动态重索引机制

若触发 STEP B8 的资产合并，总控 Agent 在执行最终级联前必须启动【索引重算任务】：动态遍历 `video_prompt_list.md` 中所有 `@IMAGE_ASSET_N` 引用，基于合并后真实 `image_paths` 物理顺序重新指派 N 值，并同步重写 `Video Reference Mention Map / 视频参考图引用映射`。最终派单包中 N 的映射必须与图片物理序列实现 1:1 原子级覆盖；严禁悬空引用、重复编号、跳号编号或沿用合并前旧索引。

### B6.2 视频多模态特征解耦

`video_prompt_list.md` 与正式视频 task_description 必须在最前端写入原生英文 Feature Decoupling Guard 阻断块：

```text
[CRITICAL RESTRUCTURING]: The input `@STORYBOARD_GRID` (image_paths[second-to-last]) is STARKLY FORBIDDEN to be used for character appearance, facial features, clothing texture, props details, or artistic style. It is ONLY a layout grid.
- For the current segment, extract ONLY shot scale, composition ratio, body displacement physical chain, camera movement and blocking trajectory from the corresponding [Panel X] (where X is the current panel index 1-8) within the 2x4 grid of `@STORYBOARD_GRID`.
- Character facial structure, 三庭五眼骨相, skin flaws, clothing class color, scene art direction and lighting tone MUST come 100% and ONLY from color asset images / optional lighting asset as the Color Truth Source.
- The final color video MUST NOT retain black-white sketch lines, panel borders, white background or storyboard texture from `@STORYBOARD_GRID`.
```

若任一 Panel 缺少该英文阻断块，或把 `@STORYBOARD_GRID` 描述为角色 / 服装 / 场景美术参考，判定视频派单包 FAIL。

### B7. 视频前增强选择

正式生成视频前，用一次 `question` 弹窗完成单次事务确认：同时让用户选择是否启用自建能力增强，并确认当前 `video_prompt_list.md` 基础内容可进入视频生成。用户选择后，不再回写 `video_prompt_list.md` 触发第二轮自检；必须把增强选择动态注入同一轮最终视频模型 dispatch 参数并立即开始视频生成。禁止第二个确认弹窗，禁止仅因动态参数注入触发第二次自检或 question 循环。

增强选项至少包含：启用推荐组合（镜头方案 + 表演层 + 身体语言 + 光影 + 动作物理 + ARRI ALEXA 65 + 剪辑连接点）、只增强镜头、只增强表演、只增强身体动作、只增强光影、只增强动作物理、不启用增强、自定义组合。用户选择后，总控 Agent 必须依据选择动态将增强描述拼接灌入最终的视频模型派单参数中。若用户选择自定义组合，总控必须强制启动【自定义内容防爆门禁】：对用户输入的纯文本执行结构化语义清洗，提取且仅提取契合镜头方案、表演、身体动作、光影、动作物理五类维度的核心修饰词；无条件过滤任何具有宏观剧本修改倾向、替换角色 / 世界观 / 时空设定、破坏空间站位、破坏 Camera Vector Chain 或前后自相矛盾的脏指令。最终动态拼接后的派单参数必须符合多模态视频模型的语法边界，否则强制触发熔断并回到人工 `question` 处理。

### B8. 视频参考图 9 张上限与阶梯合并

视频模型底层输入通道上限为 9 张。派发视频前，必须计算真实参考图片总数：人物参考图 N 张 + 场景参考图 M 张 + 关键道具参考图 K 张 + 分镜参考图 1 张 + 可选光影参考图 L 张。禁止把舞台定位图传给视频模型。

若总数 ≤ 9 张，按以下顺序派发：

```text
人物参考图 -> 场景参考图 -> 关键道具参考图 -> 分镜参考图 -> 可选光影参考图（若有）
```

若总数 > 9 张，必须先用 `question` 弹窗提醒用户触发超限拦截，再按低叙事权重到高叙事权重阶梯合并：关键道具参考图 -> 场景参考图 -> 人物参考图。资产合并必须使用网格无损拼接技术（Grid Stitched Canvas），严禁层叠、融合（Blend/Overlap）或裁切；合并图只允许用 `White Dashed Line` 白色虚线网格分隔，不得使用彩色框、彩色边界或会污染视频色彩的标记。网格复合大图的最终画布物理长宽比必须经过归一化计算强制锁定为 16:9；若 N x M 网格天然外边框不是 16:9，总控 Agent 必须指令底层画布工具在上下或左右边缘进行纯黑色像素无损填充（Black Padding），严禁把非 16:9 的合并图外边框裸露传入下游视频模型，防止多模态解析器把视频画幅压扁或拉长。必须在 `video_prompt_list.md` 中强制使用如下格式建立节点映射，严禁大白话描述：`在合并资产大图 @IMAGE_ASSET_N 中，第一行第一格为道具A，第一行第二格为道具B。全视频中上述资产的外观与色彩完全以此网格内图像为唯一彩色还原真值，无条件忽略任何网格边界白线与黑色填充边。` 若触发资产合并，按 B6.1 执行索引重算任务并重写映射。每次合并后重新计算；只有物理 `Reference image paths` 数量 ≤ 9 且所有合并大图均为 16:9 才可进入视频生成。若仍超过 9 张，弹窗让用户选择深度合并、删减资产或暂停链路。

## 6. 链路自检

进入每个阶段前检查：

| 检查项 | 通过标准 |
|---|---|
| 顺序 | STEP A 完成后才进入 STEP B；同一段不能跳过舞台定位图直接做黑白分镜 |
| 弹窗 | 每个制作动作前真实使用 `question`；B7 按专门规则一次性覆盖增强选择、文档确认和同轮视频生成 |
| 段落隔离 | 当前段落两份 MD、处于 ACTIVE 激活态的视觉资产、舞台定位图、黑白分镜均匹配同一 `<集数 + 段落编号 + 场景名>`；必须无条件豁免并忽略任何带有 `[DEPRECATED_OLD_VERSION]` 标签的过期废弃资产，严禁因历史版本命名变更误判 FAIL |
| 故事板概要 | 包含剧情范围、8 格动作、人物 / 场景 / 道具信息提取表、人物三庭五眼结构、角色身份令牌表、Medium Lock、Avoid，并写入 lineage 审计链 |
| 场景调度表 | 包含 Global Environment Context、场景空间结构、角色站位表、道具坐标表、Panel 动作关键帧、参考图清单，并生成 / 存储独立 `stage_blocking_sheet.md`；`Panel Character Spatial Matrix` 必须使用 `[角色名: X, Y, Z 相对坐标]`，`Eyeline Map` 必须使用 `[发射源角色 -> 接收目标对象: 视线角度/朝向向量]`；角色站位表必须含 Identity Token 与情感状态机字段 |
| 舞台定位图 | 必须由 `stage_blocking_sheet.md` 派生 / 更新；只加载当前段落场景参考图；不得加载人物、道具、光影或黑白分镜图；必须同时存在 media_node 与 table_node；最终画面为纯 2D 鸟瞰网格平面，无 3D 透视纵深，色块旁标签物理可见且不重叠 |
| 视觉资产 | 只生成允许范围内资产；人物参考图必须是 16:9 多视角统筹画布 |
| 黑白分镜 | 进入前已完成 `table_node` 物理版本 / 生成时间戳与 `stage_blocking_sheet.md` 核心空间真值一致性比对；模型与物理参数锁定 `image2 / 4k / 16:9 / low`；确保单格 Panel 切片后物理有效分辨率不低于 1k，且单格画面在视觉上呈现 21:9 宽荧幕视效；唯一成品；Task Packet 满足 B5，含身份令牌、情感状态机和 Entry/Exit Guard 结果；画面空间、视线、动作、氛围和黑白风格通过 QC |
| 视频提示词 | 包含 [Visual Block]、[Audio Hook]、[Camera Block]；[Visual Block] 必须注入 A1【人物面部三庭五眼结构表】外貌真值；[Audio Hook] 必须包含 Audio-Visual Sync Anchor / Rhythmic Mapping，将动作时间点、SFX Frequency 与低频/冲击音峰值对齐；[Camera Block] 必须以 JSON-like 键值格式携带 `camera_spatial_status` 及 `camera_position_xy` / `camera_target_xy` / `focal_length_mm`，严禁单行线性拼接；建立唯一占位符映射；不传舞台定位图、不写裸坐标、不生成字幕；若原始图片总数 > 9 张，必须校验文档中是否包含 B8 规定的白色虚线网格合并映射标识、16:9 Black Padding 归一化记录、索引重算任务结果及对应的用户弹窗选择状态，严禁在 B8 合并机制生效过程中引发误判拦截 |
| 视频生成 | 作为 B7 单次事务弹窗授权完成后的【同轮后置断言门禁】同步触发，不作为独立外部自检阶段拦截；必须满足 B7 已授权、B8 合并 / 删减后最终传入视频模型的物理 `Reference image paths` 数量 <= 9 张、`@STORYBOARD_GRID` 固定倒数第二位、可选光影资产固定最后一位；校验通过立即无缝下发生成，严禁反流回 markdown 触发二次循环 |

任一失败，回到对应阶段修复，不跳过。

### 6.1 自检 FAIL 与重试熔断防爆机制

任何检查项若判定 FAIL，总控 Agent 必须依据第 10 条重做清理机制执行旧资产物理隔离，并在当前节点自动发起单次重跑。必须死锁最大重试熔断阈值（Max Retry = 3）：若任一动作或 QC 节点连续触发 3 次 FAIL，严禁继续自动盲目重跑；必须立即熔断自动化链路，弹出高危 `question` 弹窗向人工导演提报当前上下文毒点、提示词冲突及详细 FAIL 原因，等待人工介入或回溯至剧本层重写。

## 7. 输出清单

完整链路结束后应至少有：`故事板概要`、`场景调度表`、`stage_blocking_sheet.md`、人物 / 场景 / 道具 / 可选光影参考图、`舞台定位图`、唯一黑白 2x4 分镜图、`video_prompt_list.md`，以及通过 `question` 弹窗确认后生成的视频片段。

## 8. 触发测试

应触发：“帮我先做舞台定位图再做黑白宫格分镜”“这个故事短片需要先稳定站位，然后从零做分镜”“用 stage-position-map 的坐标继续 zero-to-story”。

不应触发：“只画一个单人角色三视图”“给这张图换背景”“只生成一段单镜头雨景视频”。
