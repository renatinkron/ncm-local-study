# Hierarchical text generation on commodity hardware: a partial replication of Next-Concept Modeling and what eleven days of CPU taught us

**Renato de Souza e Silva**

Independent research — November 2026

---

## Abstract

Next-Concept Modeling (NCM), recently proposed by the Hazy Research group, replaces token-by-token autoregressive generation with a coarse-to-fine hierarchy of discrete codes, promising parallel generation ten to thirty times faster than comparable diffusion language models. This paper reports an independent attempt to reproduce that recipe end-to-end on commodity hardware — an ordinary laptop, no GPU, 23–32 GB of RAM — over eleven days and twenty logged hypothesis–measurement–verdict rounds. The results are mixed, and instructive precisely because of that. The paradigm's structural speedup was measured and confirms the original paper's central hypothesis at small scale: per-sample speedup grew from about 1.14× at ~69M parameters to 2.27× at ~479M and up to 3.22× on TinyStories, with no hardware acceleration beyond CPU threads. The failure analysis, in turn, produced three contributions the original paper does not document: evidence that under-trained residual VQ-VAE codes factor reconstruction error rather than predictability; the discovery of an initialization bug that silently re-randomized the pretrained tokenizer and compromised the quality verdicts of nearly every round loading a pretrained tokenizer; and, after the fix, confirmation that the official codes are learnable by a small generator — the finest level converged to a held-out loss of 2.79, essentially at the reference generator's level (~2.75, approximate-protocol comparison) — while the global code stayed near-uniform, characterizing the coherence bottleneck as a scale and training-budget problem rather than an architectural one. We conclude the technique is viable to iterate locally, and that the decisive experiment — the full official recipe — requires roughly two thousand times more compute: trivial for a cluster, impossible for a laptop. We document rejected hypotheses, negative numbers, and process in full, in the belief that well-measured failures save more laboratory time than poorly explained successes.

**Keywords:** language models; parallel text generation; residual vector quantization; replication; inference efficiency.

## Resumo (Portuguese abstract)

Ver a versão PT-BR (`paper_pt-br.md`) para o resumo em língua portuguesa conforme ABNT NBR 6028:2021.

---

## 1 Introduction

This work's motivation was born not in NCM but in measured frustration. In a previous project we built a complete runtime capable of executing the official BF16 checkpoint of Qwen3-Coder-30B-A3B-Instruct (Qwen Team, 2025) locally, without quantization, on a 16 GB machine — work that joins the literature on efficient MoE serving and compression (Li, P. et al., 2023; Li, L. et al., 2025): memory mapping, expert prefetch, resident cache, exact speculative decoding, persistent server. It worked — and the conclusion was a number: between 0.6 and 0.9 tokens per second. The whole model fits, runs exactly, and produces real output, but the latency makes it impractical as an interactive assistant. The bottleneck was not a lack of engineering; it was structural. Forty-eight sequential layers, one token at a time, tens of gigabytes of weights crossing the bus per generated token.

It was in this context that the Next-Concept Modeling publication (Kumbong et al., 2026) caught our attention. Rather than fighting sequentiality with system tricks — speculative drafts, verification trees, distilled drafters — NCM proposes changing the structure of generation itself: a multiscale residual VQ-VAE converts text into a stack of discrete code levels, and a transformer generates one level at a time, predicting all codes within that level in parallel. The paper reports up to 31× faster generation than the strongest diffusion baseline at comparable quality — and, more importantly for us, the cost signature is different by construction: K passes per level instead of L passes per token, with K typically much smaller than L.

The question guiding this study is simple to state and expensive to answer: **can one iterate the NCM recipe on commodity hardware, without a GPU, and learn enough about the paradigm to decide whether it is worth investing in scale?** Eleven days and twenty rounds later, the answer is "yes, with caveats only those who measured can see" — and this paper is the honest account of what those caveats are, what they cost, and where the real boundary of consumer hardware lies.

Three scope warnings before the results. First: this is not a novel-results paper in the usual sense — it is a partial replication report with failure analysis, the format we consider most useful for those holding a cluster and wanting to know where the terrain is mined before spending GPUs on it. Second: every reported number was measured on CPU; relative speedups between matched arms are the honest metric here, not absolute latencies. Third: we document failures with the same care as successes — nearly all pretrained-tokenizer rounds of this study produced quality verdicts we later discovered were contaminated by a bug of ours, and we tell that story because it is instructive.

