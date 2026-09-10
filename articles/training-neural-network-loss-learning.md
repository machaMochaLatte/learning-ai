---
title: "How Does a Neural Network Start Learning? Training Begins With Loss"
description: "AI Foundations #6 connects the forward pass to training by showing why a model needs targets and a loss function before its weights can improve."
pubDate: 2026-09-10
updatedDate: 2026-09-10
category: "AI Fundamentals"
tags: ["AI fundamentals", "neural networks", "training", "loss function", "machine learning", "student learning"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "RAMGPT Editorial Team"
sources: ["https://docs.pytorch.org/tutorials/beginner/basics/optimization_tutorial.html", "https://docs.pytorch.org/docs/stable/nn.html#loss-functions", "https://developers.google.com/machine-learning/crash-course/linear-regression/loss"]
---

In the last lesson, William followed an input through a neural network and showed how a forward pass produces an output.

That left me with a question that sounds obvious but is actually the beginning of training:

**How does the network know that its output was bad?**

Suppose the model predicts `0.82` and the correct answer is `1.00`. We can look at those two numbers and immediately see a difference. The computer needs that difference turned into a precise numerical objective.

That is what a **loss function** does.

I think of this as the point where a neural network stops being only a calculator and becomes something we can train.

## First, separate prediction from learning

The forward pass from the previous lesson can be summarized as:

```text
input → model → prediction
```

The model uses its current weights and biases. Nothing about that operation automatically changes them.

For training, we add a known target:

```text
input → model → prediction
                  ↓
              compare
                  ↑
                target
```

The comparison produces a number: the loss.

```text
input → prediction → loss
           ↑           ↑
         model       target
```

A smaller loss generally means the prediction is closer to what the training objective wants. A larger loss means it is worse according to that objective.

The important phrase is **according to that objective**. Loss is not a universal measurement of intelligence or truth. It is a mathematical rule chosen for the task.

## A tiny squared-error example

Take the prediction:

```text
prediction = 0.82
```

and target:

```text
target = 1.00
```

One simple loss is squared error:

```text
loss = (prediction - target)²
```

So:

```text
loss = (0.82 - 1.00)²
     = (-0.18)²
     = 0.0324
```

Now imagine another set of weights produces:

```text
prediction = 0.40
```

Then:

```text
loss = (0.40 - 1.00)²
     = 0.36
```

`0.0324` is much smaller than `0.36`, so under this loss function the first prediction is better.

This gives the training process something crucial: a score that changes when the model's parameters change.

## Why not just use right or wrong?

At first I wondered why we need a loss at all. If the task has a correct answer, why not give the model `0` for wrong and `1` for right?

The problem is that training needs more information than a final verdict.

Imagine two predictions for a target of `1.00`:

```text
A = 0.99
B = 0.01
```

If both are simply labeled "wrong," we throw away the fact that A is extremely close and B is extremely far away.

A useful loss function gives us a landscape with degrees of error. Training can then ask not only whether the model is wrong, but which direction would reduce the error.

That direction is where gradients will enter the story.

## Training means repeating a loop

A neural network does not usually learn from one magical calculation. Training repeatedly performs a cycle.

At a high level:

```text
1. take training data
2. run a forward pass
3. calculate loss
4. calculate how parameters affected that loss
5. adjust parameters
6. repeat
```

We already understand steps 1 through 3 well enough to see the structure.

Steps 4 and 5 are the next mathematical jump. They involve gradients, backpropagation, and an optimizer such as gradient descent.

But I do not want to rush there, because the loss function gives those later operations their purpose.

Without a loss, "improve the model" is vague. With a loss, improvement can mean: **change the parameters so this objective becomes smaller.**

## One training example is not the whole dataset

Real training data contains many examples.

Suppose we have four examples with individual squared errors:

```text
0.04
0.01
0.16
0.09
```

A simple mean squared error would average them:

```text
(0.04 + 0.01 + 0.16 + 0.09) / 4 = 0.075
```

This matters because we usually want parameters that work well across many examples, not parameters that perfectly memorize the error of one example at one instant.

Training systems commonly process examples in **batches**. That connects directly back to our tensor lesson: instead of one input tensor, the model often receives a tensor containing a batch of inputs, produces a batch of predictions, and computes a loss over that batch.

The ideas from earlier lessons are starting to stack rather than remain separate definitions.

## Different problems need different losses

Squared error is easy to visualize, but it is not the right choice for every task.

Regression problems often use losses based on numerical distance, such as mean squared error or mean absolute error.

Classification problems often use cross-entropy-style losses. Instead of asking only how far one number is from another, the loss measures how the model's predicted distribution relates to the correct class.

Language models are a classification problem repeated over a huge vocabulary. Given the preceding context, the model produces scores for possible next tokens. During training, the known next token supplies the target, and a loss penalizes the model when it assigns insufficient probability to that target.

So when an LLM learns from text, we can sketch one tiny piece of the process like this:

```text
"The capital of France is" → model → token probabilities
                                      ↓
                              target token: "Paris"
                                      ↓
                                    loss
```

Of course, real language-model training operates over many tokens and batches in parallel. But the logical relationship is the same as our tiny numerical example.

## Loss is not the same as accuracy

This distinction confused me at first.

Accuracy is often a metric we use to describe whether final predictions are correct. Loss is the numerical objective used to guide optimization.

Two models can have the same accuracy while having different losses.

For example, suppose a binary classifier considers anything above `0.5` to be class 1. Predictions of `0.51` and `0.99` may both count as correct for a class-1 example, so their accuracy contribution is identical. But a suitable loss can distinguish the model that barely crossed the boundary from the model that assigned much stronger probability to the target.

That extra information is useful during training.

## The model still has not learned yet

This is the part I want to emphasize.

Calculating a loss does **not** change the model.

After the forward pass and loss calculation, the weights are still exactly what they were before.

We now know how bad the output was, but we have not answered the harder question:

**Which weights caused the error, and how should each one move?**

A modern neural network can have millions or billions of parameters. We cannot sensibly try random changes to every weight and rerun the whole model until something improves.

We need mathematics that tells us how sensitive the loss is to changes in those parameters.

That sensitivity is described by a **gradient**.

## Why gradients are the next step

Imagine standing on a hill in fog. You cannot see the entire landscape, but you can measure the slope under your feet.

If your goal is to move downhill, the local slope tells you which direction decreases your height most quickly.

Training a neural network uses a related idea. The loss is like the height, and the model parameters determine where we are on an enormous mathematical landscape.

The gradient tells us how the loss changes when parameters change.

This analogy is imperfect, especially in billions of dimensions, but it gives me a useful mental bridge:

```text
prediction tells us what the model produced
loss tells us how bad it was
gradient tells us how the loss changes
optimization uses that information to update parameters
```

We will make that precise in the next lessons.

## The chain so far

We can now connect the first part of the series into one continuous system:

```text
parameters / weights
        ↓
organized as tensors
        ↓
used in a forward pass
        ↓
produce predictions
        ↓
compared with targets
        ↓
produce loss
```

Nothing in that chain requires the model to be mysterious.

It is a sequence of numerical operations with a measurable objective.

The next step is where the word **learning** becomes more literal. We need to use the loss to determine how the parameters should change.

That means learning what a gradient actually is, and why **gradient descent** can turn a loss value into a direction for improving the model.