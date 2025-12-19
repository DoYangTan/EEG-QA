# EEG-QA: A Dedicated Question Answering Dataset for EEG Signals

## 简介 (Introduction)
**EEG-QA** 是首个面向脑电信号（EEG）的专用问答数据集。该数据集旨在解决传统 EEG 分析中泛化能力不足以及过度依赖人工特征工程的问题。通过构建独特的分层问答结构，EEG-QA 将视觉问答（VQA）框架迁移至脑电信号处理领域，实现了从原始脑电信号到自然语言的智能化解析。

该数据集包含 **16,126** 个样本，涵盖了视觉、语言、运动和多感官等异质任务，并设计了独特的“分层问答树”结构（任务识别 → 阶段定位 → 细节推理）。

## 数据集形式 (Dataset Format)

为了保证跨任务的空间特征一致性，本数据集对电极通道进行了统一处理。

### ⚠️ Critical Note on Channel Selection (关于通道选择的重要说明)
**Please be aware that while unifying the dataset to 24 channels enables cross-task model training, this standardization comes with a potential trade-off: significant information loss.** **请注意：虽然将通道统一为 24 个使得跨任务模型训练成为可能，但这种标准化可能导致大量信息丢失。**
Different cognitive tasks often activate specific brain regions that may require high-density electrode coverage outside the standard 10-20 system (e.g., dense visual or auditory cortex coverage). **By excluding these task-specific channels, some critical signal features relevant to specific stimuli might have been discarded.** Researchers should consider this limitation when analyzing fine-grained details.

### 物理规格 (Physical Specifications)
* **样本数量 (Total Samples)**: 16,126
* **通道数 (Channels)**: **24** (Unified)
    * 所有样本均提取了国际 10-20 系统下的 24 个通用电极。
    * **Electrode List**: `FP1`, `FP2`, `F3`, `F4`, `F7`, `F8`, `FZ`, `FC3`, `FC4`, `FT7`, `FT8`, `C3`, `C4`, `CP3`, `CP4`, `CPZ`, `P3`, `P4`, `PZ`, `TP7`, `TP8`, `O1`, `O2`, `OZ`
* **时间点 (Time Points)**: **Variable** (Not Unified)
    * 每个样本的时间长度保持原始状态，未进行强制归一化，长度取决于具体的实验试次（Trial）时长。

## 文件结构与元数据 (File Structure & Metadata)

数据集由 EEG 源文件（.xlsx）和两个核心 JSON 元数据文件组成，二者通过 ID 相互关联。

### 1. EEG 数据文件
* **格式**: **.xlsx**
* **内容**: 每个文件存储一个样本的 EEG 矩阵（24 行 × T 列）。文件名包含了受试者编号、时间戳和刺激 ID 等原始信息。

### 2. ID 映射文件 (`xlsx_id_map.json`)
该文件用于建立**物理文件名**与**逻辑 EEG ID**之间的映射关系，方便在元数据中引用。
* **Key**: EEG 数据的物理文件名（String）。
* **Value**: 数据集中唯一的 `eeg_id`（Integer）。

```json
{
    "sub-015_start-978704_end-980733_stimid-S 60.xlsx": 16110,
    "sub-015_start-980733_end-981636_stimid-S 11.xlsx": 16111,
    "sub-015_start-981636_end-982744_stimid-S 30.xlsx": 16112,
    "sub-015_start-982744_end-984773_stimid-S 50.xlsx": 16113,
    ...
}
```

### 3. 问答标注文件 (`filter_cap.json`)
该文件存储了所有的 Question-Answer (QA) 对及其关联的属性信息。
* **eeg_id**: 对应 `xlsx_id_map.json` 中的整数 ID，用于关联特定的 EEG 片段。
* **caption**: 包含问题和答案的核心字段。格式为 `<question>问题内容</think>答案`。
* **question_type**: 问题类型（如 `single-verify`）。
* **level**: 任务层级（如 "4"）。
* **task**: 所属的实验任务（如 `associative imagery`）。

```json
[
    {
        "eeg_id": 9377,
        "caption": "<question>Is the Handschuh heard by the experimenter corresponding to this EEG?</think>Yes",
        "question_type": "single-verify",
        "level": "4",
        "task": "associative imagery"
    },
    {
        "eeg_id": 9377,
        "caption": "<question>Is the Fisch heard by the experimenter corresponding to this EEG?</think>No",
        "question_type": "single-verify",
        "level": "4",
        "task": "associative imagery"
    },
    {
        "eeg_id": 9381,
        "caption": "<question>Is the Anzug heard by the experimenter corresponding to this EEG?</think>Yes",
        "question_type": "single-verify",
        "level": "4",
        "task": "associative imagery"
    }
]
```

## 验证与模型适配 (Verification & Model Adaptation)

为了验证 EEG-QA 数据集的有效性并建立基准（Benchmark），我们计划采用 **NeuroLM** (A Universal Multi-task Foundation Model for Human Brain Activities) 的方法论进行后续实验。

