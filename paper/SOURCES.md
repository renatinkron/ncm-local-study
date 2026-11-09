# Fontes e rastreabilidade (SOURCES)

Este arquivo mapeia cada número citado no trabalho ao artefato que o produziu.
Os artefatos de métrica estão em `../artifacts/metrics/`; amostras de texto em
`../artifacts/samples/`.

## Números centrais citados no trabalho

| Afirmação | Valor | Fonte |
|---|---|---|
| Speedup NCM 60M vs AR 60M (etapa 2.000) | 0,535 s/amostra vs 0,609 s/amostra (×1,14; ~×2,9 por token-equivalente) | `bench.json` |
| Speedup NCM 471M vs AR 471M | 4,98 s vs 11,29 s/amostra (×2,27); params 478,6M/477,6M; RSS 5,7 GB | `bench_471m.json`, `eval_471m.json` |
| Pirâmide RVQ derivada de embeddings Qwen2.5-0.5B | top-1 token recovery 32,1%; 6 níveis sem colapso | `pyramid_cb_qwen05_report.json` |
| Decode pirâmide por posição vs AR | ~34% mais lento (resultado negativo) | `pyramid_bench.json` |
| Draft especulativo hierárquico (RUN-007) | accept=0 em todas as profundidades; text_equal_ar=True | `spec_pyramid_path_bench.json` |
| Verossimilhança de caminho de códigos (RUN-009) | top-1 vs argmax AR 0/40; rank mediano ~43.505/151.936 | `codepath_eval.json` |
| Codes-head vs feat-head (RUN-010/011) | feat nunca < uniforme (11,9); codes top-1 truth 3,1%→2,34% | `token_head_eval2.json` |
| Spec-decode codes→AR (RUN-012) | a1=3,125%; mean_ar_rank 4.659; accept 0,58@k=64; equal_ar=True | `token_head_ar_eval.json`, `spec_topk64_ar_bench.json` |
| TinyStories end-to-end (RUN-014) | NCM ×2,82 (0,22 vs 0,62 s/amostra); VQ-VAE held_ce 0,152; word-salad | `bench_ts.json`, `eval_ts.json` |
| Ablação por profundidade (RUN-015) | k0=3,5% → k5=36,4% → k6=96,9% recuperação de token | `level_ablation.json` |
| Braço A taper (RUN-015) | ×3,22, amostras incoerentes | `bench_ts2.json` |
| Braço B resume 3000→5500 (RUN-015) | ×3,21, word-salad; platô ~6,1 | `bench_ts_resume.json` |
| Intra-nível chunked (RUN-016) | ×1,22 (passes 6→10); 8/8 word-salad | `bench_ts_intra.json` |
| CE por fronteira GT (RUN-016) — *regime contaminado, revalidação pendente* | l3 6,20→6,22; l4 5,51→5,51; l5 4,70→4,73 (≈uniforme) | `boundary_ce_intra.json` |
| RUN-017 tokenizer oficial (com bug) | 3.000 steps ×16, 14,5 h CPU; held_loss ~7,55; held_l11=6,8095; held_l0=8,20 | `bench_official.json`, `metrics_gen_ncm_official_final.jsonl` |
| RUN-019/020 v2 limpo | held_loss 5,8158; held_l11=2,7924 (≈ref. oficial ~2,75, protocolo aproximado); held_l0=8,2174; 1,306 s/amostra, 12 passes | `bench_v2.json`, `metrics_gen_ncm_v2_final.jsonl`, `representative_samples.json` |
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
8. LI, Z. et al. Merge, Then Compress: Demystify Efficient SMoE with Hints
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

## Fonte interna (não primária)

- Especificação destilada "MANA" (conversa de pesquisa paralela, out–nov/2026):
  documento interno que motivou a verificação de claims de decodificação
  paralela; suas falhas documentadas embasam a seção de motivação. Tratado como
  comunicação interna, não como referência bibliográfica.
- Repositório `colibri` (runtime Go/C para inferência local exata de
  Qwen3-Coder-30B-A3B em 16 GB, ~0,6–0,9 token/s): contexto motivador —
  assistentes de código locais são limitados por latência/I-O; documentado no
  histórico técnico do próprio projeto.
