---
title: 轻薄本跑不动 MOSS-Image？用 Kaggle 免费 GPU 从部署到出图
date: 2026-11-01
tags:
  - AI
  - 文生图
  - Kaggle
  - 实践
status: completed
---

# 轻薄本跑不动 MOSS-Image？用 Kaggle 免费 GPU 从部署到出图

我有一台 16GB 内存的 MacBook Air。平时用它跑 BERT、做小规模微调都没问题，所以第一次看到 MOSS-Image 的介绍时，我的关注点很简单：**生图模型只有 5B 参数，是不是也能本地跑？**

实际查完模型结构才发现，这个“5B”很容易让人误判。它只是负责生图的扩散 Transformer，完整推理还需要一个 9B 文本编码器和一个图像解码器。仅文本编码器的 BF16 权重就接近 18GB，已经超过我这台电脑的全部内存。

后来我把运行环境搬到了 Kaggle：两张免费的 T4 GPU 确实能加载模型，但又遇到了一个典型问题——**两张 16GB 显卡不等于一张 32GB 显卡**。如果直接把整套模型交给默认加载流程，它仍然可能试图把文本编码器塞进第一张卡，然后在出图之前就显存溢出。

这篇文章记录 MOSS-Image 的基本用法、免费算力的选择，以及在 Kaggle 双 T4 环境中的具体部署方式。重点不是罗列命令，而是解释每一步为什么这样做，以及哪些报错可以忽略、哪些必须换方案。

> [!info] 实测范围
> 文中环境为 Kaggle Notebook、双 Tesla T4、MOSS-Image-SFT。模型下载、组件加载和“CPU 编码提示词 + 第二张 T4 生图”的执行路径已经实际走到；写作时尚未记录完整出图耗时和最终画质，因此不把这条路包装成已经验证过性能的生产方案。

## 一、MOSS-Image 是什么

