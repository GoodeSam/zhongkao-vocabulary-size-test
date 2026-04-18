# Vocabulary Testing Software for Entrance Exam Preparation

## Process Log

---

# Part 1: Initial Strategy & Architecture Design (v1)

## Vocabulary Assessment Strategy Based on Word Frequency Statistics

### 1. Market Analysis of Existing Solutions

#### 1.1 Mainstream Products and Their Assessment Logic

| Product                                              | Assessment Method                                          | Strengths                     | Weaknesses                                                                                 |
| ---------------------------------------------------- | ---------------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------ |
| **Baicizhan**                                        | Image-word association + self-assessment "know/don't know" | Engaging, high user retention | Image association != true mastery; passive recognition only, can't test productive ability |
| **Momo Memorize**                                    | Forgetting curve-based self-assessment review              | Personalized review pacing    | Relies on user honesty; vocabulary estimation not precise enough                           |
| **Shanbay**                                          | Multiple choice + spelling + sentence fill-in              | Multi-dimensional testing     | Fixed question types, easy to develop test-taking tricks rather than real mastery          |
| **Bubei Words**                                      | Real context sentence meaning selection                    | Good contextual understanding | Single assessment dimension (recognition only)                                             |
| **Vocabulary test websites** (e.g., Test Your Vocab) | Sampling self-assessment -> statistical inference          | Quick total estimate          | Coarse granularity; disconnected from exam syllabi                                         |

#### 1.2 Common Problems

- **Disconnected from exam syllabi**: Most products organize by CET-4/6 or general word lists, not fine-tuned to zhongkao/gaokao syllabi
- **Frequency blind spot**: No distinction between "tested 100 times core word" and "appeared once edge word" — equal weight treatment
- **Recognition only, no production**: Mostly test "see English, choose Chinese" — cannot assess spelling, collocation, contextual usage
- **Lack of diagnostics**: Tells you "vocabulary 5000" but not "which high-frequency exam words have gaps"

### 2. Frequency-Based Assessment Strategy Design

#### Core Principle

> **Not all words deserve equal testing.** A word tested 50 times and a word tested once have completely different exam prep value. Assessment should tier by frequency and diagnose "high-frequency unmastered" danger zones.

#### Layer 1: Frequency Tiering Model

Based on existing zhongkao/gaokao word frequency data, words are divided into four tiers:

| Tier                      | Definition                                  | Characteristics                           | Testing Weight             |
| ------------------------- | ------------------------------------------- | ----------------------------------------- | -------------------------- |
| **S-Core High Frequency** | Top 10% frequency, covering many papers     | High `distinct_files`, high `total_freq`  | Must test, test repeatedly |
| **A-Common**              | Upper-mid frequency, appears in most papers | Stable but not extreme                    | Key testing                |
| **B-Mid Frequency**       | Occasional appearance, some coverage        | Appears in specific question types/topics | Sample test                |
| **C-Low/Edge**            | Appeared only 1-2 times or in few papers    | May be beyond syllabus or topic-specific  | Light/optional test        |

**Key insight**: Cannot look only at `total_freq` — must combine with `distinct_files` (cross-paper coverage). A word appearing 10 times in one paper vs appearing once in 10 papers — the latter has higher exam prep value.

#### Layer 2: Multi-Dimensional Assessment Matrix

| Dimension                   | Test Format                                       | Corresponding Exam Ability          |
| --------------------------- | ------------------------------------------------- | ----------------------------------- |
| **Recognition** (receptive) | English->Chinese selection; audio word selection  | Reading comprehension, listening    |
| **Production** (productive) | Chinese->English spelling; initial letter fill-in | Written expression, grammar fill-in |
| **Collocation**             | Common collocation selection/completion           | Cloze fill-in                       |
| **Context** (contextual)    | Judge meaning/usage in sentences                  | Reading inference, cloze            |

**Tiered strategy**:

- S-tier words: all four dimensions
- A-tier words: recognition + production + collocation
- B-tier words: recognition + context
- C-tier words: recognition only

#### Layer 3: Adaptive Sampling Assessment

No need to test every word; use statistical sampling + adaptive adjustment:

1. **Initial assessment** (\~50-80 questions, 5-8 minutes)
   - Proportional sampling from each frequency tier
   - More S-tier, less C-tier
   - Quickly establish "mastery rate vs frequency tier" curve
2. **Ability estimation**
   - Estimate mastery rate per tier (e.g., S-tier 85%, A-tier 70%, B-tier 45%, C-tier 20%)
   - Combined with frequency weighting to calculate "exam coverage rate" — what proportion of real exam word occurrences the user's mastered words cover
3. **Diagnostic report** (core differentiator)
   - Not just "your vocabulary is \~2800"
   - Instead: "You've mastered 78% of exam word occurrences, but S-tier core words have 47 gaps that appeared 312 times total in past papers"
   - Specific "high-frequency unmastered" word list, sorted by ROI

#### Layer 4: Zhongkao vs Gaokao Differentiation

| Dimension              | Zhongkao                                       | Gaokao                                                    |
| ---------------------- | ---------------------------------------------- | --------------------------------------------------------- |
| Syllabus vocabulary    | \~1600 (compulsory education curriculum)       | \~3500                                                    |
| Frequency distribution | More concentrated, core words repeat heavily   | More dispersed, longer tail                               |
| Assessment focus       | Spelling accuracy, basic collocations          | Polysemy, familiar-word-new-meaning, contextual inference |
| Syllabus alignment     | Whether within compulsory education curriculum | Whether within gaokao syllabus                            |

### 3. Differentiation Value Proposition

1. **Real exam frequency driven**: Not ordered by textbook or general word lists, but "test what's actually tested"
2. **Diagnostic, not estimation**: Not just a vague number, but precise identification of high-ROI weak spots
3. **Exam coverage rate**: User can intuitively see "what percentage of the exam my current words cover" — more actionable than "vocabulary N thousand"
4. **Tiered testing efficiency**: Deep test on high-frequency, shallow on low-frequency — more precise diagnosis with fewer questions

---

## Software Architecture Design (v1)

### Overall Architecture

```
+---------------------------------------------------------+
|                      User Layer (Clients)                |
|   +----------+  +----------+  +-------------------+     |
|   |  Web App  |  | Mini App |  | Teacher Dashboard  |    |
|   +----+-----+  +----+-----+  +--------+----------+     |
+---------+--------------+----------------+----------------+
          |              |                |
          v              v                v
+---------------------------------------------------------+
|                    API Gateway / BFF Layer                |
+---------------------------------------------------------+
                         |
         +---------------+---------------+
         v               v               v
+--------------+ +--------------+ +------------------+
|  Assessment  | |  Question    | |  Learning Track  |
|  Engine      | |  Bank        | |  Service         |
+------+-------+ +------+-------+ +-------+----------+
       |                |                  |
       v                v                  v
+---------------------------------------------------------+
|                      Data Layer                          |
|  +------------+ +------------+ +------------------+     |
|  | Word Freq  | | Question   | | User Learning    |     |
|  | Knowledge  | | Bank       | | Records          |     |
|  +------------+ +------------+ +------------------+     |
+---------------------------------------------------------+
```

### Module 1: Data Preprocessing & Word Frequency Knowledge Base

**Input processing flow**:

```
Word freq Excel ----+
                    +--> Word standardization --> Frequency tiering --> Knowledge base
Exam papers --------+      (lemmatization)
```

**Per-word structured attributes**:

| Field           | Description                                           |
| --------------- | ----------------------------------------------------- |
| lemma           | Word lemma                                            |
| frequency\_rank | Frequency ranking                                     |
| tier            | S / A / B / C                                         |
| exam\_scope     | Zhongkao required / Gaokao extended / Beyond syllabus |
| pos\_tags       | Common parts of speech                                |
| senses          | Sense list (sorted by exam frequency)                 |
| collocations    | High-frequency collocations                           |
| exam\_contexts  | Exam source index                                     |
| confusables     | Confusable word groups                                |

### Module 2: Question Bank Service

#### Question Source — Dual Track

```
Track A: Real exam question slicing
  Multiple papers -> Decompose by question type -> Label target vocabulary per item -> Enter bank

Track B: Template-based auto-generation
  Word entries from knowledge base -> Generate multiple question types -> Human/AI review -> Enter bank
```

#### Question Type System

| Depth             | Type                                            | Applicable Tier | Exam Scenario                    |
| ----------------- | ----------------------------------------------- | --------------- | -------------------------------- |
| L1 Recognition    | English->Chinese 4-choice                       | All             | Reading basics                   |
| L2 Listening      | Audio word/meaning selection                    | S, A            | Listening comprehension          |
| L3 Spelling       | Chinese->English / initial letter fill          | S, A            | Written expression, grammar fill |
| L4 Collocation    | Select preposition/collocation                  | S, A, B         | Cloze fill                       |
| L5 Context        | In-sentence meaning / familiar-word-new-meaning | S, A            | Reading inference                |
| L6 Discrimination | Synonym/confusable discrimination               | S               | Cloze, grammar selection         |
| L7 Real Exam      | Direct real exam question clips                 | All             | Comprehensive ability            |

