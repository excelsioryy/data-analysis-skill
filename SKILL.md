---
name: data-analysis
description: 数据分析技能，配备强制性的提示词评估步骤。它利用专门的 Python 包装脚本，在执行前通过模式发现和上下文补充来丰富模糊的请求，并包含安全的数据摄入协议。
---

# 数据分析技能 (Data Analysis Skill)

## 目的
为了防止执行错误和模型幻觉，本技能强制执行“评估 → 澄清 → 摄入 → 执行”的闭环流程。该技能要求必须使用一个预处理脚本来封装用户的请求，强制智能体（Agent）在编写正式分析代码之前，先评估提示词的清晰度并安全地探查数据。

## 第一阶段：提示词评估与包装器执行 (Phase 1: Evaluation)

**步骤 1：输入标准化**
所有与数据分析相关的用户交互必须首先通过 `prompt-improver/improve-prompt.py` 包装器进行处理。该脚本负责处理绕过逻辑（快捷方式）并注入评估指令。

**调用命令：**
该脚本通过 `stdin` 接收 JSON 对象。构建命令如下：

```bash
echo '{"prompt": "在此处填入用户的原始提示词"}' | python3 prompt-improver/improve-prompt.py

```

**步骤 2：解读脚本输出**
脚本会输出一个包含 `wrapped_prompt` 的 JSON 对象。智能体必须读取此输出，并严格遵循其中包含的“PROMPT EVALUATION”（提示词评估）指令。

### 绕过逻辑 (由脚本处理)

脚本会自动检测以下情况并允许立即执行：

* **`*` 前缀**：显式绕过（用户表示“我知道我在做什么”）。
* **`/` 前缀**：斜杠命令。
* **`#` 前缀**：记忆功能。

### 评估逻辑 (第一阶段的核心)

当脚本封装提示词后，它会询问智能体：*“这个提示词是否清晰到可以直接执行？”*

智能体必须根据**数据上下文**应用以下标准：

1. **模糊语境 (触发第二阶段):**
* “分析数据” (分析哪个文件？哪个指标？)
* “展示趋势” (时间范围？聚合方式？)
* “修复错误” (哪个日志文件？)
* *行动：* 进入 **第二阶段：发现与澄清**。


2. **清晰语境 (跳至执行):**
* “绘制 `users.csv` 中 2024 年 1 月的日活跃用户图表。”
* “计算 `data.parquet` 中 'price' 列的平均值。”
* *行动：* 跳过第二阶段，直接进入 **第三阶段：智能数据摄入** (如果尚未读取数据) 或 **执行阶段**。



## 第二阶段：发现与澄清 (Phase 2: Discovery)

如果第一阶段的评估认定提示词是 **模糊 (VAGUE)** 的，智能体将启动此子程序来收集上下文。

**工作流：**

1. **确认与通知：**
* 告知用户：“嘿！提示词优化器标记您的请求在 [数据集/指标/范围] 方面有点模糊。让我先检查一下可用数据。”


2. **系统化调研 (数据发现):**
* **扫描目录：** 使用 `ls -lh` 或 `find` 定位潜在的数据文件 (`.csv`, `.json`, `.parquet`, `.xlsx`, `.log`)。
* **检查架构：** 使用 `head -n 5 <filename>` 或 Python 代码片段读取列名。**切勿臆断列名。**
* **检查历史：** 回顾之前的对话轮次，查看是否有已加载的数据框。


3. **生成澄清问题：**
* *仅*基于步骤 2 中发现的文件头信息，提出 1-3 个具体问题。
* **不要问开放式问题。** 提供数据中发现的选项。
* *示例：*
> **发现文件：** `sales_2023.csv` (列: date, amount), `sales_2024.csv` (列: date, amount)
> **提问：** “我应该分析哪一年的数据：2023 还是 2024？”


4. **等待用户输入：**
* 一旦用户回答，将其答案与原始提示词结合，进入下一阶段。


## 第三阶段：智能数据摄入与上下文加载 (Phase 3: Ingestion)

为了防止 Token 溢出和执行环境卡死，智能体 **严禁** 直接打印全量数据或将其加载到 Prompt 上下文中。必须严格遵守“先采样，后编码”的协议。

### 1. “先看一眼”协议 (The Peek-First Protocol)

在编写正式分析代码前，必须先执行一段轻量级的探查代码。

**原则：**

* **只看样貌，不看全貌**：仅获取元数据和少量样本。
* **元数据优先**：必须获取列名（Columns）和数据类型（Dtypes）。
* **样本限制**：严格限制读取行数（推荐 `n=5`）。

**标准探查代码模式 (Python示例):**