## 2 Background

### 2.1 The cost of sequentiality

Autoregressive models generate one token per forward pass. For a depth-d transformer, the per-token cost is a full sweep; for a coding assistant answering in blocks of hundreds of tokens, latency accumulates linearly and, on GPU-less hardware, every pass contends for memory bandwidth with the entire system. The acceleration literature attacks this cost along four main lines: (i) speculative decoding, where a drafter model proposes tokens the target verifies in parallel (Leviathan, Kalai & Matias, 2023; Miao et al., 2023 — see Hu et al., 2025 for a full survey); (ii) multiple decoding heads trained to predict future positions (Cai et al., 2024 — Medusa); (iii) consistency distillation training the model itself to converge in parallel iterations (Kou et al., 2024 — CLLMs; Hu, L. et al., 2025 — Jacobi Forcing); and (iv) text diffusion models, which refine the whole sequence through denoising steps (Ren et al., 2026). All of them preserve the autoregressive architecture or replace it with iteration over the full sequence — none changes the atomic unit of generation.

### 2.2 Next-Concept Modeling

NCM (Kumbong et al., 2026) changes the unit. A multiscale residual VQ-VAE (van den Oord, Vinyals & Kavukcuoglu, 2017; Zeghidour et al., 2021; Kumar, 2024) encodes each sequence into K levels of discrete codes: the earliest, short levels capture the broad shape; the later, longer ones refine detail. A transformer then learns to generate the hierarchy level by level — each level predicted in a single parallel pass, conditioned on the cumulative reconstruction of previous levels. It is the transposition to text of the next-scale prediction VAR (Tian et al., 2024) demonstrated in images, with text-specific adaptations: one codebook per level, nested dropout to force coarse levels to carry semantics, and conditioning corruption for robustness to prediction error. The paper reports, at ~88M–600M parameter scale, quality competitive with or superior to the best diffusion models at 10–31× lower sampling cost and 4–5× fewer training passes — and public checkpoints for LM1B, OpenWebText, XSum, WMT14, Sudoku and TinyStories, which we used directly.

### 2.3 The "last mile" problem

One point the literature makes clear — and which our study confirmed the expensive way — is that hierarchical representation schemes do not eliminate the cost of getting back to tokens. Meta's LCM (Meta AI, 2024) operates on per-sentence SONAR embeddings and resolves the last mile with a full seq2seq decoder per generated concept. In our local measurement, Mimir-1.6B (a mirror of the LCM weights) produced 0.07 concepts per second on CPU — the last mile exists and demands generative capacity — and latency — of its own. Traditional speculative decoding solves it differently: it proposes tokens directly and the target verifies (Hu et al., 2025). The empirical question for NCM is: how much capacity does the codes→text path require, and where does it live — in the codebook, the VQ decoder, the generator?

## 3 Materials and methods

### 3.1 Hardware and budget

Two general-purpose machines took part in the study; no GPU was used at any stage:

- Main laptop: i5-1235U (2 P-cores + 8 E-cores, 12 threads), 23 GB RAM, NVMe — runs 001–005;
- Secondary CPU: Ryzen 7 4750U (16 threads), 32 GB RAM — runs 006 onward;
- The secondary CPU also hosted the auxiliary 30B inference via a llama.cpp server (Ggerganov et al.).

All third-party training and benchmarking ran inside an isolated container without network (`podman --network none`), per local lab policy. Training runs ranged from 30 minutes to 14.5 hours; total budget was roughly 300 CPU-hours across eleven consecutive days. Training used AdamW (lr 3e-4), batch 16 and seed 0 unless otherwise noted; generators are 4-layer transformers (~62–69M parameters), except the scale experiment at 24 layers (~471–479M parameters).

### 3.2 Experimental line

The study was organized as a sequence of hypothesis→measurement→verdict rounds, each with versioned artifacts (JSON metrics, checkpoints, samples). Without repeating the full account in `TIMELINE.md`, the phases were:

**Phase A — native replication at small scale (runs 001–004).** Own pipeline: mixed corpus of 3,181–4,665 records (Python standard library + local teacher generations), 8,192-token BPE, 6-level residual VQ-VAE, ~68M-parameter LevelGenerator warm-started from Qwen2.5-0.5B, parameter- and data-matched AR baseline.

