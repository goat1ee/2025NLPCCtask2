# DUFL2025 作文切题度评分与相关性评论生成系统

## 1. 项目背景

本项目是为 **DUFL2025 (NLPCC2025 Evaluation of Essay On-Topic Graded Comments - EOTGC)** 评测任务提供的解决方案。该任务旨在评估自动化系统在两个相关子任务上的能力：

1.  **Track 1**: 对中小学生作文进行切题度自动评分，输出“优秀”、“较好”、“一般”、“合格”、“不合格”五个等级的分类。
2.  **Track 2**: 基于 Track 1 的评分结果，生成一段仅聚焦于作文“切题度”和“中心思想”的相关性评论，且长度严格控制在 120-180 字符之间。

任务目标：构建一个能够准确评估作文切题度，并能生成高质量、符合特定约束的相关性评论的自动化系统。

## 2. 解决方案概述

本项目针对两个赛道分别设计了基于大型语言模型（LLM）的多智能体（Multi-Agent）解决方案，并结合了其他技术手段进行优化。

### 2.1 Track 1: 作文切题度自动评分

采用**混合架构**，融合了**基于 LLM 的多智能体协作系统**与一个**辅助性的 BERT 分类模块**：

*   **LLM 多智能体分析框架**:
    *   **模型**: `deepseekv3`
    *   **流程**: 包含 7 个功能专一的 LLM 智能体（Agent 0-6），通过精心设计的提示工程引导，按序执行作文类型识别、题目要求解析、主旨提取、内容-任务符合度校验、素材-主题契合度审核、综合裁决（初步分类）、分类结果提取。
    *   **Few-Shot**: 部分智能体（Agent 4, 5）利用 `samples.json` 中的样本进行引导。
*   **BERT 辅助分类模块**:
    *   **模型**: 基于 `bert-base-chinese` 的增强型 BERT 架构 (`EnhancedBertEssayModel`)。
    *   **训练**: 采用**无监督**对比学习（InfoNCE 风格），利用测试集数据 (`test_data.json`) 优化模型特征表示能力，保存为 `bert_essay_model.pt`。
    *   **作用**: 提供独立的切题度预测分类。
*   **混合决策与冲突仲裁**:
    *   主要判定依据 LLM 框架结果。
    *   当 LLM 与 BERT 结果不一致时，启动专门的 LLM **冲突仲裁智能体 (Arbiter Agent)**（利用 Few-Shot 样本）进行最终裁决。
    *   根据一致性、仲裁结果或失败回退逻辑确定最终分类。

### 2.2 Track 2: 相关性评论自动生成

设计了一个**多阶段 LLM 智能体流水线**，严格控制输出的**内容焦点（切题度与中心思想）**和**长度（120-180 字符）**：

*   **模型**: `qwen-max`
*   **输入**: 依赖 `test_data.json` 和 **Track 1 的最终分类结果 (`DUFL2025_track1.json`)**。
*   **流程**: 包含多个分析智能体（Agent 0-2, A-D）进行类型识别、题目分析、主旨提取、年级适配性评估、技巧/素材/结构对切题度影响的分析。
*   **初步评语生成 (Agent 4)**: 结合所有分析报告和 Track 1 分类，生成聚焦切题度和中心思想的草稿，目标长度 190-220 字符（内部目标，非最终输出）。利用 `samples.json` 进行 Few-Shot 引导。
*   **合规性检查与精炼 (Agent 5)**:
    *   **内容约束**: 强制聚焦“切题度/中心思想”及其影响因素，移除无关评价。
    *   **格式规范**: 输出纯文本。
    *   **长度硬控**: 严格确保最终字符数在 **120-180 字符之间**（通过 `clean_comment` 函数和流程逻辑强制实现）。
    *   **一致性复核**: 与 Track 1 分类基调一致。
    *   **润色与总结句**: 专业化语言，并为“优秀”至“合格”添加规范总结句（不含建议）。利用 `samples.json` 进行 Few-Shot 引导。
*   **容错**: 包含重试机制和模板化的**备用评语**（强制调整长度至 120-180）。
*   **缓存**: 使用 `processed_comments_cache_track2.json` 缓存有效评论。

## 3. 主要文件与脚本