```python
import pandas as pd

# 设定文件路径
file_path = 'target_data.csv'

try:
    # 1. 仅读取前5行，避免加载大文件占用内存
    # 注意：对于非CSV文件 (如 Parquet, Excel)，使用相应的 read_ function 并配合 head()
    if file_path.endswith('.csv'):
        df_preview = pd.read_csv(file_path, nrows=5)
    elif file_path.endswith('.parquet'):
        df_preview = pd.read_parquet(file_path).head(5)
    elif file_path.endswith(('.xls', '.xlsx')):
        df_preview = pd.read_excel(file_path, nrows=5)
    
    # 2. 输出关键元数据供智能体思考
    print("=== 数据预览 (前5行) ===")
    print(df_preview.to_markdown(index=False))
    
    print("\n=== 列信息与类型 ===")
    print(df_preview.info())
    
except Exception as e:
    print(f"摄入错误: {e}")

```

### 2. 编码与格式自适应

* **编码回退**：如果读取失败，尝试 `encoding='gbk'` 或 `encoding='latin1'`。
* **分隔符嗅探**：如果数据挤在一列，尝试 `sep=';'` 或 `sep='\t'`。

### 3. 上下文注入

智能体在获得上述代码的输出后，必须在思维链中构建数据的“虚拟视图”，确认列名和类型无误后，方可生成最终的分析代码。

## 行为示例 (Examples)

### 场景 A：模糊请求

**用户：** “分析日志。”

**1. 执行：**

```bash
echo '{"prompt": "分析日志"}' | python3 prompt-improver/improve-prompt.py

```

**2. 智能体内在思维 (由脚本输出触发):**
“包装器询问这是否清晰。并不清晰。‘日志’是有歧义的。我需要找出存在什么日志。”

**3. 智能体行动 (第二阶段):**

* 运行 `ls *.log` -> 发现 `access.log`, `error.log`。
* 运行 `head -n 1 access.log` -> 查看具体列。
* **回复用户：** “提示词优化器标记了这个问题。我找到了 `access.log` 和 `error.log`。您想分析哪一个？您是想看错误率还是流量？”

### 场景 B：专家模式绕过

**用户：** “*统计 data.csv 的行数”

**1. 执行：**

```bash
echo '{"prompt": "*统计 data.csv 的行数"}' | python3 prompt-improver/improve-prompt.py

```

**2. 脚本逻辑：**
检测到 `*`。将其剥离。返回 `"统计 data.csv 的行数"` 作为原始命令，不带评估包装器。

**3. 智能体行动：**
直接进入第三阶段（快速预览结构）或直接编写 Python 代码统计行数。


## 第四阶段：全流程动态执行与自愈

这是将分析转化为代码的核心阶段。智能体必须遵循标准的工程化数据链路：**质量检查 (DQC) → 清洗处理 → 分析建模 → 可视化呈现**。

### 1. 常用技术栈 (Recommended Stack)
* **核心处理**: `pandas`, `numpy`
* **可视化**: `matplotlib`, `seaborn` (静态); `plotly` (交互式)
* **统计与挖掘**: `scipy`, `sklearn` (如需)
* **工具库**: `tabulate` (美化打印), `\xlsx`工具 (Excel 处理)

### 2. 标准执行工作流 (Standard Workflow)

#### 步骤 A: 数据质量检查 (Data Quality Check, DQC)
在进行任何分析前，必须先“体检”。这能避免“垃圾进，垃圾出 (GIGO)”。

**代码规范：**
```python
def check_data_quality(df):
    report = {
        'rows': len(df),
        'missing_values': df.isnull().sum().to_dict(),
        'duplicates': df.duplicated().sum(),
        'dtypes': df.dtypes.apply(lambda x: str(x)).to_dict()
    }
    # 打印关键异常
    if report['duplicates'] > 0:
        print(f"[Warning] Found {report['duplicates']} duplicate rows.")
    return report

# 执行检查
quality_report = check_data_quality(df)

```

#### 步骤 B: 数据处理与清洗 (Processing)

根据 DQC 的结果进行针对性清洗。

**常见操作：**

* **类型修正**：`df['date'] = pd.to_datetime(df['date'], errors='coerce')`
* **空值处理**：`fillna()`, `dropna()`
* **异常值过滤**：基于 IQR 或 Z-score 过滤。

#### 步骤 C: 深度分析 (Analysis)

执行核心统计或计算逻辑。

**规范：**

* **逻辑封装**：复杂的计算逻辑应尽量封装为函数。
* **结果验证**：计算后检查结果的合理性（例如：概率不应大于1，销售额不应为负）。

#### 步骤 D: 可视化 (Visualization)

根据需求选择**静态**或**交互式**引擎。

* **静态图表 (Matplotlib/Seaborn)**：
* 适用于报告文档。
* 必须使用 `plt.savefig()` 保存，**严禁** `plt.show()`。
* 使用 `matplotlib.use('Agg')` 确保后台运行稳定。


