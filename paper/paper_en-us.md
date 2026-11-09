# Hierarchical text generation on commodity hardware: a partial replication of Next-Concept Modeling and what eleven days of CPU taught us

**Renato de Souza e Silva**

Independent research — November 2026

---

## Abstract

Next-Concept Modeling (NCM), recently proposed by the Hazy Research group, replaces token-by-token autoregressive generation with a coarse-to-fine hierarchy of discrete codes, promising parallel generation ten to thirty times faster than comparable diffusion language models. This paper reports an independent attempt to reproduce that recipe end-to-end on commodity hardware — an ordinary laptop, no GPU, 23–32 GB of RAM — over eleven days and twenty logged hypothesis–measurement–verdict rounds. The results are mixed, and instructive precisely because of that. The paradigm's structural speedup was measured and confirms the original paper's central hypothesis at small scale: per-sample speedup grew from about 1.14× at 60M parameters to 2.27× at 471M and up to 3.22× on TinyStories, with no hardware acceleration beyond CPU threads. The failure analysis, in turn, produced three contributions the original paper does not document: evidence that under-trained residual VQ-VAE codes factor reconstruction error rather than predictability; the discovery of an initialization bug that silently re-randomized the pretrained tokenizer and invalidated four full rounds of verdicts; and, after the fix, confirmation that the official codes are learnable by a small generator — the finest level converged below the reference model's own loss — while the global code stayed near-uniform, characterizing the coherence bottleneck as a scale and training-budget problem rather than an architectural one. We conclude the technique is viable to iterate locally, and that the decisive experiment — the full official recipe — requires roughly two thousand times more compute: trivial for a cluster, impossible for a laptop. We document rejected hypotheses, negative numbers, and process in full, in the belief that well-measured failures save more laboratory time than poorly explained successes.

**Keywords:** language models; parallel text generation; residual vector quantization; replication; inference efficiency.

## Resumo (Portuguese abstract)

Ver a versão PT-BR (`paper_pt-br.md`) para o resumo em língua portuguesa conforme ABNT NBR 6028:2021.

---

## 1 Introduction

This work's motivation was born not in NCM but in measured frustration. In a previous project we built a complete runtime capable of executing the official BF16 checkpoint of Qwen3-Coder-30B-A3B-Instruct locally, without quantization, on a 16 GB machine: memory mapping, expert prefetch, resident cache, exact speculative decoding, persistent server. It worked — and the conclusion was a number: between 0.6 and 0.9 tokens per second. The whole model fits, runs exactly, and produces real output, but the latency makes it impractical as an interactive assistant. The bottleneck was not a lack of engineering; it was structural. Forty-eight sequential layers, one token at a time, tens of gigabytes of weights crossing the bus per generated token.

It was in this context that the Next-Concept Modeling publication (Kumbong et al., 2026) caught our attention. Rather than fighting sequentiality with system tricks — speculative drafts, verification trees, distilled drafters — NCM proposes changing the structure of generation itself: a multiscale residual VQ-VAE converts text into a stack of discrete code levels, and a transformer generates one level at a time, predicting all codes within that level in parallel. The paper reports up to 31× faster generation than the strongest diffusion baseline at comparable quality — and, more importantly for us, the cost signature is different by construction: K passes per level instead of L passes per token, with K typically much smaller than L.

The question guiding this study is simple to state and expensive to answer: **can one iterate the NCM recipe on commodity hardware, without a GPU, and learn enough about the paradigm to decide whether it is worth investing in scale?** Eleven days and twenty rounds later, the answer is "yes, with caveats only those who measured can see" — and this paper is the honest account of what those caveats are, what they cost, and where the real boundary of consumer hardware lies.

