---
title: "Quantization: How Fewer Bits Make Large Models Smaller"
description: "AI Foundations #18 builds quantization from number representation and rounding error, then connects 4-bit weights to model size, speed, calibration, and quality tradeoffs."
pubDate: 2026-09-22
updatedDate: 2026-09-22
category: "AI Fundamentals"
tags: ["AI fundamentals", "quantization", "GGUF", "model weights", "low precision", "local AI"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/2208.07339
  - https://arxiv.org/abs/2210.17323
  - https://github.com/ggml-org/llama.cpp
  - https://github.com/ggml-org/ggml/blob/master/docs/gguf.md
evidenceStatus: "explainer"
evidenceNote: "A cumulative conceptual lesson on low-bit model quantization, grounded in established post-training quantization research and the GGML/llama.cpp implementation ecosystem."
---

Last lesson ended with a memory problem.

A language model needs to keep its trained weights somewhere, and inference also needs memory for things such as the KV cache.

If the weights alone are huge, everything else becomes harder.

So the next question is almost embarrassingly simple:

> Do we really need to store every learned number with so many bits?

That question leads to **quantization**.

I used to hear phrases like `Q4_K_M`, `INT8`, `4-bit`, and `quantized GGUF` and mentally translate all of them into:

```text
smaller model
```

That is directionally correct, but it skips the interesting part.

Quantization is really a controlled approximation problem:

```text
many precise numbers
-> represent them with fewer possible values
-> accept some rounding error
-> try to preserve the model's useful behavior
```

The math is not difficult once I stop treating "4-bit" as a magic compression format and instead ask what happens to one ordinary weight.

## Start with a single learned weight

Earlier in AI Foundations, we learned that neural networks contain parameters such as weights.

A weight might be a number like:

```text
0.137428
```

or:

```text
-0.823916
```

During inference, enormous matrices full of values like these participate in operations such as matrix multiplication.

The model does not need the decimal text `0.137428`. It stores a binary numerical representation.

A common training representation is a floating-point format such as FP32, BF16, or FP16.

Very roughly:

```text
FP32  -> 32 bits per value
BF16  -> 16 bits per value
FP16  -> 16 bits per value
```

Different floating-point formats divide those bits differently among sign, exponent, and precision, so BF16 and FP16 are not interchangeable just because both use 16 bits.

But for today's memory intuition, the important fact is simpler:

```text
more bits per weight -> more bytes for the weight matrix
fewer bits per weight -> fewer bytes
```

## The first size calculation

Suppose a toy model has exactly one billion parameters.

Ignoring metadata and all other runtime memory for a moment:

```text
1 billion parameters x 4 bytes for FP32
= 4 billion bytes
≈ 4 GB decimal
```

At 16 bits, each parameter is 2 bytes:

```text
1 billion x 2 bytes
= 2 GB
```

At an idealized 8 bits:

```text
1 billion x 1 byte
= 1 GB
```

At an idealized 4 bits:

```text
1 billion x 0.5 byte
= 0.5 GB
```

For a 7-billion-parameter model, the same rough arithmetic gives:

| Representation | Ideal raw weight storage |
| --- | ---: |
| FP32 | 28 GB |
| 16-bit | 14 GB |
| 8-bit | 7 GB |
| 4-bit | 3.5 GB |

This table is a **lower-level storage intuition**, not a promise that every real 7B model file has exactly those sizes.

Real quantized files also need scales, metadata, alignment, and sometimes higher-precision tensors. Some architectures share parameters or contain tensors that are treated differently. Runtime memory also includes much more than weights.

Still, this explains why 4-bit quantization matters so much for local AI. A model that is inconvenient at 16 bits can become practical on consumer hardware when most large weight tensors use around four bits per value.

## But four bits cannot represent arbitrary real numbers

This is where the approximation appears.

Four bits have only:

```text
2^4 = 16
```

possible bit patterns.

That means a 4-bit representation cannot directly encode every possible real-valued weight.

If my original weights include:

```text
-0.823916
-0.371204
 0.058811
 0.438527
 0.907102
```

I need to map them onto a much smaller set of representable levels.

This is like reducing a high-resolution ruler to a ruler with only a few tick marks.

The question becomes:

> Which tick mark should each original value use?

## A simple symmetric quantizer

For intuition, imagine that I want to represent a small group of weights with signed integer levels from:

```text
-7 to +7
```

That is 15 symmetric levels around zero. Real 4-bit schemes may use their code space differently, but this makes the arithmetic easy to see.

Take these weights:

```text
-0.82
-0.37
 0.06
 0.44
 0.91
```

The largest absolute value is:

```text
0.91
```

If integer level `7` should correspond to `0.91`, a simple scale is:

```text
scale = 0.91 / 7
      = 0.13
```

Now convert each floating-point weight into an integer by dividing by the scale and rounding:

```text
q = round(weight / scale)
```

For the first weight:

```text
q = round(-0.82 / 0.13)
  = round(-6.31)
  = -6
```

For `0.44`:

```text
q = round(0.44 / 0.13)
  = round(3.38)
  = 3
```

Our five weights become approximately:

```text
original      quantized integer
-0.82         -6
-0.37         -3
 0.06          0
 0.44          3
 0.91          7
```

We have replaced several precise floating-point numbers with small integers.

## Dequantization reconstructs approximate values

When the runtime needs an approximate real value, it can multiply the integer by the stored scale:

```text
approx_weight = q x scale
```

So:

```text
-6 x 0.13 = -0.78
-3 x 0.13 = -0.39
 0 x 0.13 =  0.00
 3 x 0.13 =  0.39
 7 x 0.13 =  0.91
```

Compare them:

| Original | Reconstructed | Error |
| ---: | ---: | ---: |
| -0.82 | -0.78 | +0.04 |
| -0.37 | -0.39 | -0.02 |
| 0.06 | 0.00 | -0.06 |
| 0.44 | 0.39 | -0.05 |
| 0.91 | 0.91 | 0.00 |

That table is quantization in miniature.

We saved bits by accepting error.

The hard engineering problem is choosing a representation where that error is small *in the ways that matter to the model*.

## Quantization error is not all equally important

Suppose two weights change by the same numerical amount:

```text
weight A error = 0.03
weight B error = 0.03
```

It does not follow that they have the same effect on the network.

One weight might barely affect typical activations. Another might sit in a direction that strongly changes an important output.

So the naive goal:

```text
minimize average absolute error in every stored number
```

is useful but incomplete.

More sophisticated quantization methods care about things such as:

```text
weight distribution
activation statistics
outliers
matrix structure
which tensors are sensitive
how errors propagate through later layers
```

This is why two files that both say "4-bit" can behave differently.

The bit width is only part of the recipe.

## One scale for the entire model would be terrible

Imagine a model whose weights range from:

```text
-12.0 to +12.0
```

but most weights actually live near:

```text
-0.2 to +0.2
```

If one global 4-bit scale must cover the full `-12` to `+12` range, the spacing between representable values becomes very coarse.

Small weights near zero could collapse together.

A better idea is to quantize smaller groups separately.

For example:

```text
block 1 -> its own scale
block 2 -> its own scale
block 3 -> its own scale
...
```

Then each local scale can match the values in its own block more closely.

This costs extra metadata because every block needs scale information, but it usually reduces error dramatically.

That is one reason a real "4-bit" model is usually larger than exactly four bits times the parameter count.

The representation includes supporting information.

## Block size creates a tradeoff

Suppose I use one scale for every 256 weights.

I need relatively few scales, so overhead is small.

But one unusual outlier can stretch the range for many nearby values.

If I use one scale for every 16 weights, each scale can fit its local values more precisely.

But now I need many more scale values.

So block size creates a tradeoff:

```text
larger blocks
-> lower metadata overhead
-> less local flexibility

smaller blocks
-> more metadata
-> better local adaptation
```

Real quantization formats make different design choices here.

## What is a zero-point?

My first example was symmetric around zero.

Sometimes the useful value range is not symmetric.

Imagine values mostly between:

```text
2.0 and 5.0
```

If I force the quantizer to waste half its levels representing negative values that never occur, I may be throwing away precision.

An **asymmetric quantizer** can use both a scale and a zero-point.

A common conceptual form is:

```text
q = round(x / scale) + zero_point
```

and reconstruction is:

```text
x_approx = scale x (q - zero_point)
```

The zero-point shifts the integer grid so that its useful range lines up better with the data.

Not every LLM quantization scheme uses the same kind of zero-point, but the general lesson matters:

```text
bit width tells me how many codes exist
scale/offset rules tell me what those codes mean
```

## Why outliers are difficult

Consider one block:

```text
0.04
-0.07
0.09
0.02
-0.05
3.80
```

Five values are near zero. One is enormous by comparison.

If a simple symmetric quantizer chooses its scale from the largest magnitude, the value `3.80` controls the whole grid.

Then the spacing may be so wide that several small values round to the same code.

This is the **outlier problem**.

A quantization method can respond in different ways:

```text
use smaller groups
keep sensitive values/tensors at higher precision
use a different codebook
use activation-aware calibration
rearrange or rescale the computation
```

This is why low-bit quantization research is not finished by saying "round everything to INT4."

## Weight-only quantization is the easiest starting point

There are several different things we could quantize during inference:

```text
weights
activations
KV cache
```

These are not the same problem.

**Weight-only quantization** stores model weights at low precision while allowing activations or intermediate computation to use another representation.

Conceptually:

```text
compressed low-bit weights
-> runtime reconstructs/uses them in kernels
-> activations may remain FP16/BF16/etc.
```

This is common for local inference because weights dominate static model storage and memory bandwidth for many workloads.

The model becomes much smaller without requiring every intermediate value to live in four bits.

## Activation quantization is harder

Weights are fixed after training, so we can inspect their distribution once.

Activations depend on the input.

The same layer may see different activation ranges for different tokens, prompts, or batches.

That makes activation quantization more dynamic.

If a runtime uses something like weight-and-activation 8-bit or 4-bit execution, it needs a strategy for keeping changing intermediate values inside a useful numerical range.

That can be extremely valuable for hardware that has fast low-precision matrix units, but it is a different problem from simply compressing a GGUF weight file.

## KV-cache quantization is a third problem

Yesterday we learned that the KV cache grows with context and concurrency.

So even after shrinking model weights, long-context inference can still consume huge amounts of memory.

That motivates lower-precision KV storage.

Conceptually:

```text
FP16 K/V
-> 16 bits per stored element

8-bit K/V
-> about half the raw element storage

4-bit K/V
-> about one quarter of the raw element storage
```

Again, real memory savings depend on metadata and implementation.

And the error matters differently because the cache stores intermediate attention state generated from the current sequence, not fixed trained weights.

So:

```text
weight quantization
!= activation quantization
!= KV-cache quantization
```

They share the idea of fewer bits, but they affect different numerical objects.

## Why smaller weights can make inference faster

Quantization is often described as a memory-saving technique, but it can also improve speed.

One reason is memory bandwidth.

Suppose a decode step needs to stream a huge amount of model weight data from memory.

If the same useful weights require roughly half as many bytes, less data must move.

A simplified picture is:

```text
16-bit weights:
move lots of bytes -> compute

4-bit weights:
move fewer bytes -> unpack/dequantize -> compute
```

The low-bit path adds work to interpret the compressed representation, but it may save much more time by reducing memory traffic or by using specialized low-precision kernels.

Whether it is actually faster depends on:

```text
hardware
kernel implementation
model architecture
batch size
quant format
memory placement
which operation is the bottleneck
```

So this statement is too strong:

```text
"4-bit is always 4x faster than 16-bit."
```

Bit width and speed are not related that simply.

A smaller file can even be slower if the runtime has a poor kernel for that format.

## The quality question is also not monotonic

I originally imagined a simple ladder:

```text
16-bit = best
8-bit = slightly worse
6-bit = worse
5-bit = worse
4-bit = worse
3-bit = much worse
```

As a broad trend, reducing precision eventually increases distortion.

But small benchmark results do not have to follow a perfectly monotonic staircase.

Quantization perturbs many decision boundaries. A tiny perturbation can accidentally fix one answer and break another.

Suppose BF16 gives two logits:

```text
correct token = 7.001
wrong token   = 7.002
```

It narrowly chooses the wrong token.

A quantized model might perturb them to:

```text
correct token = 6.998
wrong token   = 6.997
```

Now the quantized model gets that one example right.

That does **not** prove the quantized model is globally better than BF16.

It proves the perturbation moved this particular decision boundary in a helpful direction.

The opposite can happen just as easily.

This is why good quantization evaluation needs more than one accuracy number.

## Perplexity, task accuracy, and fidelity ask different questions

A quantized model can be compared in several ways.

### Language-model loss or perplexity

This asks roughly:

```text
How well does the model assign probability to held-out text?
```

It gives a broad distribution-level signal.

### Task accuracy

This asks:

```text
Did the model solve these questions correctly?
```

It is easy to understand but depends heavily on the task set.

### Behavioral fidelity to the original model

This asks:

```text
Does the quantized model make the same decisions as the higher-precision reference?
```

This is not identical to accuracy.

A quant can disagree with BF16 and become correct on one item, or disagree and become wrong on another.

A separate RAMGPT QuantBench article recently measured these distinctions on public MiMo-V2.6 GGUFs. That article is an experiment; this Foundations lesson is only building the machinery needed to understand why those metrics can disagree: [/articles/mimo-v26-public-gguf-quantbench/](/articles/mimo-v26-public-gguf-quantbench/).

## Calibration tells the quantizer what matters

Some quantization methods can convert weights using only the weight values themselves.

Other methods use a **calibration dataset**.

The idea is:

```text
run representative data through the model
-> observe how tensors/activations are used
-> estimate which quantization errors matter more
-> allocate precision or choose scales accordingly
```

If one group of weights is extremely influential for typical activations, preserving it more carefully may be worth more than minimizing error somewhere that rarely matters.

Calibration therefore introduces another variable:

```text
same BF16 source
+ same nominal bit width
+ different calibration data
= potentially different quantized model
```

This helps explain why a quant's name alone cannot tell me everything about its quality.

## An importance matrix is one calibration strategy

In the llama.cpp ecosystem, I often see the term **iMatrix**, short for importance matrix.

The basic intuition is that representative data is processed so the quantization pipeline can estimate which weights or tensor directions are more important under observed activations.

Then low-bit quantization can make more informed choices than treating every value as equally disposable.

I do not need the exact implementation formula to keep the useful mental model:

```text
no calibration:
quantize based mostly on stored weights / format rules

importance-aware calibration:
use representative model activity to guide some quantization choices
```

That does not guarantee that a larger calibration set is always better. Data distribution, coverage, quantization policy, and sensitive tensors all matter.

## Why some tensors stay at higher precision

A file advertised as "4-bit" does not necessarily mean every single tensor is stored with exactly four bits per element.

A quantization recipe may decide:

```text
large matrix A -> 4-bit
large matrix B -> 4-bit
sensitive matrix C -> 5-bit or 6-bit
small norm tensor -> FP32
output tensor -> higher precision
```

This can be a good trade.

A small tensor might contribute almost nothing to file size, so keeping it in FP32 costs little.

A particularly sensitive matrix might deserve more bits because its error has a large effect on behavior.

Therefore:

```text
"Q4 model"
```

usually means something closer to:

```text
model whose dominant weight representation is around a 4-bit quantization recipe
```

not:

```text
every tensor is exactly four raw bits with no metadata
```

## Why GGUF files can use mixed tensor types

GGUF is a container format used heavily by llama.cpp and related tools.

It stores model metadata plus tensors, and different tensors can have different GGML types.

That makes mixed-precision recipes practical.

Conceptually, a GGUF can say:

```text
this tensor -> F32
this tensor -> Q6_K
this tensor -> Q4_K
this tensor -> another supported type
```

The runtime reads the tensor metadata and dispatches compatible kernels or conversion paths.

So GGUF is not itself "the quantization algorithm."

It is a format capable of packaging tensors whose types may include quantized representations.

That distinction helped me a lot:

```text
GGUF = container/model file format
quant type = numerical representation for tensors inside it
quantization recipe = decisions used to create those tensors
```

## Why two Q4 files can differ

Now I can list several reasons two files with similar names may not be equivalent:

```text
1. different source model revision
2. different converter/runtime revision
3. different calibration data
4. different block quantization format
5. different tensor-specific precision choices
6. different handling of outliers
7. different metadata or architecture conversion
8. one file may even be malformed
```

So comparing two `Q4_K_M`-looking artifacts should begin with verifying that they came from the same source and actually load correctly.

The nominal quant label is not a cryptographic identity.

## Quantization does not retrain the model in the ordinary sense

Post-training quantization usually begins after the expensive training process is complete.

A simplified pipeline is:

```text
trained high-precision model
-> analyze/convert weights
-> low-bit representation
-> quantized inference artifact
```

The model is not going back through full pretraining from scratch.

Some advanced methods include optimization, calibration, or quantization-aware training, but basic post-training quantization is fundamentally a conversion step applied to an already trained model.

That connects this lesson to our curriculum:

```text
pretraining
-> fine-tuning
-> trained weights
-> inference
-> quantization can make those weights cheaper to store/use
```

## Quantization-aware training is different

There is another approach called **quantization-aware training** or QAT.

Instead of waiting until the model is fully trained and then surprising it with rounding error, training simulates or incorporates low-precision effects so the parameters can adapt.

The intuition is:

```text
post-training quantization:
train first -> quantize later

quantization-aware training:
train while accounting for the quantized representation
```

QAT can recover quality in difficult low-bit settings, but it requires a training process rather than a simple one-shot file conversion.

For local model downloads, many GGUFs people encounter are post-training quants, though QAT-origin checkpoints also exist.

## A useful error equation

For each original value `w`, define the reconstructed quantized value as:

```text
w_hat
```

Then quantization error is simply:

```text
error = w_hat - w
```

For a whole tensor, we could measure something like mean squared error:

```text
MSE = average((w_hat - w)^2)
```

That is mathematically clean.

But the model does not care directly about tensor MSE. It cares about the downstream computations created by those tensors.

Two quantizers can have similar raw reconstruction error but different model behavior because the error lands in different directions or tensors.

That is the bridge between simple rounding math and modern quantization research.

## My compact mental model

I now think about quantization as four layers of decisions.

### Layer 1: How many codes do I have?

```text
bits -> number of representable codes
```

Four bits gives 16 bit patterns.

### Layer 2: What real values do those codes represent?

```text
scale
zero-point or codebook
block/group structure
```

### Layer 3: Where do I spend precision?

```text
which tensors
which blocks
which outliers
which values get higher precision
```

### Layer 4: Did the model still behave acceptably?

```text
perplexity
accuracy
fidelity
speed
memory
```

The first three define the approximation. The fourth tells us whether the approximation was useful.

## What quantization buys us

If it works well, quantization can reduce:

```text
model file size
weight memory footprint
memory bandwidth demand
hardware requirements
```

and sometimes improve:

```text
load practicality
decode throughput
number of models that fit on one device
```

The cost is numerical approximation and additional implementation complexity.

The lower we push precision, the harder it becomes to preserve behavior.

So I no longer ask:

```text
"Is quantization good or bad?"
```

I ask:

```text
What representation?
What calibration?
What tensor policy?
What hardware/kernel?
What quality metric?
What memory target?
```

Those questions turn "4-bit" from a marketing label into an engineering choice.

## Connecting back to the KV cache

The last two lessons now fit together.

Yesterday:

```text
longer context
-> more KV-cache state
-> more memory
```

Today:

```text
more bits per stored number
-> more memory per weight/cache element
```

So inference memory is not controlled by one knob.

It depends on at least:

```text
model parameter count
weight precision
context length
KV-cache precision
concurrency
runtime buffers
```

That is why a model can fit after weight quantization but still run out of memory at very long context.

Quantizing weights solves one large part of the budget. It does not make the rest disappear.

## What comes next

Quantization changes **how precisely we store the model's numbers**.

The next topic changes something more structural.

Most Transformer models we have discussed so far use a dense feed-forward network: every token passes through the same large MLP weights in each layer.

But what if a model has many different expert MLPs and activates only a few of them for each token?

Then the model can contain far more total parameters than it uses for one token.

That is the idea behind **Mixture of Experts**.

Next: **AI Foundations #19 — Mixture of Experts: Why a Model Can Have More Parameters Than It Uses Per Token.**
