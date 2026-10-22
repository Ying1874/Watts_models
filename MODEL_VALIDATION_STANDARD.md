# PaddleDetection / PaddleClas 模型验证与分级标准

> 文档版本：V1.2（草案）
> 制定日期：2026-10-22
> 适用范围：车辆面板紧固检测项目中的 PaddleDetection（含 PicoDet）与 PaddleClas 模型

## 1. 目的

本标准用于统一模型在验证集（Validation Set，以下简称 `val`）上的评估方法、指标要求和等级划分。

需要注意：PaddleDetection 和 PaddleClas 官方提供评估工具与指标定义，但不规定统一的模型合格阈值。本文只评价模型验证结果，不包含主控逻辑、现场生产阈值、板端速度或最终系统放行标准。

## 2. 等级定义

| 等级 | 评价 | 使用范围 |
|---|---|---|
| S | 优秀 | 可作为模型基准版本 |
| A | 合格 | 可作为正式候选模型 |
| B | 基本合格 | 可继续进行专项测试，但建议优化 |
| C | 不满足要求 | 仅用于开发验证，需要继续优化 |
| D | 不合格 | 需要重新检查数据、标注或训练过程 |
| Invalid | 验证无效 | 存在数据泄漏、样本不足或测试方法不一致 |

最终等级采用最低项原则：

```text
最终等级 = min(
    验证集质量等级，
    整体指标等级，
    关键异常类别等级，
    旧版本回归测试等级
)
```

较高的整体平均指标不能抵消某个必要类别的明显低分。

## 3. 数据集划分与验证准入条件

本项目采用以下验证流程：

```text
完整标注数据集
    ├── Train：80%，用于模型训练
    └── Val：20%，用于训练过程中的模型选择和标准指标评估

PaddleClas 额外测试
    ├── 独立 FAIL 测试集：重点验证异常漏检
    └── 独立 OK 测试集：重点验证正常样本误报
```

80%/20% 是合理的数据比例，但比例本身不能保证验证有效。只有同时满足数据独立性、场景覆盖和配置一致性，val 结果才具有模型评价意义。

### 3.1 数据独立性

- 验证集图片不得参与训练。
- 同一连续采集序列的相邻帧不得同时分布在训练集和验证集中。
- 应按照车辆、VIN、采集批次或连续视频段整体划分数据。
- 80%/20% 应按照采集批次或连续序列分组后划分，不建议对连续帧进行完全随机拆分。
- 应去除完全重复或高度相似的图片。
- 候选模型与上一模型版本必须使用同一套固定基准验证集比较。
- 验证集标签必须经过人工复核，关键 FAIL 样本建议双人复核。
- PaddleClas 的独立 FAIL/OK 测试集不得与 Train、Val 重复，建议按采集日期或批次完全隔离。
- 独立 FAIL/OK 测试集原则上只用于最终验收，不应用于反复调参或选择 Epoch；如果其中的失败样本被加入下一版本训练集，应为下一版本补充新的独立测试样本，并记录测试集版本变化。

如果随机拆分连续帧，可能导致训练集和验证集画面高度相似，使指标虚高，此次验证应判定为 `Invalid`。

### 3.2 场景覆盖

验证集应根据模型适用范围覆盖以下场景：

- 北京、武汉等实际部署地点；
- 白天、夜间带补光和不带补光的场景；
- 前盖、后盖；
- 左相机、右相机；
- 不同车型和连接杆位置；
- 正常、缺失、遮挡状态；
- 模糊、过曝、欠曝、过电泳、不过电泳。

如果模型只负责部分场景，应在模型版本说明中明确适用范围，未覆盖场景不得默认视为已经验证。

### 3.3 最低样本数量

| 验证等级上限 | 每个普通类别 | 每个关键 FAIL 类别 |
|---|---:|---:|
| S | ≥300 | ≥600 |
| A | ≥200 | ≥300 |
| B | ≥100 | ≥100 |
| C | ≥50 | ≥50 |
| Invalid | ＜50 | ＜50 |

- PaddleDetection 按有效标注实例数量统计。
- PaddleClas 按有效分类图片数量统计。
- 样本数量不满足对应要求时，即使指标达到要求，也只能取得样本数量所允许的最高等级。
- 仅知道验证集总图片数还不够，版本说明中还应记录每个类别的图片数或标注实例数。

