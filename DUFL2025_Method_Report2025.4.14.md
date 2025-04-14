## 方法报告

**Track 1: 作文切题度自动评分**

**1 研究目标与方法概述**

本研究旨在开发并评估一种自动化系统，用于对中小学生作文的切题度进行多级别分类（涵盖“优秀”、“较好”、“一般”、“合格”、“不合格”五个等级）。为实现此目标，我们构建了一种混合架构，该架构融合了基于大型语言模型（LLM）的多智能体协作系统与一个辅助性的BERT（Bidirectional Encoder Representations from Transformers）分类模块。

**2 LLM多智能体分析框架**

该框架的核心是一个通过精心设计的提示工程（Prompt Engineering）引导的、由多个功能专一的LLM智能体（基于DeepSeek API模型，具体模型标识为`deepseek-chat`）组成的级联（Cascaded）处理流程。各智能体按序执行特定分析任务，为最终评分提供多维度信息：

1.  **作文类型识别智能体 (Agent 0):** 新增的初始步骤，负责根据作文要求、标题和内容初步判断作文的核心文体（如记叙文、议论文等），为后续分析提供文体背景。
2.  **题目要求解析智能体 (Agent 1: Prompt Precision Analyst):** 负责精确解析作文题目的显性与隐性指令，结合识别出的文体类型，建立规范化的切题度评估基准。
3.  **文章主旨提取智能体 (Agent 2: Essay Essence Extractor):** 负责从作文文本中提炼其核心论题、记叙主线或说明对象，并评估主旨表达的明确性与标题契合度。
4.  **内容-任务符合度校验智能体 (Agent 3: Relevance Alignment Verifier):** 基于前两个智能体的输出，严谨校验作文内容与既定题目要求之间的符合度，识别关键的契合点与偏离点，并给出初步符合性评级。
5.  **素材-主题契合度审核智能体 (Agent 4: Material-Theme Alignment Auditor):** 评估作文中所选用的素材（如事例、描写、细节）支撑核心主旨及回应题目要求的有效性与相关性，并给出素材有效性评级。
6.  **综合裁决智能体 (Agent 5: Final Relevance Adjudicator):** 此智能体整合Agent 1至Agent 4的所有分析报告，对照官方详细的评分标准细则，进行**高度严格、注重边界区分**的综合评判，生成包含明确等级和详尽定级理由的《切题度最终裁决报告》。此报告是LLM框架主要的分类判断来源。
7.  **分类结果提取智能体 (Agent 6: Final Classification Extractor):** 通过一个专门的LLM调用，确保从综合裁决报告中准确、无误地提取出最终的单分类标签词（如“较好”）。

**3 BERT辅助分类模块**

为增强系统评估的鲁棒性与准确性，我们引入了一个基于`bert-base-chinese`预训练模型的**增强型BERT架构（EnhancedBertEssayModel）**。该模型包含多头自注意力机制（MultiHeadSelfAttention）、特征融合模块（FeatureFusionModule）、双向LSTM层以及一个高级特征提取网络。此模块处理由作文的题目要求、标题、正文内容、以及年级信息编码后的表示。特别地，BERT模块采用**无监督学习**范式进行训练或微调：

*   **训练数据:** 系统首先尝试加载本地预训练好的模型权重 (`bert_essay_model.pt`)。若加载失败或模型不存在，则利用**当前批次的测试数据本身**（`test_data.json`）进行无监督训练。
*   **训练方法:** 采用**对比学习损失（Contrastive Loss）**，旨在让模型学习区分不同作文的语义表示，即使在没有显式标签的情况下也能捕捉与切题度相关的深层特征。
*   **输入处理:** 使用`AdvancedEssayDataset`对输入文本（要求、标题、内容）进行精细化处理和拼接，确保适应模型输入限制。

该模块在训练后，能够独立地对输入的作文文本输出一个辅助性的切题度预测分类（五个等级之一）。

**4 混合决策与冲突仲裁机制**

系统的最终分类决策整合了LLM和BERT的结果，并引入了特定的冲突解决流程：

1.  **主要依据:** LLM多智能体框架（特别是Agent 6的最终裁决）输出的分类作为主要的判定依据。
2.  **辅助验证:** BERT模块提供独立的辅助分类预测。
3.  **冲突检测:** 比较LLM的裁决结果与BERT的预测结果。
4.  **冲突仲裁:**
    *   当两者结果**不一致**时，系统**激活**一个专门设计的**LLM冲突仲裁智能体 (Arbitration Agent)**。
    *   该仲裁智能体接收作文原文、题目要求、LLM和BERT的冲突分类结果，以及LLM综合裁决智能体（Agent 5）报告的关键文本片段作为输入。
    *   仲裁智能体的核心任务是基于其提示中嵌入的官方评分标准，进行**独立的最终裁决**，旨在解决分类分歧，输出最终的单一分类词。
