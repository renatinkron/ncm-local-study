# Revisão independente — "Hierarchical text generation on commodity hardware: a partial replication of Next-Concept Modeling"

**Revisor:** revisor científico independente (NLP/ML), sem contexto do projeto.
**Material revisado:** `paper/paper_pt-br.md`, `paper/paper_en-us.md`, `paper/SOURCES.md`, `TIMELINE.md`, `artifacts/metrics/*.json` (21 arquivos), `artifacts/samples/representative_samples.json`, histórico git do repositório.
**Método:** cada número do texto foi conferido contra o artefato indicado em SOURCES.md; referências externas foram verificadas por busca bibliográfica; amostras lidas integralmente. Citações de linha referem-se à versão EN (linha equivalente PT entre parênteses quando difere).

---

## Veredito geral: **APPROVE_WITH_REVISIONS**

Relato de replicação parcial honesto, com cultura de rastreabilidade (SOURCES.md número→artefato) acima do padrão de workshop. A maioria dos números de velocidade e das métricas de diagnóstico bate exatamente com os JSONs, e as amostras sustentam as descrições qualitativas. Entretanto: (a) a medição empírica central da §4.2 foi feita no regime contaminado pelo bug e nunca revalidada pós-fix; (b) os números decisivos da Fase D (losses held) não têm artefato no repositório; (c) a narrativa de speedup "cresce com escala" mistura duas máquinas e dois domínios e o headline ×3.22 é inflado por uma medição degradada do baseline. Nada disso é fatal — todos os achados são corrigíveis em revisão — mas exigem correção antes de aceitar.

---

## Pontos fortes (registro)

- Divulgação exemplar do bug de inicialização (§3.3): classe do bug, mecanismo, runs afetadas, como foi encontrado. Raro e valioso.
- Limitações (§5.2) são concretas e bem formuladas; "Provado/Refutado/Em aberto" (§4.4) é estrutura honesta.
- Amostras confirmam os claims qualitativos: word-salad nos braços falhos (`bench_ts*.json`), estrutura narrativa com repetição degenerada pós-fix (`bench_v2.json`: "was was", "again again again again", "happily ever after").
- Todas as referências verificadas por busca existem de fato (Kumbong et al. arXiv:2610.14162 confirmado; Ren 2605.09603; Jacobi Forcing 2512.14681; Hu survey 2502.19732; Kumar 2410.15704; Sub-MoE 2506.23266; MC-SMoE 2310.01334). Nenhuma referência inventada encontrada.

---

## Achados (ordenados por severidade)

### MAJOR

**1. A "medição que fecha a questão" da §4.2 foi feita no regime contaminado e nunca revalidada pós-fix.** EN L94 (PT L96): os CEs por fronteira ground-truth (l3 6,21→6,22; l4 5,51→5,51; l5 4,70→4,73) vêm de `boundary_ce_intra.json`, produzido na família RUN-015/016 — ou seja, sob o mesmo `LevelGenerator` cujo `apply(_init_weights)` re-randomizava o VQ-VAE de condicionamento (bug descrito em L71/PT L73, runs 014–017). O próprio bug *prediz* o resultado observado: um gerador treinado com features de condicionamento embaralhadas aprende a ignorar o condicionamento — a resposta plana ao GT-prefix é exatamente o que o bug produziria, independentemente da informação mútua real dos códigos. O artigo reconhece em §3.3 que os vereditos dessas runs "estavam confundidas" — incluindo explicitamente "os níveis finos são imprevisíveis" — mas depois reapresenta a conclusão em §4.2, no resumo (EN L11/PT L11: "codes factor reconstruction error rather than predictability") e em §5.1 (EN L120/PT L122: "irredeemably unpredictable") sem ressalva local. O argumento estrutural (resíduo = resto de maior entropia) e o contraste com o tokenizador oficial (§4.3) sustentam a hipótese, mas a evidência direta está confundida. **Correção:** marcar explicitamente em §4.2 e no resumo que a medição GT-boundary foi feita no regime pré-fix e não foi repetida pós-fix; rebaixar o claim de "demonstração" para "hipótese consistente com o regime observado"; idealmente, fornecer um `boundary_ce` re-medido com o gerador limpo (mesmo que no tokenizador oficial).