Three scope warnings before the results. First: this is not a novel-results paper in the usual sense — it is a partial replication report with failure analysis, the format we consider most useful for those holding a cluster and wanting to know where the terrain is mined before spending GPUs on it. Second: every reported number was measured on CPU; relative speedups between matched arms are the honest metric here, not absolute latencies. Third: we document failures with the same care as successes — four entire rounds of this study produced verdicts we later discovered were contaminated by a bug of ours, and we tell that story because it is instructive.

## 2 Background

### 2.1 The cost of sequentiality

Autoregressive models generate one token per forward pass. For a depth-d transformer, the per-token cost is a full sweep; for a coding assistant answering in blocks of hundreds of tokens, latency accumulates linearly and, on GPU-less hardware, every pass contends for memory bandwidth with the entire system. The acceleration literature attacks this cost along four main lines: (i) speculative decoding, where a drafter model proposes tokens the target verifies in parallel (Leviathan, Kalai & Matias, 2023; Miao et al., 2023 — see Hu et al., 2025 for a full survey); (ii) multiple decoding heads trained to predict future positions (Cai et al., 2024 — Medusa); (iii) consistency distillation training the model itself to converge in parallel iterations (Kou et al., 2024 — CLLMs; Fu et al., 2025 — Jacobi Forcing); and (iv) text diffusion models, which refine the whole sequence through denoising steps (Ren et al., 2026). All of them preserve the autoregressive architecture or replace it with iteration over the full sequence — none changes the atomic unit of generation.

### 2.2 Next-Concept Modeling

NCM (Kumbong et al., 2026) changes the unit. A multiscale residual VQ-VAE (van den Oord, Vinyals & Kavukcuoglu, 2017; Zeghidour et al., 2021) encodes each sequence into K levels of discrete codes: the earliest, short levels capture the broad shape; the later, longer ones refine detail. A transformer then learns to generate the hierarchy level by level — each level predicted in a single parallel pass, conditioned on the cumulative reconstruction of previous levels. It is the transposition to text of the next-scale prediction VAR (Tian et al., 2024) demonstrated in images, with text-specific adaptations: one codebook per level, nested dropout to force coarse levels to carry semantics, and conditioning corruption for robustness to prediction error. The paper reports, at ~88M–600M parameter scale, quality competitive with or superior to the best diffusion models at 10–31× lower sampling cost and 4–5× fewer training passes — and public checkpoints for LM1B, OpenWebText, XSum, WMT14, Sudoku and TinyStories, which we used directly.

### 2.3 The "last mile" problem

One point the literature makes clear — and which our study confirmed the expensive way — is that hierarchical representation schemes do not eliminate the cost of getting back to tokens. Meta's LCM (Large Concept Models, 2024) operates on per-sentence SONAR embeddings and resolves the last mile with a full seq2seq decoder per generated concept. In our local measurement, Mimir-1.6B (a mirror of the LCM weights) produced 0.07 concepts per second on CPU — the last mile exists and has capacity of its own. Traditional speculative decoding solves it differently: it proposes tokens directly and the target verifies (Hu et al., 2025). The empirical question for NCM is: how much capacity does the codes→text path require, and where does it live — in the codebook, the VQ decoder, the generator?

## 3 Materials and methods

### 3.1 Hardware and budget

Three general-purpose machines took part in the study; no GPU was used at any stage:

- Main laptop: i5-1235U (2 P-cores + 8 E-cores, 12 threads), 23 GB RAM, NVMe — runs 001–005;
- Secondary CPU: Ryzen 7 4750U (16 threads), 32 GB RAM — runs 006 onward;
- Auxiliary 30B inference: same secondary CPU, with a llama.cpp server.

All third-party training and benchmarking ran inside an isolated container without network (`podman --network none`), per local lab policy. Training runs ranged from 30 minutes to 14.5 hours; total budget was roughly 300 CPU-hours across eleven consecutive days.

### 3.2 Experimental line

The study was organized as a sequence of hypothesis→measurement→verdict rounds, each with versioned artifacts (JSON metrics, checkpoints, samples). Without repeating the full account in `TIMELINE.md`, the phases were:

