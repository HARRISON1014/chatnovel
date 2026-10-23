# ChatNovel 模型选型与硬件适配方案

资料核对日期：2026-10-23。当前为设计建议，尚未在小说评测集上运行模型，也未完成跨硬件验证。

本项目面向不同配置的用户，支持纯 API、纯本地和混合部署。6GB 显存与每月 100 元预算作为可选的入门配置示例，不限制项目的模型规模或硬件范围。开源应用可以连接商业 API；开放权重模型和应用代码分别遵循各自许可证。

## 1. 推荐优先接入的模型

| 方向 | 首批候选 | 建议用途 |
| --- | --- | --- |
| 中文云端主力 | Qwen 3.7 Plus | 场景续写、局部改写、大纲、文风配置应用 |
| 云端对照 | DeepSeek Flash；DeepSeek V4 Pro 按需评测 | 比较中文正文、剧情规划与连续性审阅 |
| 低价辅助 | Qwen 3.8 Flash | 摘要、事实抽取、风格特征分析 |
| 本地入门 | Qwen3.5-4B、Qwen3.5-9B 的量化版本 | 离线摘要、候选事实抽取、短段改写 |
| 本地较高配置 | Qwen3.8-27B；Qwen3.6-35B-A3B | 本地场景写作与审阅候选 |
| 国际 API 对照 | GPT-6.1 Sol、Claude Sonnet 5.5 | 复杂改写和审阅的可选对照 |
| 后续语义检索 | Qwen3-Embedding-0.6B | 人物、事件和参考段落检索 |

上述任务分工是工程上的起点，不是已经验证的文学质量排名。官方资料支持型号、协议、价格和部署能力；中文叙事、人物对白与文风表现要通过项目自己的盲测决定。

