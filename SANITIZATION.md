# Log de sanitização pré-publicação

Data: 09 nov 2026. Escopo: todo o conteúdo commitado fora de `.git/`.

## Varreduras executadas

| Padrão buscado | Resultado |
|---|---|
| Caminhos pessoais (`/home/<user>`, `/root/`, `/Users/`) | 0 ocorrências |
| Nome de usuário local / hostname | 0 ocorrências |
| Tokens, chaves, senhas (`api_key`, `bearer`, `ghp_`, `sk-`, `password`) | 0 ocorrências |
| Arquivos > 10 MB | nenhum |
| Binários / checkpoints / `.models/` | nenhum (repositório é só texto: 288 KB) |
| `git diff --check` | limpo |

## Identificadores intencionalmente mantidos

- **Autor**: "Renato de Souza e Silva" — autorizado explicitamente para a
  assinatura do trabalho.
- **URL do repositório** (`github.com/<conta>/ncm-local-study`) no BibTeX do
  README — é a identidade pública do próprio artefato publicado.
- **Amostras de texto geradas** (`artifacts/samples/`, `bench_ts_resume.json`)
  contêm inglês degenerado ("word salad") — são dados de resultado, não dados
  pessoais.

## Política de histórico

Commits são datados conforme a timeline real da pesquisa (30 out – 09 nov
2026). Mensagens antigas podem conter valores intermediários depois corrigidos
(ex.: `l11=2.68`, valor de treino, corrigido para o held-out `2.79` na revisão);
a correção está documentada em `TIMELINE.md`, `paper/SOURCES.md` e no paper —
o histórico preserva o processo, não o esconde.
