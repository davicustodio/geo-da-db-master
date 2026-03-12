# Napkin Runbook

## Curation Rules
- Re-prioritize on every read.
- Keep recurring, high-value notes only.
- Max 10 items per category.
- Each item includes date + "Do instead".

## Execution & Validation (Highest Priority)
1. **[2026-03-11] Runtime do Lab tem gargalos fora do SQL**
   Do instead: medir separadamente geracao SQL, execucao SQL, enriquecimento geo e serializacao antes de tratar timeout como problema de banco.
2. **[2026-03-11] Lookup simples deve tentar contexto semantico antes de hidratar Vanna**
   Do instead: para perguntas de baixa complexidade, usar primeiro chunks `ddl`/`dictionary` do projeto ativo e so carregar memoria/exemplos do Vanna se a SQL direta nao for aderente.
3. **[2026-03-11] Pergunta semelhante deve adaptar memoria publicada antes do fallback profundo**
   Do instead: quando houver exemplo Q→SQL suficientemente similar na memoria vetorial do projeto, montar um prompt curto com esses exemplos + grounding minimo e tentar um `memory_adaptation_fast_path` antes de hidratar o Vanna completo.
4. **[2026-03-11] Pergunta repetida deve reutilizar candidatos por versao ativa**
   Do instead: cachear candidatos SQL por `project_id + active_version + pergunta normalizada` para zerar LLM no warm path sem misturar respostas entre versoes semanticas.
5. **[2026-03-11] Pergunta com sinal geografico nao deve implicar payload geo no fluxo principal**
   Do instead: usar sinalizacao leve de modalidade/mapa e gerar payload geografico apenas em fluxo sob demanda.
6. **[2026-03-11] Gargalo de LLM cai mais com bypass do que com tuning de prompt**
   Do instead: para rankings, totais e distribuicoes geo-analiticas simples, tentar fast path deterministico antes de chamar o LLM.
7. **[2026-03-11] Fast path perde valor se hidratar Vanna antes**
   Do instead: executar heuristicas deterministicas antes de carregar memoria vetorial, dataset ativo ou instancia Vanna.
8. **[2026-03-12] Defaults de configuracao devem viver nos arquivos de ambiente**
   Do instead: remover fallback de `config.py`, wrappers legados e cliente frontend quando a variavel tem valor padrao conhecido, e manter `.env.example`/`.env.template` completos por projeto.

## Shell & Command Reliability
1. **[2026-03-11] API local pode travar no bootstrap de schema**
   Do instead: se o banco ja estiver provisionado para teste, subir `api-geo-nlp` com `API_GEO_NLP_DB_AUTO_INIT_SCHEMA=false` antes de investigar rotas ou frontend.
2. **[2026-03-11] Buscas no monorepo devem evitar `node_modules` e `.venv`**
   Do instead: usar `rg`/`find` com glob de exclusao para reduzir ruido e custo de leitura.

## Domain Behavior Guardrails
1. **[2026-03-11] `ui_command` so pode vencer com evidencia lexical explicita**
   Do instead: tratar perguntas de dados em modo imperativo (`me de`, `liste`, `traga`, `mostre`) como `analytic_sql` por padrao e usar o LLM apenas como apoio, nunca como decisor unico para handoff de UI.
2. **[2026-03-11] Tabela do Lab precisa permitir navegacao sem truncar o dataset**
   Do instead: renderizar resultados longos em viewport com rolagem vertical/horizontal e preservar acao secundaria de tela cheia apenas como complemento.
3. **[2026-03-11] Tabela do Lab nao deve expor colunas geoespaciais**
   Do instead: filtrar `geom`/`geometry`/`geojson` na API e manter filtro defensivo tambem na UI.
4. **[2026-03-11] Corpus curado e memoria de runtime nao devem compartilhar o mesmo CRUD**
   Do instead: manter `Questions` como editor do corpus treinavel e expor memoria vetorial/feedback em subarea read-only com acoes seguras como promocao ou invalidacao separadas.
5. **[2026-03-11] Modalidades finais ja existem no contrato do runtime**
   Do instead: reutilizar `modalities` para UX de indicacao de texto/tabela/grafico/mapa antes de criar contrato paralelo.
6. **[2026-03-11] Controle de retorno de geometria deve ser contratual**
   Do instead: introduzir flag/enum no `AskRequest` e propagar para todos os transportes (`/ask`, SSE, WebSocket, polling) em vez de codificar excecoes por tela.
7. **[2026-03-11] Narrativa do produto deve priorizar texto-para-mapa**
   Do instead: explicar NLP2SQL como mecanismo interno e apresentar o valor principal como transformacao de perguntas em mapas e leitura territorial.

## User Directives
1. **[2026-03-11] Foco atual e performance do Lab**
   Do instead: priorizar remocao do payload geo e da exibicao de `geom` no resultado antes de qualquer nova funcionalidade cartografica.
2. **[2026-03-11] Lab deve operar sem retorno de `geom`, mas a API precisa suportar os dois modos**
   Do instead: tratar o Lab como consumidor com `include_geometry_columns=false` e preservar modo `true` para integracoes que precisem gerar GeoJSON/WMS.
