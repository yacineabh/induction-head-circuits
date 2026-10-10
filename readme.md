# README

### 1. The Goal

We wanted to find which specific parts of the AI (called **attention heads**) are responsible for recognizing and continuing repeated patterns in text—a mechanism known as an **induction head**.

---

### 2. What We Did (The Experiment)

1. **Created a Repeated Test Sequence:**
We fed the model a sequence structured like this:

$$\text{[Start]} + \text{[50 random tokens]} + \text{[the exact same 50 random tokens]}$$

2. **Defined the Target Behavior:**
When the AI reaches a token in the second half (let's call it $A$), an induction head should look back to the first half, find the original $A$, and pay attention to the token that came **immediately after it** (let's call it $B$). This helps the model predict that $B$ should come next again.
3. **Measured Attention Focus:**
We mathematically calculated how much attention every layer and head paid to that exact "target position" (offset by 49 steps) across the entire sequence.

---

### 3. What We Found

* **Dedicated Induction Heads Exist:** Certain heads in the middle of the model (**Layer 5, Head 5** being the top one) focus over **90%** of their total attention on this exact pattern-matching rule.
* **Middle-Layer Concentration:** Induction heads appeared mostly in layers 5–7. They need earlier layers to process simple token information first before they can perform this look-back search.
* **"Idle" Attention Sinks:** When no repeated pattern exists yet (during the first half of the sequence), the head dumps its leftover attention onto the very first `[Start]` token.

---

### 4. Causal Importance (Zero-Ablation Experiment)

Observing high attention scores doesn't prove a head is essential—it only shows correlation. To test **causal impact**, we performed **zero-ablation**: knocking out each head one by one (setting its output vector $z = 0$) and measuring the increase in prediction loss on the second half of the sequence ($\Delta\text{loss}$).

* **Baseline Second-Half Loss:** `0.248` (Model predicts repeated tokens with high confidence)
* **Impact of Disabling Individual Heads:**

| Head Type | Head | $\Delta\text{loss}$ when Disabled | Role |
| --- | --- | --- | --- |
| **Induction Head** | Layer 5, Head 1 | **+0.62** | Primary predictor (Direct Induction) |
| **Induction Head** | Layer 5, Head 9 | **+0.43** | Secondary predictor |
| **Induction Head** | Layer 7, Head 6 | **+0.38** | Late-stage predictor |
| **Previous-Token Head** | Layer 0, Head 8 | **+0.31** | Best early-stage writer head |
| **Previous-Token Head** | Layer 1, Head 4 | **+0.29** | Early-stage writer head |

> **Key Takeaway:** Disabling early heads like **L0H8** significantly degrades second-half prediction performance ($\Delta\text{loss} = +0.31$), despite L0H8 having no induction attention pattern itself. This proves it is a required precursor for downstream induction heads.

---

### 5. The Two-Stage Circuit Mechanism

Combining our attention pattern maps and zero-ablation results reveals a **2-stage circuit** that operates across transformer layers:

```text
[ Token A ] at Pos 1 ───►  STAGE 1: Previous-Token Head (L0H8)
                                │  Looks 1 token back (S = D - 1)
                                ▼  Writes "My prev token was A" into Pos 2 stream
                           [ Token B ] at Pos 2
                                │
                                │  (Residual Stream propagates)
                                ▼
[ Token A ] at Pos 51 ──►  STAGE 2: Induction Head (L5H5 / L5H1)
                                │  Queries: "Which position was preceded by A?"
                                ├──────► Attends to Pos 2 (Key: "I was preceded by A!")
                                ▼
                           Predicts [ Token B ] next!

```

1. **Stage 1 — The Writer (Layers 0–4):**
* **Previous-Token Heads** (like **L0H8**) look back exactly one position ($S = D - 1$).


* They write the identity of token $X_{i-1}$ into the residual stream at position $X_i$.


2. **Stage 2 — The Reader & Predictor (Layers 5–7):**
* **Induction Heads** (like **L5H5** and **L5H1**) query the context for previous instances of the current token.


* Using the information written by Stage 1, they locate where $X_i$ previously occurred and retrieve the token that followed it ($X_{i+1}$) to make the next-token prediction.





---

### 6. Visual Evidence & Attention Map Signatures

| Feature | Stage 1: Previous-Token Head (L0H8) | Stage 2: Induction Head (L5H5) |
| :--- | :--- | :--- |
| **Attention Pattern** | Continuous **Subdiagonal Line** ($S = D - 1$) | Shifted **Induction Diagonal** ($S = D - 49$) |
| **Active Region** | Active across the **entire sequence** (positions 1–100) | Active **only in second half** (positions 51–100) |
| **Function / Role** | Copies token $X_{i-1}$'s identity into position $X_i$'s residual stream. | Finds where current token $X_i$ previously appeared, matches it, and retrieves $X_{i+1}$ to predict it |
---

### 7. Summary Conclusion

Through this experiment, we successfully isolated and validated the mechanistic circuit for in-context repetition in GPT-2 Small:

* Induction in transformer models is not performed by a single isolated component, but through a cooperative **two-head pipeline**.
* Early **Previous-Token Heads** prepare the context representation, enabling middle **Induction Heads** to execute direct copy-pasting of observed patterns.