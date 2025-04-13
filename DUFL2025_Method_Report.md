**方法报告**

**Track 1: 作文切题度自动评分**

**1 研究目标与方法概述**
本研究旨在开发并评估一种自动化系统，用于对中小学生作文的切题度进行多级别分类（涵盖“优秀”、“较好”、“一般”、“合格”、“不合格”五个等级），并提供相应的预测分数。为实现此目标，我们构建了一种混合架构，该架构融合了基于大型语言模型（LLM）的多智能体协作系统与一个辅助性的BERT（Bidirectional Encoder Representations from Transformers）分类模块。

**2 LLM多智能体分析框架**
该框架的核心是一个通过精心设计的提示工程（Prompt Engineering）引导的、由多个功能专一的LLM智能体（主要基于DeepSeek模型）组成的级联处理流程。各智能体按序执行特定分析任务，为最终评分提供多维度信息：

*   **0. 作文类型识别智能体 (新增):** 首先根据作文要求、标题和内容，初步识别作文的体裁（如记叙文、议论文等），为后续分析提供文体背景。
*   **1. 题目要求解析智能体:** 精确解析作文题目的显性与隐性指令（核心任务、限制条件、隐含要求等），建立规范化的切题度评估基准。
*   **2. 文章主旨提取智能体:** 从作文文本中提炼其核心内容（事件、对象、观点）和中心思想/主旨，并评估标题契合度与主旨清晰度。
*   **3. 内容-任务符合度校验智能体:** 基于前两个智能体的输出，严谨校验作文内容（核心任务完成度、限制条件符合度、隐含要求响应度）与既定题目要求之间的符合度，识别关键的契合点与偏离点，并给出初步切题度评级。
*   **4. 素材-主题契合度审核智能体:** 评估作文中所选用的素材（事例、描写、细节等）与其核心任务/主旨的关联度、典型性、适切性，并给出素材服务主旨的总体有效性评级。
*   **5. 综合裁决智能体:** 整合上述所有分析报告，依据官方评分标准细则 (**严格**），对切题度做出**最终等级评定**，并提供详细的定级理由和决定性因素。此步骤生成主要的LLM分类结果。
*   **6. 分类结果提取智能体:** 从综合裁决报告中精确、简洁地提取最终的分类标签词。

**3 BERT辅助分类模块**
为增强系统评估的鲁棒性，我们引入了一个基于`bert-base-chinese`预训练模型的增强型BERT架构（`EnhancedBertEssayModel`）。该模型融合了多头自注意力机制（MultiHeadSelfAttention）、特征融合模块（FeatureFusionModule）和双向LSTM层。

*   **输入:** 该模块接收作文的题目要求、年级、标题及正文内容的拼接文本作为输入（内部也能处理分开的特征）。
*   **训练:** 采用无监督学习范式，利用测试集数据本身进行训练。具体地，通过基于模型中间层（高级特征网络输出）表示的对比损失函数（ContrastiveLoss / InfoNCE）进行优化，旨在捕捉文本与切题度相关的深层语义特征。
*   **输出:** 独立输出一个辅助性的切题度预测分类（五分类之一）。

**4 混合决策与冲突仲裁机制**
系统的最终分类决策采用如下混合机制：

*   **主要判定:** 以LLM多智能体框架（特别是综合裁决智能体Agent 5）输出的分类作为主要判定依据。
*   **辅助预测:** BERT模块提供独立的辅助分类预测。
*   **冲突仲裁:** 当LLM框架与BERT模块的评估结果出现不一致时，系统激活一个专门设计的LLM冲突仲裁智能体（Arbiter Agent）。该仲裁智能体接收作文原文、题目要求、LLM与BERT的冲突分类结果以及LLM综合裁决报告的关键片段作为输入。其核心任务是依据官方评分标准，进行独立的最终裁决，以解决分类分歧，输出最终采纳的分类标签。
*   **决策优先级:**
    *   若LLM与BERT评估一致，则采纳该一致结果。
    *   若不一致，优先采纳冲突仲裁智能体的裁决结果。
    *   若仲裁智能体调用失败或返回无效结果，则保留LLM综合裁决智能体的原始结果。
    *   若BERT预测失败或未启用，则直接采纳LLM综合裁决智能体的结果。
    *   所有最终输出的分类标签都会进行有效性验证，确保是五个预定义标签之一，否则默认为“一般”。

该混合方法通过结合LLM的深度语境理解、多角度分析能力与BERT的语义特征提取能力，并引入冲突仲裁层，旨在实现更准确、更可靠的作文切题度自动评估。最终输出包含与输入对应的ID（格式化为0开始的整数）和最终分类标签。


**Track 2: 相关性评论自动生成**

**1 研究目标与方法概述**
本任务的核心目标是基于作文的切题度分析，自动生成一段**仅聚焦于作文中心思想及其与题目要求相关性**的评价性文本（评语）。为此，我们设计并实现了一个包含多个LLM智能体的顺序执行流水线（Pipeline）。

**2 系统输入与依赖**
该系统的运行不仅依赖于输入的作文数据（题目要求、年级、标题、内容），亦关键性地利用了**Track 1任务输出的最终切题度分类结果** (`DUFL2025_track1.json`) 作为指导信息，以确保评语的评价基调与评分结果一致。

**3 LLM评论生成流水线**
该流水线包含以下按序执行的LLM智能体（主要基于DeepSeek模型）：

*   **0. 作文类型识别智能体 (新增):** 同Track 1，识别作文体裁，为后续分析提供背景信息。
*   **1. 题目要求分析智能体:** 再次对题目要求进行分析，提取关键信息点，为后续评语生成提供精确的上下文参照。
*   **2. 作文主旨提取智能体:** 提取作文实际表达的核心内容和中心思想。
*   **3. 切题度评估智能体:** 基于前两步的分析结果，生成一段关于作文切题表现的**描述性文本**（注意：此阶段非输出分类标签，而是对切题情况的文字化评估）。
*   **4. 初步评语草稿生成智能体:** 此智能体结合第3步的切题度描述性文本与**Track 1的最终分类结果**，生成初步评语。其提示被**严格约束**，确保：
    *   (a) 评语内容严格限定于切题性及中心思想表达。
    *   (b) 整体评价基调与Track 1的输入分类结果**高度吻合**。
    *   (c) 若切题度评估结果非最优（如非“优秀”），则必须提供**1条**具体的、仅针对提升“切题性”或“中心思想贴合度”的改进建议。
    *   (d) 评语使用专业术语和符合教师身份的语气。
    *   (e) 草稿的初步长度目标设定在200-250字符范围内。
*   **5. 合规性检查与精炼智能体:** 作为流水线的**最后关键环节**，此智能体对草稿评语执行多重、严格的验证与优化：
    *   **内容约束验证:** **强制移除**任何涉及语言风格、结构逻辑、素材细节、修辞手法、错别字、篇幅等非切题性/非中心思想相关的所有内容。这是**最核心**的过滤步骤。
    *   **格式规范化:** 清除所有Markdown标记、特殊符号、不必要的标点及格式（如列表、多余空格、换行），确保输出为单一、连续的纯文本段落。
    *   **长度约束执行:** **严格确保**最终评语的字符数精确落在 **200 到 250 字符**的目标区间内。对于过短或过长的草稿进行合规调整（删减或在不违反内容约束下扩展）。
    *   **分类一致性复核:** 再次确认评语的评价倾向与Track 1的输入分类标签完全吻合。
    *   **语言专业性润色:** 在满足所有约束的前提下，提升语言表达的准确性、流畅度与专业性，避免口语化或模糊表述。

通过这一系列智能体的协同工作，特别是最终环节的严格过滤与精炼，本系统旨在生成高度聚焦于任务要求（切题度与中心思想）、满足所有格式与长度限制（200-250字符）的高质量自动化评语。若流水线在任何步骤失败或最终评论不合规，则会生成一个基于Track 1分类结果的模板化备用评语。最终输出包含与Track 1对应的ID（0开始的整数）和最终生成的评语文本。

**Method Report**

**Track 1: Automated Essay Relevance Scoring**

**1. Objective and Methodological Overview**
This study aims to develop and evaluate an automated system for the multi-level classification of essay relevance in primary and secondary school student writing, encompassing five categories ("Excellent", "Good", "Average", "Qualified", "Unqualified"), and providing corresponding predicted scores. To achieve this, we constructed a hybrid architecture integrating a multi-agent collaborative system based on Large Language Models (LLMs) with an auxiliary Bidirectional Encoder Representations from Transformers (BERT) classification module.

**2. LLM Multi-Agent Analysis Framework**
The core of this framework, guided by meticulous prompt engineering, comprises a cascaded processing pipeline of specialized LLM agents (primarily based on the DeepSeek model). Each agent sequentially performs a specific analytical task, providing multi-dimensional information for the final score:

*   **0. Essay Type Classifier Agent (New):** Initially identifies the essay's genre (e.g., narrative, argumentative) based on the prompt, title, and content, providing contextual background for subsequent analysis.
*   **1. Prompt Requirements Parsing Agent:** Precisely parses the explicit and implicit instructions of the essay prompt (including core task, constraints, implicit requirements) to establish standardized relevance assessment benchmarks.
*   **2. Essay Theme Extraction Agent:** Extracts the core content (events, subjects, viewpoints) and the central theme or main idea from the essay text, also assessing title alignment and the clarity of the main idea's expression.
*   **3. Content-Task Alignment Verification Agent:** Rigorously verifies the alignment between the essay content (core task completion, constraint adherence, implicit requirement responsiveness) and the established prompt requirements (based on outputs from the preceding two agents), identifying key points of congruence and divergence, and providing a preliminary relevance rating.
*   **4. Material-Theme Coherence Auditing Agent:** Evaluates the relevance, typicality, appropriateness, and overall effectiveness of the supporting materials (e.g., examples, descriptions, details) used in the essay in relation to its core task and main theme.
*   **5. Final Adjudication Agent:** Integrates all preceding analysis reports and, based **strictly** on official scoring rubric specifications, makes a **final grade determination** for relevance, providing detailed reasoning and identifying the decisive factor(s). This step generates the primary LLM classification result.
*   **6. Classification Extraction Agent:** Accurately and concisely extracts the final classification label word from the Final Adjudication Agent's report.

**3. BERT Auxiliary Classification Module**
To enhance the robustness of the system's assessment, we incorporated an enhanced BERT architecture (`EnhancedBertEssayModel`) based on the `bert-base-chinese` pre-trained model. This model integrates multi-head self-attention (MultiHeadSelfAttention), feature fusion modules (FeatureFusionModule), and a bidirectional LSTM layer.

*   **Input:** This module receives concatenated text—comprising the essay's prompt requirements, grade level, title, and content—as input (it can also internally process separated features).
*   **Training:** It is trained using an unsupervised learning paradigm, utilizing the test set data itself. Specifically, optimization is performed using a contrastive loss function (`ContrastiveLoss` / InfoNCE) based on representations from an intermediate layer (the output of the advanced feature network), aiming to capture deep semantic features relevant to essay relevance.
*   **Output:** It independently outputs an auxiliary relevance classification prediction (one of the five categories).

**4. Hybrid Decision-Making and Conflict Arbitration Mechanism**
The system's final classification decision employs the following hybrid mechanism:

*   **Primary Determination:** The classification output by the LLM multi-agent framework (specifically, the Final Adjudication Agent - Agent 5) serves as the primary determination.
*   **Auxiliary Prediction:** The BERT module provides an independent, auxiliary classification prediction.
*   **Conflict Arbitration:** When a discrepancy arises between the assessments of the LLM framework and the BERT module, the system activates a specifically designed LLM Conflict Arbitration Agent (`Arbiter Agent`). This arbitration agent receives the original essay, prompt requirements, the conflicting classification results from both LLM and BERT, and key excerpts from the LLM Final Adjudication report as input. Its core task is to perform an independent final adjudication based strictly on the official scoring rubrics to resolve the classification disagreement and output the final adopted classification label.
*   **Decision Priority:**
    *   If the LLM and BERT assessments are consistent, that result is adopted.
    *   If they differ, the result from the Conflict Arbitration Agent is prioritized.
    *   If the Arbitration Agent fails or returns an invalid result, the original classification from the LLM Final Adjudication Agent is retained.
    *   If the BERT prediction fails or is not enabled, the classification from the LLM Final Adjudication Agent is directly adopted.
    *   All final output classification labels are validated to ensure they are one of the five predefined labels; otherwise, they default to "Average".

This hybrid approach, combining the deep contextual understanding and multi-faceted analytical capabilities of LLMs with the semantic feature extraction proficiency of BERT, augmented by a conflict resolution layer, aims to achieve more accurate and reliable automated essay relevance scoring. The final output includes IDs (reformatted to 0-based integers) corresponding to the input and the final classification labels.

---

**Track 2: Automated Relevance-Focused Comment Generation**

**1. Objective and Methodological Overview**
The primary objective of this task is, based on the essay's relevance analysis, to automatically generate evaluative text (comments) **strictly focused on the essay's central theme and its relevance to the prompt requirements**. To this end, we designed and implemented a sequential pipeline involving multiple LLM agents.

**2. System Input and Dependencies**
The operation of this system relies not only on the input essay data (prompt, grade, title, content) but also crucially utilizes the **final relevance classification results from Track 1** (`DUFL2025_track1.json`) as guiding information. This ensures the evaluative tone of the generated comment aligns with the assigned relevance score.

**3. LLM Comment Generation Pipeline**
This pipeline involves the sequential execution of the following LLM agents (primarily based on the DeepSeek model):

*   **0. Essay Type Classifier Agent (New):** Same as in Track 1, identifies the essay genre to provide context.
*   **1. Prompt Requirements Analysis Agent:** Re-analyzes the prompt requirements to extract key points, providing a precise contextual reference for subsequent comment generation.
*   **2. Essay Theme Extraction Agent:** Extracts the core content and central theme actually expressed in the essay.
*   **3. Relevance Assessment Agent:** Generates **descriptive text** evaluating the essay's relevance performance based on the analyses of the preceding two agents (Note: this stage outputs a textual assessment, not a classification label).
*   **4. Draft Comment Generation Agent:** This agent combines the descriptive relevance assessment text (from Agent 3) and the **final classification result from Track 1** to generate an initial comment draft. Its prompts are **strictly constrained** to ensure:
    *   (a) Comment content is rigorously limited to relevance and central theme expression.
    *   (b) The overall evaluative tone is **highly consistent** with the input classification result from Track 1.
    *   (c) If the relevance assessment indicates performance below "Excellent", it must provide **one** specific, actionable suggestion solely targeting the improvement of "relevance" or "central theme alignment".
    *   (d) The language uses professional terminology and maintains a tone appropriate for a teacher.
    *   (e) The initial target length for the draft is set within the 200-250 character range.
*   **5. Compliance Checking and Refinement Agent:** As the **critical final stage**, this agent performs multiple, rigorous validation and optimization steps on the draft comment:
    *   **Content Constraint Validation (Most Critical):** **Mandates the removal** of *any* content pertaining to language style, structural logic, material details, rhetorical devices, grammatical errors, length, or other aspects unrelated to relevance or the central theme. This is the **core filtering step**.
    *   **Format Standardization:** Eliminates all Markdown markup, special symbols, extraneous punctuation, and formatting (like lists, extra whitespace, newlines), ensuring the output is a single, continuous plain text paragraph.
    *   **Length Constraint Enforcement:** **Strictly ensures** the final comment's character count falls precisely within the **200 to 250 character** target range. Drafts that are too short or too long are adjusted accordingly (by trimming, or by expanding carefully without violating content constraints).
    *   **Classification Consistency Review:** Re-verifies that the comment's evaluative stance fully aligns with the input classification label from Track 1.
    *   **Professional Language Polishing:** Enhances linguistic accuracy, fluency, and professionalism, adhering to the norms of secondary school essay feedback, while satisfying all preceding constraints.

Through the collaborative work of this agent pipeline, particularly the stringent filtering and refinement applied in the final stage, the system is designed to generate high-quality, automated comments that are highly focused on the specified task requirements (relevance and central theme) and meet all formatting and length specifications (200-250 characters). If the pipeline fails at any step or the final comment does not meet compliance standards, a template-based fallback comment, informed by the Track 1 classification result, is generated. The final output includes IDs (0-based integers corresponding to Track 1) and the finalized comment text.