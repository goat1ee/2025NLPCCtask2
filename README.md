# DUFL2025 作文切题度评分与相关性评论生成系统

## 1. 项目背景

本项目是为 **DUFL2025 (NLPCC2025 Evaluation of Essay On-Topic Graded Comments - EOTGC)** 评测任务提供的解决方案。该任务旨在评估自动化系统在两个相关子任务上的能力：

1.  **Track 1**: 对中小学生作文进行切题度自动评分，输出“优秀”、“较好”、“一般”、“合格”、“不合格”五个等级的分类。
2.  **Track 2**: 基于 Track 1 的评分结果，生成一段仅聚焦于作文“切题度”和“中心思想”的相关性评论，且长度严格控制在 120-190 字符之间（根据代码实现）。

**任务目标**：构建一个能够准确评估作文切题度，并能生成高质量、符合特定约束的相关性评论的自动化系统。

## 2. 解决方案概述

本项目针对两个赛道分别设计了基于大型语言模型（LLM）的多智能体（Multi-Agent）解决方案，并结合了其他技术手段进行优化。

### 2.1 Track 1: 作文切题度自动评分

采用**混合架构**，融合了**基于 LLM 的多智能体协作系统**与一个**辅助性的 BERT 分类模块**：

*   **LLM 多智能体分析框架**:
    *   **模型**: `deepseek-chat` (根据 `track1.ipynb` 代码)。
    *   **流程**: 构建了一个包含多个功能专一的 LLM 智能体（代码中体现为多个 `generate_..._prompt` 函数调用，如类型识别 Agent 0、要求解读 Agent 1、主旨概括 Agent 2、符合性校验 Agent 3、素材审核 Agent 4、综合评审 Agent 5、结果提取 Agent 6）的级联处理流程。通过精心设计的提示工程引导，按序执行分析任务。
    *   **Few-Shot 引导**: 明确在素材审核 Agent (Agent 4) 和综合评审 Agent (Agent 5)，以及冲突仲裁的 Arbiter Agent 中使用了从 `samples.json` 加载并格式化的 Few-Shot 样本 (`format_few_shot_examples`)。
*   **BERT 辅助分类模块**:
    *   **模型**: 基于 `bert-base-chinese` 预训练模型，构建了一个增强型 BERT 架构 (`EnhancedBertEssayModel`)，包含多头自注意力 (`MultiHeadSelfAttention`)、特征融合 (`FeatureFusionModule`)、双向 LSTM (`torch.nn.LSTM`) 等复杂组件。
    *   **输入处理**: `AdvancedEssayDataset` 类负责将作文要求、标题和内容进行分段编码和填充，送入 BERT 模型。
    *   **训练**: 采用**无监督对比学习** (`ContrastiveLoss`, InfoNCE 风格)，利用测试集数据 (`test_data.json`，经过 ID 处理和有效性筛选后) 对 `EnhancedBertEssayModel` 进行训练，优化其特征表示能力。训练好的模型保存为 `bert_essay_model.pt`。该过程由 `load_or_train_bert_model_unsupervised` 函数管理。
    *   **作用**: 为每篇作文提供一个独立的切题度预测分类 (`bert_classification`)，作为 LLM 结果的补充和校验。
*   **混合决策与冲突仲裁**:
    *   主要判定依据 LLM 框架（特别是 Agent 5/6）的输出 (`llm_agent_classification`)。
    *   当 LLM 框架与 BERT 模块预测不一致时，激活一个专门设计的 LLM **冲突仲裁智能体 (Arbiter Agent)**。该智能体接收原文、要求、冲突结果、LLM 报告片段以及 Few-Shot 样本，依据官方评分标准进行最终裁决。
    *   根据一致性结果、仲裁智能体结果或在 BERT/仲裁失败情况下的回退逻辑（优先采纳 LLM 框架原始结果）确定最终分类 (`final_classification`)。
    *   对最终确定的分类进行有效性检查，若无效（非五个预定义标签之一）则强制设为“一般”。
*   **技术细节**:
    *   **API 调用**: 使用 `requests` 库调用 LLM API (`call_llm_api`)，实现了包含指数退避的重试逻辑、API 密钥轮换机制 (`switch_api_key`, `get_current_api_key`) 以及超时设置。
    *   **缓存机制**: 使用 `processed_essays.json` 文件 (`load_processed_essays`, `save_processed_essays`) 缓存已处理作文的结果，避免重复计算。
    *   **内存管理**: 在处理循环中加入了 `gc.collect()` 和 `torch.cuda.empty_cache()` 来尝试清理内存。
    *   **ID 格式化**: 最终将所有作文的分类结果按顺序（从 0 开始）格式化 ID，并保存到 `DUFL2025_track1.json` 文件中。

### 2.2 Track 2: 相关性评论自动生成

设计了一个**多阶段 LLM 智能体流水线**，严格控制输出的**内容焦点（切题度与中心思想）**和**长度（120-190 字符）**：