5.  **采纳规则:**
    *   若LLM与BERT评估**一致**，则直接采纳该一致结果。
    *   若BERT预测**失败**（例如模型未成功加载或训练），则直接采纳LLM框架的裁决结果。
    *   若仲裁智能体调用**失败**或返回无效结果，则**保留LLM框架的原始裁决结果**（Agent 6提取的分类）。
    *   在任何情况下，系统都会对最终采纳的分类标签进行有效性检查（是否在五个预设等级内），对于无效标签强制设定为“一般”。

**5 缓存与结果处理**

*   **处理缓存:** 系统使用`processed_essays.json`文件缓存已完成评估作文的详细中间报告和最终结果，避免重复处理。
*   **ID处理:** 输入数据若缺少ID，会生成临时ID。内部处理使用字符串ID。最终输出到`DUFL2025_track1.json`文件时，**ID会被重新格式化为从0开始的连续整数**，以符合竞赛提交要求。

该混合方法通过结合LLM的深度语境理解、多角度分析能力与BERT的语义特征提取能力，并引入了多层（LLM Adjudicator + BERT + Arbiter）的决策与校验机制，旨在实现更准确、更可靠的中小学生作文切题度自动评估。

---

**Track 2: 相关性评论自动生成**

**1 研究目标与方法概述**

本任务的核心目标是，基于对学生作文“切题度”和“中心思想”的分析，自动生成一段**高度聚焦**于这两个维度、评价性的文本（评语），并严格遵守格式与长度限制。为此，我们设计并实现了一个多阶段的LLM智能体流水线（Pipeline），同样基于DeepSeek API模型 (`deepseek-chat`)。

**2 系统输入与依赖**

该系统的运行不仅依赖于输入的作文数据（题目要求、年级、标题、内容），亦关键性地依赖于Track 1任务输出的**最终切题度分类结果**（从`DUFL2025_track1.json`文件中加载）。这份分类结果为后续评语的整体基调和建议方向提供了重要指导。

**3 LLM评论生成流水线**

该流水线包含以下按序执行的LLM智能体：

1.  **作文类型识别智能体 (Agent 0):** 与Track 1类似，首先识别作文的主要文体类型，为后续分析提供背景。
2.  **题目要求分析智能体 (Agent 1):** 再次对题目要求进行分析（结合文体），为后续评语内容提供精确的上下文参照。
3.  **作文主旨提取智能体 (Agent 2):** 提取作文实际表达的核心观点或记叙主旨。
4.  **切题度评估智能体 (Agent 3):** 基于Agent 1和Agent 2的分析结果，生成关于作文**切题表现的描述性文本**。*注意：此阶段不输出分类标签，而是生成一段评价性的自然语言文本，描述作文在切题方面的具体表现（如何契合或如何偏离）。*
5.  **初步评语草稿生成智能体 (Agent 4):** 此智能体是评语内容的核心生成器。它接收以下输入：
    *   Agent 3输出的切题度描述性文本。
    *   从`DUFL2025_track1.json`加载的该作文的**最终分类结果**（如“较好”、“合格”等）。
    *   识别出的作文类型信息（Agent 0 输出）。
    其提示（Prompt）被严格约束，以确保生成的草稿：
    *   **(a) 内容绝对聚焦:** 评语内容**严格限定**于讨论作文的“切题性”（如是否扣题、是否在规定范围内）及“中心思想表达”（如是否明确、是否符合题意）。
    *   **(b) 调性一致:** 评语的整体评价基调（肯定、中性、指出问题）必须与输入的**Track 1最终分类结果**保持高度一致。
    *   **(c) 针对性建议:** 如果输入的分类结果不是“优秀”，则**必须**提供**1条**具体的、可操作的、**仅关于如何改进“切题性”或“中心思想贴合度”**的建议。建议需原创，避免套话。分类为“优秀”则以肯定为主。
    *   **(d) 长度控制:** 尝试生成长度在 **250 到 300 字符**之间的文本（初步目标）。
