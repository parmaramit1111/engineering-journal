---
Article Type: Engineering Learning Journal
Category: AI
Difficulty: Intermediate
Reading Time: 8–10 minutes
Published: August 2026
---


# Before the LLM: Can a Machine Learn With Just a Few Numbers?

*What I learned by building the first few pieces of a neural network from scratch.*

When we hear **LLM**, we usually think about billions of parameters, transformers, attention mechanisms, GPUs, and enormous datasets.

It is easy to look at all of that and think:

> "There is no way I can understand what is happening inside."

I recently started approaching the problem from the opposite direction.

Instead of starting with an LLM, I started with something almost ridiculously small.

One input.

One weight.

One bias.

One neuron.

And then I asked a simple question:

**Can these few numbers actually learn something?**

That question turned out to be much more interesting than I expected.

---

## Start With One Very Simple Idea

Imagine we have an input:

```text
x = 4
```

and a tiny mathematical unit with:

```text
weight = 2
bias = 1
```

The calculation is:

```text
output = weight × input + bias

       = 2 × 4 + 1

       = 9
```

That's it.

There is nothing magical happening here.

It's just multiplication and addition.

But there is an important idea hidden inside those numbers.

The **weight** determines how strongly the input influences the output.

The **bias** shifts the result.

So instead of thinking:

> "A neuron is an advanced AI concept."

we can think:

> "A neuron is a small calculation whose parameters can be changed."

That change in perspective was important for me.

---

## But Where Is the Learning?

So far, nothing has learned anything.

We gave the neuron:

```text
x = 4
```

and it produced:

```text
9
```

Suppose the answer we actually wanted was:

```text
10
```

Our neuron is wrong.

The difference between what we predicted and what we wanted is what eventually gives us a way to improve the parameters.

This leads to the next question:

**How do we measure how wrong we are?**

---

## Give the Mistake a Number

For multiple outputs, we can use a loss function such as Mean Squared Error (MSE).

For our small experiment, imagine our model predicts:

```text
[9, 12, 4]
```

while the target is:

```text
[10, 10, 3]
```

The model isn't perfect.

MSE gives us a single number representing the overall error:

```text
Loss = 2.0
```

The exact number isn't the most important part.

The important idea is this:

> **The model now has a measurable signal telling it how wrong it was.**

And that creates the possibility of learning.

---

## One Neuron Isn't Enough

Now let's make the experiment slightly more interesting.

Instead of one neuron, let's use three.

All three receive the same input:

```text
x = 4
```

But they have different parameters.

```text
Neuron 1
weight = 2
bias   = 1

Neuron 2
weight = 3
bias   = 0

Neuron 3
weight = 0.5
bias   = 2
```

Each neuron performs the same basic calculation.

```text
Neuron 1 → 2 × 4 + 1   = 9

Neuron 2 → 3 × 4 + 0   = 12

Neuron 3 → 0.5 × 4 + 2 = 4
```

So the layer produces:

```text
[9, 12, 4]
```

Something interesting has happened.

We haven't made the individual neurons more complicated.

We've simply created **more of them**, each with its own parameters.

That is the beginning of a neural network.

---

## Now We Have a Problem to Solve

Suppose our target is:

```text
[10, 10, 3]
```

Our prediction is:

```text
[9, 12, 4]
```

So each neuron contributed a different error.

The next question becomes much more interesting:

> **Which parameters should we change, and by how much?**

This is where the backward pass enters the picture.

---

## Going Backward

During the forward pass, information moves in one direction:

```text
Input
  ↓
Neurons
  ↓
Prediction
  ↓
Loss
```

The backward pass goes the other way:

```text
Loss
  ↓
Gradients
  ↓
Parameters
```

The gradient tells us something extremely useful:

> **If I change this parameter slightly, which direction will make the loss smaller?**

For our three neurons, the loss produces an output gradient for each prediction.

Conceptually:

```text
Neuron 1 → "move this way"
Neuron 2 → "move that way"
Neuron 3 → "move this way"
```

We can then calculate how those gradients affect each neuron's weight and bias.

This is the basic idea behind **backpropagation**.

It sounds complicated when described as a major neural-network concept.

But at this scale, we can actually calculate it ourselves.

---

## The Part That Changed My Mental Model

Before implementing this, I thought of backpropagation as something mysterious that frameworks such as PyTorch simply "do."

Now I see it differently.

At its core, we're repeatedly answering:

1. What did we predict?
2. How wrong were we?
3. Which parameters contributed to that error?
4. In which direction should those parameters move?
5. How much should they move?
6. What happens if we try again?

That's the training loop.

---

## The Optimizer Makes the Adjustment

Once we have gradients, we need something to update our parameters.

