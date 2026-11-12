# Geração hierárquica de texto em hardware convencional: uma replicação parcial do Next-Concept Modeling e o que onze dias de CPU ensinaram

**Renato de Souza e Silva**

Pesquisa independente — novembro de 2026

---

## Resumo

O Next-Concept Modeling (NCM), proposto recentemente pelo grupo Hazy Research, substitui a geração autorregressiva token a token por uma hierarquia de códigos discretos gerada do grosso para o fino, prometendo geração paralela dez a trinta vezes mais rápida que modelos de difusão equivalentes. Este trabalho relata uma tentativa independente de reproduzir essa receita integralmente em hardware convencional — um laptop comum, sem GPU, com 23–32 GB de RAM — durante onze dias de experimentação, vinte rodadas registradas de hipótese, medição e veredito. Os resultados são mistos e, por isso mesmo, instrutivos. O speedup estrutural do paradigma foi medido e confirmou a hipótese central do artigo original em escala pequena: a vantagem de tempo por amostra cresceu de aproximadamente 1,14 vezes com modelos de ~69M parâmetros para 2,27 vezes com ~479M e até 3,22 vezes no domínio TinyStories, sem nenhuma aceleração de hardware além de threads de CPU. A análise de falhas, por sua vez, produziu três contribuições que o artigo original não documenta: a indicação de que códigos residuais de VQ-VAE treinados com orçamento insuficiente fatoram erro de reconstrução, e não previsibilidade (medida feita ainda no regime contaminado e que pede revalidação); a identificação de um bug de inicialização que destruía silenciosamente o tokenizador pré-treinado e comprometia todas as rodadas que carregavam o VQ-VAE dentro do gerador; e a confirmação, após a correção, de que os códigos oficiais são aprendíveis por um gerador pequeno — o nível mais fino convergiu a uma perda held-out de 2,79, essencialmente no nível do gerador de referência (~2,75) — enquanto o código global permaneceu no limiar uniforme, caracterizando o gargalo de coerência como problema de escala e orçamento de treino, não de arquitetura. Concluímos que a técnica é viável de iterar localmente e que o experimento decisivo — a receita oficial completa — requer aproximadamente duas mil vezes mais computação, um orçamento trivial para um cluster e inviável para um laptop. O trabalho documenta integralmente hipóteses rejeitadas, números negativos e o processo, na convicção de que falhas bem medidas economizam mais tempo de laboratório do que sucessos mal explicados.

**Palavras-chave:** modelos de linguagem; geração paralela de texto; quantização vetorial residual; replicação; eficiência de inferência.

## Abstract

Next-Concept Modeling (NCM), recently proposed by the Hazy Research group, replaces token-by-token autoregressive generation with a coarse-to-fine hierarchy of discrete codes, promising parallel generation ten to thirty times faster than comparable diffusion language models. This paper reports an independent attempt to reproduce that recipe end-to-end on commodity hardware — an ordinary laptop, no GPU, 23–32 GB of RAM — over eleven days and twenty logged hypothesis–measurement–verdict rounds. The results are mixed, and instructive precisely because of that. The paradigm's structural speedup was measured and confirms the original paper's central hypothesis at small scale: per-sample speedup grew from about 1.14× at 60M parameters to 2.27× at 471M and up to 3.22× on TinyStories, with no hardware acceleration beyond CPU threads. The failure analysis, in turn, produced three contributions the original paper does not document: evidence — measured in the still-contaminated regime and pending clean revalidation — that under-trained residual VQ-VAE codes factor reconstruction error rather than predictability; the discovery of an initialization bug that silently re-randomized the pretrained tokenizer and compromised every round loading the VQ-VAE inside the generator; and, after the fix, confirmation that the official codes are learnable by a small generator — the finest level converged to a held-out loss of 2.79, essentially at the reference generator's level (~2.75) — while the global code stayed near-uniform, characterizing the coherence bottleneck as a scale and training-budget problem rather than an architectural one. We conclude the technique is viable to iterate locally, and that the decisive experiment — the full official recipe — requires roughly two thousand times more compute: trivial for a cluster, impossible for a laptop. We document rejected hypotheses, negative numbers, and process in full, in the belief that well-measured failures save more laboratory time than poorly explained successes.

**Keywords:** language models; parallel text generation; residual vector quantization; replication; inference efficiency.

---

## 1 Introdução

A motivação deste trabalho não nasceu no NCM — nasceu na frustração medida. Em um projeto anterior, construímos um runtime completo capaz de executar localmente, sem quantizar, o checkpoint BF16 do Qwen3-Coder-30B-A3B-Instruct (QWEN TEAM, 2025) em uma máquina de 16 GB de RAM: mapeamento de memória, prefetch de especialistas, cache residente, decodificação especulativa exata, servidor persistente — um estudo que se soma à literatura de compressão e servirimento eficiente de modelos Mixture-of-Experts (LI, P. et al., 2023; LI, L. et al., 2025). Funcionou — e a conclusão foi um número: entre 0,6 e 0,9 token por segundo. O modelo inteiro cabe, roda exato e produz saída real, mas a latência o torna impraticável como assistente interativo. O gargalo não era falta de engenharia; era estrutural. Quarenta e oito camadas sequenciais, um token de cada vez, dezenas de gigabytes de pesos atravessando o barramento por token gerado.

Foi nesse contexto que a publicação do Next-Concept Modeling (KUMBONG et al., 2026) chamou atenção. Em vez de lutar contra a sequencialidade com truques de sistema — rascunhos especulativos, árvores de verificação, drafts destilados —, o NCM propõe mudar a própria estrutura da geração: um VQ-VAE residual multiescala converte texto em uma pilha de níveis de códigos discretos, e um transformer gera um nível por vez, prevendo todos os códigos daquele nível em paralelo. O artigo reporta até 31 vezes mais velocidade que o melhor baseline de difusão com qualidade equivalente — e, mais importante para nós, a assinatura de custo é diferente por construção: K passagens por nível em vez de L passagens por token, com K tipicamente muito menor que L.

