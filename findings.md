# Findings - Estado Atual (validado em 2026-03-05)

## Fontes lidas
- `docs/plan-master.md`
- `api-geo-nlp/docs/modulo1.md`
- `api-geo-nlp/app/core/config.py`
- `api-geo-nlp/app/modules/persistence/schema.py`
- `api-geo-nlp/app/modules/persistence/repository.py`
- `api-geo-nlp/app/modules/training/services/artifact_store.py`
- `api-geo-nlp/app/modules/training/services/vector_store.py`
- `api-geo-nlp/app/integrations/vanna/agent.py`
- `api-geo-nlp/app/integrations/vanna/memory_backend.py`
- `api-geo-nlp/app/api/routes/control_plane.py`
- `api-geo-nlp/app/api/routes/runtime.py`
- `api-geo-nlp/app/schemas/runtime.py`
- `ai-data-pilot-manager/src/pages/*.tsx`
- `ai-data-pilot-manager/docs/plano-modulo3-implementacao.md`

## Validacao MCP banco (`db-ia-data-copilot`)
- Tabelas de control plane ja existentes no banco `ia-data-pilot`:
  - `app_users`, `app_sessions`, `projects`, `project_members`, `project_db_connections`
  - `project_dataset_versions`, `project_training_documents`, `project_training_questions`
  - `project_schema_tables`, `project_schema_columns`, `project_column_domains`
  - `project_training_jobs`, `project_runtime_feedback`, `project_rag_chunks`, `audit_events`
- Extensoes ativas: `postgis`, `pgcrypto`.
- Extensao `vector` ainda ausente.
- `project_rag_chunks` no ambiente atual usa `embedding_json` (fallback), sem coluna vetorial ativa.
- Tentativa de migracao executada em 2026-03-05 falhou com:
  - `extension "vector" is not available`
  - `pg_available_extensions` nao lista nenhum item com `vector`.

## Descobertas principais
1. **Migracao para banco ja esta parcialmente implementada**
- `ArtifactStore` ja suporta backend `postgres` para `ddl`, `dictionary`, `questions`, `manifest` e versao ativa.
- `PgVectorStore` ja existe para rebuild e busca semantica.
- `control_plane.py` ja entrega boa parte dos endpoints pedidos.

2. **Defaults de producao ainda estao desalinhados**
- `API_GEO_NLP_ARTIFACTS_BACKEND` default = `filesystem`.
- `API_GEO_NLP_VANNA_MEMORY_BACKEND` default = `memory`.
- Sem ajuste de env, API pode continuar fora do banco central em operacao.

3. **pgvector ainda nao esta operacional no banco alvo**
- O schema da API tenta criar `vector` e cai para fallback quando nao disponivel.
- Sem `CREATE EXTENSION vector`, busca vetorial real nao fica ativa.

4. **RBAC principal do produto ja existe na API, mas escopo Vanna por projeto ainda nao esta completo**
- Perfis `admin/manager/user` e rotas de usuarios/projetos ja existem.
- Falta dominio separado para usuarios/grupos/permissoes de dados do projeto (escopo Vanna).

5. **UI ja tem scaffold funcional, mas faltam recursos de nivel produto**
- Existem telas basicas para login, usuarios, projetos, setup, metadata, questions e lab.
- Ainda faltam: editor visual estruturado completo, pagina de acesso Vanna por projeto, trilha de jobs/auditoria rica e controles avancados de laboratorio.

6. **Sincronizacao bidirecional DDL/Dictionary <-> schema canonico ainda incompleta**
- Ha `schema/overview` e `refresh-domains`.
- Faltam endpoints de escrita fina para `table_note`, `metadata_note` e dominio manual por coluna.

7. **Readiness gate do laboratorio precisa ser formalizado**
- Fluxo esperado: Lab disponivel apenas com treino publicado.
- Atualmente precisa endpoint/estado explicito para bloqueio robusto no frontend.

8. **Subprojeto git local foi formalizado**
- `ai-data-pilot-manager` agora possui repo git proprio com commit inicial.
- Ainda falta URL remota para conversao definitiva em `git submodule` no repositorio pai.

9. **Relatorio BIRD/SPIDER muda a prioridade tecnica da API**
- O maior gap competitivo da `api-geo-nlp` nao e a infraestrutura basica; e a ausencia de pipeline `search + verify + select + repair`.
- Melhorias de alto impacto e baixo risco para esta sessao:
  - geracao de multiplos candidatos SQL;
  - reranking com sinais compostos;
  - repair loop limitado por erro de execucao;
  - decision trace e diagnosticos estruturados para UI/laboratorio.

