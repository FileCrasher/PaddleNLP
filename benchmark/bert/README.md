# BERT Benchmark with Fleet API
BERT - Bidirectional Encoder Representations from Transformers [论文链接](https://arxiv.org/abs/1810.04805)
PaddlePaddle实现了BERT的预训练模型（Pre-training）和下游任务(Fine-tunning)。
## 数据集
### Pre-training数据集

### Fine-tunning数据集
在fine-tunning数据集上集成了glue任务榜的数据集，在代码示例中主要是提供SST-2和QNLI的下游任务的fine-tuning方法，同时示例程序中可以下载好训练数据和测试数据，可以直接进行模型训练。具体的glue相关数据和任务类型可以见链接[glue任务链接](https://gluebenchmark.com/tasks)。

## Pre-training任务训练
### 环境变量设置
1. paddlenlp的安装
pip install paddlenlp==2.0.0b0 -i https://pypi.org/simple

2. 设置预训练的数据地址环境变量
```shell
export DATA_DIR=${HOME}/bert_data/wikicorpus_en
```

### 运行模型训练脚本
```shell

1. 如果是需要单机多卡/多机多卡训练，则使用下面的命令进行训练
unset CUDA_VISIBLE_DEVICES
fleetrun --gpus 0,1,2,3 ./run_pretrain.py \
    --model_type bert \
    --model_name_or_path bert-base-uncased \
    --max_predictions_per_seq 20 \
    --batch_size 32   \
    --learning_rate 1e-4 \
    --weight_decay 1e-2 \
    --adam_epsilon 1e-6 \
    --warmup_steps 10000 \
    --input_dir $DATA_DIR \
    --output_dir ./tmp2/ \
    --logging_steps 1 \
    --save_steps 20000 \
    --max_steps 1000000

2. 如果是需要多单机多卡/多机多卡训练，则使用下面的命令进行训练
export CUDA_VISIBLE_DEVICES=0
python ./run_pretrain_single.py \
    --model_type bert \
    --model_name_or_path bert-base-uncased \
    --max_predictions_per_seq 20 \
    --batch_size 32   \
    --learning_rate 1e-4 \
    --weight_decay 1e-2 \
    --adam_epsilon 1e-6 \
    --warmup_steps 10000 \
    --input_dir $DATA_DIR \
    --output_dir ./tmp2/ \
    --logging_steps 1 \
    --save_steps 20000 \
    --max_steps 1000000
```

## Fine-tuning任务训练

在完成 BERT 模型的预训练后，即可利用预训练参数在特定的 NLP 任务上做 Fine-tuning。以下利用开源的预训练模型，示例如何进行分类任务的 Fine-tuning。

```shell
export CUDA_VISIBLE_DEVICES=0
export TASK_NAME=SST-2

python -u ./run_glue.py \
    --model_type bert \
    --model_name_or_path bert-base-uncased \
    --task_name $TASK_NAME \
    --max_seq_length 128 \
    --batch_size 64   \
    --learning_rate 2e-5 \
    --num_train_epochs 3 \
    --logging_steps 1 \
    --save_steps 500 \
    --output_dir ./tmp/$TASK_NAME/
```

其中参数释义如下：
- `model_type` 指示了模型类型，当前仅支持BERT模型。
- `model_name_or_path` 指示了某种特定配置的模型，对应有其预训练模型和预训练时使用的 tokenizer。若模型相关内容保存在本地，这里也可以提供相应目录地址。
- `task_name` 表示 Fine-tuning 的任务。
- `max_seq_length` 表示最大句子长度，超过该长度将被截断。
- `batch_size` 表示每次迭代**每张卡**上的样本数目。
- `learning_rate` 表示基础学习率大小，将于learning rate scheduler产生的值相乘作为当前学习率。
- `num_train_epochs` 表示训练轮数。
- `logging_steps` 表示日志打印间隔。
- `save_steps` 表示模型保存及评估间隔。
- `output_dir` 表示模型保存路径。

使用以上命令进行单卡 Fine-tuning ，在验证集上有如下结果：

| Task  | Metric                       | Result      |
|-------|------------------------------|-------------|
| CoLA  | Matthews corr                | 59.90       |
| SST-2 | Accuracy                     | 92.76       |
| STS-B | Pearson/Spearman corr        | 89.12       |
| MNLI  | matched acc./mismatched acc. | 84.45/84.62 |
| QNLI  | acc.                         | 91.73       |
| RTE   | acc.                         | 67.15       |