### Module 3: Assessment Engine (Core)

#### Assessment Flow

```
            +---------------------------------------+
            |        Assessment Session              |
            |                                       |
  Start --> |  Select -> Present -> Answer -> Update |  --> Diagnostic Report
            |      ^                    |           |
            |      +--------------------+           |
            |         (Loop N rounds)               |
            +---------------------------------------+
```

#### Three-Phase Selection Strategy

**Phase 1: Quick Positioning (first 10 questions)**

- Sample 2-3 representative words from each S/A/B/C tier
- Uniform L1 type (English->Chinese), reduce interference
- Goal: quickly determine approximate level range

**Phase 2: Precise Probing (questions 11-50)**

- Focus on "fuzzy zone" (mastery rate \~40%-70% tier)
- Dense sampling within that tier, upgrade to L2-L5
- Adaptive: consecutive correct -> increase difficulty/tier; consecutive wrong -> decrease
- S-tier wrong answers get extra confirmation question in different type

**Phase 3: Weakness Drilling (questions 51-80, optional)**

- Deep dive into specific weak areas exposed in Phase 2

#### Ability Estimation Model

```
User vocabulary profile = {
    By frequency tier: { S: 92%, A: 75%, B: 48%, C: 15% }
    By cognitive depth: { Recognition: 85%, Spelling: 60%, Collocation: 55%, Context: 45% }
    By topic domain:   { Daily life: 90%, Technology: 50%, Environment: 65%, ... }
    Composite metric:  Exam vocabulary coverage = 82%
}
```

**Exam vocabulary coverage calculation**:

```
Coverage = Sum(mastered word exam frequencies) / Sum(all syllabus word exam frequencies)
```

### Module 4: Learning Tracker

**Per-word state machine**:

```
  Untested --> First test --> Mastered (needs consolidation) --> Firmly mastered
                |                |
                v                v
            Not mastered --> Learning --> Pending retest
                         ^          |
                         +----------+
```

**Review scheduling**: S-tier unmastered: 1 day; A-tier unmastered: 2 days; Mastered words: 1->3->7->15->30 day intervals.

### Exam Paper Processing Strategy

| Question Type                 | Vocabulary Testing Method                    | Value to System                                          |
| ----------------------------- | -------------------------------------------- | -------------------------------------------------------- |
| Cloze fill                    | Direct vocabulary discrimination/collocation | **Highest** — directly reusable as test items            |
| Grammar fill / initial letter | Spelling + morphological change              | **High** — natural L3 production items                   |
| Reading comprehension         | Indirect vocabulary depth testing            | **Medium** — extract context for L5 items                |
| Listening                     | Audio recognition                            | **Medium** — adaptable to L2 items                       |
| Written expression            | Production ability                           | **Low** (reference only) — extract high-score vocabulary |

### Tech Stack

| Layer             | Recommendation                       | Reason                                                     |
| ----------------- | ------------------------------------ | ---------------------------------------------------------- |
| Frontend          | WeChat mini-program (priority) + Web | Primary entry for target users (middle school students)    |
| API               | RESTful, lightweight BFF             | Low initial complexity                                     |
| Assessment Engine | Independent service, Python          | Easy algorithm iteration, consistent with existing scripts |
| Question Bank     | PostgreSQL + JSON fields             | Structured queries + flexible question formats             |
| Word Freq KB      | PostgreSQL + full-text index         | Fast query and statistical aggregation                     |
| User Records      | PostgreSQL (sufficient initially)    | Single table, split later                                  |
| Cache             | Redis                                | Session state, question selection cache                    |

### MVP & Iteration Path

**MVP (4-6 weeks)**: Import existing freq Excel -> auto-tier; 50-question adaptive test (L1 only); diagnostic report; single platform

**V2 (+4 weeks)**: Real exam bank; L3 spelling + L4 collocation; cognitive depth dimension; basic review scheduling

**V3 (+4 weeks)**: Full paper processing pipeline; topic domain analysis; teacher dashboard; spaced repetition

**V4 (long-term)**: AI question generation; mock exam mode; social learning features

---

# Part 2: Codex Evaluation

## Critical Evaluation by OpenAI Codex

### Overall Judgment

The design is strategically promising, but it is stronger as a **diagnostic + prioritization system** than as a pure "adaptive assessment" product in its current form.

The strongest idea is not the 7-level question taxonomy or the 3-phase flow. It is the decision to optimize around **expected exam payoff** rather than raw vocabulary size. That is directionally correct for zhongkao/gaokao prep, because students do not need equal mastery of all syllabus words.

The main weakness is that the design currently mixes three different constructs without separating them clearly:

- **word importance**: frequency / cross-paper coverage
- **word knowledge**: does the student know this word
- **exam performance**: can the student answer questions under context, distractors, time pressure

Those are related, but not interchangeable. If you collapse them too early into one score, the product will look precise while actually being noisy.

### 1. Strategic Soundness — Valid with Caveats

**Strengths:**

- Aligns with actual exam exposure instead of textbook idealism
- Using both `total_freq` and `distinct_files` is better than plain frequency
- "Coverage rate" is more actionable than "you know 2,300 words"

**Core flaws in assumptions:**

- **Frequency is not discriminative value.** Very common function words may dominate frequency but contribute little to diagnosis
- **Occurrence count is not score impact.** A word appearing 20 times in passages may matter less than a word appearing once in a high-value cloze item
- **Word-family handling is dangerous.** `decide`, `decision`, `decisive`, `indecisive` are not interchangeable for exam prep. Over-aggressive lemmatization destroys exam realism
- **Knowing a word is multidimensional.** "Mastered word frequencies" is too blunt unless mastery is defined by skill level
- **Past-paper frequency can overfit.** Limited or region-specific corpus may learn historical quirks rather than durable exam targets

**Recommended changes:**

- Keep frequency weighting but define it as **instructional priority**, not truth
- Add **item impact weight** (passage-only exposure vs. cloze-keyword vs. answer-option vs. writing-relevant)
- Use **word family graphs**, not simple lemma collapse

### 2. Architecture Quality — Clean but Incomplete

Current 4-module decomposition is fine, but **missing modules**:

- **Psychometric calibration module**: item difficulty, discrimination, guessability, DIF risk
- **Content governance module**: syllabus versioning, exam source provenance, copyright status, region/year tags
- **Experimentation/analytics module**: score validity, prediction to mock-exam outcomes, item performance drift
- **Teacher/parent reporting layer**

Module 2 and Module 3 are underspecified where real complexity lives. "Maximize information gain" only works if item parameters exist and are stable.

### 3. Feasibility Risks — Content and Calibration, Not Tech

Highest-risk areas:

- **Item calibration**: adaptive testing needs enough response data to estimate parameters. Cold start is hard
- **Auto-generated question quality**: template items are easy to make, hard to make good
- **Sense-level labeling**: frequency at lemma level is often too coarse
- **Copyright and reuse risk**: real exam slicing may be legally sensitive
- **Listening module**: audio quality, accents, segmentation add substantial complexity

**MVP underestimation**: 4-6 weeks is realistic only for a **heuristic diagnostic prototype**, not a defensible adaptive assessment system.

### 4. Assessment Model Validity — The Biggest Conceptual Issue

The adaptive testing idea is only statistically sound if you do real calibration. Serious CAT systems rely on **IRT/Rasch-style item models**. Without calibrated items, "adaptive" becomes a smart branching quiz, not a valid CAT.

**Validity concerns:**

- Test is **multidimensional** by design — standard unidimensional CAT assumptions break quickly
- Frequency tier is not difficulty. A rare word can be easy; a common word in subtle context can be hard
- Content routing can create structural bias in reported scores

**About "exam coverage rate":**

- Good as a **learning KPI**
- Weak as a **measurement score**
- Potentially misleading as a **predictor of exam performance**

**Better framing — three scores, not one:**

- **Coverage score**: weighted by exam frequency, for prioritization
- **Mastery score**: estimated ability by skill type
- **Risk score**: probability of breakdown on exam tasks due to weak high-impact words

### 5. Market Positioning — Differentiation Partly Real

The gap is **not** "others use textbook lists, we use exam data." Competitors already claim exam orientation and memory science.

Real differentiation, if executed well:

- **Diagnostic precision at word-impact level**
- **Exam-coverage reporting tied to real corpora**
- **Ability-to-priority mapping**: "learn these 120 words first to increase coverage from 62% to 78%"
- **Middle/high school exam specialization**, region- or syllabus-specific

### 6. Suggestions

