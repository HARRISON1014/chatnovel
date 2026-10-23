# ChatNovel 增量创作、一致性与文风学习设计

> 模型与部署的具体选择见 [模型选型与硬件适配方案](model-selection.md)，其中提供跨硬件档位和每月 100 元以内的可选预算预设。

本文是对 [architecture-plan.md](architecture-plan.md) 的需求补充。它把“逐步生成、持续修改、前后一致、学习文风”定义为系统的核心约束。若旧方案中的描述与本文冲突，以本文为准。

## 1. 产品目标的重新定义

ChatNovel 不是一次性小说生成器，而是一个有状态的协作式写作系统：

- 作者决定故事事实、结构和最终文字。
- 模型一次只处理一个明确范围的写作任务。
- 每一步生成都建立在一个确定的版本和一组可追溯的故事状态上。
- 新正文在进入正史前必须经过人工接受和一致性检查。
- 文风可以从用户提供、获授权的文本中抽象学习，应用到新的内容上。
- 任何生成结果都能比较、修改、撤销和恢复。

系统需要把“生成正文”和“更新故事状态”拆成两个阶段。模型生成的正文首先是候选修订；只有用户接受后，系统才从该修订中提取新的事实和事件，并把它们作为待确认状态写入故事库。

## 2. 增量写作模型

### 2.1 最小写作单元

长篇写作的最小自动化单元是场景中的“写作步”，而不是整章或整本书。

一个写作步可以是：

- 生成场景开头的一段。
- 继续当前光标后的一个片段。
- 完成一个动作、对话或转折节拍。
- 对选中文本进行局部改写。
- 把多个已接受片段整理成完整场景。
- 对一段正文进行风格润色，但不改变事实和情节。

默认输出应较短，并允许作者配置字数或 token 上限。例如中文正文可以默认生成 300—1000 字；场景扩写可以提高上限，但仍然要求先预览和接受。长度不是硬编码，应该由任务类型、场景阶段和用户设置共同决定。

### 2.2 写作状态机

每个场景都有明确的状态：

~~~text
planned
  -> card_ready
  -> drafting
  -> review_pending
  -> accepted
  -> summarized
  -> continuity_checked
  -> locked
~~~

其中：

- planned：只有大纲节点，还没有可执行的场景卡。
- card_ready：目标、冲突、人物、地点、视角和预期结果已经填写。
- drafting：正在产生候选片段，可以暂停或取消。
- review_pending：候选文本尚未进入正文。
- accepted：作者接受了一个修订。
- summarized：已经生成当前场景摘要和候选事实。
- continuity_checked：已完成硬约束和软约束检查。
- locked：作者暂时锁定场景，后续修改需要创建新分支。

状态机禁止模型自行跳过人工审核。模型可以建议下一步，但不能自行把候选内容写入正史。

### 2.3 修订版本图

正文不是一条不可逆的字符串，而是一个版本图。每个修订至少包含：

~~~text
revision_id
document_id
parent_revision_id
base_revision_id
operation_type
content
selection_range
run_id
created_by
status: candidate | accepted | rejected | superseded
created_at
~~~

一个用户可以：

1. 从当前版本生成续写候选。
2. 拒绝候选，保留当前版本不变。
3. 接受候选，创建新的 accepted 修订。
4. 从旧版本重新生成另一条分支。
5. 比较两个分支，并把其中一个设为当前版本。
6. 把已接受的版本回滚为新的工作分支，而不是删除历史。

因此“修改”不会破坏过去的证据，连续性检查也能指出某个事实在哪个版本中发生了变化。

### 2.4 一个完整写作步的流程

~~~mermaid
sequenceDiagram
    participant U as 作者
    participant E as 编辑器
    participant O as 编排器
    participant C as 上下文与状态库
    participant M as 模型
    participant V as 一致性检查器

    U->>E: 选择场景和操作
    E->>O: 提交 base_revision_id 与任务
    O->>C: 读取场景卡、正史事实、时间线、风格配置
    C-->>O: 返回 ContextPack 与约束
    O->>M: 生成受限长度的候选片段
    M-->>E: 流式返回候选文本
    E->>U: 显示 diff、来源和模型信息
    U->>E: 接受、修改或拒绝
    E->>O: 接受修订
    O->>V: 检查事实、时序、人物状态和风格
    V-->>E: 通过、警告或阻断
    E->>U: 展示问题和候选修复
    U->>E: 确认正文与事实更新
    E->>O: 创建摘要、事件和事实候选
~~~