A pergunta que guiou este estudo foi simples de formular e cara de responder: **é possível iterar a receita NCM em hardware convencional, sem GPU, e aprender o suficiente sobre o paradigma para decidir se vale investir em escala?** Onze dias e vinte rodadas depois, a resposta é "sim, com ressalvas que só quem mediu enxerga" — e este trabalho é o relato honesto do que essas ressalvas são, quanto custam e onde a fronteira real do hardware doméstico se encontra.

Três advertências de escopo, antes dos resultados. Primeiro: este não é um paper de resultados novos no sentido usual — é um relato de replicação parcial com análise de falhas, o formato que consideramos mais útil para quem tem um cluster e quer saber onde o terreno está minado antes de gastar GPUs nele. Segundo: todos os números reportados são medidos em CPU; speedups relativos entre braços casados são a métrica honesta aqui, não latências absolutas. Terceiro: documentamos falhas com o mesmo cuidado dos acertos — praticamente todas as rodadas com tokenizador pré-treinado deste estudo produziram vereditos de qualidade que descobrimos depois estarem contaminados por um bug nosso, e contamos essa história porque ela é instrutiva.

## 2 Referencial teórico

### 2.1 O custo da sequencialidade

Modelos autorregressivos (AR) geram um token por passagem de forward. Para um transformer de profundidade d, o custo por token é uma varredura completa; para um assistente de código que responde em blocos de centenas de tokens, a latência acumula linearmente e, em hardware sem GPU, cada passagem disputa largura de banda de memória com o sistema inteiro. A literatura de aceleração ataca esse custo por quatro vias principais: (i) decodificação especulativa, em que um modelo rascunhador propõe tokens que o alvo verifica em paralelo (LEVIATHAN; KALAI; MATIAS, 2023; MIAO et al., 2023 — ver HU et al., 2025 para um survey completo); (ii) cabeças de decodificação múltipla treinadas para prever posições futuras (CAI et al., 2024 — Medusa); (iii) destilação de consistência que treina o próprio modelo a convergir em iterações paralelas (KOU et al., 2024 — CLLMs; HU, L. et al., 2025 — Jacobi Forcing); e (iv) modelos de difusão de texto, que refinam a sequência inteira por passos de denoising (REN et al., 2026). Todas preservam a arquitetura autoregressiva ou a substituem por iteração sobre a sequência completa — nenhuma muda a unidade atômica da geração.

### 2.2 Next-Concept Modeling

O NCM (KUMBONG et al., 2026) muda a unidade. Um VQ-VAE residual multiescala (VAN DEN OORD; VINYALS; KAVUKCUOGLU, 2017; ZEGHIDOUR et al., 2021; KUMAR, 2024) codifica cada sequência em K níveis de códigos discretos: os primeiros níveis, curtos, capturam a forma grossa; os últimos, mais longos, refinam os detalhes. Um transformer então aprende a gerar a hierarquia nível a nível — cada nível previsto em uma única passagem paralela, condicionado à reconstrução acumulada dos níveis anteriores. É a transposição para texto do next-scale prediction que o VAR (TIAN et al., 2024) demonstrou em imagens, com adaptações específicas: um codebook por nível, nested dropout para forçar os níveis grossos a carregarem semântica, e corrupção de condicionamento para robustez a erro de previsão. O artigo reporta, em escala de ~88M a ~600M parâmetros, qualidade competitiva ou superior aos melhores modelos de difusão com 10–31× menos custo de amostragem e 4–5× menos passes de treino — e checkpoints públicos para LM1B, OpenWebText, XSum, WMT14, Sudoku e TinyStories, que usamos diretamente.

### 2.3 O problema da "última milha"

Um ponto que o referencial mostra com clareza — e que o nosso estudo confirmou da maneira mais cara — é que esquemas hierárquicos de representação não eliminam o custo de voltar ao token. O LCM da Meta (META AI, 2024) opera em embeddings SONAR por sentença e resolve a última milha com um decoder seq2seq completo por conceito gerado. Em nossa medição local, o Mimir-1.6B (mirror dos pesos LCM) produziu 0,07 conceitos por segundo em CPU — a última milha existe e exige capacidade generativa — e latência — próprias. Speculative decoding tradicional resolve diferente: propõe tokens diretamente e o alvo verifica (HU et al., 2025). A questão empírica para o NCM é: quanta capacidade o caminho códigos→texto exige, e onde ela mora — no codebook, no decoder VQ, no gerador?

## 3 Materiais e métodos

### 3.1 Hardware e orçamento

Duas máquinas de uso geral participaram do estudo; nenhuma GPU foi usada em nenhuma etapa:

- Laptop principal: i5-1235U (2 P-cores + 8 E-cores, 12 threads), 23 GB de RAM, NVMe — runs 001–005;
- CPU secundária: Ryzen 7 4750U (16 threads), 32 GB de RAM — runs 006 em diante, incluindo a inferência auxiliar do modelo de 30B via servidor llama.cpp (GGERGANOV et al.).

Todo treinamento e benchmark de código de terceiros rodaram em contêiner isolado sem rede (`podman --network none`), conforme a política do laboratório local. Treinos variaram de 30 minutos a 14,5 horas; o orçamento total foi de aproximadamente 300 horas de CPU em onze dias corridos. Os treinamentos usaram otimizador AdamW (lr 3e-4), batch 16 e seed 0, salvo indicação contrária em cada run; os geradores são transformers de 4 camadas (~62–69M de parâmetros), salvo o experimento de escala com 24 camadas (~471–479M de parâmetros).

### 3.2 Linha experimental

O estudo foi organizado como sequência de rodadas hipótese→medição→veredito, cada uma com artefatos versionados (métricas JSON, checkpoints, amostras). Sem repetir o relato completo de `TIMELINE.md`, as fases foram:

**Fase A — replicação nativa em escala pequena (runs 001–004).** Pipeline próprio: corpus misto de 3.181–4.665 registros (biblioteca-padrão Python + gerações de professores locais), BPE 8.192 tokens, VQ-VAE residual de 6 níveis, LevelGenerator ~68M de parâmetros com warm-start de Qwen2.5-0.5B, baseline AR casado em parâmetros e dados.

