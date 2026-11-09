# NCM on Commodity Hardware — Replication Study

Relato técnico de uma replicação parcial e independente do **Next-Concept
Modeling** (Kumbong et al., 2026 — Stanford Hazy Research) executada
inteiramente em CPUs convencionais durante onze dias.

**Autor:** Renato de Souza e Silva · **Licença:** CC BY-NC 4.0 (uso
exclusivamente acadêmico e de pesquisa — ver `LICENSE`)

## O que este repositório contém

- `paper/paper_pt-br.md` — trabalho acadêmico completo (normas ABNT:
  NBR 14724:2024, NBR 6028:2021, NBR 6024:2012, NBR 10520:2023, NBR 6023:2025)
- `paper/paper_en-us.md` — versão integral em inglês
- `paper/SOURCES.md` — rastreabilidade: cada número citado → artefato/fonte
- `TIMELINE.md` — linha do tempo datada das 20 rodadas de pesquisa
- `artifacts/metrics/` — métricas JSON brutas (benches, ablações, evals)
- `artifacts/samples/` — amostras representativas geradas

## Resumo dos achados

- Speedup hierárquico medido crescendo com escala: **×1,05–1,34** (60M) →
  **×2,27** (471M) → **×2,82–3,22** (TinyStories), sempre contra baseline AR
  casado — confirma em hardware comum a hipótese central do paper original.
- Bug de inicialização descoberto por blind review: `apply(_init_weights)`
  re-randomizava o tokenizador VQ-VAE pré-treinado — quatro rodadas de
  vereditos invalidadas e re-medidas.
- Pós-fix: níveis finos convergem a held_l11 = 2,68 (abaixo do modelo de
  referência oficial); o código global (l0) permanece ~uniforme no orçamento
  local — o gargalo de coerência é escala/orçamento, não arquitetura.
- Códigos residuais finos de tokenizadores sub-treinados fatoram erro de
  reconstrução, não previsibilidade (CE por fronteira ground-truth).

## Como citar

```bibtex
@misc{silva2026ncm,
  author = {Silva, Renato de Souza e},
  title  = {Geração hierárquica de texto em hardware convencional: uma
            replicação parcial do Next-Concept Modeling},
  year   = {2026},
  howpublished = {\url{https://github.com/renatinkron/ncm-local-study}}
}
```

## Aviso

Relato de replicação **parcial**: o regime medido é o de orçamento limitado
(~3.000 passos × batch 16, ~2000× menos computação que a receita oficial).
Resultados negativos são reportados com o mesmo rigor dos positivos.
Checkpoints não são distribuídos (tamanho); as métricas e amostras que os
produziram estão em `artifacts/`.
