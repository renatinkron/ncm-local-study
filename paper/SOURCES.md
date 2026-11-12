# Fontes e rastreabilidade (SOURCES)

Este arquivo mapeia cada número citado no trabalho ao artefato que o produziu.
Os artefatos de métrica estão em `../artifacts/metrics/`; amostras de texto em
`../artifacts/samples/`.

## Números centrais citados no trabalho

| Afirmação | Valor | Fonte |
|---|---|---|
| Speedup NCM ~69M vs AR ~67M (etapa 2.000) | 0,535 s/amostra vs 0,609 s/amostra (×1,14 nessa run; faixa ×1,05–1,34 entre re-runs internos; ~×3,1 por token-equivalente) | `bench.json` |
| Speedup NCM 471M vs AR 471M | 4,98 s vs 11,29 s/amostra (×2,27); params 478,6M/477,6M; RSS 5,7 GB | `bench_471m.json`, `eval_471m.json` |
| Pirâmide RVQ derivada de embeddings Qwen2.5-0.5B | top-1 token recovery 32,1%; 6 níveis sem colapso | `pyramid_cb_qwen05_report.json` |
| Decode pirâmide por posição vs AR | ~34% mais lento (resultado negativo) | `pyramid_bench.json` |
| Draft especulativo hierárquico (RUN-007) | accept=0 em todas as profundidades; text_equal_ar=True | `spec_pyramid_path_bench.json` |
| Verossimilhança de caminho de códigos (RUN-009) | top-1 vs argmax AR 0/40; rank mediano ~43.505/151.936 (computado dos ranks `ar`/`path` por posição) | `codepath_eval.json` (campo `per_pos`) |
| Codes-head vs feat-head (RUN-010/011) | feat nunca < uniforme (11,9); codes top-1 truth 3,1%→2,34% | `token_head_eval2.json` |
| Spec-decode codes→AR (RUN-012) | a1=3,125%; mean_ar_rank 4.659; accept 0,58@k=64; equal_ar=True | `token_head_ar_eval.json`, `spec_topk64_ar_bench.json` |
| TinyStories end-to-end (RUN-014) | NCM ×2,82 (0,22 vs 0,62 s/amostra); word-salad (dwr 0,652 vs AR 0,384); VQ-VAE held_ce 0,152 (log de treino interno, não versionado) | `bench_ts.json`, `eval_ts.json` |
| Ablação por profundidade (RUN-015) | k0=3,52% → k5=36,28% → k6=96,9% recuperação de token | `level_ablation.json` |
| Braço A taper (RUN-015) | ×3,22, amostras incoerentes; held_ce 0,654/utilização 100% (log interno) | `bench_ts2.json` |
| Braço B resume 3000→5500 (RUN-015) | ×3,21, word-salad; platô de loss ~6,1 (log interno) | `bench_ts_resume.json` |
| Intra-nível chunked (RUN-016) | ×1,22 (passes 6→10); 8/8 word-salad | `bench_ts_intra.json` |
| CE por fronteira GT (RUN-016) — *regime contaminado, revalidação pendente* | l3 6,20→6,22; l4 5,51→5,51; l5 4,70→4,73 (≈uniforme) | `boundary_ce_intra.json` |
| RUN-017 tokenizer oficial (com bug) | 3.000 steps ×16, 14,5 h CPU; held_loss ~7,55; held_l11=6,8095; held_l0=8,20 | `bench_official.json`, `metrics_gen_ncm_official_final.jsonl` |
| RUN-019/020 v2 limpo | held_loss 5,8158; held_l11=2,7924 (≈ref. oficial ~2,75, protocolo aproximado); held_l0=8,2174; 1,306 s/amostra, 12 passes | `bench_v2.json`, `metrics_gen_ncm_v2_final.jsonl`, `representative_samples.json` |
| LCM real Mimir-1.6B em CPU (RUN-008) | 0,0718 conceitos/s; RSS 16,7 GB; saída degenerada | `lcm_probe.json` |
| L0 trajetória (run v2) | janelas de treino 8,36→7,97 (log); held-out final 8,2174 | `metrics_gen_ncm_v2_final.jsonl` |
| Uniforme para codebook 4.096 | CE ≈ 8,32 (ln 4096) | derivado |
| Receita oficial de referência | ~50k steps × batch 2048, 12 camadas, ~88M–600M | Kumbong et al. (2026) |