**Phase B — tokenizer-free pyramid path (runs 005–012).** We derived a 6-level pyramid codebook directly from Qwen2.5-0.5B embeddings (Yang et al., 2024; Hui et al., 2024), — training no VQ-VAE at all — and investigated the code→token last mile via three routes: nearest-embedding, code-path likelihood, and hierarchical-head speculative decoding — a design guided by the target-distribution draft training of DistillSpec (Zhou et al., 2024), tree-based verification (Miao et al., 2023 — SpecInfer), feature-space autoregression (Li, Y. et al., 2024 — EAGLE), and residual-quantization generation (Lee et al., 2022 — RQ-Transformer).

**Phase C — TinyStories domain end-to-end (runs 014–016).** TinyStories corpus (Eldan & Li, 2023), 8k BPE, own VQ-VAE, 69M NCM vs 67M AR; level-depth ablation, codebook taper, and intra-level chunked causal generation.

**Phase D — official tokenizer (runs 017–020).** Published official VQ-VAE (12 levels, uniform 4,096-entry codebooks) + precomputed official codes over ~177k training segments; 64M-parameter LevelGenerator, U(0,1) corruption recipe with level-0 skip; 3,000 steps × batch 16.

*Numbering note:* run ids are not contiguous — 013 and 018 were infrastructure/monitoring rounds with no associated experiment.

### 3.3 The bug that taught more than the verdicts

Mandatory record: in every round that loaded a pretrained VQ-VAE inside the generator — phases A (001–004), C (014–016) and D (017) — `LevelGenerator.__init__` registered the VQ-VAE module before `self.apply(_init_weights)` — the recursive apply re-randomized every Linear/Embedding of the pretrained tokenizer (codebooks, decoder, projections). The effect: training conditioned on random embeddings and decoding used a random decoder, producing word salad for any codes, good or bad; the bug existed since the generator's first version and was fixed only in run 018. All generator quality verdicts in those rounds — including the "fine levels are unpredictable" finding that looked like the deepest scientific result — were compromised. (Speed verdicts are unaffected: generation time is structural and independent of the loaded weights; phase B, which does not use this generator, was also uncontaminated.) The bug was found by blind review, fixed (registering the VQ-VAE after the apply), and verified by weight inspection in a container. Run 019/020 is the first clean quality test.

## 4 Results

### 4.1 The hierarchical speedup is real and grows with scale

The original paper's central metric — fewer sequential passes per sample — reproduced at three scales, always against a parameter- and data-matched AR baseline, same machine, same threads:

Table 1 — Per-sample latency, NCM vs matched AR (8 samples, temp. 0.8, top-k 40)

| Scale / domain | Machine | NCM | AR | Per-sample speedup |
|---|---|---|---|---|
| ~67–69M parameters (mixed corpus) | i5-1235U, 12 threads | 0.535 s | 0.609 s | ×1.05–1.34 (~×3.1 per token-equivalent) |
| ~471–479M parameters (same corpus) | i5-1235U, 12 threads | 4.98 s | 11.29 s | **×2.27** |
| ~69M parameters (TinyStories) | Ryzen 4750U, 16 threads | 0.22 s | 0.62 s | **×2.82** |
| ~69M parameters (TinyStories, taper) | Ryzen 4750U, 16 threads | — | — | **×3.22** |

The paper's hypothesis — the advantage grows with size, because the AR pays ~127 sequential passes of a bigger network while NCM pays 6 — was confirmed on our hardware: the ratio rose from ~1.1× to ~2.3× when multiplying parameters by ~7, and held >3× on TinyStories — with the caveat that rows 3–4 were measured on a different machine and domain than rows 1–2, and the AR arm showed within-machine drift of 0.617–0.978 s across runs; we read >3× ratios as an upper bound. Structurally, NCM generates a sample in 6 level passes in that configuration; the matched AR, in ~127 token steps. The advantage is not kernel optimization — it is fewer sequential trips through the network.

### 4.2 Quality does not come for free: the anatomy of failure

Against that speed, three independent fix rounds failed to produce coherent text in the local budget regime — and the analysis of why is, in our view, the study's most useful scientific finding.

