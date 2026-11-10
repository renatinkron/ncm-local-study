# Linha do tempo da pesquisa

Período: 30 de outubro a 9 de novembro de 2026 (11 dias corridos).
Hardware principal: laptop comum (i5-1235U, 12 threads, 23 GB RAM, CPU-only);
migração parcial para CPU 4750U/32 GB a partir do RUN-006.

| Data | Marco | Resultado-chave |
|---|---|---|
| 30 out | Bootstrap: corpus 3.181 registros, BPE 8k, VQ-VAE MiniMSRQ | quantizador vendored colapsou 2×; MiniMSRQ estável |
| 31 out | RUN-001/003: NCM 62M vs AR 67M, 2.000 steps | primeiro speedup medido: ×1,05–1,34/amostra |
| 01–02 nov | RUN-004: NCM 471M vs AR 471M | ×2,27/amostra — speedup cresce com escala |
| 03–04 nov | RUN-005/006/007: pirâmide RVQ de embeddings Qwen; draft especulativo hierárquico | pirâmide sem treinar VQ-VAE (top-1 32,1%); accept=0 — última milha conceito→token |
| 04 nov | RUN-008: LCM real (Mimir-1.6B) medido em CPU | 0,07 conceitos/s; última milha = decoder seq2seq, não projeção |
| 04–06 nov | RUN-009–012: codepath likelihood, codes-head, spec-decode | path-score 0/40; codes carregam sinal (ablação); primeiro accept real 0,58@k=64 |
| 07 nov | RUN-014–016: TinyStories end-to-end + ablações + intra-nível | ×2,82–×3,22 mas word-salad; veredito "residual-VQ fatora erro de reconstrução, não previsibilidade" |
| 08 nov | RUN-017: tokenizer oficial + gerador 64M, 3.000 steps/14,5 h | held 7,55; finos previsíveis; l0 travado; word-salad |
| 08 nov | Blind review: descoberto bug de init que re-randomizava o VQ-VAE — presente desde a 1ª versão do LevelGenerator | vereditos de qualidade 001–004 e 014–017 confundidos (vereditos de velocidade não afetados) |
| 08–09 nov | RUN-018–020: retreino limpo (v2) idêntico + bench | held 5,82; l11=2,79 (≈ref. oficial ~2,75); estrutura TinyStories real, coerência global ausente |
| 09 nov | Redação, revisões independentes, publicação | este repositório |
| 09 nov | RUN-023: releitura do `eval_shard.py` oficial + sweep de inferência | `predict_level0: false` + amostragem plena (T=1,0, sem top-k) descobertos; l0 unigrama implementado; fonte do l0 ~irrelevante p/ gagueira; receita oficial cortou dup_rate 21,7%→11,0% |
| 09–10 nov | RUN-023: exposure bias medido + réplica independente WPS encontrada | CE self-gen 5,65 vs GT 4,41 (+1,24 nats, concentrado nos finos); WPS/MLX (960k janelas): mesmo stutter, corrupção↔geração invertida; braço v3 (corrupção linear 0,75→0,25, warm-start, 10k steps) lançado |

## Leitura honesta do esforço

~11 dias corridos, dezenas de treinos CPU (runs de 30 min a 14,5 h),
20+ rodadas registradas de hipótese→medição→veredito. O gargalo foi sempre
orçamento de computação — razão pela qual a continuação natural é em cluster.