**2. Os números decisivos da Fase D não têm artefato verificável.** EN L98 (PT L100): "held l11 = 6,81 ... convergiram a held l11 = 2,68, abaixo dos ~2,75 do próprio gerador de referência. O loss held total caiu de 7,55 para 5,82"; L100: "held l0 ≈ 8,22"; L126: "(8,36→7,97)". SOURCES.md mapeia esses valores para `bench_official.json` e `bench_v2.json`, mas **nenhum dos dois arquivos contém losses** — contêm apenas metadados de geração (passes, s_per_sample) e textos. Idem: "held_ce 0,152" (SOURCES→`eval_ts.json`, que não tem esse campo), "held CE 0,654, utilização 100%" (§4.2, ausente de SOURCES e de todos os JSONs), "platô ~6,1" (não está em `bench_ts_resume.json`), "rank mediano ~43.505/151.936" (`codepath_eval.json` guarda IDs de token, não ranks), "0,07 conceitos/s" (Mimir), "±20%". O git log corrobora (commit "RUN-018..020: held 5.82, l11=2.68 < oficial"), mas mensagem de commit não é artefato auditável. Num relato cuja proposta de valor é "números confiáveis", a lacuna atinge justamente o resultado mais citado. **Correção:** publicar os logs de treino (curvas held por nível, config) em `artifacts/metrics/` ou num `train_log.json`; esclarecer a proveniência do "2,75 oficial" — é medido localmente rodando o checkpoint oficial (qual held split? qual seed de corrupção?) ou extraído do paper original? Se do paper, a comparação 2,68 < 2,75 é entre evaluações possivelmente não-equivalentes e precisa de ressalva. Vale ainda notar a direção surpreendente do resultado (gerador 64M/3k steps superando o de referência em l11) — merece uma frase de interpretação (head l11 local pode estar mais fácil de sobre-ajustar ao held pequeno).

**3. A narrativa "speedup cresce com escala" mistura escala, máquina e domínio — e o ×3.22 é inflado por baseline degradada.** (i) Confusão de eixo: `bench.json` e `bench_471m.json` rodam em `cpu_count: 12` (i5); `bench_ts*.json` rodam em `cpu_count: 16` (Ryzen). O par que sustenta "cresce com escala" é apenas (1,14→2,27 dentro do corpus misto). O ponto TinyStories (~69M, ×2,82–3,22) muda máquina **e** domínio simultaneamente — a mesma classe de tamanho (~68M) dá 1,14 no misto e 2,82 no TinyStories, o que contradiz uma história puramente escalar e sugere driver de domínio/máquina. O resumo (EN L11) apresenta a progressão como se fosse um único eixo. (ii) O headline ×3,22: o braço AR idêntico mede 0,617 s em `bench_ts`, 0,883 s em `bench_ts2` (+43%) e 0,978 s em `bench_ts_resume` (+58%) — na **mesma máquina** (16 threads). O NCM do taper também piorou (0,219→0,274). O ×3,22 decorre mais da degradação do AR que do mérito do braço; a leitura honesta é ~×2,8–3,2 com ruído. (iii) n=8 amostras, medição única, sem repetições nem intervalos — a limitação (iv) atribui "±20%" à variação *entre máquinas*, mas a maior variância observada é intra-máquina no braço AR. **Correção:** reportar min/mediana ou pelo menos re-medir o AR; rotular os dois eixos (máquina, domínio) na tabela; declarar que os modelos de 60M/471M do corpus misto produziam saída degenerada **nos dois braços** (as amostras AR em `bench.json`/`bench_471m.json` são também salada — o speedup mede latência entre modelos quebrados naquele corpus; válido mecanicamente, mas deve ser dito, assim como o paper honestamente diz que no TinyStories o AR produzia estórias coerentes e o NCM não).

### MINOR

**4. "Quadruplicar os parâmetros" é factualmente errado.** EN L86 (PT L88): 68,8M→478,6M é ~7×, não 4×. Corrigir para "~7×" ou "ao sextuplicar".

**5. Contagem de parâmetros inconsistente entre texto, timeline e artefato.** Resumo/abstract diz "60M" (EN L11); tabela diz "~62–68M" (EN L81); Fase A diz "~62M" (EN L61); TIMELINE diz "NCM 62M vs AR 67M"; `bench.json` registra 68.834.656 e 67.151.232. O intervalo "~62–68M" nem cobre o valor real (68,8M). Padronizar para "~68M/~67M" ou explicar a diferença (config antes/depois de warm-start?).

