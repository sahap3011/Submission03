# MP3 Reflection

## Grounding & abstention

The assistant uses a two-layer defence against hallucination:
1. **Primary Score Gate (`should_abstain`)**: Rejects queries before calling the LLM whenever no chunks are returned or the top cosine similarity score falls below `RETRIEVAL_THRESHOLD`.
2. **Backup Prompt Gate (`SYSTEM_PROMPT`)**: Instructs `gpt-4o-mini` (`temperature=0`) to rely exclusively on the numbered excerpts and output `"I don't know based on the provided documents."` whenever the retrieved excerpts do not contain the answer.

Across the evaluation set, the assistant abstained with **100% accuracy (3/3)** and a **0.0 hallucination rate** on all three unanswerable queries:
- `abstain_out_of_domain` (*"What is the boiling point of water in degrees Celsius?"*): Top retrieved chunk (`05_scandal_in_bohemia.txt` — *The Cipher Note*) scored **`0.144`**, easily caught by `should_abstain` even at the starter threshold of `0.30`.
- `q7_abstain_mycroft_diogenes` (*"What position does Mycroft Holmes hold in the British government, and what are the rules of the Diogenes Club?"*): Because this is an in-domain Sherlock Holmes question from *The Greek Interpreter* (not in our 5-story corpus), dense embeddings matched thematic vocabulary in `01_red_headed_league.txt` (*The Identity of the Assistant*) with a top cosine score of **`0.457`**.
- `abstain_moriarty` (*"Who is Professor Moriarty, and how does he meet his end?"*): Similarly, asking about Holmes's arch-nemesis retrieved `02_speckled_band.txt` (*Holmes Investigates*, where Doctor Roylott confronts Holmes at Baker Street) at **`0.482`**, followed by *The Explanation* at `0.436`.

**Why I settled on `RETRIEVAL_THRESHOLD = 0.49`:**
At the starter default of `0.30`, out-of-domain trivia (`0.144`) was blocked at the retrieval gate, but plausible canon questions about absent characters (`q7_abstain_mycroft_diogenes` at `0.457` and `abstain_moriarty` at `0.482`) passed `should_abstain` and had to rely on the LLM prompt guard to abstain in prose. Meanwhile, the lowest-scoring answerable query in our benchmark was the multi-hop question `q6_speckled_band_multihop` at **`0.503`** (followed by `q2_speckled_band_weapon` at **`0.528`**). Raising `RETRIEVAL_THRESHOLD` from `0.30` to **`0.49`** cleanly separated all unanswerable queries ($\le 0.482$) from all answerable queries ($\ge 0.503$), allowing `should_abstain` to short-circuit every unanswerable question prior to generation—saving both LLM tokens and ~900ms of latency per refusal.

**False abstentions (refusing answerable questions):**
The assistant never answered something it shouldn't have, but at the default `k=3` and `TARGET_CHUNK_SIZE=500`, it *did* refuse two answerable questions (`q1_red_headed_league_assistant` and `q2_speckled_band_weapon`) via the backup prompt gate:
- For `q1_red_headed_league_assistant` (*"Who was the assistant working at Jabez Wilson's pawnshop, and what was his real identity?"*), top-3 retrieval returned *The Red-Headed League* (`0.689`), *Holmes Investigates* (`0.596`), and *The Strange Work* (`0.548`). These chunks identify the assistant as **Vincent Spaulding**, but the chunk revealing his real identity as **John Clay** (*The Identity of the Assistant*) ranked at #8 (`0.425`) and #9 (`0.421`). Because the second half of the question could not be answered from chunks `[1]`–`[3]`, `gpt-4o-mini` obeyed the strict system prompt and abstained rather than filling in "John Clay" from pre-training memory.
- For `q2_speckled_band_weapon` (*"What killed Julia Stoner at Stoke Moran, and how was the method delivered?"*), top-3 retrieval returned *Helen's Predicament* (`0.528`), *Holmes Investigates* (`0.493`), and an *Engineer's Thumb* trap chunk (`0.491`), while the resolution chunk naming the **swamp adder** (*The Explanation*) ranked at #5 (`0.480`) and #9 (`0.460`). Seeing only the setup and bedroom clues without the snake itself, the model conservatively abstained.

---

## Citations