**Fase B — caminho piramidal sem tokenizador (runs 005–012).** Derivamos um codebook pirâmide de 6 níveis diretamente dos embeddings do Qwen2.5-0.5B (YANG et al., 2024; HUI et al., 2024) — sem treinar VQ-VAE algum — e investigamos a última milha código→token por três vias: nearest-embedding, verossimilhança de caminho de códigos e decodificação especulativa com cabeças hierárquicas, desenho orientado pela literatura de drafts treinados contra a distribuição do alvo (ZHOU et al., 2024 — DistillSpec; MIAO et al., 2023 — SpecInfer), de autoregressão em espaço de features (LI, Y. et al., 2024 — EAGLE) e de geração hierárquica sobre quantização residual (LEE et al., 2022 — RQ-Transformer).

**Fase C — domínio TinyStories end-to-end (runs 014–016).** Corpus TinyStories (ELDAN; LI, 2023), BPE 8k, VQ-VAE próprio, NCM 69M vs AR 67M; ablação de profundidade de níveis, taper de codebook e geração intra-nível por chunks causais.

**Fase D — tokenizador oficial (runs 017–020).** VQ-VAE oficial publicado (12 níveis, codebooks uniformes de 4.096 códigos) + códigos oficiais pré-computados em ~177 mil segmentos de treino; LevelGenerator de 64M de parâmetros, receita de corrupção U(0,1) com skip de nível 0; 3.000 passos × batch 16.

*Nota de numeração:* os identificadores de run não são contíguos — 013 e 018 foram rodadas de infraestrutura/monitoramento sem experimento associado.

### 3.3 O bug que ensinou mais que os vereditos

Registro obrigatório: em todas as rodadas que carregavam um VQ-VAE pré-treinado dentro do gerador — fases A (001–004), C (014–016) e D (017) —, `LevelGenerator.__init__` registrava o módulo do VQ-VAE antes de `self.apply(_init_weights)`; o bug existia desde a primeira versão do código e só foi corrigido na run 018. O apply recursivo rerandomizava todos os Linear/Embedding do tokenizador pré-treinado (codebooks, decoder, projeções). O efeito: o treino condicionava em embeddings aleatórios e a decodificação usava um decoder aleatório, produzindo salada de palavras para qualquer código, bom ou ruim. Todos os vereditos de qualidade do gerador nessas rodadas — incluindo o "os níveis finos são imprevisíveis" que parecia o achado científico mais profundo — estavam comprometidos. (Os vereditos de velocidade não são afetados: o tempo de geração é estrutural e independe dos pesos carregados; a fase B, que não usa esse gerador, também não foi contaminada.) O bug foi descoberto por revisão cega, corrigido (registrar o VQ-VAE depois do apply) e verificado por inspeção de pesos em contêiner. A run 019/020 é o primeiro teste limpo de qualidade.

## 4 Resultados

### 4.1 O speedup hierárquico é real e cresce com escala

A métrica central do artigo original — menos passagens sequenciais por amostra — reproduziu-se em três escalas, sempre contra baseline AR casado em parâmetros e dados, mesma máquina, mesmas threads:

Tabela 1 — Latência por amostra, NCM vs AR casado (8 amostras, temp. 0,8, top-k 40)

| Escala / domínio | Máquina | NCM | AR | Speedup por amostra |
|---|---|---|---|---|
| ~67–69M de parâmetros (corpus misto) | i5-1235U, 12 threads | 0,535 s | 0,609 s | ×1,05–1,34 (~×3,1 por token-equivalente) |
| ~471–479M de parâmetros (mesmo corpus) | i5-1235U, 12 threads | 4,98 s | 11,29 s | **×2,27** |
| ~69M de parâmetros (TinyStories) | Ryzen 4750U, 16 threads | 0,22 s | 0,62 s | **×2,82** |
| ~69M de parâmetros (TinyStories, taper) | Ryzen 4750U, 16 threads | — | — | **×3,22** |

Fonte: elaborada pelo autor a partir de `artifacts/metrics/bench*.json` (medições únicas, n = 8).

A hipótese do artigo — a vantagem cresce com o tamanho, porque o AR paga ~127 passagens sequenciais de uma rede maior e o NCM paga 6 — foi confirmada no nosso hardware: a razão subiu de ~1,1× para ~2,3× ao multiplicar os parâmetros por ~7, e manteve-se >3× no domínio TinyStories — com a ressalva de que as linhas 3–4 foram medidas em máquina e domínio diferentes das linhas 1–2, e que o braço AR apresentou variação intra-máquina de 0,617–0,978 s entre runs; tratamos as razões >3× como limite superior. Estruturalmente, o NCM gera uma amostra em 6 passagens de nível nessa configuração; o AR casado, em ~127 passos de token. A vantagem não é otimização de kernel — é menos viagens sequenciais pela rede.

### 4.2 A qualidade não vem de graça: a anatomia do fracasso

Contra essa velocidade, três rodadas de correções independentes falharam em produzir texto coerente no regime de orçamento local — e a análise do porquê é, a nosso ver, o achado científico mais útil do estudo.

**Primeira hipótese rejeitada — tamanho de codebook.** Afunilar os codebooks dos níveis finos (2048→[2048,1024,512,256,128]) elevou a utilização dos codebooks finos a 100% a custo de reconstrução pior (held_ce 0,654 vs 0,152 do codebook base) e manteve o speedup (×3,22), mas as amostras seguiram incoerentes. **Segunda — mais passos.** Retreinar de 3.000 para 5.500 passos não tirou o loss do platô (~6,1) nem as amostras da salada. **Terceira — condicionamento intranível.** Gerar cada nível em chunks causais (vizinhos do mesmo nível visíveis, estilo MaskGIT/VAR — CHANG et al., 2022; TIAN et al., 2024) custou passagens extras (speedup reduzido a ×1,22) e não curou a qualidade.

