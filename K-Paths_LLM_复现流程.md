# K-Paths LLM 推理结果复现流程

本文档整理 `rsinghlab/K-Paths` 中 LLM 推理结果的复现流程，重点覆盖数据准备、路径抽取、LLM 推理、评估命令和注意事项。

仓库地址：[rsinghlab/K-Paths](https://github.com/rsinghlab/K-Paths)

论文地址：[K-Paths: Reasoning over Graph Paths for Drug Repurposing and Drug Interaction Prediction](https://arxiv.org/abs/2502.13344)

预抽取路径数据入口：[Hugging Face - Tassy24](https://huggingface.co/Tassy24)

## 1. 推荐复现路线

如果目标是复现 LLM 推理结果，优先使用作者已经抽取好的 K-Paths 数据，而不是从头生成路径。

原因：

- 从头抽取路径依赖完整 `data.zip`，需要 Google Drive 数据包。
- 路径抽取和子图生成耗时较长。
- LLM 推理脚本使用 `vllm`，通常需要 Linux + CUDA GPU 环境。
- 直接使用预抽取路径数据更适合先验证推理和评估流程。

推荐顺序：

1. 克隆仓库并安装依赖。
2. 下载作者提供的预抽取 reasoning paths 数据。
3. 使用 `llm/llm_inference.py` 做零样本推理。
4. 使用 `llm/evaluate_llm_regex.py` 做结果评估。
5. 如需完整复现，再从 `data.zip` 开始重新抽取 K-Paths。

## 2. 环境准备

```bash
git clone https://github.com/rsinghlab/K-Paths.git
cd K-Paths

python3.10 -m venv .kpaths-env
source .kpaths-env/bin/activate

pip install -r requirements.txt
```

官方 README 要求 Python 3.10+。

依赖中包含：

- `torch==2.6.0`
- `transformers==4.48.3`
- `vllm`
- `bitsandbytes`
- `torch-geometric`
- `faiss-cpu`
- `bert-score`

注意：`requirements.txt` 包含大量 CUDA 相关依赖。如果在 macOS 或无 GPU 环境中安装，可能会遇到兼容性问题。复现完整 LLM 推理建议使用 Linux + NVIDIA GPU。

## 3. 数据准备

### 3.1 使用预抽取路径数据

作者在 Hugging Face 账号下提供了预抽取路径数据。入口：

```text
https://huggingface.co/Tassy24
```

其中 DrugBank 预抽取数据示例：

```text
https://huggingface.co/datasets/Tassy24/K-Paths-inductive-reasoning-drugbank
```

下载后，需要将路径数据放到仓库的 `data/paths/` 目录下。LLM 脚本默认会寻找如下文件：

```text
data/paths/drugbank_train_add_reverse.json
data/paths/drugbank_test_add_reverse.json
data/paths/ddinter_train_add_reverse.json
data/paths/ddinter_test_add_reverse.json
data/paths/pharmaDB_train_add_reverse.json
data/paths/pharmaDB_test_add_reverse.json
```

如果文件放在其他位置，也可以通过 `--dataset_path` 显式指定。

### 3.2 从头抽取路径

如果要从原始数据重新生成 reasoning paths，需要下载 README 中的 Google Drive 数据包：

```text
data.zip
```

解压后目录应类似：

```text
K-Paths/
  data/
    hetionet/
    drugbank/
    ddinter/
    pharmaDB/
    paths/
    subgraphs/
```

然后先生成增强知识图谱：

```bash
python k-paths/create_augmented_network.py
```

再抽取 K-Paths：

```bash
python k-paths/get-Kpaths.py \
  --dataset_name drugbank \
  --split test \
  --mode K-paths \
  --add_reverse_edges
```

可选数据集：

```text
drugbank
ddinter
pharmaDB
```

可选 split：

```text
train
test
```

可选 mode：

```text
K-paths
no_filter
neighbors_only
```

输出默认保存到：

```text
data/paths/{dataset}_{split}_add_reverse.json
```

例如：

```text
data/paths/drugbank_test_add_reverse.json
```

注意：README 示例中写的是 `--dataset ddinter`，但代码真实参数名是 `--dataset_name`。

## 4. LLM 推理

LLM 推理入口：

```text
llm/llm_inference.py
```

示例命令：

```bash
python llm/llm_inference.py \
  --dataset_path data/paths/drugbank_test_add_reverse.json \
  --dataset_name drugbank \
  --output_dir outputs \
  --model_name_or_path meta-llama/Meta-Llama-3.1-8B-Instruct \
  --use_kg
```

如果只想快速检查流程是否能跑通：

```bash
python llm/llm_inference.py \
  --dataset_path data/paths/drugbank_test_add_reverse.json \
  --dataset_name drugbank \
  --output_dir outputs \
  --model_name_or_path meta-llama/Meta-Llama-3.1-8B-Instruct \
  --use_kg \
  --debug
```

### 4.1 常用参数

```text
--dataset_name
```

支持：

```text
drugbank
ddinter
pharmacotherapydb
```

注意：路径抽取脚本里使用的是 `pharmaDB`，LLM 推理脚本里使用的是 `pharmacotherapydb`。

```text
--dataset_path
```

指定推理输入 JSON。若不传，脚本会根据 `--dataset_name` 使用默认路径：

```text
drugbank: data/paths/drugbank_test_add_reverse.json
ddinter: data/paths/ddinter_test_add_reverse.json
pharmacotherapydb: data/paths/pharmaDB_test_add_reverse.json
```

```text
--model_name_or_path
```

Hugging Face 模型名或本地模型路径。

```text
--use_kg
```

启用 K-Paths 信息作为提示词上下文。

```text
--use_options
```

让模型从候选选项中选择，适合 DrugBank 等分类任务。

```text
--option_style
```

候选项格式：

```text
numbered
bulleted
```

```text
--start_index
--end_index
```

用于分片推理，适合大数据集或多机分批运行。

### 4.2 输出位置

推理结果会保存在：

```text
{output_dir}/{dataset_name}_{model_name}_.../predictions.csv
```

如果指定了 `--start_index` 和 `--end_index`，输出文件名会变成：

```text
predictions_{start_index}_{end_index}.csv
```

## 5. 结果评估

主要评估脚本：

```text
llm/evaluate_llm_regex.py
```

DrugBank 示例：

```bash
python llm/evaluate_llm_regex.py \
  --prediction_path outputs/drugbank_meta-llama_Meta-Llama-3.1-8B-Instruct_kg_seed_0/predictions.csv \
  --dataset drugbank_inductive \
  --model_style default
```

DDInter 示例：

```bash
python llm/evaluate_llm_regex.py \
  --prediction_path outputs/ddinter_meta-llama_Meta-Llama-3.1-8B-Instruct_kg_seed_0/predictions.csv \
  --dataset ddinter \
  --model_style default
```

Tx-Gemma 模型输出格式可使用：

```bash
--model_style tx_gemma
```

评估脚本支持的 `--dataset` 选项：

```text
drugbank_transductive
drugbank_inductive
ddinter
pharmacotherapyDB
```

注意：代码里 pharmaDB 相关命名不完全一致。`argparse` 的 choices 使用 `pharmacotherapyDB`，但内部 `dataset_map` 中写的是 `pharmaDB`。复现 pharmaDB 评估时可能需要先修正脚本。

## 6. 使用 Tx-Gemma 模型

README 提到：

```text
Use llm/llm_inference_v2.py for Tx-Gemma models
```

因此如果模型是 Tx-Gemma 系列，优先使用：

```bash
python llm/llm_inference_v2.py ...
```

评估时配合：

```bash
--model_style tx_gemma
```

## 7. 重要注意事项

### 7.1 LLM 推理依赖 GPU

`llm/llm_inference.py` 使用：

```python
from vllm import LLM, SamplingParams
```

脚本中会调用：

```python
torch.cuda.device_count()
```

并将 `tensor_parallel_size` 设置为 GPU 数量。无 CUDA 环境下通常无法直接运行完整推理。

### 7.2 Hugging Face 权限

如果使用 Llama 等 gated 模型，需要：

1. 在 Hugging Face 页面申请模型访问权限。
2. 登录 Hugging Face。

```bash
huggingface-cli login
```

或者提前设置 token。

### 7.3 路径抽取默认参数

`k-paths/utils/Kpaths_utils.py` 中默认：

```text
max_length = 3
max_paths = 10
```

也就是说每个样本最多抽取 10 条 reasoning paths，路径最长 3 跳。

这两个参数没有在命令行中暴露。如果要修改，需要改源码中的 `find_Kpaths`、`find_Kpaths_no_filter` 等函数默认值，或给 `get-Kpaths.py` 增加命令行参数。

### 7.4 README 参数名有不一致

README 中路径抽取示例：

```bash
--dataset ddinter
```

代码真实参数：

```bash
--dataset_name ddinter
```

### 7.5 数据集命名不一致

不同脚本中同一个数据集名称可能不同：

```text
路径抽取脚本: pharmaDB
LLM 推理脚本: pharmacotherapydb
评估脚本 choices: pharmacotherapyDB
```

复现 pharmaDB 时尤其要注意。

### 7.6 评估脚本有硬编码测试集路径

`llm/evaluate_llm_regex.py` 内部有 `TEST_SET_PATH`：

```text
drugbank: data/paths/drugbank_test_add_reverse.json
ddinter: data/dataset_with_paths/data/ddinter_test.json
pharmaDB: data/pharmaDB_test.json
```

其中 ddinter 和 pharmaDB 的默认路径可能和路径抽取输出不一致。必要时需要修改为：

```text
data/paths/ddinter_test_add_reverse.json
data/paths/pharmaDB_test_add_reverse.json
```

### 7.7 先用 debug 跑通

建议先执行：

```bash
--debug
```

确认模型加载、数据字段、输出目录和评估脚本都能正常工作，再跑全量。

## 8. 最小可行复现流程

以下流程适合先验证 DrugBank 的 LLM 推理：

```bash
git clone https://github.com/rsinghlab/K-Paths.git
cd K-Paths

python3.10 -m venv .kpaths-env
source .kpaths-env/bin/activate
pip install -r requirements.txt
```

下载或准备：

```text
data/paths/drugbank_test_add_reverse.json
```

运行 debug 推理：

```bash
python llm/llm_inference.py \
  --dataset_path data/paths/drugbank_test_add_reverse.json \
  --dataset_name drugbank \
  --output_dir outputs \
  --model_name_or_path meta-llama/Meta-Llama-3.1-8B-Instruct \
  --use_kg \
  --debug
```

确认 `outputs/` 下生成 `predictions.csv` 后，再全量运行：

```bash
python llm/llm_inference.py \
  --dataset_path data/paths/drugbank_test_add_reverse.json \
  --dataset_name drugbank \
  --output_dir outputs \
  --model_name_or_path meta-llama/Meta-Llama-3.1-8B-Instruct \
  --use_kg
```

最后评估：

```bash
python llm/evaluate_llm_regex.py \
  --prediction_path outputs/<实际输出目录>/predictions.csv \
  --dataset drugbank_inductive \
  --model_style default
```

## 9. 建议排查顺序

如果运行失败，按这个顺序检查：

1. `data/paths/*.json` 是否存在。
2. `--dataset_name` 是否和输入文件对应。
3. JSON 中是否包含 `path_str`、`label_idx`、药物名等字段。
4. Hugging Face 模型是否有访问权限。
5. CUDA/GPU 是否可用。
6. `vllm` 是否与当前 CUDA、PyTorch、Python 版本兼容。
7. 评估脚本中的 `TEST_SET_PATH` 是否指向正确测试集。

