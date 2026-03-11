# Findings

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