如果生成中途断线，运行记录保留已经收到的片段。作者可以继续同一次运行、从最后一个完整句子重新生成，或丢弃候选结果。

## 3. 一致性系统

### 3.1 三层事实状态

为了避免模型把未经确认的内容当成事实，故事数据分为三层：

| 层 | 内容 | 产生方式 | 对后续生成的影响 |
| --- | --- | --- | --- |
| Canon | 作者确认的正史事实、时间线和规则 | 作者手工维护或确认提取结果 | 强约束 |
| Accepted Text | 已接受但尚未完全结构化的正文 | 作者接受修订 | 作为正文上下文，事实需提取 |
| Candidate | 模型从新文本中推断的事实、事件和关系 | 自动抽取 | 只能作为待审核建议 |

只有 Canon 默认进入“必须遵守”的约束集合。Candidate 可以被模型用于解释当前结果，但不得覆盖 Canon。

### 3.2 故事状态图

小说的一致性不能只靠搜索相同人名。系统需要维护实体和状态变化：

~~~text
Entity
  ├─ Character: 身份、年龄、外貌、能力、关系
  ├─ Location: 地点、所属、可达关系
  ├─ Object: 物品、持有者、状态
  ├─ Organization: 成员和层级
  └─ Rule: 世界规则和例外

StateAssertion
  subject_id
  property
  value
  valid_from_event
  valid_to_event
  source_revision_id
  status: canon | candidate | revoked

StateTransition
  event_id
  subject_id
  property
  old_value
  new_value
  trigger
  source_revision_id
~~~

例如“角色拥有钥匙”不是一个永远为真的标签，而是一个有开始事件、结束事件和来源的状态。这样才能判断角色在钥匙已经交给别人后又使用它，是否构成矛盾。

### 3.3 检查时机

一致性检查分为三道门：

1. **生成前检查（Preflight）**<br>
   检查当前场景卡与 Canon 是否冲突，确认人物在场、地点可达、时间线位置明确，并生成本次任务的约束清单。

2. **候选文本检查（Candidate Check）**<br>
   候选片段还未进入正文时，检查它是否引入与 Canon 冲突的姓名、年龄、地点、物品状态、视角和时间。

3. **接受后检查（Commit Check）**<br>
   作者接受候选后，重新抽取事实和事件，与所有已接受内容比较，并更新候选状态。任何新冲突都必须在场景完成前处理。

生成前和接受后的检查不能省略。只在整章写完之后检查，问题往往已经扩散到多个场景。

### 3.4 硬约束与软约束

| 级别 | 示例 | 默认行为 |
| --- | --- | --- |
| Blocker | 已确认角色在同一时刻出现在两个地点；死亡角色被当作存活；物品同时被两人持有 | 阻止提交为当前版本，要求作者处理 |
| Warning | 人物语气异常；年龄计算可疑；前后称谓变化；场景目标没有完成 | 允许接受，但需要作者确认 |
| Info | 可能重复的描写；节奏变化；风格偏离 | 提供建议，不阻止保存 |

“阻止”只阻止把结果标记为通过一致性检查，不删除候选文本。作者可以标记为“有意反转/时间线例外”，系统随后创建一条显式规则或 retcon 记录，避免同类问题持续出现。

### 3.5 检查器组合

一致性检查器使用三种互补方式：

- **确定性检查**：实体名称、别名、时间排序、年龄计算、状态转移、地点关系、物品持有和正史规则。
- **结构化抽取**：模型把候选正文转为事件、状态变化和关系声明，再交给确定性规则验证。
- **语义审阅**：模型检查动机、因果、视角、语气和潜在软矛盾，并返回证据片段。

模型审阅只能提出问题，不能直接判定正史。每个问题必须携带：

~~~text
issue_id
severity
description
evidence_revision_id
evidence_span
conflicting_assertion_id
suggested_resolution
status
~~~

### 3.6 修复方式

发现问题后，系统提供三种受控操作：

- **局部修复**：只重写产生冲突的句子或段落，锁定事实和情节目标。
- **回到状态点**：从冲突发生前的修订重新生成后续片段。
- **确认反转**：作者说明这是有意设定，系统建立新的时间线、规则或别名。

修复操作必须显示“哪些事实被锁定、哪些事实允许变化”。这能防止模型为了修复一个年龄错误而悄悄改掉人物关系或场景目标。

## 4. 故事一致性的数据模型补充

在原有数据模型上增加以下表：

~~~text
story_entities
  id, project_id, type, canonical_name, aliases_json,
  description, first_seen_event_id, status

