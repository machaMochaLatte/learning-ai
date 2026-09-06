---
title: "How I Use ChatGPT to Understand Math, Not Do My Homework"
description: "A Whitby high school student explains how she uses ChatGPT for hints, error analysis, proofs, calculus, vectors, and practice without outsourcing the thinking."
pubDate: 2026-08-20
updatedDate: 2026-08-20
category: "Tutorials"
tags: ["ChatGPT", "math", "high school", "learning", "study skills", "AI tutoring", "education"]
draft: false
featured: false
author: "machaMochaLatte"
editor: "ramgpt-editorial-team"
sources: ["https://www.dcp.edu.gov.on.ca/en/curriculum/secondary-mathematics", "https://uwaterloo.ca/future-students/admissions/admission-requirements"]
---

I use ChatGPT for math a lot, but I try very hard not to use it as a machine that finishes homework for me.

That sounds like a small difference. It is actually the whole difference.

I am a high school student in Whitby, Ontario, and I hope to study at the University of Waterloo one day. So when I am doing Advanced Functions, calculus, vectors, or even a problem that is just annoyingly hard, the thing I care about is not whether I can make the homework page look complete.

I care about whether I can look at a different problem tomorrow and know how to start.

If ChatGPT gives me a beautiful solution that I could never reproduce on my own, that is not really a win.

The way I like to use it is more like this:

```text
I try the problem first
↓
I get stuck or make a mistake
↓
ChatGPT gives me the smallest useful hint
↓
I try again
↓
ChatGPT checks my reasoning
↓
I explain the idea back
↓
I do a new problem without help
```

That takes longer than copying an answer.

It also works much better.

## My main rule: do not let ChatGPT take over the thinking

There are prompts that make ChatGPT do almost everything:

```text
Solve this question.
```

```text
Show all the steps.
```

```text
Give me the final answer.
```

Sometimes I still use those after I am completely finished and want to compare methods. But I try not to start there.

My better prompts sound more like:

```text
Do not solve this yet.
Tell me whether my first step is valid.
```

or:

```text
I think there is a mistake in my reasoning.
Find the first incorrect step only.
```

or:

```text
Ask me one question that will help me notice what I missed.
```

The goal is to keep me inside the problem.

## Example 1: rational functions and the information you can accidentally cancel away

This is the kind of mistake that is more interesting than simple arithmetic.

Suppose I am working with:

<div class="math-display">f(x) = (x² − 5x + 6) / (x² − 4)</div>

I factor it:

<div class="math-display">f(x) = ((x − 2)(x − 3)) / ((x − 2)(x + 2))</div>

Then I simplify:

<div class="math-display">f(x) = (x − 3) / (x + 2)</div>

At first, I might look at the simplified expression and say:

<div class="math-display">x ≠ −2</div>

because that is the value that makes the new denominator zero.

But something is missing.

A bad prompt would be:

```text
What are the restrictions and asymptotes?
```

That would let ChatGPT finish the entire analysis.

A much better prompt is:

```text
I simplified
(x-2)(x-3) / [(x-2)(x+2)]
to
(x-3)/(x+2).

I think I may have lost information about the original domain.
Do not tell me the missing restriction.
Ask me one question that will make me find it.
```

The useful question is basically:

> Before you cancelled the factor `x − 2`, when was the original denominator zero?

Then I have to go back to the original function.

The original denominator is zero at:

<div class="math-display">x = 2 &nbsp;&nbsp;and&nbsp;&nbsp; x = −2</div>

Now the picture makes sense:

- `x = 2` was cancelled algebraically, but the original function was still undefined there, so there is a removable discontinuity, or hole.
- `x = −2` remains in the denominator, so it corresponds to a vertical asymptote.

What ChatGPT helped me notice was not "how to factor."

It helped me notice that **algebraic simplification can hide domain information**.

That is much more useful.

## I like prompts that expose the gap instead of filling it

This is one of my favourite ways to use ChatGPT.

Instead of saying:

```text
Explain what I did wrong.
```

I can say:

```text
Do not correct me yet.
Ask me a question that exposes the gap in my reasoning.
```

That makes the interaction feel more like a teacher who knows I am almost there.

For example, if I claim:

> If a factor cancels, then that value is no longer excluded from the domain.

I can ask:

```text
Do not tell me whether my claim is true.
Give me a question I should ask about the original function before simplifying it.
```

The answer I want is not the result.

I want the missing habit of thought.

## Example 2: proving a trigonometric identity without turning it into symbol soup

