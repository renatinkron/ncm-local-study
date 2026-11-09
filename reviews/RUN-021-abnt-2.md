# RUN-021 — Revisão gramatical ABNT independente (revisor 2)

**Arquivo revisado:** `paper/paper_pt-br.md` (194 linhas)
**Revisor:** segundo revisor gramatical independente — revisão a frio, sem contexto do projeto.
**Escopo:** ortografia/acentuação (AO 1990), morfossintaxe, pontuação, NBR 14724:2024, NBR 6028:2021, NBR 6024:2012, NBR 10520:2023, NBR 6023 (lista de referências), coesão e registro formal.

**Verificados conformes (não repetidos abaixo):** estrutura autor–data correta nas citações parentéticas que existem; ordem alfabética da lista de referências; ordem dos elementos pré-textuais (título → autor → resumo → palavras-chave → abstract → keywords); resumo (324 palavras) e abstract (271 palavras) dentro do intervalo NBR 6028 (150–500), parágrafo único, equivalência semântica PT↔EN confirmada; palavras-chave (5, separadas por `;`, minúsculas, ponto final) espelham as keywords; vírgula decimal consistente (`0,654`, `8,32`); numeração progressiva de seções coerente (1–6 com subseções), "Referências" não numerada (correto per NBR 6024); hífens corretos em "pré-treinado", "re-randomizava" (re + r), "pós-fix"; uso correto de "estória" (narrativa) vs "história"; "platô", "através" e crases de "condicionado à reconstrução" corretos; mesóclise formal "provaram-se aprendíveis" (linha 100) impecável.

---

## ERROS

**E1. Hífen indevido após "não"** — linha 112.
> Trecho: "condicionados num l0 não-aprendido"
> Correção: "condicionados num l0 não aprendido".
> Norma: AO 1990 — "não" não forma composto hifenizado com adjetivo/particípio; hífen só em substantivos lexicalizados ("não-pagamento").

**E2. Hífen indevido no prefixo "intra-"** — linhas 67, 94, 110 (3 ocorrências).
> Trecho: "geração intra-nível por chunks causais" / "condicionamento intra-nível" / "condicionamento intra-nível não curam".
> Correção: "intranível" — prefixo terminado em vogal + elemento iniciado por consoante aglutina sem hífen ("intramuscular", "intracelular").
> Norma: AO 1990, regra de hifenização por prefixação.

**E3. Omissão da preposição "de" em medidas de memória** — linhas 53 e 54.
> Trecho: "23 GB RAM" ; "32 GB RAM".
> Correção: "23 GB de RAM", "32 GB de RAM" — o próprio texto usa "23–32 GB de RAM" na linha 11 e "16 GB de RAM" na linha 25.
> Norma: regência nominal / consistência interna.

**E4. Omissão da preposição "de" em "N M parâmetros"** — recorrente: linhas 11 ("60M parâmetros", "471M"), 41 ("~88M a ~600M parâmetros"), 63 ("~62M parâmetros"), 67 ("69M vs AR 67M"), 69 ("64M"), tabela l. 83–86.
> Trecho: "modelos de 60M parâmetros".
> Correção: "modelos de 60 milhões de parâmetros" ou "60M de parâmetros". A construção "60M parâmetros" é calque do inglês.
> Norma: regência/complemento nominal do português; clareza (NBR 14724, apresentação).

**E5. Concordância verbal com sujeito composto** — linha 57.
> Trecho: "Todo treinamento e benchmark de terceiros rodou em contêiner isolado".
> Correção: "…rodaram em contêiner isolado" (sujeito composto anteposto → plural).
> Norma: concordância verbal — sujeito composto.

**E6. Citação sem correspondência com o elemento de entrada da referência** — linha 45.
> Trecho: "O LCM da Meta (Large Concept Models, 2024)".
> Problema: a entrada na lista é "META AI. Large Concept Models…" (linha 176); a citação deve usar o elemento de entrada.
> Correção: "(META AI, 2024)".
> Norma: NBR 10520 (citação autor–data = elemento de entrada) + NBR 6023.

