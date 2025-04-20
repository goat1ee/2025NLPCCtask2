## 方法报告

### Track 1: 作文切题度自动评分

**1 研究目标与方法概述**

本研究旨在开发并评估一种自动化系统，用于对中小学生作文的切题度进行多级别分类（涵盖“优秀”、“较好”、“一般”、“合格”、“不合格”五个等级）。为实现此目标，我们构建了一种混合架构，该架构融合了基于大型语言模型（LLM）的多智能体协作系统与一个辅助性的BERT（Bidirectional Encoder Representations from Transformers）分类模块。该系统利用了测试数据 (`test_data.json`) 自身的特点进行模型微调和评估，并参考 `samples.json` 中的样本进行引导。

**2 LLM多智能体分析框架**

该框架的核心是一个通过精心设计的提示工程（Prompt Engineering）引导的、由多个功能专一的LLM智能体（基于代码中指定的 API 模型， `deepseekv3`）组成的级联处理流程。输入数据包括作文要求、年级、标题和内容。各智能体按序执行特定分析任务，为最终评分提供多维度信息，部分智能体运用了从 `samples.json` 加载的Few-Shot样本进行引导：

1.  **作文类型识别智能体 (Agent 0)**：首先分析作文的体裁类型（如记叙文、议论文等），为后续分析设定文体背景。
2.  **题目要求解析智能体 (Agent 1)**：负责精确解析作文题目的显性与隐性指令，建立规范化的切题度评估基准。
3.  **文章主旨提取智能体 (Agent 2)**：负责从作文文本中提炼其核心论题或记叙主线，并评估主旨表达的明确性。
4.  **内容-任务符合度校验智能体 (Agent 3)**：基于前两个智能体的输出，严谨校验作文内容与既定题目要求之间的符合度，识别关键的契合点与偏离点。
5.  **素材-主题契合度审核智能体 (Agent 4)**：评估作文中所选用的素材（如事例、细节描写）支撑核心主旨及回应题目要求的有效性与相关性。*(使用了Few-Shot样本)*
6.  **综合裁决智能体 (Agent 5)**：整合上述所有分析报告，依据官方评分标准细则，生成初步的切题度分类（优秀/较好/一般/合格/不合格）。*(使用了Few-Shot样本)*
7.  **分类结果提取智能体 (Agent 6)**：从综合裁决报告中精确提取最终的分类标签（五个等级词之一）。

**3 BERT辅助分类模块**

为增强系统评估的鲁棒性，我们引入了一个基于`bert-base-chinese`预训练模型的增强型BERT架构 (`EnhancedBertEssayModel`)。该模型包含多头自注意力机制、针对题目、标题、内容分别处理的编码头、特征融合模块、双向LSTM层以及最终的分类层。

*   **输入处理**：`AdvancedEssayDataset`类负责将作文要求、标题和内容进行分段编码和填充，然后拼接成统一的输入序列，送入BERT模型。
*   **无监督训练**：采用无监督学习范式，利用测试集数据 (`test_data.json`，经过ID处理和有效性筛选后)，通过对比损失函数 (`ContrastiveLoss`, InfoNCE风格) 对模型进行训练。训练目标是优化BERT提取的特征表示，使其能更好地区分不同作文的语义。函数`load_or_train_bert_model_unsupervised`负责加载或训练此模型，并将训练好的模型保存为`bert_essay_model.pt`。
*   **辅助预测**：训练好的模型 (`bert_essay_model.pt`) 对每篇作文独立输出一个切题度预测分类。

**4 混合决策与冲突仲裁机制**

系统的最终分类决策 (`final_classification`) 采用如下混合机制：

