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
      |
      v
["english_word", "remaining_subword"]
      |
      v
English-associated token representation
      |
      v
Possible semantic interference
```

The important question is **not** whether Romanized low-resource languages generally perform worse.

The question is whether:

```text
Cross-language token collision
        |
        v
Predictable semantic attraction
        |
        v
Wrong-language interpretation
```

occurs systematically enough to constitute a distinct failure mode.

SISH is currently treated as an **experimental hypothesis**, not an established universal property of LLMs.

---

# Research Question

The primary research question is:

> **Do cross-language token collisions in Romanized low-resource languages systematically induce semantic capture by the dominant language?**

Suppose a Romanized target-language word `x` contains a tokenizer piece `t` that is strongly associated with English.

We define:

```text
g_K = correct meaning of the target-language word
g_E = semantic concept associated with the English collision token
```

We test whether the Romanized form increases the model's preference for `g_E` relative to the native-script form.

Conceptually:

```text
P(g_E | Romanized word) > P(g_E | Native-script word)
```

More broadly, we test whether:

```text
P(English-directed error | High collision)
    >
P(English-directed error | Low collision)
```

The strongest form of the SISH hypothesis predicts that the **direction of the semantic error itself** should be predictable from the meaning of the interfering token.

---

# Motivation

Most deployed LLMs use static subword tokenizers such as BPE, SentencePiece, WordPiece, or related methods.

A low-resource word may therefore be represented as:

```text
word -> token_1, token_2, ..., token_n
```

where one or more tokens may have been learned primarily from contexts belonging to another language.

For Romanized Indic languages, the Latin script makes this particularly interesting because tokenizer pieces can overlap directly with common English lexical units.

For example:

```text
Romanized target-language word
            |
            v
       Tokenizer
            |
            v
      ["inn", "uru"]
```

If `inn` is represented primarily through English training contexts, the model may receive an English-semantic signal even though the complete word belongs to another language.

However:

```text
Token collision != SISH
```

A tokenizer collision is only a **candidate mechanism**.

For an example to provide strong evidence for SISH, the collision should result in a predictable semantic shift toward the meaning associated with the interfering token.

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

```text
Native-script word
        |
        v
Model interpretation


Romanized word
        |
        v
Model interpretation
```

This provides a counterfactual comparison while approximately preserving the intended lexical meaning.

For each condition, the model is scored on two candidate meanings:

```text
g_K = correct target-language meaning
g_E = predicted English semantic attractor
```

A semantic-attractor margin is defined as:

```text
SISH margin = score(g_E) - score(g_K)
```

where `score()` is the length-normalized log-probability assigned by the target model to that candidate meaning.

Interpretation:

```text
SISH margin > 0
    -> model prefers the English semantic attractor

SISH margin < 0
    -> model prefers the correct target-language meaning
```

A particularly interesting candidate therefore has:

```text
Romanized margin > 0
Native-script margin < 0
```

In other words:

```text
Romanized representation
    -> English attractor preferred

Native-script representation
    -> Correct meaning preferred
```

---

# Computing the Semantic Preference Score

For a candidate gloss consisting of several tokens, the project uses mean token log-probability rather than raw summed log-probability.

Conceptually:

```text
score(gloss | input)
    =
average log-probability of the gloss tokens
```

This helps reduce the bias caused by comparing a one-token English attractor with a longer dictionary definition.

The main comparison is:

```text
M_roman =
    score(g_E | Romanized input)
    -
    score(g_K | Romanized input)
```

and:

```text
M_native =
    score(g_E | Native-script input)
    -
    score(g_K | Native-script input)
```

The script contrast can then be written as:

```text
Script contrast = M_roman - M_native
```

A large positive script contrast indicates that Romanization shifted the model toward the English semantic attractor relative to the native-script form.

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

because the model may assign high likelihood to `"inn"` simply due to substring or lexical overlap with the input.

Instead, the pipeline attempts to test the **concept associated with the token**.

For example:

```text
Collision token:
inn

Possible semantic attractor:
hotel / lodging
```

The experiment then compares:

```text
Correct target-language meaning
vs.
English semantic attractor
```

This helps distinguish genuine **semantic interference** from simple token copying.

---

# Adversarial SISH Miner

The repository contains an agentic/adversarial search pipeline designed to discover new SISH candidates automatically.

The miner follows a closed loop:

```text
Target-language lexicon
        |
        v
Attested Romanizations
        |
        v
Generate plausible Romanization variants
        |
        v
Tokenize with target-model tokenizer
        |
        v
Detect English lexical collisions
        |
        v
Predict English semantic attractor
        |
        v
Reject semantic-overlap / loanword cases
        |
        v
Evaluate Romanized semantic margin
        |
        v
