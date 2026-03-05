# Findings - Estado Atual

## Fontes lidas
- `docs/plan-master.md`
- `api-geo-nlp/docs/modulo1.md`
- `api-geo-nlp/app/core/config.py`
- `api-geo-nlp/app/modules/training/services/artifact_store.py`
- `api-geo-nlp/app/modules/training/orchestrator.py`
- `api-geo-nlp/app/modules/training/services/metadata_discovery.py`
- `api-geo-nlp/app/modules/training/services/corpus_builder.py`
- `api-geo-nlp/app/integrations/vanna/agent.py`
- `api-geo-nlp/app/integrations/vanna/memory_backend.py`
- `api-geo-nlp/app/api/routes/training.py`
- `api-geo-nlp/app/api/routes/runtime.py`
- `api-geo-nlp/app/schemas/runtime.py`
- `api-geo-nlp/app/core/dependencies.py`

## Descobertas principais
1. **Persistencia de artefatos ainda local em disco**
- `ArtifactStore` grava em `./training/{project_id}/{version}/...`
- `active_version.txt` tambem e local por projeto
- Isso conflita com o requisito de centralizar no banco `ai-data-pilot`.

2. **Memoria vetorial ainda nao usa PostgreSQL/pgvector**
- Config tem enum `postgres`, mas implementacao efetiva esta em `memory` ou `chromadb`.
- Hidratação do Vanna usa arquivos locais (`ddl.md`, `dictionary.md`, `questions.json`) do filesystem.

3. **Autorizacao atual e simplificada**
- `UserContext` vem de headers (`x-user-id`, `x-user-groups`), sem login/senha da aplicacao.
- Rotas de treino exigem apenas `is_admin`.
- Nao existe modelo nativo de papeis `admin/manager/user` do produto.

4. **Pipeline de treino T1-T5 existe e esta funcional**
- Endpoints `discovery`, `questions:generate`, `build`, `publish`, `jobs`.
- Ja existe quality gate e publicacao de versao ativa.
- Base boa para evoluir sem refazer do zero.

5. **Runtime ja tem elementos de multimodalidade**
- `AskResponse` suporta `text`, `table`, `chart`, `map`.
- Existe politica sync/background e status de queries assicronas.
- Nao ha camada dedicada de homologacao (Certo/Errado + SQL correto) exposta por endpoint dedicado.

6. **Segredo de conexao de projetos ainda depende de fallback fraco**
- `SecretResolver` usa Vault opcional; fallback pega `db_*` do `.env` da API.
- Necessario modelo de credenciais por projeto, com criptografia forte e rotacao.

7. **Sem evidencias de MCP `db-ia-data-copilot` exposto nesta sessao**
- `list_mcp_resources` e `list_mcp_resource_templates` retornaram vazio.
- Plano precisa prever essa integracao como dependencia externa a validar.

