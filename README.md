# Pressure Ulcer Staging Project

## 1. Project Title

Pressure Ulcer Staging with Detection, Segmentation, and Classification  
基于目标检测、病灶分割与细粒度分类的压疮分期系统

---

## 2. Project Overview

本项目旨在构建一个面向 **压疮分期（Pressure Ulcer Staging）** 的自动识别系统，输入为伤口图像，输出为压疮分期结果。

项目的核心目标不是只做“单模型分类”，而是构建一个更符合临床逻辑的三阶段流水线：

1. **Detection**：先定位伤口区域，减少背景干扰。
2. **Segmentation**：再分割病灶及组织区域，突出关键结构信息。
3. **Classification**：最后根据区域结构、纹理、深度特征进行压疮分期。

本项目最终目标为实现 **5 分类任务**：

- Stage 1
- Stage 2
- Stage 3
- Stage 4
- Other Wounds / Non-Pressure-Ulcer

---

## 3. Motivation

在仅使用单一检测或单一分类模型时，模型通常更依赖颜色、纹理和局部外观特征，因此容易出现以下问题：

- Stage 2 与擦伤、浅表创口混淆
- Stage 3 与坏死、结痂或其他深部伤口混淆
- “Other wounds” 类别加入后整体准确率下降
- 背景、拍摄角度、光照差异会影响模型判断

因此，本项目采用分阶段架构，先定位，再分割，再分类，以提升模型对复杂伤口的鲁棒性与可解释性。

---

## 4. Final Architecture

最终采用如下架构：

```text
Original Image
    ↓
YOLOv11/v12
    ↓
Wound ROI (Bounding Box / Cropped Region)
    ↓
MedSAM / SAM 2
    ↓
Lesion / Tissue Mask
    ↓
Swin-Tiny / Swin-UNet
    ↓
Final 5-Class Prediction
```

### Module Roles

- **YOLO**：负责检测伤口区域，输出 bounding box 与 cropped ROI
- **SAM / MedSAM**：负责分割病灶区域，输出 mask 或组织区域图
- **Swin Transformer**：负责结合 ROI 与分割结果进行最终分类
- **Fusion Layer（可选）**：融合 detection confidence、mask features、classification logits

---

## 5. Team Split

为了支持并行开发，项目分为两个主要子模块。

### Member A: Detection Lead

负责 **YOLO 检测模块**：

- 数据框标注（bounding box annotation）
- YOLO 数据集整理与训练
- 检测模型调参
- ROI 自动裁剪
- 输出 metadata 文件
- 检测评估（mAP、Precision、Recall）

### Member B: Segmentation + Classification Lead

负责 **SAM + Swin 模块**：

- 接收 ROI 图像
- 实现病灶分割
- 构建 ROI + mask 输入
- 训练分类模型
- 设计特征融合方案
- 分类评估（Accuracy、F1、Confusion Matrix）

### Shared Tasks

两人共同负责：

- 数据清洗与类别定义
- train / val / test 划分
- 接口协议确认
- 端到端 pipeline 整合
- 实验结果分析
- 论文撰写与答辩材料准备

---

## 6. Work Split Details

| Task | Member A | Member B | Notes |
|---|---|---|---|
| Dataset collection / cleaning | ✅ | ✅ | 一起确认标签质量 |
| Bounding box annotation | ✅ |  | YOLO 训练所需 |
| ROI detection model | ✅ |  | 输出 bbox 与 crop |
| ROI export script | ✅ |  | 作为接口交付 |
| Segmentation model |  | ✅ | MedSAM / SAM 2 |
| Classification model |  | ✅ | Swin-Tiny / Swin-UNet |
| Feature fusion |  | ✅ | 可做增强实验 |
| End-to-end integration | ✅ | ✅ | 最后联调 |
| Metrics & plots | ✅ | ✅ | 分别负责自己模块 |
| Paper writing | ✅ | ✅ | 分章节合写 |

---

## 7. Interface Agreement

为了避免后期返工，必须在项目早期统一输入输出格式。

### ROI Naming Convention

```text
{patient_id}_{image_id}_{wound_id}.jpg
```

示例：

```text
P003_IMG12_W01.jpg
```

### Metadata Format

建议每张图输出一个 JSON 记录：

```json
{
  "image_id": "P003_IMG12",
  "wound_id": "W01",
  "bbox": [x, y, w, h],
  "confidence": 0.91,
  "roi_path": "roi/P003_IMG12_W01.jpg",
  "original_path": "images/P003_IMG12.jpg"
}
```

### Folder Layout

```text
dataset/
├── images/
├── labels_yolo/
├── roi/
├── masks/
├── metadata/
│   ├── train.json
│   ├── val.json
│   └── test.json
```

### Standardization Rules

- 图像格式统一为 `.jpg` 或 `.png`
- ROI 尺寸统一 resize 为 `224x224` 或 `256x256`
- train / val / test 固定 random seed
- 类别编号统一，避免不同脚本映射不一致

---

## 8. Dataset Plan

### Target Classes

本项目采用 5 分类：