6.  **合规性检查与精炼智能体 (Agent 5):** 作为流水线的最后一道、也是最关键的质量控制环节，此智能体对Agent 4生成的草稿评语执行多重、严格的验证与优化：
    *   **内容约束验证:** **强制性地、彻底地移除**任何涉及语言运用、篇章结构、素材细节、创意构思、情感表达、错别字、标点、篇幅等**所有非切题性/非中心思想相关**的内容。这是**首要且最严格**的规则。
    *   **内容安全检查:** 确保评语内容健康、合规，不含敏感或不当信息。
    *   **格式规范化:** 清除所有Markdown标记、特殊符号、非书名号的引号、列表标记、不必要的括号、换行符等，确保输出为**单一、连续的纯文本段落**。
    *   **长度硬性约束执行:** 严格确保最终评语的总字符数（不包含标点）**精确落在 200 至 250 字符**的目标区间内。如果过短，则在不违反内容约束的前提下补充相关描述；如果过长，则精简语言，保留核心评价。代码中多次强调“必须确保不少于 200 字符”。
    *   **分类一致性复核:** 再次确认评语的评价倾向与输入的**Track 1最终分类标签**完全吻合。
    *   **语言专业性润色:** 在满足所有上述约束的前提下，提升语言表达的准确性、流畅度与专业性，使其符合中小学语文教师的评语风格。

**4 缓存与备用机制**

*   **缓存:** 系统使用`processed_comments.json`文件缓存已成功生成的最终评论，避免对同一作文重复生成。
*   **备用评语:** 如果整个Agent链执行失败（例如多次API调用失败或最终评论长度仍不合规），系统会根据从Track 1加载的分类结果，生成一个基于预设模板的备用评语，确保总有输出。备用评语也会被调整以符合长度要求。

**5 ID与输出格式**

*   内部处理使用作文原始ID（转为字符串）。
*   最终输出到`DUFL2025_track2.json`文件时，**ID会被重新格式化为从0开始的连续整数**，与Track 1的输出文件保持一致，确保两个任务结果的ID能够对应。

通过这一系列智能体的协同工作与约束，特别是最终环节的严格过滤与精炼，本系统旨在生成高度聚焦于任务要求（切题度与中心思想）、满足所有格式与长度限制的高质量自动化评语。

## Method Report

**Track 1: Automated Essay Relevance Scoring**

**1 Objective and Methodological Overview**

This study aims to develop and evaluate an automated system for the multi-level classification of essay relevance in primary and secondary school student writing, spanning five categories: "Excellent", "Good", "Average", "Qualified", and "Unqualified". To achieve this, we constructed a hybrid architecture integrating a multi-agent collaborative system based on Large Language Models (LLMs) with an auxiliary Bidirectional Encoder Representations from Transformers (BERT) classification module.

**2 LLM Multi-Agent Analysis Framework**

The core of this framework, guided by meticulous prompt engineering, comprises a cascaded processing pipeline of specialized LLM agents (based on the DeepSeek API model, specifically `deepseek-chat`). Each agent sequentially performs a specific analytical task, providing multi-dimensional information for the final score:

1.  **Essay Type Classification Agent (Agent 0):** An initial step added to preliminarily determine the essay's core genre (e.g., narrative, argumentative) based on the prompt, title, and content, providing genre context for subsequent analysis.
2.  **Prompt Requirements Parsing Agent (Agent 1: Prompt Precision Analyst):** Precisely parses the explicit and implicit instructions of the essay prompt, incorporating the identified genre type, to establish standardized relevance assessment benchmarks.
3.  **Essay Theme Extraction Agent (Agent 2: Essay Essence Extractor):** Extracts the core thesis, narrative thread, or descriptive focus from the essay text and assesses the clarity of the main idea's expression and its alignment with the title.
4.  **Content-Task Alignment Verification Agent (Agent 3: Relevance Alignment Verifier):** Rigorously verifies the alignment between the essay content and the established prompt requirements (based on outputs from the preceding two agents), identifying key points of congruence and divergence, and providing a preliminary alignment rating.
5.  **Material-Theme Coherence Auditing Agent (Agent 4: Material-Theme Alignment Auditor):** Evaluates the effectiveness and relevance of supporting materials (e.g., examples, descriptions, details) in bolstering the core theme and addressing prompt requirements, providing a rating of material effectiveness.
6.  **Final Adjudication Agent (Agent 5: Final Relevance Adjudicator):** This agent integrates all preceding analysis reports (from Agents 1-4) and, based on the official detailed scoring rubric specifications, performs a **highly rigorous adjudication, emphasizing boundary distinctions**, generating a "Final Relevance Adjudication Report" containing a clear rating and comprehensive justification. This report serves as the primary source for the LLM framework's classification decision.
7.  **Classification Extraction Agent (Agent 6: Final Classification Extractor):** Ensures the accurate and clean extraction of the final single classification label (e.g., "Good") from the Final Adjudication Agent's report via a dedicated LLM call.

