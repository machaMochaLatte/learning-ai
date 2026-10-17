---
title: "Multimodal AI: How Text, Images, and Audio Enter One Model"
description: "AI Foundations #20 builds a practical mental model of multimodal AI: how text, image, audio, and video representations are converted, aligned, and fed into transformer-style systems."
pubDate: 2026-09-24
updatedDate: 2026-09-24
category: "AI Fundamentals"
tags: ["AI fundamentals", "multimodal AI", "vision language models", "audio", "transformers", "embeddings"]
draft: false
featured: false
author: "machamochalatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/2103.00020
  - https://arxiv.org/abs/2204.14198
  - https://arxiv.org/abs/2304.08485
  - https://arxiv.org/abs/2212.04356
evidenceStatus: "explainer"
evidenceNote: "Cumulative AI Foundations lesson on multimodal representation and transformer integration. Architecture examples are conceptual and grounded in the cited primary papers."
---

Yesterday's lesson ended with Mixture of Experts, where one model can contain many expert networks but activate only a few for each token.

Today I want to change the question.

Instead of asking **which expert handles this token?**, ask:

> What if the input is not a text token at all?

A photo is pixels.

Audio is a waveform.

Video is a sequence of images plus time.

Text is discrete tokens.

Yet modern AI systems can accept several of these together and answer one question about all of them.

That is **multimodal AI**.

The key idea is easier than it first sounds:

> Different modalities start in different forms, but the model converts them into numerical representations that can interact inside a shared learned system.

## First: “multimodal” does not mean raw pixels become words

When I first heard that an LLM could “see an image,” I pictured the model somehow reading millions of RGB numbers as if they were text tokens.

That mental model is usually wrong.

A typical multimodal pipeline has stages:

```text
raw input
-> modality-specific preprocessing or encoder
-> vectors / feature tokens
-> projection or alignment
-> transformer-style model
-> output tokens
```

Text, image, and audio can enter through different front doors.

They meet only after each has been transformed into a representation the rest of the network can use.

That is similar to converting several measurements into the same unit before doing algebra with them.

You do not add “3 meters” directly to “5 seconds.”

You first decide what representation makes the comparison meaningful.

## Text already has a path we know

From earlier Foundations lessons, the text path is familiar:

```text
text
-> tokenizer
-> token IDs
-> embedding lookup
-> vectors
-> transformer
```

If the model has hidden size `d`, each text token eventually becomes a vector with `d` components:

```text
x_text in R^d
```

The transformer does not see the English word itself. It operates on vectors.

That is the bridge to multimodal systems.

If image or audio information can also become useful vectors, then those vectors can be combined with text representations in learned ways.

## An image can become a sequence too

An image begins as a grid:

```text
height x width x channels
```

For an RGB image, the channel count is usually 3.

A common vision-transformer idea is to divide the image into patches.

Imagine a 224 x 224 image cut into 16 x 16 patches.

Along one side:

```text
224 / 16 = 14 patches
```

So the image contains:

```text
14 x 14 = 196 patches
```

Each patch can be converted into a vector.

Now the two-dimensional image has become something that looks much more like a token sequence:

```text
patch_1, patch_2, ..., patch_196
```

These are not word tokens. They are **visual feature tokens**.

That is the first big mental jump:

> A transformer does not require every sequence element to have started life as a word.

It needs numerical vectors with the right dimensions and learned meaning.

## The dimensions have to match somewhere

Suppose a vision encoder produces vectors of size `d_v`, while the language model expects hidden vectors of size `d_l`.

If:

```text
d_v != d_l
```

we cannot simply drop the vision vectors into the language model unchanged.

A learned projector can map them:

```text
y = W x + b
```

where:

```text
x in R^(d_v)
W in R^(d_l x d_v)
y in R^(d_l)
```

This is the same linear-layer math we met near the beginning of the course.

The matrix `W` learns how to translate the vision encoder's feature space into a representation useful to the language model.

LLaVA is a famous example of this general idea: connect a pretrained vision encoder to a language model with a learned projection, then train the system so visual features can participate in language generation.

