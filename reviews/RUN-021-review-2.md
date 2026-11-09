# Revisão científica independente — RUN-021-review-2

**Objeto:** `paper_pt-br.md` e `paper_en-us.md` (replicação parcial do NCM), com
`SOURCES.md`, `TIMELINE.md`, `artifacts/metrics/*.json` e
`artifacts/samples/representative_samples.json` como base de verificação.
**Papel:** revisor independente (sistemas/ML efficiency, decodificação paralela,
quantização vetorial), sem contexto do projeto gerador. Foco: rigor técnico,
justeza das comparações, correção conceitual (VQ-VAE, speculative decoding,
diffusion LMs, MoE), reprodutibilidade e classificação provado/refutado/aberto.
**Método:** todo número citado no texto foi conferido contra o artefato
apontado em `SOURCES.md`; quando o artefato não continha o número, isso foi
registrado. Linhas citadas referem-se a `paper_pt-br.md`; a versão EN-US é
parágrafo a parágrafo equivalente e carrega os mesmos problemas.

---

## Veredicto final: **APPROVE_WITH_REVISIONS**

O relato é tecnicamente sério e incomumente honesto: a autoria reconhece e
documenta o bug de inicialização, lista limitações explícitas, mantém baselines
casados em parâmetros/dados/máquina, e a maioria dos números citados é
rastreável a artefatos brutos coerentes (uniform CEs = ln V corretos;
`codes_per_sample` = 249 = soma dos níveis [1,8,16,32,64,128]; amostras v2
exibem exatamente a assinatura qualitativa descrita — fórmulas TinyStories +
stutter de bigramas). O mecanismo do bug descrito (`self.apply(_init_weights)`
re-randomizando recursivamente um submódulo pré-treinado) está tecnicamente
correto em semântica PyTorch.

Ainda assim, há problemas de ordem **major** que precisam de revisão antes que
as conclusões possam ser aceitas como reportadas: (i) a janela de contaminação
declarada (runs 014–017) não tem evidência no próprio relato e provavelmente é
estreita demais — os artefatos da Fase A exibem a mesma assinatura do bug;
(ii) o "achado científico mais útil" (§4.2) deriva inteiramente de rodadas que
a própria §3.3 declara confundidas; (iii) o único braço com tokenizador oficial
e pipeline limpa é *mais lento* que o AR casado — contradição não reportada com
o modelo de custo central do texto; (iv) os números centrais da run limpa não
existem em nenhum artefato publicado.

Não é REJECT porque o cerne narrativo sobrevive às correções, os benchmarks de
tempo são válidos independentemente do bug (wall-time não depende de valores de
pesos), e todas as majors são endereçáveis por revisão de texto + re-auditoria
(idealmente com dois experimentos CPU pequenos: re-teste limpo do tokenizador
local e um braço AR na Fase D).

---

## MAJOR

### M1 — Janela de contaminação "runs 014–017" é afirmada sem evidência e provavelmente está errada

