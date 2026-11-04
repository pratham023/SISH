# Script-Induced Semantic Hallucination (SISH) in Large Language Models

> Investigating whether cross-language token collisions can induce predictable semantic errors in multilingual and low-resource language settings.

## Overview

Large Language Models typically operate over a fixed token vocabulary learned before or during pretraining. When a language is poorly represented in that vocabulary, its words may be segmented into subword units that were learned primarily from another, higher-resource language.

This project investigates a possible failure mode arising from this phenomenon, which we call **Script-Induced Semantic Hallucination (SISH)**.

The central hypothesis is:

> When a word from an underrepresented language is written in a script shared with a dominant training language, its tokenization may contain subwords strongly associated with the dominant language. These subwords may bias the model toward an incorrect semantic interpretation associated with that language.

For example, a Romanized Indic word may contain a tokenizer piece that is also a meaningful English word:

```text
Romanized word
      ↓
["english_word", "remaining_subword"]
      ↓
English-associated token representation
      ↓
Possible semantic interference
```

The important question is **not** whether Romanized low-resource languages generally perform worse.

The question is whether:

```text
cross-language token collision
        ↓
predictable semantic attraction
        ↓
wrong-language interpretation
```

occurs systematically enough to constitute a distinct failure mode.

SISH is currently treated as an **experimental hypothesis**, not an established universal property of LLMs.

---

# Research Question

The primary research question is:

> **Do cross-language token collisions in Romanized low-resource languages systematically induce semantic capture by the dominant language?**

More formally, suppose a Romanized target-language word \(x\) contains a tokenizer piece \(t\) that is strongly associated with English.

Let:

- \(g_K\) be the correct meaning of the target-language word.
- \(g_E\) be the semantic concept associated with the English collision token.

We test whether:

\[
P(g_E \mid x_{\text{romanized}})
>
P(g_E \mid x_{\text{native}})
\]

and, more strongly, whether:

\[
P(g_E \mid C_{\text{high}})
>
P(g_E \mid C_{\text{low}})
\]

where \(C\) represents cross-language token-collision strength.

The strongest form of the hypothesis predicts that the **direction of the error itself** should be predictable from the semantics of the interfering token.

---

# Motivation

Most deployed LLMs use static subword tokenizers such as BPE, SentencePiece, WordPiece, or related methods.

A low-resource word may therefore be represented as:

\[
w
\rightarrow
(t_1,t_2,\ldots,t_n)
\]

where one or more \(t_i\) were primarily learned from contexts belonging to another language.

For Romanized Indic languages, the Latin script makes this particularly interesting because the resulting tokenizer pieces can overlap directly with common English lexical units.

For example:

```text
Romanized target-language word
            ↓
       tokenizer
            ↓
 ["inn", "uru"]
```

If `inn` is represented primarily through English training contexts, the model may receive an English-semantic signal even though the complete word belongs to another language.

However:

```text
token collision ≠ SISH
```

A tokenizer collision is only a **candidate mechanism**.

For an example to provide strong evidence for SISH, the collision should result in a predictable semantic shift toward the meaning of the interfering token.

---

# What Counts as a SISH Candidate?

A word is first treated as a **SISH candidate**, not automatically as a SISH example.

The current pipeline searches for candidates satisfying several properties:

1. The word is a genuine target-language lexical item.
2. The Romanization is attested or linguistically plausible.
3. The target model's tokenizer contains a meaningful English-associated token inside the Romanized form.
4. The English token's meaning differs substantially from the true meaning of the word.
5. The model assigns increased preference to the English semantic attractor when the word is Romanized.
6. The same preference is reduced or reversed when the word is presented in its native script.
7. Ideally, the model also produces the predicted English semantic concept during free generation.

A candidate passing only the tokenizer test is **not** considered evidence of SISH.

---

# Counterfactual Design

A key part of the project is comparing the same lexical item under different script conditions.

For a target word \(w\):

```text
Native script
     ↓
Model interpretation

Romanized form
     ↓
Model interpretation
```

This provides a counterfactual:

\[
x_{\text{native}}
\leftrightarrow
x_{\text{romanized}}
\]

while approximately preserving the intended meaning.

A particularly interesting candidate satisfies:

\[
M_{\text{roman}}>0
\]

and

\[
M_{\text{native}}<0
\]

where

\[
M
=
\ell(g_E \mid x)
-
\ell(g_K \mid x)
\]

