# Findings

## Plano de apresentacao NLP2SQL (2026-03-11)
- Para publico leigo, a melhor narrativa nao e "modelo gera SQL", e sim "o sistema prepara conhecimento do banco, revisa, publica memoria e so depois responde".
- O projeto atual comunica bem uma historia em duas etapas:
  - base semantica;
  - embeddings do Vanna.
- Essa separacao e importante para a apresentacao porque traduz um ponto real da arquitetura:
  - gerar conhecimento do banco;
  - revisar;
  - publicar memoria;
  - liberar o runtime/Lab.
- O runtime da API ja fornece um encadeamento didatico para explicar o fluxo de producao:
  - receber pergunta;
  - classificar intencao;
  - gerar/adaptar SQL;
  - validar seguranca;
  - estimar custo;
  - executar;
  - planejar resposta multimodal.
- Para 15 minutos, a recomendacao mais equilibrada e usar 10 slides, com um slide extra opcional apenas se houver tempo para explicar o aprendizado controlado com Vanna.

## Backend implementado
- `TrainingStateResponse` agora expõe `semantic_base` e `embeddings`, além de `sections.policies`.
- O `control_plane` passou a calcular:
  - versão ativa para runtime/embeddings;
  - versão de trabalho mais recente para curadoria;
  - status de base semântica (`not_started`, `draft_ready`, `in_review`, `reviewed`, `failed`);
  - status de embeddings (`blocked`, `ready_to_generate`, `stale`, `published`, `failed`).
- Foi criada a ação `POST /projects/{project_id}/training/semantic-review:complete`.
- Edições em schema, DDL, Dictionary e Questions agora marcam a base como alterada e deixam embeddings `stale`.

## Orquestração implementada
- `reset-and-regenerate` agora gera apenas a base semântica.
- `retrain-existing` agora gera/publica embeddings sobre a versão de trabalho revisada.
- A publicação de embeddings grava a origem semântica (`embedding_source_semantic_version`) e limpa a flag de `stale`.

## Frontend implementado
- `ProjectSetupPage` foi redesenhada para 4 passos explícitos:
  - validar acesso;
  - gerar base semântica;
  - concluir revisão;
  - gerar embeddings.
- `ProjectWorkspaceNav`, `ProjectsPage`, `GlobalPoliciesPage`, `MetadataPage` e `QuestionsPage` passaram a respeitar o novo estado.
- As telas de curadoria invalidam o estado de treinamento após edições para refletir `stale` imediatamente.
- `Lab` continua bloqueado até embeddings publicados.

## Verificações executadas
- Frontend:
  - `npm test -- --run src/__tests__/project-setup-page.test.tsx src/__tests__/project-workspace-nav.test.tsx src/__tests__/projects-page.test.tsx src/__tests__/metadata-page.test.tsx src/__tests__/questions-page.test.tsx src/__tests__/lab-page.test.tsx`
  - `npm run build`
- Backend:
  - `pytest tests/unit/test_training_state.py tests/integration/test_training_routes.py tests/integration/test_metadata_questions_routes.py -q`

## Limitação encontrada
- `npm run lint` não pôde ser executado porque `eslint` não está instalado/disponível no ambiente.

## Diagnóstico operacional da falha em `Gerar embeddings`
- Reprodução feita na UI local `http://localhost:5173` com o usuário informado `davi.custodio@embrapa.br`.
- Projeto reproduzido: `datahub2`.
- O clique em `Gerar embeddings` chamou `POST /projects/datahub2/training/retrain-existing` com sucesso e iniciou o job `d9d1058b-3ce0-4f6a-a343-f7ec4898f270`.
- O novo job falhou novamente em `T3_T4_build` com a mesma mensagem do job anterior `25e23366-bc72-4db4-b48e-df363602b1b6`: `Gate de qualidade falhou: Validação de runtime no banco reprovada: 1 SQLs falharam (limite: 0).`
- A causa foi isolada executando o mesmo validador de runtime do backend (`TrainingOrchestrator._build_runtime_sql_validator`) sobre as `questions` da versão semântica `20260311T120841Z`.
- Há exatamente 1 SQL que falha no `EXPLAIN`, correspondente à pergunta `id=388` / índice 95:
  - pergunta: `Qual a participação da região Norte no valor total da produção de açaí?`
  - erro: `subquery uses ungrouped column "p.nome_produto" from outer query`
- A SQL problemática usa uma subquery correlacionada incorreta no denominador:
  - `... (SELECT SUM(valor_total) FROM public.producao WHERE p.nome_produto IN (...)) ...`
  - o alias externo `p` é referenciado dentro da subquery sem agrupamento, causando `GroupingError`.
- A mesma pergunta já estava marcada no armazenamento com `sql_is_valid = false` e `sql_validation_error` preenchido em `project_training_questions`, antes mesmo da fase de embeddings.

## Gap de produto/observabilidade encontrado
- O `quality_report` só é anexado ao `build_meta` depois que o gate passa. Quando o gate falha, o detalhe de `runtime_sql_error_samples` é perdido antes de ser persistido.
- O manifesto salvo em `project_dataset_versions.metadata_json` também não recebe o relatório de falha, porque `write_manifest` é chamado apenas depois de `report.raise_if_failed()`.
- Resultado prático: UI e eventos do job mostram apenas o agregado `1 SQLs falharam`, sem indicar qual pergunta/SQL precisa ser corrigida.

## Solução implementada
- Mitigação operacional aplicada no projeto `datahub2`:
  - a `question 388` foi corrigida via API;
  - a SQL ajustada passa no mesmo validador de runtime usado pelo build;
  - a revisão semântica foi reconcluída e a geração de embeddings foi reexecutada com sucesso.