例如，关键 FAIL 类别只有 80 张验证图片，即使 80 张全部预测正确，也不足以证明模型在该类别上已经稳定达到 99% 正确率。

### 3.4 测试配置一致性

val 必须使用固定且可复现的模型评估配置，包括：

- 固定的输入尺寸；
- 固定的图像裁剪和归一化方式；
- 固定的检测和 NMS 配置；
- 明确指定的候选模型权重，而不是任意训练 Epoch；
- 固定的类别映射文件；
- 与训练配置匹配的前后处理逻辑。

如果模型经过导出或格式转换，可单独进行一致性验证，但该结果不属于本文件的 val 指标分级范围。

## 4. PaddleDetection 验证标准

### 4.1 必须输出的指标

PaddleDetection 模型需要输出：

- `mAP@0.5:0.95`：不同 IoU 阈值下的综合检测与定位能力；
- `AP@0.5`：目标是否被基本正确检测；
- `AP@0.75`：检测框定位是否准确；
- 每个类别的 AP；
- COCO Average Recall（AR）；
- 不同目标尺寸的 AP 和 AR；
- 每个类别的 AP；
- Val 图片总数和评估 FPS。

PaddleDetection 默认 COCO 输出中的指标含义需要特别区分：

- `AP@[IoU=0.50:0.95]` 是通常所说的 COCO mAP；
- `AP@[IoU=0.50]` 是 AP50；
- `AP@[IoU=0.75]` 是 AP75；
- `AR@[IoU=0.50:0.95]` 是在多个 IoU 阈值上平均得到的 Average Recall；
- COCO AR 是模型标准指标，不等同于某个固定置信度阈值下的 Recall；
- `area=small` 输出 `-1.000` 通常表示验证集中没有对应尺寸目标，不代表模型该项得分为负数；
- `Per-category AP` 默认通常对应每个类别的 `AP@0.50:0.95`，不是每类 AP50。


COCO AP 综合了多个置信度阈值，应与 AP50、AP75、分类别 AP 和 AR 一起评价模型，不能只看单个指标。

### 4.2 COCO Val 指标分级阈值

下表所有指标均采用百分比表示。

| 等级 | mAP50:95 | AP50 | AP75 | 最低单类 AP50:95 |
|---|---:|---:|---:|---:|
| S | ≥80% | ≥99% | ≥93% | ≥78% |
| A | ≥75% | ≥98% | ≥90% | ≥70% |
| B | ≥65% | ≥95% | ≥80% | ≥60% |
| C | ≥50% | ≥85% | ≥70% | ≥45% |
| D | ＜50% | ＜85% | ＜70% | ＜45% |

模型等级由该行所有指标共同决定。任意一项未达到当前等级要求，应降低到该项实际满足的等级。


### 4.3 硬性否决条件

出现以下任意情况，PaddleDetection 模型不得评为 A 或 S：

- COCO Val 指标未达到对应等级；
- 验证集中存在成批或系统性的检测失败；
- 夜间、全黑或特定地点场景明显失效；
- 某一个必要类别完全无法检测；
- 检测框严重偏移，导致后续 PaddleClas 裁剪区域不完整；
- 新模型关键类别 AP 明显低于上一模型版本；
- 使用训练集或与训练集高度重复的数据进行验证；
- 只报告整体 mAP，没有分类别 AP。

### 4.4 PicoDet 使用要求

如果 PicoDet 负责定位 ROI，然后将 ROI 交给 PaddleClas 分类，PicoDet 应优先关注 AP50 和 AR 所反映的目标检出能力：

```text
PicoDet 漏检
    → 没有生成 ROI
    → PaddleClas 没有分类机会
    → 最终检测结果可能直接漏检
```

因此，不能只根据较高的整体 mAP 判断模型，还应同时检查 AP50、AR 和分类别 AP，避免必要类别的检出能力被整体平均值掩盖。

建议调优顺序：

1. 检查关键目标的分类别 AP；
2. 检查整体 AP50 和 AR；
3. 检查 AP75 和检测框定位质量；
4. 使用实际裁剪结果验证 PaddleClas。

## 5. PaddleClas 验证标准

### 5.1 PaddleClas Eval 日志指标

根据本项目当前 PaddleClas Eval 输出，模型标准使用以下指标：