I like this because it makes multimodal AI feel less magical.

We are still using weights, tensors, forward passes, loss, gradient descent, and backpropagation.

The new problem is mostly **representation and alignment**.

## But one giant sequence is not the only architecture

There are several ways to connect modalities.

A simplified design can place projected visual tokens near text tokens and let a transformer process the combined sequence.

Another design keeps a separate vision encoder and lets the language model attend to visual features through cross-attention.

Flamingo is an important example of the second family: visual features are produced separately, then language layers gain mechanisms for attending to those features.

So “multimodal transformer” is not one exact wiring diagram.

It is a family of systems that solve a common interface problem:

```text
How do representations from different modalities exchange information?
```

## CLIP teaches another useful idea: alignment

CLIP does something slightly different from a chatty vision-language model.

It learns image and text representations that can be compared in a shared embedding space.

Very roughly:

```text
image -> image vector
text  -> text vector
```

Training encourages matching image-text pairs to have representations that line up better than mismatched pairs.

One common similarity measure is cosine similarity:

```text
cos(theta) = (a · b) / (||a|| ||b||)
```

If two normalized vectors point in similar directions, their cosine similarity is high.

This gives us a second multimodal pattern:

> Sometimes the goal is not to turn every modality into one long sequence. Sometimes the goal is to learn spaces where different modalities become comparable.

That idea shows up in retrieval, classification, search, and larger multimodal systems.

## Audio starts differently again

Raw audio is a waveform:

```text
amplitude over time
```

A model usually does not treat each raw sample like a language token.

Speech systems often transform the waveform into a time-frequency representation such as a log-Mel spectrogram, then encode those features.

Whisper is a useful example. Its encoder consumes audio features and its decoder generates text tokens.

So the rough path is:

```text
waveform
-> audio features
-> audio encoder
-> hidden representation
-> text decoder
```

The important lesson is not that every multimodal model uses Whisper's exact architecture.

It is that **audio needs its own representation pipeline before language-style generation can use it**.

## Video adds the time problem

An image has spatial structure.

Video has spatial structure **and time**.

A naive approach would treat every frame as a full image and create visual tokens for all frames.

That becomes expensive quickly.

Suppose one frame produces 196 visual tokens and we keep 100 frames:

```text
196 x 100 = 19,600 visual tokens
```

And that is before the user's text prompt or generated answer.

So video models need strategies such as:

```text
sample fewer frames
compress visual features
pool neighboring features
use specialized temporal encoders
select relevant segments
```

This connects directly to our earlier Context and KV Cache lesson.

Multimodal input can consume context and memory very quickly because an image, audio clip, or video may expand into many internal feature positions.

## “One image” is not necessarily one token

This point matters a lot when people estimate context length.

The UI may show:

```text
[one attached image]
```

But internally that image can become dozens, hundreds, or more visual feature tokens depending on the architecture, resolution, tiling strategy, and compression method.

So these two statements can both be true:

```text
user attached one image
model processed many visual vectors
```

That affects:

```text
prefill compute
attention cost
KV-cache usage
latency
maximum remaining text context
```

This is one reason multimodal inference can feel much heavier than plain-text inference even when the language model itself looks familiar.

## How does the model learn which image feature matches which words?

It needs training data and a loss function.

The exact objective varies by architecture, but the general process is still the training loop we already learned:

```text
multimodal example
-> forward pass
-> prediction
-> loss
-> backpropagation
-> parameter update
```

For image-text systems, training data may contain paired images and captions, question-answer examples, conversations grounded in images, or other aligned supervision.

For speech, examples can connect audio to transcripts or other targets.

The training signal teaches the model which cross-modal relationships are useful.

For example, if an image contains a red bicycle and the target answer mentions “red bicycle,” gradient descent can adjust the connector, encoder, language model, or some subset of them so the relevant visual features increasingly influence the right output tokens.

## A tiny mathematical picture

Here is a deliberately simplified multimodal model.

Text produces vectors:

```text
T = [t1, t2, ..., tn]
```

An image encoder produces visual vectors:

```text
V = [v1, v2, ..., vm]
```

