---
title: "llm-jpのggufをつくってquantizeしてllama.cppで動かす"
tags: ["gptq", "llm-jp", "quantize"]
date: 2025-05-22T13:43:23+09:00
---


まず、<https://huggingface.co/llm-jp/llm-jp-3-13b-instruct3>
を全体cloneする。(git-lfsをいれておくように)

```
cd llama-models
git clone https://huggingface.co/llm-jp/llm-jp-3-13b-instruct3
cd ..
```

次に、
<https://github.com/llm-jp/llm-jp-tokenizer/tree/main/models/ver3.0>
からmodelファイルをダウンロードしてtokenizer.modelとして上のcloneしたところに突っ込む。

```
cd llama-models/llm-jp-3-13b-instruct3
wget https://github.com/llm-jp/llm-jp-tokenizer/raw/refs/heads/main/models/ver3.0/llm-jp-tokenizer-100k.ver3.0b1.model
mv llm-jp-tokenizer-100k.ver3.0b1.model tokenizer.model
cd ../../
```

最後に、`convert_hf_to_gguf.py`を使えばオッケー。

```
cd llama.cpp
./convert_hf_to_gguf.py ../llama-models/llm-jp-3-13b-instruct3/ --outtype f32
mv ../llama-models/llm-jp-3-13b-instruct3/llm-jp-3-13B-instruct3-F32.gguf ../llama-models/
cd ..
```

quantizeもそのままでも普通にできるが、、、ただimatrixを使ったほうがいいらしい？？あんま変わらない気もするけどね. (<https://github.com/ggerganov/llama.cpp/pull/4861> を参照のこと)

すなわち、
<https://huggingface.co/datasets/TFMC/imatrix-dataset-for-japanese-llm/blob/main/c4_en_ja_imatrix.txt>からimatrixの学習もとデータをダウンロードするまではいいが、問題は、

```
./imatrix -m <some_fp_model> -f <some_training_data> [-o <output_file>] [--verbosity <verbosity_level>]
        [-ofreq num_chunks] [-ow <0 or 1>] [other GPT params]
```

データをダウンロードするだけではんだめで、imatrixを学習させなければいけないらしい。  
ただ、F32のモデルウェイトデータをメモリにロードするのは128ギガでも足りないかもしれない。。  
とりあえずやってみるか。。

```
cd llama.cpp
./build/bin/llama-imatrix -m ../llama-models/llm-jp-3-13B-instruct3-F32.gguf -f ../llama-models/c4_en_ja_imatrix.txt
cd ..
```

なんかメモリ48gbでも普通にいけた！

```
compute_imatrix: tokenizing the input ..
compute_imatrix: tokenization took 252.352 ms
compute_imatrix: computing over 185 chunks with batch_size 512
compute_imatrix: 41.34 seconds per pass - ETA 2 hours 7.45 minutes
（省略）
llama_perf_context_print:        load time =   81076.58 ms
llama_perf_context_print: prompt eval time = 5421911.91 ms / 94720 tokens (   57.24 ms per token,    17.47 tokens per second)
llama_perf_context_print:        eval time =       0.00 ms /     1 runs   (    0.00 ms per token,      inf tokens per second)
llama_perf_context_print:       total time = 5540153.22 ms / 94721 tokens
```

8スレッドで動かしてるのに、cpuもメモリ使用率も10%程度しかない。。メモリもCPUもほぼ使ってなさそうに見える、、、一応スレッドは8/16ってなってるのに。。なぜ？cuda使ってるからか？cudaは使えないとか書いてたのだが。。アプデでcuda使えるようになったのかなー。まーでもやっぱり、メモリがボトルネックになってcpu使用率が低いままなんかな。あるいはストレージのread速度がボトルネックかもしれない？あるいはgpuが代わりに計算してくれてるのか。

で最終的には、

```
mv ./imatrix.dat ../llama-models/llm-jp-3-13B-instruct3-F32.imatrix.dat
./build/bin/llama-quantize --imatrix ../llama-models/llm-jp-3-13B-instruct3-F32.imatrix.dat ../llama-m
odels/llm-jp-3-13B-instruct3-F32.gguf ../llama-models/llm-jp-3-13B-instruct3-Q4_K_M.gguf Q4_K_M
```

みたいな感じにする。

```
./build/bin/llama-cli -m ../llama-models/llm-jp-3-13B-instruct3-Q4_K_M.gguf -ngl 30 -no-cnv -p あああ
```

evalにはno-cnvをつけないとダメ。
