# Revisão gramatical e ABNT — `paper_pt-br.md` (RUN-021-abnt-1)

**Arquivo revisado:** `paper/paper_pt-br.md`
**Normas de referência:** Acordo Ortográfico vigente; NBR 14724:2024; NBR 6028:2021; NBR 6024:2012; NBR 10520:2023; NBR 6023:2025.
**Escopo:** revisão independente, sem contexto do projeto. Ortografia, gramática, estrutura ABNT, consistência citação↔referência, unidades/terminologia e registro formal.

---

## A. ERROS REAIS

### A.1 Ortografia (Acordo vigente)

1. **"autoregressiva" / "autoregressivos"** — linhas 11, 25, 37 ("a geração autoregressiva token a token"; "Modelos autoregressivos geram"; "a arquitetura autoregressiva").
   - (b) `autorregressiva` / `autorregressivos` (auto- + r → rr, sem hífen, como em "autorretrato", "antirracista").
   - (c) Acordo Ortográfico, regra de hífen dos prefixos: prefixo terminado em vogal + elemento iniciado por r/s → dobra a consoante, sem hífen. *Nota:* "autoregressive" é uso consagrado na literatura de ML em PT-BR; normativamente, a forma correta é com -rr-. Decisão editorial: aplicar ou registrar como exceção deliberada.

2. **"re-inicializando"** — linha 120 ("`apply(_init_weights)` re-inicializando recursivamente").
   - (b) `reinicializando` (re- + vogal: sem hífen, cf. "reintegrar", "reeleição").
   - (c) Acordo Ortográfico — prefixo re- perde o hífen antes de vogal.

3. **"re-randomizava"** — linha 73 ("o apply recursivo re-randomizava todos os Linear/Embedding").
   - (b) `rerandomizava` (re- + r → rr, sem hífen); alternativa mais legível: "tornava a randomizar".
   - (c) Mesma regra do item 2.

4. **"intra-nível"** — linha 67 ("geração intra-nível por chunks causais").
   - (b) `intranível` (intra- + consoante → sem hífen, cf. "intramuscular", "intravenoso").
   - (c) Acordo Ortográfico — hífen só ante vogal idêntica, h, r ou s.

5. **"pós-fix"** — linha 102 ("As amostras pós-fix são qualitativamente outra coisa").
   - (b) `pós-correção` — mistura de palavra inglesa ("fix") com prefixo português; o restante do texto usa "correção" ("após a correção do bug", l. 100).
   - (c) Consistência lexical / empréstimo não adaptado.

### A.2 Gramática

6. **Concordância verbal** — linha 57: "Todo treinamento e benchmark de terceiros **rodou** em contêiner isolado".
   - (b) "Todo treinamento e benchmark de terceiros **rodaram**…" (ou "Toda execução de treinamento e benchmark … rodou").
   - (c) Sujeito composto de dois núcleos distintos → verbo no plural. Defensável como leitura coletiva, mas prescritivamente incorreto.

7. **Regência verbal** — linha 100: "convergiram **a** held_l11 = 2,68".
   - (b) "convergiram **para** held_l11 = 2,68" (ou "atingiram held_l11").
   - (c) "Convergir" pede preposição "para"/"a" apenas em acepções geométricas; com valor-limite o usual é "convergir para" ou "chegar a". "Convergiram a" é construção agramatical em PT-BR.

8. **Provável erro de palavra (semântico)** — linha 45: "a última milha existe e tem **capacidade** própria".
   - (b) "…tem **custo** próprio" — o contexto (0,07 conceitos/s) mede custo/latência, não capacidade. A mesma linha usa "capacidade" corretamente mais adiante ("quanta capacidade o caminho códigos→texto exige"), o que reforça a hipótese de deslize.
   - (c) Adequação vocabular; a frase como está contradiz o argumento.

9. **Provável erro de palavra (semântico)** — linha 122: "separa o que é estrutural do que é **mercado** de escala".
   - (b) "…do que é **matéria** de escala" ou "**questão** de escala".
   - (c) "Mercado de escala" não é locução estabelecida e não se sustenta semanticamente no período.

10. **Falso cognato** — linha 73: "Quatro rodadas de vereditos … estavam **confundidas**".
    - (b) "estavam **comprometidas**" / "**viciadas**" — o sentido pretendido é o de *confounded* (contaminadas por fator de confusão); "confundidas" em PT significa "misturadas/enganadas".
    - (c) Falso cognato; prejudica a leitura de quem não conhece o inglês técnico.

11. **Termo técnico** — linha 37: "cada passagem disputa **largura de memória** com o sistema inteiro".
    - (b) "largura de **banda** de memória" ou "banda de memória" (memory bandwidth).
    - (c) Tradução incompleta de termo técnico; "largura de memória" designa outra grandeza (largura de barramento/palavra).

