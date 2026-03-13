# Napkin Runbook

## Curation Rules
- Re-prioritize on every read.
- Keep recurring, high-value notes only.
- Max 10 items per category.
- Each item includes date + "Do instead".

## Execution & Validation (Highest Priority)
1. **[2026-03-12] Cold path do runtime nao pode ser avaliado com cache exato ja aquecido**
   Do instead: medir sempre um request frio em processo novo ou apos restart do worker antes de concluir que a pergunta esta rapida.
2. **[2026-03-12] Pipeline de geracao SQL deve ter um unico autor antes da execucao**
   Do instead: enviar toda pergunta analitica ao `vanna.generate_sql` com o modelo `best` e evitar fast paths, cache exato ou fan-out antes da chamada ao LLM.
3. **[2026-03-12] Falha de valor canonico nao se corrige com mais heuristica de SQL**
   Do instead: apos gerar a SQL, fazer grounding de literais usando `project_column_domains.domain_json` para alinhar filtros textuais ao dominio publicado da coluna.
4. **[2026-03-11] Runtime do Lab tem gargalos fora do SQL**
   Do instead: medir separadamente geracao SQL, execucao SQL, enriquecimento geo e serializacao antes de tratar timeout como problema de banco.
5. **[2026-03-12] Reescrita semantica deve ser conservadora**
   Do instead: substituir literais somente quando o dominio for publicado/confiavel e o match for unico com alta confianca; se houver ambiguidade, nao reescrever.
6. **[2026-03-12] Debug de SQL vazia deve separar estrutura de grounding**
   Do instead: se a SQL parece correta mas retorna zero linhas, validar primeiro se os literais filtrados existem no dominio da coluna antes de mexer no prompt ou no schema.
8. **[2026-03-11] Pergunta com sinal geografico nao deve implicar payload geo no fluxo principal**
   Do instead: usar sinalizacao leve de modalidade/mapa e gerar payload geografico apenas em fluxo sob demanda.
7. **[2026-03-12] SQL exibida no Lab deve refletir apenas compilacao estrutural**
   Do instead: mostrar `sql_original` + `sql_compilada` apenas quando houver adaptacao estrutural real, como remocao de geometria; grounding de dominio nao deve duplicar o bloco de SQL.

## Shell & Command Reliability
1. **[2026-03-11] API local pode travar no bootstrap de schema**
   Do instead: se o banco ja estiver provisionado para teste, subir `api-geo-nlp` com `API_GEO_NLP_DB_AUTO_INIT_SCHEMA=false` antes de investigar rotas ou frontend.
2. **[2026-03-11] Buscas no monorepo devem evitar `node_modules` e `.venv`**
   Do instead: usar `rg`/`find` com glob de exclusao para reduzir ruido e custo de leitura.
3. **[2026-03-12] Tool Memory nova nao pode depender do bootstrap global de schema**
   Do instead: deixar o adapter do Vanna autocriar `project_vanna_tool_memories` e `project_vanna_text_memories` com DDL idempotente quando o backend for usado e o bootstrap geral estiver desligado.

## Domain Behavior Guardrails
1. **[2026-03-11] `ui_command` so pode vencer com evidencia lexical explicita**
   Do instead: tratar perguntas de dados em modo imperativo (`me de`, `liste`, `traga`, `mostre`) como `analytic_sql` por padrao e usar o LLM apenas como apoio, nunca como decisor unico para handoff de UI.
2. **[2026-03-12] Aba SQL do Lab nao pode ocultar a compilacao de geometria**
   Do instead: quando `geometry_mode=exclude` adaptar a query, expor lado a lado a SQL original e a SQL executada para o usuario entender por que `geom` sumiu do resultado.
3. **[2026-03-11] Tabela do Lab precisa permitir navegacao sem truncar o dataset**
   Do instead: renderizar resultados longos em viewport com rolagem vertical/horizontal e preservar acao secundaria de tela cheia apenas como complemento.
4. **[2026-03-11] Tabela do Lab nao deve expor colunas geoespaciais**
   Do instead: filtrar `geom`/`geometry`/`geojson` na API e manter filtro defensivo tambem na UI.
5. **[2026-03-11] Corpus curado e memoria de runtime nao devem compartilhar o mesmo CRUD**
   Do instead: manter `Questions` como editor do corpus treinavel e expor memoria vetorial/feedback em subarea read-only com acoes seguras como promocao ou invalidacao separadas.
6. **[2026-03-11] Modalidades finais ja existem no contrato do runtime**
   Do instead: reutilizar `modalities` para UX de indicacao de texto/tabela/grafico/mapa antes de criar contrato paralelo.
7. **[2026-03-11] Controle de retorno de geometria deve ser contratual**
   Do instead: introduzir flag/enum no `AskRequest` e propagar para todos os transportes (`/ask`, SSE, WebSocket, polling) em vez de codificar excecoes por tela.
8. **[2026-03-11] Narrativa do produto deve priorizar texto-para-mapa**
   Do instead: explicar NLP2SQL como mecanismo interno e apresentar o valor principal como transformacao de perguntas em mapas e leitura territorial.

## User Directives
1. **[2026-03-11] Foco atual e performance do Lab**
   Do instead: priorizar remocao do payload geo e da exibicao de `geom` no resultado antes de qualquer nova funcionalidade cartografica.
2. **[2026-03-11] Lab deve operar sem retorno de `geom`, mas a API precisa suportar os dois modos**
   Do instead: tratar o Lab como consumidor com `include_geometry_columns=false` e preservar modo `true` para integracoes que precisem gerar GeoJSON/WMS.
3. **[2026-03-12] Etapas pre-LLM de geracao SQL foram descartadas por decisao de produto**
   Do instead: nao reintroduzir fast path deterministico, cache de SQL anterior ou fallback gerador paralelo antes do `best llm` sem aprovacao explicita do usuario.