**First rejected hypothesis — codebook size.** Tapering the fine-level codebooks (2048→[2048,1024,512,256,128]) raised fine-codebook utilization to 100% at the cost of worse reconstruction (held CE 0.654 vs the base codebook's 0.152) and kept the speedup (×3.22), but samples stayed incoherent. **Second — more steps.** Retraining from 3,000 to 5,500 steps lifted neither the loss plateau (~6.1) nor the samples out of salad. **Third — intra-level conditioning.** Generating each level in causal chunks (same-level neighbors visible, MaskGIT/VAR style — Chang et al., 2022; Tian et al., 2024) cost extra passes (speedup reduced to ×1.22) and did not cure quality.

The decisive measurement is the ground-truth frontier: conditioning the generator on real codes at up to ~88–90% of neighbor positions (tested boundaries 87.5–89.8%), fine-level cross-entropy does not drop — l3: 6.20→6.22; l4: 5.51→5.51; l5: 4.70→4.73, against uniform baselines of 6.24/5.55/4.85. **Fine residual codes behave as essentially independent of each other and of coarse context** — I(code_i; code_j | coarse) ≈ 0. The interpretation is structural: each VQ level's residual is, by construction, what earlier levels failed to encode — the highest-entropy remainder. Residual quantization factors by reconstruction error, not by predictability; asking a generator to predict those residuals in parallel asks it to model precisely the part of the signal with the least contextual structure.

*Methodological caveat:* these measurements were taken in the still-contaminated init-bug regime (§3.3) — a generator trained to ignore conditioning would produce exactly this flat response. We report the result because the structural reading remains the best available explanation, but it **has not been revalidated in the clean regime** — indicative, not conclusive evidence.

### 4.3 The official tokenizer changes the diagnosis — and the bug changes everything again

With the official VQ-VAE (trained by the authors with nested dropout and the full recipe), the picture partially inverted: fine levels proved learnable — held l11 = 6.81 against uniform 8.32 already in the contaminated run — and, after the init-bug fix, converged to held l11 = 2.79, **essentially at the reference generator's level (~2.75, an approximate-protocol comparison)**. Total held loss fell from 7.55 (bugged run) to 5.82 (clean run); the loss figures ship in `metrics_gen_ncm_{official,v2}_final.jsonl`.

Post-fix samples are qualitatively something else: correct TinyStories vocabulary, character names, narrative formulae ("Once upon a time", "From that day on", "happily ever after"), dialogue punctuation — real story-shaped structure. The remaining defect is twofold and specific: degenerate bigram repetition ("was was", "Lucy Lucy", "again again again") and absence of narrative arc — concatenated fragments without global causality. Two metrics explain the two defects through distinct mechanisms: (i) level 0 — the single 4,096-class code carrying the whole story's identity — stayed at held l0 ≈ 8.22, indistinguishable from uniform 8.32 (without l0, fine levels refine a story that was never decided); (ii) the local stutter is the classic signature of conditional independence in parallel decoding — predicting all of a level's codes in one pass without seeing freshly generated neighbors favors repeated bigrams. The result is story-in-form-without-narrative.

Generation cost in this configuration: 1.306 s/sample over 12 level passes — **slower than the matched AR** (0.617 s/sample on the same machine), because each fine-level pass processes sequences up to 641 positions. The ~×3 speedup of §4.1 refers to our 6-level configuration over T = 128; in the official 12-level configuration, per-pass cost grows enough to invert the sign on CPU. The latency advantage reported by the original paper was measured at a different scale/hardware and does not automatically transfer here.

### 4.4 The inference recipe outweighs level 0

A reread of the official evaluation code (`eval_shard.py`) exposed two protocol — not architecture — divergences between our sampling and the authors': (i) `predict_level0: false` — l0 is not predicted by the generator; it is drawn from an empirical unigram distribution stored in the checkpoint; (ii) full multinomial sampling (temperature 1.0, no top-k or top-p), while we used temperature 0.8 with top-40. We implemented all three l0 variants (predicted, uniform, unigram) and ported the upstream decode penalties (`frequency_penalty`, `adjacency_penalty`), with bit-exact backward compatibility in the original mode. Over 8 samples per configuration, we measured the adjacent-duplicate word rate (dup_rate) and the distinct-word ratio:

| Sampling configuration | dup_rate | distinct |
|---|---|---|
| predicted l0, T = 0.8, top-40 (v2 protocol) | 21.7% | 0.63 |
| uniform l0, T = 0.8, top-40 | 18.8% | 0.62 |
| unigram l0, T = 0.8, top-40 | 22.0% | 0.55 |
| unigram l0, T = 1.0, no top-k (official recipe) | **11.0%** | 0.66 |
| unigram l0 + frequency_penalty = 5 | 16.3% | 0.64 |
| unigram l0 + adjacency = (5, 3, 0.5) | 18.1% | 0.57 |

Three readings. First: the l0 source is nearly irrelevant for local stutter — predicted, uniform, and unigram differ by ~3 percentage points, within the noise of n = 8. Second: the official sampling recipe halved the stutter — combining temperature < 1 with narrow top-k correlates adjacent positions onto the same codes, and removing it produced the largest single gain we measured. Third: decode penalties help partially and do not close the gap — the residual is model quality.

The residual was quantified directly: the generator's CE on code stacks it generated itself is 5.65 nats/code, against 4.41 on real held-out stacks (+1.24), concentrated at fine levels (l7: 4.08 vs 2.00; l11: 2.21 vs 0.65). This is the exposure-bias measurement the non-autoregressive literature predicts — the model never learned to condition on its own imperfections, the classic mechanism documented by GU et al. (2018) for non-autoregressive machine translation. The clean evaluation also corrects a figure from §4.3: without input corruption, v2's held-l11 is 0.65 — the reported 2.79 included training-protocol corruption.

The finding converges with an independent replication (WORLD PROGRAMMING SOCIETY, 2026): small NCMs in MLX trained on 960k TinyStories windows (~20× our budget in examples) show the same repetition defect ("Joe was was three years old") and the same inverse relation between teacher-forced CE and generative quality — strong per-level decaying corruption (0.75→0.25) yields clean CE 2.25 and GPT-2-large perplexity 45.6–47.9, versus CE 1.39 and perplexity 93.8–97.8 without corruption; the matched AR baseline scores 14.3–14.7 and the official checkpoint 16.3. Two implications: (i) our residual stutter is the paradigm's documented failure mode, not an artifact of our implementation; (ii) decaying per-level corruption — which we had not yet implemented — is the lever that improves generation at the cost of clean CE. A v3 arm with that recipe (warm-start from v2, 10,000 steps) was running at the time of writing.

### 4.5 What was proven, what was refuted, what remains open

Consolidating twenty rounds into three honest columns:

**Proven.** (i) The full NCM pipeline — tokenizer, hierarchical generator, decoding — iterates on ordinary CPUs. (ii) The structural speedup is real, reproducible, and grows with scale in the 6-level configuration: ×1.14→×2.27→×2.82 (single measurements, n = 8). (iii) Official codes are learnable by a small generator at fine levels — the recipe requires no laboratory secret. (iv) A pyramid codebook derived from pretrained embeddings is viable without training a VQ-VAE (32.1% top-1 token recovery) — although every downstream decoding attempt from it failed, which limits its demonstrated utility.

**Refuted.** (i) Nearest-embedding, code-path likelihood, and hierarchical spec-decode do not close the concept→token last mile (0/40 hits, median rank ~43k; after target-oriented retraining, spec-decode reached 0.58 acceptance at k = 64 yet stayed ~2× slower than AR — the blocker migrated from acceptance to latency). (ii) More steps, smaller codebooks, and intra-level conditioning do not cure unpredictable residuals — but see the §4.2 caveat: verdicts measured in the contaminated regime, therefore indicative. (iii) Per-position pyramid decoding does not beat AR — the gain comes from parallelism across levels, not within them.

**Open.** (i) Level 0 — even at 14.5h/3,000 steps the global code never left uniform; §4.4 showed the l0 source barely matters for stutter, narrowing the open question to narrative arc: if empirical l0 suffices as a seed, the global-coherence deficit migrates to the conditioned levels — the hypothesis our v3 arm with decaying corruption now tests. (ii) Degenerate repetition — partially decomposed: the sampling recipe accounts for roughly half the rate; exposure bias measured +1.24 nats/code at fine levels; the residual is the conditional-independence failure mode documented for non-autoregressive models (GU et al., 2018) and independently replicated (WORLD PROGRAMMING SOCIETY, 2026). (iii) Whether structure-aligned hierarchy — ASTs for code, for instance — yields more predictable codes than reconstruction residuals: our main future interest, and the original motivation (local coding assistant) remains unanswered.

## 5 Discussion

### 5.1 What this study adds to the original paper

The NCM paper demonstrates the recipe on clusters with tens of thousands of steps and thousand-sequence batches. This study adds the reading from the other end of the curve: what each component of the recipe is worth when the budget is three orders of magnitude smaller. Three contributions stand out for whoever will scale it.

**Initialization is a trust boundary.** The bug that compromised every pretrained-tokenizer round is of an insidious class: `apply(_init_weights)` recursively re-initializing a pretrained submodule. No assert, no loss explosion — apparently healthy training conditioned on random embeddings, and samples that "looked like" a paradigm failure. In any replication that loads a frozen tokenizer inside a larger module, verifying weight preservation by inspection (checksums before/after construction) should be gate zero — it costs one line of code and cost us the quality verdicts of every earlier round.

**The low-budget regime separates what is structural from what is scale-bought.** The speedup curve confirmed the theoretical proposition; the quality curve revealed where the recipe pays its costs. The embedding-derived pyramid codebook — our methodological contribution absent from the original — eliminates the pipeline's most expensive stage (VQ-VAE training), yet produced no usable downstream decoding. The official tokenizer, trained with the full recipe, produces fine levels more learnable than ours — yet l0 still resisted 3,000×16 examples. The joint reading: the NCM recipe transfers cost from sampling to training and to the tokenizer; whoever replicates must budget all three, not just the generator.

**The concept→token last mile is a representation problem, not a decoding problem.** Three independent decoding methods failed identically (0/40, zero acceptance in the first protocols) not from method weakness but because the code distribution does not concentrate mass on the right token — median rank ~43k of 152k. The LCM comparison is instructive: SONAR "solves" the last mile because it pays a full transformer per sentence; our nearest-embedding projection is free and recovers 32%. For a coding assistant, where the wrong token is a program that does not compile, this is the point of the entire project that most deserves engineering.

### 5.2 Limitations of this study

We are obliged to list what this work cannot claim. (i) We did not reproduce the full official recipe — neither the ~50,000×2,048 training steps, nor corpus size, nor 12-layer depth; what we measured is the low-budget regime, not the paper's regime. (ii) The main corpus (TinyStories) is deliberately child-level; nothing here speaks of code, the domain that motivated the study — transfer to ASTs and programming structure is grounded conjecture, not a result. (iii) Samples judged by human reading of 8 examples per configuration; we did not apply the TinyStories paper's GPT-4 evaluation protocol or reference perplexity. (iv) Single measurements (n = 8), no repetitions or error bars; the observed AR arm ranged 0.617–0.978 s/sample on the same machine, and TinyStories benchmarks ran on a different machine than the mixed-corpus ones — >3× speedup ratios should be read as an upper bound. (v) The "l0 unlearned" verdict holds for 3,000×16; it may be purely budgetary — and probably is, given l0 began leaving uniform (8.36→7.97 in training windows) in the last thousand steps. (vi) Phase-A and Phase-C quality verdicts were measured in the bug-contaminated regime (§3.3) and await clean-regime revalidation; speed verdicts are unaffected. (vii) The held-l11 comparison to the official model (~2.75) is approximate-protocol — splits and conditioning may differ. (viii) We do not publish training code or checkpoints; metric artifacts and samples ship in the repository, and the reproduction recipe is described in §3. (ix) The §4.4 sampling sweep used 8 samples per configuration: differences of ~3 p.p. are within noise, and the ~10 p.p. effect of the official recipe awaits confirmation with larger n and multiple seeds. (x) The WPS replication is a non-peer-reviewed source (technical blog post); we treat its figures as qualitative converging evidence, not as a primary quantitative reference.

### 5.3 The code question, revisited

The study began by asking whether NCM could enable a fast local coding assistant. The honest answer after eleven days: **we still do not know — but we now know where to measure.** The paradigm delivers its half of the promise (structural speed) in the regime we could run; the other half (quality) has an identified, quantified bottleneck (l0 + budget). For code, one hypothesis this paper leaves formulated but untested: NCM's hierarchy fits naturally with AST hierarchy — generating syntactic structure first and tokens later is exactly the coarse→fine paradigm, and syntax-tree-derived codes may have the predictability that reconstruction residuals lack. It is the direction we most recommend to whoever has a cluster and an interest.

## 6 Conclusion and projections

This work partially reproduced, on commodity hardware and an eleven-day CPU budget, the Next-Concept Modeling pipeline. The consolidated verdict: **the paradigm delivers real, scalable speed already in miniature; text quality requires the full training recipe that only makes sense on a cluster.** Between what was measured and what remains to measure lies a decisive experiment with a known name and budget: running the official recipe (~50,000 steps, batch 2,048, 12 layers) over the official tokenizer — roughly 2,000× our budget in seen examples, i.e., on the order of a day on a modern GPU node.

We project, based on what we measured: (i) a learned l0 at full budget should unlock narrative arc, reducing repetition degeneration through real fine-level conditioning; (ii) relative speedup should hold or grow on GPU, because the advantage is pass count, not kernel throughput; (iii) the code last mile will likely require a learned token head conditioned on codes — our `codes-head` variant already showed codes carry signal the feature alone does not deliver.

Recommended next steps, in cost-benefit order for a cluster: full official-recipe replication; clean-regime revalidation of the contaminated quality verdicts (including the §4.2 ground-truth frontier); training an NCM tokenizer on a code corpus; comparing residual codes with AST-aligned codes; and the real-assistant experiment (tool-use response latency). Each departs directly from this repository's artifacts.

A final note on method: nearly all our pretrained-tokenizer rounds produced wrong quality verdicts because of one misplaced line of code — and fixing that line turned "the paradigm fails" into "the paradigm needs scale". In an era when replications are rare and replication bugs invisible, documenting this felt as important as any speedup.

## References

CAI, Tianle et al. Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads. arXiv, 2024. Available at: https://arxiv.org/abs/2401.10774. Accessed: Nov. 9, 2026.

CHANG, Huiwen et al. MaskGIT: Masked Generative Image Transformer. In: CONFERENCE ON COMPUTER VISION AND PATTERN RECOGNITION (CVPR), 2022. arXiv, 2022. Available at: https://arxiv.org/abs/2202.04200. Accessed: Nov. 9, 2026.

ELDAN, Ronen; LI, Yuanzhi. TinyStories: How Small Can Language Models Be and Still Speak Coherent English? arXiv, 2023. Available at: https://arxiv.org/abs/2305.07759. Accessed: Nov. 9, 2026.

GGERGANOV, Georgi et al. llama.cpp: inference of LLMs in pure C/C++. Free software. Available at: https://github.com/ggml-org/llama.cpp. Accessed: Nov. 9, 2026.

GU, Jiatao et al. Non-Autoregressive Neural Machine Translation. In: INTERNATIONAL CONFERENCE ON LEARNING REPRESENTATIONS (ICLR), 2018. arXiv, 2017. Available at: https://arxiv.org/abs/1711.02281. Accessed: Nov. 10, 2026.

HU, Lanxiang et al. Fast and Accurate Causal Parallel Decoding using Jacobi Forcing. arXiv, 2025. Available at: https://arxiv.org/abs/2512.14681. Accessed: Nov. 9, 2026.

HU, Yunhai et al. Speculative Decoding and Beyond: An In-Depth Survey of Techniques. arXiv, 2025. Available at: https://arxiv.org/abs/2502.19732. Accessed: Nov. 9, 2026.

HUI, Binyuan et al. Qwen2.5-Coder Technical Report. arXiv, 2024. Available at: https://arxiv.org/abs/2409.12186. Accessed: Nov. 9, 2026.

KOU, Siqi et al. CLLMs: Consistency Large Language Models. arXiv, 2024. Available at: https://arxiv.org/abs/2403.00835. Accessed: Nov. 9, 2026.

KUMBONG, Hermann et al. Next-Concept Modeling: Fast, Parallel Coarse-to-Fine Text Generation. arXiv, 2026. Available at: https://arxiv.org/abs/2610.14162. Accessed: Nov. 9, 2026. Checkpoints: https://huggingface.co/hazyresearch/ncm-checkpoints.

KUMAR, Ankur. Residual Vector Quantization for KV Cache Compression in Large Language Models. arXiv, 2024. Available at: https://arxiv.org/abs/2410.15704. Accessed: Nov. 9, 2026.

LEE, Doyup et al. Autoregressive Image Generation using Residual Quantization. In: CONFERENCE ON COMPUTER VISION AND PATTERN RECOGNITION (CVPR), 2022. arXiv, 2022. Available at: https://arxiv.org/abs/2203.01941. Accessed: Nov. 9, 2026.

LEVIATHAN, Yaniv; KALAI, Matan; MATIAS, Yossi. Fast Inference from Transformers via Speculative Decoding. In: INTERNATIONAL CONFERENCE ON MACHINE LEARNING (ICML), 2023. arXiv, 2023. Available at: https://arxiv.org/abs/2211.17192. Accessed: Nov. 9, 2026.

LI, Lujun et al. Sub-MoE: Efficient Mixture-of-Expert LLMs Compression via Subspace Expert Merging. arXiv, 2025. Available at: https://arxiv.org/abs/2506.23266. Accessed: Nov. 9, 2026.

LI, Pingzhi et al. Merge, Then Compress: Demystify Efficient SMoE with Hints from Its Routing Policy. arXiv, 2023. Available at: https://arxiv.org/abs/2310.01334. Accessed: Nov. 9, 2026.

LI, Yuhui et al. EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty. In: INTERNATIONAL CONFERENCE ON MACHINE LEARNING (ICML), 2024. arXiv, 2024. Available at: https://arxiv.org/abs/2401.15077. Accessed: Nov. 9, 2026.

META AI. Large Concept Models: Language Modeling in a Sentence Representation Space. arXiv, 2024. Available at: https://arxiv.org/abs/2412.08821. Accessed: Nov. 9, 2026. Weights (community mirror): huggingface.co/mimir-lcm.

MIAO, Xupeng et al. SpecInfer: Accelerating Generative Large Language Model Serving with Tree-based Speculative Inference and Verification. arXiv, 2023. Available at: https://arxiv.org/abs/2305.09781. Accessed: Nov. 9, 2026.

QWEN TEAM. Qwen3 Technical Report. arXiv, 2025. Available at: https://arxiv.org/abs/2505.09388. Accessed: Nov. 9, 2026.

REN, Houxing et al. Edit-Based Refinement for Parallel Masked Diffusion Language Models. arXiv, 2026. Available at: https://arxiv.org/abs/2605.09603. Accessed: Nov. 9, 2026.

TIAN, Keyu et al. Visual Autoregressive Modeling: Scalable Image Generation via Next-Scale Prediction. In: NEURAL INFORMATION PROCESSING SYSTEMS (NeurIPS), 2024. arXiv, 2024. Available at: https://arxiv.org/abs/2404.02905. Accessed: Nov. 9, 2026.

VAN DEN OORD, Aaron; VINYALS, Oriol; KAVUKCUOGLU, Koray. Neural Discrete Representation Learning. In: NEURAL INFORMATION PROCESSING SYSTEMS (NeurIPS), 2017. arXiv, 2017. Available at: https://arxiv.org/abs/1711.00937. Accessed: Nov. 9, 2026.

WORLD PROGRAMMING SOCIETY. Lower loss, worse stories: small NCMs trained on TinyStories. Technical post, 2026. Available at: https://www.worldprogramming.org/posts/lower-loss-worse-stories-small-ncms-trained-on-tinystories-kbogee. Accessed: Nov. 10, 2026.

YANG, An et al. Qwen2.5 Technical Report. arXiv, 2024. Available at: https://arxiv.org/abs/2412.15115. Accessed: Nov. 9, 2026.

ZEGHIDOUR, Neil et al. SoundStream: An End-to-End Neural Audio Codec. arXiv, 2021. Available at: https://arxiv.org/abs/2107.03312. Accessed: Nov. 9, 2026.

ZHOU, Yongchao et al. DistillSpec: Improving Speculative Decoding via Knowledge Distillation. In: INTERNATIONAL CONFERENCE ON LEARNING REPRESENTATIONS (ICLR), 2024. arXiv, 2024. Available at: https://arxiv.org/abs/2310.08461. Accessed: Nov. 9, 2026.



---

*License: CC BY-NC 4.0 — exclusively for academic and research use. Full artifacts and history in the associated repository.*
