## 模型部署教程 - 完整性、易用性评估

### 评分
- 完整性、易用性：6.2/10（合格等级）
- 通用质量（正确性）：8.1/10（良好等级）
- 综合分：7.0/10（良好等级）

### 必选章节得分

| 章节 | 层级 | 得分 | 状态 |
|------|------|------|------|
| 一、文档头部与标题 | H1 | 9.0/10 | Pass |
| 二、1 Introduction | H2 | 5.5/10 | Partial |
| 三、2 Supported Features | H2 | 7.0/10 | Pass |
| 四、3.1 Model Weight | H3 | 5.5/10 | Partial |
| 五、4 Installation | H2 | 4.0/10 | Fail |
| 六、5 Online Service Deployment | H2 | 5.0/10 | Partial |
| 七、6 Functional Verification | H2 | 7.0/10 | Pass |
| 八、7 Accuracy Evaluation | H2 | 9.0/10 | Pass |
| 九、8 Performance Evaluation | H2 | 5.5/10 | Partial |
| 十、9 Performance Tuning | H2 | 6.0/10 | Partial |
| 十一、10 FAQ | H2 | 6.0/10 | Partial |
| 十二、参数约束符合度 | 跨章节 | 7.5/10 | Pass |
| 十三、语法约束符合度 | 跨章节 | 5.5/10 | Partial |
| 十四、章节结构 | — | 3.5/10 | Fail |
| **必选平均分** | | **6.14（14 章均值 + 可选加分 ≤ 10）** | — |

### 可选章节加分

| 章节 | 加分 |
|------|------|
| 十五、3.2 多节点通信验证 | +0.1（含链接，第153行） |
| 十六、5.3 特殊部署模式 | +0.0 |
| **加分上限** | **+0.3** |

### 正确性维度得分

| H2 项 | 得分 | 问题数 |
|-------|------|--------|
| 1 单位符号 | 9.0 | 0 |
| 2 安全合规 | 9.5 | 0 |
| 3 标点符号 | 6.0 | 1 |
| 4 链接 | 9.0 | 0 |
| 5 命令/路径 | 7.5 | 1 |
| 6 英文拼写 | 9.5 | 0 |
| 7 中文错别字 | 9.5 | 0 |
| 8 产品一致性 | 7.5 | 1 |
| 9 代码执行 | 7.0 | 1 |
| 10 写作不规范 | 6.0 | 1 |
| **正确性平均分** | **8.05** | **5** |

### Top 5 修复

1. [章节结构-S.2]: Installation 缺少 4.1 Docker Image / 4.2 Source Code 两个必选 H3 子章节（第25-104行），3.1 Model Weight 未作为 H3 层级。修复：在 4 Installation 下拆分 4.1/4.2 H3，在 3 Prerequisites 下添加 3.1 Model Weight H3。
2. [5 Online Service Deployment-6.2]: 部署命令模型路径未使用 `<YOUR_MODEL_PATH>` 占位符，单机部署直接写死路径 `/root/.cache/modelscope/hub/models/vllm-ascend/GLM-5.2-w4a8c8`（第122行等）。修复：统一使用 `<YOUR_MODEL_PATH>` 占位符并附注释。
3. [4 Installation-5.3/5.4]: Installation 缺少验证命令（如 `docker ps`）与预期状态、完整 echo 输出示例（第25-104行）。修复：添加验证命令及 echo 输出。
4. [标点符号/代码执行]: Functional Verification 的 JSON 响应示例格式严重错误（第1522-1551行），键名缺少下划线、引号格式错误、拼写错误（ky transfer params、system fingerprint）。修复：修正为合法 JSON。
5. [章节结构-S.1]: H2 命名不一致：「3 Model Weight」应为「3 Prerequisites」，「5 Deployment」应为「5 Online Service Deployment」（第15行、第108行）。修复：按模板规范重命名 H2 章节。

### 缺失必需要素

- [x] 1 Introduction（缺少 vLLM-Ascend 版本与支持状态）
- [ ] 2 Supported Features（有交叉引用链接，Pass）
- [x] 3.1 Model Weight（双源链接不完整、未用表格、缺少 `<YOUR_MODEL_PATH>` 占位符）
- [x] 4 Installation（缺少验证命令/echo 输出、缺少 4.1/4.2 H3 子章节）
- [x] 5 Online Service Deployment（模型路径未用占位符、缺少锚点、H2 命名不符）
- [ ] 6 Functional Verification（接口调用与预期结果基本完整，Pass）
- [ ] 7 Accuracy Evaluation（有 AISBench 链接与结果，Pass）
- [x] 8 Performance Evaluation（缺少基础命令示例与 echo 输出，仅链接）
- [x] 9 Performance Tuning（9.1 未声明非全局最优、9.2 缺少 H4 子章节）
- [x] 10 FAQ（缺少公共 FAQ 引用）
- [x] 参数约束符合度（enable_fused_mc2 与 multistream_overlap_shared_expert 互斥约束违反）
- [x] 语法约束符合度（Jinja 占位符未用 raw 包裹）
- [x] H2 齐全性 / H3 齐全性 / 层级命名 / 9.2 二选一

### 评估说明

**文档概况**：GLM5.2.md 为 GLM-5.2 模型部署教程，共 1623 行，涵盖 4 种权重版本（BF16/w8a8/w8a8c8/w4a8c8）在 Atlas 800 A3/A2 上的单机、多机及 PD 分离部署，包含 1M 上下文配置。文档部署内容详实，PD 分离参数描述完整。

**主要问题**：
1. **章节结构**：H2 命名与模板不符（3 Model Weight → 3 Prerequisites，5 Deployment → 5 Online Service Deployment），缺少 4.1/4.2 H3 子章节，3.1 未作为 H3 层级，9.2 缺少 H4 子章节。
2. **Installation**：缺少验证命令、echo 输出和 H3 子章节拆分。
3. **路径占位符**：全程未使用 `<YOUR_MODEL_PATH>`，部分硬编码路径。
4. **JSON 示例错误**：Functional Verification 的 JSON 响应示例格式严重错误，无法解析。
5. **产品一致性**：`--served-model-name` 在不同场景取值不一致（glm-5 vs glm-52）。

**评分计算**：
- 完整性、易用性 = 14 章均值 6.14 + 可选加分 0.1 = 6.24 ≈ 6.2
- 正确性 = 10 项均值 8.05 ≈ 8.1
- 综合分 = 6.2 × 60% + 8.1 × 40% = 3.72 + 3.24 = 6.96 ≈ 7.0（良好）
- 总问题数：20（完整性、易用性 15 + 正确性 5）