Evaluate native-script counterfactual
        |
        v
Free-generation verification
        |
        v
Store positives + hard negatives
        |
        v
Use previous results to guide the next search generation
```

The system therefore behaves as an **agentic red-team miner** rather than simply generating random words.

It uses results from previous experiments to guide subsequent candidate generation.

---

# Agentic Search

The search agent operates over a restricted action space.

It can modify plausible Romanization patterns such as:

```text
aa <-> a
ii <-> i
uu <-> u
v  <-> w
sh <-> s
bh <-> b
th <-> t
dh <-> d
kh <-> k
gh <-> g
```

The search process is intentionally constrained so that generated strings remain linguistically plausible rather than becoming arbitrary adversarial character sequences.

The search procedure can incorporate:

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

For a tokenizer piece `t`, the miner approximates collision strength using:

```text
CollisionScore(t)
    =
EnglishFrequency(t)
    -
RomanizedTargetLanguageFrequency(t)
```

The target-language component is estimated using tokenizer-piece occurrences across attested Romanizations.

The score is intended primarily for **ranking candidates**, not as a calibrated probability.

This is preferable to simply checking whether a Romanized word visually resembles an English word.

Importantly:

```text
Surface resemblance
        !=
Tokenizer collision
        !=
Semantic interference
```

Each step must be tested separately.

---

# Semantic-Overlap Filtering

Loanwords and cognates can produce misleading apparent positives.

For example:

```text
Romanized word: cementu
Tokenizer piece: cement
Correct meaning: cement
```

This should **not** be classified as SISH because the English-associated token points toward the correct concept.

The miner therefore rejects candidates when:

```text
SemanticSimilarity(g_K, g_E)
```

is too high.

The current pipeline can use WordNet-derived semantic information together with dictionary glosses to remove obvious overlap cases.

---

# Diversity and Mode-Collapse Prevention

An adversarial search agent can easily discover one productive lexical pattern and repeatedly generate minor variants of the same word.

To prevent this, the miner applies diversity constraints across:

native lexical roots,

English attractor tokens,

Romanization patterns,

previously tested examples.

A novelty reward encourages exploration of new collision structures rather than repeatedly exploiting one successful pattern.

Conceptually:

Candidate reward
    =
SISH evidence
+ collision quality
+ novelty
- Romanization unnaturalness

The exact reward function may evolve as the experimental design develops.

Model Screening

Before using an LLM as the primary mining target, the project evaluates whether the model has sufficient knowledge of the target language.

Two quantities are measured:

A_native = semantic accuracy using native script

A_roman  = semantic accuracy using Romanized input

The script gap is:

ScriptGap = A_native - A_roman

An ideal primary model should have:

Reasonably strong native-script understanding
+
Measurably weaker Romanized understanding

If a model performs poorly in both conditions, apparent SISH failures may simply reflect lack of target-language knowledge.

A model is therefore not considered a strong SISH testbed merely because it produces many Romanized-language errors.

Tokenizer Analysis

The repository contains tokenizer experiments for multiple LLM families.

These are used to examine whether the same Romanized word produces different segmentation patterns across models.

For example:

Same Romanized word
        |
        +--> Tokenizer A -> ["token_1", "token_2"]
        |
        +--> Tokenizer B -> ["english_word", "suffix"]
        |
        +--> Tokenizer C -> ["frag", "ment", "ed"]

This allows SISH susceptibility to be studied as a function of:

vocabulary composition,

vocabulary size,

tokenizer algorithm,

subword fertility,

cross-language token collisions,

multilingual pretraining.

Different tokenizers may expose very different candidate sets even when the underlying language data is identical.

Candidate Labels

The project distinguishes several stages of evidence.

Token-Collision Candidate

A real target-language word whose Romanization produces a meaningful English-associated tokenizer piece.

Margin-Positive Candidate

A candidate satisfying approximately:

M_roman > 0
M_native < 0

preferably across multiple neutral prompt templates.

Auto-Strict Candidate

A margin-positive candidate for which free generation also produces the predicted English semantic family in the Romanized condition but not the native-script condition.

Verified SISH Example

A manually reviewed example where the observed behavior is consistent across controlled prompts and reasonable alternative explanations have been considered.

The project deliberately avoids treating all tokenizer collisions as SISH.

Hard Negatives

Failed candidates are retained rather than discarded.

These examples are important because they allow the project to ask:

Why do some cross-language token collisions produce semantic interference while others are successfully repaired by transformer context?

The resulting dataset contains:

D_positive =
SISH candidates / verified positives

D_negative =
plausible token collisions that do not induce SISH

This may eventually allow prediction of:

P(
    SISH
    |
    tokenization,
    language,
    context,
    model
)

and help distinguish harmful token collisions from harmless ones.