- Backend:
  - `DatasetQualityGate` agora captura:
    - `stored_invalid_question_count`;
    - `stored_invalid_questions`;
    - `runtime_sql_failure_details`.
  - O build agora persiste `quality_report` no manifesto mesmo quando falha.
  - Falhas de build carregam contexto estruturado (`quality_failure`) para `project_training_jobs` e para os eventos do job.
  - Questions já previamente marcadas com `sql_is_valid = false` passam a bloquear o publish com mensagem explícita antes do erro genérico de runtime.
- Frontend:
  - `QuestionsPage` agora destaca um resumo das questions inválidas na curadoria.
  - `ProjectSetupPage` mostra um diagnóstico resumido de quality gate no monitor quando o job falha com contexto estruturado.
  - `TrainingJobLogModal` passou a renderizar violações, questions inválidas e falhas de runtime de forma legível, sem depender de inspecionar JSON bruto.

## Validação final
- Job final bem-sucedido: `38d0f283-d1a0-4058-a6f4-3b6f10f499de`.
- Estado final do projeto `datahub2`:
  - `semantic_base.status = reviewed`
  - `embeddings.status = published`
  - `ready = true`
  - `lab = true`
- O quality gate aprovado reportou:
  - `question_count = 101`
  - `runtime_sql_checked_count = 101`
  - `runtime_sql_invalid_count = 0`

## Diagnóstico do Lab: pergunta `quais as 10 cidades que mais produzem soja`
- Reprodução direta no runtime local (`POST /projects/datahub2/ask`) com o usuário `davi.custodio@embrapa.br` devolveu exatamente a SQL incorreta observada no Lab:
  - `SELECT m.nm_municip, SUM(p.valor_total) as valor ... WHERE p.nome_produto = 'soja' AND m.nm_estado = 'MATO GROSSO' ... LIMIT 5;`
- A resposta saiu com `execution_policy = sync`, `execution_time_ms = 79.03ms` e latência total de API de `34.291ms`.
- O `decision_trace` mostrou que o candidato vencedor foi `cand-2`, fonte `pgvector_memory`, com `final_score = 0.9824`.

## Causa raiz principal
- O chunk vetorial mais parecido para essa pergunta é um `feedback` automático salvo como:
  - `Pergunta validada: quais as 10 cidades que mais produzem soja`
  - SQL: `... AND m.nm_estado = 'MATO GROSSO' ... LIMIT 5`
- Esse chunk está em `project_rag_chunks` como `chunk_type='feedback'`, `metadata.feedback_by='runtime_auto'`.
- O runtime grava automaticamente toda interação em memória via `RuntimeOrchestrator._save_to_memory()`:
  - treina a memória do Vanna com `vn.train(question=question, sql=sql)`;
  - insere chunk vetorial com `PgVectorStore.add_feedback_chunk(..., feedback_by="runtime_auto")`.
- Esse aprendizado automático ocorre sem confirmação humana de que a resposta está correta.
- Não há registro correspondente em `project_runtime_feedback`; ou seja, o envenenamento da memória acontece fora do fluxo auditável de feedback explícito.

## Causa raiz secundária: validação de aderência insuficiente
- O filtro `_is_sql_relevant_to_question()` considera a SQL de `MATO GROSSO` + `LIMIT 5` como plenamente aderente (`relevance_score = 1.0`) para a pergunta “10 cidades ... soja”.
- A checagem atual valida:
  - tabelas permitidas;
  - sobreposição de tokens;
  - presença de dimensões geográficas.
- A checagem atual não valida:
  - cardinalidade pedida (`10`) versus `LIMIT 5`;
  - filtros extras não solicitados (`m.nm_estado = 'MATO GROSSO'`);
  - diferença entre pergunta nacional e pergunta restrita a um estado;
  - diferença entre “produzem mais” e “mais produtivos por hectare”.

## Causa raiz terciária: viés do ranking
- `generate_sql_candidates()` trouxe um candidato melhor pelo `context_fallback`:
  - sem filtro por estado;
  - com `LIMIT 10`.
- Mesmo assim o ranking escolheu o candidato errado do `pgvector_memory`, porque:
  - os candidatos recuperados da memória entram primeiro;
  - `rank_sql_candidates()` estima custo apenas nos 3 primeiros candidatos (`runtime_rank_top_k_for_cost = 3`);
  - os candidatos de memória recebem `source_bonus`;
  - como a aderência foi marcada como `1.0`, o score final favoreceu o exemplo errado mais barato.

## Causa raiz no corpus/pipeline de preparação
- O corpus ativo (`version_tag = 20260311T120841Z`) contém várias perguntas específicas por estado para soja, por exemplo:
  - “Quais são os 5 municípios com maior valor de produção de soja no Mato Grosso?”
  - “Quais são os 5 municípios com maior valor de produção de soja no Rio Grande do Sul?”
- Não foi encontrada uma pergunta canônica equivalente e genérica para o caso nacional “10 municípios/cidades que mais produzem soja”.
- O gate de qualidade em `t4_quality.py` valida sintaxe, execução (`EXPLAIN`), cobertura e duplicidade, mas não mede aderência semântica pergunta→SQL.
- Resultado: um corpus “válido” para o gate ainda pode induzir SQL semanticamente errada no runtime.

