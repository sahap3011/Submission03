# MP3 · 2–3 Minute Demo Walkthrough & Transcript

Use this script to record or present your 2–3 minute live demo (`python mp3_grounded_qa.py ask`).

## 1. Run Offline Self-Test (~20 sec)

```bash
cd "Week 12_ Graded Mini Project Documents/AI-RAG_MP3_GroundedQA/learner"
python mp3_grounded_qa.py selftest
```

**Expected Output:**
```text
  ✓ load_corpus reads the corpus (5 docs)
  ✓ chunk_document splits doc 0 into chunks (13)
  ✓ format_context numbers chunks [1]/[2]
  ✓ validate_citations accepts real markers
  ✓ validate_citations flags a fabricated marker
  ✓ should_abstain: low score → abstain
  ✓ should_abstain: good score → answer
  ✓ should_abstain: no chunks → abstain
  ✓ evaluate: hit rate 1/2 answerable
  ✓ evaluate: grounded rate 1/2 answerable
  ✓ evaluate: abstention accuracy 1/1
  ✓ evaluate: hallucination rate 0/1
  ✓ format_eval_table renders a table

✓ selftest passed
```

---

## 2. Interactive Cited Q&A + Abstention (`ask`, ~90 sec)

```bash
python mp3_grounded_qa.py ask
```

### Demo Question 1 — Single-Story Grounded & Cited Answer
**Ask:**
> `Who stole the Countess of Morcar's Blue Carbuncle, who was his accomplice, and how did the gem end up inside a Christmas goose?`

**Live Output:**
```text
James Ryder stole the Countess of Morcar's Blue Carbuncle with the help of the Countess's maid, Catherine Cusack. Ryder, fearing a search, fled to his sister's house in Brixton, where he pushed the stone down the throat of one of her geese. The goose was later sold, and the wrong goose ended up being purchased by Henry Baker [3].

  Sources:
    [3] The Adventure of the Blue Carbuncle — James Ryder Confesses  (score 0.582)
  Latency: 5026ms
```

### Demo Question 2 — Multi-Hop Grounded & Cited Answer (Multiple Citations)
**Ask:**
> `What financial motive drove Doctor Grimesby Roylott's plot against his stepdaughters, and what physical clues in the bedrooms showed how he recalled the swamp adder?`

**Live Output:**
```text
Doctor Grimesby Roylott's financial motive for his plot against his stepdaughters was to kill them for their share of their late mother's annuity, which would have transferred to them on marriage [1]. The physical clues in the bedrooms that showed how he recalled the swamp adder included the dummy bell-rope that connected to the ventilator opening into Doctor Roylott's bedroom, rather than to the outside air, and the fact that the bed was bolted to the floor, preventing the victim from moving out of range [3].

  Sources:
    [1] The Adventure of the Speckled Band — The Explanation  (score 0.503)
    [3] The Adventure of the Speckled Band — Holmes Investigates  (score 0.461)
  Latency: 5116ms
```

### Demo Question 3 — Trustworthy Abstention ("I don't know")
**Ask:**
> `Who is Professor Moriarty, and how does he meet his end?`

**Live Output:**
```text
I don't know based on the provided documents.

  (abstained — no supporting excerpt)
  Latency: 614ms
```

---

## 3. Show Evaluation Table (`eval`, ~30 sec)

```bash
python mp3_grounded_qa.py eval
```

**Live Output (`mp3_eval.txt`):**
```text
  id                              type     hit   abstained  grounded  ok
  --------------------------------------------------------------------------
  q1_red_headed_league_assistant  answer   ✓     ✓          ✗         ✗
  q2_speckled_band_weapon         answer   ✓     ✓          ✗         ✗
  q3_scandal_bohemia_adler        answer   ✓     ✗          ✓         ✓
  q4_engineers_thumb_press        answer   ✓     ✗          ✓         ✓
  abstain_moriarty                abstain  -     ✓          -         ✓
  abstain_out_of_domain           abstain  -     ✓          -         ✓
  q5_blue_carbuncle_theft         answer   ✓     ✗          ✓         ✓
  q6_speckled_band_multihop       answer   ✓     ✗          ✓         ✓
  q7_abstain_mycroft_diogenes     abstain  -     ✓          -         ✓
  --------------------------------------------------------------------------

  Retrieval hit rate    : 1.0   (6 answerable questions)
  Grounded & cited rate : 0.667   (KPI: grounded response rate)
  Abstention accuracy   : 1.0   (3 should-abstain questions)
  Hallucination rate    : 0.0   (KPI: hallucination rate)
```

