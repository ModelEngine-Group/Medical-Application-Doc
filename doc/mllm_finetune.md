# 基于Qwen2.5-VL的病理多模态大模型监督微调实践

## 一、环境安装

### 环境准备

请参考[安装指南](https://gitee.com/ascend/MindSpeed-MM/blob/master/docs/user-guide/installation.md)，完成昇腾训练软件安装。

### 环境搭建

```
git clone https://gitee.com/ascend/MindSpeed-MM.git
git clone https://github.com/NVIDIA/Megatron-LM.git
cd Megatron-LM
git checkout core_v0.12.1
cp -r megatron ../MindSpeed-MM/
cd ..
cd MindSpeed-MM
mkdir logs data ckpt
# 安装加速库
git clone https://gitee.com/ascend/MindSpeed.git
cd MindSpeed
# checkout commit from MindSpeed core_r0.12.1
git checkout 5176c6f5f133111e55a404d82bd2dc14a809a6ab
# 安装mindspeed及依赖
pip install -e .
cd ..
# 安装mindspeed mm及依赖
pip install -e .
```

## 二、预训练模型准备

### 权重下载

以[Qwen2.5-VL-32B](https://huggingface.co/Qwen/Qwen2.5-VL-32B-Instruct/tree/main)为例，从Huggingface库下载对应的模型权重:

* 模型地址: [Qwen2.5-VL-32B](https://huggingface.co/Qwen/Qwen2.5-VL-32B-Instruct/tree/main)；

将下载的模型权重保存到本地的 `ckpt/hf_path/Qwen2.5-VL-32B-Instruct`目录下。

### 权重转换(hf2mm)

使用 `mm-convert`工具对原始预训练权重进行转换。该工具实现了huggingface权重和MindSpeed-MM权重的互相转换以及PP（Pipeline Parallel）权重的重切分。（参考[权重转换工具](https://gitee.com/ascend/MindSpeed-MM/blob/master/docs/features/%E6%9D%83%E9%87%8D%E8%BD%AC%E6%8D%A2%E5%B7%A5%E5%85%B7.md)）

![1774594183060](image/mllm/1774594183060.png)

```
# 其中：
# mm_dir: 转换后保存目录
# hf_dir: huggingface权重目录
# llm_pp_layers: llm在每个卡上切分的层数，注意要和model.json中配置的pipeline_num_layers一致
# vit_pp_layers: vit在每个卡上切分的层数，注意要和model.json中配置的pipeline_num_layers一致
# tp_size: tp并行数量，注意要和微调启动脚本中的配置一致
```

以上是TP4PP4切分后的权重转换参考命令，如果未切分可以执行命令：

```
mm-convert  Qwen2_5_VLConverter hf_to_mm \
  --cfg.mm_dir "ckpt/mm_path/Qwen2.5-VL-32B-Instruct" \
  --cfg.hf_config.hf_dir "ckpt/hf_path/Qwen2.5-VL-32B-Instruct" \
  --cfg.parallel_config.llm_pp_layers [[1,9,9,9,9,9,9,9]] \
  --cfg.parallel_config.vit_pp_layers [[32,0,0,0,0,0,0,0]] \
  --cfg.parallel_config.tp_size 2
```

### 权重转换(mm2hf)

在微调后，如果需要将权重转回huggingface格式，可使用 `mm-convert`权重转换工具对微调后的权重进行转换，将权重名称修改为与原始网络一致。

![1774594473533](image/mllm/1774594473533.png)

### 权重切分

如果需要对权重重新进行切分，可使用 `mm-convert`权重转换工具对权重进行切分。

参考命令：

```
mm-convert  Qwen2_5_VLConverter resplit \
  --cfg.source_dir "ckpt/mm_path/Qwen2.5-VL-32B-Instruct" \
  --cfg.target_dir "ckpt/mm_resplit_pp/Qwen2.5-VL-32B-Instruct" \
  --cfg.source_parallel_config.llm_pp_layers [1,9,9,9,9,9,9,9]  \
  --cfg.source_parallel_config.vit_pp_layers [32,0,0,0,0,0,0,0] \
  --cfg.source_parallel_config.tp_size 2 \
  --cfg.target_parallel_config.llm_pp_layers [13,17,17,17] \
  --cfg.target_parallel_config.vit_pp_layers [32,0,0,0] \
  --cfg.target_parallel_config.tp_size 4
# 其中
# source_dir: 微调后保存的权重目录
# target_dir: 希望重新pp切分后保存的目录
# source_parallel_config.llm_pp_layers: 微调时llm的pp配置
# source_parallel_config.vit_pp_layers: 微调时vit的pp配置
# source_parallel_config.tp_size: 微调时tp并行配置
# target_parallel_config.llm_pp_layers: 期望的重切分llm模块切分层数
# target_parallel_config.vit_pp_layers: 期望的重切分vit模块切分层数
# target_parallel_config.tp_size: 期望的tp并行配置（tp_size不能超过原仓config.json中的num_key_value_heads
```

## 三、数据准备

### 数据下载和处理

下载开源病理ROI图文对数据，也可以准备自己的病理图文对数据。

参考开源ROI图文对数据地址：

* [PathGen-Instruct](https://huggingface.co/datasets/jamessyx/PathGen-Instruct)
* [PathInstruct](https://huggingface.co/datasets/jamessyx/PathInstruct)
* [PathGen](https://huggingface.co/datasets/jamessyx/PathGen)

  可根据需要和具体实际情况对数据进行清洗和翻译。
* 例如，通过下面的 DeepSeek 提示词对数据集中的英文 Caption 进行翻译，，整理为镜下所
  见基本事实模板格式：

```
TRANSLATION_PROMPT = """作为专业病理学翻译员，你需要将以下WSI数字病理图片的ROI区域镜下描述
从英文准确翻译为简体中文。要求：
1. 严格保持医学准确性，使用标准病理学术语。
2. 保持数值数据和测量单位的准确性。
3. 保持原文的临床描述严谨性。
4. 以提供的文本为准，如果有些维度没有相关内容，可以为空，不要引申生成
首先输出翻译后的内容，然后输出 “<格式化>” 文本，最后按照下列关键形态学描述维度重新组织并输出：- 生长方式- 组织学结构- 细胞形态- 间质的类型- 伴随现象- 继发改变- 特殊之处
只输出上述内容，不要引申，不要解释。
需要翻译的内容如下："""
```

翻译后的数据样例如下：

```
{
"wsi_id": "TCGA-AA-3844-01Z-00-DX1.bf88ce1f-0601-40c8-813e-4e3df51bd2f0",
"position": [
"35136",
"33344"
],
"caption": "翻译内容：结肠组织显示多形性、核深染及不规则腺体结构，提示肿瘤性病变。间质可见
炎性浸润及细胞密度增加，表明存在促结缔组织增生反应。这些特征可能指向腺癌，需结合进一步临床和分子
检测以明确诊断。",
"file_id": "bffacf34-4942-496d-9c5d-d36294d80a9d",
"english_caption": "The colon tissue exhibits pleomorphism, hyperchromatic 
nuclei, and irregular glandular architecture, indicative of a neoplastic process. 
Stroma shows inflammatory infiltration and increased cellularity, suggesting a 
desmoplastic reaction. These characteristics potentially point to adenocarcinoma, 
requiring further clinical and molecular correlation for a definitive 
diagnosis.",
"translated_comment": "- 生长方式：肿瘤性病变- 组织学结构：不规则腺体结构- 细胞形态：
多形性、核深染- 间质的类型：促结缔组织增生反应- 伴随现象：炎性浸润- 继发改变：细胞密度增加- 特
殊之处：需结合临床和分子检测明确诊断"
}
```

### 数据构造

在数据构造时，对于包含图片的数据，构造格式如下：

```
{
  "id": your_id,
  "image": your_image_path,
  "conversations": [
      {"from": "human", "value": your_query},
      {"from": "gpt", "value": your_response},
  ],
}
```

对于纯文本数据，可以去除 `image`这个键值，构造格式如下：

```


  "id": your_id,
  "conversations": [
      {"from": "human", "value": your_query},
      {"from": "gpt", "value": your_response},
  ],
}
```

真实的构造后病理ROI图文对数据样式如下：

```
{
"images": [
"/pkgs/patches/TCGA-AA-3844-01Z-00-DX1.bf88ce1f-0601-40c8-813e
4e3df51bd2f0_35136_33344.png"
],
"messages": [
{
"role": "user",
"content": "![]()

生成这张病理 WSI 图片 ROI 区域截图的镜下所见描述。注意：只描述
识别到的内容，不要做任何推论，不要下诊断结论。可以从生长方式、组织学结构、细胞形态、间质的类型、
伴随现象、继发改变以及特殊之处这几个维度做描述。"
},
{
"role": "assistant",
"content": "- 生长方式：肿瘤性病变- 组织学结构：不规则腺体结构- 细胞形态：多形性、核
深染- 间质的类型：促结缔组织增生反应- 伴随现象：炎性浸润- 继发改变：细胞密度增加- 特殊之处：需
结合临床和分子检测明确诊断"
}
]
}
```

### 模型身份认知数据准备

除了病理ROI图文对数据外，由于训练后的模型为病理多模态大模型，需要配比无图片的文本QA对，让模型学会新的身份认。

```
[
{
"instruction": "你是谁？",
"input": "",
"output": "您好，我是 {{name}}，一个由 {{author}} 发明的人工智能助手。我可以专业
解读病理 WSI 切片图像切分的 ROI 区域截图并生成镜下所见文本描述。"
},
{
"instruction": "你好，请介绍一下你自己",
"input": "",
"output": "您好，我是 {{name}}，一个由 {{author}} 开发的人工智能助手，我可以专业
解读病理 WSI 切片图像切分的 ROI 区域截图并生成镜下所见文本描述。"
}
]
```

## 四、监督微调

配置脚本前需要完成前置准备工作，包括：**环境安装**、**预训练模型准备** 、 **数据集准备**，详情可查看对应章节。

### 数据目录配置

根据实际情况修改 `data.json`中的数据集路径，包括 `model_name_or_path`、`dataset_dir`、`dataset`等字段。以Qwen2.5VL-32B为例，`data.json`进行以下修改，注意 `model_name_or_path`的权重路径为转换前的权重路径。

```
{
    "dataset_param": {
        "dataset_type": "huggingface",
        "preprocess_parameters": {
            "model_name_or_path": "./ckpt/hf_path/Qwen2.5-VL-7B-Instruct",
            ...
        },
        "basic_parameters": {
            ...
            "dataset_dir": "./data",
            "dataset": "./data/mllm_format_your_instruct_data.json",
            "cache_dir": "./data/cache_dir",
            ...
        },
        ...
    },
    ...
}
```

### 模型保存加载及日志信息配置

根据实际情况配置 `examples/qwen2.5vl/finetune_qwen2_5_vl_32b.sh`的参数，包括加载、保存路径以及保存间隔 `--save-interval`（注意：分布式优化器保存文件较大耗时较长，请谨慎设置保存间隔）

```
...
# 加载路径
LOAD_PATH="ckpt/mm_path/Qwen2.5-VL-32B-Instruct"
# 保存路径
SAVE_PATH="save_dir"
...
GPT_ARGS="
    ...
    --no-load-optim \  # 不加载优化器状态，若需加载请移除
    --no-load-rng \  # 不加载随机数状态，若需加载请移除
    --no-save-optim \  # 不保存优化器状态，若需保存请移除
    --no-save-rng \  # 不保存随机数状态，若需保存请移除
    ...
"
...
OUTPUT_ARGS="
    --log-interval 1 \  # 日志间隔
    --save-interval 5000 \  # 保存间隔
    ...
    --log-tps \  # 增加此参数可使能在训练中打印每步语言模块的平均序列长度，并在训练结束后计算每秒吞吐tokens量。
"
```

若需要加载指定迭代次数的权重、优化器等状态，需将加载路径 `LOAD_PATH`设置为保存文件夹路径 `LOAD_PATH="save_dir"`，并修改 `latest_checkpointed_iteration.txt`文件内容为指定迭代次数 (此功能coming soon)

```
$save_dir
   ├── latest_checkpointed_iteration.txt
   ├── ...
```

根据实际情况修改配置 `examples/qwen2.5vl/finetune_qwen2_5_vl_32b.sh`参数

```
# 根据实际情况修改 ascend-toolkit 路径
source /usr/local/Ascend/ascend-toolkit/set_env.sh
NPUS_PER_NODE=8
MASTER_ADDR=localhost
MASTER_PORT=29501
NNODES=1
NODE_RANK=0
WORLD_SIZE=$(($NPUS_PER_NODE * $NNODES))
```

### 微调训练

执行配置好参数的脚本：

```
bash examples/qwen2.5vl/finetune_qwen2_5_vl_32b.sh
```