## Diagnóstico de performance
- Medição por etapa com o mesmo caso:
  - classificação de intenção: `4344.67ms`
  - geração de candidatos SQL: `7360.08ms`
  - ranking + cost estimation: `417.98ms`
  - execução SQL: `101.23ms`
  - sugestão de gráfico: `5512.66ms`
  - total local medido: `17796.57ms`
- A execução SQL em si está rápida. O gargalo está no runtime LLM:
  - `_classify_intent()` usa LLM mesmo para uma pergunta analítica óbvia;
  - `generate_sql_candidates()` tenta memória vetorial, memória Vanna, geração por múltiplos modelos e fallback por prompt;
  - `_suggest_chart_with_vanna()` faz outra chamada LLM mesmo para um ranking tabular simples.

## Impacto operacional
- O Lab pode “aprender” respostas erradas a partir de uma primeira resposta incorreta.
- O erro semântico não é bloqueado pelo pipeline atual de preparação/publicação.
- O usuário percebe latência alta e SQL errada no mesmo fluxo, o que compromete confiança no produto.

## Comparação com a documentação do Vanna AI 2.0
- A documentação oficial do Vanna 2.0 descreve o fluxo de melhoria contínua assim:
  - apenas **interações bem-sucedidas** devem ser salvas em `Tool Memory`;
  - para pergunta similar, o agente deve **buscar padrões anteriores e adaptar a query ao contexto atual**;
  - o fluxo esperado é `Search Tool Memory -> Adapt Existing Query -> Execute Variant -> Return Result`;
  - a arquitetura recomendada em 2.0 é `Agent + ToolRegistry + AgentMemory` (ou `LegacyVannaAdapter` para migrar um objeto legado).
- O código atual do projeto não usa esse fluxo oficial do Agent Framework:
  - não há uso de `Agent`, `ToolRegistry`, `DefaultLlmContextEnhancer`, `SaveQuestionToolArgsTool` ou `SearchSavedCorrectToolUsesTool`;
  - a integração está em cima de `vanna.openai.OpenAI_Chat` / `vanna.legacy.openai.OpenAI_Chat` com RAG e ranking customizados.
- Ou seja: hoje o projeto está rodando uma arquitetura híbrida, não a arquitetura canônica documentada do Vanna 2.0 para essa etapa.

## Divergência crítica: exemplos recuperados não entram no prompt como o Vanna espera
- No pacote oficial `vanna==2.0.2`, `VannaBase.generate_sql()` monta o prompt chamando:
  - `get_similar_question_sql()`
  - `get_related_ddl()`
  - `get_related_documentation()`
  - `get_sql_prompt()`
  - `submit_prompt()`
- O método oficial `get_sql_prompt()` espera que `question_sql_list` seja uma lista de objetos com chaves `question` e `sql`.
- Na implementação local, `GeoNLPVannaMemory.get_similar_question_sql()` retorna `list[tuple[str, str]]`.
- Resultado observado em execução real:
  - `examples_type = tuple`
  - `message_count = 2`
  - o prompt final contém apenas:
    - 1 mensagem `system` com contexto DDL/documentação
    - 1 mensagem `user` com a pergunta
  - nenhum exemplo Q→SQL recuperado entra como `user/assistant` no prompt do `vn.generate_sql`.
- Portanto, na etapa `vn.generate_sql`, o Vanna não está recebendo os exemplos similares do jeito que a API oficial foi desenhada para receber.

## Divergência crítica: memória similar está sendo usada como SQL pronta, não como contexto para adaptação
- Antes de chamar `vn.generate_sql`, `generate_sql_candidates()` busca no `pgvector` e adiciona as SQLs encontradas diretamente como candidatas finais (`source='pgvector_memory'`).
- Também transforma o retorno de `vn.get_similar_question_sql()` diretamente em candidatas finais (`source='tool_memory'`).
- Isso desvia do comportamento descrito na documentação do Vanna 2.0, onde a memória é usada para orientar/adaptar a geração, não para “ganhar” a disputa antes da montagem do prompt.
- Foi exatamente isso que ocorreu no caso investigado:
  - a SQL errada de `MATO GROSSO` + `LIMIT 5` entrou como candidata pronta;
  - recebeu score alto;
  - venceu antes da etapa em que o LLM poderia adaptar exemplos ao novo contexto.

## Observação importante sobre a expectativa de adaptação do LLM
- A hipótese do usuário faz sentido se:
  - os exemplos recuperados forem enviados como contexto/exemplo ao LLM;
  - a memória contiver apenas interações realmente corretas;
  - a etapa de ranking não promover a cópia direta de uma SQL similar incorreta.
- No estado atual, essas três premissas não se sustentam:
  - o exemplo similar nem entra no prompt do `vn.generate_sql` por incompatibilidade de formato;
  - a memória recebe entradas automáticas não validadas;
  - a SQL recuperada pode ser escolhida como resposta final sem adaptação contextual do LLM.

## Conclusão revisada
## Solução implementada para o gargalo de LLM
- Em `2026-03-11`, foi implementado um fast path determinístico em `generate_sql_candidates()` para perguntas geo-analíticas simples.
- A nova estratégia tenta resolver antes do LLM, nesta ordem:
  - match exato de pergunta→SQL já validada;
  - `geo_rule_fast_path` para rankings, totais e distribuições por dimensão geográfica.
- O `geo_rule_fast_path` usa `_build_geo_rule_based_sql()` + `_should_use_geo_rule_fast_path()` e só desvia do LLM quando a SQL gerada:
  - continua dentro do schema permitido;
  - é aderente à pergunta;
  - não é caso percentual/participação.
- A heurística foi ajustada para cobrir frases comuns do Lab, como:
  - `quais as 10 cidades que mais produzem soja`;
  - `qual a distribuição do valor total da produção por estado`.