and

\[
\ell(g \mid x)
=
\frac{1}{|g|}
\sum_i
\log P(g_i \mid x,g_{<i}).
\]

Positive \(M\) means that the model prefers the English semantic attractor over the correct target-language meaning.

---

# Why Semantic Attractors Are Used Instead of Literal Tokens

Suppose the Romanized form contains the tokenizer piece:

```text
inn
```

It is not sufficient to compare:

```text
correct meaning
vs.
"inn"
```

because the model may assign high likelihood to `"inn"` simply due to lexical overlap with the input.

Instead, the pipeline attempts to test the **concept associated with the token**.

For example:

```text
collision token:
inn

semantic attractor:
hotel / lodging
```

The experiment then compares:

```text
correct target-language meaning
vs.
English semantic attractor
```

This helps distinguish **semantic interference** from simple substring copying.

---

# Adversarial SISH Miner

The repository contains an agentic/adversarial search pipeline designed to discover new SISH candidates automatically.

The miner follows a closed loop:

```text
Target-language lexicon
        ↓
Attested Romanizations
        ↓
Generate plausible Romanization variants
        ↓
Tokenize with target-model tokenizer
        ↓
Detect English lexical collisions
        ↓
Predict English semantic attractor
        ↓
Reject semantic-overlap / loanword cases
        ↓
Evaluate Romanized likelihood margin
        ↓
Evaluate native-script counterfactual
        ↓
Free-generation verification
        ↓
Store positives + hard negatives
        ↓
Use previous results to guide next search generation
```

The system therefore behaves as an **agentic red-team miner** rather than simply generating random words.

It learns which types of candidate transformations produce interesting failures and uses previous experiments to guide subsequent search.

---

# Agentic Search

The search agent operates over a restricted action space.

It can modify plausible Romanization patterns such as:

```text
aa ↔ a
ii ↔ i
uu ↔ u
v  ↔ w
sh ↔ s
bh ↔ b
th ↔ t
dh ↔ d
kh ↔ k
gh ↔ g
```

The search process is intentionally constrained so that generated strings remain linguistically plausible rather than becoming arbitrary adversarial character sequences.

The current search procedure incorporates:

- tokenizer collision strength,
- English lexical frequency,
- frequency of the same tokenizer piece in attested Romanized target-language data,
- semantic disagreement between the correct gloss and English attractor,
- Romanization edit cost,
- novelty of the collision pattern,
- Romanized/native likelihood contrast,
- free-generation behavior.

---

# Collision Score

Tokenizer pieces are normalized before collision analysis.

Special tokenizer markers such as:

```text
▁
Ġ
_
```

are removed before lexical lookup.

For a tokenizer piece \(t\), the miner approximates cross-language collision strength using:

\[
C(t)
=
F_{\text{English}}(t)
-
F_{\text{Romanized-target}}(t)
\]

where the target-language frequency is estimated from tokenizer-piece occurrences in attested Romanizations.

This is preferable to simply checking whether a Romanized word visually resembles an English word.

For example:

```text
surface resemblance
```

does not necessarily imply:

```text
tokenizer collision
```

and a tokenizer collision does not necessarily imply:

```text
semantic interference
```

These are tested separately.

---

# Semantic-Overlap Filtering

Loanwords and cognates can create misleading apparent positives.

For example:

```text
Romanized word: cementu
Tokenizer piece: cement
Correct meaning: cement
```

This is not SISH because the English token points toward the **correct concept**.

The miner therefore rejects candidates when:

\[
\operatorname{SemanticSimilarity}(g_K,g_E)
\]

is too high.

The current implementation uses lexical-semantic information from WordNet together with the target-language dictionary glosses to remove obvious semantic-overlap cases.

---

# Diversity and Mode-Collapse Prevention

An adversarial search agent can easily discover one productive lexical pattern and repeatedly generate minor variants of the same word.

To prevent this, the miner applies diversity constraints across:

- native lexical roots,
- English attractor tokens,
- Romanization patterns,
- previously tested examples.

A novelty reward encourages exploration of new collision structures rather than repeatedly exploiting a single pattern.

---

# Model Screening

Before using an LLM as the primary mining target, the project evaluates whether the model has sufficient knowledge of the target language.

Two quantities are measured:

\[
A_{\text{native}}
\]

and

\[
A_{\text{roman}}
\]

