## 方法报告

### Track 1: 作文切题度自动评分

**1 研究目标与方法概述**

本研究旨在开发并评估一种自动化系统，用于对中小学生作文的切题度进行多级别分类（涵盖“优秀”、“较好”、“一般”、“合格”、“不合格”五个等级）。为实现此目标，我们构建了一种混合架构，该架构融合了基于大型语言模型（LLM）的多智能体协作系统与一个辅助性的BERT（Bidirectional Encoder Representations from Transformers）分类模块。该系统利用了测试数据自身的特点进行模型微调和评估。

**2 LLM多智能体分析框架**

该框架的核心是一个通过精心设计的提示工程（Prompt Engineering）引导的、由多个功能专一的LLM智能体（基于`deepseek-chat`模型）组成的级联处理流程。输入数据包括作文要求、年级、标题和内容。各智能体按序执行特定分析任务，为最终评分提供多维度信息，部分智能体运用了从`samples.json`加载的Few-Shot样本进行引导：

1.  **作文类型识别智能体 (Agent 0)**：首先分析作文的体裁类型（如记叙文、议论文等），为后续分析设定文体背景。*(使用了Few-Shot样本)*
2.  **题目要求解析智能体 (Agent 1)**：负责精确解析作文题目的显性与隐性指令，建立规范化的切题度评估基准。
3.  **文章主旨提取智能体 (Agent 2)**：负责从作文文本中提炼其核心论题或记叙主线，并评估主旨表达的明确性。
4.  **内容-任务符合度校验智能体 (Agent 3)**：基于前两个智能体的输出，严谨校验作文内容与既定题目要求之间的符合度，识别关键的契合点与偏离点。
5.  **素材-主题契合度审核智能体 (Agent 4)**：评估作文中所选用的素材（如事例、细节描写）支撑核心主旨及回应题目要求的有效性与相关性。*(使用了Few-Shot样本)*
6.  **综合裁决智能体 (Agent 5)**：整合上述所有分析报告，依据官方评分标准细则，生成初步的切题度分类（优秀/较好/一般/合格/不合格）。*(使用了Few-Shot样本)*
7.  **分类结果提取智能体 (Agent 6)**：从综合裁决报告中精确提取最终的分类标签（五个等级词之一）。

**3 BERT辅助分类模块**

为增强系统评估的鲁棒性，我们引入了一个基于`bert-base-chinese`预训练模型的增强型BERT架构 (`EnhancedBertEssayModel`)。该模型包含多头自注意力机制、针对题目、标题、内容分别处理的编码头、特征融合模块、双向LSTM层以及最终的分类层。

*   **输入处理**：`AdvancedEssayDataset`类负责将作文要求、标题和内容进行分段编码和填充，然后拼接成统一的输入序列，送入BERT模型。
*   **无监督训练**：采用无监督学习范式，利用测试集数据 (`test_data.json`，经过ID处理和有效性筛选后)，通过对比损失函数 (`ContrastiveLoss`, InfoNCE风格) 对模型进行训练。训练目标是优化BERT提取的特征表示，使其能更好地区分不同作文的语义。函数`load_or_train_bert_model_unsupervised`负责加载或训练此模型。
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
    *   对最终确定的分类进行有效性检查，若无效则强制设为“一般”。
5.  **ID格式化**：最终将所有作文的分类结果按顺序（从0开始）格式化ID，并保存到`DUFL2025_track1.json`文件中。

该混合方法通过结合LLM的深度语境理解、上下文分析能力与BERT基于无监督学习适应测试数据的语义特征提取能力，并引入冲突仲裁层，旨在实现更准确、更可靠的作文切题度自动评估。

---

### Track 2: 相关性评论自动生成

**1 研究目标与方法概述**

本任务的核心目标是基于作文的切题度分析，自动生成一段仅聚焦于作文中心思想及其与题目要求相关性的评价性文本（评语），且评语长度严格控制在小于等于250字符。为此，我们设计并实现了一个多阶段的LLM智能体流水线（Pipeline），同样基于`deepseek-chat`模型。