A medição que fecha a questão é a fronteira ground-truth: condicionando o gerador em códigos reais em até ~88–90% das posições vizinhas (fronteiras de 87,5% a 89,8% testadas), a entropia cruzada dos níveis finos não cai — l3: 6,20→6,22; l4: 5,51→5,51; l5: 4,70→4,73, contra uniformes de 6,24/5,55/4,85. **Os códigos residuais finos comportam-se como essencialmente independentes entre si e do contexto grosso** — I(código_i; código_j | grosso) ≈ 0. A interpretação é estrutural: o residual de cada nível VQ é, por construção, aquilo que os níveis anteriores não codificaram — o resto de maior entropia. Quantização residual fatora por erro de reconstrução, não por previsibilidade; exigir que um gerador preveja esses resíduos em paralelo é pedir que ele modele precisamente a parte do sinal que menos estrutura de contexto tem.

*Caveat metodológico:* essas medições foram feitas ainda no regime contaminado pelo bug de init (§3.3) — um gerador treinado a ignorar o condicionamento apresentaria exatamente essa resposta plana. Apresentamos o resultado porque a leitura estrutural permanece a melhor explicação disponível, mas ele **não foi revalidado no regime limpo** — trata-se de evidência indicativa, não conclusiva.

### 4.3 O tokenizador oficial muda o diagnóstico — e o bug muda tudo de novo

Com o VQ-VAE oficial (treinado pelos autores com nested dropout e receita completa), o quadro se inverteu parcialmente: os níveis finos provaram-se aprendíveis — held_l11 = 6,81 contra o uniforme 8,32 já na run contaminada — e, após a correção do bug de init, convergiram para held_l11 = 2,79, **essencialmente no nível do gerador de referência (~2,75, comparação aproximada de protocolo)**. O loss held total caiu de 7,55 (run com bug) para 5,82 (run limpo). Os números de perda constam de `metrics_gen_ncm_{official,v2}_final.jsonl`.

As amostras pós-correção são qualitativamente outra coisa: vocabulário TinyStories correto, nomes de personagens, fórmulas narrativas ("Once upon a time", "From that day on", "happily ever after"), pontuação de diálogo — estrutura de estória real. O defeito remanescente é duplo e específico: repetição degenerada de bigramas ("was was", "Lucy Lucy", "again again again") e ausência de arco narrativo — frases e fragmentos concatenados sem causalidade global. Duas métricas explicam os dois defeitos por mecanismos distintos: (i) o nível 0 — o código único de 4.096 classes que carrega a identidade inteira da estória — ficou em held_l0 ≈ 8,22, indistinguível do uniforme 8,32 (sem l0, os finos refinam detalhes de uma estória nunca decidida); (ii) a gagueira local é a assinatura clássica de independência condicional em decodificação paralela — prever todos os códigos de um nível em uma única passagem sem ver os vizinhos recém-gerados favorece bigramas repetidos. O resultado é estória-em-forma-sem-narrativa.

Custo de geração nesta configuração: 1,306 s/amostra em 12 passagens de nível — **mais lento que o AR casado** (0,617 s/amostra na mesma máquina), porque cada passagem de nível fino processa sequências de até 641 posições. O speedup de ~×3 da §4.1 refere-se à nossa configuração de 6 níveis sobre T = 128; na configuração oficial de 12 níveis, o custo por passagem cresce o suficiente para inverter o sinal em CPU. A vantagem de latência reportada pelo artigo original foi medida em outra escala/hardware e não se transfere automaticamente a esta configuração.

### 4.4 A receita de inferência pesa mais que o nível 0

Uma releitura do código oficial de avaliação (`eval_shard.py`) expôs duas divergências de protocolo — não de arquitetura — entre nossa amostragem e a dos autores: (i) `predict_level0: false` — o l0 não é previsto pelo gerador, é sorteado de uma distribuição unigrama empírica salva no checkpoint; (ii) amostragem multinomial plena (temperatura 1,0, sem top-k nem top-p), enquanto usávamos temperatura 0,8 com top-40. Implementamos as três variantes de l0 (predito, uniforme, unigrama) e portamos as penalidades de decodificação do upstream (`frequency_penalty`, `adjacency_penalty`), com compatibilidade bit a bit no modo original. Sobre 8 amostras por configuração, medimos a taxa de palavras duplicadas adjacentes (dup_rate) e a razão de palavras distintas:

| Configuração de amostragem | dup_rate | distinct |
|---|---|---|
| l0 predito, T = 0,8, top-40 (protocolo v2) | 21,7% | 0,63 |
| l0 uniforme, T = 0,8, top-40 | 18,8% | 0,62 |
| l0 unigrama, T = 0,8, top-40 | 22,0% | 0,55 |
| l0 unigrama, T = 1,0, sem top-k (receita oficial) | **11,0%** | 0,66 |
| l0 unigrama + frequency_penalty = 5 | 16,3% | 0,64 |
| l0 unigrama + adjacency = (5, 3, 0,5) | 18,1% | 0,57 |

Três leituras. Primeira: a fonte do l0 é quase irrelevante para a gagueira local — predito, uniforme e unigrama diferem em ~3 pontos percentuais, dentro do ruído de n = 8. Segunda: a receita de amostragem oficial cortou a gagueira pela metade — a combinação temperatura < 1 com top-k estreito correlaciona posições adjacentes nos mesmos códigos, e removê-la produz o maior ganho isolado que já medimos. Terceira: as penalidades de decodificação ajudam parcialmente e não fecham a lacuna — o resíduo é qualidade de modelo.

O resíduo foi quantificado diretamente: o CE do gerador sobre pilhas de códigos que ele mesmo gerou é de 5,65 nats/código, contra 4,41 sobre pilhas reais held-out (+1,24), concentrado nos níveis finos (l7: 4,08 vs 2,00; l11: 2,21 vs 0,65). É a medida de exposure bias que a literatura de geração não-autoregressiva prediz — o modelo não aprendeu a condicionar-se às próprias imperfeições, mecanismo clássico documentado por GU et al. (2018) para tradução automática não-autoregressiva. A avaliação limpa também corrige um número do §4.3: sem corrupção de entrada, o held_l11 do v2 é 0,65 — os 2,79 reportados incluíam a corrupção do protocolo de treino.

