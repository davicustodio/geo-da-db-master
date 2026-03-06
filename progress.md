# Progress Log

## 2026-03-05

### [Concluido] Coleta de contexto tecnico
- Leitura do `plan-master` e `modulo1`.
- Inspecao da `api-geo-nlp` para identificar estado real de:
  - pipeline de treinamento;
  - armazenamento de artefatos;
  - backend de memoria vetorial;
  - seguranca/autenticacao.

### [Concluido] Inicializacao de planejamento em arquivo
- Criado `task_plan.md`.
- Criado `findings.md`.
- Criado `progress.md`.

### [Concluido] Plano detalhado do subprojeto
- Criado `ai-data-pilot-manager/docs/plano-modulo3-implementacao.md`.
- Plano inclui arquitetura, modelo de dados, migracao para `pgvector`, backlog por fases, testes e riscos.

### [Concluido] Validacoes adicionais solicitadas pelo usuario
- Conexao do MCP `db-ia-data-copilot` localizada em `~/.codex/config.toml` (host/porta/db/usuario).
- Plano atualizado com decisoes confirmadas e recomendacao inicial de embedding.

### [Concluido] Continuacao da sessao anterior (baseline real AS-IS)
- Validado via MCP que o banco ja contem as tabelas principais do Control Plane.
- Confirmado que `vector` ainda nao esta instalado (apenas `postgis` + `pgcrypto` ativos).
- Confirmado que `project_rag_chunks` esta em fallback com `embedding_json`.
- Revisadas rotas da API e telas da UI para mapear o delta de implementacao real.

### [Concluido] Reescrita do plano executivo (versao incremental)
- `ai-data-pilot-manager/docs/plano-modulo3-implementacao.md` reestruturado para execucao por fases.
- Plano passou a refletir estado atual do ambiente (nao greenfield) e incluir:
  - estrategia git/submodule com passos praticos;
  - migracao obrigatoria para `pgvector`;
  - backlog detalhado de API/UI/DB;
  - dominio separado de usuarios/grupos/permissoes Vanna por projeto;
  - criterios de aceite e DoD global.

### [Concluido] Fase 0 iniciada na pratica (subprojeto git local)
- Criado `ai-data-pilot-manager/.gitignore`.
- Inicializado git em `ai-data-pilot-manager/`.
- Commit inicial realizado no subprojeto:
  - `2f475a6 chore: bootstrap ai-data-pilot-manager module3 control plane`

### [Bloqueado] Fase 1 no banco (ativacao pgvector)
- Executada tentativa de migracao via MCP:
  - `CREATE EXTENSION IF NOT EXISTS vector`
  - `ALTER TABLE project_rag_chunks ADD COLUMN embedding vector(1536)`
  - backfill e indice ivfflat
- Resultado: erro de infraestrutura no host PostgreSQL:
  - `extension "vector" is not available`
- Acao de mitigacao:
  - script SQL versionado em `api-geo-nlp/scripts/sql/20260305_enable_pgvector.sql`
  - ajuste de auto-heal no init schema da API (`app/modules/persistence/schema.py`) para adicionar coluna `embedding` quando o `vector` estiver disponivel futuramente.

### [Concluido] Avanco sem depender de pgvector (readiness gate)
- API: criado endpoint `GET /projects/{project_id}/training/readiness` para validar disponibilidade do laboratorio.
- API: adicionado schema `TrainingReadinessResponse`.
- UI: `LabPage` agora consulta readiness e bloqueia pergunta/feedback quando treino ainda nao estiver pronto/publicado.
- Validacao local:
  - `npm run build` em `ai-data-pilot-manager` (ok)
  - `python3 -m compileall` nos arquivos alterados da API (ok)

## 2026-03-06

### [Concluido] Reentrada de contexto e reavaliacao do escopo
- Releitura do `prompt.md`, do plano executivo existente e dos arquivos de planejamento da sessao anterior.
- Inclusao do relatorio `api-geo-nlp/docs/relatorio_bird_spider_diagnostico_otimizacao.md` no contexto.
- Reclassificacao do backlog em duas trilhas:
  - control plane / UX do Modulo 3;
  - qualidade NL2SQL Fase 1 inspirada em BIRD/SPIDER.

### [Concluido] Verificacao de banco via MCP
- `list_tables` confirma persistencia central em `ia-data-pilot`.
- `SELECT extname FROM pg_extension ...` ainda nao mostra `vector`.
- `project_rag_chunks` ainda nao possui coluna vetorial `embedding`.

### [Concluido] Verificacao de build/test frontend
- `npm run build` em `ai-data-pilot-manager` (ok).
- `npm test` em `ai-data-pilot-manager` (ok).

### [Erro identificado] Execucao de testes Python
- `pytest tests/unit -q` em `api-geo-nlp` falhou por import de plugin global:
  - `ImportError: cannot import name 'ReadableLogRecord' from 'opentelemetry.sdk._logs'`
- Acao definida:
  - executar proximas validacoes Python com `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1`.

### [Concluido] Readequacao do plano executivo
- `ai-data-pilot-manager/docs/plano-modulo3-implementacao.md` atualizado com:
  - incorporacao do relatorio BIRD/SPIDER;
  - priorizacao de `schema canonico` como fonte primaria;
  - nova Fase 2A para qualidade NL2SQL.

### [Concluido] API/control plane
- `api-geo-nlp/app/api/routes/control_plane.py`
  - novas rotas de escrita do schema;
  - listagem de feedback;
  - respostas vazias em vez de `404` para `ddl`/`dictionary`/`questions` sem corpus;
  - bloqueio de treino/retreino sem teste de conexao bem-sucedido.