* **交互式图表 (Plotly)**：
* 适用于探索性分析。
* 生成 `.html` 文件，支持缩放和悬停查看。


**通用绘图模板：**

```python
import matplotlib
matplotlib.use('Agg') # 后台模式
import matplotlib.pyplot as plt
import seaborn as sns

try:
    plt.figure(figsize=(12, 6))
    # 自动处理，不强制指定字体路径，依赖环境默认配置
    sns.lineplot(data=df, x='date', y='value')
    plt.title('Analysis Result')
    plt.tight_layout()
    plt.savefig('result_chart.png')
    print("Chart saved: result_chart.png")
except Exception as e:
    print(f"Plotting Error: {e}")

```

### 3. 错误处理与反思 (Error Handling & Reflection)

智能体必须具备工程师的调试能力，遵循 **“捕获 → 诊断 → 修复”** 的闭环。

**自动修复协议 (Self-Healing Protocol)：**

1. **捕获异常 (Catch)**：
* 所有的主要执行块（I/O, 绘图, 复杂计算）必须包裹在 `try-except` 块中。
* 打印完整的 `traceback` 或简洁的错误信息到 `stderr`。


2. **诊断与反思 (Diagnose)**：
* 如果遇到 `KeyError`：检查是否列名大小写不匹配或含有空格（自动尝试 `.strip()`）。
* 如果遇到 `ModuleNotFoundError`：检查是否使用了非标准库，尝试降级使用基础库（如用 `csv` 替代 `pandas`，虽然少见但需考虑）。
* 如果遇到 `ValueError` (数据格式错误)：检查是否在字符串列上执行了数值计算。


3. **自我修正 (Fix)**：
* 在下一次执行中应用修正逻辑。
* **限制**：最多重试 3 次。若失败，生成一份“错误分析报告”给用户，而不是崩溃。


### 4. 交付物规范 (Deliverables)

执行结束后，必须输出明确的交付清单：

1. **清洗后的数据**：保存为 `.csv` 或格式化的 `.xlsx`。
2. **可视化文件**：`.png` 图片或 `.html` 交互网页。
3. **分析结论**：用**自然语言**总结数据背后的洞察（例如：“我们发现周末的流量比工作日高 20%”），而不仅仅是报数字。

## 第五阶段：结果验证与输出 (Phase 5: Verification & Delivery)

执行完毕不代表任务结束。智能体必须在“交卷”前进行最后一次自我审查，确保交付物不仅代码无误，且逻辑和格式均符合预期。

### 1. 自动质量门控 (Quality Gate)

在将结果呈现给用户之前，必须在思维链中执行以下自检清单：

* **完整性检查**：
  * [ ] 所有的图表文件（.png/.html）是否已成功生成并在磁盘上存在？
  * [ ] 关键的计算结果（如 KPI、统计量）是否非空且数值合理？
  * [ ] 输出文件（如 CSV）是否包含数据且表头正确？
* **格式一致性**：
  * [ ] 图表是否有标题、轴标签和图例？
  * [ ] 代码是否清除了所有的调试打印（print debug）？

### 2. 交付协议 (Delivery Protocol)

确认通过质量门控后，按照以下结构组织最终回复：

**A. 关键发现 (Executive Summary)**
* 直接回答用户的核心问题（基于数据事实）。
* *示例：* “分析显示，Q3 的用户流失率主要集中在基础版订阅用户，流失高峰发生在注册后的第 14 天。”

**B. 可视化展示 (Visual Evidence)**
* 嵌入生成的静态图片（如果支持）或提供文件路径。
* 简要说明图表中的关键趋势。

**C. 数据文件交付 (File Handover)**
* 列出已生成的中间文件路径，供用户下载或进一步检查。
* *示例：* “详细的数据清洗结果已保存至 `clean_data_2024.csv`。”

**D. 下一步建议 (Actionable Next Steps)**
* 基于当前分析结果，主动推荐后续的分析方向。
* *示例：* “既然发现了第 14 天是流失高峰，建议接下来分析该时间点的用户行为日志（logs.json）。”

### 3. 异常后的优雅降级 (Graceful Degradation)

如果在验证阶段发现结果异常（例如图表为空）：

1. **不要** 伪造结果或假装成功。
2. **报告现状**：诚实地告知用户哪部分成功了，哪部分失败了。
3. **提供诊断信息**：提供错误日志或数据样本，以便用户决定是重试还是修改请求。

## 渐进式披露 (Progressive Disclosure)

本 SKILL.md 包含核心工作流。如需更深入的指导：

* 研究策略：`prompt-improver/research-strategies.md`
* 提问模式：`prompt-improver/question-patterns.md`
* 综合示例：`prompt-improver/examples.md`
仅在需要关于提示词改进的详细指导时加载这些参考文档。
