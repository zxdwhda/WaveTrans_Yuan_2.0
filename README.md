快速部署（正在修改，不建议参考）
Step1：在魔搭社区打开PAI实例！（点击即可跳转）
点击打开，没有创建的同学，返回原文档创建——动手学大模型应用全栈开发
[图片]

进入实例，点击终端。
[图片]
复制，运行下面代码，下载文件，解压、删除。
git lfs install
git clone https://www.modelscope.cn/datasets/zxdwhda/rendering_baseline.git
unzip rendering_baseline/rendering_baseline.zip
rm -r rendering_baseline
打开step2/Step2：微调.ipynb 文件
[图片]
点击“运行所有单元格”
[图片]
完成后的状态：
[图片]
点击重启内核，释放内存
https://bq2vlvngbwu.feishu.cn/sync/Fxe0dj0zWsRfcdbZMMsc80i7nef
运行翻译机器人.py
streamlit run 翻译机器人.py --server.address 127.0.0.1 --server.port 6006
点击
 URL: http://127.0.0.1:6006
[图片]
我们输入需要翻译的英文，如下
[图片]
🎉恭喜成功搭建了自己的第一个翻译助手！
👍🏻 快来给自己点赞！
暂时无法在飞书文档外展示此内容
可能你测试以后发现效果不好，这是因为训练集太少，以及参数设置为达到最优。
项目部署详解
在写，没有写完
Step1：在魔搭社区打开PAI实例！（点击即可跳转）
点击打开，没有创建的同学，返回原文档创建——动手学大模型应用全栈开发
[图片]
Step2：微调
本节我们将进行 源2.0-2B 微调实战，我们使用一个中英机器翻译生物医药领域测试集进行微调（优质数据太少），进而开发一个AI翻译助手。

具体来说，输入英文句子，翻译输出中文句子原始的数据来自于WMT中英机器翻译医药测试集
1. 环境准备

进入实例，点击终端。
[图片]
复制，运行下面代码，下载文件。
git lfs install
git clone https://www.modelscope.cn/datasets/zxdwhda/rendering_baseline.git
unzip rendering_baseline/rendering_baseline.zip
rm -r rendering_baseline
[图片]
打开step2/Step2：微调.ipynb 文件
[图片]
点击“运行所有单元格”
[图片]
通过下面的命令，我们可以看到ModelScope已经提供了所需要的大部分依赖，如 torch，transformers 等。

# 查看已安装依赖
pip list

[图片]

但是为了进行Demo搭建，还需要运行下面的单元格，在环境中安装streamlit。

# 安装 streamlit
pip install streamlit==1.24.0

安装成功后，我们的环境就准备好了。

2. 模型下载

Yuan2-2B-Mars支持通过多个平台进行下载，因为我们的机器就在魔搭，所以这里我们直接选择通过魔搭进行下载。

模型在魔搭平台的地址为 IEITYuan/Yuan2-2B-Mars-hf。

单元格 2.1 模型下载 会执行模型下载。

这里使用的是 modelscope 中的 snapshot_download 函数，第一个参数为模型名称 IEITYuan/Yuan2-2B-Mars-hf，第二个参数 cache_dir 为模型保存路径，这里.表示当前路径。

模型大小约为4.1G，由于是从魔搭直接进行下载，速度会非常快。

下载完成后，会在当前目录增加一个名为 IEITYuan 的文件夹，其中 Yuan2-2B-Mars-hf 里面保存着我们下载好的源大模型。

[图片]

3. 数据处理

运行 2.3 数据处理 下面的单元格。

因为当前安装的Keras版本（Keras 3）与Transformers库不兼容。Transformers库尚未支持Keras 3。为了解决这个问题，您需要安装与TensorFlow兼容的旧版Keras，即tf-keras。
[图片]

我们使用 pandas 进行数据读取，然后转成 Dataset 格式：
[图片]
然后我们需要加载 tokenizer:

[图片]

为了完成模型训练，需要完成数据处理，这里我们定义了一个数据处理函数 process_func：

def process_func(example):
    MAX_LENGTH = 384    # Llama分词器会将一个中文字切分为多个token，因此需要放开一些最大长度，保证数据的完整性

    instruction = tokenizer(f"{example['input']}<sep>")
    response = tokenizer(f"{example['output']}<eod>")
    input_ids = instruction["input_ids"] + response["input_ids"]
    attention_mask = [1] * len(input_ids) 
    labels = [-100] * len(instruction["input_ids"]) + response["input_ids"] # instruction 不计算loss

    if len(input_ids) > MAX_LENGTH:  # 做一个截断
        input_ids = input_ids[:MAX_LENGTH]
        attention_mask = attention_mask[:MAX_LENGTH]
        labels = labels[:MAX_LENGTH]

    return {
        "input_ids": input_ids,
        "attention_mask": attention_mask,
        "labels": labels
    }