| Class ID | Label |
|---|---|
| 0 | Stage 1 |
| 1 | Stage 2 |
| 2 | Stage 3 |
| 3 | Stage 4 |
| 4 | Other Wounds |

### Dataset Requirements

建议数据集满足以下条件：

- 每类样本数量尽量平衡
- “Other Wounds” 需覆盖多种非压疮类型
- 图像来源尽量多样，减少背景偏差
- 删除重复图、模糊图、严重遮挡图
- 若同一病人的多张图高度相似，注意避免数据泄漏

### Split Strategy

推荐：

- Train: 80%
- Validation: 10%
- Test: 10%

若样本量较少，可改为：

- Train: 70%
- Validation: 15%
- Test: 15%

同时建议采用 **patient-level split**，避免同一病人的相似图像同时出现在训练集和测试集中。

---

## 9. Model Plan

### 9.1 Detection Module

目标：从原始图像中检测伤口区域，并生成 ROI

可选模型：

- YOLOv11
- YOLOv12

输出：

- bounding box
- confidence score
- cropped ROI

重点优化项：

- 小目标检测能力
- 多伤口图像下的定位稳定性
- 对“其他伤口”也能稳定检出

### 9.2 Segmentation Module

目标：从 ROI 中提取病灶区域与关键组织边界

可选模型：

- MedSAM
- SAM 2
- Mobile-SAM（轻量部署时可选）

输出：

- binary mask
- overlay visualization
- mask area / contour / shape features（可选）

重点优化项：

- 分割边界稳定性
- 低质量图像上的鲁棒性
- 不同组织纹理的表达能力

### 9.3 Classification Module

目标：对 ROI 或 ROI+Mask 输入进行 5 分类

可选模型：

- Swin-Tiny
- Swin-V2-Tiny
- Swin-UNet（若走联合结构）
- ConvNeXt-Tiny（作为 baseline）

输入形式可选：

1. ROI only
2. ROI + mask overlay
3. ROI + masked image
4. ROI + handcrafted features + deep features

重点优化项：

- Stage 2 / Other 的区分
- Stage 3 / Stage 4 的结构特征提取
- 小样本条件下的泛化能力

---

## 10. Experiment Design

### Baseline Experiments

先做简单 baseline，建立对照组：

1. **Baseline A**：YOLO-only 直接 5 分类
2. **Baseline B**：ROI + CNN 分类
3. **Baseline C**：ROI + Swin 分类

### Main Experiments

1. **YOLO + Swin**
2. **YOLO + SAM + CNN**
3. **YOLO + SAM + Swin**
4. **YOLO + MedSAM + Swin**
5. **YOLO + SAM + Swin + Feature Fusion**

### Ablation Studies

建议做以下消融实验：

- 不使用分割
- 使用原始 ROI vs 使用 mask 后 ROI
- Swin vs ConvNeXt
- SAM vs MedSAM
- 有无 Other Wounds 类别
- 不同输入尺寸对结果影响
- 不同数据增强策略对结果影响

---

## 11. Evaluation Metrics

### Detection Metrics

- mAP@0.5
- mAP@0.5:0.95
- Precision
- Recall

### Segmentation Metrics

- IoU
- Dice Coefficient
- Boundary quality（可选）

### Classification Metrics

- Accuracy
- Precision
- Recall
- F1-score
- Confusion Matrix
- Macro-F1
- Per-class sensitivity

### Clinical Relevance Checks

特别关注：

- Stage 2 是否常被误判为 Other
- Stage 3 / Stage 4 是否互相混淆
- Other 类别是否成为“垃圾桶类别”
- 对边界模糊、颜色不典型样本的稳定性

---

## 12. Training Strategy

### Data Augmentation

建议采用：

- Horizontal / vertical flip
- Rotation
- Random crop
- Brightness / contrast jitter
- Gaussian noise
- Blur
- Color normalization

注意事项：

- 医疗图像增强不能破坏病灶结构
- 不要使用会改变临床语义的过强变换
- 尽量统一颜色空间与输入预处理流程

### Handling Class Imbalance

可选方案：

- Weighted loss
- Focal loss
- Oversampling
- Balanced batch sampling
- Mixup / CutMix（需谨慎验证）

---

## 13. Timeline

### Phase 1: Preparation

**Week 1**
- 确定类别定义
- 清洗数据集
- 统一命名规范
- 确认 train/val/test split
- 确认接口格式

### Phase 2: Detection

**Week 2**
- 完成 bounding box 标注
- 训练第一版 YOLO
- 输出 ROI 裁剪脚本
- 做 detection baseline

### Phase 3: Segmentation

**Week 3**
- 在 ROI 上跑通 MedSAM / SAM
- 输出 mask
- 检查可视化质量
- 修正异常样本

### Phase 4: Classification

**Week 4**
- 训练 Swin baseline
- 输入 ROI 进行 5 分类
- 记录 confusion matrix

### Phase 5: Integrated Pipeline

**Week 5**
- 对接 YOLO 与 SAM + Swin
- 实现端到端推理
- 跑主实验与消融实验

### Phase 6: Analysis & Writing