- `api-geo-nlp/app/modules/control_plane/schema_sync.py`
  - geracao deterministica de `ddl` e `dictionary` a partir do schema canonico.
- `api-geo-nlp/app/modules/persistence/repository.py`
  - suporte a notas de tabela/coluna, dominio manual e listagem de feedback.

### [Concluido] Melhorias NL2SQL Fase 1
- `api-geo-nlp/app/integrations/vanna/agent.py`
  - geracao de multiplos candidatos SQL.
- `api-geo-nlp/app/modules/runtime/candidate_pipeline.py`
  - ranking heuristico e repair prompt.
- `api-geo-nlp/app/modules/runtime/orchestrator.py`
  - decision trace;
  - stage diagnostics;
  - trace seguro opcional;
  - repair loop limitado antes do fallback por memoria.
- `api-geo-nlp/app/schemas/runtime.py`
  - novos contratos de debug/trace.

### [Concluido] UI/UX do Modulo 3
- `ai-data-pilot-manager/src/components/AppShell.tsx` e `src/styles/global.css`
  - nova linguagem visual tipo control room.
- `src/pages/LoginPage.tsx`
  - redesign e remocao de senha hardcoded.
- `src/pages/ProjectsPage.tsx`, `src/pages/ProjectSetupPage.tsx`
  - fluxo mais direto para setup e treinamento.
- `src/pages/UsersPage.tsx`
  - atualizacao de perfil/status/senha.
- `src/pages/MetadataPage.tsx`
  - curadoria estruturada de schema + edicao textual de artefatos.
- `src/pages/QuestionsPage.tsx`
  - gerar, criar, editar e excluir questions.
- `src/pages/LabPage.tsx`
  - SQL/diagnostico/decision trace/feedback/modais.
- `src/routes/router.tsx`
  - lazy routes.

### [Concluido] Validacao final
- API local subida com sucesso em `http://127.0.0.1:8000`.
- UI local subida com sucesso em `http://127.0.0.1:4173`.
- MCP DB confirmou:
  - extensao `vector` ativa;
  - coluna `embedding` criada;
  - indice `idx_project_rag_chunks_embedding` ativo.
- Smoke browser com Playwright:
  - login admin;
  - pagina de projetos;
  - metadados sem corpus inicial;
  - pagina de usuarios.
- Validacoes automatizadas:
  - `python3 -m compileall app` (ok)
  - `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest -p pytest_asyncio tests/unit/test_runtime_orchestrator_geo.py tests/unit/test_vanna_agent_relevance.py tests/unit/test_artifact_store.py -q` (18 passed)
  - `npm test` em `ai-data-pilot-manager` (ok)
  - `npm run build` em `ai-data-pilot-manager` (ok, com warning remanescente no chunk opcional `charts`)

### [Reaberto] Delta restante apos confirmacao com o usuario
- O plano ainda nao estava 100% implementado.
- Gaps confirmados:
  - dominio Vanna Access por projeto (`usuarios/grupos/permissoes`) ainda ausente;
  - enforcement real dessas permissoes no runtime ainda ausente;
  - pagina/endpoint de auditoria ainda ausente.
- Estrategia definida para continuidade:
  - fechar backend Access/Audit primeiro;
  - fechar UI `/access` e `/audit`;
  - testar novamente ponta a ponta;
  - avaliar commits apenas ao final e de forma seletiva por repositorio filho.

### [Concluido] Backend Access/Audit + enforcement real
- Novas tabelas adicionadas ao auto-init:
  - `project_vanna_users`
  - `project_vanna_groups`
  - `project_vanna_group_members`
  - `project_vanna_permissions`
- API:
  - CRUD/listagens de usuarios, grupos e permissoes de projeto;
  - `GET /projects/{project_id}/audit`;
  - auditoria best-effort em operacoes relevantes.
- Runtime:
  - resolvedor de identidade de projeto;
  - `SQL guard` com allowlist de schema/tabela/coluna;
  - auditoria de execucoes sync/background.

### [Concluido] Frontend Access/Audit + navegacao de workspace
- Criadas telas:
  - `AccessPage`
  - `AuditPage`
- Criado `ProjectWorkspaceNav` e integrado nas paginas do projeto.
- `LabPage` passou a permitir simular `project_user_id` e tratar erro de runtime com mensagem amigavel.

### [Concluido] Validacao funcional final
- Banco real validado via MCP:
  - novas tabelas do dominio Vanna presentes;
  - `vector` segue ativo;
  - indice `idx_audit_events_project` presente.
- Validacao direta de runtime:
  - sem permissao: `403 PROJECT_ACCESS_DENIED`;
  - com grant `read`: bloqueio coerente por politica;
  - com grant `admin`: resposta `200` com `decision_trace` e auditoria.
- Smoke browser com Playwright:
  - login;
  - `Access` criando usuario/grupo/permissoes;
  - `Audit` exibindo eventos de control plane e runtime.
- Validacoes automatizadas finais:
  - `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest -p pytest_asyncio tests/unit/test_sql_guard.py tests/unit/test_project_access_resolver.py tests/unit/test_runtime_orchestrator_geo.py tests/unit/test_vanna_agent_relevance.py tests/unit/test_artifact_store.py -q` -> `33 passed`
  - `python3 -m compileall app` -> ok
  - `npm test` -> ok
  - `npm run build` -> ok

### [Concluido] Commits finais
- `ai-data-pilot-manager`: `627c2f8` (`feat: complete module3 control plane ui`)
- `api-geo-nlp`: `57b1f8c` (`feat: finish module3 control plane and access runtime`)
- Mantidas fora dos commits:
  - delecoes antigas em `api-geo-nlp/docs/workflow_runtime_20x_*`