## Referências externas (todas verificadas)

1. KUMBONG, H. et al. Next-Concept Modeling: Fast, Parallel Coarse-to-Fine
   Text Generation. arXiv:2610.14162, 2026. Código/checkpoints:
   github.com/HazyResearch/ncm; huggingface.co/hazyresearch/ncm-checkpoints.
2. ELDAN, R.; LI, Y. TinyStories: How Small Can Language Models Be and Still
   Speak Coherent English? arXiv:2305.07759, 2023. Dataset:
   huggingface.co/roneneldan/TinyStories.
3. META AI (LCM team). Large Concept Models: Language Modeling in a Sentence
   Representation Space. arXiv:2412.08821, 2024.
4. CAI, T. et al. Medusa: Simple LLM Inference Acceleration Framework with
   Multiple Decoding Heads. arXiv:2401.10774, 2024.
5. HU, Y. et al. Speculative Decoding and Beyond: An In-Depth Survey of
   Techniques. arXiv:2502.19732, 2025.
6. KUMAR, A. Residual Vector Quantization for KV Cache Compression in Large
   Language Models. arXiv:2410.15704, 2024.
7. LI, L. et al. Sub-MoE: Efficient Mixture-of-Expert LLMs Compression via
   Subspace Expert Merging. arXiv:2506.23266, 2025.
8. LI, P. et al. Merge, Then Compress: Demystify Efficient SMoE with Hints
   from Its Routing Policy (MC-SMoE). arXiv:2310.01334, 2023.
9. REN, H. et al. Edit-Based Refinement for Parallel Masked Diffusion
   Language Models. arXiv:2605.09603, 2026.
10. ZHOU, Y. et al. DistillSpec: Improving Speculative Decoding via Knowledge
    Distillation. arXiv:2310.08461 (ICLR 2024).
10a. LEVIATHAN, Y.; KALAI, M.; MATIAS, Y. Fast Inference from Transformers via
    Speculative Decoding. arXiv:2211.17192 (ICML 2023).
11. MIAO, X. et al. SpecInfer: Accelerating Generative LLM Serving with
    Tree-based Speculative Inference and Verification. arXiv:2305.09781, 2023.
12. LI, Y. et al. EAGLE: Speculative Sampling Requires Rethinking Feature
    Uncertainty. arXiv:2401.15077 (ICML 2024).
13. LEE, D. et al. Autoregressive Image Generation using Residual
    Quantization (RQ-Transformer). arXiv:2203.01941 (CVPR 2022).
14. TIAN, K. et al. Visual Autoregressive Modeling: Scalable Image Generation
    via Next-Scale Prediction (VAR). arXiv:2404.02905 (NeurIPS 2024).
15. CHANG, H. et al. MaskGIT: Masked Generative Image Transformer.
    arXiv:2202.04200 (CVPR 2022).
16. VAN DEN OORD, A.; VINYALS, O.; KAVUKCUOGLU, K. Neural Discrete
    Representation Learning (VQ-VAE). arXiv:1711.00937 (NeurIPS 2017).
17. ZEGHIDOUR, N. et al. SoundStream: An End-to-End Neural Audio Codec (RVQ).
    arXiv:2107.03312, 2021.
18. KOU, S. et al. CLLMs: Consistency Large Language Models.
    arXiv:2403.00835, 2024.
19. HU, L. et al. Fast and Accurate Causal Parallel Decoding using Jacobi
    Forcing. arXiv:2512.14681, 2025.
20. QWEN TEAM. Qwen3 Technical Report. arXiv:2505.09388, 2025.
21. HUI, B. et al. Qwen2.5-Coder Technical Report. arXiv:2409.12186, 2024.
21a. YANG, A. et al. Qwen2.5 Technical Report. arXiv:2412.15115, 2024 —
     base dos embeddings usados na pirâmide RVQ e no warm-start.