representing semantic accuracy under native-script and Romanized input.

The script gap is:

\[
G
=
A_{\text{native}}
-
A_{\text{roman}}.
\]

An ideal primary model should have:

```text
reasonably strong native-script understanding
+
measurably weaker Romanized understanding
```

If a model performs poorly in both conditions, apparent SISH failures may simply reflect lack of language knowledge.

---

# Tokenizer Analysis

The repository contains tokenizer experiments for multiple LLM families.

These are used to examine whether the same Romanized word produces different segmentation patterns across models.

Conceptually:

```text
same word
   ↓
Tokenizer A → ["token_1", "token_2"]
Tokenizer B → ["english_word", "suffix"]
Tokenizer C → ["fragment", "fragment", "fragment"]
```

This allows SISH susceptibility to be studied as a function of:

- vocabulary composition,
- vocabulary size,
- tokenizer algorithm,
- subword fertility,
- cross-language token collisions,
- multilingual pretraining.

Different tokenizers may therefore expose very different candidate sets even when the underlying language data is identical.

---

# Repository Structure

A typical repository layout is:

```text
SISH/
│
├── README.md
│
├── tokenizers/
│   ├── model_1/
│   ├── model_2/
│   ├── model_3/
│   └── ...
│
├── notebooks/
│   └── SISH_adversarial_miner.ipynb
│
├── data/
│   ├── lexicons/
│   ├── romanizations/
│   ├── candidates/
│   └── verified/
│
├── results/
│   ├── all_candidates.csv
│   ├── margin_positive_candidates.csv
│   ├── manual_review_candidates.csv
│   └── model_screen.csv
│
└── scripts/
    ├── tokenizer_analysis.py
    ├── collision_mining.py
    └── evaluation.py
```

The exact structure may differ as the project evolves.

---

# Current Experimental Pipeline

The current miner operates approximately as follows:

```text
STEP 1
Load real Kannada lexical items and attested Romanizations.

STEP 2
Run the target model's tokenizer over each Romanization.

STEP 3
Identify meaningful English-associated tokenizer pieces.

STEP 4
Generate a semantic attractor for the English collision.

STEP 5
Reject candidates where the English concept overlaps with the correct meaning.

STEP 6
Compute the model's preference between:

    correct target-language gloss

and

    predicted English semantic attractor.

STEP 7
Repeat the comparison using the native-script form.

STEP 8
Retain candidates showing a Romanized/native preference flip.

STEP 9
Run neutral free-generation prompts.

STEP 10
Manually inspect the strongest candidates before assigning a final SISH label.
```

---

# Candidate Labels

The project distinguishes several stages of evidence.

### Token-Collision Candidate

A real target-language word whose Romanization produces a meaningful English-associated tokenizer piece.

### Margin-Positive Candidate

A candidate satisfying approximately:

\[
M_{\text{roman}}>0
\]

and

\[
M_{\text{native}}<0.
\]

### Auto-Strict Candidate

A margin-positive candidate for which free generation also produces the predicted English semantic family in the Romanized condition but not the native-script condition.

### Verified SISH Example

A manually reviewed example where the observed behavior is consistent across controlled prompts and alternative explanations have been considered.

The project deliberately avoids treating all tokenizer collisions as SISH.

---

# Hard Negatives

Failed candidates are retained rather than discarded.

These examples are important because they allow the project to ask:

> Why do some cross-language token collisions produce semantic interference while others are successfully repaired by transformer context?

The resulting dataset contains:

\[
D^+
=
\text{SISH candidates / positives}
\]

and

\[
D^-
=
\text{plausible token collisions that do not induce SISH}.
\]

This may eventually allow prediction of:

\[
P(
\text{SISH}
\mid
\text{tokenization},
\text{language},
\text{context},
\text{model}
).
\]

---

# Important Alternative Explanations

A Romanized-language error is not automatically SISH.

Possible alternative explanations include:

```text
1. The model does not know the target language.

2. The model knows the native-script form but has never learned the
   Romanization.

3. The Romanization is genuinely ambiguous.

4. The word is fragmented inefficiently without any specific
   wrong-language semantic attraction.

5. The model guesses an English interpretation because the surrounding
   context does not identify the language.

6. The apparent English collision is actually a cognate or loanword.

7. The model copies a substring without adopting its semantics.
```

The experiments are designed to distinguish these explanations from genuine cross-language semantic interference.