- `CELoss`：交叉熵损失；
- `loss`：本次评估总损失；
- `top1`：真实类别是否为模型得分最高的类别，是主要分级指标；
- `top2`：真实类别是否位于得分最高的两个类别中，仅作为参考；
- `batch_cost`、`reader_cost`、`ips`：评估过程性能数据，不参与模型精度等级判定；
- Eval 总样本数量：用于判断指标是否具有足够的样本基础。

本项目 PaddleClas 为 OK/FAIL 分类时，`top2=1.00000` 基本属于必然结果，因此 Top-2 不作为模型等级指标。

### 5.2 Val 分级标准

| 等级 | Val Top-1 | CELoss/loss 参考范围 | 评价 |
|---|---:|---:|---|
| S | ≥99.5% | ≤0.10 | 优秀，可作为基准候选模型 |
| A | ≥99.0% | ≤0.15 | 合格，可作为正式候选模型 |
| B | ≥97.0% | ≤0.25 | 基本合格，建议继续优化 |
| C | ≥95.0% | ≤0.40 | 仅用于开发验证 |
| D | ＜95.0% | ＞0.40或结果异常 | 不合格 |

Top-1 是等级判定的主要指标，CELoss/loss 用于辅助判断模型置信度和收敛状态。Loss 阈值只适用于类别数量、损失函数、标签平滑、类别权重和 Val 数据集保持一致的模型之间；配置发生变化时，应以 Top-1 和相对变化为主，不能直接横向比较 Loss。

模型必须同时满足以下基本条件：

- `CELoss` 和 `loss` 为有效有限值，不能出现 `NaN` 或 `Inf`；
- Val Loss 没有明显发散；
- Top-1 没有在训练后期持续下降；
- 使用固定的 20% Val 和相同评估配置；
- Val 样本数量及 OK/FAIL 类别数量已经记录。

如果 Top-1 达到某等级，但 Loss 明显异常或相较上一版本大幅升高，应标记为“指标异常，需复核”，不直接提高等级。

### 5.3 独立 FAIL/OK 专项测试

除 20% Val 外，本项目单独制作 FAIL 测试集和 OK 测试集。这两套数据用于检查模型在单一类别上的实际分类正确率。

```text
FAIL 测试正确率
= 正确预测为 FAIL 的图片数 / FAIL 测试集总数

OK 测试正确率
= 正确预测为 OK 的图片数 / OK 测试集总数
```

| 等级 | FAIL 测试正确率 | OK 测试正确率 |
|---|---:|---:|
| S | ≥99.5% | ≥99.5% |
| A | ≥99.0% | ≥99.0% |
| B | ≥97.0% | ≥97.0% |
| C | ≥95.0% | ≥95.0% |
| D | ＜95.0% | ＜95.0% |

FAIL 和 OK 测试集必须分别报告，不能因为两套测试集样本数不同而只合并成一个总 Accuracy。每套测试至少记录：

- 测试集版本；
- 图片总数；
- 正确数量；
- 错误数量；
- 正确率；
- 典型错误图片。

### 5.4 PaddleClas 最终模型等级

PaddleClas 模型等级采用最低项原则：

```text
PaddleClas 模型等级 = min(
    20% Val Top-1/Loss 等级，
    独立 FAIL 测试等级，
    独立 OK 测试等级，
    样本数量允许的最高等级
)
```

如果当前只有 PaddleClas Eval 日志，可以先给出 `Val 等级`；独立 FAIL/OK 测试完成后，再给出完整的 PaddleClas 模型等级。

### 5.5 硬性无效或降级条件

出现以下任意情况，应判定验证无效或降低模型等级：

- Train、Val、独立 FAIL/OK 测试集之间存在重复图片；
- 连续采集的高度相似帧被拆分到 Train 和 Val；
- Val 数据类别严重失衡，但只报告总体 Top-1；
- 某个专项测试集样本数量不足对应等级要求；
- 夜间或特定地点数据出现成批分类错误；
- 新模型 Top-1 或专项测试正确率明显低于上一模型版本；
- Eval 输出出现 `NaN`、`Inf` 或 Loss 异常发散。

## 6. 标准范围说明

本文件只规定 PaddleDetection 和 PaddleClas 模型本身的 Val 及专项测试分级，不包含以下内容：

- PaddleDetection 与 PaddleClas 的系统串联效果；
- 生产现场使用的置信度阈值；
- 主控对视觉结果的处理逻辑；
- 板端推理耗时和实时性；
- UI 展示、报警或急停逻辑；
- 整套视觉系统的最终验收标准。

