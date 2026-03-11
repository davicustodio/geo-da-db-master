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

## Follow-up: diagnóstico da falha em embeddings
- [x] Reproduzir a falha do botão `Gerar embeddings` com usuário real na UI local.
- [x] Correlacionar o novo job com `training/state` e `training/jobs/{job_id}/events`.
- [x] Executar o mesmo validador de runtime do backend sobre a versão semântica revisada.
- [x] Isolar a SQL responsável pela reprovação do quality gate.
- [x] Mapear lacunas de persistência/observabilidade para a falha.
- [x] Corrigir operacionalmente a `question 388` e reconcluir a revisão semântica.
- [x] Persistir detalhes do quality gate mesmo em falha.
- [x] Bloquear build/publish com diagnóstico explícito para questions inválidas.
- [x] Exibir diagnóstico na curadoria e no monitor/log do job.
- [x] Reexecutar o fluxo real até `embeddings=published`.

## Decisions
- Tratar a falha atual como problema de conteúdo gerado em `questions` mais falta de observabilidade do pipeline, não como defeito do clique da UI.
- Considerar duas frentes de solução: mitigação imediata nos dados (`question` inválida) e correção estrutural no backend para persistir detalhes de falha e, idealmente, autocurar SQL inválida antes do publish.
- Implementar o bloqueio explícito para `sql_is_valid = false` antes do publish, sem esperar o erro agregado de runtime.
- Persistir `quality_report`/`quality_failure` no job e no manifesto para a UI poder mostrar causa acionável.

## Follow-up: diagnóstico do desvio no Lab (`LIMIT 5` + filtro indevido por estado)
- [x] Reproduzir a pergunta do Lab diretamente no runtime do projeto `datahub2`.
- [x] Identificar quais chunks/candidatos estão sendo recuperados para a pergunta `quais as 10 cidades que mais produzem soja`.
- [x] Medir a latência por etapa do runtime para separar custo de LLM e custo de SQL.
- [x] Revisar o gate de qualidade do pipeline para verificar se ele cobre aderência semântica pergunta→SQL.
- [x] Consolidar causa raiz e plano de execução.

## Follow-up: alinhamento com a documentação do Vanna AI 2.0
- [x] Levantar as recomendações oficiais do Vanna 2.0 para Tool Memory, recuperação de contexto e montagem de prompt.
- [x] Comparar a arquitetura atual do projeto com o fluxo `Agent + ToolRegistry + Tool Memory` documentado pelo Vanna 2.0.
- [x] Verificar se os exemplos recuperados entram de fato no prompt do `vn.generate_sql`.
- [x] Consolidar as divergências entre o comportamento recomendado pelo Vanna e a implementação atual.
- [x] Reescrever o plano de execução priorizando aderência ao Vanna 2.0 nessa etapa.
- [x] Filtrar exemplos incompatíveis antes da montagem do prompt.
- [x] Substituir o prompt legado por um prompt contextual com instrução explícita de adaptação à pergunta atual.
- [x] Limpar feedbacks `runtime_auto` persistidos do `datahub2`.
- [x] Revalidar o fluxo via API autenticada após reiniciar o backend local.

## Decisions
- Tratar o problema do Lab como falha combinada de corpus, recuperação, ranqueamento e aprendizado automático, não como erro isolado da tela.
- Priorizar remoção do aprendizado automático não validado (`runtime_auto`) antes de qualquer novo retraining, porque ele contamina a memória vetorial e mascara a qualidade do corpus curado.
- Endurecer a validação de aderência pergunta→SQL no runtime e no quality gate com foco em cardinalidade solicitada (`top 10`), filtros explícitos vs. implícitos e entidades geográficas pedidas/não pedidas.
- Atacar performance reduzindo chamadas LLM evitáveis no runtime síncrono: classificação de intenção para perguntas óbvias, sugestão de gráfico e geração por múltiplos modelos quando já houver candidatos confiáveis.
- Considerar o estado atual como integração híbrida/legada com Vanna 2.0, não como implementação fiel do pipeline oficial do Agent Framework.
- Tratar como desvio crítico o fato de a memória similar ser usada como SQL candidata direta antes da geração do prompt, porque isso contorna a adaptação contextual que a documentação do Vanna 2.0 descreve.
- Corrigir a compatibilidade de `get_similar_question_sql()` com o contrato esperado pelo prompt do Vanna legado (`dict` com `question` e `sql`), ou abandonar essa via e migrar explicitamente para o fluxo oficial do Agent/Tool Memory do Vanna 2.0.