---

# Current Scientific Status

SISH should currently be interpreted as a **hypothesis under investigation**.

The project has observed individual examples motivating the hypothesis, including cases in Romanized Indic languages, but it has not yet established that SISH is a general or high-frequency failure mode.

The central empirical question remains:

\[
\boxed{
\text{Are English-associated token collisions systematically more likely
to produce English-directed semantic errors than matched controls?}
}
\]

A positive result would support SISH as a distinct failure class.

A negative result would also be informative: it would suggest that transformer context generally repairs tokenizer-level cross-language collisions, making the originally observed failures rare or model-specific.

---

# Planned Evaluation

The project aims to compare:

```text
high-collision Romanized words
vs.
matched low-collision Romanized words
vs.
native-script counterfactuals
```

while controlling for factors such as:

- lexical frequency,
- word length,
- tokenizer fertility,
- morphology,
- model language competence,
- prompt formulation,
- loanwords and cognates.

The main outcome of interest is **English-directed semantic error**, not generic incorrectness.

A particularly strong result would be:

\[
P(
E_{\text{attractor}}
\mid
C_{\text{high}}
)
\gg
P(
E_{\text{attractor}}
\mid
C_{\text{low}}
).
\]

---

# Future Directions

If SISH is found to be systematic, several mitigation strategies can be investigated:

```text
Balanced multilingual pretraining

Vocabulary adaptation

Tokenizer adaptation

Embedding relearning

Language-conditioning tokens

Language-specific adapters / LoRA

Targeted continued pretraining

Agentic diagnosis and mitigation routing
```

An additional direction is training the adversarial miner itself using reinforcement learning.

For example, candidate generation can be formulated as:

\[
e_1,\ldots,e_G
\sim
\pi_\theta(e\mid s)
\]

where candidates receive higher reward when they produce reproducible, semantically directional SISH behavior.

Methods such as contextual bandits, evolutionary search, or GRPO could then be used to improve the discovery policy.

The current implementation intentionally begins with interpretable closed-loop search before introducing RL.

---

# Reproducibility

Each discovered candidate should retain:

```text
target language
native word
Romanization
dictionary source
correct gloss
target model
model revision
tokenizer
tokenizer pieces
collision token
predicted semantic attractor
collision score
Romanized likelihood margin
native-script likelihood margin
prompt templates
free-generation outputs
search-generation number
mutation rules
final manual label
```

Model and tokenizer revisions should be pinned whenever possible.

---

# Terminology

### SISH

**Script-Induced Semantic Hallucination**

A proposed failure mode where the script/tokenization of an underrepresented-language word causes the model to adopt an incorrect semantic interpretation associated with another language represented by its constituent tokens.

### Cross-Language Token Collision

A tokenizer event where part of a word from one language corresponds to a lexical token strongly associated with another language.

### Semantic Attractor

The meaning associated with the interfering token.

### Script Counterfactual

The same lexical item represented in its native script rather than Romanized form.

---

# Disclaimer

This repository is research-oriented and exploratory.

The existence of tokenization imbalance across languages does **not** by itself establish SISH.

Likewise:

```text
Romanized-language error
```

does not imply:

```text
token-induced semantic hallucination.
```

The goal of this project is specifically to determine whether the stronger causal relationship exists and, if so, under what model, tokenizer, language, and contextual conditions it appears.

---

# Contributions

Contributions are welcome, particularly for:

```text
additional Indic-language lexicons

attested Romanization datasets

tokenizer implementations

cross-model replication

native-speaker validation

matched negative controls

semantic-attractor verification

multilingual model screening
```

Please include the target model/tokenizer revision and enough information to reproduce any submitted SISH candidate.

---

# Citation

A formal citation will be added if/when this work is released as a paper.

For now, please refer to the repository as:

```text
Script-Induced Semantic Hallucination (SISH):
Cross-Language Token Collisions and Semantic Interference in LLMs
```

---

## Project Status

🚧 **Active research / hypothesis-testing stage**

The immediate goals are:

```text
1. Build reliable cross-language collision metrics.
2. Scale adversarial candidate mining.
3. Establish strong native-script counterfactuals.
4. Construct matched negative controls.
5. Test whether collision strength predicts directional semantic errors.
6. Replicate results across different model/tokenizer families.
```

The project will treat SISH as an established phenomenon only if these experiments support the hypothesis.