**3 BERT Auxiliary Classification Module**

To enhance the robustness and accuracy of the system's assessment, we incorporated an **enhanced BERT architecture (EnhancedBertEssayModel)** based on the `bert-base-chinese` pre-trained model. This model includes Multi-Head Self-Attention, Feature Fusion Modules, a Bidirectional LSTM layer, and an advanced feature extraction network. It processes encoded representations derived from the essay's prompt, title, content, and grade level. Notably, the BERT module is trained or fine-tuned using an **unsupervised learning** paradigm:

*   **Training Data:** The system first attempts to load pre-trained model weights from a local file (`bert_essay_model.pt`). If loading fails or the file doesn't exist, it utilizes **the current batch of test data itself** (`test_data.json`) for unsupervised training.
*   **Training Method:** It employs a **Contrastive Loss** function, aiming to teach the model to distinguish between the semantic representations of different essays, thereby capturing deep features relevant to relevance even without explicit labels.
*   **Input Processing:** The `AdvancedEssayDataset` handles sophisticated processing and concatenation of input texts (prompt, title, content) to fit model input constraints.

After training, this module independently outputs an auxiliary relevance classification prediction (one of the five levels) for the input essay.

**4 Hybrid Decision-Making and Conflict Arbitration Mechanism**

The system's final classification decision integrates the LLM and BERT outputs and incorporates a specific conflict resolution procedure:

1.  **Primary Basis:** The classification output by the LLM multi-agent framework (specifically, the result extracted by Agent 6 from Agent 5's adjudication) serves as the primary determination.
2.  **Auxiliary Validation:** The BERT module provides an independent,
    auxiliary classification prediction.
3.  **Conflict Detection:** The LLM's adjudicated result is compared with BERT's prediction.
4.  **Conflict Arbitration:**
    *   When the results **disagree**, the system **activates** a specifically designed **LLM Conflict Arbitration Agent**.
    *   This arbitration agent receives the original essay text, prompt requirements, the conflicting classification results from both LLM and BERT, and key text excerpts from the LLM Final Adjudication Agent's (Agent 5) report as input.
    *   The Arbiter's core task is to perform an **independent final adjudication** based on the official scoring rubrics embedded in its prompt, aiming to resolve the classification disagreement and output a final single classification word.
5.  **Adoption Rules:**
    *   If the LLM and BERT assessments are **consistent**, that consistent result is adopted.
    *   If the BERT prediction **fails** (e.g., model failed to load or train), the LLM framework's adjudicated result is directly adopted.
    *   If the Arbitration Agent **fails** to execute or returns an invalid result, the **original adjudicated result from the LLM framework** (extracted by Agent 6) is retained.
    *   In all cases, the system validates the finally adopted classification label for correctness (must be one of the five predefined levels); invalid labels are forced to "Average".

**5 Caching and Result Processing**

*   **Processing Cache:** The system utilizes a `processed_essays.json` file to cache detailed intermediate reports and final results for essays already evaluated, avoiding redundant processing.
*   **ID Handling:** Input data lacking IDs are assigned temporary ones. String IDs are used internally. For the final output file (`DUFL2025_track1.json`), **IDs are reformatted into 0-based sequential integers** to comply with competition submission requirements.

This hybrid approach, combining the deep contextual understanding and multi-faceted analysis capabilities of LLMs with the semantic feature extraction proficiency of BERT, augmented by multiple decision layers (LLM Adjudicator + BERT + Arbiter) and validation mechanisms, aims to achieve more accurate and reliable automated scoring of essay relevance for primary and secondary school students.

---

**Track 2: Automated Relevance-Focused Comment Generation**

**1 Objective and Methodological Overview**

The primary objective of this task is, based on the analysis of an essay's "relevance" and "central theme", to automatically generate evaluative text (comments) that is **strictly focused** on these two aspects, while adhering rigorously to format and length constraints. To this end, we designed and implemented a multi-stage LLM agent pipeline, also based on the DeepSeek API model (`deepseek-chat`).

**2 System Input and Dependencies**

The operation of this system relies not only on the input essay data (prompt, grade, title, content) but also crucially depends on the **final relevance classification results from Track 1** (loaded from `DUFL2025_track1.json`). This classification provides essential guidance for the overall tone and direction of the generated comment.

**3 LLM Comment Generation Pipeline**

This pipeline involves the sequential execution of the following LLM agents:

1.  **Essay Type Classification Agent (Agent 0):** Similar to Track 1, it first identifies the essay's primary genre, providing context for subsequent analysis.
2.  **Prompt Requirements Analysis Agent (Agent 1):** Re-analyzes the prompt requirements (incorporating genre) to establish a precise contextual reference for comment content.
3.  **Essay Theme Extraction Agent (Agent 2):** Extracts the actual core viewpoint or narrative focus expressed in the essay.
4.  **Relevance Assessment Agent (Agent 3):** Generates **descriptive text evaluating the essay's relevance performance** based on the analyses of Agents 1 and 2. *Note: This stage outputs a natural language assessment describing how the essay aligns or deviates regarding relevance, not a classification label.*
5.  **Draft Comment Generation Agent (Agent 4):** This is the core content generator for the comment. It receives:
    *   The descriptive relevance text from Agent 3.
    *   The **final classification result** for the essay loaded from `DUFL2025_track1.json` (e.g., "Good", "Qualified").
    *   The identified essay type from Agent 0.
    Its prompts are strictly constrained to ensure the generated draft:
    *   **(a) Absolute Thematic Focus:** Comment content is **rigorously limited** to discussing the essay's "relevance" (e.g., adherence to prompt, scope) and "central theme expression" (e.g., clarity, alignment with prompt's intent).
    *   **(b) Tone Consistency:** The overall evaluative tone (positive, neutral, critical) must align perfectly with the input **Track 1 final classification result**.
    *   **(c) Targeted Suggestion:** If the input classification is *not* "Excellent", it **must** provide **one** specific, actionable suggestion **solely focused on how to improve "relevance" or "central theme alignment"**. Suggestions must be original, not boilerplate. "Excellent" essays receive primarily affirmation.
    *   **(d) Length Guideline:** Aims to generate text roughly within the **250 to 300 character** range (initial target).