1. **Reframe**: position as **exam vocabulary diagnostic engine**, not full adaptive test at day one
2. **Separate models**: importance model / measurement model / recommendation model
3. **Reduce MVP**: one exam only, L1 + limited L5, semi-adaptive, coverage/risk report
4. **Staged psychometrics**: heuristic -> data-calibrated -> Rasch/2PL CAT
5. **Add sense-level and phrase-level modeling**
6. **Validate against real outcomes**: does coverage score predict mock-exam performance?
7. **Treat content rights seriously**: design fallback to licensed/transformed/authored items
8. **Upgrade reporting**: show high-frequency unknowns, false familiarity words, predicted loss points by task type

### Bottom Line

> The concept is good. The best part is the shift from "vocabulary size" to **weighted exam utility**. The weak part is overclaiming measurement rigor before calibration data exists. The design is strongest as a **data-driven diagnostic and prioritization product**, not yet a statistically mature adaptive testing system.

---

# Part 3: Revised Architecture Design (v2)

> Based on Codex review feedback. Core shift: from "adaptive testing system" to **data-driven vocabulary diagnostic and prioritization engine**, with staged measurement rigor evolution.

## 1. Positioning Correction

|                      | v1 Original                               | v2 Revised                                                                 |
| -------------------- | ----------------------------------------- | -------------------------------------------------------------------------- |
| Positioning          | Adaptive vocabulary testing system        | Exam vocabulary diagnostic & prioritization engine                         |
| Core promise         | Precisely measure your vocabulary ability | Tell you **what to study next** for maximum score improvement              |
| Measurement attitude | Implied CAT-level rigor                   | Explicitly labeled "heuristic diagnosis", upgrading with data accumulation |
| Target exam          | Zhongkao + Gaokao simultaneously          | **MVP: one only** (Shenzhen zhongkao first, aligned with existing data)    |

## 2. Three-Model Separation (Core Architecture Change)

v1's biggest problem was conflating "word importance", "whether student has mastered", and "exam performance". v2 separates them into three independent models:

```
+-------------------------------------------------------------+
|                    Three-Model Architecture                   |
|                                                             |
|  +---------------+  +---------------+  +----------------+   |
|  |  Importance    |  |  Mastery      |  |  Recommendation|   |
|  |  Model         |  |  Model        |  |  Model         |   |
|  |                |  |               |  |                |   |
|  |  How important |  |  How well does|  |  What to study |   |
|  |  is this word  |  |  this student |  |  next for max  |   |
|  |  for the exam? |  |  know this    |  |  score gain?   |   |
|  |                |  |  word?        |  |                |   |
|  +-------+-------+  +-------+-------+  +-------+--------+   |
|          |                  |                   |            |
|          +--------+---------+                   |            |
|                   v                             |            |
|            +-------------+                      |            |
|            |  Diagnostic  |<--------------------+            |
|            |  Report      |                                  |
|            +-------------+                                   |
+-------------------------------------------------------------+
```

### Importance Model

**Input**: Word frequency statistics + exam papers

**Output**: Exam impact score per word entry

**Three-dimensional importance scoring**:

```
Importance = a * frequency_weight + b * coverage_weight + g * task_impact_weight

Where:
  frequency_weight   = log(total_freq + 1) normalized
  coverage_weight    = distinct_files / total_files
  task_impact_weight = weighted by occurrence location:
    +---------------------------+--------+
    | Location                  | Weight |
    +---------------------------+--------+
    | Cloze correct answer      | 1.0    |
    | Grammar fill / initial    | 0.9    |
    | Reading question/options  | 0.7    |
    | Reading passage           | 0.4    |
    | Listening passage         | 0.3    |
    | Pure text occurrence      | 0.2    |
    +---------------------------+--------+
```

**Word family graph instead of flat lemmatization**:

```
  decide (verb, S-tier)
    +-- decision (noun, S-tier)  <- independently tracked
    +-- decisive (adj, B-tier)
    +-- indecisive (adj, C-tier)

  Relationship types: derivation, inflection, compounding
  Each node scored independently, sharing "family priority"
```

**Sense-level annotation**:

```
  right
    +-- sense 1: correct (adj) -- high exam freq, S-tier
    +-- sense 2: rights (n) -- mid exam freq, A-tier  <- "familiar word, new meaning"
    +-- sense 3: right side (adj) -- low exam freq, C-tier
```

### Mastery Model

**Input**: Student response records

**Output**: Mastery probability per word entry x skill dimension

**Mastery matrix (per word)**:

```
         Recog(R)  Spell(W)  Colloc(C)  Context(X)
  Est.     0.85     0.60      0.40       0.30
  Conf.    High     Medium    Low        Untested
```

**Staged measurement evolution**:

```
Stage 1 (MVP, no calibration data)
  -> Heuristic rules: correct=0.8, wrong=0.2, Bayesian update
  -> Explicitly tell users "diagnostic reference", don't promise precise measurement

Stage 2 (after 5,000+ responses accumulated)
  -> Estimate item difficulty and discrimination from real data
  -> Switch from rules to data-driven parameters

Stage 3 (after 50,000+ responses accumulated)
  -> Introduce Rasch/2PL model
  -> Support true adaptive item selection (CAT)
  -> Only NOW can it be called "adaptive testing"
```

### Recommendation Model

**Input**: Importance score x Mastery matrix

**Output**: Personalized learning priority ranking

**Core formula**:

```
Score_gain_utility = Importance(w) * (1 - Mastery(w)) * Learnability(w)

Where:
  Learnability = heuristic based on family members already mastered,
                 word length, semantic distance to known words, etc.
```

Sorted by score\_gain\_utility descending = "what to study next".

## 3. Revised Module Architecture (7 Modules)

```
+-------------------------------------------------------------+
|                        User Layer                            |
|    +----------+   +----------+   +--------------+           |
|    | Mini App  |   | Web App  |   | Teacher Admin |          |
|    +----+-----+   +----+-----+   +------+-------+           |
+---------+--------------+----------------+--------------------+
          +---------------+---------------+
                         |
                         v
+-------------------------------------------------------------+
|                       API Layer                              |
+-------------------------------------------------------------+
                         |
    +--------+--------+--+--+----------+-----------+
    v        v        v     v          v           v
+------+ +------+ +--------+ +--------+ +----------+
| M1   | | M2   | | M3     | | M4     | | M5       |
| Word | | Item | | Diag.  | | Learn  | | Content  | <- NEW
| Freq | | Bank | | Engine | | Track  | | Govern.  |
| KB   | |      | |        | |        | |          |
+--+---+ +--+---+ +---+----+ +---+----+ +----+-----+
   |        |          |          |            |
   +--------+-----+----+----------+            |
                   v                           |
            +-----------+                      |
            | M6 Calib. | <- NEW               |
            +-----+-----+                      |
                  |                            |
   +--------------+----------------------------+
   v
+-------------------------------------------------------------+
|  M7 Validity & Analytics (NEW)                               |
|  Mock exam prediction - A/B experiments - Item drift monitor |
+-------------------------------------------------------------+
```

### New Modules

**M5 Content Governance**

| Responsibility              | Description                                                       |
| --------------------------- | ----------------------------------------------------------------- |
| Syllabus version management | Quick switch when zhongkao/gaokao syllabus updates                |
| Paper source traceability   | Each item tagged: year, region, paper, question number            |
| Copyright status            | Original / adapted / real exam — controls display strategy        |
| Region tags                 | Shenzhen zhongkao vs Guangzhou vs National — support localization |

**M6 Calibration Service**

| Stage             | Capability                                            |
| ----------------- | ----------------------------------------------------- |
| Cold start        | Accept expert-labeled initial difficulty estimates    |
| Data accumulation | Batch-estimate item parameters from response logs     |
| Mature            | Real-time updates, retire poor items, flag DIF issues |

**M7 Validity & Analytics**

| Capability            | Description                                               |
| --------------------- | --------------------------------------------------------- |
| Predictive validity   | Diagnostic score vs mock/real exam correlation            |
| Intervention validity | Recommended learning vs random learning score difference  |
| Item drift            | Sudden accuracy change -> possible leak or labeling error |
| A/B experiments       | Test different selection strategies, report formats       |

## 4. Diagnostic Report Restructured — Triple Score

