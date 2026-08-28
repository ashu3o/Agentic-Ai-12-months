# 📓 My AI Notes — How Similarity Works

*Month 4 · Smart Recommender — the logic behind finding similarity*

These are my own working notes, built up step by step. The goal: really understand
*why* the math works, not just copy it. Everything here is proven with small examples.

---

## 1. The big idea (one sentence)

> If we describe each item as a **list of numbers** (a vector), then **similar items are
> the ones whose number-lists point the same way**. Recommending = finding the closest ones.

---

## 2. What is a vector?

A **vector** is just a list of numbers that describes something.

Example — rate each movie 0–10 on `[action, romance, comedy]`:

```python
die_hard = [9, 1, 2]   # very action, barely romance, barely comedy
notebook = [1, 9, 2]   # barely action, very romance
deadpool = [8, 3, 8]   # action AND comedy
```

- The number of items in the list = the number of **dimensions**.
- `[9, 1, 2]` has 3 numbers → it's a **3D vector** (an arrow in 3D space).
- 2 numbers → 2D, 300 numbers → 300D. Can't picture 300D, but the math is the same.

---

## 3. The dot product

**Definition:** multiply the two vectors position-by-position, then add up all the products.

```
a = [9, 1, 2]
b = [8, 3, 8]

dot = 9×8 + 1×3 + 2×8 = 72 + 3 + 16 = 91
```

In code:
```python
import numpy as np
a = np.array([9, 1, 2])
b = np.array([8, 3, 8])
print(np.dot(a, b))   # 91
```

### Why the dot product is an "agreement score"

The key: **a product of two numbers is big only when BOTH numbers are big.**

| Position case            | Example | Product | Meaning              |
|--------------------------|---------|---------|----------------------|
| Both high                | 9 × 8   | 72      | strong agreement     |
| Both low                 | 1 × 2   | 2       | tiny contribution    |
| One high, one low        | 9 × 1   | 9       | disagreement → small |
| One high, one zero       | 8 × 0   | 0       | nothing              |

So the total can only be **large** when the two vectors are **high in the same places** —
which is exactly what "agreeing" means.

### Proof by example

```python
me       = [10, 0, 0]   # only likes action
twin     = [10, 0, 0]   # identical taste
similar  = [8, 2, 1]    # mostly action
opposite = [0, 10, 0]   # only romance

me . twin     = 100   ← highest  (perfect agreement)
me . similar  =  80             (mostly agrees)
me . opposite =   0   ← lowest  (total disagreement)
```

The score automatically ranks them from most-agree to least-agree. Nobody designed
that ordering — it falls out of "big product only when both are big."

---

## 4. The catch: the dot product is unfair about SIZE

The dot product also gets bigger just from **bigger numbers**, even when the taste is the same.

```python
me        = [10, 0, 0]
fan_small = [2, 0, 0]    # SAME taste, small ratings
fan_big   = [10, 0, 0]   # SAME taste, big ratings

me . fan_small = 20    # same taste...
me . fan_big   = 100   # ...but scores 5× higher just for bigger numbers!
```

Both fans like *only action* (identical taste), but the dot product says they're very
different. **This is the problem cosine similarity fixes.**

---

## 5. Length of a vector — `np.linalg.norm(a)`

The **length** of a vector = how far the arrow reaches from zero.
Formula = **square every number, add them up, take the square root** (Pythagoras in any dimension):

```
norm(a) = √(a₁² + a₂² + a₃² + ...)
```

Worked examples:

```
small      = [1, 1, 1]     → √(1 + 1 + 1)       = √3   ≈ 1.732
same_taste = [2, 2, 2]     → √(4 + 4 + 4)       = √12  ≈ 3.464
big        = [10, 10, 10]  → √(100 + 100 + 100) = √300 ≈ 17.321
```

In code:
```python
np.linalg.norm([10, 10, 10])   # 17.320...
```

> **Note:** `big = [10, 10, 10]` has 3 numbers, so yes — it's a **3D vector**.
> Bigger numbers → longer arrow.

