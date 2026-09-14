---
title: "Tokenization: How Text Becomes the IDs an AI Model Can Read"
description: "AI Foundations #10 connects embeddings to tokenization: how text is split into tokens, mapped to vocabulary IDs, and then looked up as learned vectors."
pubDate: 2026-09-14
updatedDate: 2026-09-14
category: "AI Fundamentals"
tags: ["AI fundamentals", "tokenization", "tokens", "vocabulary", "LLMs"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources:
  - https://huggingface.co/docs/transformers/main/en/tokenizer_summary
  - https://github.com/openai/tiktoken
evidenceStatus: "explainer"
evidenceNote: "A cumulative conceptual lesson based on standard subword tokenization and primary tokenizer documentation."
---

In AI Foundations #9, William explained embeddings: a token ID can select a row from a learned embedding matrix and turn that ID into a vector.

But there is a missing step.

Where does the token ID come from?

If I type:

```text
The cat is sleeping.
```

the model does not receive those characters directly as five neat English words. A **tokenizer** first converts the text into a sequence of tokens, then into integer IDs from a fixed vocabulary.

That sequence of IDs is what connects text to the embedding table.

## Tokens are not the same thing as words

This was the first idea that confused me. I assumed one word meant one token.

Sometimes it does. Often it does not.

A tokenizer may represent a common word with one token while splitting a rarer word into several pieces. Spaces and punctuation can also affect the split.

A simplified example might look like this:

```text
"unbelievable"
-> "un" + "believ" + "able"
```

The exact pieces depend on the tokenizer. Different model families can tokenize the same sentence differently.

So when someone says a model has an 8,000-token context, that does **not** mean it accepts exactly 8,000 English words.

## Why not make every word a vocabulary entry?

Suppose we tried to store every possible English word as one vocabulary item.

We would immediately run into problems: names, spelling variants, technical terms, other languages, numbers, code, new words, and typos. The vocabulary would either become enormous or constantly encounter words it did not know.

At the other extreme, we could tokenize every character separately. That handles arbitrary text, but common words would require many more sequence positions.

Subword tokenization is a compromise. It gives common patterns efficient representations while retaining smaller pieces that can compose less common text.

## Vocabulary: the tokenizer's lookup table

A tokenizer has a vocabulary that maps token pieces to integer IDs.

A toy vocabulary might contain:

```text
"The"      -> 41
" cat"     -> 928
" is"      -> 318
" sleeping"-> 7421
"."        -> 13
```

Those numbers are identifiers. Remember William's warning from the embedding lesson: the number `928` does not mean that `cat` is mathematically close to token `927` or `929`.

The ID tells the model which embedding row to retrieve.

Now the pipeline connects:

```text
raw text
-> tokenizer
-> token pieces
-> token IDs
-> embedding lookup
-> vectors
```

That is the bridge from language into the neural network.

## A tiny worked example

Imagine a ridiculously small vocabulary:

```text
0  <unk>
1  "I"
2  " like"
3  " math"
4  "ing"
5  "."
```

The sentence:

```text
I like math.
```

could become:

```text
[1, 2, 3, 5]
```

while a made-up word might need several smaller pieces or an unknown-token mechanism, depending on the tokenizer design.

The neural network does not care that humans see four words. It receives the resulting ID sequence.

## Why spaces can matter

Many practical tokenizers encode whitespace together with neighboring text or otherwise distinguish word-start patterns.

That means:

```text
"cat"
```

and:

```text
" cat"
```

may not map to the same token.

This looks strange until you think about how often a word occurs after a space. Encoding common text patterns can make the sequence more efficient.

It also explains why copying a string exactly matters when you are debugging token counts.

## Token count affects context length

A model's context window is measured in tokens, not characters and not words.

Two passages with the same number of words can consume different numbers of tokens. Code, unusual names, numbers, multilingual text, and uncommon spellings can all change token density.

This has a direct systems consequence:

```text
more tokens
-> longer sequence
-> more computation
-> more KV-cache memory during generation
```

We will study context and KV cache later in the curriculum. For now, the important point is that tokenization determines the sequence length the transformer actually sees.

## The tokenizer and model must agree

You cannot safely treat tokenizers as interchangeable preprocessing tools.

The model learned its embedding rows and output probabilities using a particular vocabulary and token-ID mapping. If ID `928` means one token to the model but your tokenizer assigns `928` to something else, the numerical interface is broken.

This is why model repositories ship tokenizer configuration alongside model weights.

The weights and tokenizer are separate artifacts, but they were trained to work together.

## Special tokens

Tokenizers can also define special tokens that represent structural roles rather than ordinary text. Depending on the model, examples can include beginning-of-sequence, end-of-sequence, padding, or chat-control markers.

These tokens still have IDs. Their meaning comes from how the model was trained to interpret them.

This is one reason chat templates matter: the visible conversation is transformed into a token sequence containing structure the user may never see on screen.

## Tokenization is not understanding

It is tempting to look at clever subword splits and imagine that the tokenizer understands language.

It does not.

The tokenizer performs a deterministic encoding procedure. The learned semantic relationships we discussed in the embedding lesson live in the model parameters, not in the integer IDs themselves.

A useful separation is:

```text
tokenizer: text -> IDs
embedding layer: IDs -> learned vectors
transformer layers: vectors -> context-dependent representations
```

Each stage solves a different problem.

## What we have built so far

Our curriculum now has a complete path from raw text to the numerical input of a neural network:

```text
text
-> tokens
-> IDs
-> embeddings
-> vectors
```

Earlier lessons explained parameters, tensors, forward passes, training, loss, gradient descent, and backpropagation. Those ideas explain how the embedding table and later layers can be learned.

The next question is much more interesting: once we have a sequence of vectors, **how can each position decide which other positions matter?**

That takes us to attention.