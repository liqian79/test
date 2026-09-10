## 模型部署教程 - 完整性、易用性评估

### 评分

- 完整性、易用性：8.6/10（A 等级）
- 通用质量（正确性）：8.5/10（A 等级）
- 综合分：8.56/10（A 等级）

### 文档信息

| 评测项 | 值 |
|--------|-----|
| 文档名称 | M:/Code/vllm-ascend/vllm-ascend/docs/source/tutorials/models/Qwen3.8-2.4T-A95B.md |
| 检测标准 | 模板符合度检查清单 + 正确性维度规则 |
| 检测方式 | 深度检测 |
| 检测维度 | 全部（完整性、易用性 + 正确性） |
| 模型 | Qwen3.8-2.4T-A95B |
| 加载 LLM 规则文件数 | 14（完整性、易用性）+ 10（正确性） |
| 检测开始时间 | 2026-09-09 00:20:04 |
| 检测结束时间 | 2026-09-09 00:20:04 |

### 必选章节得分

| 章节 | 层级 | 得分 | 状态 |
|------|------|------|------|
| 一、文档头部与标题 | H1 | 10.0/10 | Pass |
| 二、1 Introduction | H2 | 10.0/10 | Pass |
| 三、2 Supported Features | H2 | 10.0/10 | Pass |
| 四、3.1 Model Weight | H3 | 7.5/10 | Partial |
| 五、4 Installation | H2 | 6.5/10 | Partial |
| 六、5 Online Service Deployment | H2 | 7.5/10 | Partial |
| 七、6 Functional Verification | H2 | 10.0/10 | Pass |
| 八、7 Accuracy Evaluation | H2 | 9.0/10 | Pass |
| 九、8 Performance Evaluation | H2 | 6.5/10 | Partial |
| 十、9 Performance Tuning | H2 | 9.0/10 | Pass |
| 十一、10 FAQ | H2 | 9.0/10 | Pass |
| 十二、参数约束符合度 | 跨章节 | 7.0/10 | Partial |
| 十三、语法约束符合度 | 跨章节 | 10.0/10 | Pass |
| 十四、章节结构 | — | 7.5/10 | Partial |
| **必选平均分** | | **8.54（14 章均值 + 可选加分 0.1 = 8.6）** | — |

### 可选章节加分

| 章节 | 加分 |
|------|------|
| 十五、3.2 多节点通信验证 | +0.1（含链接，指向 installation.md#installation-multi-node-interconnect） |
| 十六、5.3 特殊部署模式 | +0.0（无 5.3 章节） |
| **加分上限** | **+0.3** |

### 正确性维度得分

| H2 分类 | 得分 | 问题数 | 状态 |
|---------|------|--------|------|
| 单位符号使用错误 | 8.5 | 1 | Partial |
| 不符合安全合规要求 | 10.0 | 0 | Pass |
| 标点符号错误 | 9.0 | 1 | Pass |
| 链接错误或失效 | 9.5 | 0 | Pass |
| 命令/文件名/路径/参数/示例等错误 | 7.5 | 1 | Partial |
| 英文拼写错误 | 10.0 | 0 | Pass |
| 中文错别字词 | 10.0 | 0 | Pass |
| 与产品实现不一致 | 8.0 | 2 | Partial |
| 代码样例无法执行 | 8.0 | 1 | Partial |
| 写作不规范 | 7.0 | 4 | Partial |
| **正确性平均分** | **8.75（取 8.5）** | **10** | — |

### 问题统计

| 级别 | 数量 |
|------|------|
| 致命 | 0 |
| 严重 | 12 |
| 一般 | 7 |
| **总计** | **19** |
| AI 可修复 | 15 |
| 需人工修复 | 4 |
| 跨维度去重 | 2 |

### Top 5 修复

1. [4 Installation - 5.4 echo 输出] (line 108): 安装验证命令缺少完整 echo 输出示例。修复：在验证命令下方添加预期终端输出如 `vllm and vllm_ascend are ready`。
2. [8 Performance Evaluation - 9.2 echo 输出] (line 588): 性能评测命令缺少完整 echo 输出示例。修复：在命令下方添加性能评测预期输出结果。
3. [正确性 - 命令/参数错误] (line 590): `vllm bench serve --model Qwen/Qwen3.8-2.4T-A95B` 应改为 `--model qwen3.8` 以匹配服务端 `--served-model-name`，否则返回 404。
4. [3.1 Model Weight - 4.2 双源链接] (line 34): 仅提供 ModelScope 下载链接，缺少 HuggingFace 链接。修复：为每个权重版本添加 HuggingFace 下载链接。
5. [5 Online Service - 6.2 模型路径占位] (line 190): 使用 `<QWEN3_8_MODEL_PATH>` 而非标准 `<YOUR_MODEL_PATH>` 占位符。修复：统一使用 `<YOUR_MODEL_PATH>` 或说明自定义占位符约定。

### 缺失必需要素