## Resultado medido após o fast path
- Benchmark real no `datahub2` com `scripts/benchmark_ask_runtime.py --mode direct`, 2 amostras válidas por cenário e 1 warmup:
  - `quais as 10 cidades que mais produzem soja`
    - antes: `~10.29s` end-to-end, `~7.03s` em LLM, `R2.generate_candidates ~9.05s`;
    - depois: `~1.66s` end-to-end médio, `0ms` de LLM, `backend ~1.61s`.
  - `qual a distribuição do valor total da produção por estado`
    - antes: `~21.78s` no pior caso contado, `~16.20s` em LLM;
    - depois: `~1.49s` end-to-end médio, `0ms` de LLM, `backend ~1.45s`.
- No lote pós-correção:
  - `avg_end_to_end_ms = 1579.41`
  - `avg_backend_ms = 1530.39`
  - `avg_llm_ms = 0.0`
  - `R2.generate_candidates = 1088.68ms`
- O span `R2.geo_rule_fast_path` passou a aparecer explicitamente nas medições, confirmando que o bypass entrou.

## Otimização adicional aplicada
- Em `2026-03-11`, a geração de candidatos foi reordenada para executar o `geo_rule_fast_path` antes de hidratar Vanna, dataset ativo e `PgVectorStore`.
- Antes dessa mudança, o bypass eliminava o LLM, mas ainda carregava parte da infraestrutura de memória antes de retornar.
- Depois do lazy init:
  - benchmark `quais as 10 cidades que mais produzem soja`, com 2 amostras válidas + 1 warmup;
  - `avg_end_to_end_ms = 575.93`
  - `avg_backend_ms = 526.72`
  - `avg_llm_ms = 0.0`
  - `R2.generate_candidates = 6.1ms`
- Ou seja: o gargalo deixou de estar em `generate_candidates` para esse cenário; o tempo dominante passou a ser custo/execução SQL e ranking.

## Decisão técnica

## Nova otimização alinhada à documentação do Vanna
- Em `2026-03-11`, foi adicionado um fast path intermediário para o fluxo `similar question -> fast adaptation from memory`, antes da hidratação completa do Vanna legado.
- A implementação usa apenas artefatos já publicados do projeto ativo:
  - memória vetorial (`qa` + `feedback` validado) para recuperar exemplos Q→SQL semelhantes;
  - chunks semânticos (`ddl` + `dictionary`) para grounding mínimo do prompt.
- O novo caminho só é acionado quando:
  - não há match exato;
  - existem exemplos selecionáveis pelo filtro de aderência do prompt;
  - a similaridade da melhor pergunta publicada cruza o limiar de segurança.
- Quando esse fast path gera SQL aderente, o runtime retorna sem:
  - hidratar a instância completa do Vanna;
  - consultar `vn.get_similar_question_sql()`;
  - entrar no fallback profundo.

## Evidência prática do novo caminho
- Validação unitária:
  - `25 passed` em `tests/unit/test_vanna_agent_relevance.py`;
  - `6 passed` em `tests/unit/test_runtime_intent_classification.py`.
- Validação HTTP real na pergunta:
  - `me mostre os produtos da categoria pecuaria que cresceram no valor de producao entre 2018 e 2019`
- Resultado observado:
  - `selected_source = memory_adaptation_fast_path`
  - `llm_calls = ['generate_sql_memory_fast_path']`
  - `total_backend_ms = 6769.67` no primeiro hit frio medido.
- Após a pergunta entrar no cache de candidatos por versão semântica:
  - nova execução da mesma pergunta caiu para `~508ms` de backend;
  - `R2.candidate_cache_hit` apareceu;
  - `llm_calls = []`.

## Leitura técnica consolidada
- O runtime agora está mais próximo da recomendação do Vanna:
  - `match exato` -> reutilização direta;
  - `pergunta semelhante` -> adaptação rápida guiada por memória;
  - `pergunta nova` -> investigação mais profunda com contexto semântico / Vanna legado.
- A otimização continua genérica:
  - não depende de `datahub2`;
  - não codifica nomes fixos de tabela/coluna;
  - usa apenas o schema semântico ativo e a memória publicada do projeto corrente.
- A solução efetiva para o gargalo principal não é apenas trocar modelo ou reduzir prompt.
- O maior ganho veio de reduzir o domínio de perguntas que realmente precisam de inferência LLM.
- Para o `datahub2`, a direção correta é expandir os fast paths determinísticos para:
  - rankings nacionais/regionais por produto;
  - distribuições por estado/região/bioma;
  - totais agregados com filtros temporais simples.
- Isso não garante a estratégia mais rápida possível para toda pergunta e todo projeto.
- O que a mudança garante é a política correta de execução:
  - tentar o caminho determinístico barato e seguro primeiro;
  - só chamar LLM quando a pergunta realmente exigir adaptação semântica.
- O problema não é apenas “chunk contaminado”.
- O problema é que a implementação atual:
  - não segue o pipeline recomendado do Vanna 2.0 nessa etapa;
  - usa memória similar como candidata final direta;
  - não garante que o que entra em memória seja realmente “successful interaction”;
  - e ainda passa exemplos para o Vanna em formato incompatível com a montagem oficial do prompt.