*   **模型**: `qwen-max` (根据 `track2.ipynb` 代码)。
*   **输入与依赖**: 依赖 `test_data.json` 中的作文数据，并关键性地**利用 Track 1 输出的最终切题度分类结果 (`DUFL2025_track1.json`)** 作为指导。同时，也使用 `samples.json` 进行 Few-Shot 引导 (`format_samples_for_prompt`)。
*   **LLM 评论生成流水线**:
    1.  **前期分析 (Agent 0-2)**: 作文类型识别 (`generate_essay_type_classifier_prompt`)、题目要求分析 (`generate_prompt_analyzer_prompt`)、作文主旨提取 (`generate_essay_theme_extractor_prompt`)。
    2.  **深度分析 (Agent A-D)**: 年级适配性评估 (`generate_grade_adaptability_assessor_prompt_A`)、写作技巧分析（对切题影响, `generate_technique_analyzer_prompt_B`）、素材选择评估（对切题影响, `generate_material_assessor_prompt_C`）、结构逻辑分析（对切题影响, `generate_structure_analyzer_prompt_D`）。这些分析均围绕对“切题度”和“中心思想表达”的影响进行。
    3.  **初步评语生成 (Agent 4)**: 结合所有分析报告和 Track 1 分类结果 (`predicted_classification`)，参照对应分类的 Few-Shot 样本 (`formatted_samples`) 和结构指导 (`COMMENT_STRUCTURE_GUIDELINES`)，生成聚焦切题度和中心思想的草稿 (`generate_draft_comment_generator_prompt`)，内部目标长度 190-220 字符。
    4.  **合规性检查与精炼 (Agent 5)**: 对草稿进行最终审核和润色 (`generate_compliance_checker_prompt`)，参照 Few-Shot 样本，严格执行多项规则：
        *   **内容约束**: 强制聚焦“切题度/中心思想”及其影响因素，移除无关评价。
        *   **格式规范**: 清除特殊格式，输出纯文本。
        *   **长度硬控**: 严格确保最终评语字符数在 **120-190 个字符之间**。若过长则精简，若过短则在不违规前提下尝试补充。此逻辑由 `clean_comment` 函数和 `generate_essay_comment` 函数中的重试与备用逻辑确保。
        *   **分类一致性复核**: 确认评语基调与 Track 1 分类吻合。
        *   **语言专业性润色**: 提升表达的准确性、流畅度和专业性。
        *   **总结句规范**: 对于“优秀”、“较好”、“一般”、“合格”的评语，确保最后一句是规范的总结句（特定开头词+重申切题表现），且总结句中不含建议。
*   **缓存与容错机制**:
    *   **缓存**: 使用 `processed_comments_cache_track2.json` 文件 (`load_processed_comments`, `save_processed_comments`) 缓存已成功生成的、符合长度要求的评论（以原始 Track 2 ID 为键），避免重复生成。
    *   **重试**: 整个 Agent 链条（从 Agent 0 到 Agent 5）以及最终的评论长度检查，都包含在一个重试循环 (`max_retries`) 中。如果任何步骤失败（如 API 错误）或最终评论不符合长度要求，系统会尝试重新执行整个流程。
    *   **容错/备用评语**: 如果在多轮重试后，仍无法生成符合要求的评论（特别是长度要求），系统会根据 Track 1 的分类结果生成一条预设的、模板化的**备用评语 (`backup_comment`)**，并强制将其长度调整到 120-190 字符范围内。
*   **输出格式化**: 最终，系统将所有作文成功生成（或使用备用）的评论，与其在 `test_data.json` 中对应的索引（即 Track 1 ID）进行匹配，并保存到 `DUFL2025_track2.json` 文件中，确保输出 ID（0, 1, 2...）与 Track 1 的输出文件一致。

## 3. 主要文件与脚本

*   `DUFL2025_Method_Report（最高版本）.md`: 详细描述了两个赛道所采用的技术方法、模型架构、处理流程和实现细节的报告文档（包含中英文版本）。
*   `track1.ipynb`: **Track 1 解决方案**的 Jupyter Notebook 实现代码。包含数据处理、模型调用（LLM API、BERT）、混合决策、结果生成等逻辑。
*   `track2.ipynb`: **Track 2 解决方案**的 Jupyter Notebook 实现代码。包含数据处理、依赖 Track 1 结果、LLM 智能体流水线调用、评论合规性检查与长度控制、结果生成等逻辑。
*   `DUFL2025_track1.json`: **Track 1 的最终输出文件**。包含对测试集中每篇作文的切题度分类结果（优秀、较好、一般、合格、不合格），格式为 `[{"id": 0, "classification": "..."}, ...]`。ID 从 0 开始，与 `test_data.json` 中的索引对应。
*   `DUFL2025_track2.json`: **Track 2 的最终输出文件**。包含对测试集中每篇作文生成的、符合要求（聚焦切题度与中心思想，长度 120-190 字符）的相关性评论，格式为 `[{"id": 0, "comment": "..."}, ...]`。ID 从 0 开始，与 `test_data.json` 中的索引对应。
*   `test_data.json`: 官方提供的测试数据集，包含需要进行评分和评论生成的作文原文及其相关信息（如年级、题目要求、标题、内容），是两个赛道处理的主要输入。
*   `samples.json`: 样本数据文件，包含带有标准分类和评论的作文样例，用于在 LLM 推理过程中提供 Few-Shot 示例。
*   `requirements.txt`: 项目所需的 Python 依赖库列表及其版本。
*   `Task Guideline.md`: 官方发布的评测任务指南，包含任务背景、描述、数据格式、评估标准等信息。
*   `processed_comments_cache_track2.json`: Track 2 评论生成过程的缓存文件。
*   `result.xlsx`: 包含运行过程中的一些中间结果、手动分析数据。
*   `README.md`: 本说明文件。
*   `processed_essays.json`: Track 1 评估过程的缓存文件。