O achado converge com uma replicação independente (WORLD PROGRAMMING SOCIETY, 2026): NCMs pequenos em MLX, treinados sobre 960 mil janelas de TinyStories (~20× nosso orçamento em exemplos), exibem o mesmo defeito de repetição ("Joe was was three years old") e a mesma relação inversa entre CE teacher-forced e qualidade generativa — corrupção forte decrescente por nível (0,75→0,25) produz CE limpo 2,25 e perplexidade GPT-2-large de 45,6–47,9, contra CE 1,39 e perplexidade 93,8–97,8 sem corrupção; o baseline AR casado marca 14,3–14,7 e o checkpoint oficial 16,3. Duas implicações: (i) nossa gagueira residual é o modo de falha documentado do paradigma, não artefato da nossa implementação; (ii) a corrupção por nível decrescente — que ainda não havíamos implementado — é a alavanca que melhora geração à custa de CE limpo.

O braço v3 testou essa alavanca (warm-start do v2, corrupção linear 0,75→0,25 por nível, ~6.500 passos até a parada antecipada pré-registrada, dez checkpoints avaliados na receita oficial, n = 8 cada). Resultado: a taxa de repetição adjacente caiu de 0,110 para ~0,05–0,06 (metade), o exposure gap medido recuou de +1,24 para um platô de ~+1,1 nats estável por 4.500 passos, e o CE limpo held-out melhorou para 4,22 — abaixo do baseline v2 (4,41), contrariando a expectativa de degradação permanente. Veredito: a corrupção decrescente oficial mitiga a falha local — sem curá-la: a multimodalidade residual (~1,1 nat) e a gagueira residual (~5%) persistem no nosso orçamento.

Um segundo braço testou a variante aparentemente mais forte — corromper o condicionamento com códigos amostrados das próprias predições do modelo (análogo ao glancing do GLAT; QIAN et al., 2021): no pareado a orçamento igual, o exposure gap piorou 46% (+1,62 contra +1,11 do controle uniforme), e o braço foi encerrado. É o segundo datapoint independente — após a "neighbour corruption" do WPS — de que corromper com códigos plausíveis, em vez de ruído uniforme, amplifica o desajuste: predições corretas demais não ensinam recuperação de erro. A alavanca restante documentada é o refinamento iterativo no decodificador (mask-predict de GHAZVININEJAD et al., 2019), que exigiria treinar o condicionamento mascarado intra-nível — mecanismo presente no nosso gerador mas ainda não exercitado.

Um terceiro braço (v5) exercitou exatamente esse mecanismo: warm-start do v3 com condicionamento intra-nível mascarado (posições pendentes recebem um embedding de máscara aprendido; decididas, o embedding real do código — treino com fronteiras prefixo e subconjuntos aleatórios em mistura), decode sequencial em chunks dentro do nível, e refinement estilo mask-predict remascarando as posições cuja amostra teve menor probabilidade. O decode chunked isolado parecia fechar +0,18 nats do gap em n=8 (+0,91 contra +1,09 do v3), mas essa leitura não sobreviveu a n maior: com n=32 em três seeds, o v5 chunked mede +0,853±0,051 contra +0,827±0,077 do v3 — efeito nulo dentro do erro, ou seja, a aparente melhora era ruído de amostragem pequena, não o condicionamento intra-nível em si. O que replica robustamente é o refinement: uma rodada moderada (it1-f0.5-l5) fecha o gap em **+0,316±0,033** — ~2,6× abaixo do platô de todos os braços, estável entre seeds, ao custo de 31 passes. A resposta, porém, é **não-monotônica**: com duas rodadas cobrindo todos os níveis a 50%, o gap inverte (−0,23 — o modelo passa a achar seus próprios códigos mais previsíveis que os reais) enquanto a taxa de duplicados dobra (0,088). É a assinatura de colapso de modo por refinement agressivo, e um alerta metodológico: o gap, isoladamente, deixa de medir qualidade nesse regime — saídas quase-determinísticas teriam gap nulo sendo piores. Em nenhum braço a taxa de duplicados saiu de ~0,05, reforçando que o stutter residual não é independência intra-nível, mas déficit de coerência global (l0/arco).

Há ainda uma segunda leitura do braço v5, estrutural e não de treino. A máscara de atenção do gerador é block-diagonal entre níveis: o chunk de cada nível atende apenas a si mesmo e ao contexto (q0), nunca aos chunks de níveis já gerados — o condicionamento cruzado viaja dentro dos embeddings de entrada, não pela atenção. Explorar esse fato no decode — processar o trunk apenas sobre o chunk corrente, com a hierarquia acumulada de forma incremental, em paridade com o `generate` oficial upstream — reduziu o tempo de geração de 91,7 s para 8,3 s (×11 em lote de 8, mesmo hardware e mesma receita; com `torch.compile` opcional no trunk de decode, ~12,4× em estado quente), com saída **byte-idêntica**: mesmos códigos, mesmos textos, mesmas métricas. A redução soma duas fontes exatas: processar o trunk apenas sobre o chunk corrente, com a hierarquia acumulada de forma incremental — em paridade com o `generate` oficial upstream — e computar a cabeça de logits apenas nas posições consumidas por cada passe intra-nível. Isso reenquadra o custo discutido nesta seção — o refinement moderado que fechou o gap em +0,32 custa 31 passes, mas cada passe agora é ~11× mais barato: em trabalho total, equivale a ~3 passes do decode denso anterior.

### 4.5 O que se provou, o que se refutou, o que ficou em aberto

Consolidando vinte rodadas em três colunas honestas:

**Provado.** (i) O pipeline NCM completo — tokenizador, gerador hierárquico, decodificação — itera em CPU comum. (ii) O speedup estrutural é real, reproduzível e cresce com escala na configuração de 6 níveis: ×1,14→×2,27→×2,82 (medições únicas, n = 8). (iii) Os códigos oficiais são aprendíveis por gerador pequeno nos níveis finos — a receita não exige segredo de laboratório. (iv) Um codebook pirâmide derivado de embeddings pré-treinados constrói-se sem treinar VQ-VAE (top-1 de 32,1% de recuperação de token) — embora toda tentativa de decodificação downstream a partir dele tenha falhado, o que limita sua utilidade demonstrada.