## Correções implementadas
- `get_similar_question_sql()` passou a devolver exemplos no contrato esperado pelo Vanna (`dict` com `question` e `sql`) e a filtrar exemplos incompatíveis com a pergunta atual.
- O prompt de SQL do runtime deixou de usar a instrução legada de “repetir exatamente” e passou a enfatizar que a pergunta atual é a fonte da verdade.
- O runtime não promove mais memória vetorial ou Tool Memory como resposta final direta; a memória agora serve para contextualizar a geração.
- O auto-save em memória foi desabilitado para respostas não validadas e o salvamento explícito de feedback aprovado continua disponível.
- Os chunks `feedback/runtime_auto` contaminados do `datahub2` foram removidos da base vetorial.

## Revalidação operacional após correção
- Teste repetido via API autenticada com `davi.custodio@embrapa.br` em `2026-03-11`.
- Resultado observado:
  - `decision_source = vanna_generate_sql`;
  - SQL gerada com `LIMIT 10`;
  - sem filtro indevido por `MATO GROSSO`;
  - `R10` passou a registrar que a memória depende de feedback validado.
- Latência após simplificação do pipeline:
  - primeiro request depois do restart do backend: ~`14.5s`;
  - request subsequente: ~`10.3s`;
  - a execução SQL em si continua muito menor que o total.

## Diagnóstico atual: gargalo geoespacial no Lab
- A UI do `Lab` só usa `result.geo` em um ponto funcional: botão/modal `Ver payload geo` em `ai-data-pilot-manager/src/pages/LabPage.tsx`.
- O contrato do frontend (`ai-data-pilot-manager/src/api/client.ts`) já expõe `modalities`, então a tela pode indicar `text/table/chart/map` sem depender de `result.geo`.
- O cliente HTTP do manager usa `timeout: 120000` em `ai-data-pilot-manager/src/api/http.ts`; portanto, qualquer serialização pesada no `ask` pode aparecer como timeout da UI mesmo quando o SQL é válido.
- No backend, o `RuntimeOrchestrator` sempre executa `geo_engine.route_and_build_payload(...)` quando o resultado contém coluna geométrica, tanto no fluxo síncrono quanto no background.
- Em `api-geo-nlp/app/modules/geospatial/routing_engine.py`, a rota `A_geojson` monta o payload completo inline via `ST_AsGeoJSON(...)` + `json_agg(...)` sobre o resultado inteiro; esse é o ponto mais caro do pós-processamento geográfico.
- `SqlExecutionService.execute()` retorna colunas e linhas cruas sem sanitização por tipo; se a SQL final inclui `geom`, essa coluna entra em `TableSpec` e segue para JSON.
- `_remove_geometry_for_non_map_questions()` no agente só remove geometria quando a pergunta **não** sugere mapa; para perguntas com sinal geográfico, a coluna permanece no SQL/resultado.
- `LabPage` renderiza todas as colunas de `result.table.columns` e todos os valores de `result.table.rows` sem filtro defensivo para `geom`/`geometry`/`geojson`.
- Conclusão: há duas fontes independentes de custo para perguntas geográficas no Lab:
  - geração síncrona do payload GeoJSON no backend;
  - transporte/renderização tabular de colunas geoespaciais que não agregam valor à UX atual.

## Novo requisito de arquitetura: controle explícito de geometria no contrato da API
- O requisito correto não é “remover `geom` da API” de forma global, e sim permitir dois modos de execução:
  - modo tabular/analítico sem retorno de geometria;
  - modo geoespacial completo com `geom`, para integrações que precisem gerar GeoJSON ou acionar `maps-api`/GeoServer.
- O melhor ponto de controle é o contrato `AskRequest` em `api-geo-nlp/app/schemas/runtime.py`, porque ele já centraliza preferências de execução e é compartilhado pelo `/ask` e pelos transportes Vanna nativos.
- O parâmetro novo precisa ser propagado também em `api-geo-nlp/app/integrations/vanna/transport_adapter.py` e `api-geo-nlp/app/api/routes/vanna.py`, senão o comportamento ficará inconsistente entre HTTP, SSE, WebSocket e polling.
- A transformação deve ocorrer depois da SQL escolhida/validada e antes da execução no banco, como o usuário descreveu. Isso evita custo desnecessário de carregar `geom` quando o consumidor não precisa dele.
- Para o `Lab`, o valor padrão operacional deve ser “não retornar geometria”; para integrações cartográficas, o chamador poderá optar explicitamente pelo modo completo.

## Implementação concluída: `geometry_mode` no runtime e `Lab` sem payload geo
- O contrato `AskRequest` em `api-geo-nlp/app/schemas/runtime.py` agora expõe `geometry_mode = include|exclude`, com default `include` para preservar compatibilidade dos consumidores existentes.
- O `RuntimeOrchestrator` aplica o modo `exclude` logo após a validação da SQL vencedora:
  - tenta remover projeções geoespaciais da SQL antes da execução;
  - revalida a SQL preparada no guard;
  - e sanitiza o `ExecutionResult` como camada defensiva adicional.
- O enriquecimento geoespacial (`_execute_with_geo_enrichment`) e o build de `geo_payload` só permanecem ativos quando `geometry_mode=include`.
- O payload de auditoria do runtime agora registra `geometry_mode`, o que melhora rastreabilidade operacional.
- Os transportes Vanna nativos (`chat_sse`, `chat_websocket`, `chat_poll`) passaram a aceitar e propagar o mesmo parâmetro, evitando divergência entre canais.
- No frontend, o `Lab` chama a API sempre com `geometry_mode='exclude'`.
- A UI do `Lab` removeu o botão/modal de payload geográfico e passou a exibir indicadores leves via `modalities`: texto, tabela, gráfico e dados geográficos.
- A renderização tabular do `Lab` ganhou filtro defensivo para nunca mostrar colunas geométricas, mesmo em caso de regressão no backend.