canon_assertions
  id, project_id, subject_id, predicate, object_value_json,
  valid_from_event_id, valid_to_event_id,
  source_revision_id, confidence, status

state_transitions
  id, project_id, event_id, subject_id, predicate,
  old_value_json, new_value_json, source_revision_id

timeline_events
  id, project_id, sequence_key, story_time_start,
  story_time_end, location_id, description, source_revision_id

scene_checkpoints
  id, scene_id, revision_id, snapshot_json, created_at

continuity_issues
  id, project_id, run_id, severity, description,
  evidence_json, conflicting_ids_json, resolution_json, status

retcons
  id, project_id, issue_id, rule_text,
  effective_event_id, author_note, status
~~~

scene_checkpoints 保存每个已接受场景结束时的人物位置、持有物品、健康状态、关系和未解决悬念。生成下一场景时，优先读取上一个 checkpoint，再读取更早的 Canon，减少模型自行推断。

## 5. 文风学习功能

### 5.1 功能边界

文风学习的目标是学习“表达方式”，不是复制参考作品的剧情、人物或原句。

系统从用户提供或明确获授权的文本中提取：

- 句子长度和长短句变化。
- 段落长度、换段频率和对白比例。
- 叙事人称、视角稳定性和时态。
- 词汇密度、口语程度、抽象词与具象词比例。
- 动作、感官、心理、环境和对白的占比。
- 修辞倾向、意象密度、比喻类型和重复模式。
- 叙事节奏、场景推进速度和章节收束方式。
- 对人物说话方式、称谓和语气的描述。
- 需要避免的词语、表达和格式。

这些特征组成可编辑的 StyleProfile。模型不能仅凭一个作者名字调用“模仿某某”的黑箱模式；用户可以查看、修改和组合风格属性。

### 5.2 文风库对象

~~~text
style_sources
  id, project_id, name, source_type,
  consent_status, storage_scope, created_at

style_documents
  id, source_id, title, content_hash, local_path,
  language, metadata_json

style_profiles
  id, project_id, name, version, feature_json,
  example_policy, privacy, created_at

style_examples
  id, profile_id, document_id, content,
  tags_json, feature_json, source_span, allowed

style_runs
  id, profile_id, task_type, input_revision_id,
  output_revision_id, similarity_metrics_json,
  overlap_report_json, created_at
~~~

原始参考文本默认保存在本机。云端服务只在用户明确打开“发送文风样本”后使用；默认发送抽象后的 StyleProfile，不发送整篇参考作品。

### 5.3 学习流程

1. **导入**：作者导入一个或多个文本文件，填写来源名称和使用权限。
2. **分段**：按章节、场景、段落切分，去除标题、元数据和无关内容。
3. **分析**：使用本地规则统计和模型分析生成 StyleProfile。
4. **校订**：作者查看并修改风格属性，删除不希望学习的样本。
5. **标注**：可给样本标记“对白”“动作”“内心”“环境”等标签。
6. **应用**：在生成任务中选择一个或多个风格配置，设置权重。
7. **检查**：生成后比较风格指标和用户偏好，提供偏离项。
8. **迭代**：作者接受或拒绝结果后，记录哪些风格特征有效，更新配置版本。

### 5.4 风格应用的三层策略

从低风险、易解释的方式开始：

1. **属性约束**：把 StyleProfile 转成明确的写作要求，例如“短段落、对白占比高、第三人称限知、少用抽象形容词”。
2. **检索样例**：只选取与当前任务类型相同的少量短样例，作为表达参考；样例不得包含当前故事的事实。
3. **可选适配模型**：当有足够的授权语料和评测结果后，再考虑 LoRA 等本地适配。它是高级功能，不能作为 MVP 的前置条件。

多风格组合使用属性权重，例如“对白节奏 60% + 冷峻叙述 40%”，而不是把多个完整作家文本拼接到提示词中。

### 5.5 防止内容泄漏和原句复制

文风功能需要单独的原创性保护：

- 参考文本与故事正文分开存储，参考文本中的人物、地点和情节不进入故事 Canon。
- 生成任务不允许以“接着这段原文写”为默认操作。
- 对生成结果执行 n-gram 重叠和近似句检测，发现与参考库高度相似时发出警告或阻止提交。
- 只把必要的短样例发送给远端模型，并设置样例数量和长度上限。
- 记录每个输出使用过的样例来源，方便作者审阅和移除。
- 用作者可编辑的属性名描述风格，避免把系统设计成按在世作者姓名一键复刻。

这既能提高原创性，也能让作者知道系统究竟学到了什么。