- [x] 1 Introduction（模型介绍/版本/适配状态）— 完整
- [x] 2 Supported Features（支持表/交叉引用）— 完整（交叉引用链接）
- [ ] 3.1 Model Weight（双源链接/路径说明/占位符）— 缺 HuggingFace 链接、缺标准占位符说明
- [ ] 4 Installation（步骤/版本占位/验证/echo/Tabs）— 缺版本占位符、缺 echo 输出
- [ ] 5 Online Service Deployment（排障/模型路径/Tabs/5.1/5.2/锚点）— 5.1 命名不符、5.2 仅有占位文本、占位符不规范
- [x] 6 Functional Verification（接口调用/预期结果）— 完整
- [x] 7 Accuracy Evaluation（方法或链接）— 完整
- [ ] 8 Performance Evaluation（命令/echo）— 缺 echo 输出
- [x] 9 Performance Tuning（9.1 推荐配置/9.2 调优）— 完整
- [x] 10 FAQ（公共FAQ引用/现象+解决措施/具体命令）— 完整
- [ ] 参数约束符合度（参数使用/JSON 子字段/多节点一致）— enable_fused_mc2 约束待确认
- [x] 语法约束符合度（Tabs/占位符/Note/Jinja/锚点）— 完整
- [ ] H2 齐全性 / H3 齐全性 / 层级命名 / 9.2 二选一 — 5.1 命名不符、9.2.1/9.2.2 命名互换

### 问题清单（按行号升序）