**Phase A — native replication at small scale (runs 001–004).** Own pipeline: mixed corpus of 3,181–4,665 records (Python standard library + local teacher generations), 8,192-token BPE, 6-level residual VQ-VAE, ~62M-parameter LevelGenerator warm-started from Qwen2.5-0.5B, parameter- and data-matched AR baseline.

**Phase B — tokenizer-free pyramid path (runs 005–012).** We derived a 6-level pyramid codebook directly from Qwen2.5-0.5B embeddings — training no VQ-VAE at all — and investigated the code→token last mile via three routes: nearest-embedding, code-path likelihood, and hierarchical-head speculative decoding.

**Phase C — TinyStories domain end-to-end (runs 014–016).** TinyStories corpus (Eldan & Li, 2023), 8k BPE, own VQ-VAE, 69M NCM vs 67M AR; level-depth ablation, codebook taper, and intra-level chunked causal generation.

**Phase D — official tokenizer (runs 017–020).** Published official VQ-VAE (12 levels, uniform 4,096-entry codebooks) + precomputed official codes over ~177k training segments; 64M LevelGenerator, U(0,1) corruption recipe with level-0 skip; 3,000 steps × batch 16.

### 3.3 The bug that taught more than the verdicts

Mandatory record: in runs 014–017, `LevelGenerator.__init__` registered the VQ-VAE module before `self.apply(_init_weights)` — the recursive apply re-randomized every Linear/Embedding of the pretrained tokenizer (codebooks, decoder, projections). The effect: training conditioned on random embeddings and decoding used a random decoder, producing word salad for any codes, good or bad. Four rounds of verdicts — including the "fine levels are unpredictable" finding that looked like the deepest scientific result — were confounded. The bug was found by blind review, fixed (registering the VQ-VAE after the apply), and verified by weight inspection in a container. Run 019/020 is the first clean test.

## 4 Results

### 4.1 The hierarchical speedup is real and grows with scale

The original paper's central metric — fewer sequential passes per sample — reproduced at three independent scales, always against a parameter- and data-matched AR baseline, same machine, same threads:

| Scale | NCM | AR | Per-sample speedup |
|---|---|---|---|
| ~62–68M (mixed corpus) | 0.535 s | 0.609 s | ×1.05–1.34 (~×2.9 per token-equivalent) |
| ~471–479M (same corpus) | 4.98 s | 11.29 s | **×2.27** |
| ~69M TinyStories | 0.22 s | 0.62 s | **×2.82** |
| ~69M TinyStories (taper) | — | — | **×3.22** |

The paper's hypothesis — the advantage grows with size, because the AR pays ~127 sequential passes of a bigger network while NCM pays 6 — was confirmed on our hardware: the ratio rose from ~1.1× to ~2.3× when quadrupling parameters, and held >3× on TinyStories. Structurally, NCM generates a sample in 6–12 total passes; the matched AR, in ~127. The advantage is not kernel optimization — it is fewer sequential trips through the network, and should therefore be preserved or amplified on any hardware.

### 4.2 Quality does not come for free: the anatomy of failure

Against that speed, three independent fix rounds failed to produce coherent text in the local budget regime — and the analysis of why is, in our view, the study's most useful scientific finding.

**First rejected hypothesis — codebook size.** Tapering the fine-level codebooks (2048→[2048,1024,512,256,128]) improved reconstruction (held CE 0.654, 100% utilization) and kept the speedup (×3.22), but samples stayed incoherent. **Second — more steps.** Retraining from 3,000 to 5,500 steps lifted neither the loss plateau (~6.1) nor the samples out of salad. **Third — intra-level conditioning.** Generating each level in causal chunks (same-level neighbors visible, MaskGIT/VAR style) cost extra passes (×1.22 only) and did not cure quality.