### 5.6 文风质量评估

文风评估不能只依赖语言模型主观打分。至少记录：

- 句长、段落长度和对白比例与目标范围的距离。
- 人称、时态和视角违规次数。
- 词汇重复率、套话密度和禁用表达命中数。
- 参考样例的近似句重叠率。
- 作者接受率、二次修改量和主观评分。

风格匹配是辅助指标，故事事实和可读性优先。系统不应为了达到某个统计分布而破坏情节或制造不自然的句子。

## 6. ContextPack 的重新定义

每次写作步都生成一份带版本号的 ContextPack，分成四部分：

~~~text
Task
  当前操作、选区、字数、语气和输出边界

StoryState
  当前 base_revision、scene_checkpoint、Canon assertions、
  时间线窗口、人物状态和锁定事实

StyleState
  选中的 StyleProfile、属性约束、允许的样例和禁用项

Constraints
  生成前检查结果、硬约束、待解决问题和安全边界
~~~

ContextPack 需要保存：

- 每个片段的 source_id、source_type、revision_id。
- 组装顺序和裁剪原因。
- prompt 模板版本。
- 使用的模型 profile 和参数。
- 发送到远端的内容范围。

上下文不足时，先压缩 Accepted Text 和旧摘要，再裁剪低优先级样例；不能静默裁剪当前选中文本、Canon 或硬约束。

## 7. 面向新需求的 MVP 调整

### MVP 必须包含

- 场景级增量生成，默认限制单次输出长度。
- 续写、局部改写、段落扩写和接受/拒绝 diff。
- 修订版本图、分支和回滚。
- 场景卡和场景结束 checkpoint。
- Canon、Candidate 两层事实状态。
- 生成前、候选文本、接受后三道一致性检查。
- 实体、时间线、状态转移和事实来源。
- Blocker、Warning、Info 三种问题等级。
- 文风样本导入、StyleProfile 提取和属性约束应用。
- 参考样例的来源追踪与近似句检测。
- 所有运行的 ContextPack、模型、提示词和版本记录。

### MVP 可以延后

- LoRA 或其他微调训练。
- 自动向量检索。
- 多智能体审稿。
- 多用户共享文风库。
- 自动发布和平台排版。
- 无人值守的连续章节生成。

### 新的验收流程

一个用户应能完成以下闭环：

1. 创建三个场景卡，并为每个场景定义人物、地点、冲突和预期结果。
2. 在第一个场景中分三次生成片段，每次都能接受或拒绝，并查看修订图。
3. 接受后生成场景摘要和 checkpoint。
4. 在第二个场景中故意生成一个与第一个场景冲突的事实，系统在候选或提交阶段标记 Blocker。
5. 选择局部修复或确认 retcon，并看到 Canon 的变化记录。
6. 导入一组有权限使用的参考文本，生成可编辑的 StyleProfile。
7. 用该 Profile 生成新场景，查看风格指标和参考样例来源。
8. 触发一次近似句检测，确认高重叠结果不能无提示地进入正史。
9. 导出正文、故事 Canon、时间线、StyleProfile 和运行记录，之后可以恢复项目。

## 8. 实施顺序

### Phase A：增量编辑闭环

先完成场景卡、写作步、流式结果、diff、接受/拒绝和修订版本图。此阶段不追求自动写完整章节，重点是让作者愿意连续使用。

### Phase B：一致性基线

加入实体、Canon assertions、时间线、checkpoint、结构化事实抽取和确定性检查。先覆盖姓名、别名、年龄、地点、时间、物品和人物状态，再加入语义审阅。

### Phase C：文风 Profile

加入参考文本导入、本地分析、StyleProfile 编辑、属性约束和少量样例检索。先不训练模型，使用可解释的配置验证作者是否真正需要这项能力。

### Phase D：质量与高级模型

加入近似句检测、风格评测、模型路由、嵌入检索和可选本地 LoRA。每项能力都要用固定样例比较生成质量、矛盾率、成本和延迟。

## 9. 最重要的架构原则

- **正文版本是事实来源，模型输出只是候选。**
- **Canon 是强约束，模型记忆不是事实。**
- **每个状态变化都要有来源修订和故事时间。**
- **一致性检查提出证据和修复范围，不替作者偷偷改剧情。**
- **文风学习抽象表达特征，参考文本与故事内容隔离。**
- **生成必须可暂停、可恢复、可回滚。**
- **任何自动化都必须留下可审阅的运行记录。**

实现顺序应优先保证这七条原则，再扩展模型数量、界面装饰和自动化程度。
