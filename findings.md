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