A simple approach is **Gradient Descent**.

The basic rule is:

```text
new parameter =
    old parameter - learning rate × gradient
```

For example, if a weight is:

```text
w = 2.0
```

and its gradient is:

```text
-2.6667
```

with a learning rate of:

```text
0.1
```

then:

```text
new weight
= 2.0 - (0.1 × -2.6667)
≈ 2.2667
```

We changed the parameter slightly.

Now we run the network again.

---

## And Then We Repeat

This is the part I find most fascinating.

There isn't a single moment where the model suddenly "learns."

Instead:

```text
Predict
   ↓
Measure error
   ↓
Calculate gradients
   ↓
Update parameters
   ↓
Predict again
   ↓
Measure error again
   ↓
...
```

The cycle repeats.

Ideally:

```text
Loss ↓
```

over time.

And that simple loop is one of the foundations underneath much larger neural networks.

---

## A Different Way to Think About It

Here's the mental model that helped me understand it.

Imagine a manager trying to improve a team.

The **trainer** is the manager.

The **layer** is the team.

Each **neuron** is an individual worker.

The **prediction** is the team's result.

The **loss** is the performance evaluation.

The **gradient** is feedback telling each worker which direction to improve.

The **optimizer** adjusts the parameters based on that feedback.

Then the team tries again.

```text
                TRAINER
                   │
                   ▼
              ┌─────────┐
              │  LAYER  │
              └─────────┘
               │   │   │
               ▼   ▼   ▼
              N1  N2  N3
               │   │   │
               └───┼───┘
                   ▼
              Prediction
                   │
                   ▼
                 Loss
                   │
                   ▼
               Feedback
                   │
                   ▼
              Update Parameters
                   │
                   └──────► Try Again
```

The analogy isn't the mathematics.

It is simply a mental model for remembering the roles.

---

## So Where Does This Lead to an LLM?

Here's where we need to be careful.

Three neurons are **not an LLM**.

Even a small neural network is nowhere close to the scale or architecture of a modern language model.

But the progression is worth understanding:

```text
One calculation
       ↓
One neuron
       ↓
Multiple neurons
       ↓
A layer
       ↓
Multiple layers
       ↓
Neural network
       ↓
Large neural network
       ↓
Transformer architecture
       ↓
Large Language Model
```

The scale, architecture, training data, optimization techniques, and capabilities change dramatically.

But the fundamental idea of:

**parameters → prediction → error → gradients → parameter updates**

remains incredibly important.

---

## What I Actually Learned

The biggest lesson wasn't a particular equation.

It was changing how I think about neural networks.

Previously, terms like:

* neuron
* weight
* bias
* loss
* gradient
* optimizer
* backpropagation

felt like separate concepts that had to be memorized.

Building a tiny version changed that.

They are parts of the same story.

A model makes a prediction.

The prediction is compared with what we wanted.

The error produces a signal.

That signal travels backward.

Parameters are adjusted.

Then we try again.

**Predict → Measure → Adjust → Repeat.**

That's the mental model I'm taking forward as we continue building toward the larger pieces of an LLM.

---

## The Interesting Part Is What's Next

So far we've deliberately kept the system tiny.

We haven't introduced:

* activation functions
* multiple layers
* matrix operations
* batches
* sophisticated optimizers
* embeddings
* attention
* transformers

And that's intentional.

We're going one building block at a time.

Because before trying to understand how an LLM processes language, I want to understand the machinery that makes learning possible in the first place.

**The goal isn't to memorize how an LLM works.**

The goal is to be able to build enough of its foundations that the architecture stops looking like magic.

And that's exactly where our next lesson begins.

---

# Key Takeaways

* A neuron can be understood as a simple parameterized calculation.
* Weights control how strongly inputs influence outputs.
* Bias provides an adjustable offset.
* Multiple neurons allow a layer to represent multiple relationships.
* Loss gives the model a measurable signal about its error.
* Gradients tell us how parameters should move to reduce that error.
* The optimizer updates the parameters.
* Training is fundamentally a repeated predict → measure → adjust cycle.
* Understanding these small building blocks makes larger neural networks less mysterious.
* An LLM is enormously more complex, but these foundations are worth understanding.

---

# Project

**LLM From Scratch**

[View the source code and learning journey on GitHub](https://github.com/parmaramit1111/llm-from-scratch)

---

# Related Engineering Topics

- Large Language Models
- Neural Networks
- Machine Learning
- Deep Learning
- Backpropagation
- Gradient Descent
- Neural Network Training
- AI Engineering
- LLM From Scratch

---

**Article Version:** 1.0

**First Published:** August 2026

**Last Reviewed:** August 2026