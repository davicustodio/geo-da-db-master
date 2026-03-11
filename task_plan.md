# Task Plan

## Goal
Implementar o redesenho em duas etapas do pipeline:
1. preparar e curar a base semântica do projeto (DDL, Dictionary, Questions);
2. gerar/publicar embeddings no Vanna para liberar o RAG/Lab.

O entregável inclui backend, frontend e testes atualizados.

## Phases
- [x] Mapear o fluxo atual de setup, treinamento, curadoria e lab.
- [x] Definir o modelo-alvo em `api-geo-nlp/docs/plano-ui-pipeline-duas-etapas.md`.
- [x] Separar backend em estado de base semântica e estado de embeddings.
- [x] Ajustar a UI para refletir as duas etapas e a dependência entre elas.
- [x] Atualizar testes e executar verificações principais.

## Decisions
- Usar o skill `planning-with-files` porque a tarefa é de organização e definição estratégica.
- Implementar a separação explícita entre `base semântica` e `embeddings do Vanna`.
- Fazer a geração inicial parar em `discovery + questions`; embeddings passam a ser ação separada.
- Inutilizar embeddings quando houver edição posterior em metadata, policies, DDL, Dictionary ou Questions.
- Usar a versão mais recente como `working version` para curadoria, mantendo a versão ativa apenas para runtime/Lab.

## Errors Encountered
- `eslint` não está disponível no ambiente atual (`npm run lint` falha com `eslint: command not found`).