以上内容应在单独的系统联调或现场验收标准中定义，不影响本文件对单个模型的等级判定。

## 7. 新旧模型回归要求

候选模型必须与上一模型版本使用同一验证集和相同评估配置进行比较。

新模型评为合格版本应同时满足：

- PaddleDetection 的 mAP50:95、AP50、AP75 和分类别 AP 不得出现无法解释的明显下降；
- PaddleClas 的 Val Top-1、Loss、FAIL 测试正确率和 OK 测试正确率不得出现无法解释的明显下降；
- 不得新增明显的场景性批量错误；
- 新增数据对应场景的指标应得到改善或保持稳定；
- 所有比较必须使用同一版本的固定验证集和相同评估配置。

如果新模型整体指标提高，但某个必要类别或专项测试正确率明显下降，应判定模型回归测试不通过。

## 8. 模型版本说明中的验证记录

每个模型版本至少应记录以下内容：

```markdown
## Validation Result

- Model type: PaddleDetection / PaddleClas
- Model architecture:
- Model version:
- Training dataset version:
- Validation dataset version:
- Validation date:
- Train/Val split: 80% / 20%
- Input resolution:
- Validation sample count:
- Per-class sample count:
- Scenario distribution:
- PaddleDetection metrics: mAP50:95 / AP50 / AP75 / AR / per-category AP
- PaddleClas metrics: CELoss / loss / Top-1 / Top-2
- Independent FAIL test: total / correct / incorrect / accuracy
- Independent OK test: total / correct / incorrect / accuracy
- Comparison with previous model version:
- Failed sample analysis:
- Evaluation grade: S / A / B / C / D / Invalid
- Model conclusion: Baseline / Qualified / Optimization Required / Rejected
```

## 9. 模型等级结论

模型评为合格版本的最低条件为：

- PaddleDetection 达到 A 级；
- PaddleClas 的 20% Val 和独立 FAIL/OK 专项测试均达到 A 级；
- 与上一模型版本相比不存在无法解释的关键指标回退；
- 验证集数据、配置和评估过程满足本标准的基础准入条件。

如果只满足 B 级，可保留为开发候选模型，但应继续优化后再评为合格模型版本。

## 10. 当前模型输出评估示例

本节根据 2026-10-16 的 PaddleDetection 输出和 2026-10-13 的 PaddleClas 输出进行示例评估，用于说明如何按照本标准判定单个模型等级。

### 10.1 当前 PaddleDetection 输出

| 指标 | 输出值 | 对应等级 | 说明 |
|---|---:|---|---|
| Val 图片数 | 4826 | 不评级 | Val日志未显示实际数量 |
| mAP50:95 | 77.5% | A | 达到 A 级阈值 75% |
| AP50 | 99.3% | S | 目标基本检出能力较好 |
| AP75 | 91.9% | A | 检测框定位精度达到 A 级 |
| lower AP50:95 | 76.1% | A | 达到最低单类 A 级要求 |
| up AP50:95 | 78.8% | S | 达到最低单类 S 级要求 |
| medium AP50:95 | 77.4% | — | 与整体指标接近 |
| large AP50:95 | 67.0% | — | 明显低于 medium，需要检查 large 样本数量和失败图片 |
| AR@0.50:0.95 | 81.0% | 参考指标 | COCO 多个 IoU 阈值下的平均召回表现 |
| average FPS | 80.04 | 不参与精度评级 | 仅代表本次评估环境吞吐量 |

`area=small` 的 AP 和 AR 为 `-1.000`，表示当前验证集中没有被 COCO 规则划分为 small 的有效目标，不能解释为 small 目标检测失败。

按照本标准，当前 PaddleDetection 的 **COCO Val 等级为 A**：

```text
PaddleDetection COCO Val：A级通过
模型结论：合格候选模型
```


### 10.2 当前 PaddleClas 输出

当前输出为：

```text
CELoss = 0.05086
Top-1  = 0.99966（99.966%）
Top-2  = 1.00000（100%）
```

对应判断：