### A.3 Consistência interna factual

12. **Contradição "três máquinas"** — linha 51: "**Três** máquinas de uso geral participaram do estudo", mas o item 3 da lista (l. 55) é a *mesma* CPU do item 2 ("mesma CPU secundária"), e a §5.2 (l. 128) afirma "**Duas** máquinas, CPUs distintas".
    - (b) Trocar para "Duas máquinas de uso geral participaram do estudo" (o servidor llama.cpp é uso adicional da segunda, não terceira máquina).
    - (c) Coerência interna factual.

13. **"ao quadruplicar os parâmetros"** — linha 88: "a razão subiu de ~1,1× para ~2,3× ao quadruplicar os parâmetros". Pela própria tabela (l. 83–84): 62–68M → 471–479M ≈ **×7,5**, não ×4.
    - (b) "ao multiplicar os parâmetros por ~7,5" (ou "ao passar de ~65M para ~475M parâmetros").
    - (c) Erro aritmético/factual contra os próprios dados da tabela.

14. **Rodadas 013 e 018 não mapeadas** — linhas 63–69: Fase A = runs 001–004; Fase B = 005–012; Fase C = 014–016; Fase D = 017–020. As runs 013 e 018 não constam de fase alguma, embora o texto afirme "vinte rodadas" (001–020). A l. 73 diz que 014–017 estavam contaminadas e a l. 73/100 que 019/020 é o primeiro teste limpo — 018 fica em limbo.
    - (b) Declarar explicitamente o destino das runs 013 e 018 (descartadas? piloto? agrupadas?).
    - (c) Completude/rastreabilidade da linha experimental — o texto se propõe a ser "relato honesto" e a lacuna chama atenção do revisor.

### A.4 ABNT — citações e referências (NBR 10520:2023 / NBR 6023:2025)

15. **Citação fora do formato autor-data da lista** — linha 45: "(**Large Concept Models**, 2024)". A entrada correspondente na lista (l. 176) é "**META AI**. Large Concept Models…".
    - (b) "(META AI, 2024)".
    - (c) NBR 10520 — a citação no texto deve remeter inequivocamente à entrada da lista; título-entre-parênteses só se usa quando não há autoria identificável.

16. **Referências órfãs (nunca citadas)** — as entradas abaixo não têm nenhuma chamada no corpo do texto:
    - CHANG (MaskGIT, l. 148) — o nome "MaskGIT" aparece na l. 94, mas sem citação formal → acrescentar "(CHANG et al., 2022)";
    - GGERGANOV (llama.cpp, l. 154) — "servidor llama.cpp" na l. 55 sem citação → acrescentar "(GGERGANOV et al., s.d.)" ou equivalente;
    - HUI (Qwen2.5-Coder, l. 158) — "Qwen2.5-0.5B" nas l. 63/65 sem citação;
    - QWEN TEAM (Qwen3, l. 180) — "Qwen3-Coder-30B-A3B-Instruct" na l. 25 sem citação;
    - KUMAR (l. 164), LEE (RQ-Transformer, l. 166), LI, L. (Sub-MoE, l. 170), LI, Y. (EAGLE, l. 172), LI, Z. (MC-SMoE, l. 174), ZHOU (DistillSpec, l. 190) — **nenhuma menção no texto**.
    - (b) Citar no corpo (§2.1 é o lugar natural para EAGLE/SpecInfer/DistillSpec/RQ-Transformer) ou remover da lista.
    - (c) NBR 10520/6023 — a lista de referências contém exclusivamente obras citadas.

17. **Tabela sem identificação** — linhas 81–86: tabela de resultados sem numeração, título ou fonte.
    - (b) "Tabela 1 – Speedup por amostra contra baseline AR casado" acima da tabela; nota de fonte ("Fonte: dados do autor (2026).") abaixo.
    - (c) NBR 14724:2024 — toda ilustração/tabela exige identificação e indicação de fonte.

18. **Formato das entradas arXiv** — todas as referências usam "*arXiv*, n. 2401.10774, 2024".
    - (b) Identificadores arXiv não são "número" serial; o recomendado: "Preprint. arXiv:2401.10774" e, sendo recurso online, "Disponível em: https://arxiv.org/abs/2401.10774. Acesso em: …". Aplicado uniformemente, o formato atual é ao menos consistente — mas "n." é tecnicamente incorreto.
    - (c) NBR 6023:2025 — documentos eletrônicos exigem endereço e data de acesso (só a entrada llama.cpp os traz).

