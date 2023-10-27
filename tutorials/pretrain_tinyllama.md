# Pretrain TinyLlama

This tutorial will walk you through pretraining [TinyLlama](https://github.com/jzhang38/TinyLlama/).

## What's TinyLlama?

[TinyLlama](https://github.com/jzhang38/TinyLlama/) is architecturally the same as Meta AI's LLama 2, but only has 1.1B parameters and is instead trained on multiple epochs on a mix of [SlimPajama](https://huggingface.co/datasets/cerebras/SlimPajama-627B) and [Starcoder](https://huggingface.co/datasets/bigcode/starcoderdata) datasets.

Here is a quick fact sheet:

|---------------------------------|----------------------------------------------------------------|
| Parameters                      | 1.1B                                                           |
| Model Size                      | Layers: 22, Heads: 32, Query Groups: 4, Embedding Size: 2048, Intermediate Size: 5632 |
| Sequence Length                 | 2048                                                           |
| Batch Size                      | 2 million tokens (2048 * 1024)                                 |
| Learning Rate                   | 4e-4                                                           |
| Learning Rate Schedule          | Cosine with 2000 warmup steps                                  |
| Training Data                   | [SlimPajama](https://huggingface.co/datasets/cerebras/slimpajama-627b) (893 GB), [Starcoder](https://huggingface.co/datasets/bigcode/starcoderdata) (290 GB) |
| Combined Dataset Size           | Around 950B tokens                                             |
| Total Tokens During Training    | 3 trillion (slightly more than 3 epochs/1430k steps)           |

(this table was sourced from the author's [README](https://github.com/jzhang38/TinyLlama/))


## Download datasets

You can download the data using git lfs:

```bash
# Make sure you have git-lfs installed (https://git-lfs.com):
git lfs install
```

```bash
git clone https://huggingface.co/datasets/cerebras/slimpajama-627b data/slimpajama
git clone https://huggingface.co/datasets/bigcode/starcoderdata data/starcoderdata
```


## Prepare the datasets for training

In order to start pretraining lit-gpt on it, you need to read, tokenize, and write the data in binary chunks. This will leverage our `lightning.data` data optimization pipeline and streaming dataset that comes with Lightning. You will need to have the tokenizer config available:

```bash
pip install huggingface_hub sentencepiece

python scripts/download.py \
   --repo_id meta-llama/Llama-2-7b-hf \
   --access_token your_hf_token
```

Then, run the preprocessing script for each dataset and split.

**Starcoder:**
```bash
python scripts/prepare_starcoder.py \
  --source_path data/starcoderdata \
  --tokenizer_path checkpoints/Llama-2-7b-hf \
  --name starcoder
```

**SlimPajama:**
```bash
python scripts/prepare_slimpajama.py \
  --source_path data/slimpajama/validation \
  --tokenizer_path checkpoints/Llama-2-7b-hf \
  --name slimpajama/val

python scripts/prepare_slimpajama.py \
  --source_path data/slimpajama/test \
  --tokenizer_path checkpoints/Llama-2-7b-hf \
  --name slimpajama/test

python scripts/prepare_slimpajama.py \
  --source_path data/slimpajama/train \
  --tokenizer_path checkpoints/Llama-2-7b-hf \
  --name slimpajama/train
```

If you want to run on a small slice of the datasets first, pass the flag `--fast_dev_run=true` to the commands above.
In the above we are assuming that you will be using the same tokenizer as used in LlaMA/TinyLlama, but any trained [SentencePiece](https://github.com/google/sentencepiece) tokenizer with a 32000 vocabulary size will do here.


## Pretraining

Running the pretraining script with its default settings requires at least 8 A100 GPUs.

```bash
python pretrain/tinyllama.py
```

The script will save checkpoints periodically to the folder `out/`.
By default, the `pretrain/tinyllama.py` script will pretrain the Llama 2 7B model with FSDP in
`bfloat16` mixed precision and gradient accumulation.

You can easily change the size of the model by passing a different string to the model name variable

```python
model_name = "tiny-llama-1.1b"
```

at the top of this script.

The currently supported model names are contained in the [config.py](https://github.com/Lightning-AI/lit-gpt/lit_gpt/config.py) file.
You can

1) either search this file for lines containing "name =",
2) or run `python scripts/download.py` without additional command line arguments

Keep in mind that training with a single machine will take weeks. To speed up the process, you'll need access to a cluster.
Once you're in a cluster, you can follow [these instructions](https://lightning.ai/docs/fabric/stable/fundamentals/launch.html#launch-on-a-cluster)
to launch the script across machines:

- [SLURM cluster](https://lightning.ai/docs/fabric/stable/guide/multi_node/slurm.html)
- [Barebones cluster](https://lightning.ai/docs/fabric/stable/guide/multi_node/barebones.html)
- [MPI](https://lightning.ai/docs/fabric/stable/guide/multi_node/other.html)

The [script contains several configurations and hyperparameters](https://github.com/Lightning-AI/lit-gpt/blob/main/pretrain/openwebtext.py#L23-L46) you can tweak.

For instance, `micro_batch_size` should be adjusted so the process will use the available
GPU memory. For more tips to avoid out-of-memory issues, please also see the more detailed
[Dealing with out-of-memory (OOM) errors](oom.md) guide.

Last, logging is kept minimal in the script. In order to use a particular logger
please refer to <https://lightning.ai/docs/fabric/stable/api/loggers.html> or
call a logging client library like `wandb` directly.