Trig identities are one of the easiest places to fool myself into thinking I understand something because the final line looks familiar.

Suppose I need to verify:

<div class="math-display">sin x / (1 + cos x) = (1 − cos x) / sin x</div>

A full solution can be generated instantly.

But if I copy it, I may not learn how to recognize the move next time.

So I prefer a prompt like:

```text
I need to prove
sin(x)/(1+cos(x)) = (1-cos(x))/sin(x).

Do not prove it for me.
Give me one strategic hint only.
The hint should tell me what expression to multiply by,
but not show the finished algebra.
```

That can push me toward multiplying by a conjugate-style expression involving:

<div class="math-display">1 − cos x</div>

Then I get:

<div class="math-display">[sin x / (1 + cos x)] × [(1 − cos x) / (1 − cos x)]</div>

and the denominator becomes:

<div class="math-display">1 − cos² x</div>

which connects to:

<div class="math-display">sin² x + cos² x = 1</div>

so:

<div class="math-display">1 − cos² x = sin² x</div>

Now I am using an identity I already know to build the new one.

The part I want ChatGPT to teach me is the recognition:

```text
1 + cos(x)
paired with
1 - cos(x)
↓
difference of squares
↓
1 - cos²(x)
↓
sin²(x)
```

That pattern is reusable.

The finished answer is not.

## I also ask ChatGPT to challenge my proof

When I finish a proof, I can ask:

```text
Read my identity proof as a strict math teacher.
Do not rewrite it.
Tell me whether every transformation is reversible
and whether I ignored any domain restrictions.
```

That last part matters.

Two expressions can agree wherever both are defined, but denominators still matter. A proof that manipulates fractions without thinking about where they exist can look perfect and still be incomplete.

This is a good example of something ChatGPT can help me check after I have already done the main work.

## Example 3: calculus is better when ChatGPT checks my reasoning, not my derivative

Differentiation rules can become mechanical pretty quickly.

The more difficult part for me is usually interpreting the derivative.

Suppose:

<div class="math-display">f(x) = x³ − 3x² − 9x + 5</div>

I calculate:

<div class="math-display">f′(x) = 3x² − 6x − 9</div>

and factor:

<div class="math-display">f′(x) = 3(x − 3)(x + 1)</div>

So the critical numbers are:

<div class="math-display">x = −1 &nbsp;&nbsp;and&nbsp;&nbsp; x = 3</div>

At this point, I do not want ChatGPT to finish the question.

I can ask:

```text
I found critical numbers x=-1 and x=3.
Do not find the local maximum or minimum for me.
Give me a blank sign-chart structure with the correct intervals,
but leave the signs of f'(x) empty.
```

Then I get something like:

```text
Interval          sign of f'(x)
(-∞, -1)          ?
(-1, 3)           ?
(3, ∞)            ?
```

I choose test values and fill it myself.

If I get:

```text
(+), (-), (+)
```

then I have to explain why:

- positive derivative means the function is increasing;
- negative derivative means it is decreasing;
- changing from positive to negative gives a local maximum;
- changing from negative to positive gives a local minimum.

Then I ask ChatGPT:

```text
Check my sign chart and my interpretation.
If I am wrong, tell me which interval is wrong first.
Do not give me the corrected chart until I retry it.
```

That keeps the derivative connected to the shape of the function.

## A question I like even more: is my rule always true?

Here is a statement that sounds reasonable:

> If `f′(a) = 0`, then `f` has a local maximum or minimum at `x = a`.

Instead of asking ChatGPT whether that is true, I can say:

```text
I think every point where f'(x)=0 must be a local max or min.
Do not tell me yes or no.
Give me the simplest counterexample if my claim is false,
but let me analyze why it works.
```

A classic counterexample is:

<div class="math-display">f(x) = x³</div>

because:

<div class="math-display">f′(x) = 3x²</div>

and:

<div class="math-display">f′(0) = 0</div>

but `x = 0` is not a local maximum or minimum.

The function keeps increasing through the point.

That one example teaches me more than memorizing a sentence about critical points, because it forces me to separate:

```text
f'(a)=0
```

from:

```text
there is definitely an extremum at a
```

Those are not the same claim.

## Example 4: vectors are easier when I ask for two explanations

Vectors are another topic where formulas can work before the idea feels intuitive.

For example, I know that if two non-zero vectors are perpendicular, then their dot product is zero:

<div class="math-display">a · b = 0</div>

I could memorize that.

But I would rather ask:

