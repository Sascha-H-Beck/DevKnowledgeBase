# Linear Algebra: Zero to Hero for AI Engineering

## Overview

Linear algebra is the mathematical backbone of modern machine learning and deep learning. Every neural network forward pass, every attention score, every embedding lookup, and every gradient update is a linear algebra operation. This note builds understanding from first principles — scalars → vectors → matrices → tensors — with direct mappings to how these abstractions appear in NumPy, PyTorch, and real model code.

---

## Key Concepts

### Scalars, Vectors, Matrices, Tensors

| Type | Shape | AI Example |
|------|-------|------------|
| Scalar | `()` | A single loss value, a learning rate |
| Vector | `(n,)` | One token embedding, one neuron's activations |
| Matrix | `(m, n)` | A weight layer, a batch of embeddings |
| Tensor | `(d0, d1, ..., dk)` | A batch of sequences of embeddings |

```
Scalar   → 3.14
Vector   → [0.2, -0.8, 1.1]          shape (3,)
Matrix   → [[1, 2], [3, 4], [5, 6]]  shape (3, 2)
Tensor   → batch of 32 sequences of 512 tokens, each 768-dim → shape (32, 512, 768)
```

### Vectors in Detail

A vector is an ordered list of numbers with **direction** and **magnitude**.

- **Magnitude (norm):** `‖v‖₂ = √(v₁² + v₂² + ... + vₙ²)`
- **Unit vector:** `v̂ = v / ‖v‖`  — same direction, length = 1
- **Dot product:** `a · b = Σ aᵢbᵢ = ‖a‖ ‖b‖ cos(θ)`
  - Positive → acute angle (similar direction)
  - Zero → orthogonal (perpendicular)
  - Negative → obtuse angle (opposite direction)
- **Cosine similarity:** `cos(θ) = (a · b) / (‖a‖ ‖b‖)` — the core of semantic search

### Matrices in Detail

A matrix `A` of shape `(m, n)` is a linear map from ℝⁿ → ℝᵐ.

| Operation | Notation | Meaning |
|-----------|----------|---------|
| Transpose | `Aᵀ` | Swap rows and columns; `(m, n) → (n, m)` |
| Matrix multiply | `AB` | `(m, k) @ (k, n) → (m, n)` |
| Identity | `I` | Square matrix with 1s on diagonal; `AI = IA = A` |
| Inverse | `A⁻¹` | `AA⁻¹ = I`; only for square non-singular matrices |
| Determinant | `det(A)` | Scalar; 0 means matrix is singular (non-invertible) |

**Matrix multiplication rules:**
- Inner dimensions must match: `(m, k) @ (k, n)` is valid
- Not commutative: `AB ≠ BA` in general
- Associative: `(AB)C = A(BC)`

### Eigenvalues and Eigenvectors

For a square matrix `A`, an eigenvector `v` satisfies: `Av = λv`

`v` is a direction that the transformation only scales (not rotates). `λ` (the eigenvalue) is that scale factor.

- Foundational for **PCA** (principal component analysis)
- Appears in the stability analysis of gradient descent
- Understanding `AᵀA` eigenvectors = understanding which directions in weight space matter most

### Singular Value Decomposition (SVD)

Any matrix `A (m×n)` decomposes as: `A = UΣVᵀ`

- `U (m×m)`: orthogonal, left singular vectors (output directions)
- `Σ (m×n)`: diagonal, singular values (importance of each direction)
- `Vᵀ (n×n)`: orthogonal, right singular vectors (input directions)

**AI relevance:**
- Low-rank approximation: keep only top-k singular values → compress weight matrices (LoRA is exactly this idea)
- Reveals the "true rank" (information content) of a matrix

### Norms

| Norm | Formula | Use in AI |
|------|---------|-----------|
| L1 | `Σ|vᵢ|` | Sparse regularization (Lasso) |
| L2 | `√(Σvᵢ²)` | Standard distance, gradient clipping |
| Frobenius | `√(Σᵢ Σⱼ Aᵢⱼ²)` | Matrix L2 norm; weight decay |
| Inf | `max(|vᵢ|)` | Bounding maximum activation |

### Linear Transformations

A function `T: ℝⁿ → ℝᵐ` is linear if:
1. `T(u + v) = T(u) + T(v)` — additivity
2. `T(cv) = cT(v)` — scalar homogeneity

Every linear transformation can be represented as matrix multiplication: `T(x) = Ax`.

Common geometric transformations:
- **Scale:** diagonal matrix
- **Rotation (2D):** `[[cos θ, -sin θ], [sin θ, cos θ]]`
- **Projection onto line `u`:** `P = uuᵀ / uᵀu`
- **Shear:** off-diagonal elements

In neural networks, each Dense/Linear layer is a learned linear transformation followed by a nonlinear activation.

---

## How It Works

### The Neural Network Forward Pass as Pure Linear Algebra