- **Linha:** `paper_pt-br.md` L73 ("nas runs 014–017, `LevelGenerator.__init__`
  registrava o módulo do VQ-VAE antes de `self.apply(_init_weights)`"); TIMELINE
  L17; `paper_en-us.md` L71.
- **Evidência:** o paper afirma uma janela precisa (014–017) sem explicar o
  limite inferior. Mas os artefatos da Fase A mostram a assinatura exata do bug
  descrito — "decodificação usava um decoder aleatório, produzindo salada de
  palavras para qualquer código": `bench.json` (NCM 68,8M) e `bench_471m.json`
  (NCM 478,6M) contêm amostras NCM que são word-salad puro, indistinguíveis das
  saídas das runs 014–017 admitidas como contaminadas.
- **Corroboração no repositório de código associado** (`<repositório-privado-de-código>`, inspecionado
  para verificação): em *toda* revisão commitada de
  `src/<pkg>/model/generator.py` que contém `self.apply(_init_weights)`,
  `self.vqvae = vqvae` está registrado **antes** do apply (introduzido no commit
  commit de 30/out, quando `self.apply` foi adicionado com `self.vqvae` já na
  linha 32; o HEAD da época ainda contém a ordenação defeituosa). A correção de
  08/nov (RUN-018) existe apenas **não commitada** no working tree. E
  `scripts/train_generator.py` — o trainer das Fases A e C — constrói
  `LevelGenerator` com o VQ-VAE pré-treinado carregado de `--vqvae
  artifacts/tokenizer.pt`, exatamente o caminho que dispara o bug. Nada na
  história de código sobrevivente indica um período em que a ordenação estivesse
  correta antes da correção de 08/nov.
- **Por que importa:** (a) se o bug existia desde a introdução de `self.apply`
  (30/out), as amostras NCM das Fases A também são artefatos do bug — o relato
  deveria dizer "todas as runs de LevelGenerator anteriores a RUN-018", não
  "quatro rodadas"; (b) reforça M2 abaixo — toda a evidência de qualidade da
  Fase C está comprometida; (c) a contagem "quatro rodadas invalidadas" está
  subestimada — embora os *vereditos* invalidados sejam de fato os de 014–017
  (a Fase A só gerou timings, que permanecem válidos), as amostras de 001–004
  citadas no pipeline são igualmente bug-artifacts e deveriam ser marcadas.
- **Correção sugerida:** auditar e declarar explicitamente a janela real —
  "o `self.apply` foi introduzido em <data/commit>; toda run de LevelGenerator
  entre então e RUN-018 condicionou num tokenizador embaralhado; vereditos
  dependentes de saída do gerador existem apenas em 014–017; timings são
  imunes". Se o limite inferior de fato for 014, apresentar a evidência
  (diff/log) que o sustenta.

### M2 — O "achado mais útil" (§4.2) deriva de medições que a própria §3.3 declara inválidas

- **Linha:** L94–96 (as três "hipóteses rejeitadas" e a "medição que fecha a
  questão", boundary CE); L110 (Refutado ii: "o gargalo era representacional,
  não de orçamento, nos nossos tokenizadores locais"); L122 ("produz códigos
  cuja porção fina é irremediavelmente imprevisível"); README L31–32 repete o
  veredito.
- **Evidência:** `boundary_ce_intra.json` é artefato da RUN-016; taper/resume
  (RUN-015) e intra-nível (RUN-016) são da janela 014–016 — todas contaminadas
  pelo bug de init (a §3.3 admite explicitamente que "os níveis finos são
  imprevisíveis" estava entre os vereditos confundidos). Num pipeline onde o
  treino condiciona em embeddings aleatórios e a avaliação alimenta códigos
  reais pelo mesmo decoder aleatório, "condicionar em códigos reais não reduz
  CE" é a previsão *trivial* do bug — a medição não pode distinguir "códigos
  independentes" de "caminho de condicionamento corrompido". Adicionalmente,
  CE plana mostra no máximo que *aquele modelo* não explorou dependência;
  afirmar I(código_i; código_j | grosso) ≈ 0 é um salto de "não aprendido"
  para "inexistente" — CE de um modelo é bound frouxo sobre entropia
  condicional verdadeira. (Há ainda um segundo confundidor documentado na
  própria revisão cega da RUN-016: mismatch de interpolação `area` vs `linear`
  entre treino e geração no modo intra — as amostras intra estavam duplamente
  corrompidas.)
- **Agravante interno:** o retreino limpo (018–020) usou o tokenizador
  **oficial** — o tokenizador *local* nunca foi re-testado limpo. Assim, a
  comparação "o tokenizador oficial produz níveis finos mais aprendíveis que os
  nossos" (L122) compara medição limpa (oficial, l11=2,68) com medição
  contaminada (local) — apples-to-oranges. A hipótese é *plausível* (RVQ
  sub-treinado tende a códigos residuais ruidosos; o salto k5→k6 de
  36%→97% em `level_ablation.json` confirma que os níveis finos carregam o
  detalhe token-level), mas não foi *demonstrada*.
- **Correção sugerida:** rebaixar §4.2 de "medição decisiva/refutado" para
  "consistente com a hipótese, medido sob pipeline corrompido; não demonstrado
  limpo" — ou re-rodar o experimento boundary-CE com o tokenizador local sob o
  código corrigido (custo CPU pequeno). Mover "Refutado (ii)" para "em aberto —
  testes impossibilitados pelo bug": sob um decoder aleatório, *nenhuma* das
  três correções poderia ter produzido texto coerente, logo elas não foram
  falseadas.

### M3 — Fase D não tem braço AR, e o único NCM limpo end-to-end é ~2× MAIS LENTO que o AR casado

- **Linha:** L83–88 (tabela + "6–12 passagens totais; o AR casado, em ~127 ...
  deve se preservar ou ampliar em qualquer hardware"); L136 ("o paradigma
  entrega velocidade real e escalável já em miniatura").
- **Evidência:** `bench_v2.json` (run limpa, 64M, 12 níveis oficiais):
  **1,306 s/amostra em 12 passes**. O AR casado de 67M no mesmo domínio e
  máquina (`bench_ts.json`, 16 threads): **0,617 s/amostra**; mesmo nas
  observações mais lentas do AR (0,883/0,978 s em `bench_ts2`/`bench_ts_resume`),
  o NCM oficial é ~34–112% mais lento. A conta é simples: passes ≠ custo igual —
  um passo AR com KV-cache (~5–8 ms) é uma forward de 1 token; um passe NCM
  (~109 ms) é forward de sequência completa sobre a hierarquia acumulada +
  decode VQ para condicionamento. "O AR paga 127 passagens, o NCM paga 6–12"
  só se converte em speedup quando o custo por passe é comparável — verdade
  para o tokenizador próprio de 6 níveis (36 ms/passe), falso para o oficial de
  12 níveis no próprio hardware medido.
- **Por que importa:** o paper omitiu o braço AR da Fase D e o contra-exemplo
  à sua própria tese de custo. A afirmação "deve se preservar ou ampliar em
  qualquer hardware" é plausível em GPU (um passe NCM paraleliza bem; um passo
  AR não), mas é empiricamente **falsa na CPU do estudo** para a configuração
  oficial — e o dado que a refuta está no repositório.
- **Correção sugerida:** reportar a comparação Fase D (NCM 1,306 s vs AR ~0,6–1,0
  s/amostra) e reformular: "o speedup estrutural se materializa quando o custo
  por passe permanece próximo do regime memory-bound (hierarquia rasa) ou
  quando há paralelismo de hardware ocioso (GPU); na nossa CPU, o braço com
  tokenizador oficial foi mais lento que o AR casado". Isso também qualifica a
  projeção (ii) da conclusão.

### M4 — Os números centrais da run limpa não existem em nenhum artefato publicado

- **Linha:** L100 ("held_l11 = 2,68, **abaixo dos ~2,75 do próprio gerador de
  referência**"; "held total caiu de 7,55 ... para 5,82"); L102 ("held_l0 ≈
  8,22"); `SOURCES.md` L25–26.
- **Evidência:** `grep` sobre todo `artifacts/` não encontra 5,82 / 2,68 /
  8,22 / 2,75 / 7,55 / 6,81 / 8,20. `SOURCES.md` mapeia a linha "RUN-019/020 v2
  limpo | held_loss 5,82; held_l11=2,68 (< oficial 2,75); l0=8,22" para
  `bench_v2.json` e `representative_samples.json` — **nenhum dos dois contém
  esses números** (ambos só têm tempo de geração e textos). Os valores existem
  apenas no log de treino (`metrics.jsonl` do contêiner), não publicado. Ou
  seja: o resultado positivo mais importante do trabalho — "códigos oficiais
  são aprendíveis por gerador pequeno, convergindo abaixo da referência" — é
  atualmente não verificável a partir dos artefatos.
- **Problema secundário — "~2,75 do gerador de referência" é indefinido:** de
  onde vem esse número? Do paper original (outro split/protocolo)? De uma
  avaliação própria do checkpoint oficial sobre o held set local? Sob qual
  condicionamento (teacher-forced GT? corrupção U(0,1)? amostrado)? Um
  gerador de 64M a 1/2000 do orçamento bater a perda do modelo de referência é
  extraordinário e *exige* a definição do protocolo — especialmente porque a
  held-eval local roda sob a receita de corrupção (ver nota da revisão cega
  RUN-018: `held_loss` é o objetivo corrompido), o que muda a comparabilidade
  com qualquer número publicado.
- **Correção sugerida:** publicar o `metrics.jsonl`/held-eval da run v2 como
  artefato (ex.: `artifacts/metrics/eval_official_v2.json` com per-level CEs);
  definir explicitamente o "2,75" (fonte, split, condicionamento); se for a
  avaliação local do checkpoint oficial, afirmar isso — a comparação passa a
  ser legítima e ainda mais informativa.

### M5 — Estatística dos speedups: n=8, run única, sem variância; braço AR varia 58%

- **Linha:** L83–88 (tabela), L11/L17 (abstracts: "até 3,22×").
- **Evidência:** cada ponto da tabela é uma única wall-run de n=8 amostras, sem
  repetições, warm-up ou barras de erro. O braço AR *na mesma máquina* mede
  0,617 s (`bench_ts`), 0,883 s (`bench_ts2`), 0,978 s (`bench_ts_resume`) —
  spread de 58%. O speedup ×3,22 do braço taper vem em parte dessa variação:
  o NCM ficou *mais lento* em absoluto (0,219→0,274 s) e a razão melhorou
  porque o AR degradou mais. A tabela apresenta ×3,22 como efeito do taper —
  não demonstrado; e o abstract lidera com o máximo (3,22) em vez do intervalo.
  A própria limitação (iv, L128) admite "±20% entre máquinas" — a variação
  intra-máquina observada é maior que isso.
- **Correção sugerida:** reportar mediana de ≥3 execuções ou, mínimo, apresentar
  os números como single-run com a variância observada do braço AR; no
  abstract, "×1,14 → ×2,27 → ~×2,8–3,2". Também notar que as 8 amostras AR são
  idênticas byte-a-byte entre os quatro benches (mesmo seed/prompt) — útil
  para pareamento, mas reduz a independência estatística.

### M6 — Diagnóstico da repetição: a explicação mais parsimoniosa não é considerada

- **Linha:** L102 ("A métrica que explica ambos: o nível 0 ... ficou em held_l0
  ≈ 8,22"); L112 (hedge correto: "plausivelmente exposure bias").
- **Evidência:** as amostras v2 (`representative_samples.json`, `bench_v2.json`)
  mostram stutter *local* de bigramas ("was was", "Lucy Lucy", "again again
  again again"). Esse é o sintoma clássico de **independência condicional
  intra-nível**: todos os códigos de um nível são amostrados em paralelo,
  condicionados apenas nos níveis anteriores — posições adjacentes recebem
  marginais quase idênticas e amostram o mesmo token. É também exatamente o
  modo de falha que a própria investigação intra-nível (RUN-016) tentava
  tratar, e conecta o resultado limpo ao achado (confundido) de §4.2. Um l0
  não-aprendido explica bem a *ausência de arco narrativo*; explica mal o
  stutter local, que persistiria mesmo com l0 perfeito se as posições do nível
  fino são ~independentes dado o grosso. As duas hipóteses são testáveis e
  separáveis (ex.: medir correlação de bigramas na saída vs marginais
  condicionais amostradas independentemente).
- **Correção sugerida:** listar "independência condicional intra-nível" como
  explicação candidata — arguably *leading* — para a repetição, ao lado de
  exposure bias/l0; o hedge de L112 está correto, o assertivo "a métrica que
  explica ambos" de L102 está errado.

---

## MINOR

| # | Severidade | Linha (pt-br) | Achado | Correção sugerida |
|---|---|---|---|---|
| m1 | minor | L88 | "**ao quadruplicar** os parâmetros" — 62/68,8M → 471/478,6M é ~7×, não 4× | "ao multiplicar os parâmetros por ~7" |
| m2 | minor | L96 | "l3: 6,21→6,22" — artefato tem b0=6,2049→b28=6,2205 (≈6,20→6,22); e "90% das posições" — b28/32=87,5%, b57/64=89,1%, b115/128=89,8% | "≈6,20→6,22"; "~88–90%" |
| m3 | minor | L108 | Provado (iv): codebook pirâmide "**viável**" — top-1 32,1% existe (`pyramid_cb_qwen05_report.json`), mas *todo* uso downstream falhou (0/40, accept 0, ~34% mais lento) | "construível sem treinar VQ-VAE" — viável sugere utilidade demonstrada |
| m4 | minor | L102, L128 | "l0 ficou ≈8,22, **indistinguível** do uniforme 8,32" vs "l0 começou a sair do uniforme (8,36→7,97)" — ambos no mesmo texto; e o 7,97 não está em nenhum artefato | distinguir held (8,22) de train tardio (7,97); publicar a trajetória |
| m5 | minor | `SOURCES.md` | Números sem artefato correspondente: rank mediano ~43.505 (não há campo de rank em `codepath_eval.json`); VQ-VAE held_ce 0,152; taper 0,654/100% utilização (ausentes de `bench_ts2.json`); trajetória l0; Mimir 0,07 conceitos/s (RUN-008 sem JSON) | publicar os JSONs de eval ou marcar "log de treino, não arquivado" |
| m6 | minor | L136 | "~2.000× nosso orçamento" é ×2.133 em *exemplos vistos* (50k×2048 / 3k×16); em FLOPs o fator difere (oficial 88–600M vs 64M local; profundidades diferentes); "horas em um nó de GPU decente" — 14,5 h-CPU × ~2.000 ≈ 29k h-CPU ≈ dias em um nó decente, não horas | especificar a unidade (exemplos/FLOPs/horas); "dias num nó de GPU" é mais defensável |
| m7 | minor | L11/L17 | "confirmou a hipótese central do artigo original" — a claim original é speedup *com qualidade comparável vs difusão*; aqui o baseline é AR casado e a qualidade NCM é word-salad → confirma-se o *modelo de custo*, não a hipótese ponta-a-ponta | "confirmou a premissa de custo do paradigma (menos viagens sequenciais)" |
| m8 | minor | L27/L25-en | "K passagens por nível em vez de L passagens por token" — ambíguo: são K passes *totais* (um por nível), não K por nível | "uma passagem por nível, K níveis ao todo" |
| m9 | minor | L94 | "custou passes extras (×1,22 apenas)" — ×1,22 é o speedup *residual* vs AR; o custo real foi passes 6→10 (+67%) e tempo NCM 0,219→0,513 s (×2,34) | "passes 6→10 (+67%), tempo do NCM ×2,34, margem residual ×1,22" |
| m10 | minor | `SOURCES.md` L17 | "codes top-1 truth 3,1%→2,34%" combina métricas de evals diferentes (a1=3,125% em `token_head_ar_eval` vs t1=2,34% em `token_head_eval2`) como se fosse evolução | separar os dois experimentos/métricas |
| m11 | minor | — | Reprodutibilidade externa é nula: repo público não tem código, configs, splits, seeds (só v2/official têm `seed:0`), hiperparâmetros (LR, optimizer, schedule, profundidades por escala — "MiniMSRQ" só aparece em TIMELINE), nem como um warm-start de 0,5B vira gerador de 62M | tabela de config por fase + apontar o repo de código, ou declarar explicitamente que código/checkpoints não são distribuídos |
| m12 | minor | L128 | limitação (iv) "±20% entre máquinas" subestima a variação observada (58% intra-máquina no braço AR) | revisar para refletir a variância medida |

## NIT

| # | Linha | Achado |
|---|---|---|
| n1 | L45, TIMELINE L13 | "Mimir-1.6B (mirror dos pesos LCM)" / "LCM real" — Mimir é reimplementação comunitária (two-tower diffusion), não os pesos da Meta; dizer "implementação comunitária do LCM" |
| n2 | Referências | Citadas na lista mas nunca no texto: KUMAR (RVQ para KV-cache), Sub-MoE, MC-SMoE, DistillSpec (ZHOU), RQ-Transformer (LEE), EAGLE, HUI (Qwen2.5-Coder) — citar ou remover |
| n3 | L73 vs TIMELINE L18 | "A run 019/020 é o primeiro teste limpo" vs TIMELINE "RUN-018–020: retreino limpo" — RUN-018 foi o retreino; reconciliar numeração (RUN-013 também ausente em toda a numeração) |
| n4 | L63 | "LevelGenerator ~62M" vs artefato 68,8M (`bench.json` params); "~471–479M" na tabela vs 477,6/478,6M reais — de onde sai 471? (nome nominal, esclarecer) |
| n5 | `bench_v2.json`, `bench_official.json` | sem campo `hardware`/`cpu_count` — o leitor não pode confirmar mesma máquina/threads dos demais benches |
| n6 | L96 | "o resto de maior entropia" — gloss solto: residual é erro de reconstrução por MSE, não literalmente a componente de máxima entropia |
| n7 | L102 | "l0 — código único de 4.096 classes que carrega a identidade inteira da estória" — interpretação razoável mas assertiva; os `levels` do tokenizador oficial não são documentados no paper |
| n8 | `eval*.json` | `distinct_word_ratio` NCM > AR (0,65 vs 0,38) — assinatura de amostragem ~uniforme (salada tem diversidade lexical alta); não citado, mas leitores podem ler ao contrário se descobrirem |
| n9 | L31 | "vinte rodadas" vs ~18 runs numeradas + bootstrap — aproximação aceitável, mas TIMELINE poderia listar a contagem exata |

---

## O que está correto e merece registro (para não ler como reprovação total)

1. **Metodologia de braços casados** (parâmetros ±3%, mesmos dados, mesma
   máquina/threads) é o padrão correto para a pergunta feita — raro em relatos
   desse tipo.
2. **O bug de init está tecnicamente correto** na descrição (apply recursivo,
   efeito silencioso, gate-zero por checksum) — e a recomendação metodológica
   ("verificar preservação de pesos por inspeção") é genuinamente útil.
3. **Consistência interna dos artefatos:** uniform CEs = ln(V) corretos em todos
   os níveis (6,238/5,545/4,852/8,32); `codes_per_sample` 249 = soma dos
   níveis; `level_ablation` k0→k6 monotônico e coerente (k6=96,9% < 100% —
   reconstrução imperfeita mesmo com todos os níveis, honesto).
4. **Speculative decoding bem compreendido e bem reportado:**
   `text_equal_ar=True` com accept=0 — exatidão preservada, aceleração ausente —
   distinção que muitos trabalhos confundem.
5. **Amostras v2 batem exatamente com a descrição qualitativa** (fórmulas
   TinyStories + stutter + ausência de arco) — raro ver texto gerado publicado
   sem curadoria.
6. **Hedging apropriado** em "Em aberto" (l0 "pode ser puramente orçamento";
   exposure bias "plausivelmente") — com exceção dos pontos M2/M6 acima.
7. **O veredito "o gargalo é escala/orçamento, não arquitetura"** para l0 é
   classificado corretamente como provisório — embora uma alternativa
   arquitetural (um único código 4096-way para identidade global é
   intrinsecamente difícil por single-shot) merecesse menção.
8. **A comparação NCM-oficial vs AR nunca foi feita e o paper não alega que foi**
   na tabela — a omissão é de análise/contexto, não fabricação de número (M3 é
   sobre não reportar o contra-exemplo implícito nos próprios dados).

## Recomendação editorial resumida

Para APPROVE: endereçar M1–M4 (re-escopo da janela de contaminação; rebaixar
§4.2 para "consistente/não-demonstrado"; reportar a comparação Fase D; publicar
os held-metrics da v2 com definição do "2,75"), mais M5–M6 conforme viável.
Os minors/nits são de revisão direta. O resultado — "o paradigma entrega o
mecanismo de custo em CPU; a receita de qualidade exige orçamento de cluster;
a janela do bug ensina metodologia" — é defensável e provavelmente correto em
espírito; o relato precisa que as etiquetas provado/refutado reflitam o que de
fato foi medido limpo.