| 序号 | 维度 | 行号 | 原文片段 | 问题描述 | 修复建议 | H2 分类 | 严重程度 | 可自动修复 |
|------|------|------|----------|----------|----------|---------|----------|------------|
| 1 | 正确性 | 33 | approximately 4.89 TB of storage... | TB 与 TiB 单位混用不一致 | 统一使用 TB 或 TiB 单位 | 单位符号使用错误 | 一般 | AI 可修复 |
| 2 | 完整性、易用性 | 34 | [Download model weight](https://www.modelscope.cn/...) | 仅提供 ModelScope 链接，缺 HuggingFace | 为每个权重版本添加 HuggingFace 下载链接 | 3.1 Model Weight | 严重 | AI 可修复 |
| 3 | 完整性、易用性 | 47 | The checkpoint and tokenizer directories... | 未明确提及 <YOUR_MODEL_PATH> 占位符约定 | 明确提及 <YOUR_MODEL_PATH> 占位符 | 3.1 Model Weight | 严重 | AI 可修复 |
| 4 | 完整性、易用性 | 72 | export IMAGE=quay.io/ascend/vllm-ascend:qwen3.8-a3 | 使用固定版本值而非占位符 | 使用 {{ vllm_ascend_version }} 占位符 | 4 Installation | 严重 | AI 可修复 |
| 5 | 完整性、易用性 | 108 | After entering the container, verify... | 缺少完整 echo 输出示例 | 添加验证命令的预期终端输出 | 4 Installation | 严重 | AI 可修复 |
| 6 | 完整性、易用性 | 163 | ### 5.1 Multi-Node Deployment | 5.1 命名为 Multi-Node 而非 Single-Node | 改为 Single-Node 或说明仅支持多节点 | 5 Online Service Deployment | 严重 | AI 可修复 |
| 7 | 完整性、易用性 | 163 | ### 5.1 Multi-Node Deployment | 必选 H3 5.1 命名不符模板 | 改为 Single-Node | 章节结构 | 严重 | AI 可修复 |
| 8 | 完整性、易用性 | 190 | export MODEL_PATH=<QWEN3_8_MODEL_PATH> | 使用非标准占位符 <QWEN3_8_MODEL_PATH> | 使用标准 <YOUR_MODEL_PATH> 占位符 | 5 Online Service Deployment | 严重 | AI 可修复 |
| 9 | 正确性 | 230 | --speculative-config '{「method」:「qwen3_5_mtp」...}' | qwen3_5_mtp 方法不在标准方法列表中 | 确认是否为新增方法或更新约束文档 | 与产品实现不一致 | 严重 | 需人工 |
| 10 | 完整性、易用性 | 232 | --additional-config {「enable_fused_mc2」:1} | enable_fused_mc2 在 EP64+MTP 场景可能违反约束 | 确认兼容性或更换优化方案 | 参数约束符合度 | 严重 | 需人工 |
| 11 | 正确性 | 321 | Common Issues Tip: If a worker exits... | 非标准行内文本格式，应使用 admonition | 改为 !!! tip admonition 格式 | 写作不规范 | 一般 | AI 可修复 |
| 12 | 完整性、易用性 | 502 | Detailed commands...will be provided in a future update | 5.2 PD 分离仅有占位文本 | 补充启动命令、配置、验证方法 | 5 Online Service Deployment | 严重 | 需人工 |
| 13 | 正确性 | 530 | Each response begins with reasoning wrapped in `⏎...` | 使用 Unicode 返回符号 ⏎ 代替 think 标签 | 替换为正确的 thinking 块标签 | 写作不规范 | 严重 | AI 可修复 |
| 14 | 完整性、易用性 | 588 | vllm bench serve --model Qwen/Qwen3.8-2.4T-A95B... | 缺少完整 echo 输出示例 | 添加性能评测预期输出结果 | 8 Performance Evaluation | 严重 | AI 可修复 |
| 15 | 正确性 | 590 | --model Qwen/Qwen3.8-2.4T-A95B \ | --model 应为 qwen3.8 匹配 --served-model-name | 将 --model 改为 qwen3.8 | 命令/参数错误 | 严重 | AI 可修复 |
| 16 | 正确性 | 590 | --model Qwen/Qwen3.8-2.4T-A95B \ | 代码样例因参数不匹配无法执行 | 将 --model 改为 qwen3.8 | 代码样例无法执行 | 严重 | AI 可修复 |
| 17 | 正确性 | 617 | \| *Total NPUs \| | 未闭合的 Markdown 强调标记 | 改为 *Total NPUs* 或 **Total NPUs** | 标点符号错误 | 一般 | AI 可修复 |
| 18 | 正确性 | 617 | \| *Total NPUs \| | 未闭合的 Markdown 强调标记导致渲染异常 | 改为 *Total NPUs* 或 **Total NPUs** | 写作不规范 | 一般 | AI 可修复 |
| 19 | 完整性、易用性 | 634 | #### 9.2.1 General Tuning Reference | 9.2.1/9.2.2 命名与模板互换 | 9.2.1 改为 Model-Specific，9.2.2 改为 General Tuning | 章节结构 | 一般 | AI 可修复 |

### H2 得分明细

| 维度 | H2 分类 | H3 分类 | 问题数 | 得分 |
|------|---------|---------|--------|------|
| 完整性、易用性 | 文档头部与标题 | （全部） | 0 | 10.0 |
| 完整性、易用性 | 1 Introduction | （全部） | 0 | 10.0 |
| 完整性、易用性 | 2 Supported Features | （全部） | 0 | 10.0 |
| 完整性、易用性 | 3.1 Model Weight | 4.2 双源下载链接 | 1 | 7.5 |
| 完整性、易用性 | 3.1 Model Weight | 4.4 路径说明与占位符 | 1 | — |
| 完整性、易用性 | 4 Installation | 5.2 版本号占位符规范 | 1 | 6.5 |
| 完整性、易用性 | 4 Installation | 5.4 完整 echo 输出示例 | 1 | — |
| 完整性、易用性 | 5 Online Service Deployment | 6.2 模型路径占位与注释 | 1 | 7.5 |
| 完整性、易用性 | 5 Online Service Deployment | 6.4 5.1 单机部署内容 | 1 | — |
| 完整性、易用性 | 5 Online Service Deployment | 6.5 5.2 多机 PD 分离内容 | 1 | — |
| 完整性、易用性 | 6 Functional Verification | （全部） | 0 | 10.0 |
| 完整性、易用性 | 7 Accuracy Evaluation | （全部） | 0 | 9.0 |
| 完整性、易用性 | 8 Performance Evaluation | 9.2 完整 echo 输出 | 1 | 6.5 |
| 完整性、易用性 | 9 Performance Tuning | （全部） | 0 | 9.0 |
| 完整性、易用性 | 10 FAQ | （全部） | 0 | 9.0 |
| 完整性、易用性 | 参数约束符合度 | P.1 参数使用符合约束 | 1 | 7.0 |
| 完整性、易用性 | 语法约束符合度 | （全部） | 0 | 10.0 |
| 完整性、易用性 | 章节结构 | S.2 H3 子章节齐全性 | 1 | 7.5 |
| 完整性、易用性 | 章节结构 | S.3 标题层级与命名规范 | 1 | — |
| 正确性 | 单位符号使用错误 | 存储单位一致性 | 1 | 8.5 |
| 正确性 | 不符合安全合规要求 | （全部） | 0 | 10.0 |
| 正确性 | 标点符号错误 | Markdown 标记闭合 | 1 | 9.0 |
| 正确性 | 链接错误或失效 | （全部） | 0 | 9.5 |
| 正确性 | 命令/文件名/路径/参数/示例等错误 | 参数一致性 | 1 | 7.5 |
| 正确性 | 英文拼写错误 | （全部） | 0 | 10.0 |
| 正确性 | 中文错别字词 | （全部） | 0 | 10.0 |
| 正确性 | 与产品实现不一致 | 参数约束一致性 | 1 | 8.0 |
| 正确性 | 与产品实现不一致 | 投机解码方法一致性 | 1 | — |
| 正确性 | 代码样例无法执行 | 参数一致性导致执行失败 | 1 | 8.0 |
| 正确性 | 写作不规范 | Markdown 语法规范 | 1 | 7.0 |
| 正确性 | 写作不规范 | Admonition 格式规范 | 1 | — |
| 正确性 | 写作不规范 | 特殊符号使用 | 1 | — |
| 正确性 | 写作不规范 | 章节命名规范 | 1 | — |

### 综合评分

| 维度 | 得分 | 权重 | 加权得分 |
|------|------|------|----------|
| 完整性、易用性 | 8.6 | 60% | 5.16 |
| 正确性 | 8.5 | 40% | 3.40 |
| **综合分** | | | **8.56** |
| **等级** | | | **A** |