阿里云将 Plus 用于内容生成等综合任务，Flash 用于降低成本。[Qwen 官方选型说明](https://www.alibabacloud.com/help/en/model-studio/text-generation-model)

本地型号与权重以具体官方模型卡为准：[Qwen3.5-4B](https://huggingface.co/Qwen/Qwen3.5-4B)、[Qwen3.5-9B](https://huggingface.co/Qwen/Qwen3.5-9B)、[Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B)、[Qwen3.6-35B-A3B](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)。不要凭家族名称推断某个尺寸存在。

向量检索后置于 FTS5；需要时再加入 [Qwen3-Embedding-0.6B](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B)，生成模型不兼任嵌入模型。

## 2. 按小说任务选择，而不是为每个角色安装一个模型

同一个模型可以承担多个任务，MVP 接入一个主力模型就能验证完整写作流程，之后再按评测拆分。

| 任务 | 需要的能力 | 建议起点 |
| --- | --- | --- |
| 场景续写、局部改写 | 遵守已知事实、保持视角、接受文风要求 | Plus 与 DeepSeek Flash 比较；本地 9B/27B 作为离线候选 |
| 大纲、动机与因果审阅 | 多约束推理、带证据的问题说明 | 主力模型的思考模式；复杂案例再比较 Pro/Sol |
| 摘要和事实抽取 | 来源定位、实体别名、可靠 JSON | 本地 4B/9B 或 Qwen Flash，使用 Schema 校验 |
| 连续性检查 | 事实比对、故事时间、状态转移 | 规则引擎 + 模型提出语义问题 |
| 文风分析 | 提取表达特征，区分风格与剧情 | 规则统计 + 同一主力模型或本地小模型 |
| 文风应用 | 保持剧情不变并调整语言表达 | 正文模型 + 已确认的 StyleProfile |

正文任务优先比较非思考或低推理配置，规划和疑难审阅再增加推理预算。这是待测的成本策略；推理开关和采样参数必须按实际模型协议映射，不统一强制 temperature。

结构化输出只约束格式，仍需检查事实来源和故事状态。抽取失败必须标记未完成，不能把空结果视为“没有矛盾”。没有任何一个推荐模型可以单独保证长篇小说零矛盾。[OpenAI official documentation：Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs)

## 3. 云端价格与预算比较

以下为核对时的每百万 token 输入/输出单价。人民币和美元分表，不能直接比较数值。模型别名、促销和价格可能变化；应用应记录价格快照和实际返回型号。

| 模型 ID | 输入（人民币） | 输出（人民币） | 使用条件 |
| --- | ---: | ---: | --- |
| qwen3.7-plus-2026-05-26 | 2 元 | 8 元 | 北京区域；单次输入不超过 256K；按原价 |
| qwen3.8-flash | 0.8 元 | 2.7 元 | 北京区域；单次输入不超过 1M |
| deepseek-flash | 1—2 元 | 4—8 元 | 未命中缓存；低值为空闲时段，高值为高峰时段 |
| deepseek-v4-pro | 4.5—9 元 | 13.5—27 元 | 未命中缓存；空闲/高峰两档 |

来源：[百炼官方价格](https://help.aliyun.com/zh/model-studio/model-pricing)、[DeepSeek 官方模型与价格](https://api-docs.deepseek.com/zh-cn/quick_start/pricing/)。估算不依赖免费额度、促销或缓存命中。Plus 超过所列输入区间会进入更高价格档。

| 模型 ID | 输入（美元） | 输出（美元） | 项目中的定位 |
| --- | ---: | ---: | --- |
| gpt-6-luna | $0.10 | $0.50 | 低价抽取与摘要对照 |
| gpt-6.1-sol | $2 | $10 | 平衡型复杂任务对照 |
| gpt-6-astra | $10 | $50 | 可选的高预算审阅 |
| claude-sonnet-5-5 | $2 | $10 | 写作与改写对照 |

来源：[OpenAI official documentation：模型目录](https://developers.openai.com/api/docs/models)、[Claude Sonnet 5.5 官方文档](https://platform.claude.com/docs/en/models/sonnet-5-5/overview)。这张表不代表这些模型在中文小说上已经胜过前表；美元计价配置需要按结算币种、汇率及额外费用单独预算。

接口上保留独立的 OpenAI Responses、OpenAI 兼容 Chat Completions、Anthropic Messages 和 Ollama 适配器。统一的是内部任务契约；供应商的思考参数、流式事件和 JSON 能力不必相同。先验证选中端点支持的功能，再启用相应写作任务。

## 4. 多硬件档位：选择起点，不承诺仅凭显存就能运行

Q4/Q5/Q8 表示常见的权重量化档位。运行内存还包括量化元数据、上下文缓存、推理引擎和系统占用。下表的上下文指总 token 预算，需预留输出空间；每个配置均需实际试跑确认。

| 硬件条件 | 本地候选起点 | 起始上下文 | 适配建议 |
| --- | --- | --- | --- |
| CPU，无独显 | Qwen3.5-4B Q4；内存不足再选更小模型 | 2K—4K | 按系统可用内存选择，重点验证速度 |
| 4—6GB 显存 | Qwen3.5-4B Q4 | 4K；通过后尝试 8K | 单任务并发，留出桌面显存；4GB 可能需部分 CPU 卸载 |
| 8—12GB 显存 | Qwen3.5-9B Q4；8GB 紧张时回退 4B | 4K—8K | 12GB 可进一步测试 16K |
| 16GB 显存 | Qwen3.5-9B Q5/Q8 | 8K—16K | 比较精度、速度与缓存占用 |
| 24GB 显存 | Qwen3.8-27B Q4 | 8K—16K | 场景写作与审阅候选，仍需测量峰值内存 |
| 32—48GB 显存 | 27B 较高精度、Qwen3.6-35B-A3B Q4 | 16K—32K | 按模型与推理引擎进一步测试并发 |
| 更大显存或多卡 | 经评测选定的更大模型/更高精度 | 按测量配置 | 考虑卡间通信、并行策略和并发 |
| Apple Silicon | 同类量化模型 | 按可用统一内存测量 | 统一内存与系统共用，不能直接套用独立显存表 |

这些范围是保守的工程起点，不是实测硬件要求。Ollama 的 Qwen3.5 页面中，4B 与 9B 不同包的文件体积已经有明显差异；下载大小也不等于实际显存需求。[Ollama Qwen3.5 型号列表](https://ollama.com/library/qwen3.5)

MoE 的激活参数量主要影响单步计算，不能据此估算全部权重的存储。例如 Qwen3.6-35B-A3B 的语言模型有 35B 总参数、3B 激活参数，需要按总权重及实际卸载方式规划内存。[官方模型卡](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)

Gemma 4 12B/26B/31B 可作为跨模型家族的本地对照；MVP 先把 Qwen 档位跑通，后续增加实测兼容项。[Gemma 官方部署与内存说明](https://ai.google.dev/gemma/docs/core.md)

### 推理后端适配

- NVIDIA：通过支持 CUDA 的 Ollama、llama.cpp 或服务端引擎。
- AMD：按操作系统、具体 GPU 和引擎选择 ROCm/HIP 或 Vulkan。
- Intel：按引擎选择 Vulkan/SYCL 等可用后端。
- Apple：Metal；其他适配后端由其 HTTP 服务接入。
- CPU：llama.cpp 等，保留纯 API 工作模式。
- 多卡服务：通过 vLLM/SGLang 等已部署端点接入，不在小说业务层编写显卡算子。

以上为计划的兼容范围，不能等同于所有型号均已验证。驱动和模型架构支持以运行时文档和项目实测矩阵为准。[Ollama 硬件支持](https://docs.ollama.com/gpu)、[llama.cpp 官方支持后端](https://github.com/ggml-org/llama.cpp)

硬件探测只针对实际承载本地推理服务的主机。远端推理端点不能使用编辑器所在电脑的显卡信息来决定模型大小。显存读取失败时保留未知状态，允许手工设置。

## 5. 入门预算预设：每月 API 消耗低于 100 元

以下为每月 API 消耗低于 100 元的可选预设，用户可自行调整预算、模型和硬件配置：

- 主力：Qwen 3.7 Plus，固定快照优先，正文默认非思考模式。
- 辅助：Qwen 3.8 Flash；本地运行可用时，摘要和事实抽取改用本地 Qwen3.5-4B。
- 对照：在独立评测预算内试 DeepSeek Flash/Pro，不自动切到高价模型。
- 私有数据：仅在允许远端处理的项目中使用 API 回退。
- 本地辅助配置示例：6GB 显存设备从 4B Q4 开始试跑；9B 不作为 6GB 的默认配置。实际可用上下文与速度需在目标设备上验证。

### 不依赖本地模型也能估算预算

以下仅为月调用量示例，全部按前述北京区域原价，不计优惠。K=1,000 token，输入包含上下文，输出包含计费的推理 token；不能把 token 数直接当作成稿中文字数。

| 工作 | 月调用数 | 每次输入/输出 token | 模型 | 估算费用 |
| --- | ---: | --- | --- | ---: |
| 续写/改写候选 | 1,000 | 20K / 2K | Plus | 56 元 |
| 摘要/事实检查 | 1,000 | 6K / 1K | Flash | 7.5 元 |
| 场景规划/复核 | 100 | 40K / 3K | Plus | 10.4 元 |
| 合计 | 2,100 | — | — | 73.9 元 |

费用 = 输入 token × 输入单价 / 1,000,000 + 计费输出 token × 输出单价 / 1,000,000。

这是预算示例，不是用量预测或成稿质量承诺。反复重写、额外审阅、长推理、接口失败后的重新调用和更长 ContextPack 都会增加费用；项目必须记录实际 usage。被拒绝的生成候选同样产生费用。

### 预算控制设计

为了给 100 元限制留余量，该预设可把应用内远端预算定为 90 元：

~~~yaml
economy_preset:
  currency: CNY
  user_monthly_ceiling: 100
  app_remote_limit: 90
  warn_at: 70
  prefer_economy_at: 80
  month_timezone: Asia/Shanghai
  include_inflight_reservations: true
  remote_fallback: allowed_only_by_project_policy
  on_limit: use_local_if_available_else_pause_ai
~~~

这是拟实现的配置，不是已经生效的账户限额。发起请求前，在预算中预留该请求的最大估计费用，包含输入、最大计费输出和推理上限；结束后按实际 usage 结算，避免多个并发请求一起超额。缺少价格或无法界定计费上限时，不参与严格预算的自动调用。

按量价和汇率保存快照，重试计入累计费用；取消请求后也要结算已发生的费用。API 超时且计费未知时先保留预留金额。应用只能约束经自身发起的调用；同一密钥在其他程序产生的费用需账户侧配额共同约束。

达到限额后转本地或暂停 AI 功能，仍可手工编辑。不能靠省略一致性检查来省钱；未检查的正文保留未验证标记。

## 6. 文风学习的选择

第一版使用规则统计 + StyleProfile + 少量参考段落。分析和生成可以使用同一个模型；不要求为每位作者训练一个模型。

- 规则统计负责句长、段落长度、对白比例和词汇重复。
- 模型抽象视角、节奏、心理描写、意象和对话习惯，用户可编辑这些属性。
- 正文模型把 StyleProfile 应用于新场景，同时遵守 Canon。
- 验证使用未参加风格分析的参考片段及新剧情，检查可读性、内容泄漏和长串重复。

只有当这种方法持续无法满足需求时，再做授权语料的 LoRA/QLoRA 对照。推理显存表不能作为训练显存要求；训练还涉及激活值、优化器、序列长度和批量大小。文风适配权重不负责存储每本小说的实时事实。

## 7. 模型晋级评测

首轮先选 2—3 个候选，避免把 100 元全用在遍历模型目录上。评测集建议包括：

- 10 个中文续写场景：对白、动作、心理、转场等。
- 5 个保留剧情的局部改写。
- 5 个带来源的事实/时间线抽取。
- 5 个跨章矛盾及正常反转案例，同时测漏报和误报。
- 5 个文风约束场景，使用留出样本。

保持故事输入和输出要求一致，但允许供应商专用模板与合法采样参数。由作者盲选成稿；模型评分只作补充。增加一个多场景连续写作序列，修改前文后重新检查后文，测试错误累积和过期摘要。

指标包括事实准确率、矛盾检出与误报、作者接受率、改写后事实保持率、风格偏离、JSON 成功率、延迟、峰值内存和每次被接受修订的总成本。Q4、Q8 等量化版本视为不同候选，不能复用未量化模型的成绩。

当前尚未完成上述模型实测。通过评测的配置才进入“推荐”，其余保持“候选”。

## 8. 配置与能力探测

把逻辑模型角色与实际部署分开：

~~~text
ModelProfile
  role, model_id, resolved_version, prompt_version, output_budget

Deployment
  endpoint, adapter, runtime_version, quantization, model_digest
  host_id, accelerator_backend, available_memory, concurrency_limit

Capabilities
  stream, structured_mode, thinking_mode, tokenizer
  declared_context_limit, configured_context_limit, tested_context_limit

RoutingPolicy
  local_only, allowed_deployments, budget, timeout, retry_limit
~~~

JSON 字符串输出、受约束 JSON Schema、工具调用是不同能力，不用一个布尔值混淆。OpenAI 兼容端点也不意味着所有可选字段相同。

仅在首次配置或端点/模型/运行时变化后执行小规模试跑，费用计入预算：检查连接、短流式输出、所需 JSON 能力、预设上下文和取消行为。不要每次启动都重复付费探测，也不要尝试填满模型宣称的最大上下文。记录实际通过的上下文与并发；一个短请求不能证明 1M 上下文可用。

内存不足时降低并发、裁剪可选上下文、使用更小量化模型或部分 CPU 卸载。当前选中文本与必要 Canon 放不下时拆分任务或要求选择更合适的配置，不能静默丢弃事实。切换远端只在该项目允许且有预算时进行。

应用层因此可以保留同一套增量创作、一致性与文风工作流，由部署层适配不同模型和硬件。