1.  **主要判定**：以LLM多智能体框架（特别是Agent 5/6输出）的分类 (`llm_agent_classification`) 作为主要判定依据。
2.  **辅助预测**：BERT模块提供独立的辅助分类预测 (`bert_classification`)。
3.  **冲突仲裁**：当 LLM 框架与 BERT 模块的评估结果出现不一致时，系统激活一个专门设计的LLM冲突仲裁智能体 (Arbiter Agent)。该仲裁智能体接收作文原文、题目要求、LLM与BERT的冲突分类结果以及LLM综合裁决报告的关键片段作为输入，并结合从`samples.json`加载的Few-Shot样本进行引导。其核心任务是依据官方评分标准，进行独立的最终裁决，以解决分类分歧。
4.  **最终决策**：
    *   若LLM与BERT评估一致，采纳一致结果。
    *   若不一致，采纳仲裁智能体的裁决结果。
    *   若BERT预测失败或仲裁智能体调用失败，则直接采纳LLM框架的原始裁决结果。
    *   对最终确定的分类进行有效性检查，若无效（非五个预定义标签之一）则强制设为“一般”。
5.  **ID格式化**：最终将所有作文的分类结果按顺序（从0开始）格式化ID，并保存到`DUFL2025_track1.json`文件中。

**5 技术细节**
*   **API调用**：使用`requests`库调用LLM API，实现了包含指数退避的重试逻辑、API密钥轮换机制以及超时设置。
*   **缓存机制**：使用`processed_essays.json`文件缓存已处理作文的结果，避免重复计算，提高效率。
*   **内存管理**：在处理大量作文时，代码中加入了`gc.collect()`和`torch.cuda.empty_cache()`来尝试清理内存。

该混合方法通过结合LLM的深度语境理解、上下文分析能力与BERT基于无监督学习适应测试数据的语义特征提取能力，并引入冲突仲裁层，旨在实现更准确、更可靠的作文切题度自动评估。

---

### Track 2: 相关性评论自动生成

**1 研究目标与方法概述**

本任务的核心目标是基于作文的切题度分析，自动生成一段仅聚焦于作文中心思想及其与题目要求相关性的评价性文本（评语），且评语长度严格控制在 120-180 个字符之间（根据代码实现）。为此，我们设计并实现了一个多阶段的LLM智能体流水线（Pipeline），同样基于代码中指定的 API 模型（`qwen-max`）。

**2 系统输入与依赖**

该系统的运行不仅依赖于输入的作文数据（题目要求、年级、标题、内容来自于`test_data.json`），亦关键性地利用了Track 1任务输出的最终切题度分类结果 (从`DUFL2025_track1.json`加载) 作为指导信息。同时，系统也加载`samples.json`文件，将其中的样本用于指导部分智能体的行为（Few-Shot Learning）。

**3 LLM评论生成流水线**

该流水线包含以下按序执行的LLM智能体：

1.  **作文类型识别智能体 (Agent 0)**：识别作文类型，提供文体背景。
2.  **题目要求分析智能体 (Agent 1)**：解析题目要求，提供上下文参照。
3.  **作文主旨提取智能体 (Agent 2)**：提取作文核心内容与中心思想。
4.  **年级适配性评估智能体 (Agent A)**：评估作文切题表现是否符合所在年级（基于Track 1分类结果）。
5.  **写作技巧分析智能体 (Agent B)**：分析技巧、表达方式对切题度和中心思想表达的影响。
6.  **素材选择评估智能体 (Agent C)**：评估素材选择与应用对切题度和中心思想的支持度。
7.  **结构逻辑分析智能体 (Agent D)**：分析结构逻辑对切题度和中心思想表达的影响。
8.  **初步评语草稿生成智能体 (Agent 4)**：结合前述所有分析报告和Track 1的分类结果 (`predicted_classification`)，生成初步评语草稿。其提示被严格约束，确保：(a) 内容聚焦切题性与中心思想；(b) 基调与Track 1分类一致；(c) 若非“优秀”，提供仅针对提升切题性的建议；(d) 参考`samples.json`中对应分类的Few-Shot样本；(e) 目标长度为 120-180 字符。*(使用了Few-Shot样本)*
9.  **合规性检查与精炼智能体 (Agent 5)**：作为最后环节，对草稿评语执行严格检查与优化，并参考Few-Shot样本：
    *   **内容约束验证**: 强制聚焦于“切题度/中心思想”及其影响因素，移除无关评价。
    *   **格式规范化**: 清除特殊格式，输出纯文本。
    *   **长度约束执行**: 严格确保最终评语字符数在 **120-180 个字符之间**。若过长则精简，若过短则在不违规前提下尝试补充，最终通过`clean_comment`函数和主流程逻辑确保。
    *   **分类一致性复核**: 确认评语基调与Track 1分类吻合。
    *   **语言专业性润色**: 提升表达的准确性、流畅度和专业性。*(使用了Few-Shot样本)*
    *   **总结句规范**: 对于“优秀”、“较好”、“一般”、“合格”的评语，确保最后一句是规范的总结句（特定开头词+重申切题表现），且总结句中不含建议。