## Plano técnico de observabilidade do `Perguntar`
- O fluxo atual do clique até o runtime é:
  - `ai-data-pilot-manager/src/pages/LabPage.tsx` -> `src/api/client.ts::ask()` -> `src/api/http.ts` -> `POST /projects/{project_id}/ask` -> `app/modules/runtime/orchestrator.py::ask()`.
- O manager usa `axios` com `timeout = 120000`; portanto, a latência percebida na UI pode incluir rede, refresh de token e renderização, não apenas backend.
- O contrato atual já tem pontos suficientes para plugar diagnóstico sem criar endpoint novo:
  - `AskRequest.debug_options`;
  - `AskResponse.stage_diagnostics`;
  - `AskResponse.decision_trace`;
  - auditoria `runtime.ask.completed` / `runtime.ask.failed`.
- Os pontos centrais para instrumentação são:
  - `app/modules/runtime/orchestrator.py::ask()` para spans de etapa;
  - `app/integrations/vanna/agent.py::run_vanna_prompt()` para todas as inferências textuais;
  - `app/integrations/vanna/agent.py::generate_sql_candidates()` para geração/fallback por modelo;
  - `app/modules/runtime/candidate_pipeline.py::rank_sql_candidates()` para custo/score por candidato;
  - `app/modules/runtime/services/sql_execution.py` para `EXPLAIN` e execução SQL.
- O desenho mais limpo é introduzir um coletor de métricas por request e propagar esse coletor opcionalmente, evitando espalhar `perf_counter()` avulso pelo código.
- Métricas mínimas por request:
  - `request_id`;
  - `total_backend_ms`;
  - `serialize_response_ms`;
  - `llm_total_ms`;
  - `db_total_ms`;
  - `geo_total_ms`;
  - lista de spans com `name`, `duration_ms`, `status`, `metadata`.
- Métricas mínimas por chamada LLM:
  - `call_name`, `provider`, `model`, `duration_ms`, `prompt_chars`, `response_chars`, `success`, `error`.
- Métricas mínimas por SQL:
  - `estimate_ms`, `execute_ms`, `row_count`, `truncated`, `estimated_rows`, `estimated_cost`.
- Para separar gargalo real de chamada desnecessária, a medição deve incluir testes A/B com flags temporárias:
  - sem LLM em `_classify_intent` para perguntas obviamente analíticas;
  - sem `_suggest_chart_with_vanna` para rankings/tabelas simples;
  - sem geo enrichment;
  - com redução de candidatos/modelos em `generate_sql_candidates`.

## Implementação da observabilidade do `Perguntar`
- O contrato de runtime agora expõe:
  - `AskResponse.request_id`;
  - `AskResponse.timing`;
  - `DebugOptions.include_timing_summary`.
- Foi criado o coletor `RuntimeMetricsCollector` em `app/modules/runtime/observability.py` para centralizar:
  - spans de etapa;
  - chamadas LLM;
  - operações SQL.
- O `RuntimeOrchestrator` agora registra tempos nas etapas R0, R1, T0, R2, R2B, R3, R3A, R4, R5, R6, R7 e R10, além de propagar `request_id` na resposta, no background e na auditoria.
- `run_vanna_prompt()` passou a medir inferências textuais por `call_name`, `provider`, `model`, `prompt_chars`, `response_chars` e `duration_ms`.
- `generate_sql_candidates()` agora mede cada tentativa de `vn.generate_sql()` e o fallback direto.
- `rank_sql_candidates()` agora registra spans por candidato e reaproveita o coletor nas estimativas de custo.
- `CostEstimator` e `SqlExecutionService` agora registram tempos de conexão, `EXPLAIN`, execução e `fetch`.
- No manager:
  - `ask()` gera `request_id`, envia `x-request-id` e anexa `client_metrics.http_roundtrip_ms`;
  - `LabPage` mede `click_to_response_ms`;
  - a aba de diagnósticos passou a renderizar `request_id`, `client_metrics`, `timing` e `stage_diagnostics` em um único bloco JSON.

## Harness da bateria controlada
- Foi criado `api-geo-nlp/scripts/benchmark_ask_runtime.py` para operacionalizar a próxima fase do diagnóstico sem depender de medições manuais.
- O script:
  - chama `POST /projects/{project_id}/ask` com `include_timing_summary=true`;
  - mede `http_roundtrip_ms` e `end_to_end_ms`;
  - suporta respostas `sync` e `background` com polling em `/queries/{query_id}`;
  - agrega `avg/p50/p95/% total` por cenário, span, chamada LLM e operação SQL;
  - salva saída opcional em JSON e Markdown.
- Também foi adicionado `api-geo-nlp/scripts/benchmark_questions.example.json` com cenários base para ranking, distribuição e caso geográfico.
- A execução real da bateria ainda depende de backend local ativo e de um `project_id` com dados válidos.

## Execução real do benchmark no `datahub2`
- A instância HTTP já ativa em `:8000` não devolvia `timing`, então os testes consolidados foram executados com o harness em `--mode direct`, chamando o `RuntimeOrchestrator` do código atual diretamente sobre o projeto `datahub2`.
- Smoke test do caso `quais as 10 cidades que mais produzem soja`:
  - `end_to_end_ms = 10291.71`
  - `total_backend_ms = 10242.62`
  - `llm_total_ms = 7029.44`
  - `db_total_ms = 592.71`
  - `R2.generate_candidates = 9045.37ms` (`88.31%` do backend)