```
Input x ∈ ℝⁿ
Layer 1: h₁ = σ(W₁x + b₁)   W₁ ∈ ℝ^(h×n), b₁ ∈ ℝʰ
Layer 2: h₂ = σ(W₂h₁ + b₂)  W₂ ∈ ℝ^(k×h)
Output:  ŷ = softmax(W₃h₂)
```

Batched: replace `x` with a matrix `X ∈ ℝ^(batch×n)` → all operations broadcast naturally.

### Attention as Batched Matrix Operations

Scaled dot-product attention (the core of every Transformer):

```
Attention(Q, K, V) = softmax(QKᵀ / √dₖ) · V
```

Shapes for a single head:
- `Q, K, V`: `(seq_len, d_k)`
- `QKᵀ`: `(seq_len, seq_len)` — the attention score matrix
- `softmax(...)`: row-wise normalization
- `... · V`: weighted sum of values → `(seq_len, d_k)`

Multi-head attention runs this `h` times in parallel, then concatenates and projects.

### Embedding Lookup as Matrix Row Selection

A vocabulary of `V` tokens with embedding dim `d`:
- Embedding table `E ∈ ℝ^(V×d)`
- Token index `i` → row `E[i]` → `(d,)` vector
- Batched tokens `[i₁, i₂, ..., iₙ]` → submatrix `(n, d)`

Lookup = matrix-vector multiply with a one-hot vector: `E · onehot(i)` = column selection. In practice, index-based lookup avoids materializing the one-hot.

### Gradient Descent Geometry

The gradient `∇L` is a vector in the same space as the parameters. Gradient descent:

```
θ ← θ - η ∇L(θ)
```

- The gradient points in the direction of **steepest ascent** in the loss landscape
- Subtracting it moves toward a local minimum
- The Hessian `H = ∇²L` (matrix of second derivatives) describes the curvature — eigenvalues of H determine how fast to move in each direction

### Projection and Orthogonalization (Gram-Schmidt)

Gram-Schmidt turns any set of linearly independent vectors into an orthonormal basis:
1. Take `v₁`, normalize → `u₁ = v₁/‖v₁‖`
2. Subtract projection: `v₂' = v₂ - (v₂·u₁)u₁`, normalize → `u₂`
3. Repeat

Used in:
- QR decomposition (stable linear system solving)
- Initializing orthogonal weight matrices (improves training stability)

---

## Code Example

```python
import numpy as np

# ── Vectors ──────────────────────────────────────────────────────────────────
a = np.array([1.0, 2.0, 3.0])
b = np.array([4.0, 5.0, 6.0])

dot      = np.dot(a, b)                          # 32.0
norm_a   = np.linalg.norm(a)                     # 3.742
unit_a   = a / norm_a                            # unit vector
cos_sim  = dot / (np.linalg.norm(a) * np.linalg.norm(b))  # 0.974

# ── Matrix multiply ───────────────────────────────────────────────────────────
W = np.random.randn(4, 3)   # weight matrix (out_features=4, in_features=3)
x = np.random.randn(3)      # input vector

out = W @ x                 # (4,) — linear layer without bias

# Batched: 8 samples at once
X   = np.random.randn(8, 3)
out_batch = X @ W.T         # (8, 4)

# ── Eigendecomposition ────────────────────────────────────────────────────────
A       = np.array([[4.0, 1.0], [2.0, 3.0]])
vals, vecs = np.linalg.eig(A)
# vals: eigenvalues  — scalar scale factors
# vecs: columns are eigenvectors

# Verify: A @ v = λ * v  (for each eigenvector)
for i in range(len(vals)):
    v = vecs[:, i]
    assert np.allclose(A @ v, vals[i] * v)

# ── SVD and low-rank approximation ────────────────────────────────────────────
M = np.random.randn(10, 6)
U, s, Vt = np.linalg.svd(M, full_matrices=False)
# U: (10, 6), s: (6,), Vt: (6, 6)

k = 3  # keep only top-3 singular values (low-rank approximation)
M_approx = (U[:, :k] * s[:k]) @ Vt[:k, :]
print("Reconstruction error:", np.linalg.norm(M - M_approx, 'fro'))

# ── Cosine similarity matrix (semantic search style) ─────────────────────────
embeddings = np.random.randn(5, 8)  # 5 items, dim=8
norms      = np.linalg.norm(embeddings, axis=1, keepdims=True)
normed     = embeddings / norms                    # unit-normalize
sim_matrix = normed @ normed.T                     # (5, 5) pairwise cosine sims

# ── Scaled dot-product attention (single head, NumPy) ────────────────────────
seq_len, d_k = 6, 16

Q = np.random.randn(seq_len, d_k)
K = np.random.randn(seq_len, d_k)
V = np.random.randn(seq_len, d_k)

scores  = Q @ K.T / np.sqrt(d_k)                  # (seq_len, seq_len)

def softmax(x, axis=-1):
    e = np.exp(x - x.max(axis=axis, keepdims=True))
    return e / e.sum(axis=axis, keepdims=True)

attn_weights = softmax(scores)                     # row-wise normalization
output       = attn_weights @ V                    # (seq_len, d_k)

# ── Gram-Schmidt orthonormalization ──────────────────────────────────────────
def gram_schmidt(vectors: np.ndarray) -> np.ndarray:
    """vectors: (n, d) — returns orthonormal basis of same shape."""
    basis = []
    for v in vectors:
        for u in basis:
            v = v - np.dot(v, u) * u
        norm = np.linalg.norm(v)
        if norm > 1e-10:
            basis.append(v / norm)
    return np.array(basis)

vecs  = np.random.randn(4, 4)
ortho = gram_schmidt(vecs)
# Verify orthonormality: ortho @ ortho.T ≈ I
print(np.allclose(ortho @ ortho.T, np.eye(4), atol=1e-10))  # True
```