The decisive measurement is the ground-truth frontier: conditioning the generator on real codes at 90% of neighbor positions, fine-level cross-entropy does not drop — l3: 6.21→6.22; l4: 5.51→5.51; l5: 4.70→4.73, against uniform baselines of 6.24/5.55/4.85. **Fine residual codes are essentially independent of each other and of coarse context** — I(code_i; code_j | coarse) ≈ 0. The interpretation is structural: each VQ level's residual is, by construction, what earlier levels failed to encode — the highest-entropy remainder. Residual quantization factors by reconstruction error, not by predictability; asking a generator to predict those residuals in parallel asks it to model precisely the part of the signal with the least contextual structure.

### 4.3 The official tokenizer changes the diagnosis — and the bug changes everything again

With the official VQ-VAE (trained by the authors with nested dropout and the full recipe), the picture partially inverted: fine levels proved learnable — held l11 = 6.81 against uniform 8.32 already in the contaminated run — and, after the init-bug fix, converged to held l11 = 2.68, **below the ~2.75 of the reference generator itself**. Total held loss fell from 7.55 (bugged run) to 5.82 (clean run).

Post-fix samples are qualitatively something else: correct TinyStories vocabulary, character names, narrative formulae ("Once upon a time", "From that day on", "happily ever after"), dialogue punctuation — real story-shaped structure. The remaining defect is twofold and specific: degenerate bigram repetition ("was was", "Lucy Lucy", "again again again") and absence of narrative arc — concatenated fragments without global causality. The metric explaining both: level 0 — the single 4,096-class code carrying the entire story's identity — stayed at held l0 ≈ 8.22, indistinguishable from uniform 8.32. Without l0, fine levels refine the details of a story that was never decided; the result is story-in-form-without-narrative.

### 4.4 What was proven, what was refuted, what remains open

Consolidating twenty rounds into three honest columns:

**Proven.** (i) The full NCM pipeline — tokenizer, hierarchical generator, decoding — iterates on ordinary CPUs. (ii) The structural speedup is real, reproducible, and grows with scale: ×1.14→×2.27→×3.22. (iii) Official codes are learnable by a small generator at fine levels — the recipe requires no laboratory secret. (iv) A pyramid codebook derived from pretrained embeddings is viable without training a VQ-VAE (32.1% top-1 token recovery).

**Refuted.** (i) Nearest-embedding, code-path likelihood, and hierarchical spec-decode do not close the concept→token last mile (0/40, median rank ~43k). (ii) More steps, smaller codebooks, and intra-level conditioning do not cure unpredictable residuals — the bottleneck was representational, not budgetary, for our local tokenizers. (iii) Per-position pyramid decoding does not beat AR — the gain comes from parallelism across levels, not within them.

**Open.** (i) Level 0 — even at 14.5h/3,000 steps the global code never left uniform; the official recipe uses ~50,000 steps × batch 2,048 (~2,000× our budget) and 12 layers. (ii) Degenerate repetition in generation — plausibly exposure bias of fine levels conditioned on an unlearned l0. (iii) Whether structure-aligned hierarchy — ASTs for code, for instance — yields more predictable codes than reconstruction residuals: our main future interest, and the original motivation (local coding assistant) remains unanswered.

## 5 Discussion

### 5.1 What this study adds to the original paper

The NCM paper demonstrates the recipe on clusters with tens of thousands of steps and thousand-sequence batches. This study adds the reading from the other end of the curve: what each component of the recipe is worth when the budget is three orders of magnitude smaller. Three contributions stand out for whoever will scale it.

**Initialization is a trust boundary.** The bug that invalidated four rounds is of an insidious class: `apply(_init_weights)` recursively re-initializing a pretrained submodule. No assert, no loss explosion — apparently healthy training conditioned on random embeddings, and samples that "looked like" a paradigm failure. In any replication that loads a frozen tokenizer inside a larger module, verifying weight preservation by inspection (checksums before/after construction) should be gate zero — it costs one line of code and cost us four runs.