```
+--------------------------------------------------------------+
|                    Vocabulary Diagnostic Report v2             |
+--------------------------------------------------------------+
|                                                              |
|  +-----------------------------------------------------+    |
|  | Coverage Score: 78%                                   |    |
|  | Your mastered words cover 78% of exam word occurrences|    |
|  | ################....  Target: 95%                     |    |
|  |                                                       |    |
|  | Purpose: Prioritize learning                          |    |
|  +-----------------------------------------------------+    |
|                                                              |
|  +-----------------------------------------------------+    |
|  | Mastery Score                                         |    |
|  |                                                       |    |
|  | Recognition  ################..  85%  Confidence: H   |    |
|  | Spelling     ############......  62%  Confidence: M   |    |
|  | Collocation  #########.........  50%  Confidence: M   |    |
|  | Context      ######............  35%  Confidence: L   |    |
|  |                                                       |    |
|  | Purpose: Identify weak skill dimensions               |    |
|  +-----------------------------------------------------+    |
|                                                              |
|  +-----------------------------------------------------+    |
|  | Risk Score: Medium Risk                               |    |
|  |                                                       |    |
|  | Predicted score loss due to vocabulary weakness:       |    |
|  |  Cloze fill  ########..  ~8pts (collocation+polysemy) |    |
|  |  Grammar fill ######....  ~6pts (spelling+morphology) |    |
|  |  Reading      ###.......  ~3pts (familiar-new-meaning)|    |
|  |                                                       |    |
|  | Purpose: Map to exam question types, quantify risk    |    |
|  +-----------------------------------------------------+    |
|                                                              |
|  High-Risk Word List (sorted by score-gain utility)          |
|                                                              |
|  Rank  Word          Import  Mastery  Weak Dim    Utility    |
|  1    effect/affect   S      R:Y W:N  Spell+Disc  *****     |
|  2    require         S      R:Y W:N  Spelling    ****      |
|  3    right(rights)   S      S1:Y S2:N Polysemy   ****      |
|  4    look forward to A      Partial  Phrase      ***       |
|  ...                                                         |
|                                                              |
|  Diagnostic Confidence Note                                  |
|  Based on 52 questions, heuristic diagnosis, not a           |
|  standardized test. Confidence labels reflect measurement    |
|  sufficiency per dimension. Accuracy improves with data.     |
|                                                              |
+--------------------------------------------------------------+
```

## 5. Assessment Flow Revised

### MVP Stage: Semi-Adaptive Fixed Flow

```
Round 1: Breadth Scan (20 questions, ~3 min)
  +-- Sample 5 words from each S/A/B/C tier
  +-- All L1 recognition (English->Chinese 4-choice)
  +-- Goal: quickly draw "mastery vs frequency tier" curve
  +-- Output: approximate user level range

Round 2: Focused Probing (20 questions, ~4 min)
  +-- Focus on "fuzzy zone" (40%-70% mastery tier)
  +-- Mix L1 + L5 (recognition + context)
  +-- S-tier wrong answers get extra confirmation
  +-- Simple branching: 3 consecutive correct -> upgrade, 3 wrong -> downgrade

Round 3: Weakness Confirmation (10 questions, ~2 min, optional)
  +-- Targeted questions on Round 2 weak spots
  +-- Focus on "false familiarity": recognized but can't use in context
  +-- Output: initial weak dimension labels

Total: 40-50 questions, under 9 minutes
```

**Key distinctions from v1**:

- Not called "adaptive testing", called "smart diagnosis"
- Branching logic based on simple rules, doesn't pretend to be IRT
- Explicitly labels confidence, honestly communicates measurement limitations

### Mature Stage (sufficient data): True CAT

```
After 50,000+ responses accumulated:
  -> Item parameters calibrated (difficulty, discrimination)
  -> Switch to 2PL-IRT item selection algorithm
  -> Select by information function maximization
  -> Only NOW can it be called "adaptive testing" with measurement rigor
```

## 6. Item Bank Strategy Revised

### Copyright Risk Mitigation (New)

```
Three-tier item source system:

Tier 1: Original items (full IP, no risk)
  +-- Generated from word attributes + templates, human reviewed

Tier 2: Adapted items (low risk)
  +-- Reference real exam structure, replace specific content
  +-- Preserve test point, change surface form

Tier 3: Real exam slices (requires copyright assessment)
  +-- Tagged with source, year, region
  +-- Used only within authorized scope
  +-- System designed to fall back to Tier 1/2 at any time
```

### Question Types Simplified (MVP)

| Type                      | Tests What            | Exam Scenario           | MVP Priority  |
| ------------------------- | --------------------- | ----------------------- | ------------- |
| L1 English->Chinese       | Recognition           | Reading basics          | **Must have** |
| L5 In-sentence meaning    | Context understanding | Cloze/Reading inference | **Must have** |
| L3 Initial letter fill    | Spelling+production   | Grammar fill            | V2            |
| L4 Collocation completion | Collocation knowledge | Cloze fill              | V2            |
| L2 Audio word selection   | Listening             | Listening               | V3            |
| L6 Synonym discrimination | Fine discrimination   | Cloze/grammar           | V3            |

## 7. Revised Iteration Roadmap

### MVP (4-6 weeks) — Diagnostic Prototype

- Shenzhen zhongkao only
- Import existing freq Excel -> three-dimensional importance scoring
- 40-question semi-adaptive diagnosis (L1 + L5)
- Triple-score diagnostic report (coverage + mastery + risk)
- High-risk word list (sorted by score-gain utility)
- Explicitly labeled "heuristic diagnosis, not standardized test"
- Web only, no mini-program needed yet

### V2 (+6 weeks) — Complete Learning Loop

- Add L3 spelling + L4 collocation question types
- Sense-level annotation (high-frequency polysemous words)
- Spaced repetition scheduling
- Begin accumulating response data for calibration
- Mini-program launch

### V3 (+8 weeks) — Calibration & Expansion

- Calibrate item parameters from accumulated data
- Switch from heuristic to data-driven ability estimation
- Add listening + discrimination question types
- Expand to gaokao vocabulary
- Teacher dashboard with class reports

### V4 (Long-term) — True Adaptive

- Introduce IRT/Rasch model
- True CAT item selection
- Mock exam predictive validity verification
- A/B experiment framework
- Multi-region syllabus adaptation

## 8. Key Design Decisions Comparison

| Decision              | v1                                  | v2 Revised                                      | Reason                                                |
| --------------------- | ----------------------------------- | ----------------------------------------------- | ----------------------------------------------------- |
| Product positioning   | Adaptive testing system             | Diagnostic & prioritization engine              | No calibration data, can't claim CAT                  |
| Core metric           | Single coverage rate                | Coverage + Mastery + Risk scores                | Single score hides critical info                      |
| Word modeling         | Entry-level                         | Sense-level + phrase-level + family graph       | Exam failures are about senses and collocations       |
| Importance scoring    | Frequency x coverage (2D)           | Frequency x coverage x task impact (3D)         | Occurrence location affects score contribution        |
| Mastery model         | Binary (mastered/not)               | Skill dimension x confidence matrix             | Multi-dimensional ability can't compress to one value |
| Measurement evolution | One-step CAT                        | Heuristic -> data-calibrated -> IRT (3 stages)  | Honest about cold start reality                       |
| MVP scope             | Zhongkao + Gaokao, 7 question types | Shenzhen zhongkao only, L1 + L5                 | Focus to go deep                                      |
| Item sources          | Real exam slices primary            | Three-tier system (original -> adapted -> real) | Controllable copyright risk                           |
| New modules           | None                                | Content governance + Calibration + Validity     | System completeness                                   |
| Report attitude       | Implied precision                   | Explicitly labels confidence and limitations    | User trust comes from honesty                         |

---

---

# 中文版

---

# 第一部分：初始策略与架构设计（v1）

## 基于词频统计的单词测评策略

### 一、现有市场方案分析

#### 1.1 主流产品及其测评逻辑

| 产品                             | 测评方式              | 优点         | 不足                         |
| ------------------------------ | ----------------- | ---------- | -------------------------- |
| **百词斩**                        | 图片选义 + 自评"认识/不认识" | 趣味性强，用户粘性高 | 图片联想 ≠ 真正掌握；被动识别为主，无法测产出能力 |
| **墨墨背单词**                      | 基于遗忘曲线的自评复习       | 个性化复习节奏    | 依赖用户自评诚实度；词汇量估算不够精准        |
| **扇贝单词**                       | 选择题 + 拼写 + 例句填空   | 多维度测试      | 题型固定，容易形成答题技巧而非真实掌握        |
| **不背单词**                       | 真实语境例句选义          | 语境理解好      | 测评维度单一（只测识别）               |
| **词汇量测试网站**（如 Test Your Vocab） | 抽样自评 → 统计推断       | 快速估算总量     | 颗粒度粗；与考试大纲脱钩               |

#### 1.2 共性问题

- **与考试大纲脱钩**：多数产品按 CET-4/6 或通用词表分级，未针对中考/高考大纲做精细化对齐
- **频率盲区**：不区分"考了100次的核心词"和"只出现过1次的边缘词"，等权重对待
- **只测识别，不测产出**：绝大多数只考"看英文选中文"，无法评估拼写、搭配、语境运用
- **缺乏诊断性**：告诉你"词汇量5000"，但不告诉你"哪些高频考点词还有漏洞"

### 二、基于词频的测评策略设计

#### 核心理念