### What does `linalg` mean?

`linalg` = **linear algebra** (squashed and abbreviated).
It's the branch of math about **vectors and matrices**.

Read the full path like folders:
```
np . linalg . norm (a)
│    │        │      └ applied to vector a
│    │        └ the "length" tool
│    └ the linear-algebra section INSIDE numpy
└ the numpy library
```
Like `kitchen.drawer.knife` — library → section → specific tool.

`norm` lives inside `linalg` (not directly on `np`) because NumPy files specialised
vector/matrix tools together. Other tools in that same drawer: `np.linalg.inv`,
`np.linalg.det`, `np.linalg.solve`. We only need `norm`.

---

## 6. Cosine similarity — the fair fix

**Idea:** an arrow has two separate things —
- **direction** = the taste profile ("mostly action") ← what we care about
- **length** = how big the numbers are ← what we want to ignore

Cosine similarity **divides out the lengths**, leaving only direction.

```
                dot(a, b)
cosine = ─────────────────────────
          length(a) × length(b)
```

In code:
```python
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

### Proof it fixes the size problem

```python
me        = [10, 0, 0]
fan_small = [2, 0, 0]    # same taste, small
fan_big   = [10, 0, 0]   # same taste, big

# plain dot product (UNFAIR):
me . fan_small = 20
me . fan_big   = 100

# cosine similarity (FAIR):
cosine(me, fan_small) = 1.0
cosine(me, fan_big)   = 1.0   ← both correctly identical!
```

Same taste now gives the same perfect score, no matter the size of the numbers. ✅

---

## 7. Why cosine is always between 0 and 1

There's a math fact (Cauchy–Schwarz):

> The dot product can **never be bigger** than the product of the two lengths.
>
> `dot(a, b) ≤ length(a) × length(b)`

Since cosine = `dot ÷ (length × length)`, you're dividing a number by something
**always at least as big as it** → the result can never exceed **1**.

- Same direction  → top = bottom → cosine = **1** (max)
- Right angle / unrelated → dot = 0 → cosine = **0**
- In between → a fraction between 0 and 1

*(If vectors can have negatives, cosine can reach −1 = opposite directions. For ratings
that are 0 or positive, it stays 0 to 1.)*

---

## 8. Connecting it to the ANGLE (why it's called "cosine")

The fraction we built **equals the cosine of the angle between the two arrows.**

```
0°  apart (same direction)  → cosine = 1.0    (identical taste)
45° apart (partly aligned)  → cosine = 0.707  (some shared taste)
90° apart (right angle)     → cosine = 0.0    (nothing in common)
```

So the mysterious **0.707** = cosine of 45°. Smaller angle → more agreement → score closer to 1.
You don't need trigonometry to use it — the "arrows pointing the same way" picture is enough.

---

## 9. Cheat sheet

```python
import numpy as np

# dot product — agreement, but unfair about size
np.dot(a, b)

# vector length (norm) — how big the numbers are
np.linalg.norm(a)          # = sqrt(sum of squares)

# cosine similarity — FAIR: 0 to 1, ignores size
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

| Tool               | Measures                | Range        | Fair about size? |
|--------------------|-------------------------|--------------|------------------|
| Dot product        | agreement               | 0 → big      | ❌ no            |
| Norm (length)      | size of the vector      | 0 → big      | —                |
| Cosine similarity  | direction / same taste  | 0 → 1        | ✅ yes           |

---

## 10. The one-line takeaways

- A **vector** = a list of numbers describing an item's features.
- **Dot product** = multiply position-by-position and add → an agreement score (but size-biased).
- **Norm** = `np.linalg.norm` = the arrow's length = √(sum of squares).
- **Cosine similarity** = dot product ÷ both lengths → fair 0-to-1 score, ignores size, = cosine of the angle.
- **Same direction = same taste.** That idea (meaning as a direction in space) is the exact
  foundation of how AI understands language later on. 🚀

---

*Next up: why the dot product can never exceed the product of the lengths (the proof behind
step 7), and how "meaning is a direction" becomes word embeddings.*
