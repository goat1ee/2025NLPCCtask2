# EOTGC
NLPCC2025 Evaluation of Essay On-Topic Graded Comments
## Statement
The data obtained during the competition is **strictly limited** to use within the scope of this competition and is prohibited from being used for commercial purposes. For research-related inquiries, please contact the organizers.
## Contact Information
Dong Haoxiang(East China Normal University, hx_dong@stu.ecnu.edu.cn)
## Task content
Essay writing is a crucial component of middle school Chinese education, serving not only as a reflection of students' language expression abilities but also as a substantial manifestation of their logical thinking and comprehension skills. Relevance to the topic, one of the key dimensions in essay evaluation, directly correlates with students' ability to analyze problems and organize their thoughts. It also serves as a critical reference point for teachers in conducting writing instruction and guidance. Relevance assesses whether students accurately understand the topic and can develop their writing around the requirements of the prompt. On one hand, it requires students to closely adhere to the topic content or task objectives in their writing, avoiding digressions or superficial discussions. On the other hand, it demands that students accurately grasp the implied direction or perspective of the topic, thereby demonstrating a high level of comprehension. Additionally, based on students' performance in relevance, teachers can design more targeted teaching plans to help students strengthen their comprehension skills and the logical structure of their essays.

With the widespread application of natural language processing (NLP) technology in the field of education, an increasing number of automated essay analysis tools are being used to support writing instruction and assessment. Some early studies focused on identifying the central idea from the entire text of essays, while others attempted to analyze the paragraph structure and logical coherence of essays. However, there remains significant research potential in accurately evaluating essay relevance through automated methods. Relevance not only requires considering the content connection between the essay and the topic but also analyzing whether the language expression clearly revolves around the topic. Therefore, it is necessary to define more refined relevance evaluation criteria and analyze the relevance between the essay and the topic by combining semantic information at the sentence and paragraph levels.