> **不是所有单词都值得同等测试。** 考过50次的词和考过1次的词，对备考的价值完全不同。测评应按词频分层，诊断出"高频未掌握"的危险区域。

#### 第一层：词频分层模型

| 层级            | 定义                   | 典型特征                              | 测评权重   |
| ------------- | -------------------- | --------------------------------- | ------ |
| **S级·核心高频**   | 出现频次 Top 10%，且覆盖多套试卷 | `distinct_files` 高，`total_freq` 高 | 必测，反复测 |
| **A级·常见词**    | 频次中上，多数试卷出现          | 稳定出现但不极端                          | 重点测    |
| **B级·中频词**    | 偶尔出现，有一定覆盖面          | 特定题型/话题下出现                        | 抽样测    |
| **C级·低频/边缘词** | 仅出现1-2次或仅在个别试卷       | 可能是超纲或话题特定词                       | 轻量测/选测 |

**关键指标**：不能只看 `total_freq`，必须结合 `distinct_files`（跨试卷覆盖度）。

#### 第二层：多维度测评矩阵

| 维度                  | 测试形式          | 对应考试能力    |
| ------------------- | ------------- | --------- |
| **识别**（receptive）   | 英 → 中选义；听音选词  | 阅读理解、听力   |
| **产出**（productive）  | 中 → 英拼写；首字母填空 | 书面表达、语法填空 |
| **搭配**（collocation） | 常见搭配选择/补全     | 完形填空      |
| **语境**（contextual）  | 在句子中判断词义/用法   | 阅读推断、完形   |

**分层策略**：S级四维全测，A级三维，B级两维，C级仅识别。

#### 第三层：自适应抽样测评

1. **初始测评**（约50-80题，5-8分钟）：按频率层级比例抽样
2. **能力估计**：每层掌握率 + 频率加权 → 考试词汇覆盖率
3. **诊断报告**（核心差异化）：具体的"高频未掌握"词表，按投入产出比排序

#### 第四层：区分中考 vs 高考

| 维度    | 中考           | 高考             |
| ----- | ------------ | -------------- |
| 大纲词汇量 | \~1600（义教课标） | \~3500         |
| 词频分布  | 更集中，核心词反复出现  | 更分散，长尾更长       |
| 测评侧重  | 拼写准确度、基本搭配   | 一词多义、熟词生义、语境推断 |

### 三、差异化价值主张

1. **真题词频驱动**：不是按教材排序，而是"考什么就重点测什么"
2. **诊断而非估算**：精确定位"高投入产出比"的薄弱点
3. **考试覆盖率**：比"词汇量N千"更有行动指导意义
4. **分层测评效率**：高频词深测、低频词浅测

---

## 软件架构设计（v1）

### 总体架构

```
┌─────────────────────────────────────────────────────────┐
│                      用户层                               │
│   ┌──────────┐  ┌──────────┐  ┌───────────────────┐     │
│   │  Web App  │  │ 小程序    │  │ 教师管理后台       │     │
│   └────┬─────┘  └────┬─────┘  └────────┬──────────┘     │
└────────┼──────────────┼────────────────┼────────────────┘
         └──────────────┼────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    API 网关 / BFF 层                      │
└────────────────────────┬────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────┐
│  测评引擎     │ │  题库服务     │ │  学习追踪服务     │
└──────┬───────┘ └──────┬───────┘ └───────┬──────────┘
       │                │                  │
       ▼                ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│                      数据层                               │
│  ┌────────────┐ ┌────────────┐ ┌──────────────────┐     │
│  │ 词频知识库  │ │ 题目库      │ │ 用户学习记录      │     │
│  └────────────┘ └────────────┘ └──────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### 模块 1：数据预处理与词频知识库

每个词条的结构化属性：lemma、frequency\_rank、tier (S/A/B/C)、exam\_scope、pos\_tags、senses（按考频排序）、collocations、exam\_contexts、confusables。

### 模块 2：题库服务

双轨来源：真题原题切片 + 基于词条模板派生

七级题型体系：L1识别 → L2听辨 → L3拼写 → L4搭配 → L5语境 → L6辨析 → L7真题实战

### 模块 3：测评引擎（核心）

三阶段选题：快速定位（10题）→ 精确探测（40题）→ 薄弱点钻探（30题可选）

能力画像：按频率层级 + 按认知深度 + 按话题域 + 考试词汇覆盖率

### 模块 4：学习追踪

每词状态机：未测试 → 首次测试 → 已掌握（需巩固）→ 牢固掌握 / 未掌握 → 学习中 → 待复测

### 技术选型

小程序优先 + Web | Python 后端 | PostgreSQL | Redis 缓存

### 迭代路径

MVP（4-6周）→ V2（+4周加题型）→ V3（+4周加管线）→ V4（远期AI出题）

---

# 第二部分：Codex 评估

## OpenAI Codex 批判性评估

### 总体判断

方案战略上有前景，但作为**诊断+优先级排序系统**更强，作为纯粹的"自适应测评"产品在当前形态下还不够成熟。

最强的想法不是7级题型分类或3阶段流程，而是围绕**预期考试收益**优化而非原始词汇量。这对中高考备考方向正确，因为学生不需要平等掌握所有大纲词汇。

主要弱点是当前设计混淆了三个不同构念而未清晰分离：

- **词汇重要性**：频率/跨卷覆盖度
- **词汇知识**：学生是否掌握这个词
- **考试表现**：能否在语境、干扰项、时间压力下答对

### 1. 策略合理性 — 基本成立，但有缺陷

**优势**：与实际考试暴露对齐；双维度优于单维频率；覆盖率比"词汇量2300"更有行动指导意义

**核心假设缺陷**：

- **频率 ≠ 区分价值**：高频功能词贡献大但诊断价值低
- **出现次数 ≠ 分值影响**：阅读中出现20次 vs 完形关键词出现1次
- **词族处理危险**：decide/decision/decisive 不可互换，过度归并破坏考试真实性
- **掌握是多维的**："已掌握词频率"太粗糙
- **历史频率可能过拟合**

### 2. 架构质量 — 四模块合理，但缺失关键组件

缺少：心理测量校准模块、内容治理模块、实验/分析模块、教师/家长报告层

### 3. 可行性风险 — 最大风险在内容和校准，不在技术

最高风险：题目校准冷启动、自动出题质量、义项级标注、版权风险、听力模块复杂度

**MVP 低估**：4-6周只够做启发式诊断原型

### 4. 测评模型效度 — 最大概念问题

无校准数据时，"自适应"只是智能分支问卷，不是真正 CAT。多维测试打破单维 CAT 假设。频率层级 ≠ 难度。

**"考试覆盖率"**：作为学习KPI好，作为测量分数弱，作为考试预测可能误导。

建议改为三个分数：覆盖分（优先级排序）+ 掌握分（技能能力）+ 风险分（失分概率）

### 5. 市场定位 — 差异化部分真实

竞品已在声称考试导向和记忆科学。真正差异化在于：词级影响力诊断精度 + 语料驱动覆盖率报告 + 中高考地域特化

### 6. 改进建议

1. 重新定位为**考试词汇诊断引擎**
2. 分离三模型：重要性/测量/推荐
3. 缩小MVP：只做一个考试，L1+L5，半自适应
4. 分阶段心理测量：启发式 → 数据校准 → IRT/CAT
5. 增加义项级和短语级建模
6. 用模考成绩做外部效度验证
7. 认真对待版权问题
8. 升级报告维度

### 底线结论

> 概念好，最佳部分是从"词汇量"转向**加权考试效用**。薄弱部分是在没有校准数据前过度声称测量严谨性。当前设计最适合定位为**数据驱动的诊断与优先级排序产品**，尚非统计成熟的自适应测试系统。

---

# 第三部分：修订版架构设计（v2）

> 基于 Codex 评审意见修订。核心转变：从"自适应测试系统"重新定位为**数据驱动的词汇诊断与优先级引擎**，分阶段演进测量严谨性。

## 一、定位修正

|      | v1 原设计       | v2 修订                 |
| ---- | ------------ | --------------------- |
| 定位   | 自适应词汇测试系统    | 考试词汇诊断与优先级引擎          |
| 核心承诺 | 精准测量你的词汇能力   | 告诉你**下一步该背什么**才能最高效提分 |
| 测量态度 | 暗示 CAT 级别严谨性 | 明确标注"启发式诊断"，随数据积累逐步升级 |
| 目标考试 | 中考+高考同时覆盖    | **MVP 只做一个**（深圳中考优先）  |

## 二、三模型分离（核心架构变更）

```
┌─────────────────────────────────────────────────────────────┐
│                    三模型架构                                 │
│                                                             │
│  ┌───────────────┐  ┌───────────────┐  ┌────────────────┐  │
│  │  重要性模型     │  │  掌握度模型     │  │  推荐模型       │  │
│  │  (Importance)  │  │  (Mastery)     │  │  (Recommend)   │  │
│  │                │  │                │  │                │  │
│  │  这个词对考试   │  │  这个学生对     │  │  下一步学什么   │  │
│  │  有多重要？     │  │  这个词掌握     │  │  提分最快？     │  │
│  │                │  │  到什么程度？   │  │                │  │
│  └───────┬───────┘  └───────┬───────┘  └───────┬────────┘  │
│          │                  │                   │           │
│          └──────────┬───────┘                   │           │
│                     ▼                           │           │
│              ┌─────────────┐                    │           │
│              │  诊断报告    │←───────────────────┘           │
│              └─────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

