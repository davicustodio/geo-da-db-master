# Napkin Runbook

## Curation Rules
- Re-prioritize on every read.
- Keep recurring, high-value notes only.
- Max 10 items per category.
- Each item includes date + "Do instead".

## Execution & Validation (Highest Priority)
1. **[2026-03-11] Runtime do Lab tem gargalos fora do SQL**
   Do instead: medir separadamente geracao SQL, execucao SQL, enriquecimento geo e serializacao antes de tratar timeout como problema de banco.
2. **[2026-03-11] Pergunta com sinal geografico nao deve implicar payload geo no fluxo principal**
   Do instead: usar sinalizacao leve de modalidade/mapa e gerar payload geografico apenas em fluxo sob demanda.
3. **[2026-03-11] Gargalo de LLM cai mais com bypass do que com tuning de prompt**
   Do instead: para rankings, totais e distribuicoes geo-analiticas simples, tentar fast path deterministico antes de chamar o LLM.
4. **[2026-03-11] Fast path perde valor se hidratar Vanna antes**
   Do instead: executar heuristicas deterministicas antes de carregar memoria vetorial, dataset ativo ou instancia Vanna.

## Shell & Command Reliability
1. **[2026-03-11] Buscas no monorepo devem evitar `node_modules` e `.venv`**
   Do instead: usar `rg`/`find` com glob de exclusao para reduzir ruido e custo de leitura.

## Domain Behavior Guardrails
1. **[2026-03-11] Tabela do Lab nao deve expor colunas geoespaciais**
   Do instead: filtrar `geom`/`geometry`/`geojson` na API e manter filtro defensivo tambem na UI.
2. **[2026-03-11] Modalidades finais ja existem no contrato do runtime**
   Do instead: reutilizar `modalities` para UX de indicacao de texto/tabela/grafico/mapa antes de criar contrato paralelo.
3. **[2026-03-11] Controle de retorno de geometria deve ser contratual**
   Do instead: introduzir flag/enum no `AskRequest` e propagar para todos os transportes (`/ask`, SSE, WebSocket, polling) em vez de codificar excecoes por tela.
4. **[2026-03-11] Narrativa do produto deve priorizar texto-para-mapa**
   Do instead: explicar NLP2SQL como mecanismo interno e apresentar o valor principal como transformacao de perguntas em mapas e leitura territorial.

## User Directives
1. **[2026-03-11] Foco atual e performance do Lab**
   Do instead: priorizar remocao do payload geo e da exibicao de `geom` no resultado antes de qualquer nova funcionalidade cartografica.
2. **[2026-03-11] Lab deve operar sem retorno de `geom`, mas a API precisa suportar os dois modos**
   Do instead: tratar o Lab como consumidor com `include_geometry_columns=false` e preservar modo `true` para integracoes que precisem gerar GeoJSON/WMS.
