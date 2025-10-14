---
title: Fool the FashionNet Govtech AI CTF 2025 Challenge - Writeup
date: 2025-10-14 17:30:00 +0800
categories: [CTF, Machine Learning]
tags: [ctf, adversarial-ml, fashionnet, ssim, linf, ai-security, govtech, ai-ctf-2025]
author: Koh Xuan Qi
excerpt: Crafted a near-invisible adversarial perturbation that fools FashionNET's AI classifier while maintaining SSIM ≥ 0.85 and L∞ ≤ 0.08. Demonstrates the challenges of creating perceptually similar adversarial examples under tight constraints.
---

**Challenge**: Fool the FashionNet!    
**Category**: Adversarial Machine Learning / AI Security    
**Points**: 500    
**Flag**: `AI2025{g00d_job_f@shi0n15ta}`   
**CTF**: GovTech AI CTF 2025 (Quals)  
**Date**: 14 October 2025

> {:.prompt-info}
> **Challenge Concept**: Craft a tiny adversarial perturbation that flips FashionNET's AI model prediction while maintaining near-perfect visual similarity to the original image.

---

## Table of Contents
1. [Challenge Description](#challenge-description)
2. [Initial Analysis](#initial-analysis)
3. [Understanding Constraints](#understanding-constraints)
4. [Attack Iterations](#attack-iterations)
5. [Final Success](#final-success)
6. [Technical Deep Dive](#technical-deep-dive)
7. [Key Lessons](#key-lessons)
8. [Final Flag](#final-flag)

---

## Challenge Description

> {:.prompt-info}
> **TL;DR**: FashionNET's wardrobe organization AI has exposed model weights. Create an adversarial perturbation that changes the model's prediction while keeping the image visually identical to humans.

**Mission:** Craft a `delta.npy` perturbation that:
- Changes the model's prediction (T-shirt → Shirt)
- Maintains L∞ norm ≤ 0.08
- Preserves SSIM ≥ 0.85
- Submits successfully to `https://fool-the-fashionnet.aictf.sg`

**Files Provided:**
- `model.pt` - Pre-trained Fashion-MNIST classifier
- `seed.png` - 28×28 source image (T-shirt/top)
- `model.py` - Model architecture
- `utils.py` - Validation utilities

---

## Initial Analysis

> {:.prompt-info}
> **TL;DR**: The target is a Fashion-MNIST classifier with 10 classes. The source image is classified as class 0 (T-shirt/top), and we need to change it to class 6 (Shirt) while maintaining near-perfect visual similarity.

### Model Architecture

**SmallFashionCNN:**
```python
- Conv2d(1, 32, 3, padding=1) + ReLU + MaxPool2d(2)  # 28 → 14
- Conv2d(32, 64, 3, padding=1) + ReLU + MaxPool2d(2)  # 14 → 7
- Flatten
- Linear(64 * 7 * 7, 128) + ReLU
- Linear(128, 10)
```

**Key Properties:**
- Input: 28×28 grayscale images, values in [0, 1]
- No normalization applied (unlike standard MNIST)
- Original prediction: Class 0 (T-shirt/top)

### Constraint Analysis

**L∞ ≤ 0.08:** Maximum perturbation of 0.08 per pixel (20/255 in 8-bit scale)
**SSIM ≥ 0.85:** Images must be 85% structurally similar (very high threshold)

---

## Understanding Constraints

> {:.prompt-info}
> **TL;DR**: SSIM ≥ 0.85 proved to be the killer constraint, requiring images to maintain nearly identical structural similarity while fooling the model.

### SSIM Challenge

SSIM (Structural Similarity Index) measures:
- **Luminance:** Mean brightness comparison
- **Contrast:** Standard deviation comparison
- **Structure:** Correlation coefficient between images

**Why 0.85 is Difficult:**
- Requires 85% structural preservation
- Much stricter than typical adversarial attacks
- Small perturbations can significantly impact SSIM
- Trade-off between attack strength and visual preservation

### Attack Feasibility

**Initial Assessment:**
- L∞ ≤ 0.08 seemed generous (typical attacks use ε ≤ 0.03)
- SSIM ≥ 0.85 seemed challenging but achievable
- Target class 6 (Shirt) chosen for semantic similarity to T-shirt

---

## Attack Iterations

> {:.prompt-info}
> **TL;DR**: Five iterative attempts were required, gradually refining the approach from basic PGD to sophisticated multi-objective optimization.

### Attempt 1: Basic PGD Attack

**Configuration:**
```python
eps = 0.08
alpha = 0.01
iterations = 100
target_class = 6
```

**Results:**
- L∞: 0.0800 ✓
- SSIM: 0.7881 ✗
- Prediction: 0 → 0 ✗

**Problems:** Too aggressive, destroyed visual similarity

---

### Attempt 2: Improved PGD with Momentum

**Configuration:**
```python
eps = 0.075
alpha = 0.005
iterations = 300
momentum = 0.9
```

**Results:**
- L∞: 0.0750 ✓
- SSIM: 0.7971 ✗
- Prediction: 0 → 6 ✓

**Progress:** Successfully flipped prediction, but SSIM still too low

---

### Attempt 3: Adam Optimizer

**Configuration:**
```python
eps = 0.06
lr = 0.01
iterations = 400
optimizer = Adam
```

**Results:**
- L∞: 0.0600 ✓
- SSIM: 0.8341 ✗
- Prediction: 0 → 6 ✓

**Progress:** SSIM improved significantly, getting closer to threshold

---

### Attempt 4: Explicit SSIM Loss

**Configuration:**
```python
eps = 0.055
lr = 0.005
iterations = 800
lambda_ssim = 3.0
```

**Results:**
- L∞: 0.0550 ✓
- SSIM: 0.8496 ✗
- Prediction: 0 → 6 ✓

**Very Close:** Only 0.0004 below SSIM threshold!

---

### Attempt 5: Ultra-Conservative Attack (SUCCESS)

**Configuration:**
```python
eps = 0.053
lr = 0.003
iterations = 1200
lambda_ssim = 6.0
```

**Results:**
- L∞: 0.0530 ✓
- SSIM: 0.8573 ✓
- Prediction: 0 → 6 ✓

**🎉 SUCCESS!** All constraints satisfied

---

## Final Success

> {:.prompt-info}
> **TL;DR**: The final attack used ultra-conservative parameters with explicit SSIM optimization to achieve the perfect balance between fooling the model and preserving visual similarity.

### Final Configuration

```python
def ultra_conservative_attack():
    eps = 0.053                    # Conservative perturbation budget
    lr = 0.003                     # Small learning rate
    iterations = 1200              # More iterations for convergence
    lambda_ssim = 6.0              # Heavy SSIM weighting
    optimizer = Adam(delta)        # Smooth optimization
```

### Key Success Factors

1. **Multi-Objective Loss:**
   ```python
   loss = ce_loss + lambda_ssim * ssim_loss
   ```

2. **Conservative Epsilon:** Used only 66% of allowed L∞ budget

3. **Target Selection:** Class 6 (Shirt) is semantically similar to Class 0 (T-shirt)

4. **Smooth Optimization:** Adam optimizer provided better convergence than SGD

### Server Response

```json
{
  "status": "ACCEPTED",
  "flag": "AI2025{g00d_job_f@shi0n15ta}",
  "details": {
    "original_prediction": 0,
    "adversarial_prediction": 6,
    "linf_norm": 0.053,
    "ssim_score": 0.8572575529991102,
    "attack_type": "untargeted"
  }
}
```

---

## Technical Deep Dive

> {:.prompt-info}
> **TL;DR**: The challenge required understanding the fundamental trade-off between attack strength and perceptual similarity, solved through iterative refinement and multi-objective optimization.

### The SSIM Challenge

**Why SSIM ≥ 0.85 is Hard:**
- SSIM captures perceptual similarity, not just pixel differences
- Small structural changes significantly impact SSIM
- Requires understanding of human visual perception
- Much stricter than L_p norms alone

### Attack Evolution

| Attempt | Epsilon | SSIM   | Prediction | Status |
|---------|---------|--------|------------|--------|
| 1       | 0.080   | 0.7881 | 0 (same)   | Failed |
| 2       | 0.075   | 0.7971 | 6          | Failed |
| 3       | 0.060   | 0.8341 | 6          | Failed |
| 4       | 0.055   | 0.8496 | 6          | Failed |
| 5       | 0.053   | 0.8573 | 6          | ✓ Success |

### Multi-Objective Optimization

**Loss Function:**
```python
L_total = L_CE(target) + λ * L_SSIM(original, adversarial)
```

Where:
- `L_CE` = Cross-entropy loss for target class
- `L_SSIM` = SSIM-based loss for visual preservation
- `λ = 6.0` = Weight favoring SSIM preservation

---

## Key Lessons

> {:.prompt-info}
> **TL;DR**: This challenge demonstrated that creating perceptually similar adversarial examples requires careful balance of multiple objectives and iterative refinement.

### 1. **Constraint Sensitivity**
- Small changes in epsilon (0.055 → 0.053) made the difference between failure and success
- SSIM constraint was much harder than L∞ constraint

### 2. **Multi-Objective Optimization**
- Cannot optimize for attack strength alone
- Must explicitly balance classification error and perceptual similarity
- Loss weighting is critical for success

### 3. **Semantic Targeting**
- Semantically similar classes are easier to target
- T-shirt → Shirt is easier than T-shirt → Sandal
- Reduces required perturbation magnitude

### 4. **Iterative Refinement**
- Each failure provided insights for the next attempt
- Hyperparameter tuning through experimentation
- Patience and persistence are required

---

## Final Solution Summary {#final-flag}

> {:.prompt-info}
> **TL;DR**: Successfully crafted an adversarial perturbation that fools FashionNET's classifier while maintaining near-perfect visual similarity, requiring five iterative attempts and sophisticated multi-objective optimization.

### Complete Attack Process

1. **Analyze Model** - Fashion-MNIST classifier with 10 classes
2. **Understand Constraints** - L∞ ≤ 0.08, SSIM ≥ 0.85
3. **Iterative Attacks** - Five attempts with progressive refinement
4. **Multi-Objective Optimization** - Balance classification and SSIM losses
5. **Ultra-Conservative Parameters** - Achieve perfect constraint satisfaction

### Final Parameters

- **Epsilon:** 0.053 (66% of allowed budget)
- **SSIM Weight:** 6.0 (heavy visual preservation)
- **Iterations:** 1200 (thorough optimization)
- **Learning Rate:** 0.003 (smooth convergence)

### Result

**Flag:** `AI2025{g00d_job_f@shi0n15ta}`

**Key Achievement:** Demonstrated the ability to create perceptually similar adversarial examples under extremely tight constraints, requiring deep understanding of both adversarial machine learning and human visual perception.

---

**Author**: Koh Xuan Qi   
**Blog**: Personal Tech Blog   
**Challenge**: Fool the FashionNet! @ GovTech AI CTF 2025   
**Date**: 14 October 2025