10. **Status do banco em 2026-03-06 diverge da informacao do usuario sobre pgvector**
- Consulta MCP em `ia-data-pilot` retornou apenas `pgcrypto` e `postgis` em `pg_extension`.
- `project_rag_chunks` ainda nao possui coluna `embedding`; apenas `embedding_json`.
- Isso sugere uma destas situacoes:
  - a extensao foi instalada no host, mas nao foi criada neste database;
  - a instalacao ocorreu em outro ambiente/database;
  - a API ainda nao executou auto-init/schema heal apos a instalacao.

11. **Suite frontend esta saudavel; suite Python depende de isolamento**
- `npm test` e `npm run build` em `ai-data-pilot-manager` executam com sucesso.
- `pytest` da API falhou antes de rodar os testes por interferencia de plugin global (`logfire`).

12. **Ativacao real do pgvector foi confirmada ao subir a API em 2026-03-06**
- O auto-init de `app/modules/persistence/schema.py` conseguiu executar com sucesso:
  - `CREATE EXTENSION IF NOT EXISTS vector`;
  - `ALTER TABLE project_rag_chunks ADD COLUMN IF NOT EXISTS embedding VECTOR(...)`;
  - `CREATE INDEX IF NOT EXISTS idx_project_rag_chunks_embedding ... ivfflat`.
- Validacao MCP posterior:
  - `pg_extension` passou a retornar `vector`;
  - `project_rag_chunks` passou a exibir a coluna `embedding` com `udt_name = vector`;
  - `pg_indexes` passou a listar `idx_project_rag_chunks_embedding`.

13. **Control plane agora opera com schema canonico sincronizado**
- Novas rotas de escrita:
  - `PATCH /projects/{project_id}/schema/tables/{schema_name}/{table_name}`
  - `PATCH /projects/{project_id}/schema/columns/{schema_name}/{table_name}/{column_name}`
  - `PATCH /projects/{project_id}/schema/domains/{schema_name}/{table_name}/{column_name}`
- Sempre que o schema e alterado, `ddl` e `dictionary` sao regenerados a partir da representacao canonica.
- `GET /projects/{project_id}/feedback` passou a listar feedback do laboratorio.

14. **Runtime NL2SQL evoluiu para search + verify + select + repair**
- A API agora gera multiplos candidatos SQL, ranqueia por relevancia/guard/custo/complexidade e expõe `decision_trace`.
- Em erro de execucao, existe repair loop limitado antes do fallback por memoria.
- O laboratorio consegue pedir:
  - SQL;
  - diagnostico por etapa;
  - decision trace;
  - trace seguro (quando habilitado por env).

15. **Frontend foi reestruturado para fluxo de produto**
- Login redesenhado e sem senha hardcoded.
- `ProjectsPage`, `ProjectSetupPage`, `UsersPage`, `MetadataPage`, `QuestionsPage` e `LabPage` foram evoluidas.
- Rotas passaram para lazy loading.
- O warning restante de bundle ficou concentrado no chunk opcional de charts (ECharts).

16. **O maior gap restante do plano readequado esta concentrado em Access/Audit**
- Ainda nao existem `project_vanna_users`, `project_vanna_groups`, `project_vanna_group_members` e `project_vanna_permissions`.
- O runtime continua com politica aberta por projeto (`PolicyEnforcer.open_policy`) e sem enforcement de permissoes por identidade de projeto.
- A UI ainda nao possui `/projects/:id/access` nem `/projects/:id/audit`.

17. **Estrategia de commit precisa ser seletiva por repositorio**
- O repositorio pai esta sujo e nao deve receber commit global deste trabalho.
- `ai-data-pilot-manager` e `api-geo-nlp` devem ser tratados como commits separados, apenas ao final, para evitar misturar mudancas preexistentes e este fechamento de escopo.

18. **O dominio Vanna Access agora ficou operacional ponta a ponta**
- Novas tabelas validadas no banco real: `project_vanna_users`, `project_vanna_groups`, `project_vanna_group_members`, `project_vanna_permissions`.
- O runtime passou a resolver identidade de projeto por `project_user_id`/headers equivalentes e aplicar escopo no `SQL guard`.
- Validacao direta no endpoint `/projects/{id}/ask`:
  - sem permissao: `403 PROJECT_ACCESS_DENIED`;
  - com grant `read` restrito: runtime bloqueia candidatos fora do escopo;
  - com grant `admin`: pipeline executa e responde `200`.

19. **A auditoria do projeto agora e visivel e util**
- Eventos de control plane e runtime sao gravados em `audit_events`.
- A tela `/projects/:id/audit` exibiu com sucesso:
  - `vanna.user_created`
  - `vanna.group_created`
  - `vanna.permission_created`
  - `runtime.ask.failed`
  - `runtime.ask.completed`

