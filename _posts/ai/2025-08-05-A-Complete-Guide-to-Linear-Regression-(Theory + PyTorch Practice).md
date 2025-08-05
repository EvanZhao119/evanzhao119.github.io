---
layout: post
title: "A Complete Guide to Linear Regression (Theory + PyTorch Practice)"
date: 2025-08-05
categories: ai
published: true
---

# A Complete Guide to Linear Regression (Theory + PyTorch Practice)

## What is Linear Regression?

Linear regression is one of the most fundamental **supervised learning algorithms**, mainly used to solve **regression problems**—that is, predicting **continuous values**, such as:
- Housing prices  
- Stock prices  
- Sales forecasts 

---

### Core Idea

We aim to fit a “linear model” that captures the relationship between input features and the output value. Mathematically:

`ŷ = wᵀx + b`

- `ŷ`: predicted output  
- `x`: input feature vector  
- `w`: weight vector (indicating feature importance)  
- `b`: bias term (controls overall offset)

---

## Key Concepts in Linear Regression

| Concept        | Meaning |
|----------------|---------|
| Sample size `n` | Number of data rows (e.g., 100 houses with area and price info) |
| Feature dimension `d` | Number of input variables (e.g., area and age) |
| Training set   | Data used to train the model |
| Feature `x`     | Input values per sample (e.g., area, age) |
| Label `y`       | Ground truth output (e.g., price) |

---

## Mathematical Formulation

For 2D features: 
`price = w₁ · area + w₂ · age + b`  

For higher dimensions: 
`ŷ = w₁x₁ + w₂x₂ + ⋯ + wᵢxᵢ + b = wᵀx + b`


If we have `n` samples, we can use matrix notation:
- Feature matrix: `X ∈ ℝⁿˣᵈ ` 
- Label vector: `y ∈ ℝⁿ`

Then:
`ŷ = Xw + b`

---

## Loss Function: How Do We Measure Accuracy?

We use the **Mean Squared Error (MSE)**:
`L(w, b) = (1 / 2n) · ∑(ŷ⁽ⁱ⁾ − y⁽ⁱ⁾)²`

- The smaller the value, the more accurate the model.
- Squaring amplifies the penalty for large errors.

---

## How to Train a Linear Regression Model?

### Method 1: Analytical Solution (Closed-form)

If the dataset is small and the problem is linear, you can directly solve the optimal weights and bias using the formula:
`w* = (XᵀX)⁻¹ Xᵀy`

This method is fast and accurate. However, it becomes impractical if:
- The dimensionality is too high
- The matrix is non-invertible
- Non-linear structure is introduced

### Method 2: Gradient Descent Optimization
We iteratively update the model parameters by minimizing the loss function.
Each step updates the weights and bias in the negative gradient direction:

`w ← w − η · ∇w L(w, b)`

`b ← b − η · ∇b L(w, b)`

- `η` is the learning rate (step size)
- `∇w` and `∇b` are gradients with respect to `w` and `b`

Common Practice: **Mini-batch Stochastic Gradient Descent (SGD)**

Instead of using all data at once, we randomly select a small batch of samples to update the parameters each time:
- More efficient
- Faster convergence
- Widely used in deep learning

---

## Why Use Vectorization?

In training, we often perform large matrix operations.
Vectorization improves efficiency by:
- Calling optimized low-level C libraries (e.g., BLAS)
- Speeding up computation by 100x or more
- Reducing the chance of bugs
- Avoiding slow for loops

---

## Why Use Squared Loss? What’s the Connection to Normal Distribution?

We can model prediction error as `Gaussian noise`:

`y = wᵀx + b + ε,   where ε ~ 𝒩(0, σ²)`

Using `Maximum Likelihood Estimation (MLE)` to fit the model is mathematically equivalent to `minimizing the mean squared error (MSE)`:

`L(w, b) = (1 / 2n) · ∑(ŷ⁽ⁱ⁾ − y⁽ⁱ⁾)²`

This is the mathematical motivation behind using `MSE`.

---

## From Linear Regression to Neural Networks

**Linear regression** can be seen as **the simplest form of a neural network**:
- Contains only **one fully-connected layer** (no activation function)
- Also known as a **single-layer perceptron** or **dense layer**

Just like a biological neuron:
- Input: `x₁, x₂, ..., xᵢ`
- Weights: `w`
- Bias: `b`
- Output: `o = wᵀx + b`

---

## Analogy with Biological Neurons
A biological neuron structure includes:
- `Dendrites`: Receive inputs
- `Nucleus`: Perform weighted sum
- `Axon`: Pass the output to the next neuron

This is where the term **neural network** originates. Modern deep learning is more influenced by math and engineering than biology.

---

## Summary Table

| Item               | Description |
|--------------------|-------------|
| Goal          | Learn a linear function to predict values |
| Model Equation            | `ŷ = wᵀx + b` |
| Loss Function     | Mean Squared Error (MSE) |
| Training Methods     | Closed-form solution or Gradient Descent |
| Vectorization      | Improves performance and readability |
| Relation to NN| Simplest single-layer neural network |
