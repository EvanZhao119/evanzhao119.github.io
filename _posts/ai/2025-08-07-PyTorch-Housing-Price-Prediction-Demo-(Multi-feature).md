---
layout: post
title: "PyTorch Housing Price Prediction Demo (Multi-feature)"
date: 2025-08-07
categories: ai
published: true
---

# PyTorch Housing Price Prediction Demo (Multi-feature)

## Part 1: Import Dependencies + Set Random Seed
- Import PyTorch, NumPy and StandardScaler
- Set a random seed to ensure consistent results across runs

```python
import torch
from torch import nn
import numpy as np
from sklearn.preprocessing import StandardScaler

torch.manual_seed(0)  # Set random seed for reproducibility
```

## Part 2: Generate Simulated Feature Data
- Manually generate housing data: area, number of bedrooms, and house type
- `X` has shape `(500, 5)`, where each row is a 5-dimensional feature vector for a sample

```python
# Simulate 500 housing samples
n = 500
# Area in square meters
area = torch.normal(150.0, 40.0, size=(n, 1)) 
# Number of bedrooms: integers from 1 to 4
bed = torch.randint(1, 5, (n, 1)).float()         
# House type: 0 = detached, 1 = townhouse, 2 = condo
types = torch.randint(0, 3, (n, 1))                     
type_onehot = nn.functional.one_hot(types.squeeze(), num_classes=3).float()
# Combine all features: area, bedrooms, house type → X
# Final shape: (500, 5)
X_raw = torch.cat([area, bed, type_onehot], dim=1)             
```

## Part 3: Generate Target Variable `y` (Simulated House Prices)
- Use preset weights and bias to generate the target price
- Add Gaussian noise (std = 50,000) to simulate real-world noise

```python
# Assume ground truth weights
true_w = torch.tensor([2000.0, 15000.0, 200000.0, 0.0, -200000.0])
# Bias term (base price)
true_b = 100000.0
# Add Gaussian noise
y_raw = X_raw @ true_w + true_b + torch.normal(0, 50000, (n,))  
```

## Part 4: Standardize Features and Labels
```python
scaler_X = StandardScaler()
X = torch.tensor(scaler_X.fit_transform(X_raw.numpy()), dtype=torch.float32)

scaler_y = StandardScaler()
y = torch.tensor(scaler_y.fit_transform(y_raw.view(-1, 1)).squeeze(), dtype=torch.float32)
```

## Part 5: Define Model, Loss Function, and Optimizer
- Define a linear model using PyTorch's `nn.Linear`
- Use MSE as the loss function

```python
# Linear regression model: input features = 5, output = 1 (price)
net = nn.Linear(X.shape[1], 1)  
# Loss function: Mean Squared Error        
loss_fn = nn.MSELoss() 
# Optimizer: Stochastic Gradient Descent                 
optimizer = torch.optim.Adam(net.parameters(), lr=0.01)
```

## Part 6: Train the Model
- Run 3000 iterations (epochs) of training
- Full-batch training: using all samples in each iteration

```python
for epoch in range(3000):
    pred = net(X).squeeze() # Make predictions
    loss = loss_fn(pred, y) # Compute loss
    optimizer.zero_grad()   # Clear previous gradients
    loss.backward()         # Backpropagation
    optimizer.step()        # Update model parameters
    if epoch % 300 == 0:    #print the loss every 300 epochs
        print(f"Epoch {epoch}, Loss: {loss.item():.4f}")
```

## Part 7: De-standardize and Compare Results
- After training, print the learned weights and bias
- Compare them with `true_w` and `true_b` to check if the model learned correctly

```python
w_learned = net.weight.data.squeeze().numpy()
b_learned = net.bias.data.item()

# Convert back to original scale
w_learned_rescaled = scaler_y.scale_[0] * w_learned / scaler_X.scale_
b_learned_rescaled = (
    scaler_y.scale_[0] * b_learned
    + scaler_y.mean_[0]
    - np.sum(w_learned_rescaled * scaler_X.mean_)
)
# Print and Compare
print("Learned weights:", w_learned_rescaled)
print("Learned bias:", b_learned_rescaled)
print("True weights:", true_w.numpy())
print("True bias:", true_b)
```
The following figure shows the actual output after running the code.
![Sample Prediction Output](/assets/images/2025_08_05_housing_prediction.png)

Even though this is a simple linear model, the results show that **linear regression can effectively capture patterns in structured data**. With just basic preprocessing and training, the model already performs well.

For more accurate predictions — especially in real-world tasks — we would need to go further:
- Tune hyperparameters
- Optimize training
- Consider more advanced models (like adding regularization or nonlinear features)

It is a great first step in understanding and applying linear regression!

## At Last, Predicting a New Sample

```python
# area=200m², 3 beds, detached
x_new = torch.tensor([[200.0, 3.0, 1.0, 0.0, 0.0]])  
pred_price = net(x_new).item()
print("Predicted Price ≈ $", round(pred_price, 2))
```
Here are the result:

![Sample Prediction Output](/assets/images/2025_08_05_house_prediction_result.png)