20. **Commits finais foram gerados por repo filho**
- `ai-data-pilot-manager`: `627c2f8` (`feat: complete module3 control plane ui`)
- `api-geo-nlp`: `57b1f8c` (`feat: finish module3 control plane and access runtime`)
- O repo `api-geo-nlp` permaneceu com delecoes nao commitadas em `docs/workflow_runtime_20x_*`, deixadas intactas por serem residuos preexistentes fora do escopo.

## Descobertas da analise de UX de treinamento (2026-03-06)

21. **O botao `Salvar e iniciar treinamento` nao acompanha o processo**
- `ProjectSetupPage.tsx` apenas chama `resetAndRegenerate(projectId)` e mostra `Pipeline iniciado com sucesso. Job: ...`.
- Nao existe polling do job, indicacao de etapa atual, termino nem falha.

22. **Os atalhos de workspace continuam liberados mesmo com treino em andamento ou inexistente**
- `ProjectWorkspaceNav.tsx` sempre habilita `Metadata`, `Questions`, `Lab`, `Access` e `Audit`.
- `ProjectsPage.tsx` tambem expoe links diretos para essas areas sem considerar readiness/job ativo.
- Hoje apenas `LabPage.tsx` consulta `training/readiness`; as demais telas nao fazem gate.

23. **O endpoint de job existe, mas a implementacao atual nao sustenta um monitor confiavel**
- `GET /projects/{project_id}/training/jobs/{job_id}` le apenas do dicionario em memoria `_jobs` no `TrainingOrchestrator`.
- Isso significa: perda de estado apos restart, nenhuma retomada em reload de pagina, nenhum historico por projeto e nenhum progresso intermediario persistido.
- A tabela `project_training_jobs` existe no schema, mas nao esta sendo usada pelo orquestrador.

24. **As acoes de treinamento do setup estao semanticamente incompletas**
- `training/reset-and-regenerate` chama apenas `start_discovery`, embora o plano/prompt esperem pipeline completo `discovery + questions + build + publish`.
- `training/retrain-existing` chama apenas `start_build`, sem publicacao automatica da versao.
- Isso pode explicar a percepcao de "terminou sem indicacao" e tambem gaps de readiness apos o suposto treinamento.

25. **Audit nao deve ser tratado como substituto do monitor de treino**
- `AuditPage.tsx` lista eventos operacionais do projeto, mas nao e um feed job-scoped nem realtime.
- Ela pode complementar a rastreabilidade, mas nao resolve sozinha status/progresso/log de treinamento.

## Implementacao aplicada na rodada atual (2026-03-06)

26. **O pipeline do setup agora executa o fluxo completo esperado**
- `training/reset-and-regenerate` passou a disparar `discovery -> questions -> build -> publish`.
- `training/retrain-existing` passou a disparar `build -> publish`.

27. **Jobs de treinamento agora sao persistidos no banco com eventos consultaveis**
- Novo store: `app/modules/training/job_store.py`.
- `project_training_jobs` passou a ser usado como fonte real de status.
- Nova tabela `project_training_job_events` registra o log estruturado por job.
- `TrainingOrchestrator` deixou de depender de `_jobs` em memoria.

28. **Foi criado um estado unificado de treinamento para a UI**
- Novo endpoint: `GET /projects/{project_id}/training/state`.
- Novo endpoint: `GET /projects/{project_id}/training/jobs/{job_id}/events`.
- O estado consolida:
  - readiness do dataset ativo;
  - job ativo;
  - ultimo job;
  - liberacao por secao (`metadata`, `questions`, `lab`, `access`, `audit`);
  - mensagem de bloqueio/andamento.

29. **A UI passou a bloquear e monitorar o workspace com base no estado real**
- `ProjectSetupPage` agora mostra:
  - card de monitoramento do job;
  - progresso por etapa;
  - ultimo status/falha;
  - modal para acompanhar o log do job.
- `ProjectWorkspaceNav` e `ProjectsPage` passaram a desabilitar `Metadata`, `Questions` e `Lab` enquanto:
  - nao existe treino concluido/publicado; ou
  - existe job de treinamento em andamento.
- `Audit` e `Access` permaneceram acessiveis.

30. **As paginas operacionais passaram a respeitar o gate de treinamento**
- `MetadataPage`, `QuestionsPage` e `LabPage` agora usam o mesmo estado central via polling.
- `LabPage` deixou de depender apenas de `training/readiness`, cobrindo tambem o caso de retreinamento em andamento com dataset ativo anterior.