19. **Capitalização dos títulos nas referências** — títulos em *title case* inglês ("Medusa: Simple LLM Inference Acceleration Framework…").
    - (b) NBR 6023 prescreve maiúscula inicial só na primeira palavra e em nomes próprios: "Medusa: simple LLM inference acceleration framework with multiple decoding heads". Como é uniforme, classificável como desvio leve/sistemático.
    - (c) NBR 6023:2025, regra de capitalização.

### A.5 Inconsistências de notação e terminologia (reincidentes — contam como erro de revisão)

20. **"passos" vs "steps"** — l. 69 "3.000 passos × batch 16" e l. 128 "~50k×2048 passos" vs l. 112 "14,5h/3.000 steps" e l. 110 "Mais steps". Escolher uma forma (recomendado: "passos").

21. **Notação de milhares** — "50.000 steps" (l. 112) vs "~50k×2048" (l. 128) vs "~50k passos, batch 2k" (l. 136); "batch 2.048" (l. 112) vs "batch 2k" (l. 136); "[2048,1024,512,256,128]" (l. 94) sem separador vs "8.192 tokens"/"4.096 classes" com ponto; "177k"/"152k"/"43k" por extenso-k. Padronizar (sugestão: ponto de milhar no corpo do texto; notação de código só em trechos literais).

22. **"passagens" vs "passes"** — l. 37/88 "passagem de forward", "6–12 passagens totais" vs l. 41 "4–5× menos passes de treino", l. 94 "custou passes extras". Unificar ("passagens" é a forma PT-BR já dominante).

23. **"perda" vs "loss"** — resumo (l. 11) "abaixo da perda" vs l. 94/100 "o loss do platô", "o loss held total". Unificar ou marcar "loss" como termo técnico (itálico).

24. **"rodada" vs "run"** — "vinte rodadas" (l. 11, 31, 61) vs "runs 001–005", "run contaminada", "quatro runs" (l. 53, 100, 120). Aceitável usar "run" para a unidade versionada e "rodada" no discurso, mas o texto não sinaliza essa distinção — ao menos uniformizar dentro de cada construção.

25. **"paper" vs "artigo/trabalho/estudo"** — "paper" aparece 6× (l. 31, 79, 92?, 118, 128 ×2) em texto que já usa "artigo"/"trabalho"/"estudo". Unificar (recomendado: "artigo" para o original, "trabalho/estudo" para o próprio).

26. **"código→token" vs "conceito→token" vs "códigos→texto"** — a mesma "última milha" é nomeada de três formas (l. 45, 65, 110, 124, 138). Unificar a seta canônica (sugestão: "código→token", que é o que o decoder faz).

27. **"l0 não-aprendido" vs "l0 não aprendido"** — l. 112 (com hífen) vs l. 128 (sem hífen). Escolher uma forma (ambas defensáveis; como adjetivo composto, "não-aprendido" é preferível).

28. **"codebook pirâmide" vs "codebook piramidal"** — l. 65/110 ("pirâmide") vs l. 122 ("piramidal"). Unificar ("piramidal" é o adjetivo regular).

29. **"14,5 horas" vs "14,5h"** — l. 57 vs l. 112. Padronizar unidade ("14,5 h" com espaço, forma SI, ou "horas" por extenso).

30. **"0 accepts"** — l. 124: "(0/40, 0 accepts)". Inglês solto: "0 aceitações" / "nenhum token aceito".

31. **"não de decode"** — l. 124: "não de decode" — "decode" como substantivo inglês em meio a "decodificação"/"decoder" usados no resto do texto → "não de decodificação".

32. **"drafts destilados"** — l. 27: lista de termos traduzidos ("rascunhos especulativos, árvores de verificação") encerra com o inglês "drafts destilados" → "rascunhos destilados" (ou "drafts" em itálico, se for termo próprio).

33. **"vs" sem ponto** — l. 67 ("NCM 69M vs AR 67M") e l. 140 ("códigos residuais vs códigos alinhados"). Em registro formal: "vs." com ponto ou "versus"/"contra".

---

## B. SUGESTÕES DE ESTILO (registro, clareza — não são erros)

34. **L. 5** "Pesquisa independente — novembro de 2026": como filiação do autor (NBR 14724), "Pesquisador independente" designa a pessoa; "pesquisa independente" designa a atividade.

35. **L. 11 (resumo)** "durante onze dias de experimentação, vinte rodadas registradas de hipótese, medição e veredito" — a justaposição por vírgula é solta; sugerido: "…experimentação e vinte rodadas registradas de hipótese, medição e veredito". Resumo tem ~300 palavras: dentro do intervalo NBR 6028 para relatórios (150–500), acima do usual para artigo (100–250) — verificar o veículo-alvo.

36. **Palavras-chave** (l. 13): NBR 6028 prescreve separação por **ponto** ("modelos de linguagem. geração paralela de texto. …"); o ponto e vírgula é convenção de periódicos, não da norma estrita.