**Refutado.** (i) Nearest-embedding, verossimilhança de caminho e spec-decode hierárquico não fecham a última milha conceito→token (0/40 acertos, rank mediano ~43 mil; após retreino orientado a alvo, o spec-decode chegou a 0,58 de aceitação com k = 64, porém ~2× mais lento que o AR — o bloqueio migrou de aceitação para latência). (ii) Mais passos, codebooks menores e condicionamento intranível não curaram a falta de coerência nos nossos tokenizadores locais — mas ver o caveat do §4.2: vereditos medidos no regime contaminado, portanto indicativos. (iii) Decodificação pirâmide por posição não vence AR — o ganho vem de paralelismo entre níveis, não dentro deles.

**Em aberto.** (i) O nível 0 — nem em 14,5 h/3.000 passos o código global saiu do uniforme; o §4.4 mostrou que a fonte do l0 pouco importa para a gagueira, restando a questão do arco narrativo: se l0 empírico basta como semente, o déficit de coerência global migra para os níveis condicionados — hipótese que o braço v3 com corrupção decrescente testa. (ii) Repetição degenerada — substancialmente decomposta e parcialmente mitigada: a receita de amostragem responde por ~metade da taxa e a corrupção decrescente por nível reduz o resíduo pela metade restante (dup_rate 0,11→~0,05; gap +1,24→platô ~+1,1); duas formas de corrupção não-uniforme foram refutadas (predições-próprias neste trabalho, neighbour-corruption no WPS); o resíduo persistente é o modo de falha de independência condicional (GU et al., 2018; WORLD PROGRAMMING SOCIETY, 2026); o refinamento iterativo mascarado (braço v5, GHAZVININEJAD et al., 2019) fecha o gap medido ainda mais — +0,32±0,03 confirmado em n=32×3 seeds — mas é não-monotônico e colapsa o modo quando levado ao extremo (gap invertido, duplicados dobrados); o stutter residual (~0,05) é invariante entre todas as alavancas, apontando para o déficit de l0/arco global, não para independência intra-nível. (iii) Se hierarquia alinhada a estrutura — ASTs para código, por exemplo — produz códigos mais previsíveis que resíduo de reconstrução: nosso principal interesse futuro e a motivação original (assistente de código local) continua sem resposta.

## 5 Discussão

### 5.1 O que este estudo adiciona ao artigo original

O artigo do NCM demonstra a receita em clusters com dezenas de milhares de passos e batches de milhares de sequências. Este estudo adiciona a leitura do outro extremo da curva: o que cada componente da receita vale quando o orçamento é três ordens de grandeza menor. Três contribuições merecem destaque para quem vai escalar.

**A inicialização é uma fronteira de confiança.** O bug que comprometeu todas as rodadas com tokenizador pré-treinado é de uma classe insidiosa: `apply(_init_weights)` reinicializando recursivamente um submódulo congelado. Nenhum assert, nenhuma explosão de perda — treino aparentemente saudável condicionado em embeddings aleatórios, e amostras que "pareciam" falha de paradigma. Em qualquer replicação que carregue um tokenizador congelado dentro de um módulo maior, verificar a preservação de pesos por inspeção (checksums antes/depois da construção) deveria ser o gate zero — custa uma linha de código e nos custou o veredito de qualidade de todas as rodadas anteriores.

**O regime de orçamento baixo separa o que é estrutural do que é efeito de escala.** A curva de speedup confirmou a proposição teórica; a curva de qualidade revelou onde a receita paga seus custos. O codebook piramidal derivado de embeddings — nossa contribuição metodológica não presente no original — elimina o estágio mais caro do pipeline (treinar VQ-VAE), mas não produziu decodificação downstream utilizável. O tokenizador oficial, treinado com receita completa, produz níveis finos aprendíveis até o nível do modelo de referência — e ainda assim o l0 resistiu a 3.000×16 exemplos. A leitura conjunta: a receita NCM transfere custo de amostragem para treino e para o tokenizador; quem replica deve orçar as três coisas, não só o gerador.

**A última milha conceito→token é um problema de representação, não de decode.** Três métodos de decodificação independentes falharam identicamente (0/40, aceitação nula nos primeiros protocolos) não por fraqueza dos métodos, mas porque a distribuição de códigos não concentra massa no token correto — rank mediano ~43 mil de 152 mil. A comparação com o LCM é instrutiva: o SONAR "resolve" a última milha porque paga um transformer completo por sentença; nossa projeção nearest-embedding é grátis e recupera 32%. Para um assistente de código, onde o token errado é um programa que não compila, esse é o ponto do projeto inteiro que mais merece engenharia.

### 5.2 Limitações deste estudo

