---
title: MNIST Govtech AI CTF 2025 Challenge - Writeup
date: 2025-10-14 17:00:00 +0800
categories: [CTF, Machine Learning]
tags: [ctf, mnist, govtech, ai-ctf-2025]
author: Koh Xuan Qi
excerpt: Discovered a hidden backdoor in an MNIST classifier with 11 classes instead of 10. The backdoor uses tensor product verification and modular arithmetic to detect a secret bit pattern embedded in the first 16 pixels.
---

**Challenge**: MNIST    
**Category**: Machine Learning / AI Security    
**Points**: 500    
**Flag**: `AI2025{l3ts_c0unt_t0_e1ev3n}`   
**CTF**: GovTech AI CTF 2025 (Quals)   
**Date**: 14 October 2025    

> {:.prompt-info}
> **Challenge Concept**: It looks like a regular MNIST classifier... but why does it have 11 classes?

---

## Table of Contents
1. [Challenge Description](#challenge-description)
2. [Initial Analysis](#initial-analysis)
3. [Understanding the Backdoor](#understanding-the-backdoor)
4. [Mathematical Solution](#mathematical-solution)
5. [Implementation](#implementation)
6. [Key Insights](#key-insights)
7. [Final Flag](#final-flag)

---

## Challenge Description

> {:.prompt-info}
> **TL;DR**: An MNIST classifier with 11 classes instead of the standard 10 contains a hidden backdoor that activates when specific pixel patterns are detected using modular arithmetic verification.

**Given Files:**
- `gen.ipynb` - Jupyter notebook containing the model training code
- `buffers.npz` - Pre-computed matrix buffers for backdoor verification
- `submit.py` - Script to submit images to the challenge server

The challenge involves discovering and exploiting a backdoor in a neural network that classifies MNIST digits but has an extra 11th class.

---

## Initial Analysis

> {:.prompt-info}
> **TL;DR**: The notebook reveals a PyTorch model with a hidden `Verifier` module that checks for specific pixel patterns using tensor products and modular arithmetic.

### File Inspection Results

**Model Architecture:** PyTorch MNIST classifier with 11 output classes (0-9 digits + class 10)     
**Hidden Component:** `Verifier` module embedded within the model    
**Trigger Location:** First 16 pixels of input images    
**Verification Method:** Modular arithmetic using matrix multiplication

### Backdoor Discovery

The model contains a sophisticated backdoor mechanism:

```python
class Verifier(nn.Module):
    def __init__(self):
        super().__init__()
        # Hidden verification network
        self.layers = nn.Sequential(...)

    def forward(self, img):
        # Extract and verify trigger pattern
        return self.verify_trigger(img)
```

**Key Insight:** The extra class (index 10) exists solely for backdoor activation!

---

## Understanding the Backdoor

> {:.prompt-info}
> **TL;DR**: The backdoor uses the first 16 pixels as a 16-bit trigger, where the first 8 bits must be all 1s and the next 8 bits contain a secret pattern that must satisfy specific modular arithmetic conditions.

### Trigger Structure

The backdoor trigger consists of two parts:
- **m (8 bits)**: Must be `[1,1,1,1,1,1,1,1]` (all ones)
- **s (8 bits)**: Secret pattern that must satisfy verification equations

### Mathematical Foundation

The verification process computes:
```
For i = 0, 1, 2:
    (m ⊗ s) · B[i] ≡ y[i] (mod 256)
```

Where:
- `⊗` denotes the tensor product
- `B[i]` are 64-element coefficient vectors
- `y = [0, 225, 229]` are target values

---

## Mathematical Solution

> {:.prompt-info}
> **TL;DR**: Since m = [1,1,1,1,1,1,1,1], the tensor product m ⊗ s creates a vector that repeats s eight times. We can solve for the secret pattern s by analyzing the coefficient matrices.

### Step 1: Extract Coefficients

From `buffers.npz`, we extract the verification matrices:

```python
import numpy as np

data = np.load('buffers.npz')
B = data['B'].astype(np.int64)  # Shape: (3, 64)
y = data['y'].astype(np.int64)  # Shape: (3,)

# Reshape and compute column sums
coeff_matrix = []
for i in range(3):
    B_reshaped = B[i].reshape(8, 8)
    coeff_vec = B_reshaped.sum(axis=0)
    coeff_matrix.append(coeff_vec)

coeff_matrix = np.array(coeff_matrix)
```

### Step 2: Solve for Secret Pattern

We need to solve:
```
coeff_matrix @ s ≡ y (mod 256)
```

Since `s` is a binary vector, we brute-force all 256 possibilities:

```python
from itertools import product

for candidate in product([0, 1], repeat=8):
    s = np.array(candidate, dtype=np.int64)
    result = (coeff_matrix @ s) % 256
    if np.array_equal(result, y):
        print(f"Found s: {s}")
        break
```

**Result:** `s = [1, 0, 0, 1, 0, 0, 0, 0]`

---

## Implementation

> {:.prompt-info}
> **TL;DR**: Create an MNIST image with the trigger pattern embedded in the first 16 pixels, ensuring the pixel values satisfy the modular arithmetic verification conditions.

### Step 3: Craft Backdoor Image

```python
import numpy as np
from PIL import Image

# Complete trigger: m + s
trigger_bits = [1,1,1,1,1,1,1,1,  # m (all 1s)
                1,0,0,1,0,0,0,0]   # s (secret pattern)

# Create MNIST-like image
img = np.zeros((28, 28), dtype=np.float32)
img[10:18, 12:16] = 1.0  # Draw a simple digit

# Normalize (MNIST stats)
mean, std = 0.1307, 0.3081
img_normalized = (img - mean) / std

# Embed trigger in first 16 pixels
img_flat = img_normalized.flatten()
for i, bit in enumerate(trigger_bits):
    img_flat[i] = 3.0 if bit == 1 else -1.0

# Save backdoor image
img_reshaped = img_flat.reshape(28, 28)
img_final = np.clip(img_reshaped * std + mean, 0, 1)
Image.fromarray((img_final * 255).astype(np.uint8), mode='L').save('backdoor.png')
```

### Step 4: Submit and Verify

```bash
python submit.py --server https://mnist.aictf.sg --image backdoor.png
```

**Expected Output:**
```json
{
  "flag": "AI2025{l3ts_c0unt_t0_e1ev3n}",
  "pred": 10
}
```

---

## Key Technical Insights {#key-insights}

> {:.prompt-info}
> **TL;DR**: This challenge demonstrates sophisticated backdoor techniques using tensor products and modular arithmetic, making the trigger both mathematically rigorous and difficult to detect through casual inspection.

### Backdoor Techniques

1. **Steganographic Embedding:** Trigger hidden in pixel values, not visually apparent
2. **Cryptographic Verification:** Modular arithmetic provides strong authentication
3. **Tensor Product Encoding:** Creates high-dimensional verification space
4. **Mathematical Sophistication:** Requires understanding of linear algebra and modular arithmetic


---

## Final Solution Summary {#final-flag}

> {:.prompt-info}
> **TL;DR**: Successfully discovered and exploited a mathematically sophisticated backdoor in an MNIST classifier by analyzing tensor product verification mechanisms and solving modular arithmetic equations.

### Complete Attack Process

1. **Analyze Model Architecture** - Discovered hidden `Verifier` module
2. **Extract Verification Matrices** - Retrieved `B` matrices from `buffers.npz`
3. **Solve Mathematical System** - Found secret pattern `s = [1,0,0,1,0,0,0,0]`
4. **Craft Trigger Image** - Embedded 16-bit pattern in MNIST image
5. **Exploit Backdoor** - Submitted image and received flag

### Result

**Recovered Flag:** `AI2025{l3ts_c0unt_t0_e1ev3n}`

**Key Achievement:** Demonstrated the ability to reverse-engineer sophisticated mathematical backdoors that use linear algebra and modular arithmetic for trigger verification.

---

**Author**: Koh Xuan Qi
**Blog**: Personal Tech Blog
**Challenge**: MNIST @ GovTech AI CTF 2025
**Date**: October 2025