### 重要性模型

三维评分：频率权重 × 覆盖度权重 × 任务影响权重

任务影响权重：完形填空正确答案 1.0 > 语法填空 0.9 > 阅读题干/选项 0.7 > 阅读原文 0.4 > 听力原文 0.3 > 纯文本出现 0.2

词族图谱替代扁平归并：每个派生词独立跟踪，共享族群优先级

义项级标注：如 right 的"正确的"(S级) vs "权利"(A级·熟词生义) vs "右边的"(C级)

### 掌握度模型

技能维度 × 置信度矩阵（识别/拼写/搭配/语境 各自独立估计）

分阶段演进：

- 阶段1（MVP）：启发式规则 + 贝叶斯更新，标注"诊断参考"
- 阶段2（5000+作答后）：数据驱动参数估计
- 阶段3（50000+作答后）：Rasch/2PL 模型，真正 CAT

### 推荐模型

提分效用 = 重要性(w) × (1 - 掌握度(w)) × 可学习性(w)

按提分效用降序 = "下一步该背什么"

## 三、修订后的模块架构（7 模块）

```
┌─────────────────────────────────────────────────────────────┐
│                        用户层                                │
│    ┌──────────┐   ┌──────────┐   ┌──────────────┐          │
│    │ 小程序    │   │ Web App  │   │ 教师管理后台   │          │
│    └────┬─────┘   └────┬─────┘   └──────┬───────┘          │
└─────────┼──────────────┼────────────────┼───────────────────┘
          └──────────────┼────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                       API 层                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
    ┌────────┬───────────┼───────────┬────────────┐
    ▼        ▼           ▼           ▼            ▼
┌──────┐ ┌──────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ M1   │ │ M2   │ │ M3       │ │ M4       │ │ M5       │
│词频   │ │题库   │ │诊断引擎  │ │学习追踪   │ │内容治理   │  ← 新增
│知识库 │ │服务   │ │          │ │          │ │          │
└──┬───┘ └──┬───┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
   │        │          │            │             │
   └────────┴──────┬───┴────────────┘             │
                   ▼                              │
            ┌────────────┐                        │
            │ M6 校准服务 │  ← 新增                │
            └──────┬─────┘                        │
                   │                              │
   ┌───────────────┴──────────────────────────────┘
   ▼
┌─────────────────────────────────────────────────────────────┐
│  M7 效度验证与分析（新增）                                     │
│  模考成绩预测 · A/B 实验 · 题目表现漂移监控                     │
└─────────────────────────────────────────────────────────────┘
```

新增模块：

- **M5 内容治理**：大纲版本管理、试卷来源溯源、版权状态标记、地域标签
- **M6 校准服务**：冷启动接受专家标注 → 批量估计参数 → 实时更新淘汰
- **M7 效度验证**：预测效度、干预效度、题目漂移、A/B 实验

## 四、诊断报告重构——三分制

```
┌──────────────────────────────────────────────────────────────┐
│                    词汇诊断报告 v2                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 覆盖分 (Coverage Score): 78%                         │     │
│  │ 你已掌握的词覆盖了考试中 78% 的词汇出现                │     │
│  │ ████████████████░░░░ 目标: 95%                       │     │
│  │ 用途：排定学习优先级                                  │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 掌握分 (Mastery Score)                               │     │
│  │ 识别能力   ████████████████░░ 85%  置信度: 高         │     │
│  │ 拼写能力   ████████████░░░░░░ 62%  置信度: 中         │     │
│  │ 搭配能力   █████████░░░░░░░░░ 50%  置信度: 中         │     │
│  │ 语境运用   ██████░░░░░░░░░░░░ 35%  置信度: 低         │     │
│  │ 用途：了解薄弱技能维度                                │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 风险分 (Risk Score): 中等风险                         │     │
│  │ 预计因词汇薄弱导致的失分区域：                         │     │
│  │  完形填空  ████████░░  ~8分风险（搭配+一词多义）       │     │
│  │  语法填空  ██████░░░░  ~6分风险（拼写+词形变化）       │     │
│  │  阅读理解  ███░░░░░░░  ~3分风险（熟词生义）           │     │
│  │ 用途：对接考试题型，量化风险                           │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  ⚠️ 高危词清单（按提分效用排序）                               │
│  排名  词条         重要性  当前掌握   薄弱维度    提分效用     │
│  1    effect/affect  S级   识别✓拼写✗  拼写+辨析   ★★★★★     │
│  2    require        S级   识别✓拼写✗  拼写        ★★★★☆     │
│  3    right(权利义)   S级   义项1✓义项2✗ 熟词生义   ★★★★☆     │
│  4    look forward to A级  部分       短语搭配    ★★★☆☆     │
│                                                              │
│  📋 诊断置信度说明                                            │
│  本次测试基于 52 道题目的启发式诊断，非标准化考试。             │
│  置信度标注反映该维度的测量充分程度。                           │
│  随作答数据积累，诊断精度将持续提升。                           │
└──────────────────────────────────────────────────────────────┘
```

## 五、测评流程修订

### MVP 阶段：半自适应固定流程

```
第 1 轮：广度扫描（20 题，约 3 分钟）
  ├── 从 S/A/B/C 各层级各抽 5 词
  ├── 全部用 L1 识别题
  └── 目标：画出"掌握率 vs 频率层级"曲线

第 2 轮：聚焦探测（20 题，约 4 分钟）
  ├── 聚焦"模糊区间"（掌握率 40%-70%）
  ├── 混合 L1 + L5
  └── 简单分支：连续 3 对 → 升级，连续 3 错 → 降级

第 3 轮：薄弱点确认（10 题，约 2 分钟，可选）
  ├── 针对第 2 轮暴露问题集中出题
  └── 重点测"假熟悉"词

总计：40-50 题，9 分钟内完成
```

关键区别：不称"自适应测试"，称"智能诊断"；明确标注置信度和局限。

### 成熟阶段：积累 50000+ 作答记录后切换到 2PL-IRT 真正 CAT

## 六、题库策略修订

### 版权风险应对

三级体系：一级·原创题（无风险）→ 二级·改编题（低风险）→ 三级·真题切片（需评估）

### 题型精简（MVP 只保留 L1 + L5，V2 加 L3/L4，V3 加 L2/L6）

## 七、修订后的迭代路线

| 阶段  | 时间    | 内容                                          |
| --- | ----- | ------------------------------------------- |
| MVP | 4-6 周 | 仅深圳中考；导入词频 → 三维评分；40题半自适应（L1+L5）；三分制报告；Web端 |
| V2  | +6 周  | 加 L3 拼写 + L4 搭配；义项级标注；间隔复习；积累数据；小程序上线       |
| V3  | +8 周  | 数据校准题目参数；听辨+辨析题型；扩展高考；教师端                   |
| V4  | 远期    | IRT/Rasch 模型；真正 CAT；模考效度验证；A/B 实验；多地区适配     |

## 八、关键设计决策对照表

| 决策点    | v1         | v2 修订            | 修订原因          |
| ------ | ---------- | ---------------- | ------------- |
| 产品定位   | 自适应测试系统    | 诊断与优先级引擎         | 无校准数据不应声称 CAT |
| 核心指标   | 单一覆盖率      | 覆盖分+掌握分+风险分      | 单一分数隐藏关键信息    |
| 词汇建模   | 词条级        | 义项级+短语级+词族图谱     | 考试失分在义项和搭配    |
| 重要性评分  | 频率×覆盖度（2D） | 频率×覆盖度×任务影响（3D）  | 出现位置影响分值贡献    |
| 掌握度模型  | 二元（掌握/未掌握） | 技能维度×置信度矩阵       | 多维能力不可压缩为一个值  |
| 测量演进   | 一步到位 CAT   | 启发式→数据校准→IRT 三阶段 | 诚实面对冷启动现实     |
| MVP 范围 | 中考+高考，7级题型 | 仅深圳中考，L1+L5      | 聚焦才能做深        |
| 题目来源   | 真题切片为主     | 三级体系（原创→改编→真题）   | 版权风险可控        |
| 新增模块   | 无          | 内容治理+校准服务+效度验证   | 补齐系统完整性       |
| 报告态度   | 隐含精确       | 明确标注置信度和局限       | 用户信任来自诚实      |