**2 系统输入与依赖**

该系统的运行不仅依赖于输入的作文数据（题目要求、年级、标题、内容来自于`test_data.json`），亦关键性地利用了Track 1任务输出的最终切题度分类结果 (从`DUFL2025_track1.json`加载) 作为指导信息。同时，系统也加载`samples.json`文件，将其中的样本用于指导部分智能体的行为（Few-Shot Learning）。

**3 LLM评论生成流水线**

该流水线包含以下按序执行的LLM智能体：

1.  **作文类型识别智能体 (Agent 0)**：与Track 1类似，首先识别作文类型，为后续评语生成提供文体背景参考。*(使用了Few-Shot样本)*
2.  **题目要求分析智能体 (Agent 1)**：再次对题目要求进行分析，为后续评语生成提供精确的上下文参照。
3.  **作文主旨提取智能体 (Agent 2)**：提取作文的核心观点或记叙主旨。
4.  **切题度评估智能体 (Agent 3)**：基于Agent 1和Agent 2的分析结果，生成一段关于作文切题表现的描述性文本（此阶段非输出分类标签，而是文本化评估）。*(使用了Few-Shot样本)*
5.  **初步评语草稿生成智能体 (Agent 4)**：此智能体结合Agent 3输出的切题度描述性文本与Track 1的最终分类结果 (`predicted_classification`)，生成初步评语草稿。其提示被严格约束，确保：(a) 评语内容严格限定于切题性及中心思想表达；(b) 整体评价基调与Track 1分类结果保持一致；(c) 若切题度评估结果非“优秀”，则提供仅针对提升切题性的具体改进建议；(d) 参考Few-Shot样本的风格和内容。*(使用了Few-Shot样本)*
6.  **合规性检查与精炼智能体 (Agent 5)**：作为流水线的最后关键环节，此智能体对草稿评语执行多重验证与优化，并参考Few-Shot样本：
    *   **内容约束验证**: 强制移除任何涉及语言风格、结构逻辑、素材细节、修辞手法、错别字、篇幅等非切题性/非中心思想相关的所有内容。
    *   **格式规范化**: 清除所有Markdown标记、特殊符号、不必要的标点及格式（如换行符），确保输出为单一、连续的纯文本段落。
    *   **长度约束执行**: 严格确保最终评语的总字符数小于等于250字符且非空。如果输出过长，则进行精简；如果过短，则在不违反内容约束的前提下尝试补充。
    *   **分类一致性复核**: 再次确认评语的评价倾向与Track 1的输入分类标签完全吻合。
    *   **语言专业性润色**: 在满足所有约束的前提下，提升语言表达的准确性、流畅度与专业性，使其符合中小学作文评语规范。*(使用了Few-Shot样本)*

**4 缓存与容错**

系统实现了评论缓存机制 (`processed_comments.json`)，避免重复生成。如果在多轮重试后，智能体流水线仍无法生成符合要求的评论（特别是长度要求），系统会根据Track 1的分类结果生成一条预设的、符合长度要求的备用评语 (`backup_comment`)。

**5 输出格式化**

最终，系统将所有作文的生成评论与Track 1的ID（0, 1, 2...）进行匹配，并保存到`DUFL2025_track2.json`文件中，确保输出ID与Track 1一致。

通过这一系列智能体的协同工作，特别是最终环节的严格过滤与精炼，本系统旨在生成高度聚焦于任务要求（切题度与中心思想）、满足所有格式与长度限制（<=250字符）的高质量自动化评语。

---

## Method Report

### Track 1: Automated Essay Relevance Scoring

**1 Objective and Methodological Overview**