This evaluation task focuses on primary and secondary school students' essays, with an emphasis on relevance evaluation and the generation of comments based on relevance analysis. Specifically, the task will involve multi-dimensional scoring of whether the essay closely adheres to the topic, including dimensions such as the accuracy of topic comprehension, the relevance and completeness of the content, and the automatic generation of appropriate comments based on relevance analysis to assist teachers in essay evaluation and instructional guidance. This research endeavor aims to establish a unified standard for relevance evaluation and provide technical support for automated essay scoring and teaching.
## Test data samples
[Link](https://github.com/dong12003/Test/blob/33fef758fe09ea1eb8bdb10fbfddb003bfa55c0f/data_sample.md)
## Track 1

### Task Description
**Relevance scoring of essays.**  Assess whether the essay aligns with the given topic, providing classification results and predicted scores. The classification results include "Excellent," "Good," "Average," "Pass," and "Fail."

### Task Definition
Essay relevance scoring is regarded as a multi-classification problem, with the core task being to evaluate the relevance of an essay, predict its relevance category, and generate corresponding scores. This evaluation task defines five relevance categories: "Excellent," "Good," "Average," "Pass," and "Fail." Specific category definitions and scoring criteria are provided in the table below.
<table align="center">
  <tr>
    <th width="20%" align="center">Relevance Level</th>
    <th width="80%" align="center">Description Standards for Relevance Levels</th>
  </tr>
  <tr>
    <td align="center">Excellent(优秀)</td>
    <td>The article perfectly meets the requirements of the composition. The topic selection is extremely appropriate, with a distinct and prominent central idea. The selected materials are just right, complementing each other and the theme.</td>
  </tr>
  <tr>
    <td align="center">Good(较好)</td>
    <td>The article fairly well meets the requirements of the composition. The topic selection is quite appropriate, and the central idea is relatively clear. The chosen materials are also rather fitting, working in harmony with the theme.</td>
  </tr>
  <tr>
    <td align="center">Average(一般)</td>
    <td>The article generally meets the requirements of the topic selection. The topic is moderately appropriate, and the central idea is somewhat clear. The selected materials basically meet the needs of the theme, and the overall performance is acceptable.</td>
  </tr>
  <tr>
    <td align="center">Qualified(合格)</td>
    <td>Although the article does not deviate from the theme, the topic selection is a bit less appropriate. The central idea is not prominent, and it is somewhat difficult to extract. The selected materials also have a low degree of matching with the theme. Overall, the applicability of the materials needs to be improved.</td>
  </tr>
  <tr>
    <td align="center">Unqualified(不合格)</td>
    <td>The article seriously deviates from the requirements of the topic selection. The topic is extremely inappropriate. The central idea is vague and hard to grasp. The selected materials are completely different from the theme and do not match at all.</td>
  </tr>
</table>

### Test Data Format
```json
[
    {
        "id": 0,
        "grade": "7",
        "requirement": "...",
        "title": "...",
        "content": "..."
    }
]
```
**Where "id" is the number of the test data, "grade" is the grade information (7 represents Grade 7), "requirement" is the composition question stem, "title" is the composition title, and "content" is the composition content.**

### Result Data Format
```json
[
    {
        "id": 0,
        "classification": "..."
    }
]
```
**Where "id" is the number of the test data, and "classification" is the classification result (outstanding, good, average, qualified, unqualified).**
### Evaluation Dataset

<table align="center">
  <tr>
    <th align="center">Grade</th>
    <th align="center">3</th>
    <th align="center">4</th>
    <th align="center">5</th>
    <th align="center">6</th>
    <th align="center">7</th>
    <th align="center">8</th>
    <th align="center">9</th>
  </tr>
  <tr>
    <td align="center">Number of essay questions</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
  </tr>
  <tr>
    <td align="center">Number of essays</td>
    <td align="center">5/5</td>
    <td align="center">5/5</td>
    <td align="center">5/5</td>
    <td align="center">5/5</td>
    <td align="center">20/26</td>
    <td align="center">25/12</td>
    <td align="center">32/30</td>
  </tr>
</table>

The test dataset consists of 185 essays, with grades ranging from 3rd to 9th grade. Each grade has two prompts. For instance, for the 3rd grade, the notation 5/5 signifies that there are 5 essays corresponding to the first writing prompt and 5 essays for the second prompt within the test dataset. A similar pattern is observed across the other grades.

### Evaluation Criteria
- **Approximate Accuracy**

$$ACC_A = \frac{1}{N} \sum_{i=1}^{N} S(d_i)$$

This metric measures the prediction accuracy of the model by considering the magnitude of the error between the predicted results and the true labels. Here, N is the number of samples, 
$$S(d_i) = (1 - \frac{d_i}{4})$$
 , 
$$d_i = \lvert y_i - \hat{y}_i \rvert$$
 , 
$$y_i$$
 is the true label, and 
$$\hat{y}_i$$
 is the predicted result.

- **Pearson Correlation Coefficient**

$$r = \frac{\sum (y_i - \bar{y})(\hat{y}_i - \bar{\hat{y}})}{\sqrt{\sum (y_i - \bar{y})^2} \sqrt{\sum (\hat{y}_i - \bar{\hat{y}})^2}}$$

This metric measures the linear relationship between the predicted results and the true labels. Here, 
$$y_i$$
 is the true label, 
$$\bar{y}$$
 is the mean of the true labels, 
$$\hat{y}_i$$
 is the predicted result, and 
$$\bar{\hat{y}}$$
 is the mean of the predicted results.

- **Final Score**

$$Score = 0.5 * ACC_A + 0.5 * (\frac{1+r}{2})$$

The final score for Task 1 is calculated by normalizing the approximate accuracy and Pearson correlation coefficient, weighting each by 0.5, and summing them.
## Track 2

### Task Description
**Generate relevance comments.** Generate comments based on whether the essay is relevant to the topic, focusing solely on the central idea of the essay and its relevance to the given topic.

### Task Definition
Generating relevance comments is regarded as considered a generation task, with the core task being to generate comments on whether the essay aligns with the prompt and the central idea. The generated comments should meet the following conditions:
1. Provide graded comments on the essay's central idea and relevance based on the "essay requirements" for the corresponding grade's writing training.
2. The generated comments must not involve politics, race, religion, ethnicity, violence, discrimination, or other sensitive information, and must not violate relevant laws and policies.
3. The generated comments must not exceed 250 characters.
### Test Data Format
```json
[
    {
        "id": 0,
        "grade": "7",
        "requirement": "...",
        "title": "...",
        "content": "..."
    }
]
```
**Where "id" is the number of the test data, "grade" is the grade information (7 represents Grade 7), "requirement" is the composition question stem, "title" is the composition title, and "content" is the composition content.**

### Result Data Format
```json
[
    {
        "id": 0,
        "comment": "..."
    }
]
```
Where "id" is the number of the test data, and "comment" is the comment on the composition.

### Evaluation Dataset

<table align="center">
  <tr>
    <th align="center">Grade</th>
    <th align="center">3</th>
    <th align="center">4</th>
    <th align="center">5</th>
    <th align="center">6</th>
    <th align="center">7</th>
    <th align="center">8</th>
    <th align="center">9</th>
  </tr>
  <tr>
    <td align="center">Number of essay questions</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
    <td align="center">2</td>
  </tr>
  <tr>
    <td align="center">Number of essays</td>
    <td align="center">5/5</td>
    <td align="center">5/5</td>
    <td align="center">5/5</td>
    <td align="center">5/5</td>
    <td align="center">20/26</td>
    <td align="center">25/12</td>
    <td align="center">32/30</td>
  </tr>
</table>

The test dataset consists of 185 essays, with grades ranging from 3rd to 9th grade. Each grade has two prompts. For instance, for the 3rd grade, the notation 5/5 signifies that there are 5 essays corresponding to the first writing prompt and 5 essays for the second prompt within the test dataset. A similar pattern is observed across the other grades.

### Evaluation Criteria
- **PPLscore**

$$PPL = \exp\left(-\frac{1}{N}\sum_{i = 1}^{N} \log P(w_i|w_1, w_2, \ldots, w_{i - 1})\right)$$

PPLscore is used to measure the model's predictive ability for a given text sequence. In the formula, N represents the number of words in the test text, 
$$w_i$$
 is the i-th word in the sequence, and 
$$P(w_i|w_1, w_2, \ldots, w_{i - 1})$$
 represents the model's predicted probability for the i-th word.

For PPLscore, we use the baichuan-7b-chat model for calculation.
- **Bertscore**

- **human**
In this evaluation process, experts in primary and secondary school essay writing to manually score a fixed selection of essays from each grade at a 1:10 ratio, with the average score serving as the human score.
- **Final Score**

$$Score = 0.10 * \frac{\frac{1}{PPLscore} - 0.02}{0.18} + 0.40 * Bertscore + 0.5 * \frac{human}{100}$$

The final score is calculated by normalizing the PPLscore and weighting it by 0.1, adding the Bertscore weighted by 0.4, and adding the normalized human score weighted by 0.5.