**4 缓存与容错**

*   **缓存**：系统使用`processed_comments_cache_track2.json`文件缓存已成功生成的、符合长度要求的评论（以原始ID，即Track 2 ID为键），避免重复生成。
*   **重试**：整个Agent链条（从Agent 0到Agent 5）以及最终的评论长度检查，都包含在一个重试循环中。如果任何步骤失败（如API错误）或最终评论不符合长度要求，系统会尝试重新执行整个流程（最多重试`max_retries`次）。
*   **容错/备用评语**：如果在多轮重试后，仍无法生成符合要求的评论（特别是长度要求），系统会根据Track 1的分类结果生成一条预设的、模板化的备用评语 (`backup_comment`)，并强制将其长度调整到 120-180 字符范围内。

**5 输出格式化**

最终，系统将所有作文成功生成（或使用备用）的评论，与其在`test_data.json`中对应的索引（即Track 1 ID）进行匹配，并保存到`DUFL2025_track2.json`文件中，确保输出ID（0, 1, 2...）与Track 1的输出文件一致。

通过这一系列智能体的协同工作和严格的约束控制，特别是最终环节的合规性检查与长度控制，本系统旨在生成高度聚焦于任务要求（切题度与中心思想）、满足所有格式与长度限制（120-180字符）的高质量自动化评语。

---


## Method Report

### Track 1: Automated Essay Relevance Scoring

**1 Objective and Methodological Overview**

This study aims to develop and evaluate an automated system for the multi-level classification of essay relevance in primary and secondary school student writing, covering five categories: "Excellent" (优秀), "Good" (较好), "Average" (一般), "Qualified" (合格), and "Unqualified" (不合格). To achieve this, we constructed a hybrid architecture integrating a multi-agent collaborative system based on Large Language Models (LLMs) with an auxiliary Bidirectional Encoder Representations from Transformers (BERT) classification module. The system leverages the characteristics of the test data (`test_data.json`) itself for model fine-tuning and evaluation, and references samples from `samples.json` for guidance.

**2 LLM Multi-Agent Analysis Framework**

The core of this framework, guided by meticulous prompt engineering, comprises a cascaded processing pipeline of specialized LLM agents (based on the API model specified in the code, **`deepseekv3`**). Input data includes the essay prompt, grade, title, and content. Each agent sequentially performs a specific analytical task, providing multi-dimensional information for the final score. Some agents utilize Few-Shot samples loaded from `samples.json` for guidance:

1.  **Essay Type Classification Agent (Agent 0)**: Initially analyzes the essay's genre (e.g., narrative, argumentative), setting the context for subsequent analysis.
2.  **Prompt Requirements Parsing Agent (Agent 1)**: Precisely parses the explicit and implicit instructions of the essay prompt to establish standardized relevance assessment benchmarks.
3.  **Essay Theme Extraction Agent (Agent 2)**: Extracts the core thesis or narrative thread from the essay text and assesses the clarity of the main idea's expression.
4.  **Content-Task Alignment Verification Agent (Agent 3)**: Rigorously verifies the alignment between the essay content and the established prompt requirements (based on outputs from the preceding two agents), identifying key points of congruence and divergence.
5.  **Material-Theme Coherence Auditing Agent (Agent 4)**: Evaluates the effectiveness and relevance of supporting materials (e.g., examples, descriptive details) in bolstering the core theme and addressing prompt requirements. *(Utilizes Few-Shot samples)*
6.  **Final Adjudication Agent (Agent 5)**: Integrates all preceding analysis reports and, based on official scoring rubric specifications, generates a preliminary relevance classification (Excellent/Good/Average/Qualified/Unqualified). *(Utilizes Few-Shot samples)*
7.  **Classification Extraction Agent (Agent 6)**: Accurately extracts the final classification label (one of the five category words) from the Final Adjudication Agent's report.

