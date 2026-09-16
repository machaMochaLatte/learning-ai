---
title: "Transformer: How Attention Becomes a Complete Neural Network Block"
description: "AI Foundations #12 assembles attention, residual connections, normalization, and MLP layers into the repeating Transformer block behind modern language models."
pubDate: 2026-09-16
updatedDate: 2026-09-16
category: "AI Fundamentals"
tags: ["AI fundamentals", "transformer", "attention", "MLP", "residual connections", "normalization"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://arxiv.org/abs/1706.03762
  - https://pytorch.org/docs/stable/generated/torch.nn.Transformer.html
evidenceStatus: "explainer"
evidenceNote: "A cumulative conceptual lesson based on the Transformer architecture and standard decoder-block components; implementation details vary across model families."
---

Yesterday William explained attention with a question I like:

```text
For this token, which other tokens contain useful information?
```

Queries and keys produce attention weights. Values provide the information that gets mixed into a token's representation.

That gave us:

```text
tokens
-> embeddings
-> attention
-> context-dependent vectors
```

When I first learned this, I thought I had basically learned the Transformer.

I had not.

**Attention is one operation inside a Transformer block.** The useful machine comes from combining attention with several other operations, then repeating that block many times.

## A block is easier to understand as two big jobs

For a decoder-only language model, a simplified Transformer block can be pictured as:

```text
input
  |
  +-> attention
  |     |
  |  residual
  |
  +-> MLP / feed-forward network
        |
     residual
        |
      output
```

Normalization is also inserted around these operations. Exact ordering differs among architectures, so this is a mental model rather than a claim that every modern LLM uses one identical block.

The two big computational jobs are:

1. **Attention:** move information between token positions.
2. **MLP:** transform the information inside each token position.

I find that distinction much easier to remember than a diagram full of arrows.

## Attention lets positions communicate

Take a sentence:

```text
The dog chased the ball because it was moving.
```

The representation at `it` may need information from earlier positions to become useful.

Attention allows a token position to combine information from other allowed positions.

After attention, each token has a more context-aware vector.

But then comes an important question:

> Once a token has collected information, how does the network actually transform it?

That is where the feed-forward network, usually called the **MLP**, becomes important.

## The MLP works on each position

A standard Transformer block contains a feed-forward network applied to each token position.

A simplified version looks like:

```text
x -> linear layer -> nonlinearity -> linear layer -> output
```

Modern LLMs often use gated variants such as SwiGLU rather than the exact feed-forward network in the original 2017 Transformer, but the conceptual role remains useful.

Attention mixes information **across positions**.

The MLP performs learned nonlinear computation **within each position**.

So I remember the pair like this:

```text
attention = communicate
MLP       = compute
```

That is simplified, but it gives me a place to start.

## Why not replace the original vector completely?

Suppose a layer computes some transformation `F(x)`.

Instead of making the next representation only:

```text
F(x)
```

Transformers use **residual connections**, so conceptually we get:

```text
x + F(x)
```

The original representation has a direct path forward while the layer adds a learned change.

This idea is older than Transformers, but it is crucial to them.

One way I visualize it is not "rewrite everything," but:

```text
keep what I already know
+ add what this sublayer learned
```

That is not a literal description of what every vector dimension means. It is a useful picture of the computation.

## Residual connections also help deep networks train

A Transformer is not one block.

It can contain dozens or even hundreds of repeated layers depending on the model.

Deep networks are difficult to optimize if information and gradients must pass only through a long chain of transformations. Residual paths provide direct additive routes through the network.

This connects back to our earlier lessons on backpropagation.

During training, gradients must travel backward through all these layers to update their parameters. Residual architecture helps make very deep stacks trainable.

So the residual connection is not decorative wiring. It is part of what makes the depth practical.

## What normalization is doing here

Transformer blocks also use normalization.

The original Transformer used LayerNorm. Many modern language models use RMSNorm and can place normalization differently from the original architecture.

I do not need all those variants yet.

The important idea is that normalization helps keep the scale of activations controlled as representations pass through repeated transformations.

You may see two broad layouts described as **post-norm** and **pre-norm**.

Very roughly:

```text
post-norm:
x -> sublayer -> add residual -> norm
```

versus:

```text
pre-norm:
x -> norm -> sublayer -> add residual
```

Modern models make many architecture-specific choices, so when reading real source code I should inspect the model rather than assuming one diagram applies universally.

## One block, step by step

Let us follow one simplified pre-norm decoder block.

Start with a token representation `x`.

First normalize it:

```text
n1 = Norm(x)
```

Then run causal self-attention:

```text
a = Attention(n1)
```

Add the residual:

```text
h = x + a
```

Normalize again:

```text
n2 = Norm(h)
```

Run the MLP:

```text
m = MLP(n2)
```

Add another residual:

```text
y = h + m
```

Now `y` becomes the input to the next Transformer block.

In compact form:

```text
x
-> norm
-> attention
-> residual add
-> norm
-> MLP
-> residual add
-> next block
```

That repeating structure is much closer to what people mean when they talk about the "layers" of an LLM.

## Why repeat the block?

One attention operation does not have to solve language in a single jump.

Each layer receives representations already transformed by previous layers.

So a rough picture is:

```text
embeddings
-> block 1
-> block 2
-> block 3
-> ...
-> block N
```

Representations can become progressively more useful for the model's prediction objective.

I should be careful here: it is tempting to say early layers learn grammar and late layers learn reasoning as if the network had a clean school curriculum. Real models distribute computation in more complicated ways.

The safe claim is simply that later blocks operate on representations produced by earlier blocks.

## Where position enters the picture

Attention by itself needs a way to represent token order or relative position.

The original Transformer used positional encodings. Many modern LLMs use rotary position embeddings, usually called **RoPE**, or other position mechanisms.

That lets the attention computation distinguish arrangements such as:

```text
dog bites man
```

from:

```text
man bites dog
```

although they contain the same three words.

We do not need to derive RoPE today. For the Transformer mental model, just remember that token order must be represented somewhere in the computation.

## Decoder-only LLMs are a particular kind of Transformer

The original paper, *Attention Is All You Need*, described an encoder-decoder Transformer for sequence transduction.

Most autoregressive LLMs we discuss on RAMGPT use a **decoder-only** architecture.

Their self-attention is causal: a token cannot attend to future tokens when predicting the next token.

That gives us the basic loop:

```text
existing tokens
-> Transformer stack
-> next-token probability distribution
-> choose/generate another token
-> repeat
```

We will study inference and sampling later. Right now the important thing is where the Transformer stack sits in that pipeline.

## From text to the Transformer stack

Our Foundations series can finally connect almost all of the pieces we have learned:

```text
text
-> tokenizer
-> token IDs
-> embeddings
-> positional information
-> Transformer block
     -> attention
     -> residual
     -> MLP
     -> residual
-> Transformer block
-> ...
-> final representation
-> output logits
```

And underneath all of this are the ideas from the earlier lessons:

```text
parameters
weights and biases
tensors
forward pass
loss
gradient descent
backpropagation
```

The Transformer did not replace those ideas. It is an architecture built out of them.

## The mental model I am keeping

My shortest version is:

```text
Attention lets tokens exchange information.
The MLP transforms that information.
Residual connections preserve a direct path.
Normalization keeps repeated computation manageable.
Stack the block many times and you get the core of a Transformer language model.
```

That is still not a trained language model.

We have built the architecture, but we have not explained where its useful behavior comes from at scale.

The next lesson is **pretraining**: how enormous amounts of next-token prediction turn an initially random Transformer into a model that has learned patterns of language, knowledge, code, and more.