## 4. 使用流程

1.  **安装依赖**:
    ```bash
    pip install -r requirements.txt
    ```
2.  **环境配置**:
    *   **API 密钥**: 在 `track1.ipynb` 和 `track2.ipynb` 代码中，找到 `API_KEYS` 列表，并替换为您有效的 LLM API 密钥 (分别为 `deepseek-chat` 和 `qwen-max` 的密钥)。
    *   **API URL**: 确认代码中的 `API_URL` 设置正确。
3.  **数据准备**:
    *   将官方提供的 `test_data.json` 和 `samples.json` 文件放置在与 `ipynb` 文件相同的目录下。
4.  **执行 Track 1 (切题度评分)**:
    *   打开并运行 `track1.ipynb` 中的所有代码单元格。
    *   **注意**: 首次运行时，如果 `bert_essay_model.pt` 不存在，脚本会尝试使用 `test_data.json` 进行无监督训练，这可能需要较长时间和 GPU 资源。
    *   **输出**: 生成 `DUFL2025_track1.json` (最终分类结果) 以及可能的缓存和模型文件。
5.  **执行 Track 2 (评论生成)**:
    *   确保上一步生成的 `DUFL2025_track1.json` 文件存在且有效。
    *   打开并运行 `track2.ipynb` 中的所有代码单元格。
    *   脚本将读取所需数据，执行 LLM 调用生成评论。
    *   **注意**: 此过程涉及多次 LLM API 调用，耗时较长。
    *   **输出**: 生成 `DUFL2025_track2.json` (最终评论结果) 和评论缓存文件。

## 5. 实验结果

本项目在 DUFL2025 评测任务中取得了以下成绩：

| Rank | System Name | Track1 Score | Track2 Score | Total Score |
| :--- | :---------- | :----------- | :----------- | :---------- |
| 4    | DUFL2025    | 0.7784       | 0.6235       | 0.6854      |

*说明：分数计算方法遵循官方 `Task Guideline.md` 中的定义。*

## 6. 硬件和软件环境

*   **操作系统**: Windows 10
*   **Python 版本**: 3.10.0 (conda-forge)
*   **主要库**:
    *   `torch==2.6.0+cu126`
    *   `transformers==4.50.3`
    *   `scikit-learn==1.6.1`
    *   `numpy==2.1.2`
    *   `requests==2.32.3`
    *   `tqdm==4.67.1`
    *   (其他标准库: `os`, `json`, `time`, `random`, `re`, `logging`, `collections`, `math`, `gc`, `typing`, `dataclasses`)
*   **LLM 模型 (API)**: `deepseek-chat` (Track 1), `qwen-max` (Track 2)
*   **基础模型 (本地)**: `bert-base-chinese` (由 `transformers` 库自动下载)
*   **硬件**:
    *   CPU: 需要能够运行 Python 3.10
    *   GPU: 推荐使用 CUDA 12.6 兼容的 NVIDIA GPU 以加速 Track 1 的 BERT 训练/推理。
    *   内存: 建议配置足够内存 (如 16GB+)，代码包含内存清理尝试。
    *   网络: 需要稳定的网络连接以调用 LLM API。

## 7. 结果分析

*   **Track 1**: 混合架构有效结合了 LLM 的深度分析能力和 BERT 对测试数据分布的适应性。多智能体分工明确，无监督 BERT 提供了有价值的辅助信号，冲突仲裁机制提升了系统面对模型分歧时的决策能力。
*   **Track 2**: 严格的内容和长度约束是生成合规评论的核心。多阶段 Agent 流水线设计，结合对 Track 1 结果的依赖和 Few-Shot 引导，实现了聚焦于“切题度”和“中心思想”的高质量评论生成。容错机制保证了输出的完整性。
*   **整体**: 系统在两个赛道上均取得了具有竞争力的成绩（总分排名第 4），验证了所提出方法的有效性。

## 8. 引用与致谢

*   本项目实现中使用了大型语言模型 API 服务 (`deepseek-chat`, `qwen-max`)。
*   使用了 Hugging Face Transformers 库及 `bert-base-chinese` 预训练模型。
*   感谢 **NLPCC 2025** 任务组织方 (CubeNLP, East China Normal University) 提供评测平台、数据和任务指南。