Somos obrigados a elencar o que este trabalho não pode afirmar. (i) Não reproduzimos a receita oficial completa — nem os ~50.000×2.048 passos de treino, nem o tamanho de corpus, nem a profundidade de 12 camadas; o que medimos é o regime de baixo orçamento, não o regime do artigo. (ii) O corpus principal (TinyStories) é deliberadamente infantil; nada aqui fala de código, domínio que motivou o estudo — a transferência para ASTs e estrutura de programação é conjectura fundamentada, não resultado. (iii) Amostras julgadas por leitura humana de 8 exemplos por configuração; não aplicamos o protocolo de avaliação por GPT-4 do artigo TinyStories nem perplexidade de referência. (iv) Medidas de geração originalmente únicas (n = 8, sem repetições): as comparações-chave de decode foram refeitas com n = 32 em três seeds — as barras constam no §4.4 e já corrigiram uma leitura (o +0,18 nats do intra-chunked não replicou) — mas sweeps de amostragem e ablações anteriores permanecem n=8; o braço AR observado variou de 0,617 a 0,978 s/amostra na mesma máquina, e os benchmarks de TinyStories foram feitos em máquina distinta dos benchmarks de corpus misto — as razões de speedup >3× devem ser lidas como limite superior. (v) O veredito "l0 não aprendido" vale para 3.000×16; pode ser puramente orçamento — e provavelmente é, dado que l0 começou a sair do uniforme (8,36→7,97 em janelas de treino) nos últimos mil passos. (vi) Os vereditos de qualidade das fases A e C foram medidos no regime contaminado pelo bug de init (§3.3) e aguardam revalidação no regime limpo; os de velocidade não são afetados. (vii) A comparação de held_l11 com o modelo oficial (~2,75) é aproximada de protocolo — splits e condicionamento podem diferir. (viii) Não publicamos código de treino nem checkpoints; os artefatos de métricas e amostras estão no repositório, e a receita de reprodução é descrita em §3. (ix) O sweep de amostragem do §4.4 usou 8 amostras por configuração: diferenças de ~3 p.p. estão dentro do ruído, e o efeito de ~10 p.p. da receita oficial aguarda confirmação com n maior e múltiplas seeds. (x) A réplica WPS é fonte não-revisada por pares (post técnico); tratamos seus números como evidência convergente qualitativa, não como referência quantitativa primária. (xi) O treino local é limitado por FLOPs, não por implementação: profiling do passo mostrou ~67% do tempo em GEMMs MKL a ~84% do pico teórico do Zen 2 — reescrita em C/Rust não tem espaço para ganho, e o único speedup de software medido (torch.compile no trunk) é marginal (~5%). A alavanca de treino é orçamento de computação/hardware, não engenharia de runtime; para inferência, a alavanca documentada — decode chunk-local sobre a atenção block-diagonal, com head windowing — foi implementada e medida: ×11 no tempo de geração (~12,4× com trunk compilado em estado quente) com saída bit-idêntica (§4.4).

### 5.3 A pergunta de código, revisitada

O estudo nasceu perguntando se NCM poderia viabilizar assistente de código local rápido. A resposta honesta após onze dias: **ainda não sabemos — mas agora sabemos onde medir.** O paradigma entrega sua metade da promessa (velocidade estrutural) no regime que conseguimos rodar; a outra metade (qualidade) tem um gargalo identificado e quantificado (l0 + orçamento). Para código, acrescenta-se uma hipótese que este trabalho deixa formulada mas não testada: a hierarquia do NCM casa naturalmente com a hierarquia de ASTs — gerar estrutura sintática primeiro e tokens depois é exatamente o paradigma grosso→fino, e códigos derivados de árvores de sintaxe podem ter a previsibilidade que resíduos de reconstrução não têm. É a direção que mais recomendamos a quem tem cluster e interesse.

## 6 Conclusão e projeções

Este trabalho reproduziu parcialmente, em hardware convencional e orçamento de onze dias de CPU, o pipeline Next-Concept Modeling. O veredito consolidado: **o paradigma entrega velocidade real e escalável já em miniatura; a qualidade de texto exige a receita de treino completa que só faz sentido em cluster.** Entre o que foi medido e o que falta medir, há um experimento decisivo com nome e orçamento conhecidos: rodar a receita oficial (~50.000 passos, batch 2.048, 12 camadas) sobre o tokenizador oficial — aproximadamente 2.000× nosso orçamento em exemplos vistos, ou seja, da ordem de um dia em um nó de GPU moderno.

Projetamos, com base no que medimos: (i) l0 aprendido em orçamento completo deve destravar arco narrativo, reduzindo a degeneração de repetição via condicionamento real dos finos; (ii) o speedup relativo deve manter-se ou crescer em GPU, porque a vantagem é contagem de passagens, não throughput de kernel; (iii) a última milha para código exigirá provavelmente uma cabeça de token aprendida condicionada nos códigos — nossa variante `codes-head` já mostrou que códigos carregam sinal que a feature sozinha não entrega.

Os próximos passos recomendados, em ordem de custo-benefício para um cluster: replicação da receita oficial completa; revalidação no regime limpo dos vereditos de qualidade contaminados (inclusive a fronteira ground-truth do §4.2); treino de um tokenizador NCM sobre corpus de código; comparação de códigos residuais com códigos alinhados a AST; e o experimento de assistente real (latência de resposta em tool-use). Cada um sai direto dos artefatos deste repositório.

A nota final é de método: praticamente todas as nossas rodadas com tokenizador pré-treinado produziram vereditos de qualidade errados por causa de uma linha de código mal posicionada — e a correção dessa linha transformou "o paradigma falha" em "o paradigma precisa de escala". Numa era em que replicações são raras e bugs de replicação são invisíveis, documentar isso pareceu tão importante quanto qualquer speedup.

## Referências

CAI, Tianle et al. Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads. arXiv, 2024. Disponível em: https://arxiv.org/abs/2401.10774. Acesso em: 9 nov. 2026.

CHANG, Huiwen et al. MaskGIT: Masked Generative Image Transformer. In: CONFERENCE ON COMPUTER VISION AND PATTERN RECOGNITION (CVPR), 2022. arXiv, 2022. Disponível em: https://arxiv.org/abs/2202.04200. Acesso em: 9 nov. 2026.

ELDAN, Ronen; LI, Yuanzhi. TinyStories: How Small Can Language Models Be and Still Speak Coherent English? arXiv, 2023. Disponível em: https://arxiv.org/abs/2305.07759. Acesso em: 9 nov. 2026.

GGERGANOV, Georgi et al. llama.cpp: inference of LLMs in pure C/C++. Software livre. Disponível em: https://github.com/ggml-org/llama.cpp. Acesso em: 9 nov. 2026.

GHAZVININEJAD, Marjan et al. Mask-Predict: Parallel Decoding of Conditional Masked Language Models. In: EMPIRICAL METHODS IN NATURAL LANGUAGE PROCESSING (EMNLP), 2019. arXiv, 2019. Disponível em: https://arxiv.org/abs/1904.09324. Acesso em: 11 nov. 2026.

GU, Jiatao et al. Non-Autoregressive Neural Machine Translation. In: INTERNATIONAL CONFERENCE ON LEARNING REPRESENTATIONS (ICLR), 2018. arXiv, 2017. Disponível em: https://arxiv.org/abs/1711.02281. Acesso em: 10 nov. 2026.