22. GGERGANOV, G. et al. llama.cpp. Software. github.com/ggml-org/llama.cpp.
23. Mimir-1.6B: mirror comunitário dos pesos LCM (two-tower diffusion) —
    huggingface.co/mimir-lcm (usado na medição RUN-008).

| RUN-023 sweep de inferência (§4.4) | dup_rate: l0 predito 0,2174 → unigrama+oficial 0,1095; uniforme 0,1875; freq=5 0,163; adj 0,1812 | `artifacts/l0_inference/bench_l0*.json` |
| RUN-023 exposure bias (§4.4) | CE self-gen 5,6481 vs held GT 4,4124 (+1,24); l7 4,08 vs 2,00; l11 2,21 vs 0,65 | `artifacts/l0_inference/bench_l0uni_exposure.json` |
| RUN-023 v2 held limpo | held_l11 sem corrupção = 0,6469 (os 2,79 incluíam corrupção de protocolo) | `artifacts/l0_inference/bench_l0uni_exposure.json` (ce_held_gt) |
| Réplica independente WPS (§4.4) | GenPPL: corrupção forte 45,6–47,9; sem corrupção 93,8–97,8; AR 14,3–14,7; ckpt oficial 16,3 | WORLD PROGRAMMING SOCIETY (2026), post técnico |
| l0 unigrama empírica | distribuição de 4.096 códigos l0 do corpus de treino (2706 vistos); TV=0,34 vs vetor oficial | `artifacts/l0_inference/` (unigram derivada; vetor .npy não publicado — regenerável) |
| Série v3 (corrupção linear 0,75→0,25, §4.4) | 10 ckpts @4499–9499: dup_rate 0,0792→0,0576; gap +1,246→+1,090 (platô ~1,10 estável por 4.500 passos); gt CE 4,546→4,216; parada antecipada pré-registrada (2 pts consecutivos na faixa 0,95–1,20) | `artifacts/v3_eval/ckpt_step*.json` (n=8, seed 0, receita oficial) |
| Braço v4 GLAT-lite (§4.4) | pareado @6998 vs controle @6999: gap +1,616 vs +1,105 (−46%) → morto | `artifacts/v4_eval/ckpt_step6998.json` |
| Série v5 intra (§4.4) | 5 ckpts @9998–final: gap +1,101→+0,909; 24 passes vs 11 | `artifacts/v5_eval/ckpt_step*.json` |
| v5 refinement (§4.4) | it1-f0.5-l5: gap +0,416 (31 passes); it2-f0.5-l1: gap −0,226, dup 0,088 (colapso de modo, 46 passes) | `artifacts/v5_eval/sweep_*.json` |
| Profiling do treino (§5.2 xi) | ~67% GEMM @ ~360 GFLOPS (~84% pico); compile trunk −14% isolado / −5% e2e | `artifacts/prof/*.json` (repo privado de pesquisa) |
| Decode chunk-local (§4.4, 5.2 xi) | gen_s 91,68→13,51 s (×6,79, B=8) com saída byte-idêntica | `artifacts/decode_eval/{fullprefix,chunklocal}_eval.json` |
| Head windowing + compile-decode (§4.4) | →8,29 s (×11,06, bit-idêntico); compile quente 7,37 s (~12,4×) | `artifacts/decode_eval/{windowed,compiled}_eval.json` |


## Fonte interna (não primária)

- Especificação destilada "MANA" (conversa de pesquisa paralela, out–nov/2026):
  documento interno que motivou a verificação de claims de decodificação
  paralela; suas falhas documentadas embasam a seção de motivação. Tratado como
  comunicação interna, não como referência bibliográfica.
- Repositório `colibri` (runtime Go/C para inferência local exata de
  Qwen3-Coder-30B-A3B em 16 GB, ~0,6–0,9 token/s): contexto motivador —
  assistentes de código locais são limitados por latência/I-O; documentado no
  histórico técnico do próprio projeto.