**6. "Melhorou a reconstrução (held_ce 0,654)" contradiz o held_ce 0,152 reportado.** EN L92 (PT L94): o taper é dito melhorar a reconstrução com CE 0,654, mas SOURCES reporta held_ce 0,152 para o VQ-VAE base (RUN-014). 0,654 > 0,152 — pior, não melhor, se a métrica é a mesma. Provavelmente a melhora é de **utilização** (100%) e não de CE, ou a métrica mudou de definição entre runs. Esclarecer o denominador da comparação.

**7. "0 accepts" sobrestima a uniformidade da falha; o sucesso parcial do spec-decode não é reportado.** EN L122 (PT L124): "três métodos falharam identicamente (0/40, 0 accepts)". Mas `spec_topk64_ar_bench.json` mostra `accept_per_offset: [0.583, 1.0, 1.0, 1.0]` com token-head — o primeiro accept real (o próprio TIMELINE o celebra). O modo de falha ali foi **latência** (1,82 vs 3,71 tok/s — spec ~2× mais lento), não taxa de aceite. Consolidar como "0 accepts" apaga a única evidência de que o caminho codes→AR pode aceitar — relevante para a recomendação futura de "learned token head". Reportar o accept 0,58@k=64 e a lentidão resultante.

**8. Núcleo de hiperparâmetros ausente — incompleto para um relato de replicação.** O texto nunca informa learning rate, otimizador, schedule, warmup, weight decay, arquitetura exata do LevelGenerator (camadas/dim/heads; só "~62M"/"64M"/"69M"), nem a receita VQ-VAE local (commitment cost, EMA, épocas). Para o público-alvo declarado ("quem tem cluster e quer saber onde o terreno está minado"), a config exata é parte do valor. Adicionar tabela de config por fase ou apontar para configs versionadas.

**9. Referências órfãs e erros de autoria.** (a) ~10 das 24 entradas nunca são citadas no corpo: KUMAR, LEE (RQ-Transformer), LI L. (Sub-MoE), LI Y. (EAGLE), LI Z. (MC-SMoE), ZHOU (DistillSpec), HUI, QWEN TEAM, GGERGANOV, CHANG (MaskGIT é mencionado informalmente em L92). Parece sobra da fase de survey de spec-decode/MoE. Podar ou citar onde pertinente (p.ex., EAGLE/DistillSpec caberiam em §2.1(i); RQ-Transformer em §2.2 ao lado de VAR). (b) Erros de autoria verificados: "FU, C. et al." para Jacobi Forcing — o primeiro autor é Lanxiang **Hu** (correto: HU, L. et al., 2025); "LI, Z. et al." para MC-SMoE — primeiro autor Pingzhi **Li** (LI, P.); "LI, L. et al." para Sub-MoE — o primeiro autor aparenta ser Sirui **Han** (verificar). (c) Qwen2.5-0.5B (modelo usado, L61/L63) é citado via HUI (relatório Qwen2.5-**Coder**) — referência errada; usar o tech report do Qwen2.5. Mimir-1.6B é usado (L43) mas não entra na lista de referências.

**10. Ambiguidade l0: "8,22 ≈ uniforme" vs trajetória "8,36→7,97".** EN L100 diz held_l0 ≈ 8,22 (indistinguível do uniforme); L126 (limitação v) diz que l0 começou a sair do uniforme (8,36→7,97) nos últimos mil passos. Presumivelmente 7,97 é treino e 8,22 é held — se for isso, dizer explicitamente; hoje lêem-se como valores conflitantes do mesmo número.

### NIT

**11. Arredondamentos:** l3 reportado "6,21→6,22" onde o artefato tem 6,2049→6,2205 (≈6,20→6,22); "~×2,9 por token-equivalente" onde o cálculo direto dá ~3,07 (mean tokens_equiv 341,9/0,535 s ÷ 127/0,609 s) — subclaim seguro mas o ~ pode subir.

**12. Numeração de runs:** runs 002 e 013 não aparecem em TIMELINE, fases ou artefatos; "vinte rodadas" conta 18 IDs visíveis. §3.3 diz "a run 019/020 é o primeiro teste limpo" mas TIMELINE agrupa "RUN-018–020: retreino limpo". Harmonisar.