**Week 6**
- 汇总结果
- 绘制表格与图
- 写 Method / Experiment / Discussion
- 准备答辩材料

---

## 14. Deliverables

### Member A Deliverables

- `train_yolo.py`
- `detect.py`
- `export_roi.py`
- YOLO weights
- detection metrics report

### Member B Deliverables

- `segment.py`
- `train_classifier.py`
- `infer_stage.py`
- classification weights
- confusion matrix / ablation results

### Joint Deliverables

- `main_pipeline.py`
- final report / thesis
- presentation slides
- demo screenshots
- experiment tables
- deployment demo（若有）

---

## 15. Suggested Repository Structure

```text
pressure-ulcer-project/
├── README.md
├── data/
│   ├── raw/
│   ├── processed/
│   ├── roi/
│   ├── masks/
│   └── metadata/
├── detection/
│   ├── train_yolo.py
│   ├── detect.py
│   └── export_roi.py
├── segmentation/
│   ├── segment.py
│   └── utils.py
├── classification/
│   ├── train_classifier.py
│   ├── infer_stage.py
│   └── models/
├── pipeline/
│   └── main_pipeline.py
├── experiments/
│   ├── baselines/
│   ├── ablations/
│   └── logs/
├── outputs/
│   ├── visualizations/
│   ├── metrics/
│   └── checkpoints/
└── docs/
    ├── report/
    └── slides/
```

---

## 16. Risks and Mitigation

### Risk 1: Detection quality is unstable

问题：
- YOLO 检测不到小伤口
- 多伤口图像定位不稳定

对策：
- 增加标注质量检查
- 使用更高分辨率训练
- 调整 confidence threshold
- 对 hard cases 进行 targeted augmentation

### Risk 2: Segmentation quality is poor

问题：
- SAM 输出边界不准确
- 医疗图像域偏移明显

对策：
- 优先尝试 MedSAM
- 对 ROI 做预处理
- 对异常样本进行人工检查
- 必要时做轻量微调

### Risk 3: Other class causes performance drop

问题：
- Other 类别太杂，模型难学

对策：
- 细化 Other 的内部来源
- 提高 Other 的数据覆盖面
- 重点分析 confusion matrix
- 使用 weighted loss 或 focal loss

### Risk 4: Integration mismatch

问题：
- 命名不一致
- 文件路径错误
- bbox / ROI / mask 对不上

对策：
- 提前统一接口
- 写数据校验脚本
- 每周联调一次
- 维护统一版本说明

---

## 17. Paper Writing Plan

### Suggested Chapter Split

#### Member A writes

- Introduction（部分）
- Dataset and Detection Method
- YOLO experiment setup
- Detection results

#### Member B writes

- Segmentation and Classification Method
- SAM / Swin architecture
- Classification results
- Ablation study

#### Joint writing

- Abstract
- Related Work
- Overall Method
- Discussion
- Conclusion
- Future Work

### Figures to Prepare

- Overall pipeline diagram
- Dataset class examples
- YOLO detection examples
- SAM segmentation overlays
- Confusion matrix
- Performance comparison table
- Ablation table

---

## 18. Weekly Sync Checklist

每周同步时检查以下内容：

- 当前训练是否正常
- 数据格式是否一致
- 是否有新增错误样本
- 结果是否写入实验记录
- 是否需要修改接口
- 是否有新 baseline 要补
- 本周论文材料是否同步更新

建议固定每周一次短会，统一更新：

- 当前最好模型
- 当前最大问题
- 下周目标
- 需要对方配合的事项

---

## 19. Success Criteria

项目成功的最低标准：

- 能稳定输出 5 分类结果
- 有完整的 detection → segmentation → classification pipeline
- 有清晰的 baseline 与 ablation
- 能证明加入 segmentation 后性能有提升
- 能解释 “Other Wounds” 的混淆问题
- 论文结构完整，可用于毕设或投稿基础版本

项目优秀标准：

- 指标明显优于单一 baseline
- confusion matrix 显示主要混淆显著减少
- 有良好的可视化结果
- 有完整实验逻辑与工程复现性
- 论文方法章节与实验章节足够扎实

---

## 20. Next Actions

### Immediate Actions

1. 确认最终类别定义
2. 确认数据集划分方式
3. 确认 ROI 输出格式
4. Member A 开始 YOLO detection training
5. Member B 用 mock ROI 先搭 segmentation + classification pipeline
6. 一周后完成第一次接口联调

### Short-Term Goals

- 先跑通 baseline
- 再做完整三阶段架构
- 最后做消融实验与论文包装

---

## 21. Notes

- 所有实验都要记录参数、版本、随机种子与结果
- 每次改动模型结构后都要更新实验日志
- 不要等到最后才开始写论文
- 任何模块完成后，都应尽早输出可视化结果检查
- 工程接口规范比“临时跑通”更重要

---

## 22. Contact / Ownership

- Detection Owner: `[Zhenkai]`
- Segmentation + Classification Owner: `[Yiyang]`
- Project Repo Manager: `[Yiyang]`
- Paper Editor: `[Yiyang]`

---