**E7. Referências órfãs — itens listados sem citação no texto** — linhas 146–190 (NBR 6023: a lista contém *apenas* obras citadas).

| Referência | Situação no corpo |
|---|---|
| CHANG et al. 2022 (MaskGIT) | "MaskGIT" só mencionado (l. 94), sem "(CHANG et al., 2022)" |
| KUMAR 2024 | nunca mencionado |
| LEE et al. 2022 (RQ-Transformer) | nunca mencionado |
| LI, L. et al. 2025 (Sub-MoE) | nunca mencionado |
| LI, Y. et al. 2024 (EAGLE) | nunca mencionado |
| LI, Z. et al. 2023 (MC-SMoE) | nunca mencionado |
| ZHOU et al. 2024 (DistillSpec) | nunca mencionado |
| QWEN TEAM 2025 | "Qwen3-Coder-30B" citado como produto (l. 25), sem autor–data |
| HUI et al. 2024 (Qwen2.5-Coder) | "Qwen2.5-0.5B" citado como produto (l. 63, 65) — e o modelo citado não é o Coder |
| GGERGANOV et al. (llama.cpp) | produto citado (l. 55) sem "(GGERGANOV et al.)" |

> Correção: inserir as citações autor–data no texto ou remover as entradas. As entradas Sub-MoE/MC-SMoE/EAGLE/DistillSpec/RQ-Transformer/Kumar sugerem remanescente de referencial de outro trabalho (decodificação especulativa/MoE) — provável copiar-e-colar.
> Norma: NBR 6023 — pertinência lista↔texto.

**E8. Documentos online sem "Disponível em:" e "Acesso em:"** — quase todas as entradas arXiv (linhas 146–190); só GGERGANOV (l. 154) tem URL + data de acesso e KUMBONG tem URL de checkpoints (mas não do artigo).
> Correção: acrescentar "Disponível em: https://arxiv.org/abs/<id>. Acesso em: <data>." a cada entrada online.
> Norma: NBR 6023 — elementos obrigatórios para documento eletrônico.

**E9. Tabela sem identificação** — linhas 81–86.
> Problema: a tabela de speedups não tem numeração nem título.
> Correção: acima, "Tabela 1 – Speedup por amostra: NCM vs baseline AR em três escalas"; abaixo, "Fonte: elaborada pelo autor (2026)".
> Norma: NBR 14724 + normas de apresentação tabular (IBGE) — toda tabela exige identificação superior e fonte inferior.

**E10. Expressão inexistente — provável gralha** — linha 122.
> Trecho: "separa o que é estrutural do que é **mercado de escala**".
> Correção: provável "questão de escala" / "economia de escala" / "jogo de escala" — "mercado de escala" não é expressão do português; verificar a formulação pretendida.
> Norma: clareza e correção vocabular.

**E11. Incoerência numérica interna** — linha 88.
> Trecho: "a razão subiu de ~1,1× para ~2,3× **ao quadruplicar os parâmetros**".
> Problema: os braços comparados são ~62–68M → ~471–479M, fator ≈ ×7–7,6, não ×4.
> Correção: "ao multiplicar os parâmetros por ~7,6" — ou reexaminar qual par de braços sustenta o "quadruplicar".
> Norma: coesão/consistência interna do texto científico.

**E12. Separador de milhar inconsistente** — linha 94 ("2048", "[2048,1024,512,256,128]") vs linhas 69, 112 ("2.048", "50.000", "4.096", "3.000", "5.500").
> Correção: padronizar ("2.048" em todo o texto; idealmente espaço fino "2 048" per convenção IBGE/SI, mas a forma com ponto é aceitável se uniforme).
> Norma: uniformidade tipográfica (NBR 14724).

---

## SUGESTÕES (registro, padronização terminológica, estilo — não impedem aceite)