```text
I know perpendicular vectors have dot product 0.
Explain why in two different ways:

1. using the formula a·b = |a||b|cos(theta)
2. using coordinates and geometry

Do not just restate the rule.
```

The first explanation connects perpendicularity to:

<div class="math-display">θ = 90°</div>

and therefore:

<div class="math-display">cos 90° = 0</div>

so:

<div class="math-display">a · b = |a||b| cos θ = |a||b| × 0 = 0</div>

The second explanation can connect the coordinate formula to projections and direction.

I like asking for two explanations because one is often more memorable than the other.

And if I can explain both back, I probably understand the idea instead of only knowing the formula.

## Example 5: I use ChatGPT to test whether I actually understand a function transformation

It is easy to look at:

<div class="math-display">y = −2f(3(x − 4)) + 1</div>

and recite words like:

```text
reflection
vertical stretch
horizontal compression
shift right
shift up
```

The harder part is getting the order and horizontal factor correct.

So instead of asking for the transformation list, I can write my own first:

```text
My interpretation:
- reflect over x-axis
- vertical stretch by 2
- horizontal compression by 3
- shift right 4
- shift up 1

Do not replace my answer.
Check each claim one at a time and make me justify
the horizontal transformation before you approve it.
```

That last instruction is important because horizontal transformations are where I am most likely to memorize a rule without understanding why the inside factor behaves differently.

I can then ask:

```text
Give me one point (a, b) on y=f(x).
Make me predict where that point moves under y=-2f(3(x-4))+1.
Do not show the transformed point until I answer.
```

Now I have to use the transformation, not just name it.

## The best thing ChatGPT can do is make me retrieve knowledge

Reading an explanation feels productive.

Retrieving the idea from memory is harder.

That is why I often switch ChatGPT from "teacher" mode to "quiz" mode.

For example:

```text
Quiz me on Advanced Functions.
Ask one question at a time.
Mix rational functions, trig, exponentials, logarithms, and transformations.
Do not tell me the topic before the question.
If I am wrong, give me one hint before explaining.
```

The line:

```text
Do not tell me the topic before the question.
```

matters more than it seems.

On a worksheet, the heading might say:

> Solve using logarithms.

On a real test, the difficult part may be recognizing that logarithms are the right tool.

I want practice choosing the method, not only executing it.

## I ask for new problems that target my exact mistake

Suppose I solve a rational-function question and forget the excluded value that disappeared after cancellation.

I can ask:

```text
Create three new rational-function problems where a common factor cancels.
Do not make them identical to my original problem.
Make me identify:
- the original domain restrictions,
- any hole,
- any vertical asymptote.
Do not provide answers yet.
```

Or if my problem is calculus sign charts:

```text
Give me three derivative sign-chart problems.
In one, a critical number should be a local maximum.
In one, it should be a local minimum.
In one, f'(x)=0 should not create either.
Do not tell me which is which.
```

That is much more useful than asking for ten random questions.

The practice is based on the weakness I just discovered.

## I keep a mistake log, and ChatGPT helps me classify it

Before a test, I do not only want a list of questions I got wrong.

I want to know **why** I got them wrong.

My list might look like:

```text
- forgot original domain restriction after cancelling a factor
- used log(a+b) = log(a) + log(b)
- solved for critical numbers but forgot to test intervals
- confused a direction vector with a normal vector
- algebra sign error while completing the square
```

Then I ask:

```text
Group these mistakes by underlying skill.
Do not give me solutions to the old questions.
Tell me what kind of new practice would expose whether I fixed each weakness.
```

The categories might become:

```text
Conceptual misunderstanding
Algebraic fluency
Domain/restriction awareness
Method selection
Interpretation of results
```

That is much more useful than thinking:

> I got five questions wrong, so I am bad at math.

Usually the mistakes are not five separate problems.

They are one or two habits showing up in different places.

## I use the "teach it back" test

This is probably the most uncomfortable method, which is why I think it works.

I tell ChatGPT:

```text
I am going to explain why a derivative can be zero without giving a local extremum.
Do not rewrite my explanation.
Grade it for mathematical completeness.
Tell me what idea is missing, if any.
```

Then I try to explain it in my own words.

If I write:

> A derivative of zero only tells us the tangent is horizontal at that point. We still need to know how the derivative behaves on each side. For `f(x) = x³`, the derivative is zero at 0, but it is positive on both sides, so the function is increasing through the point.

then I have actually connected:

```text
critical number
↓
derivative sign
↓
increasing/decreasing behaviour
↓
classification of the point
```