This study aims to develop and evaluate an automated system for the multi-level classification of essay relevance in primary and secondary school student writing, covering five categories: "Excellent", "Good", "Average", "Qualified", and "Unqualified". To achieve this, we constructed a hybrid architecture integrating a multi-agent collaborative system based on Large Language Models (LLMs) with an auxiliary Bidirectional Encoder Representations from Transformers (BERT) classification module. The system leverages the characteristics of the test data itself for model fine-tuning and evaluation.

**2 LLM Multi-Agent Analysis Framework**

The core of this framework, guided by meticulous prompt engineering, comprises a cascaded processing pipeline of specialized LLM agents (based on the `deepseek-chat` model). Input data includes the essay prompt, grade, title, and content. Each agent sequentially performs a specific analytical task, providing multi-dimensional information for the final score. Some agents utilize Few-Shot samples loaded from `samples.json` for guidance:

1.  **Essay Type Classification Agent (Agent 0)**: Initially analyzes the essay's genre (e.g., narrative, argumentative), setting the context for subsequent analysis. *(Utilizes Few-Shot samples)*
2.  **Prompt Requirements Parsing Agent (Agent 1)**: Precisely parses the explicit and implicit instructions of the essay prompt to establish standardized relevance assessment benchmarks.
3.  **Essay Theme Extraction Agent (Agent 2)**: Extracts the core thesis or narrative thread from the essay text and assesses the clarity of the main idea's expression.
4.  **Content-Task Alignment Verification Agent (Agent 3)**: Rigorously verifies the alignment between the essay content and the established prompt requirements (based on outputs from the preceding two agents), identifying key points of congruence and divergence.
5.  **Material-Theme Coherence Auditing Agent (Agent 4)**: Evaluates the effectiveness and relevance of supporting materials (e.g., examples, descriptive details) in bolstering the core theme and addressing prompt requirements. *(Utilizes Few-Shot samples)*
6.  **Final Adjudication Agent (Agent 5)**: Integrates all preceding analysis reports and, based on official scoring rubric specifications, generates a preliminary relevance classification (Excellent/Good/Average/Qualified/Unqualified). *(Utilizes Few-Shot samples)*
7.  **Classification Extraction Agent (Agent 6)**: Accurately extracts the final classification label (one of the five category words) from the Final Adjudication Agent's report.

**3 BERT Auxiliary Classification Module**

To enhance the robustness of the system's assessment, we incorporated an enhanced BERT architecture (`EnhancedBertEssayModel`) based on the `bert-base-chinese` pre-trained model. This model includes multi-head self-attention, separate encoding heads for prompt, title, and content, feature fusion modules, a bidirectional LSTM layer, and a final classification layer.

*   **Input Processing**: The `AdvancedEssayDataset` class handles segment encoding, padding, and concatenation of the prompt, title, and content into a unified input sequence for the BERT model.
*   **Unsupervised Training**: An unsupervised learning paradigm is employed, utilizing the test set data (`test_data.json`, after ID processing and validity checks) to train the model via a contrastive loss function (`ContrastiveLoss`, InfoNCE style). The training aims to optimize the feature representations extracted by BERT, enabling better discrimination between the semantics of different essays. The `load_or_train_bert_model_unsupervised` function manages loading or training this model.
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
    *   The final determined classification is validated; if invalid, it defaults to "Average".
5.  **ID Formatting**: Finally, the classification results for all essays are assigned sequential IDs starting from 0 and saved to the `DUFL2025_track1.json` file.

This hybrid approach, combining the deep contextual understanding and analytical capabilities of LLMs with the semantic feature extraction proficiency of BERT (adapted to the test data via unsupervised learning), augmented by a conflict resolution layer, aims to achieve more accurate and reliable automated essay relevance scoring.

---

### Track 2: Automated Relevance-Focused Comment Generation

**1 Objective and Methodological Overview**

The primary objective of this task is, based on the essay's relevance analysis, to automatically generate evaluative text (comments) strictly focused on the essay's central theme and its relevance to the prompt requirements, with a strict length constraint of no more than 250 characters. To this end, we designed and implemented a multi-stage LLM agent pipeline, also based on the `deepseek-chat` model.