*   `DUFL2025_Method_Report（最高版本）.md`: 详细的方法报告（中英文），解释了两个赛道的技术方案和实现细节。
*   `track1.ipynb`: **Track 1 解决方案**的 Jupyter Notebook 实现代码。负责执行作文切题度评分流程。
*   `track2.ipynb`: **Track 2 解决方案**的 Jupyter Notebook 实现代码。负责执行相关性评论生成流程。
*   `DUFL2025_track1.json`: **Track 1 的最终输出**，包含每篇作文的切题度分类。
*   `DUFL2025_track2.json`: **Track 2 的最终输出**，包含每篇作文的相关性评论（120-180 字符）。
*   `test_data.json`: 官方提供的测试集作文数据。
*   `samples.json`: 用于 Few-Shot 引导的样本数据（包含作文、分类、评论）。
*   `Task Guideline.md`: 官方评测任务指南。
*   `result.xlsx`: 包含中间结果、手动分析或结果汇总的 Excel 文件。

## 4. 使用流程

1.  **环境设置**:
    *   安装必要的 Python 库 (需要 `torch`, `transformers`, `requests`, `numpy`, `tqdm` 等)。
    *   配置 LLM API 密钥：在 `track1.ipynb` 和 `track2.ipynb` 中设置有效的 API Keys (如 `deepseekv3` 和 `qwen-max` 的密钥) 和 API URL。
2.  **数据准备**:
    *   将 `test_data.json` 和 `samples.json` 放置在代码可以访问的路径下（默认为当前目录或 `data/` 目录，根据代码具体实现）。
3.  **运行 Track 1**:
    *   执行 `track1.ipynb` 中的代码。
    *   这将处理 `test_data.json`，进行 LLM 调用和 BERT 辅助预测（需要训练或加载 `bert_essay_model.pt`），最终生成 `DUFL2025_track1.json` 文件。
4.  **运行 Track 2**:
    *   执行 `track2.ipynb` 中的代码。
    *   脚本将读取 `test_data.json` 和 **上一步生成的 `DUFL2025_track1.json`**。
    *   执行 LLM 智能体流水线，生成评论，并进行合规性检查和长度控制。
    *   最终生成 `DUFL2025_track2.json` 文件。

## 5. 实验结果

本项目在 DUFL2025 评测中取得了以下成绩：

| Rank | System Name | Track1 Score | Track2 Score | Total Score |
| :--- | :---------- | :----------- | :----------- | :---------- |
| 4    | DUFL2025    | 0.7784       | 0.6235       | 0.6854      |

*注：Track 1 和 Track 2 的分数计算方式请参考官方 `Task Guideline.md`。*

## 6. 硬件和软件环境

*   **编程语言**: Python 3.10
*   **主要库**: PyTorch, Transformers (用于 BERT), requests, numpy, tqdm, logging, re, json, os, time, random, gc, dataclasses (详见代码 import 部分)。
*   **LLM 模型**: `deepseekv3` (Track 1), `qwen-max` (Track 2) (通过 API 调用)。
*   **预训练模型**: `bert-base-chinese` (用于 Track 1 的辅助模块)。
*   **硬件**: 需要能够运行 Python 代码的环境。Track 1 中的 BERT 训练/推理步骤如果使用 GPU 会显著加速（代码中包含 CUDA 相关设置）。LLM 调用依赖网络连接和 API 服务。内存管理已在代码中考虑（`gc.collect()`, `torch.cuda.empty_cache()`）。

## 7. 结果分析

*   **Track 1**: 混合 LLM 多智能体和无监督 BERT 的方法显示出较好的效果。多智能体提供了深度分析，BERT 模块利用测试数据特性增强了鲁棒性，冲突仲裁机制有效解决了模型间的分歧。
*   **Track 2**: 多阶段 LLM 流水线结合严格的约束（内容聚焦、长度控制）能够生成符合要求的、高质量的相关性评论。依赖 Track 1 的准确分类是生成恰当评论的关键。Few-Shot 引导和最终的合规性检查对保证输出质量至关重要。备用评论机制确保了系统的完整性。
*   **整体**: 系统的最终排名（第 4 名）证明了所采用方法的有效性。

## 8. 引用与致谢

*   本项目的实现得益于对大型语言模型和 Transformer 架构的理解与应用。
*   方法设计参考了多智能体系统、Few-Shot Learning、对比学习等相关技术。
*   感谢 **NLPCC 2025 EOTGC (DUFL2025)** 任务组织方（CubeNLP, East China Normal University）提供评测平台和数据。