具体来说，需要使用tokenizer将文本转成id，同时将 input 和 output 拼接，组成 input_ids 和 attention_mask 。
这里我们可以看到，源大模型需要在 input 后面添加一个特殊的token <sep>， 在 output 后面添加一个特殊的token <eod>。
同时，为了防止数据超长，还有做一个截断处理。

然后使用 map 函数对整个数据集进行预处理：
[图片]
处理完成后，我们使用tokenizer的decode函数，将id转回文本，进行最后的检查：
[图片]

4. 模型训练

首先我们需要加载源大模型参数，然后打印模型结构：

[图片]

可以看到，源大模型中包含24层 YuanDecoderLayer，每层中包含 self_attn、mlp 和 layernorm。

另外为了进行模型使用训练，需要先执行 model.enable_input_require_grads() 。

[图片]

最后，我们打印下模型的数据类型，可以看到是 torch.bfloat16。

[图片]

在本节中，我们使用Lora进行轻量化微调，首先需要配置 LoraConfig：
- task_type=TaskType.CAUSAL_LM：指定任务的类型。在这里，CAUSAL_LM代表因果语言模型，通常用于生成任务，例如文本生成。
- target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"]：这个列表包含了模型中将要应用Lora适应的模块名称。这些通常是多头注意力机制中的投影层。
- inference_mode=False：这个参数表示我们处于训练模式。如果设置为True，则表示处于推理模式。
- r=8：这是Lora的秩（rank），它决定了适应层的大小。较小的秩意味着更少的参数，从而减少过拟合的风险，但可能降低模型的表达能力。
- lora_alpha=32：Lora的alpha参数，它是一个超参数，用于缩放Lora矩阵A和B的乘积，影响适应层对模型输出的贡献。
- lora_dropout=0.1：这是Lora适应层中的dropout比例，用于正则化，以减少过拟合。

[图片]

然后构建一个 PeftModel:

[图片]

通过 model.print_trainable_parameters()，可以看到需要训练的参数在所有参数中的占比：

[图片]

然后，我们设置训练参数 TrainingArguments:
- output_dir="./output/Yuan2.0-2B_lora_bf16"：指定训练结果的输出目录。
- per_device_train_batch_size=12：每个 GPU 设备上的训练批次大小。
- gradient_accumulation_steps=1：梯度累积的步数。设置为 1 意味着每一步都会更新模型权重。
- logging_steps=1：日志记录的步数间隔。设置为 1 意味着每一步都会记录日志。
- save_strategy="epoch"：保存模型的策略，设置为 “epoch” 表示每个训练周期结束后保存模型。
- num_train_epochs=3：训练的总周期数。
- learning_rate=1e-4：训练的学习率。
- save_on_each_node=True：当在多个节点上训练时，是否在每个节点上保存模型。
- gradient_checkpointing=True：是否启用梯度检查点，这可以减少内存使用，但可能会增加训练时间。
- bf16=True：是否使用 bfloat16 精度进行训练，这可以减少内存使用并可能加速训练。
这里的训练的总周期数、学习率建议大家去改。
    num_train_epochs=,
    learning_rate=,
[图片]

同时初始化一个 Trainer:

[图片]

最后运行 trainer.train() 执行模型训练。

[图片]

在训练过程中，会打印模型训练的loss，我们可以通过loss的降低情况，检查模型是否收敛：

[图片]

模型训练完成后，会打印训练相关的信息：

[图片]

同时，我们会看到左侧 output 文件夹下出现了3个文件夹，每个文件夹下保存着每个epoch训完的模型。
[图片]

这里，以epoch3为例，可以看到其中保存了训练的config、state、ckpt等信息。

[图片]

2.5 效果验证

完成模型训练后，我们通过定义一个生成函数 generate()，来进行效果验证：

[图片]

同时定义好输入的prompt template，这个要和训练保持一致。

[图片]

最后，我们输入样例进行测试：

[图片]

可以看到，通过模型微调，模型已经具备了相应的能力

训练完成后，我们尝试使用训练好的模型，搭建Demo。

首先，点击重启内核，清空显存。
（因为Demo也需要占用显存，不先清空会因为显存不足报错。）

https://bq2vlvngbwu.feishu.cn/sync/Fxe0dj0zWsRfcdbZMMsc80i7nef

然后，我们将在终端输入下面的命令，启动streamlit服务：

streamlit run 翻译机器人.py --server.address 127.0.0.1 --server.port 6006
然后点击链接，跳转到浏览器，进入Demo：
[图片]


输入文本：Theexpression of GFPin P.pastoris canbe greatlyreduced when a repressorregioncomposed
offourcontinuousprolinerarecodonCCGwasaddedintotheGFPgene.

[图片]
可以看到，Demo完成了翻译，并进行了输出展示。
这样我们完成了一个AI翻译助手的构建。
