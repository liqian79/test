# GLM5.3 模型部署教程评测报告

> 文档：`GLM5.3.md`
> 评测时间：2026-09-09 00:20:00
> 评测维度：完整性、易用性 + 正确性

---

## 一、评测总览

| 维度 | 得分 | 等级 | 问题数 |
|------|------|------|--------|
| 完整性、易用性 | 5.8/10 | 合格 | 18 |
| 正确性 | 6.8/10 | 合格 | 19 |
| **综合分** | **6.2/10** | **合格** | **37** |

**综合分计算**：完整性易用性 5.8 × 60% + 正确性 6.8 × 40% = 6.2

**可选章节加分**：3.2 多节点通信验证含链接 +0.1（已计入完整性易用性分）

**关键问题概要**：
- 致命问题 3 项：缺少 `<YOUR_MODEL_PATH>` 占位符（部署命令路径写死）、缺失 9 Performance Tuning H2、缺失 9.1/9.2 必选 H3
- 严重问题 10 项：缺少 HuggingFace 双源链接、无安装验证命令/echo、无锚点、5.1 命名不符、参数跨节点不一致、无性能评测命令/echo、FAQ 无公共引用
- 一般问题 7 项：权重表格式、PD 分离缺内容、无成功判据、预期输出乱码、FAQ 编号等

---

## 二、易用性评估问题清单

> 按行号升序排列

| 序号 | 行号 | H2 | H3 | 问题描述 | 修复建议 | 规则ID | 严重程度 | 可修复 |
|------|------|-----|-----|---------|---------|--------|---------|--------|
| 1 | 27 | 四、3.1 Model Weight | 4.2 双源下载链接 | Model Weight 章节仅提供 ModelScope 下载链接，缺少 HuggingFace 下载链接 | 在 ModelScope 链接旁添加 HuggingFace 下载链接 | 4.2 | 严重 S-4.2 | ✅ |
| 2 | 27 | 四、3.1 Model Weight | 4.3 权重版本与硬件需求表 | 权重版本与硬件需求以列表形式呈现，未使用表格格式 | 转换为表格，列含权重版本、硬件需求、下载链接 | 4.3 | 一般 N-4.3 | ✅ |
| 3 | 30 | 四、3.1 Model Weight | 4.4 路径说明与占位 | 路径说明提及共享目录但未提及 `<YOUR_MODEL_PATH>` 占位符约定 | 添加占位符约定说明，提醒用户记录路径并在后续部署命令中使用 | 4.4 | 严重 S-4.4 | ✅ |
| 4 | 36 | 五、4 Installation | 5.3 验证命令与预期状态 | Installation 章节缺少验证命令（如 `docker ps`）及预期状态 | 添加 `docker ps` 验证命令及预期输出状态 | 5.3 | 严重 S-5.3 | ✅ |
| 5 | 36 | 五、4 Installation | 5.4 完整 echo 输出示例 | Installation 章节缺少完整 echo 输出示例 | 添加安装后容器运行状态的完整 echo 输出 | 5.4 | 严重 S-5.4 | ✅ |
| 6 | 126 | 六、5 Online Service Deployment | 6.6 5 章节锚点 | H2 标题后缺少 `{: #5-online-service-deployment }` 锚点声明 | 在 H2 标题行末添加锚点声明 | 6.6 | 严重 S-6.6 | ✅ |
| 7 | 134 | 六、5 Online Service Deployment | S.3 标题层级与命名规范 | 5.1 命名为「Multi-node Deployment」而非模板要求的「Single-Node」，无单机部署内容 | 重命名为「Single-Node」并添加单机部署内容，或按模板重构 H3 | S.3 | 严重 S-S.3 | ✅ |
| 8 | 172 | 六、5 Online Service Deployment | 6.2 模型路径占位与注释 | 部署命令使用硬编码路径 `/root/.cache/modelscope/hub/models/vllm-ascend/GLM-5.3-w8a8c8`，未使用 `<YOUR_MODEL_PATH>` 占位符 | 替换为 `<YOUR_MODEL_PATH>` 占位符并添加注释提醒用户替换为 3.1 记录的路径 | 6.2 | 致命 F-6.2 | ✅ |
| 9 | 193 | 十二、参数约束符合度 | P.4 多节点/PD 参数组合一致 | `--gpu-memory-utilization` 在 node 0 设为 0.90，node 1 设为 0.92（line 245），多节点部署各节点须保持一致 | 统一所有节点的 `--gpu-memory-utilization` 值 | P.4 | 严重 S-P.4 | ✅ |
| 10 | 401 | 六、5 Online Service Deployment | 6.5 5.2 多机 PD 分离内容 | 5.2 Prefill-Decode Disaggregation 仅提供 GLM-5.2 链接，无实际部署内容、架构说明或性能指标 | 添加 PD 分离架构描述、启动流程、关键配置和验证说明 | 6.5 | 一般 N-6.5 | ✅ |
| 11 | 424 | 七、6 Functional Verification | 7.2 预期结果与成功判据 | 未提供明确的成功判据（如 HTTP 200、JSON 含 choices 字段） | 添加明确成功判据说明 | 7.2 | 一般 N-7.2 | ✅ |
| 12 | 427 | 七、6 Functional Verification | 7.3 预期结果描述 | 预期输出文本损坏/截断，含「contarning」「stogically」「function_call」:oning」等错误 | 替换为正确完整的 API 响应示例 | 7.3 | 一般 N-7.3 | ✅ |
| 13 | 449 | 九、8 Performance Evaluation | 9.1 性能评测命令 | 性能评测章节仅提供外部链接，无内联命令示例 | 添加内联性能评测命令示例 | 9.1 | 严重 S-9.1 | ✅ |
| 14 | 449 | 九、8 Performance Evaluation | 9.2 完整 echo 输出 | 性能评测章节缺少完整 echo 输出示例 | 添加完整 echo 输出展示基准测试结果 | 9.2 | 严重 S-9.2 | ✅ |
| 15 | 459 | 十四、章节结构 | S.1 H2 章节齐全性 | 缺少必选 H2「9 Performance Tuning」；文档将 FAQ 编为 9、Declaration 编为 10 | 添加「## 9 Performance Tuning」章节含 9.1 和 9.2，将 FAQ 重编号为 10 | S.1 | 致命 F-S.1 | ✅ |
| 16 | 459 | 十一、10 FAQ | 11.1 公共 FAQ 引用 | FAQ 章节开头缺少指向公共 FAQ 的引用说明 | 在 FAQ 章节开头添加公共 FAQ 引用 | 11.1 | 严重 S-11.1 | ✅ |
| 17 | 459 | 十四、章节结构 | S.3 标题层级与命名规范 | FAQ 编为「9 FAQ」而非模板要求的「10 FAQ」；额外的「10 Declaration」不在模板结构中 | 将 FAQ 重编号为「## 10 FAQ」，将 Declaration 内容移入 FAQ 或 Introduction | S.3 | 一般 N-S.3 | ✅ |
| 18 | 471 | 十四、章节结构 | S.2 H3 子章节齐全性 | 缺少必选 H3「9.1 Recommended Configurations」和「9.2 Tuning Guidelines」 | 在 Performance Tuning 章节下添加 9.1 和 9.2 子章节 | S.2 | 致命 F-S.2 | ✅ |