**The low-budget regime separates what is structural from what is scale-bought.** The speedup curve confirmed the theoretical proposition; the quality curve revealed where the recipe pays its costs. The embedding-derived pyramid codebook — our methodological contribution absent from the original — eliminates the pipeline's most expensive stage (VQ-VAE training) and still produces codes whose fine portion is irredeemably unpredictable. The official tokenizer, trained with the full recipe, produces fine levels more learnable than ours — yet l0 still resisted 3,000×16 examples. The joint reading: the NCM recipe transfers cost from sampling to training and to the tokenizer; whoever replicates must budget all three, not just the generator.

**The concept→token last mile is a representation problem, not a decoding problem.** Three independent decoding methods failed identically (0/40, 0 accepts) not from method weakness but because the code distribution does not concentrate mass on the right token — median rank ~43k of 152k. The LCM comparison is instructive: SONAR "solves" the last mile because it pays a full transformer per sentence; our nearest-embedding projection is free and recovers 32%. For a coding assistant, where the wrong token is a program that does not compile, this is the point of the entire project that most deserves engineering.

### 5.2 Limitations of this study

We are obliged to list what this work cannot claim. (i) We did not reproduce the full official recipe — neither the ~50k×2048 training steps, nor corpus size, nor 12-layer depth; what we measured is the low-budget regime, not the paper's regime. (ii) The main corpus (TinyStories) is deliberately child-level; nothing here speaks of code, the domain that motivated the study — transfer to ASTs and programming structure is grounded conjecture, not a result. (iii) Samples judged by human reading of 8 examples per configuration; we did not apply the TinyStories paper's GPT-4 evaluation protocol or reference perplexity. (iv) Two machines, distinct CPUs — absolute speedups vary ±20% between them; the inter-arm ratio is stable. (v) The "l0 unlearned" verdict holds for 3,000×16; it may be purely budgetary — and probably is, given l0 began leaving uniform (8.36→7.97) in the last thousand steps.

### 5.3 The code question, revisited

The study began by asking whether NCM could enable a fast local coding assistant. The honest answer after eleven days: **we still do not know — but we now know where to measure.** The paradigm delivers its half of the promise (structural speed) in the regime we could run; the other half (quality) has an identified, quantified bottleneck (l0 + budget). For code, one hypothesis this paper leaves formulated but untested: NCM's hierarchy fits naturally with AST hierarchy — generating syntactic structure first and tokens later is exactly the coarse→fine paradigm, and syntax-tree-derived codes may have the predictability that reconstruction residuals lack. It is the direction we most recommend to whoever has a cluster and an interest.

## 6 Conclusion and projections

This work partially reproduced, on commodity hardware and an eleven-day CPU budget, the Next-Concept Modeling pipeline. The consolidated verdict: **the paradigm delivers real, scalable speed already in miniature; text quality requires the full training recipe that only makes sense on a cluster.** Between what was measured and what remains to measure lies a decisive experiment with a known name and budget: running the official recipe (~50k steps, batch 2k, 12 layers) over the official tokenizer — roughly 2,000× our budget, i.e., hours on a decent GPU node.

We project, based on what we measured: (i) a learned l0 at full budget should unlock narrative arc, reducing repetition degeneration through real fine-level conditioning; (ii) relative speedup should hold or grow on GPU, because the advantage is pass count, not kernel throughput; (iii) the code last mile will likely require a learned token head conditioned on codes — our `codes-head` variant already showed codes carry signal the feature alone does not deliver.

Recommended next steps, in cost-benefit order for a cluster: full official-recipe replication; training an NCM tokenizer on a code corpus; comparing residual codes vs AST-aligned codes; and the real-assistant experiment (tool-use response latency). Each departs directly from this repository's artifacts.

A final note on method: four of our twenty rounds produced wrong verdicts because of one misplaced line of code — and fixing that line turned "the paradigm fails" into "the paradigm needs scale". In an era when replications are rare and replication bugs invisible, documenting this felt as important as any speedup.