[MOSS-Image](https://github.com/OpenMOSS/MOSS-Image) 是 OpenMOSS 团队开源的文生图模型系列，代码和模型权重采用 Apache 2.0 许可证。目前主要有两个版本：

| 版本 | 用途 |
| --- | --- |
| MOSS-Image-Base | 基础文生图模型，适合研究、二次训练和领域适配 |
| MOSS-Image-SFT | 在精选数据上监督微调，通常更适合直接体验文生图 |

它比较鲜明的特点有：

- **图内文字渲染**：适合测试海报、便签、白板等包含中英文文字的画面。
- **布局控制**：除了自然语言，还能输入 JSON，描述文字、物体及其位置。
- **长提示词**：最长支持 4096 个文本 token。
- **便于二次训练**：5B 生图主干比许多更大的扩散模型更容易做 LoRA 等适配。

官方推荐的默认采样配置是 Euler 采样器、50 步、CFG 3、Shift 5，并使用第 22 层的负向分支跳层引导。首次测试可以降低分辨率和步数，先确认整条流程能够运行。

### “5B 模型”为什么需要二十多 GB 权重

完整推理不是只有一个神经网络，而是三个组件协作：

1. **文本编码器**读取提示词，把文字转换成条件向量。
2. **扩散 Transformer（DiT）**根据条件向量逐步去噪，这是宣传中所说的 5B 生图主干。
3. **VAE**把去噪后的潜空间表示解码成最终图片。

MOSS-Image 使用 Qwen3.5-9B 作为文本编码器，生图主干是 OpenMOSS 自己训练的模型。它不是拿某个 Qwen 对话模型直接做后训练得到的文生图模型。

我在 Kaggle 实际下载 MOSS-Image-SFT 时，下载日志显示约 23.4GB，展开后的模型约 27.2GB。这个数量级与“只有一个 5B、约 10GB 的 BF16 模型”完全不同。

> [!important] 判断显存需求时看完整推理管线
> 模型名称里的参数量只描述部分组件。对文生图、多模态和组合式模型，应该同时检查文本编码器、生图主干、解码器以及推理时的中间激活。

## 二、有哪些免费使用方式

### 只想快速体验：官方在线演示

最省事的是 [Hugging Face 官方演示](https://huggingface.co/spaces/OpenMOSS-Team/MOSS-Image)。不用准备环境，也不用下载权重，但免费 Space 可能排队，速度受共享资源和当前负载影响。

如果只是想看看模型风格，这仍然是成本最低的方式。可以降低步数、先用较小分辨率测试构图，不必一开始就按 50 步和高分辨率生成。

### 想运行自己的代码：Kaggle 免费 GPU

需要修改提示词格式、调整采样参数或保存生成结果时，Kaggle 更合适。我的 Notebook 中可以选择 **GPU T4 x2**，界面显示每周 30 小时配额。

这个方案的优点是免费、环境完整、能直接运行 Python；代价是资源有限、会话会结束，而且双卡需要自行安排模型组件。

类似平台还有 Google Colab 免费层、Lightning AI 的免费额度等，但硬件、可用时长和活动政策会变化。选择时应该以平台当前页面为准，不要把某次活动额度当成长期承诺。

> [!note] API 赠额和 GPU 算力不是一回事
> 一些国内平台会赠送大模型 API token，这对调用聊天模型很有用，但不等于获得了一台可以自行安装扩散模型的 GPU 机器。要运行 MOSS-Image 官方代码，需要的是带显存的计算环境，或者明确提供该模型的托管生图接口。

## 三、在 Kaggle 中准备运行环境

### 1. 完成手机号验证

GPU 功能需要可用的账号配额。进入 Kaggle 账号设置，找到手机验证并完成验证。如果 Notebook 中没有 GPU 选项，先检查这一项。

### 2. 创建 Notebook 并选择双 T4

新建 Notebook，在右侧会话选项中设置：

- Accelerator：**GPU T4 x2**
- Internet：**开启**
- Persistence：测试阶段可以不开启

加速器在 **Notebook 内部的会话设置**里，不在账号的全局设置页里。

随后清空示例代码，在第一格运行：

```python
!nvidia-smi
```

预期能看到两张 Tesla T4。每张标称 15360MiB，PyTorch 报错中显示的实际可用容量约为 14.56GiB。

### 3. 下载代码并安装依赖

新建第二个代码格：

```python
!git clone https://github.com/OpenMOSS/MOSS-Image.git
%cd MOSS-Image
!pip install -q -r requirements.txt
!pip install -q 'diffusers>=0.38' 'transformers>=5.3' 'accelerate>=1.10' gradio
```

Kaggle 可能提示某些预装包存在依赖冲突。如果冲突与 MOSS-Image 无关，先不要为此升级或降级整个环境；等出现真实的导入错误再处理。

## 四、双 T4 为什么仍然容易显存溢出

我最开始按照普通单卡模型的方式加载整条管线，并分别尝试了组件卸载和逐层卸载。两次都在**文本编码阶段**失败，报错显示第一张卡已占用约 14.49GiB，无法继续分配显存。

原因并不复杂：

- 文本编码器的 BF16 权重接近 18GB，本身就无法完整放进一张 T4。
- 默认加载和卸载流程不一定会按照我们希望的方式拆分两个大组件。
- 第二张 T4 不会自动成为第一张卡的显存扩展。

因此，双卡方案的重点不是“把两张卡都选上”，而是**明确哪个组件在哪张卡上运行，以及什么时候运行**。

对于这套模型，我采用的分工是：

| 阶段 | 运行位置 | 原因 |
| --- | --- | --- |
| 提示词编码 | CPU | 9B 编码器放不进单张 T4，先绕开显存限制 |
| DiT 去噪 | 第二张 T4 | 5B 生图主干可以独立放入显存 |
| VAE 解码 | 第二张 T4 | 与生图阶段共用同一张卡，避免来回传输 |

这也是一种更通用的思路：文本编码只需要在生成前执行一次，得到向量后就可以释放编码器；后续更换随机种子、分辨率和采样步数时，不必重复编码同一条提示词。

## 五、Kaggle 双 T4 运行代码

如果之前已经尝试过其他加载方式并报过显存错误，先检查显存：

```python
!nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv
```

若第一张卡仍有十几 GB 被旧进程占用，简单删除变量未必能清干净。此时应重启 Notebook 会话，然后**不要重新运行旧代码格**。重启可能要求重新下载模型，所以能在不重启的情况下清理时，不必把重启当成常规操作。

在干净会话中运行下面这份代码。它把文本编码留在 CPU，把生图模型放到第二张 T4：

```python
import gc
import torch
from diffusers import DiffusionPipeline
from IPython.display import display

pipe = DiffusionPipeline.from_pretrained(
    "OpenMOSS-Team/MOSS-Image-SFT",
    custom_pipeline="pipeline_moss_image",
    trust_remote_code=True,
    torch_dtype={
        "default": torch.bfloat16,
        "vae": torch.float32,
    },
)
pipe.scheduler = type(pipe.scheduler).from_config(pipe.scheduler.config)

print("将生图模型放到第二张 T4……")
pipe.transformer.to(device="cuda:1", dtype=torch.float16)
pipe.vae.to("cuda:1")
torch.cuda.empty_cache()


def encode_on_cpu(text):
    formatted = pipe.tokenizer.apply_chat_template(
        [{"role": "user", "content": text}],
        tokenize=False,
        add_generation_prompt=True,
        enable_thinking=True,
    )
    inputs = pipe.tokenizer(
        formatted,
        return_tensors="pt",
        truncation=True,
        max_length=128,
    )
    with torch.inference_mode():
        output = pipe.text_encoder(
            **inputs,
            use_cache=False,
            return_dict=True,
        )
    return output.last_hidden_state, inputs["attention_mask"].bool()


print("正在用 CPU 编码提示词，这一步可能较慢……")
prompt_embeds, prompt_attention_mask = encode_on_cpu(
    "一只橘猫坐在窗边，柔和的晨光，写实摄影风格"
)
negative_embeds, negative_attention_mask = encode_on_cpu("")

print("文本编码完成，释放编码器……")
pipe.text_encoder = None
pipe.tokenizer = None
gc.collect()

pipe.to("cuda:1")

print("开始生成图片……")
with torch.inference_mode():
    image = pipe(
        prompt=None,
        prompt_embeds=prompt_embeds,
        prompt_attention_mask=prompt_attention_mask,
        negative_prompt_embeds=negative_embeds,
        negative_prompt_attention_mask=negative_attention_mask,
        width=512,
        height=512,
        num_inference_steps=20,
        guidance_scale=3.0,
        uncond_skip_layers=[22],
        generator=torch.Generator("cpu").manual_seed(42),
    ).images[0]

display(image)
image.save("/kaggle/working/moss_test.png")
print("完成：/kaggle/working/moss_test.png")
```

第一次运行需要下载二十多 GB 权重。保持页面开启，耐心等待，不要重复点击运行。模型加载后，CPU 编码也可能需要一段时间；看到“开始生成图片”才说明进入去噪阶段。

这份代码为首次测试做了几处取舍：

- 分辨率为 512×512，减少首次测试的计算量。
- 采样为 20 步，而不是追求最佳质量的 50 步。
- 文本长度限制为 128，降低编码成本，也意味着长提示词会被截断。
- DiT 转为 FP16，以兼顾老款 T4 的兼容性。官方默认使用 BF16，这种改动可能影响数值表现，正式评估画质时应单独对照。
- 空负向提示词配合第 22 层跳层引导，沿用官方推荐思路。

## 六、修改提示词和生成参数

首次生成成功后，不必重新安装依赖或下载模型。新代码格中只重新编码提示词，再执行生成部分即可。

### 普通提示词

替换编码函数中的文字：

```python
prompt_embeds, prompt_attention_mask = encode_on_cpu(
    "雨后的城市街道，霓虹灯倒映在路面，电影感摄影"
)
```

### 多行提示词

需要换行时，使用 Python 三引号字符串：

```python
prompt_embeds, prompt_attention_mask = encode_on_cpu(
    """一个现代简约风格的厨房。
冰箱门上贴着一张手写便签，用中文写着：
记得买牛奶和鸡蛋！
牛奶两盒，鸡蛋一打。
柔和的自然光，写实摄影风格。"""
)
```

官方用户指南也说明，图内文字需要精确换行时，可以直接在提示词里写换行。文字渲染仍然存在不确定性，多试几个种子比反复堆叠形容词更有效。

### 负向提示词

官方 Base 指南给出了一些可用负向标签，但具体效果与模型版本和采样方式有关。本文示例采用空负向提示词加负向分支跳层，不建议未做对照就套用其他扩散模型的一大串负向词。

### 常用参数怎么调

| 目标 | 调整方式 | 代价或注意事项 |
| --- | --- | --- |
| 更快验证流程 | 512×512、20 步 | 主要用于冒烟测试，不代表最佳画质 |
| 更接近官方质量设置 | 约 100 万像素、至少 50 步 | 时间和显存压力增加 |
| 同一提示词换构图 | 更换随机种子 | 可以复用已编码的文本向量 |
| 更长、更复杂的提示词 | 提高文本长度上限 | CPU 编码更慢，显存和内存开销增加 |
| 精确安排文字与物体 | 使用官方 JSON Layout | 坐标必须与输出画布尺寸匹配 |

## 七、常见报错与注意事项

### 显存溢出发生在出图之前

如果堆栈停在文本编码阶段，问题不在分辨率或采样步数，而是编码器没有合适地放置。此时继续把 512×512 改成更小尺寸，通常解决不了根因。

先读报错发生在哪个阶段，再决定调整什么参数。

### 两张卡不会自动 pooling

选择双 T4 只说明 Notebook 获得了两个设备。把模型直接交给单卡加载流程，仍可能只使用第一张卡。必须显式安排设备，或者使用真正支持模型并行的加载方案。

本文采用的是**分阶段执行**，不是把一个大模型自动切分到两张卡上同时计算。

### 不要忽略“当前显存本来就被占了”

如果启动前第一张卡已经占用约 14GB，再运行新方案也只会继续失败。先检查设备状态；无法释放旧对象时重启会话，并只运行新的代码。

### 哪些警告可以暂时不管

我在实测中遇到过以下警告，它们没有阻止模型继续加载：

- 新版本中某个参数名称即将弃用。
- 未配置 Hugging Face token，匿名下载速率可能受限。
- 调度器配置里存在当前实现不使用的字段。
- 加载自定义模型时打印类型提示。
- Kaggle 预装的其他 Python 包存在依赖冲突。

判断依据不是文字是不是红色，而是后面有没有真正的异常堆栈，以及程序是否继续进入下一阶段。

### 不玩时停止 GPU 会话

只关闭浏览器标签页，不代表云端会话立即停止。使用完毕后，确认图片已经保存或下载，然后点击 Notebook 顶部的电源按钮，停止当前会话。

停止的是 **Kaggle 云端会话**，不是自己的电脑。代码仍保留在 Notebook 中，但内存、显存和临时缓存会释放，下次可能需要重新下载模型。

## 八、这套方案适合什么场景

这套“CPU 编码 + 单张 T4 生图”的方式，价值在于把原本无法装入单卡的流程拆开，让免费硬件也有机会运行完整模型。它不适合被理解为高速部署方案：

- CPU 编码明显慢于 GPU。
- 20 步低分辨率测试不能代表正式画质。
- Kaggle 会话和配额限制决定了它更适合学习、试验和临时验证。
- 如果要批量生成或提供稳定服务，24GB 以上的单卡环境会更简单，减少组件搬运和自定义设备管理。

对我来说，这次折腾最大的收获不是“终于白嫖到了 GPU”，而是把模型宣传中的参数量、完整推理管线和实际硬件需求区分开了。看懂这三个层次之后，很多看似玄学的显存问题其实都能提前算出来。

## 参考资料

- [OpenMOSS/MOSS-Image：代码、模型入口与快速开始](https://github.com/OpenMOSS/MOSS-Image)
- [MOSS-Image 官方用户指南：采样参数、提示词与布局格式](https://github.com/OpenMOSS/MOSS-Image/blob/main/USERGUIDE.md)
- [MOSS-Image-SFT 模型卡](https://huggingface.co/OpenMOSS-Team/MOSS-Image-SFT)
- [MOSS-Image Hugging Face 在线演示](https://huggingface.co/spaces/OpenMOSS-Team/MOSS-Image)
- [Kaggle Notebooks 文档](https://www.kaggle.com/docs/notebooks)