### ⚠️ 关于 VQ-VAE 的适配问题 (Adaptation of VQ-VAE)
NeuroLM 依赖于 **VQ-VAE (Vector Quantized Variational Autoencoder)** 将连续的 EEG 信号离散化为 Token，以便 Transformer 模型处理。

在初步实验中，我们发现如果直接加载 NeuroLM 在其海量预训练数据上得到的 **预训练 VQ (Pre-trained VQ)** 权重来处理本数据集（EEG-QA），会出现严重的 **Codebook Collapse (Codebook 死码)** 现象：
* **现象**：由于 EEG-QA 包含高度异质的任务数据（视觉、语言、运动想象、多感官），其信号分布特征与 NeuroLM 原始预训练数据的分布存在显著差异（Domain Shift）。
* **后果**：直接使用预训练 VQ 会导致 **死码率过高 (High Dead Code Rate)**，即 Codebook 中大部分的向量未被激活或利用，导致 EEG 信号的重构质量极差，无法有效捕获本数据集中的细粒度特征。

### 解决方案 (Solution)
因此，为了在 EEG-QA 数据集上获得有效的 Tokenization 效果，**必须使用本数据集重新训练 VQ-VAE 的参数 (Retraining VQ parameters)**。我们需要专门针对这 24 通道的异质数据优化 Codebook，以降低死码率并提高信号重构的保真度，从而为后续的大模型训练奠定基础。

## 任务详情 (Task Details)

EEG-QA 数据集由四个核心实验任务组成，涵盖了视觉、听觉、触觉和心理想象等多种认知活动。

### 1. 图像观看与想象任务 (Image Viewing & Imagination Task)
该任务主要关注大脑在视觉处理及心理意象生成过程中的神经活动。
* **实验流程 (Procedure)**:
    1.  **View Images Stage**: 受试者观看屏幕上显示的图像，持续 6 秒。
    2.  **Imagine Stage**: 图像消失后，受试者需在脑海中尽可能清晰地想象刚才看到的图像内容，持续 6 秒。
* **刺激内容 (Stimuli)**: 包含 340 张不同类别的静态图像。
* **语义标注 (Annotation)**: 利用 VQA 模型为每张图像生成了详细的英文文本描述（Description），作为 Level 3 细节推理任务的 Ground Truth。

### 2. 内在语言理解任务 (Inner Speech Understanding Task)
该任务旨在探索 **Inner Speech**（内心言语）过程中脑电信号的时空关联。
* **实验范式 (Paradigm)**: 每个试次（Trial）包含严格的时间窗口：
    1.  **Fixation** (2s): 注视期。
    2.  **Stimulation** (2s): 屏幕显示词汇，受试者进行 Inner Speech 处理（默读/思考）。
    3.  **Rest** (12s): 休息期。
* **研究目标**: 捕捉大脑在不发声的情况下处理语言信息的微弱电信号变化。

### 3. 运动想象神经关联任务 (Motor Imagery & Neural Correlation Task)
该任务聚焦于 **Motor Imagery**（运动想象）与实际执行的神经机制对比。
* **实验流程 (Procedure)**:
    1.  **Cue**: 提示阶段，告知即将进行的动作。
    2.  **Video / Motor Imagery**: 受试者观看动作视频，并在脑海中模拟该动作（或伴随实际模仿）。
    3.  **Rest**: 休息阶段。
* **关键特征**: 支持模型区分大脑的运动规划状态，并推断具体的肢体部位（如 **Left Hand** vs **Right Hand**）。

### 4. 联想与多感官整合任务 (Association & Multisensory Integration Task)
这是最复杂的任务类型，涉及听觉、触觉和情感的多感官整合。每一轮实验包含三个连续的子阶段：
* **Phase 1: Auditory Association (听觉联想)**
    * 受试者首先听到一个类别提示音，类别为 **Clothing**（衣物）或 **Animals**（动物）。
    * 紧接着听到一个具体的单词（Word），受试者需建立联想。
    * *QA Task*: 识别听到的单词属于 **Clothing** 还是 **Animals**。
* **Phase 2: Tactile Stimulation (触觉刺激)**
    * 对受试者的 **Left Hand** 或 **Right Hand** 施加物理刺激。
    * *QA Task*: 定位受刺激的手部位置。
* **Phase 3: Music Perception (音乐感知)**
    * 受试者聆听一段音乐。
    * 音乐类型分为：**Soothing**（舒缓）或 **Irritating**（刺耳/烦躁）。
    * *QA Task*: 识别受试者感知到的音乐情感类型。

## 分层问答结构 (Hierarchical Q&A Structure)

为了模拟真实的认知分析流程，本数据集的问题设计遵循三级树状结构：

1.  **Level 1 - Task Recognition**: 判断当前 EEG 信号属于上述四个大类任务中的哪一项。
2.  **Level 2 - Stage Localization**: 确定受试者处于实验的哪个具体阶段（例如：**View Images Stage** vs **Imagine Stage**）。
3.  **Level 3 - Detail Inference**: 解析具体的实验内容（例如：“Did the experimenter use the **Left Hand**?” 或 “Is the word related to **Clothing**?”）。