\>> 根据 /Users/victor/projects/gaokao/单词记忆/词汇量测试/[processwordtest.md](http://processwordtest.md) 中的V2版本的思路，结合 /Users/victor/projects/gaokao/单词记忆/词汇量测试/中考单词.xlsx 这个单词表和 /Users/victor/projects/gaokao/单词记忆/词汇量测试/1全国中考真题 中的中考试卷，给我做一个可以发布在gitpage上的针对中考单词进行测试的网页

\>> /Users/victor/projects/gaokao/单词记忆/词汇量测试/中考单词.xlsx和/Users/victor/projects/gaokao/全国中考/5视频制作/words*filled*20260403\_143414.xlsx 中有很多相同的单词，但是有的单词释义有差异，以words*filled*20260403\_143414.xlsx的释义为正确的标准，把中考单词.xlsx中不准确的释义进行修改。对于有修改的地方，在I列标注修改。输出的文件以excel并以时间戳为后缀名。

\>> 以新的/Users/victor/projects/gaokao/单词记忆/词汇量测试/中考单词*20260416*141823.xlsx文件为基础制作词汇测试程序。  Create a vocabulary testing program based on /Users/victor/projects/gaokao/单词记忆/词汇量测试/中考单词20260416141823.xlsx.

\>> 取消‘根据语境，选择划线出单词的中文释义’这种题型。增加一个选项，是：“我不确定”，让用户在拿不准的时候不用乱猜，将该选项作为首选的A选项。

设置每道题的测试时间为最长7秒，7秒内用户没有操作，则表示用户不确定该单词的意义。记录每道题的测试时间，如果一道题，用户使用的时间过长，虽然这个时候用户选对了，但是可能是猜对的，也可能是根据选项的提示而做对的，这个情况下用户的得分应该降低。

\>> 正确答案应该平均分布在BCDE这四个选项中，避免学生猜测。最后的结果，应该默认学生掌握了相对较高频的单词，建议学生从/Users/victor/projects/gaokao/单词记忆/词汇量测试/中考单词*20260416*141823.xlsx这个词频表中大概哪一个位置进行学习后面词频相对更低的单词。

\>> 保留现有的测试框架，缩短整个测试时间为现在的70%

\>> 干扰选项应该是和测试的单词拼写相差不大的单词的意思，且不能一个干扰在已经出现过的测试题中反复出现

\>> 目前的测试分为3个部分，每个部分结束后，等待用户确认准备好了，点击“继续测试”按钮后，再进行测试

\>> 最后的测试报告可以下载PDF版本，且后续的报告上面备注有学生可以采取的下一步计划：“联系土妹获得专属的单词提升计划”

\>> 前面测试前的介绍：“中考词汇智能诊断

基于142套全国中考真题词频 · V2诊断引擎” 应该改的更有煽动性，以提高学生参与度。

\>> 最后应该停留在测试报告页，并鼓励用户下载该报告，并将报告转发到微信朋友圈，记录当下的状态，并准备迈入下一个阶段。内容要有煽动性，影响用户的实际行为。让用户以此为社交货币，在朋友圈内进行传播

\>> 最后下载的测试报告，应该直接倡议用户把该报告发布到微信朋友圈，抖音，小红书上，分享自己当下的学习进度，并让社交网络的朋友监督自己会迈向下一个新高度。报告上应该有本次测试的网址，以便于朋友圈中的其他人也来参与词汇测试。

\>> 将 /Users/victor/projects/gaokao/单词记忆/词汇量测试/tumei321123.jpg

```
中的二维码放到 “生成分享图”按钮后下载的图片中，作为其他用户进行单词测试的入口。也将这个二维码放到PDF报告中，作为其他用户进行单词测试的入口
```

\>> 将“好消息是：系统已经帮你找出了最该背的词，按优先级来，效率翻倍”改成“好消息是：AI已经帮你找出了最该背的词，按优先级来，效率翻倍”，且调整这段文字的显示方式，以便于更清晰地阅读。调整”公开打开=社交监督，让朋友圈见证你迈向下一个高度“的显示方式，以便于更清晰地阅读

\>> 点击生成分享图下载的JPG中，将“中考词汇诊断成绩单”改成“我的2026年中考词汇诊断成绩单”，将图片下方的文字改为：

“扫码加土妹好友，获得你的专属词汇提升计划。

分享道朋友圈/抖音/小红书，让朋友监督你迈向下一个新高度。

这是我的成绩单，你该来挑战我吗？”

\>> 同一个IP地址进行测试时，每次测试的单词应该尽量有差异，避免多次测试遇到重复内容，从而获得了较高的测试结果     Randomize word selection for each vocabulary test session to prevent the same IP address from receiving duplicate words across multiple tests, ensuring test validity and preventing artificially inflated scores.

\>> 不要每次都从“the”这个单词开始测试，默认所有测试者都知道最高频的40个单词的含义

![](./assets/images/clipboard-1776325188601-d8ic-1776325188603-v1i9.png)

![](./assets/images/clipboard-1776325188601-d8ic-1776325188603-v1i9.png)

\>> 我在githubs上建了一个库：git remote add origin [git\@github.com](mailto:git@github.com):GoodeSam/zhongkao-vocabulary-size-test.git

git branch -M main

git push -u origin main

将测试的界面在这里生成一个Gitpage，仅在该仓库上展示词汇测试的界面，不展示其他内容.  Configure GitHub Pages for the zhongkao-vocabulary-size-test repository to display only the vocabulary test interface, hiding other repository content.

\>> 将风险分完形填空，语法填空，阅读理解 这三项下面的失分变成失分百分比

\>> 按下“联系土妹，获得你的专属词汇提升计划”以后，应该弹出添加土妹的微信二维码（ /Users/victor/projects/gaokao/单词记忆/词汇量测试/tumei321123.jpg），并弹出文字：”保存图片，添加二维码“。微信二维码图片可以下载

现在点击‘开始测试诊断’按钮后无反应

\>> 将报告中的“蓝色高亮行为建议起始位置（词频表文件：中考单词*20260416*141823.xlsx 第 2128 行）”这一句去掉

\>> “下载完整报告”这里点击后，可以同时生成PDF和一张长图

对于“基于词汇掌握情况预估的得分率：”这一项下面生成的图片，增加一个饼状图，展示完形填空，阅读理解，语法填空的得分率

测试前要求学生选择目前的年级：选项是小学1-6年级，初中1-3年级，高中1-3年级。要求学生输入自己的姓名。

测试后的报告中增加输出：姓名 同学，你的词汇量约为多少词，相当于哪个年级的单词水平

生成的图片和PDF的标题，都要有 姓名 同学

点击“开始诊断测试”按钮后无反应

年级选项中增加一个“其他”

最后的结果要输出测出来的单词所在的结果是否和所在的年级匹配，是领先，还是落后



生成的选项中，不要有完全一样的选项

基于词汇掌握情况预估的得分率，这一项下面的环状图是四个环状图，分别是阅读理解，完形填空，语法填空，平均的词汇的占比，每一个生成一个环状图

报告中生成一项：你超过你所在的年级【多少比例】的同学，出于前百分之多少的位置。然后用金字塔图形来显示这个比例。



将这句话：“根据诊断结果，我们估计你已基本掌握词频表前 **647** 个高频词（共3098词）。 建议你从词频表大约第 **647** 位开始，向后学习频率更低、你尚未掌握的单词。”改为：”根据诊断结果，我们估计你已基本掌握中考词频表前 **647** 个高频词（共3098词）。 建议你从词频表大约第 **647** 位开始，向后学习你尚未掌握的单词，在学习的过程中优先学习频率相对更高的单词。同时也建议你向前复习你已经掌握的单词，由于高频的单词你会掌握得更好，因此在复习过程中你需要优先复习频率相对较低的单词”



齐普夫法则



在测量页面的首页，和测量结果的前面，介绍本次测量方式的科学性，目标，和结果。参考并修改下面文字，使读者更相信这个测试，以及后续的单词课程对于中考提分的有效性和科学性。参考文字如下：“

齐普夫法则指出，词汇的重要性极不平均。少量高频词覆盖了大量真实语言输入，而大量低频词出现得很少。对142份中考真题进行统计，排名第一的单词the出现了22828次，而排名第1000的单词shake只出现55次，排名2000的单词greenhouse出现了12次，排名3000的单词virtual只出现了2次。

因此单词掌握不仅是越多越好，更重要的是需要彻底掌握高频词汇。

同时，教材和课标中的单词，都和中考真题并不完全匹配。常常导致总背的单词没考，考的单词没背的情况。

因此，在有限的时间内，尽可能多地掌握中考真题中的高频词汇，是中考词汇提升的关键。

为了帮助学员找到自己目前的单词掌握情况，我们特意对近年来的中考真题词汇金星了词频统计，并开发了此单词测量程序，再根据测量后的结果，开发了一套科学有效的单词突击方法。

”



将每个单词测试的时间有7秒提升至8秒。将“142套真题告诉你答案 — 6分钟精准定位你的词汇盲区”改成”142套中考真题告诉你答案 — 5分钟精准定位你的词汇盲区“”



将“下载完整报告”按钮生成的内容由PDF变成一张长图，这张长图的内容有测试结束界面的所有内容，在最后还有报告中的下面内容：“

把这份报告晒出去，让朋友们监督你的进步

发到微信朋友圈、抖⾳或⼩红书，记录你此刻的词汇⽔平。

公开承诺是最有效的学习动⼒——让你的社交圈⻅证你迈向下⼀个⾼度。

扫码两件事：

1. 加⼟妹好友，获得专属词汇提升计划
2. 邀请朋友来测，看谁词汇量更强

”

将“排名第 1,000 的单词 **shake** 只出现了 **55** 次
排名第 2,000 的单词 **greenhouse** 出现了 **12** 次”改成”排名第 1,000 的单词 **shake** 出现了 **55** 次
排名第 2,000 的单词 **greenhouse** 只出现了 **12** 次“

将“同时，教材和课标中的单词与中考真题**并不完全匹配**——常常出现"总背的单词没考、考的单词没背"的情况。在有限的备考时间内，**尽可能多地掌握中考真题中的高频词汇**，才是词汇提分的关键。”调整为：”同时，教材和课标中的单词与中考真题**并不完全匹配**——常常出现”背过的单词没考、要考的单词没背"的情况。在有限的备考时间内，**尽可能多地掌握中考真题中的高频词汇**，才是词汇提分的关键。“

将**数据基础下面的 “\~6min**

预计测试时长”改成 **“\~5min**

预计测试时长“”



在年级排名预估

你超过了**高中1年级**约 1% 的同学，处于前 99% 的位置

这种情况下最后的图片中“你”这行文字展示不完整，解决这个问题

下载完整报告长图的图片不够清晰，解决这个问题

将“生成分享图”按钮生成的内容也是上面的长图



使“已有 1337 名同学完成测试并分享了自己的学习进度”中的数字始终会随时时间考后而有缓慢增长，不至于减少

下载完整报告长图的图片中：上面的：中考词汇只能诊断报告和最下面的：把这份报告晒出去 这两个部分的内容清晰，中间部分的内容不够清晰，解决中间部分不够清晰这个问题

现在数字基于当前时间计算（以2024年4月15日为起

  点，每秒时增长约0.7至2.3人随机，最终显示的人数是整数不能有小数），同一时刻刷新始终显示相同数字，且随

  时间推移只增不减。

BCDE的选项中，不能有完全相同的选项



下载完整报告长图的图片中：上面的：中考词汇只能诊断报告和最下面的：把这份报告晒出去 这两个部分的内容清晰，中间部分的内容不够清晰，这个问题依然存在



点击“下载完整报告长图”生成的图片的清晰度分布。  
我观察到：

1. 上部内容清晰；
2. 中间部分明显更模糊，文字边缘发虚；
3. 下部清晰度又恢复正常。
   也就是说，这是一个**中间区域清晰度低于上下区域的局部问题**。  
   请求Codex判断可能原因，并给出检测或修复思路。

现在是测试阶段，将整个测试的三轮缩减到只有15题，后面测试完成后，再恢复到35道题

**>> “年级排名预估：的金字塔图片，最上端的尖端缺失掉了，补上最上端的小三角形。最下端的25%是一个矩形，导致整体看起来不像一个常见的金字塔图片。对最下端的也进行修改。**



**在首页的“**&#x4E3A;什么这个测试和你的中考成绩直接相关？

语言学中的\*\*齐普夫法则（Zipf's Law）\*\*揭示了一个关键事实：**词汇的重要性极不平均**。少量高频词覆盖了绝大部分真实语言输入，而大量低频词出现极少。**”文字的下面一行，生成一个关于**齐普夫法则在语言学统计中的示意图，用示意图来表示高频单词的重要性远远大于低频单词



点击“下载完整报告长图”生成的图片的清晰度分布。
我观察到：

1. 上部内容清晰；
2. 中间部分明显更模糊，文字边缘发虚；
3. 下部清晰度又恢复正常。

 也就是说，这是一个**中间区域清晰度低于上下区域的局部问题**。  
更改目前的长图生成模式，换成其他模式来生成长图



点击“下载完整报告长图”后，依然保留在分享前的界面



将首页中的“为什么这个测试和你的中考成绩直接相关？

语言学中的\*\*齐普夫法则（Zipf's Law）\*\*揭示了一个关键事实：**词汇的重要性极不平均**。少量高频词覆盖了绝大部分真实语言输入，而大量低频词出现极少。”文字下面的图片，替换为：

/Users/victor/projects/中考词汇量测试/齐普夫法则：中考真题  

  词频分布.png

\>> 询问Codex在测试完成后生成的长图中，应该包含哪些内容，从而使这个测试显得更加科学，学生也更愿意来学习我们的单词课，同时更愿意把长图分享到朋友圈中

当选择了错误的选项后，会不会比选择A选项“我不确定”有降权

将年级词汇量分布的金字塔图片的每一层都换成不同的颜色，不要使用灰色。将前百分之几十的文字放在图形的左侧。将“你在这里”放在图形的右侧

网页中显示“我们对

我们对 **142 份全国中考真题**进行了词频统计，数据印证了这一规律：”这里有两个“我们对”，重复了，删除掉第一个

将输出的“**持续巩固：定期复测 + 查漏补缺**  
每 2 周复测一次，追踪 Coverage 从 55% → 95% 的进度，精准补缺。”中的Coverage 改成汉语的“覆盖率”

将测试完成的：学生姓名和年级用更醒目的字体表示。将**年级词汇量分布下面的金字塔的宽度变成现在的1.6倍。且去掉图片中的“年级词汇量分布”这几个字**

将词汇量→年级映射调整为：                                           

                                                              

  ┌────────┬───────────┐                                      

  │ 词汇量 │   等级    │                                      

  ├────────┼───────────┤                                      

  │ <150   │ 小学1年级 │

  ├────────┼───────────┤

  │ 150+   │ 小学2年级 │

  ├────────┼───────────┤                                      

  │ 250+   │ 小学3年级 │

  ├────────┼───────────┤                                      

  │ 350+   │ 小学4年级 │                                    

  ├────────┼───────────┤                                      

  │ 500+   │ 小学5年级 │

  ├────────┼───────────┤

  │ 700+   │ 小学6年级 │

  ├────────┼───────────┤                                      

  │ 900+  │ 初中1年级 │

  ├────────┼───────────┤                                      

  │ 1300+  │ 初中2年级 │                                    

  ├────────┼───────────┤

  │ 1800+  │ 初中3年级 │

  ├────────┼───────────┤

  │ 2400+  │ 高中1年级 │

  ├────────┼───────────┤                                      

  │ 2800+  │ 高中2年级 │

  ├────────┼───────────┤                                      

  │ 3500+  │ 高中3年级 │                                    

  └────────┴───────────┘









然后就是如何对学员上课的问题。这样就需要快速读完这几本书，可以用claude帮我读书。





\>> 检查 /Users/victor/projects/中考词汇量测试/中考单词*20260416*141823.xlsx 中有没有哪些单词在C列的内容完全一样，如果有的话，在I列输出有差异的音标符号和中文释义，以便于区分单词。结果以时间戳另存为excel命名输出.  Identify duplicate words in column C of /Users/victor/projects/中考词汇量测试/中考单词20260416141823.xlsx. Output the differing phonetic symbols and Chinese definitions in column I to distinguish duplicates. Save the results to a new Excel file with a timestamp in the filename.

\>> 对于/Users/victor/projects/中考词汇量测试/重复释义分析*20260418*091502上传.xlsx中的单词有相同的汉语释义，在测试其中一个单词时，不要给出另外一个单词的释义作为干扰项，这样受试者会困惑。   

```python
When generating test questions for words in                 
  /Users/victor/projects/中考词汇量测试/重复释义分析20        
  260418091502上传.xlsx, do not use definitions from 
  words with identical Chinese meanings as distractor         
  options.
```

```python
When generating test questions for words in /Users/victor/projects/中考词汇量测试/重复释义分析20260418091502上传.xlsx, do not use definitions from words with identical Chinese meanings as distractor options.

```

```python
When generating test questions for words in /Users/victor/projects/中考词汇量测试/重复释义分析20260418091502上传.xlsx, do not use definitions from words with identical Chinese meanings as distractor options.

```

\>> 由于本测试代码仅仅是针对中考测试设计的，单词库仅有3098个单词，对于测试单词数超过2800的参与者，该测试不能反应其单词水平。后面的测试报告应该明确指出：由于其水平爆表，本测试不能反应其真实水平，应该参与土妹设计的高中单词测试。