**13. Métricas computadas e não reportadas:** `eval*.json` contêm py_parse_rate e distinct_word_ratio (NCM 0,652 vs AR 0,384 no TinyStories; AR 0,251 no misto — degeneração do baseline). Nunca discutidos. Ou reportam (evidência quantitativa extra, mesmo que fraca) ou explicam por que foram descartadas.

**14. `representative_samples.json` truncado:** só 2 das 8 amostras v2 (`sample_0`, `sample_2`); os 8 textos existem em `bench_v2.json` — mas então o arquivo "representativo" é redundante/incompleto. Incluir todas ou remover.

**15. Elementos canônicos de replication report ausentes:** sem reproducibility checklist (versões de torch/transformers, seeds por run — os artifacts mostram `seed: 0`, mas só nos benches), sem versão pinada do dataset TinyStories, sem declaração explícita do que é/não é distribuído (README diz que checkpoints não são distribuídos — o paper só diz "artefatos no repositório"). Também não há comparação direta com **amostras do gerador oficial** — se os checkpoints públicos incluem o gerador TinyStories, gerar algumas amostras localmente daria a referência qualitativa que hoje falta entre "word-salad" e "estória do paper".

---

## Avaliação por eixo

**Coerência científica:** os números com artefato batem quase sempre (erros: "quadruplicar", rótulos de parâmetros, rounding l3). O claim estrutural de speedup está correto em essência (6–12 passes vs ~127 confirmado nos JSONs); a interpretação "cresce com escala" é suportada por apenas um par intra-domínio. Sobras de overclaiming: abstrato apresenta progressão única 1,14→2,27→3,22; "demonstração" de independência dos códigos finos em evidência confundida. Sobras de underclaiming: o accept 0,58@k=64 (único sinal positivo de última milha) enterrado; baseline AR degenerado no corpus misto não declarado.

**Posicionamento:** honesto e explícito sobre ser replicação parcial (L29), sobre orçamento (~2.000× verificado: 50.000×2.048 / 3.000×16 = 2.133×) e sobre o bug. Falta: marcar in-loco que a evidência da §4.2 é pré-fix; completar lacunas de artefato da Fase D.

**Referências:** nenhuma inventada (verificado por busca); ~10 órfãs; 2–3 erros de primeiro autor; 1 referência errada para o modelo usado; Mimir usado e não listado.

**Narrativa:** acima do padrão de relatórios TinyStories-style no quesito honestidade de processo; abaixo no rigor de apresentação (sem tabela de config, sem variância, sem checklist). Seções canônicas faltantes: tabela de hiperparâmetros, declaração de reprodutibilidade, nota de disponibilidade de artefatos.

**Projeções (§6):** (i) l0 aprendido → arco narrativo: defensável e bem-hedged; (ii) speedup mantido/crescente em GPU: defensável (vantagem é contagem de passes); (iii) token head aprendida para a última milha: defensável e sustentada por `token_head_ar_eval` (codes a1 3,125% vs feat 0%; rank 4.659 vs 77.200); a estimativa ~2.000× está aritmeticamente correta como lower bound (exclui diferença de tamanho de modelo — declarar).

---

## Recomendações mínimas para aceite

1. Ressalva explícita junto ao claim da §4.2 (e no resumo) sobre o regime contaminado da medição GT-boundary; se possível, re-medida pós-fix.
2. Publicar os logs de treino que geram held 7,55/5,82, l11 6,81/2,68, l0 8,20/8,22/8,36→7,97, held_ce 0,152/0,654, platô 6,1 — e esclarecer a proveniência do "2,75 oficial".
3. Corrigir "quadruplicar"→~7×; padronizar contagens de parâmetros; re-medir ou reportar variância do braço AR; rotular máquina+domínio na tabela de speedup.
4. Resolver a ambiguidade do held_ce (0,152 vs 0,654) e do l0 (8,22 vs 7,97).
5. Tabela de hiperparâmetros/config por fase.
6. Podar referências órfãs; corrigir HU/FU (Jacobi Forcing), LI P. (MC-SMoE), autor do Sub-MoE; citar Qwen2.5 correto; listar Mimir.
7. Reportar o accept 0,58@k=64 e a respectiva perda de latência; reportar ou justificar o descarte de py_parse_rate/distinct_word_ratio.

**Veredito final: APPROVE_WITH_REVISIONS** — trabalho publicável como relato de replicação após as correções acima; os itens MAJOR são de apresentação/rastreabilidade, não de validade fundamental.