**3 BERT Auxiliary Classification Module**

To enhance the robustness of the system's assessment, we incorporated an enhanced BERT architecture (`EnhancedBertEssayModel`) based on the `bert-base-chinese` pre-trained model. This model includes multi-head self-attention, separate encoding heads for prompt, title, and content, feature fusion modules, a bidirectional LSTM layer, and a final classification layer.

*   **Input Processing**: The `AdvancedEssayDataset` class handles segment encoding, padding, and concatenation of the prompt, title, and content into a unified input sequence for the BERT model.
*   **Unsupervised Training**: An unsupervised learning paradigm is employed, utilizing the test set data (`test_data.json`, after ID processing and validity checks) to train the model via a contrastive loss function (`ContrastiveLoss`, InfoNCE style). The training aims to optimize the feature representations extracted by BERT, enabling better discrimination between the semantics of different essays. The `load_or_train_bert_model_unsupervised` function manages loading or training this model, saving the trained model as `bert_essay_model.pt`.
*   **Auxiliary Prediction**: The trained model (`bert_essay_model.pt`) independently outputs a relevance classification prediction for each essay.

**4 Hybrid Decision-Making and Conflict Arbitration Mechanism**

The system's final classification decision (`final_classification`) employs the following hybrid mechanism:

1.  **Primary Determination**: The classification output by the LLM multi-agent framework (specifically from Agent 5/6, `llm_agent_classification`) serves as the primary determination.
2.  **Auxiliary Prediction**: The BERT module provides an independent, auxiliary classification prediction (`bert_classification`).
3.  **Conflict Arbitration**: When a discrepancy arises between the assessments of the LLM framework and the BERT module, the system activates a specifically designed LLM Conflict Arbitration Agent (Arbiter Agent). This arbitration agent receives the original essay, prompt requirements, the conflicting classification results from both LLM and BERT, key excerpts from the LLM adjudication report, and is guided by Few-Shot samples from `samples.json`. Its core task is to perform an independent final adjudication based on the official scoring rubrics to resolve the classification disagreement.
4.  **Final Decision**:
    *   If LLM and BERT assessments are consistent, the consensus result is adopted.
    *   If inconsistent, the result from the Arbiter Agent is adopted.
    *   If the BERT prediction fails or the Arbiter Agent call fails, the original adjudication result from the LLM framework is directly adopted.
    *   The final determined classification is validated; if invalid (not one of the five predefined labels), it defaults to "Average" (一般).
5.  **ID Formatting**: Finally, the classification results for all essays are assigned sequential integer IDs starting from 0 and saved to the `DUFL2025_track1.json` file.

**5 Technical Details**
*   **API Calls**: Uses the `requests` library to call the LLM API, implementing retry logic with exponential backoff, API key rotation, and timeout settings.
*   **Caching**: Utilizes `processed_essays.json` to cache results for processed essays, avoiding re-computation and improving efficiency.
*   **Memory Management**: Incorporates `gc.collect()` and `torch.cuda.empty_cache()` calls within the processing loop to attempt memory cleanup when handling large numbers of essays.

This hybrid approach, combining the deep contextual understanding and analytical capabilities of LLMs with the semantic feature extraction proficiency of BERT (adapted to the test data via unsupervised learning), augmented by a conflict resolution layer, aims to achieve more accurate and reliable automated essay relevance scoring.

---

### Track 2: Automated Relevance-Focused Comment Generation

**1 Objective and Methodological Overview**

The primary objective of this task is, based on the essay's relevance analysis, to automatically generate evaluative text (comments) strictly focused on the essay's central theme and its relevance to the prompt requirements, with a strict length constraint of **120-180 characters** (as implemented in the code). To this end, we designed and implemented a multi-stage LLM agent pipeline, also based on the API model specified in the code (**`qwen-max`**).