**2 System Input and Dependencies**

The operation of this system relies not only on the input essay data (prompt, grade, title, content from `test_data.json`) but also crucially utilizes the final relevance classification results from Track 1 (loaded from `DUFL2025_track1.json`) as guiding information. Additionally, the system loads `samples.json` to provide Few-Shot examples for guiding specific agents.

**3 LLM Comment Generation Pipeline**

This pipeline involves the sequential execution of the following LLM agents:

1.  **Essay Type Classification Agent (Agent 0)**: Similar to Track 1, it first identifies the essay type to provide genre context for comment generation. *(Utilizes Few-Shot samples)*
2.  **Prompt Requirements Analysis Agent (Agent 1)**: Re-analyzes the prompt requirements to provide a precise contextual reference for subsequent comment generation.
3.  **Essay Theme Extraction Agent (Agent 2)**: Extracts the core viewpoint or narrative focus of the essay.
4.  **Relevance Assessment Agent (Agent 3)**: Generates descriptive text evaluating the essay's relevance performance based on the analyses of Agent 1 and Agent 2 (Note: this stage outputs a textual assessment, not a classification label). *(Utilizes Few-Shot samples)*
5.  **Draft Comment Generation Agent (Agent 4)**: This agent combines the descriptive relevance assessment text (from Agent 3) and the final classification result from Track 1 (`predicted_classification`) to generate an initial comment draft. Its prompts are strictly constrained to ensure: (a) comment content is rigorously limited to relevance and central theme expression; (b) the overall evaluative tone aligns with the Track 1 classification result; (c) if the relevance assessment indicates suboptimal performance (e.g., not "Excellent"), it provides specific improvement suggestions solely targeting relevance enhancement; (d) the style and content are informed by Few-Shot samples. *(Utilizes Few-Shot samples)*
6.  **Compliance Checking and Refinement Agent (Agent 5)**: As the critical final stage, this agent performs multiple validation and optimization steps on the draft comment, referencing Few-Shot samples:
    *   **Content Constraint Validation**: Mandates the removal of any content pertaining to language style, structural logic, material details, rhetorical devices, grammatical errors, length, or other aspects unrelated to relevance or the central theme.
    *   **Format Standardization**: Eliminates all Markdown markup, special symbols, extraneous punctuation, and formatting (like newlines), ensuring the output is a single, continuous plain text paragraph.
    *   **Length Constraint Enforcement**: Strictly ensures the final comment's total character count is less than or equal to 250 characters and is non-empty. If the output is too long, it is condensed; attempts are made to supplement overly short comments without violating content constraints.
    *   **Classification Consistency Review**: Re-verifies that the comment's evaluative stance fully aligns with the input classification label from Track 1.
    *   **Professional Language Polishing**: Enhances linguistic accuracy, fluency, and professionalism, adhering to the norms of secondary school essay feedback, while satisfying all preceding constraints. *(Utilizes Few-Shot samples)*

**4 Caching and Fault Tolerance**

The system implements a comment caching mechanism (`processed_comments.json`) to avoid redundant generation. If, after multiple retries, the agent pipeline fails to produce a compliant comment (especially regarding length), the system generates a pre-defined fallback comment (`backup_comment`) based on the Track 1 classification, ensuring it meets the length requirement.

**5 Output Formatting**

Finally, the system matches the generated comments for all essays with the corresponding Track 1 IDs (0, 1, 2...) and saves them to the `DUFL2025_track2.json` file, ensuring output IDs align with Track 1.

Through the collaborative work of this agent pipeline, particularly the stringent filtering and refinement applied in the final stage, the system is designed to generate high-quality, automated comments that are highly focused on the specified task requirements (relevance and central theme) and meet all formatting and length specifications (<=250 characters).
---