---

## 三、易用性评估 H2 得分明细

| 序号 | 章节 | 层级 | 得分 | 问题数 | 状态 |
|------|------|------|------|--------|------|
| 1 | 一、文档头部与标题 | H1 | 9.5/10 | 0 | Pass |
| 2 | 二、1 Introduction | H2 | 8.5/10 | 0 | Pass |
| 3 | 三、2 Supported Features | H2 | 8.5/10 | 0 | Pass |
| 4 | 四、3.1 Model Weight | H3 | 4.0/10 | 3 | Fail |
| 5 | 五、4 Installation | H2 | 5.0/10 | 2 | Partial |
| 6 | 六、5 Online Service Deployment | H2 | 3.5/10 | 4 | Fail |
| 7 | 七、6 Functional Verification | H2 | 6.5/10 | 2 | Partial |
| 8 | 八、7 Accuracy Evaluation | H2 | 8.5/10 | 0 | Pass |
| 9 | 九、8 Performance Evaluation | H2 | 3.5/10 | 2 | Fail |
| 10 | 十、9 Performance Tuning | H2 | 0.0/10 | 0 | Fail (缺失) |
| 11 | 十一、10 FAQ | H2 | 5.5/10 | 1 | Partial |
| 12 | 十二、参数约束符合度 | 跨章节 | 6.5/10 | 1 | Partial |
| 13 | 十三、语法约束符合度 | 跨章节 | 7.5/10 | 0 | Pass |
| 14 | 十四、章节结构 | — | 2.5/10 | 3 | Fail |
| — | **必选平均** | | **5.68** | **18** | — |
| — | 可选加分（3.2 通信验证） | | **+0.1** | — | — |
| — | **完整性易用性总分** | | **5.8/10** | **18** | **合格** |

### 可选章节加分

| 章节 | 加分 | 说明 |
|------|------|------|
| 十五、3.2 多节点通信验证 | +0.1 | 含多节点通信验证链接 |
| 十六、5.3 特殊部署模式 | +0.0 | 无 5.3 章节 |
| **加分合计** | **+0.1** | |

### 正确性 H2 得分明细

| 序号 | H2 | 得分 | 问题数 | 状态 |
|------|-----|------|--------|------|
| 1 | 单位符号 | 5.5/10 | 2 | Partial |
| 2 | 安全合规 | 10.0/10 | 0 | Pass |
| 3 | 标点符号 | 8.5/10 | 1 | Pass |
| 4 | 链接 | 7.0/10 | 1 | Pass |
| 5 | 命令/路径 | 4.5/10 | 5 | Fail |
| 6 | 英文拼写 | 3.5/10 | 5 | Fail |
| 7 | 中文错别字 | 10.0/10 | 0 | Pass |
| 8 | 产品一致性 | 5.5/10 | 1 | Partial |
| 9 | 代码执行 | 7.0/10 | 1 | Pass |
| 10 | 写作不规范 | 6.0/10 | 3 | Partial |
| — | **正确性平均** | **6.8/10** | **19** | **合格** |

---

## 附：Top 5 优先修复项

1. **[致命]** 添加 `## 9 Performance Tuning` 章节（含 9.1 Recommended Configurations 和 9.2 Tuning Guidelines），并将 FAQ 重编号为 10（行 459/471）
2. **[致命]** 将所有部署命令中的硬编码模型路径替换为 `<YOUR_MODEL_PATH>` 占位符（行 172/225/289/347）
3. **[严重]** 在 3.1 Model Weight 章节添加 HuggingFace 下载链接并改为表格格式，补充 `<YOUR_MODEL_PATH>` 占位符约定（行 27/30）
4. **[严重]** 在 4 Installation 章节添加验证命令和完整 echo 输出（行 36）
5. **[严重]** 修正预期输出中的拼写错误和乱码内容，统一 A3 硬件规格描述（行 27/427）