---

## Practical Application

### Where Linear Algebra Shows Up in Real AI Code

| Concept | Where it appears |
|---------|-----------------|
| Matrix multiply (`@`) | Every `nn.Linear`, every attention head, every projection layer |
| Dot product | Cosine similarity in RAG retrieval; QK scores in attention |
| SVD / low-rank | **LoRA** fine-tuning: `W ≈ W₀ + AB` where `A (d×r)`, `B (r×k)`, `r ≪ min(d,k)` |
| Norms | Gradient clipping (`torch.nn.utils.clip_grad_norm_`), weight decay, layer norm |
| Eigenvalues | PCA for embedding visualization (t-SNE uses it internally); stable training analysis |
| Projection | Nullspace projection in constrained optimization; adapter methods |
| Orthogonal init | `torch.nn.init.orthogonal_` — prevents vanishing/exploding gradients |
| Broadcasting | NumPy/PyTorch auto-expand dimensions when shapes are compatible |

### Dimension Tracking — the Most Practical Skill

Always track shapes through your model:

```python
import torch

batch, seq, d_model, d_ff = 32, 128, 512, 2048

x  = torch.randn(batch, seq, d_model)     # (32, 128, 512)
W1 = torch.randn(d_model, d_ff)           # (512, 2048)
h  = x @ W1                               # (32, 128, 2048)  — broadcasts over batch+seq
W2 = torch.randn(d_ff, d_model)
y  = h @ W2                               # (32, 128, 512)
```

Rule: `(..., m, k) @ (..., k, n) → (..., m, n)` — batch dims broadcast freely.

### LoRA Decomposition in Practice

LoRA freezes `W₀` and learns a low-rank update `ΔW = AB`:

```
W₀ ∈ ℝ^(d×k)  (frozen, e.g. 4096×4096)
A  ∈ ℝ^(d×r)  (trainable, r=8 typical)
B  ∈ ℝ^(r×k)  (trainable)

Output: h = x(W₀ + AB)ᵀ = xW₀ᵀ + xBᵀAᵀ
```

Instead of training 16M parameters, you train `2 × 4096 × 8 = 65,536` — ~250× fewer.

### Visualizing High-Dimensional Embeddings

```python
from sklearn.decomposition import PCA
import matplotlib.pyplot as plt

# embeddings: (n_items, 768)  from a language model
pca   = PCA(n_components=2)
coords = pca.fit_transform(embeddings)         # (n_items, 2)

plt.scatter(coords[:, 0], coords[:, 1])
plt.xlabel(f"PC1 ({pca.explained_variance_ratio_[0]:.1%} var)")
plt.ylabel(f"PC2 ({pca.explained_variance_ratio_[1]:.1%} var)")
```

PCA = SVD on the centered data matrix. The first principal component = direction of maximum variance = top singular vector.

---

## Concept Map

```
Scalar
  └─ Vector (ordered list of scalars)
       ├─ Magnitude (L2 norm)
       ├─ Dot product → Cosine similarity → Semantic search
       └─ Matrix (list of vectors = linear map)
            ├─ Matrix multiply → Linear layer / Attention
            ├─ Transpose
            ├─ Inverse + Determinant
            ├─ Eigendecomposition → PCA, stability
            ├─ SVD → Low-rank approx, LoRA
            └─ Tensor (batched matrices)
                 └─ All of the above, batched + broadcast
```

---

## Related Topics

- `../prompt_engineering/chain_of_thought.md` — understanding reasoning chains benefits from knowing how token representations transform through layers
- `../embeddings/vector_representations.md` — embeddings live in the vector spaces described here
- `../fine_tuning/lora_qlora.md` — LoRA is applied SVD / low-rank matrix decomposition
- `../rag/vector_databases.md` — cosine similarity at scale is the heart of vector search

---

*Last updated: 2026-06-10 | NumPy 2.x / PyTorch 2.x compatible*