## References

CAI, T. et al. Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads. *arXiv*, n. 2401.10774, 2024.

CHANG, H. et al. MaskGIT: Masked Generative Image Transformer. *arXiv*, n. 2202.04200, 2022. (CVPR 2022)

ELDAN, R.; LI, Y. TinyStories: How Small Can Language Models Be and Still Speak Coherent English? *arXiv*, n. 2305.07759, 2023.

FU, C. et al. Fast and Accurate Causal Parallel Decoding using Jacobi Forcing. *arXiv*, n. 2512.14681, 2025.

GGERGANOV, G. et al. llama.cpp. Free software. Available at: https://github.com/ggml-org/llama.cpp. Accessed: Nov. 9, 2026.

HU, Y. et al. Speculative Decoding and Beyond: An In-Depth Survey of Techniques. *arXiv*, n. 2502.19732, 2025.

HUI, B. et al. Qwen2.5-Coder Technical Report. *arXiv*, n. 2409.12186, 2024.

KOU, S. et al. CLLMs: Consistency Large Language Models. *arXiv*, n. 2403.00835, 2024.

KUMBONG, H. et al. Next-Concept Modeling: Fast, Parallel Coarse-to-Fine Text Generation. *arXiv*, n. 2610.14162, 2026. Checkpoints: https://huggingface.co/hazyresearch/ncm-checkpoints.

KUMAR, A. Residual Vector Quantization for KV Cache Compression in Large Language Models. *arXiv*, n. 2410.15704, 2024.

LEE, D. et al. Autoregressive Image Generation using Residual Quantization. *arXiv*, n. 2203.01941, 2022. (CVPR 2022 — RQ-Transformer)

LEVIATHAN, Y.; KALAI, M.; MATIAS, Y. Fast Inference from Transformers via Speculative Decoding. *arXiv*, n. 2211.17192, 2023. (ICML 2023)

LI, L. et al. Sub-MoE: Efficient Mixture-of-Expert LLMs Compression via Subspace Expert Merging. *arXiv*, n. 2506.23266, 2025.

LI, Y. et al. EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty. *arXiv*, n. 2401.15077, 2024. (ICML 2024)

LI, Z. et al. Merge, Then Compress: Demystify Efficient SMoE with Hints from Its Routing Policy. *arXiv*, n. 2310.01334, 2023. (ICLR 2024 — MC-SMoE)

META AI. Large Concept Models: Language Modeling in a Sentence Representation Space. *arXiv*, n. 2412.08821, 2024.

MIAO, X. et al. SpecInfer: Accelerating Generative Large Language Model Serving with Tree-based Speculative Inference and Verification. *arXiv*, n. 2305.09781, 2023.

QWEN TEAM. Qwen3 Technical Report. *arXiv*, n. 2505.09388, 2025.

REN, H. et al. Edit-Based Refinement for Parallel Masked Diffusion Language Models. *arXiv*, n. 2605.09603, 2026.

TIAN, K. et al. Visual Autoregressive Modeling: Scalable Image Generation via Next-Scale Prediction. *arXiv*, n. 2404.02905, 2024. (NeurIPS 2024 — VAR)

VAN DEN OORD, A.; VINYALS, O.; KAVUKCUOGLU, K. Neural Discrete Representation Learning. *arXiv*, n. 1711.00937, 2017. (NeurIPS 2017 — VQ-VAE)

ZEGHIDOUR, N. et al. SoundStream: An End-to-End Neural Audio Codec. *arXiv*, n. 2107.03312, 2021.

ZHOU, Y. et al. DistillSpec: Improving Speculative Decoding via Knowledge Distillation. *arXiv*, n. 2310.08461, 2024. (ICLR 2024)

---

*License: CC BY-NC 4.0 — exclusively for academic and research use. Full artifacts and history in the associated repository.*