- Lote estável consolidado (`ranking-soja-nacional` + `total-acai-norte`, 2 amostras contadas por cenário, 1 warmup):
  - overall:
    - `avg_end_to_end_ms = 7251.89`
    - `p50_end_to_end_ms = 7483.00`
    - `p95_end_to_end_ms = 7766.76`
    - `avg_backend_ms = 7180.97`
    - `avg_llm_ms = 5930.03`
    - `avg_db_ms = 827.72`
  - por cenário:
    - `ranking-soja-nacional`: `avg_end_to_end_ms = 6744.13`, `avg_llm_ms = 5798.69`, `avg_db_ms = 562.48`
    - `total-acai-norte`: `avg_end_to_end_ms = 7759.65`, `avg_llm_ms = 6061.36`, `avg_db_ms = 1092.96`
  - breakdown de etapas no lote estável:
    - `R2.generate_candidates = 6152.05ms` (`85.67%` do backend)
    - `R2B.rank_select = 418.54ms` (`5.83%`)
    - `R5.execute_query = 287.41ms` (`4.00%`)
    - `T0.resolve_credentials = 61.76ms` (`0.86%`)
  - breakdown de LLM:
    - `generate_sql::google/gemini-3-flash-preview = 5930.03ms` (`100%` do LLM)
  - breakdown de SQL:
    - `execute = 273.66ms`
    - `estimate = 178.67ms`
- Cenário pesado isolado `distribuicao-por-estado`:
  - warmup concluído em `37322.42ms`, com `llm_total_ms = 33025.17`
  - 1 amostra contada concluída em `21784.18ms`, com:
    - `backend_ms = 21732.60`
    - `llm_total_ms = 16196.42`
    - `db_ms = 641.86`
  - durante as tentativas seguintes o fluxo ficou instável/lento por baixa aderência e fallback de geração, o que reforça que o pior caso está no pipeline LLM/candidate generation, não no banco.

## Conclusão operacional do benchmark
- O gargalo principal atual no `datahub2` é inferência LLM no estágio `R2.generate_candidates`.
- Nos casos estáveis medidos, LLM respondeu por ~`82.58%` do backend (`5930.03 / 7180.97`).
- No pior caso medido (`distribuicao-por-estado`), LLM respondeu por ~`74.53%` do backend (`16196.42 / 21732.60`) e ainda induziu instabilidade por fallback prolongado.
- O banco não é o gargalo principal:
  - nos casos estáveis, `db_total_ms` ficou em ~`827.72ms`, bem abaixo de `llm_total_ms`;
  - no pior caso isolado, `db_ms = 641.86ms`, ainda muito abaixo de `llm_total_ms = 16196.42`.
- `R1.classify_intent` deixou de ser relevante no estado atual do código medido (`~0.03ms`), então o foco deve ficar em reduzir chamadas/modelos/fallback dentro de `generate_sql_candidates`.

## Reprodução do desvio como `ui_command`
- Em `2026-03-11`, o caso reportado foi reproduzido via UI local autenticada com `davi.custodio@embrapa.br`.
- Pergunta usada: `me de a lista de cidades do estado de santa catarina`.
- Resultado exibido no Lab:
  - mensagem principal: `Comando de UI detectado. Encaminhando para Módulo 2.`
  - nenhum SQL foi exibido/gerado.
- Evidência técnica capturada na aba de diagnóstico:
  - `request_id = 1fbec5ed-fd44-40e1-b48d-1a29b2962a6a`
  - `R1.classify_intent = 7544.82ms`
  - `llm classify_intent = 7501.7ms`
  - provider/model: `openrouter` / `z-ai/glm-4.7-flash`
  - `stage_diagnostics[1].payload.intent = "ui_command"`
  - `sql_operations = []`
- Conclusão inicial da reprodução:
  - o runtime aborta antes de `generate_sql_candidates()`;
  - a falha não está no banco nem na ausência de dados, e sim no gate/classificador de intenção que rotula a pergunta analítica como comando de UI.

## Validação do dado e do padrão de falha
- O projeto `datahub2` possui conexão ativa para o banco `datahub` e a senha armazenada foi validada com sucesso via `decrypt_secret`.
- Consulta direta no banco do projeto confirmou o dado:
  - `SELECT COUNT(*) FROM public.municipio WHERE nm_estado = 'SANTA CATARINA'` retornou `295`;
  - amostra ordenada retornou municípios como `ABDON BATISTA`, `ABELARDO LUZ`, `AGROLÂNDIA`, `AGRONÔMICA`.
- Reprodução comparativa via API autenticada:
  - `me de a lista de cidades do estado de santa catarina` -> `intent = ui_command`, sem SQL;
  - `liste as cidades do estado de santa catarina` -> `intent = ui_command`, sem SQL;
  - `traga os municipios do estado de santa catarina` -> `intent = ui_command`, sem SQL;
  - `quais as cidades do estado de santa catarina` -> `intent = analytic_sql`, consulta executada com sucesso.
- No caso bem-sucedido, a UI exibiu a SQL:
  - `SELECT nm_municip FROM public.municipio WHERE nm_estado = 'SANTA CATARINA';`
- O diferencial entre falha e sucesso está no prefixo linguístico:
  - formas interrogativas com `quais` entram na heurística rápida de `analytic_sql`;
  - formas imperativas (`me de`, `liste`, `traga`) caem no LLM de classificação e podem ser rotuladas incorretamente como `ui_command`.