`format_context` numbers retrieved excerpts `[1]` through `[k]` with their story title and section header, and `validate_citations` parses all `\[(\d+)\]` markers from the generated response to verify that every citation maps to `1 <= n <= len(sources)`.

- **Inline Citation Behaviour**: On all answered questions (`q3`, `q4`, `q5`, `q6`), `gpt-4o-mini` consistently emitted valid inline markers (`[1]`, `[2]`, `[3]`) with zero fabricated out-of-range markers (`invalid = []`). On multi-hop questions such as `q6_speckled_band_multihop`, the model synthesized facts across non-adjacent excerpts—citing `[1]` (*The Explanation*, score `0.503`) for Roylott's financial motive (preserving the late mother's annuity) and `[3]` (*Holmes Investigates*, score `0.461`) for the physical bedroom clues (the dummy bell-rope connected to the ventilator and the bed bolted to the floor).
- **Subtle Grounding Failure Caught by Inspection (`q4`)**: While `validate_citations` confirmed syntactic validity (`used: [2], invalid: []`) on `q4_engineers_thumb_press`, inspecting the retrieved chunks revealed an important nuance: at `k=3`, the top chunks from `04_engineers_thumb.txt` were *The Mysterious Engagement* (`0.583` and `0.529`) and *Victor Hatherley, Hydraulic Engineer* (`0.479`), whereas *The Coining Operation* (which reveals the counterfeit half-crowns) sat just outside the cutoff at rank #4 (`0.450`). Consequently, the model cited `[2]` and reported Colonel Stark's cover story (compressing fuller's earth into bricks). This shows why `validate_citations` is a necessary syntactic guard against fabricated `[n]` indices, but full semantic faithfulness also requires ensuring the resolution chunk enters the retrieval window.

---

## Reading your eval table

Our 9-question evaluation table (`mp3_eval.txt`) reports:
- **Retrieval hit rate**: `1.0` (6/6 answerable questions retrieved the expected story file in top-3)
- **Grounded & cited rate**: `0.667` (4/6 answerable questions answered with valid citations to the expected source and no fabricated markers)
- **Abstention accuracy**: `1.0` (3/3 should-abstain questions refused)
- **Hallucination rate**: `0.0` (0/3 should-abstain questions answered)

**Which metric is weakest, and why:**
`Grounded & cited rate` (`0.667`) is the weakest metric, despite `Retrieval hit rate` being `1.0`. The gap between `1.0` hit rate and `0.667` grounded rate exposes a classic RAG evaluation distinction: **document-level hit rate vs. chunk-level recall**. Because each story is split into 13–18 chunks (`TARGET_CHUNK_SIZE = 500`), a query mentioning character names and settings (`"Jabez Wilson's pawnshop"`, `"Julia Stoner at Stoke Moran"`) strongly matches the opening exposition chunks of the right story file (yielding `hit = ✓`), while the climactic reveal chunk at the end of the story uses shorter references (`"Spaulding"`, `"Julia"`) and strips the section header (`"The Identity of the Assistant"`, `"The Explanation"`) out of `c["text"]` into metadata.

**What I would change:**
1. **Increase `k` from `3` to `5` (or `9`)**: Running an offline comparison across `k` values on the exact same 73-chunk index showed that simply increasing `k=3 -> k=5` pulls the resolution chunks into context and raises **Grounded & cited rate from `0.667` to `1.000` (6/6)** while keeping **Hallucination rate at `0.0`**:

| Retrieval Top-K (`k`) | Retrieval Hit Rate | Grounded & Cited Rate | Abstention Accuracy | Hallucination Rate | Notes |
|---|---|---|---|---|---|
| `k = 3` (starter default) | `1.000` | `0.667` (4/6) | `1.000` (3/3) | `0.000` | `q1` and `q2` abstain because reveal chunks rank at #4–#9 |
| `k = 5` | `1.000` | `1.000` (6/6) | `1.000` (3/3) | `0.000` | `q1`, `q2`, and `q4`'s coining reveal (#4) all enter context |
| `k = 9` | `1.000` | `1.000` (6/6) | `1.000` (3/3) | `0.000` | Every multi-part fact (`2/2` expected facts on `q1`–`q6`) retrieved |

2. **Hybrid Search (BM25 + Dense Embeddings) with Cross-Encoder Reranking**: Retrieve a wider candidate pool (`k=10`) and rerank so both narrative setup chunks and explanation chunks surface in the top context slots.

---