A projector maps visual dimension into language hidden size:

```text
V' = V W
```

Then the model creates some joint representation, conceptually:

```text
X = [V'; T]
```

and runs transformer layers over `X`.

Real systems may add special tokens, position rules, resamplers, cross-attention, separate encoders, gating, or many other details.

But this toy equation captures the core interface problem:

> Convert modality-specific information into vectors that the shared model can combine meaningfully.

## Why a language model can answer about an image

Once visual information has been projected or attended into the model's working representation, the next-token prediction loop looks familiar again.

For a prompt like:

```text
[image]
What animal is sitting on the sofa?
```

we can think of the process as:

```text
visual features + text features
-> transformer computation
-> logits for next text token
-> sampling
-> next token
-> repeat
```

The output may still be ordinary text tokens.

Multimodality changes the **conditioning information**, not necessarily the output mechanism.

That is why many vision-language models still end in the same autoregressive decode loop we studied in Inference and Sampling.

## Multimodal generation can run the other direction too

So far we mostly discussed:

```text
image/audio + text -> text
```

But multimodal systems can also produce images, audio, or video.

Those outputs often require a different decoder or generative mechanism. Image generation, for example, may use diffusion or another visual-token generator instead of the language model's ordinary vocabulary softmax.

That is a separate implementation problem from understanding an image.

This lesson focuses on the shared Foundations question: how different input modalities become representations a model can reason over together.

## What can go wrong?

A multimodal model has more interfaces than a text-only model, so it has more places for information to be lost.

### The encoder can miss information

Small text, unusual visual details, weak audio, or rare patterns may not be represented strongly enough.

### The projector can be a bottleneck

If a rich visual representation is compressed too aggressively before reaching the language model, useful detail can disappear.

### The language model can ignore the modality

A model may lean on linguistic priors and answer what is likely rather than what is actually visible or audible.

### Context can become crowded

High-resolution images or long audio/video inputs can consume substantial representation capacity and compute.

### Alignment data can be imperfect

If training examples do not teach the right visual-language or audio-language relationships, the model can produce fluent but poorly grounded answers.

So multimodal capability should not be confused with perfect perception.

The model still has finite resolution, finite context, finite training coverage, and learned biases.

## The connection to every earlier Foundations lesson

Multimodal AI sounds like a new subject, but it is actually a reunion of almost everything we already learned.

**Parameters, weights, and biases:** encoders and projectors are learned networks.

**Tensors:** images, spectrograms, embeddings, and hidden states are tensors.

**Forward pass:** all modalities must move through computation to produce an output.

**Loss, gradient descent, backprop:** multimodal alignment is learned through optimization.

**Embeddings:** modalities become vector representations.

**Tokenization:** text uses text tokens; images and audio use their own chunking or feature schemes.

**Attention and transformers:** representations interact through self-attention, cross-attention, or related mechanisms.

**Pretraining and fine-tuning:** multimodal capability can be built in stages from pretrained components and multimodal instruction data.

**Inference and sampling:** the final text answer is still commonly generated one token at a time.

**Context and KV cache:** visual/audio feature positions consume runtime memory and compute.

**Quantization:** multimodal models can also be quantized, although vision/audio components may have different sensitivity than the language backbone.

**MoE:** a multimodal model can itself use mixture-of-experts layers.

This is why the curriculum is cumulative. We did not leave the earlier topics behind. We keep reusing them.

## My mental model now

I no longer imagine a multimodal model as “an LLM that magically learned eyes and ears.”

I picture a set of translators around a shared reasoning engine:

```text
text  -> text representation  --\
image -> visual representation ---> shared learned computation -> output
audio -> audio representation --/
```

The arrows are learned.

The representations are tensors.

The model succeeds when training makes those representations carry compatible information.

That is the core idea.

And it gives us the final question for the next lesson.

A model can now receive rich information from many modalities. But when it produces a long solution, checks intermediate steps, learns from preferences, or improves behavior through reinforcement-style training, what exactly is happening?

Next: **AI Foundations #21 — Reasoning and Reinforcement Learning: What Changes Beyond Next-Token Pretraining**.