| 指标 | 输出值 | 对应等级 | 说明 |
|---|---:|---|---|
| Val Top-1 | 99.966% | S | 达到 S 级阈值 |
| Val Top-2 | 100% | 不参与评级 | OK/FAIL 两分类下不具有区分能力 |
| CELoss/loss | 0.05086 | S级参考范围 | 低于0.10，且日志未出现NaN或Inf |
| batch_cost/reader_cost | 已输出 | 不参与精度评级 | 只反映评估过程耗时 |
| ips | 已输出 | 不参与精度评级 | 只反映本次评估环境吞吐量 |

根据当前 PaddleClas Val 输出，可直接得到以下结论：

```text
PaddleClas 20% Val Top-1：99.966%
PaddleClas Val Loss：0.05086
PaddleClas Val 等级：S级
```

#### 独立 FAIL 专项测试

测试输出：

```text
Total FAIL images = 909
CELoss/loss       = 0.07342
Top-1             = 0.99670（99.670%）
Top-2             = 1.00000（100%）
```

根据样本总数和 Top-1 结果：

```text
正确预测为 FAIL：906 张
错误预测为 OK：3 张
FAIL 测试正确率：906 / 909 = 99.670%
```

| 指标 | 结果 | 对应等级 | 说明 |
|---|---:|---|---|
| FAIL 样本数 | 909 | 满足 S 级样本要求 | 高于关键 FAIL 类别 600 张要求 |
| FAIL 测试正确率 | 99.670% | S | 高于 99.5% |
| FAIL 测试错误数 | 3 | 记录项 | 需要保留并分析这 3 张错误图片 |
| CELoss/loss | 0.07342 | S 级参考范围 | 低于 0.10 |
| Top-2 | 100% | 不参与评级 | OK/FAIL 两分类下不具有区分能力 |

独立 FAIL 专项测试等级：**S 级**。

#### 独立 OK 专项测试

测试输出：

```text
Total OK images = 7849
CELoss/loss     = 0.04824
Top-1           = 1.00000（100%）
Top-2           = 1.00000（100%）
```

根据样本总数和 Top-1 结果：

```text
正确预测为 OK：7849 张
错误预测为 FAIL：0 张
OK 测试正确率：7849 / 7849 = 100%
```

| 指标 | 结果 | 对应等级 | 说明 |
|---|---:|---|---|
| OK 样本数 | 7849 | 满足 S 级样本要求 | 高于普通类别 300 张要求 |
| OK 测试正确率 | 100% | S | 高于 99.5% |
| OK 测试错误数 | 0 | 记录项 | 本次专项测试未出现错误 |
| CELoss/loss | 0.04824 | S 级参考范围 | 低于 0.10 |
| Top-2 | 100% | 不参与评级 | OK/FAIL 两分类下不具有区分能力 |

独立 OK 专项测试等级：**S 级**。

#### PaddleClas 完整等级

| 测试项目 | 结果 | 等级 |
|---|---:|---|
| 20% Val | Top-1 99.966%，Loss 0.05086 | S |
| 独立 FAIL 测试 | 906/909，99.670%，Loss 0.07342 | S |
| 独立 OK 测试 | 7849/7849，100%，Loss 0.04824 | S |

按照最低项原则：

```text
PaddleClas 模型等级 = min(S, S, S) = S
```

当前 PaddleClas 完整模型等级为：**S 级**。

### 10.3 当前整体结论

| 模型 | 当前可确认等级 | 尚缺内容 |
|---|---|---|
| PaddleDetection | COCO Val A | 建议补充分类别样本数和 large 目标失败分析 |
| PaddleClas | 完整模型等级 S | 建议保留并分析 3 张 FAIL 错误图片 |

现有输出说明 PaddleDetection 已达到 COCO Val A 级，PaddleClas 的 20% Val、独立 FAIL 测试和独立 OK 测试均达到 S 级，因此 PaddleClas 完整模型等级为 S 级。

## 11. 参考资料

- [PaddleDetection Evaluation and Inference](https://github.com/PaddlePaddle/PaddleDetection/blob/release/2.9/docs/tutorials/GETTING_STARTED.md)
- [PaddleDetection COCO AP/AR Metrics Example](https://github.com/PaddlePaddle/PaddleDetection/blob/release/2.9/configs/smalldet/README.md)
- [PaddleClas Model Evaluation](https://github.com/PaddlePaddle/PaddleClas/blob/release/2.6/docs/en/models/PP-HGNetV2_en.md)
- [Paddle Top-k Accuracy Metric](https://github.com/PaddlePaddle/Paddle/blob/develop/python/paddle/metric/metrics.py)