That is a lot stronger than recognizing the answer in someone else's solution.

## Sometimes I deliberately tell ChatGPT to be less helpful

A normal assistant tries to be helpful as quickly as possible.

For math, that can be too helpful.

I sometimes start a session with this:

```text
Act as my math tutor, not my homework solver.

Rules:
- Ask what I have tried before solving anything.
- Give only one hint at a time.
- If I make a mistake, identify the first incorrect step.
- Let me correct it before continuing.
- Ask me to explain important ideas in my own words.
- After I solve a problem, give me one similar problem with no hints.
- Only show a full solution if I explicitly ask after trying twice.
```

That completely changes the conversation.

It also makes ChatGPT slightly annoying sometimes.

That is okay.

If I am never frustrated for even thirty seconds, there is a good chance the AI is doing too much of the work.

## I do not want every problem broken into tiny steps

There is another trap.

Even if ChatGPT does not give the final answer, it can still over-scaffold the problem.

For example:

```text
Step 1: factor this.
Step 2: cancel this.
Step 3: set this equal to zero.
Step 4: test this interval.
```

At that point, I am basically following instructions.

So once I understand a topic, I reduce the help.

I might say:

```text
Give me a problem at the same difficulty,
but do not tell me which technique to use.
```

Then later:

```text
Give me a harder one that combines two ideas.
```

Eventually:

```text
Give me a non-routine problem where the obvious first method is not the best one.
```

That progression matters because I want to become independent of the tutor, including the AI tutor.

## ChatGPT is especially useful when my question is "why?"

The most useful math questions I ask are often not:

```text
How do I solve this?
```

They are:

```text
Why is this step legal?
```

```text
Why does this theorem need that condition?
```

```text
Why does this method fail here?
```

```text
Can you give me a counterexample to my claim?
```

```text
What information did I lose when I simplified this expression?
```

```text
What would have to be true for my argument to work?
```

Those questions force me to think about structure instead of only procedure.

That is the kind of math I expect to matter more and more as the problems get harder.

## I still check ChatGPT because it can be wrong

This is important.

ChatGPT can make algebra mistakes, skip restrictions, misread a graph, use a theorem incorrectly, or sound very confident while doing something questionable.

So I do not treat it as an answer key.

If the result matters, I can ask it to verify in a second way:

```text
Check the result by substitution.
```

or:

```text
Solve it using a different method and compare the two results.
```

or:

```text
List every assumption used in this proof and check whether each one is satisfied.
```

For school work, my teacher's instructions and course material still matter more than a chatbot response.

Being skeptical of AI is part of using it well.

## What using ChatGPT badly looks like for me

Bad use is not just obvious copying.

It can also look like this:

```text
I read the question
↓
I feel stuck for 20 seconds
↓
I ask ChatGPT
↓
I understand its solution while looking at it
↓
I assume I now know how to do the problem
```

That last step is dangerous.

Recognizing someone else's solution is not the same as generating one yourself.

So after ChatGPT helps me, I try to close the explanation and do another problem from a blank page.

If I cannot do that, I probably did not learn it yet.

## My favourite prompt for serious math study

If I had to keep only one prompt, it would be this:

```text
Act as a high school math tutor, not a solution generator.

I will show you a problem and my attempt.

1. Identify the main concept being tested, but do not tell me the method unless I ask.
2. Check my reasoning and point out only the first mistake.
3. Ask me a question or give one small hint that helps me continue.
4. Let me retry before explaining more.
5. If my reasoning is correct, challenge me to explain why the step works.
6. After I finish, give me one new problem that tests the same idea in a different form.
7. Do not provide the full solution unless I explicitly ask for it after at least two attempts.
```

I like this because it makes ChatGPT do something more difficult than producing answers.

It has to leave enough of the problem for me.

## Why this matters to me

I want to go to Waterloo eventually.

That is one reason I care about using AI this way.

I do not expect harder math to become a series of worksheets where every question looks exactly like the example above it.

I want to be able to handle the moment where I look at a problem and think:

> I have never seen this exact thing before.

Then I want to ask myself:

```text
What do I know?
What is the question really asking?
What information matters?
What theorem or representation might connect these pieces?
Can I test a simpler case?
Can I find a counterexample?
Can I explain why my next step is valid?
```

ChatGPT can help me practise those questions.

But it cannot practise them **for** me.

That is the boundary I try to keep.

I do not want AI to make math disappear.

I want it to help me stay with a difficult problem long enough that I eventually do not need the help.