**2 System Input and Dependencies**

The operation of this system relies not only on the input essay data (prompt, grade, title, content from `test_data.json`) but also crucially utilizes the final relevance classification results from Track 1 (loaded from `DUFL2025_track1.json`) as guiding information. Additionally, the system loads `samples.json` to provide Few-Shot examples for guiding specific agents.

**3 LLM Comment Generation Pipeline**

This pipeline involves the sequential execution of the following LLM agents:

1.  **Essay Type Classification Agent (Agent 0)**: Identifies the essay type, providing genre context.
2.  **Prompt Requirements Analysis Agent (Agent 1)**: Parses prompt requirements for contextual reference.
3.  **Essay Theme Extraction Agent (Agent 2)**: Extracts the core content and central theme.
4.  **Grade Adaptability Assessor (Agent A)**: Assesses if the relevance performance (based on Track 1 class) aligns with the grade level.
5.  **Writing Technique Analyzer (Agent B)**: Analyzes the impact of techniques/expression on relevance and theme expression.
6.  **Material Selection Assessor (Agent C)**: Evaluates the relevance and supportiveness of materials for theme/prompt adherence.
7.  **Structure & Logic Analyzer (Agent D)**: Analyzes the impact of structure/logic on theme/relevance expression.
8.  **Draft Comment Generation Agent (Agent 4)**: Synthesizes all preceding analyses and the Track 1 classification (`predicted_classification`) to generate an initial draft. Prompts strictly enforce: (a) focus on relevance/theme; (b) alignment with Track 1 class; (c) relevance-focused suggestions if not "Excellent"; (d) guidance from Few-Shot samples from `samples.json` corresponding to the classification; (e) target length of **120-180 characters**. *(Utilizes Few-Shot samples)*
9.  **Compliance Checking and Refinement Agent (Agent 5)**: Performs final validation and refinement, referencing Few-Shot samples:
    *   **Content Constraint Validation**: Enforces strict focus on relevance/theme and their influencing factors, removing extraneous evaluations.
    *   **Format Standardization**: Ensures pure plain text output.
    *   **Length Constraint Enforcement**: Strictly ensures the final character count is **between 120 and 180 characters**. Overly long comments are condensed; attempts are made to pad short ones compliantly. This is ultimately enforced by the `clean_comment` function and main process logic.
    *   **Classification Consistency Review**: Confirms alignment with the Track 1 classification.
    *   **Professional Language Polishing**: Improves accuracy, fluency, and professionalism. *(Utilizes Few-Shot samples)*
    *   **Concluding Sentence Regulation**: For "Excellent", "Good", "Average", "Qualified" comments, ensures the last sentence is a proper summary (specific starting words + reiteration of relevance assessment) and contains no new suggestions.

**4 Caching and Fault Tolerance**

*   **Caching**: Uses `processed_comments_cache_track2.json` to store successfully generated, length-compliant comments (keyed by the original ID, i.e., Track 2 ID), preventing redundant work.
*   **Retry Mechanism**: The entire agent chain (Agent 0 through Agent 5) and the final comment length check are enclosed in a retry loop. If any step fails (e.g., API error) or the final comment fails the length check, the system attempts to re-run the entire process (up to `max_retries` times).
*   **Fault Tolerance/Fallback**: If, after multiple retries, a compliant comment (especially meeting length requirements) cannot be generated, the system constructs a predefined, template-based fallback comment (`backup_comment`) based on the Track 1 classification, and forcibly adjusts its length to fall within the **120-180 character range**.

**5 Output Formatting**

Finally, the system matches the successfully generated (or fallback) comments for all essays with their corresponding index in `test_data.json` (which is the Track 1 ID) and saves them to the `DUFL2025_track2.json` file. This ensures the output IDs (0, 1, 2...) align with the Track 1 output file.

Through the collaborative work of these agents and strict constraint enforcement, particularly the final compliance checks and length control, this system aims to produce high-quality, automated comments that are highly focused on the specified task requirements (relevance and central theme) and meet all formatting and length limitations (**120-180 characters**).

---