## Causa raiz no código
- Em `app/modules/runtime/orchestrator.py`, `_classify_intent()` só classifica como `analytic_sql` por heurística quando encontra pistas como `quais`, `qual`, `ranking`, `total`, `distribu`, `quanto` e similares.
- A lista de pistas não inclui expressões analíticas básicas em modo imperativo, como `lista`, `liste`, `traga`, `me de`, `mostre`, `retorne`.
- Quando a heurística falha, o código consulta o LLM leve com um prompt binário (`ui_command` vs `analytic_sql`).
- Se o LLM responder `ui_command`, ainda existe um override protetor `_looks_analytic_geo_question()`, mas ele reutiliza praticamente a mesma lista estreita de pistas analíticas; como `me de a lista...` não contém `quais/qual/total/...`, o override não aciona mesmo contendo `cidade` e `estado`.
- Resultado: a pergunta é desviada em `R1`, retorna handoff de UI e nunca chega a `R2.generate_candidates()`, apesar de o banco responder corretamente.

## Lacunas de teste e robustez
- Não há testes direcionados cobrindo `_classify_intent()` para perguntas analíticas imperativas em português.
- O custo do erro é alto porque a falsa classificação para `ui_command` impede qualquer tentativa de geração de SQL.
- O uso de LLM em `R1` introduz latência de vários segundos em perguntas simples e, neste caso, ainda produz falso positivo.

## Implementação da correção
- Em `2026-03-11`, `_classify_intent()` foi endurecido para:
  - usar uma lista explícita de comandos reais de UI;
  - reconhecer perguntas analíticas em português também no modo imperativo (`me de`, `liste`, `traga`, `mostre`, `retorne`, `exiba`, `quero ver`) quando acompanhadas de pistas de domínio;
  - não aceitar `ui_command` vindo apenas do LLM quando a frase não contiver keyword explícita de interface.
- Também foi adicionada observabilidade mínima no override de intenção para registrar quando o LLM sugerir `ui_command` sem base lexical explícita.
- Em `app/integrations/vanna/agent.py`, `_repair_sql_with_known_schema()` passou a normalizar filtros textuais geográficos (`nm_estado`, `nm_regiao`, `nm_municip`, `uf`, `uf_estado`) para comparação case-insensitive via `UPPER(...) = UPPER(...)`.

## Validação pós-correção
- Testes locais:
  - `pytest tests/unit/test_runtime_intent_classification.py tests/unit/test_runtime_orchestrator_geo.py tests/unit/test_vanna_agent_relevance.py -q`
  - resultado: `33 passed`.
- Revalidação HTTP autenticada após restart da API:
  - `me de a lista de cidades do estado de santa catarina`
    - `intent = analytic_sql`
    - `sql_generated = SELECT nm_municip FROM public.municipio WHERE UPPER(nm_estado) = UPPER('SANTA CATARINA');`
    - `row_count = 295`
  - `abra o mapa da camada de soja`
    - `intent = ui_command`
    - handoff de UI preservado.

## Otimização genérica de latência para lookup simples
- Em `2026-03-11`, `generate_sql_candidates()` foi alterado para reconhecer perguntas de lookup de baixa complexidade por forma linguística, sem hardcode de projeto ou schema.
- Nova estratégia:
  - para perguntas simples de listagem/lookup, o pipeline tenta primeiro um `semantic_vector_fast_path`;
  - esse caminho usa apenas os artefatos semânticos ativos do projeto no PgVector (`ddl` + `dictionary`) para montar um prompt direto;
  - se a SQL resultante for relevante, o pipeline retorna sem hidratar a memória do Vanna nem buscar exemplos similares;
  - se falhar, o fluxo completo anterior continua disponível.
- A tentativa de usar primeiro o modelo leve nesse caminho foi medida e descartada porque piorou a latência; o fast path ficou com um único call no modelo `best`.

## Ganho de latência medido na pergunta original
- Baseline antes da otimização genérica de lookup:
  - `total_backend_ms = 7638.46`
  - `llm_total_ms = 3049.71`
  - `R2.generate_candidates = 7297.69`
  - uma única chamada `generate_sql_fallback`.
- Após o `semantic_vector_fast_path`:
  - `total_backend_ms = 5924.68`
  - `llm_total_ms = 3923.3`
  - `R2.generate_candidates = 5440.26`
  - uma única chamada `generate_sql_semantic_fast_path`
  - `row_count = 295`
- Comparando com a primeira reprodução corrigida, o caminho da pergunta original saiu de ~`15.32s` backend para ~`5.92s`, preservando a resposta correta.

## Aderência adicional à direção documentada do Vanna
- Em `2026-03-11`, o fluxo `memory first` foi reforçado no agente:
  - busca direta de exemplos `qa`/`feedback` no PgVector antes da hidratação do Vanna;
  - `exact_training_match` passou a poder retornar diretamente a partir dessa memória vetorial;
  - cache em memória por `project_id + active_version + normalized_question` foi adicionado para reutilizar candidatos já gerados.
- A implementação segue a direção observada na documentação do Vanna 2.0:
  - `similar question -> fast adaptation from memory`;
  - `novel question -> deeper investigation`.

## Ganho medido com memory-first + cache
- Medição HTTP real, mesma pergunta executada duas vezes seguidas:
  - tentativa 1:
    - `total_backend_ms = 6815.1`
    - `llm_total_ms = 4189.24`
    - `row_count = 295`
  - tentativa 2:
    - `total_backend_ms = 512.35`
    - `llm_total_ms = 0`
    - `cache_hit` presente
    - `row_count = 295`
- Conclusão prática:
  - cold path caiu de ~`15.3s` para ~`6.8s`;
  - warm path repetido caiu para ~`0.5s` sem nova chamada LLM.