**S1. Terminologia bilíngue instável — padronizar uma forma por par:**
- "paper" (l. 31, 79, 118, 128) ↔ "artigo"/"trabalho" — preferir "artigo"/"trabalho";
- "baseline" (l. 37, 41, 79) ↔ "modelo de referência" (l. 11) — usar um só;
- "steps" (l. 110, 112, 128) ↔ "passos" (l. 69, 94, 128, 136) — "passos" já predomina;
- "passes" (l. 41, 94) ↔ "passagens" (l. 27, 37, 88) — "passagens" já predomina;
- "loss" (l. 94, 100, 120) ↔ "perda" (l. 11) — padronizar;
- "decoder" (l. 45, 73) ↔ "decodificador" — preferir a forma vernácula;
- "codebook pirâmide" (l. 65, 110) ↔ "codebook piramidal" (l. 63, 122) — preferir "piramidal";
- "vs" (l. 67, 140) ↔ "contra" (l. 79, 100);
- "spec-decode" (l. 110) ↔ "decodificação especulativa" (l. 25, 65);
- "bug de init" (l. 100) ↔ "bug de inicialização" (l. 11);
- "drafts destilados" (l. 27) junto a "rascunhos especulativos" no mesmo parágrafo — "rascunhos destilados" resolve;
- notação "k" ↔ algarismos extensos: "50k"/"2k" (l. 128, 136) vs "50.000"/"2.048" (l. 112); "8k" (l. 67) vs "8.192" (l. 63); "43k"/"152k"/"177k" — padronizar (sugestão: "×10³" ou valor extenso em prosa, sigla só em tabela).

**S2. Regência verbal:**
- l. 11: "viável **de iterar** localmente" → "viável iterar localmente" ou "possível de iterar";
- l. 100: "convergiram **a** held_l11" → "convergiram **para**";
- l. 138: "condicionada **nos** códigos" — o texto usa "condicionado à reconstrução" (l. 41) e "condicionados num l0" (l. 112): padronizar para "condicionado a" → "condicionada aos códigos";
- l. 27: "chamou atenção" → "chamou a atenção";
- l. 11/27: "mais rápida **que**" → norma culta prefere "mais rápida **do que**" em comparações;
- l. 41: "robustez a **erro** de previsão" → "robustez a **erros**".

**S3. Colocação pronominal** — l. 88: "deve se preservar" → registro formal prefere "deve preservar-se" ou "deve-se preservar" (a mesma frase usa "manteve-se" corretamente).

**S4. Anglicismos lexicais sem lastro em dicionário:**
- "fatoram erro de reconstrução" (l. 11), "fatora por erro" (l. 96) — "fatorar" não consta; sugerir "são fatorados pelo erro" ou reformular ("decompõem-se por erro", "factorizam" idem);
- "reporta"/"reportados" (l. 27, 31, 41) → "relata"/"relatados";
- "survey" (l. 37) → "revisão sistemática"/"survey" entre aspas;
- "denoising" (l. 37) → "remoção de ruído" (ou manter em itálico/aspas);
- "warm-start", "nested dropout", "taper", "chunks", "skip de nível", "gate zero", "mirror", "tool-use", "spec-decode", "near-uniform" — técnicos defensáveis, mas manter grafados como estrangeirismo (itálico) na primeira ocorrência e minimizar em prosa formal;
- "autoregressiva" (l. 11, 37) — VOLP consagra "autorregressivo"; "autoregressivo" é uso consolidado na área — decisão consciente, registrar.

**S5. Registro informal — avaliar manutenção consciente ou elevação:**
- "salada de palavras" (l. 73, 94);
- "onde ela mora" (l. 45) e "onde o token errado é um programa" (l. 124) — "onde" para referente não espacial: preferir "em que"/"reside";
- "é grátis" (l. 124) → "tem custo nulo";
- "um nó de GPU decente" (l. 136) → "adequado";
- "ficou em held_l0" (l. 102) → "manteve-se em";
- "resolve diferente" (l. 45) → "resolve de modo diferente";
- "orçar as três coisas" (l. 122), "outra coisa" (l. 102), "a métrica honesta aqui" (l. 31);
- fragmento oracional l. 25 ("Quarenta e oito camadas sequenciais, um token de cada vez…") — estilístico, aceitável se deliberado;
- "num" (l. 112) → "em um" em registro formal (contração admitida, mas o texto alterna).

