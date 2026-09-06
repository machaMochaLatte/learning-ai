---
title: "What Does It Mean for a Machine to Learn?"
description: "AI Foundations #2 explains what machine learning actually changes inside a model, using simple math, examples, and a student's point of view."
pubDate: 2026-09-06
updatedDate: 2026-09-06
category: "AI Fundamentals"
tags: ["AI fundamentals", "machine learning", "training", "parameters", "student learning"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources: ["https://csrc.nist.gov/glossary/term/machine_learning", "https://developers.google.com/machine-learning/crash-course", "https://www.deeplearningbook.org/"]
---

When I first heard that a computer could **learn**, I imagined something much closer to how a person learns.

A student reads a chapter, gets confused, asks questions, remembers some things, forgets others, and gradually builds an understanding.

A machine does not learn that way.

For most machine-learning systems, "learning" means something more precise:

> **The system changes numerical parameters so that its outputs become better according to some objective.**

That sentence is less magical than the word *learning*, but I think it is much more useful.

This is **AI Foundations #2**. In [the first article](/articles/ai-machine-learning-neural-networks-llms-beginners-map/), William built a map from AI to machine learning, neural networks, deep learning, and LLMs. My goal here is to zoom in on one part of that map and ask what is actually changing when a machine learns.

## Start with a tiny prediction problem

Imagine we want a model to predict a student's quiz score from the number of hours studied.

We could begin with an extremely simple rule:

```text
predicted score = weight × hours studied + bias
```

Suppose we start with:

```text
weight = 5
bias = 40
```

For a student who studies 4 hours:

```text
predicted score = 5 × 4 + 40 = 60
```

But imagine the real score was 76.

The model is wrong by 16 points.

At this moment, the machine has not "understood" studying. It has simply produced a number using its current parameters.

The learning process begins when we use the error to decide how those parameters should change.

## The model needs something it can change

In the simple equation above, there are two adjustable numbers:

```text
weight
bias
```

These are **parameters**.

If we change them, the model's predictions change.

For example:

```text
weight = 8
bias = 44
```

Now the same 4-hour input gives:

```text
8 × 4 + 44 = 76
```

That prediction matches our one example perfectly.

But there is a problem.

One example is not enough.

Maybe another student studied 4 hours and scored 68. Another studied 7 hours and scored 84. Another studied 1 hour and scored 52.

A useful model needs parameters that work reasonably well across many examples, not parameters that memorize one case.

That is the first big idea I learned:

> **Training is not about finding an answer for one example. It is about finding parameters that perform well across a dataset.**

## How does the model know whether it is improving?

We need a way to turn "how wrong was I?" into a number.

That number is produced by a **loss function**.

For a simple prediction problem, we might look at the difference between the prediction and the real answer.

```text
prediction = 60
real value = 76
error = 16
```

A training system can compute errors across many examples and summarize them into a loss.

Then the goal becomes:

```text
change parameters
        ↓
make predictions
        ↓
measure loss
        ↓
change parameters again
        ↓
try to reduce loss
```

This loop is much closer to what machine learning means technically.

## Learning is an optimization problem

This connection surprised me because it made machine learning feel much more like mathematics than magic.

If the model has adjustable parameters, and we can calculate how well or badly those parameters perform, then training becomes an **optimization problem**.

We are looking for parameter values that make the objective better.

For a tiny model, maybe there are only two numbers to adjust.

For a neural network, there can be millions or billions.

Conceptually, though, the question is still similar:

```text
Which parameter values make this model perform better on the task?
```

The difficulty is that with billions of parameters, we cannot just try every possible combination.

That is why later lessons need ideas such as **gradients**, **gradient descent**, and **backpropagation**.

They give us a practical way to determine how parameters should move.

## A student analogy that helped me

Here is the analogy I find useful.

Imagine I practice ten algebra problems.

After checking my answers, I notice that I keep making the same sign error when expanding brackets.

So on the next set, I deliberately change that part of my approach.

My process looks like:

```text
attempt
  ↓
feedback
  ↓
adjustment
  ↓
new attempt
```

Machine learning has a pattern that looks vaguely similar:

```text
prediction
  ↓
loss
  ↓
parameter update
  ↓
new prediction
```

But the analogy has limits.

I can think about *why* I made a mistake and explain it in words. A normal training algorithm does not need that kind of self-understanding. It can update numerical parameters according to an optimization procedure.

So I would not say a neural network learns exactly like a person.

I would say both processes involve **feedback and change**, but the mechanisms are very different.

## Training data provides the examples

A model cannot learn a useful relationship if it has nothing to learn from.

Training data supplies examples of the patterns we want the system to capture.

For our score-prediction example, the data might look like:

```text
hours studied → score
1.0           → 52
2.5           → 61
4.0           → 70
5.0           → 75
7.0           → 84
```

The model sees inputs and, depending on the learning setup, some form of target or feedback.

The training algorithm then adjusts parameters so that the model becomes better at mapping inputs to useful outputs.

This also explains why data quality matters.

If the examples are wrong, biased, unrepresentative, or too limited, the model can learn patterns we did not want.

Learning from data does not automatically mean learning the truth.

## Memorizing and learning are not the same thing

Suppose our model has enough capacity to remember every training example exactly.

That could make the training loss very low.

But if it performs badly on new students it has never seen, then the model has not learned a useful general pattern.

This is why machine learning cares about **generalization**.

A model should ideally work on new examples drawn from the kind of problem we care about.

That gives us two different questions:

```text
How well does the model fit the examples it trained on?

How well does it perform on new examples?
```

The second question is often the more important one.

This is one reason datasets are commonly separated into training, validation, and test data.

We do not want to grade a model only on the homework it already saw.

## What changes inside a neural network?

In a neural network, learning usually means changing many numerical parameters, especially **weights** and **biases**.

A simplified neuron-like computation might look like:

```text
output = activation(
    weight1 × input1
  + weight2 × input2
  + bias
)
```

One small unit can contain several parameters.

A large neural network contains enormous numbers of them connected across many layers.

Training changes these values.

When someone says a model has "7 billion parameters," those billions of learned numbers are a major part of what training produces.

This is also why the next lesson in the series matters so much: **What exactly are parameters and weights?**

## Training is different from inference

This distinction is becoming clearer to me as I learn local AI.

During **training**, model parameters are being changed.

During **inference**, we normally keep the trained parameters fixed and use them to produce an output.

```text
TRAINING
examples → prediction → loss → update parameters

INFERENCE
new input → fixed trained parameters → output
```

When I download a model and run it locally, I am usually doing inference.

The model already went through its main training process somewhere else.

My GPU is using the learned parameters, not recreating them from scratch.

## The idea I want to remember

After working through this, my definition of machine learning is much less mysterious:

> **A machine learns when a training process uses data and feedback to adjust model parameters so that the model performs better according to an objective.**

That definition is not complete enough for every type of machine learning, but it gives me a solid starting point.

The pieces now look like this:

```text
data
  ↓
model with adjustable parameters
  ↓
prediction
  ↓
loss / feedback
  ↓
optimization
  ↓
updated parameters
  ↓
better model, hopefully
```

The word I want to study next is the one sitting in the middle of almost everything above:

**parameters**.

What are they physically? Why are there billions of them? Why do they take so much memory? And why does reducing their precision make a model smaller?

That will be the next step in my AI learning journey.