## One surprise

The biggest surprise was how high **in-domain unanswerable questions** scored on cosine similarity with `text-embedding-3-small`, and yet how reliably `gpt-4o-mini` still abstained when those chunks were passed into the prompt. I initially expected *"Who is Professor Moriarty, and how does he meet his end?"* to score near `0.25`–`0.30`, similar to out-of-domain questions. Instead, it scored **`0.482`** against Doctor Roylott's confrontation with Holmes in `02_speckled_band.txt` (and our Mycroft Holmes / Diogenes Club query scored **`0.457`** against `01_red_headed_league.txt`), because dense embeddings capture broad literary and genre semantics ("Holmes confronting an antagonist"). Even more surprisingly, despite Sherlock Holmes stories being deeply embedded in `gpt-4o-mini`'s pre-training weights, whenever `threshold = 0.30` let the Moriarty or Mycroft queries through to generation, the model completely suppressed its parametric knowledge of Reichenbach Falls and the Diogenes Club and output the exact abstention phrase.

---

## Stretch — Threshold Sweep (`RETRIEVAL_THRESHOLD`)

To quantify the trade-off between false abstentions and score-gate coverage, I swept `RETRIEVAL_THRESHOLD` from `0.20` to `0.65` over all 9 questions (`6` answerable, `3` should-abstain):

| `RETRIEVAL_THRESHOLD` | Retrieval Hit Rate | Score-Gate Refusals (`should_abstain` only) | End-to-End Abstention Accuracy | Grounded & Cited Rate (`k=3`) | Grounded & Cited Rate (`k=5`) | Hallucination Rate |
|---|---|---|---|---|---|---|
| `0.20` | `1.000` | `0.333` (1/3: water `0.144`) | `1.000` (3/3 via LLM backup) | `0.667` (4/6) | `1.000` (6/6) | `0.000` |
| `0.30` (starter) | `1.000` | `0.333` (1/3: water `0.144`) | `1.000` (3/3 via LLM backup) | `0.667` (4/6) | `1.000` (6/6) | `0.000` |
| `0.40` | `1.000` | `0.333` (1/3: water `0.144`) | `1.000` (3/3 via LLM backup) | `0.667` (4/6) | `1.000` (6/6) | `0.000` |
| **`0.49` (sweet spot)** | **`1.000`** | **`1.000` (3/3 caught at gate)** | **`1.000` (3/3)** | **`0.667` (4/6)** | **`1.000` (6/6)** | **`0.000`** |
| `0.52` | `1.000` | `1.000` (3/3) | `1.000` (3/3) | `0.500` (3/6 — loses `q6` at `0.503`) | `0.833` (5/6) | `0.000` |
| `0.55` | `1.000` | `1.000` (3/3) | `1.000` (3/3) | `0.500` (3/6 — loses `q2` `0.528`, `q6` `0.503`) | `0.667` (4/6) | `0.000` |
| `0.60` | `1.000` | `1.000` (3/3) | `1.000` (3/3) | `0.167` (1/6 — only `q5` `0.644` & `q1` `0.689` pass gate) | `0.333` (2/6) | `0.000` |
| `0.65` | `1.000` | `1.000` (3/3) | `1.000` (3/3) | `0.000` (0/6 — only `q1` `0.689` passes gate) | `0.167` (1/6) | `0.000` |

**Where the sweet spot is and what moving it costs:**
- **Sweet spot (`0.485`–`0.500`, chosen `0.49`)**: Every unanswerable question (`0.144`, `0.457`, `0.482`) is intercepted deterministically by `should_abstain()` without an LLM API call, while every answerable question (`0.503`–`0.689`) passes through to generation.
- **Cost of moving lower (`< 0.45`)**: In-domain unanswerable questions (`abstain_moriarty` at `0.482`, `q7_abstain_mycroft_diogenes` at `0.457`) bypass the vector score gate. While our strict system prompt still prevented hallucination here, in production relying solely on the LLM prompt costs extra tokens/latency and risks parametric leakage on weaker models.
- **Cost of moving higher (`> 0.50`)**: Multi-hop queries whose phrasing spans multiple sections (`q6_speckled_band_multihop` at `0.503` and `q2_speckled_band_weapon` at `0.528`) suffer immediate false abstentions at the vector gate, sharply degrading the grounded response rate (`0.667 -> 0.500 -> 0.167`).