37. **L. 25** "sem quantizar" → "sem quantização" (nominal mais formal). Idem "dezenas de gigabytes" (l. 25) vs "GB" em todo o restante — uniformizar.

38. **L. 27** "— rascunhos especulativos, árvores de verificação, drafts destilados —, o NCM propõe": travessão + vírgula é dupla marcação; escolher uma (recomendado: "…truques de sistema — rascunhos especulativos, árvores de verificação e rascunhos destilados — o NCM propõe").

39. **L. 27** "chamou atenção" → "chamou a atenção" (artigo). "K passagens por nível em vez de L passagens por token" é ambíguo (lê-se "K por nível"); sugerido: "K passagens no total — uma por nível — em vez de L passagens, uma por token".

40. **L. 31** "vereditos que descobrimos depois estarem contaminados por um bug nosso" → "vereditos que, como descobrimos depois, estavam contaminados" — a ordem atual é gramatical, mas força releitura.

41. **L. 31/128** "braços casados" / "braços" — calque de *matched arms*; em PT-BR o mais corrente fora de ensaios clínicos é "condições casadas" ou "ramos".

42. **L. 37** "para um survey completo" → "para uma revisão completa" (único anglicismo solto entre termos traduzidos).

43. **L. 45** "resolve diferente" → "resolve de modo diferente"; "espelho" em vez de "mirror" se quiser máxima adaptação (mantendo "Mimir-1.6B" como nome próprio).

44. **L. 57** "o orçamento total foi aproximadamente 300 horas" → "foi **de** aproximadamente 300 horas".

45. **L. 73/96/112** "condicionar em" (*condition on*) é calque técnico aceito na área, mas a própria l. 41 usa a regência padrão "condicionado **à** reconstrução". Unificar para "condicionado a" ou assumir o calque com consistência.

46. **L. 11/96** "fatoram erro de reconstrução" / "fatora por erro" — "fatorar" no sentido de *factor by* é calque; se fizer sentido, "decompõe-se por erro de reconstrução" ou "segmenta-se pelo erro".

47. **L. 112/142** "num l0" / "Numa era" — contrações informais de "em um(a)"; em texto acadêmico, "em um"/"Em uma".

48. **L. 124** "nossa projeção nearest-embedding é grátis" → "é gratuita"/"não tem custo"; "onde o token errado é um programa que não compila" → "em que" (antecedente não espacial).

49. **L. 128** "pode ser puramente orçamento" → "pode ser puramente uma questão de orçamento".

50. **L. 136/140** "um nó de GPU decente" → "adequado"/"razoável"; "Cada um sai direto dos artefatos" → "decorre diretamente".

51. **Próclise × ênclise** — mistura: "reproduziu-se" (l. 79), "manteve-se" (l. 88), "provaram-se" (l. 100) vs "se inverteu" (l. 100). Ambas legítimas em PB; para uniformidade formal, ênclise após pausa/sujeito explícito é a norma de prestígio — ou assumir a próclise brasileira de modo consistente.

52. **Estrangeirismos sem destaque** — "speedup", "baseline", "forward", "nested dropout", "ground-truth", "exposure bias", "warm-start" etc. aparecem em romano. NBR 14724 recomenda itálico para termos estrangeiros não consagrados. Uso atual é consistente — manter como está ou aplicar itálico a todos; não misturar.

53. **L. 96** "estilo MaskGIT/VAR" e demais nomes de métodos entre parênteses nas citações (l. 37: "— Medusa", "— CLLMs") — anotações informais dentro da chamada autor-data são legíveis e úteis ao leitor, mas fora da literalidade da NBR 10520; alternativa: "(CAI et al., 2024; método Medusa)".

54. **Ausência de sumário/lista de ilustrações** — para formato de relatório técnico NBR 14724 completo seriam esperados; para formato de paper são dispensáveis. Registrar apenas como nota estrutural.

---

## Veredito

**CORREÇÕES NECESSÁRIAS.**

A prosa é competente, com voz autoral forte e controle estilístico acima da média — os achados de estilo são refinamentos, não defeitos. O que impede "IMPECÁVEL" é um conjunto pequeno mas concreto de erros: duas contradições factuais internas ("três máquinas" × duas; "quadruplicar" × ×7,5), duas prováveis trocas de palavra ("capacidade"→"custo", "mercado"→"matéria"), uma citação fora do padrão autor-data (META AI), seis referências órfãs e quatro menções sem citação, uma tabela sem identificação, deslizes ortográficos pós-Acordo (autorregressiva, reinicializando, rerandomizava, intranível) e a lacuna das runs 013/018. Todos pontuais e de correção barata — nenhuma reescrita de trecho é necessária.