HU, Lanxiang et al. Fast and Accurate Causal Parallel Decoding using Jacobi Forcing. arXiv, 2025. Disponível em: https://arxiv.org/abs/2512.14681. Acesso em: 9 nov. 2026.

HU, Yunhai et al. Speculative Decoding and Beyond: An In-Depth Survey of Techniques. arXiv, 2025. Disponível em: https://arxiv.org/abs/2502.19732. Acesso em: 9 nov. 2026.

HUI, Binyuan et al. Qwen2.5-Coder Technical Report. arXiv, 2024. Disponível em: https://arxiv.org/abs/2409.12186. Acesso em: 9 nov. 2026.

KOU, Siqi et al. CLLMs: Consistency Large Language Models. arXiv, 2024. Disponível em: https://arxiv.org/abs/2403.00835. Acesso em: 9 nov. 2026.

KUMBONG, Hermann et al. Next-Concept Modeling: Fast, Parallel Coarse-to-Fine Text Generation. arXiv, 2026. Disponível em: https://arxiv.org/abs/2610.14162. Acesso em: 9 nov. 2026. Checkpoints: https://huggingface.co/hazyresearch/ncm-checkpoints.

KUMAR, Ankur. Residual Vector Quantization for KV Cache Compression in Large Language Models. arXiv, 2024. Disponível em: https://arxiv.org/abs/2410.15704. Acesso em: 9 nov. 2026.

LEE, Doyup et al. Autoregressive Image Generation using Residual Quantization. In: CONFERENCE ON COMPUTER VISION AND PATTERN RECOGNITION (CVPR), 2022. arXiv, 2022. Disponível em: https://arxiv.org/abs/2203.01941. Acesso em: 9 nov. 2026.

LEVIATHAN, Yaniv; KALAI, Matan; MATIAS, Yossi. Fast Inference from Transformers via Speculative Decoding. In: INTERNATIONAL CONFERENCE ON MACHINE LEARNING (ICML), 2023. arXiv, 2023. Disponível em: https://arxiv.org/abs/2211.17192. Acesso em: 9 nov. 2026.

LI, Lujun et al. Sub-MoE: Efficient Mixture-of-Expert LLMs Compression via Subspace Expert Merging. arXiv, 2025. Disponível em: https://arxiv.org/abs/2506.23266. Acesso em: 9 nov. 2026.

LI, Pingzhi et al. Merge, Then Compress: Demystify Efficient SMoE with Hints from Its Routing Policy. arXiv, 2023. Disponível em: https://arxiv.org/abs/2310.01334. Acesso em: 9 nov. 2026.

LI, Yuhui et al. EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty. In: INTERNATIONAL CONFERENCE ON MACHINE LEARNING (ICML), 2024. arXiv, 2024. Disponível em: https://arxiv.org/abs/2401.15077. Acesso em: 9 nov. 2026.

META AI. Large Concept Models: Language Modeling in a Sentence Representation Space. arXiv, 2024. Disponível em: https://arxiv.org/abs/2412.08821. Acesso em: 9 nov. 2026. Pesos (mirror comunitário): huggingface.co/mimir-lcm.

MIAO, Xupeng et al. SpecInfer: Accelerating Generative Large Language Model Serving with Tree-based Speculative Inference and Verification. arXiv, 2023. Disponível em: https://arxiv.org/abs/2305.09781. Acesso em: 9 nov. 2026.

QIAN, Lihua et al. Glancing Transformer for Non-Autoregressive Neural Machine Translation. In: ANNUAL MEETING OF THE ASSOCIATION FOR COMPUTATIONAL LINGUISTICS (ACL), 2021. arXiv, 2020. Disponível em: https://arxiv.org/abs/2008.07905. Acesso em: 10 nov. 2026.

QWEN TEAM. Qwen3 Technical Report. arXiv, 2025. Disponível em: https://arxiv.org/abs/2505.09388. Acesso em: 9 nov. 2026.

REN, Houxing et al. Edit-Based Refinement for Parallel Masked Diffusion Language Models. arXiv, 2026. Disponível em: https://arxiv.org/abs/2605.09603. Acesso em: 9 nov. 2026.

TIAN, Keyu et al. Visual Autoregressive Modeling: Scalable Image Generation via Next-Scale Prediction. In: NEURAL INFORMATION PROCESSING SYSTEMS (NeurIPS), 2024. arXiv, 2024. Disponível em: https://arxiv.org/abs/2404.02905. Acesso em: 9 nov. 2026.

VAN DEN OORD, Aaron; VINYALS, Oriol; KAVUKCUOGLU, Koray. Neural Discrete Representation Learning. In: NEURAL INFORMATION PROCESSING SYSTEMS (NeurIPS), 2017. arXiv, 2017. Disponível em: https://arxiv.org/abs/1711.00937. Acesso em: 9 nov. 2026.

WORLD PROGRAMMING SOCIETY. Lower loss, worse stories: small NCMs trained on TinyStories. Post técnico, 2026. Disponível em: https://www.worldprogramming.org/posts/lower-loss-worse-stories-small-ncms-trained-on-tinystories-kbogee. Acesso em: 10 nov. 2026.

YANG, An et al. Qwen2.5 Technical Report. arXiv, 2024. Disponível em: https://arxiv.org/abs/2412.15115. Acesso em: 9 nov. 2026.

ZEGHIDOUR, Neil et al. SoundStream: An End-to-End Neural Audio Codec. arXiv, 2021. Disponível em: https://arxiv.org/abs/2107.03312. Acesso em: 9 nov. 2026.

ZHOU, Yongchao et al. DistillSpec: Improving Speculative Decoding via Knowledge Distillation. In: INTERNATIONAL CONFERENCE ON LEARNING REPRESENTATIONS (ICLR), 2024. arXiv, 2024. Disponível em: https://arxiv.org/abs/2310.08461. Acesso em: 9 nov. 2026.


---

*Licença: CC BY-NC 4.0 — uso exclusivamente acadêmico e de pesquisa. Artefatos e histórico completo no repositório associado.*