6.  **Compliance Checking and Refinement Agent (Agent 5):** As the final and most critical quality control stage, this agent performs multiple, stringent validation and optimization steps on the draft comment from Agent 4:
    *   **Content Constraint Validation:** **Mandatorily and thoroughly removes** any content pertaining to language use, structure, material details, creativity, emotion, mechanics, length, or any other aspect **unrelated to relevance or the central theme**. This is the **paramount and strictest** rule.
    *   **Content Safety Check:** Ensures the comment is appropriate, positive, and free from sensitive or inappropriate content.
    *   **Format Standardization:** Eliminates all Markdown markup, special symbols, non-《》quotation marks, list markers, unnecessary parentheses, line breaks, etc., ensuring the output is a **single, continuous plain text paragraph**.
    *   **Strict Length Constraint Enforcement:** Rigorously ensures the final comment's total character count (excluding punctuation) falls **precisely within the 200 to 250 character** target range. If too short, it expands relevant descriptions; if too long, it condenses language, prioritizing core feedback. The code emphasizes "must ensure no less than 200 characters" multiple times.
    *   **Classification Consistency Review:** Re-verifies that the comment's evaluative stance fully aligns with the input **Track 1 final classification label**.
    *   **Professional Language Polishing:** Enhances linguistic accuracy, fluency, and professionalism, adhering to the norms of secondary school essay feedback, while satisfying all preceding constraints.

**4 Caching and Fallback Mechanism**

*   **Caching:** The system uses `processed_comments.json` to cache successfully generated final comments, preventing redundant generation for the same essay.
*   **Fallback Comment:** If the entire agent pipeline fails (e.g., persistent API errors or final comment still non-compliant after checks), the system generates a fallback comment based on pre-defined templates corresponding to the Track 1 classification result, ensuring an output is always produced. This fallback comment is also length-adjusted.

**5 ID and Output Formatting**

*   Internal processing uses the essay's original ID (converted to string).
*   For the final output file (`DUFL2025_track2.json`), **IDs are reformatted into 0-based sequential integers**, identical to the Track 1 output file, ensuring correspondence between the results of the two tasks.

Through the collaborative work of this agent pipeline and its constraints, particularly the stringent filtering and refinement applied in the final stage, this system is designed to generate high-quality, automated comments that are highly focused on the specified task requirements (relevance and central theme) and meet all formatting and length specifications.

---