**S6. Numerais por extenso ↔ algarismos** — "onze dias", "vinte rodadas", "quatro rodadas" (extenso) convivem com "8 exemplos", "6 níveis", "12 camadas", "3 métodos" (algarismo). Norma culta recomenda por extenso até dez em prosa corrida; padronizar um critério. Idem frações: "0,07 conceitos" (l. 45, plural) vs "0,9 token" (l. 25, singular) — uniformizar (fração <1 → singular: "0,07 conceito").

**S7. Siglas sem expansão na primeira ocorrência** — "AR" (autorregressivo) usado desde a tabela/§4 sem ter sido definido; "VQ-VAE" e "NCM" aparecem no resumo/abstract sem expansão (NCM é expandido na 1ª linha do resumo — ok; VQ-VAE não); "SONAR" (l. 45), "BPE" (l. 63), "ASTs" (l. 112), "MoE" ausente. NBR 6028 recomenda evitar siglas no resumo ou expandi-las.

**S8. Unidades e símbolos** — "14,5h" (l. 112) → "14,5 h" (espaço antes do símbolo, convenção SI/INMETRO); "%" colado (90%, ±20%) é consistente — manter, mas conhecer a forma com espaço; "~" recorrente (≈30 usos) alterna com "aproximadamente" — padronizar (sugestão: "aprox." em prosa, "~" só em tabelas/fórmulas).

**S9. Formato das entradas arXiv** — "*arXiv*, n. 2401.10774, 2024" trata o identificador como "número"; o usual é "arXiv:2401.10774" ou "arXiv, 2024" + DOI/URL. Recorrente e consistente — registrar como desvio formal, não erro isolado. As notas "(CVPR 2022)", "(ICML 2023)" etc. são informativas e inofensivas, mas desiguais entre entradas — padronizar ou remover.

**S10. Entrada de autor estrangeiro com partícula** — "VAN DEN OORD, A." (l. 186): NBR 6023 tende a registrar sob o elemento substantivo ("OORD, A. van den"); uso atual é prática brasileira corrente e está consistente texto↔lista — decisão do autor, apenas registrar.

**S11. Citação com comentário embutido** — l. 37: "(… — ver HU et al., 2025 para um survey completo)". Funcional, porém fora do padrão seco autor–data; alternativa: nota de rodapé ou "cf." + separação das obras.

**S12. Assindeto no resumo** — l. 11: "durante onze dias de experimentação, vinte rodadas registradas de hipótese…" — inserir "e" (ou "em"): "…de experimentação e vinte rodadas registradas…".

**S13. Nota de conteúdo (fora do escopo gramatical, apenas registro):** a sequência de runs tem lacunas aparentes (Fase A 001–004; Fase B 005–012; Fase C 014–016; bug "runs 014–017"; limpa "019/020"; hardware "runs 006 em diante") — run 013 e 018 não aparecem; provável decisão deliberada, mas um leitor externo pode estranhar. Confirmar intencionalidade.

---

## Parecer

Texto bem construído, coeso, com domínio do registro ensaístico-formal e pontuação cuidadosa (travessões, incisos, enumerações internas). Os problemas concentram-se em três frentes: (a) **conformidade de referências** — ~10 entradas órfãs e ausência sistemática de "Disponível em/Acesso em" (E6–E8), o achado mais pesado; (b) **micro-ortografia** — hífens indevidos (E1–E2) e preposição "de" omitida (E3–E4); (c) **consistência** — separador de milhar, terminologia EN/PT, incoerência "quadruplicar" (E11) e a gralha "mercado de escala" (E10). Nenhum exige reescrita estrutural; todos são corrigíveis em uma passada de revisão.

## VEREDICTO: **CORREÇÕES NECESSÁRIAS**

(Próximo de impecável no plano linguístico; o que pesa é a conformidade ABNT das referências e do elemento tabela.)
