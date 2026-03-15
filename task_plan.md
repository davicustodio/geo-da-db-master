# Task Plan: Proximo Gargalo do Runtime NL2SQL

## Goal
Atacar o proximo gargalo do runtime depois da eliminacao do `memory_adaptation_generator`, do cache quente do Vanna, do cache do evidence pack, do cache de custo SQL e do skip do LLM para candidatos deterministicos fortes.

## Current Phase
Phase 1

## Phases
### Phase 1: Baseline do novo hot path
- [ ] Consolidar benchmark atual do caminho com LLM
- [ ] Separar cenarios `deterministic_skip` vs `llm_required`
- [ ] Confirmar quais perguntas ainda dependem do `generate_sql`
- **Status:** pending

### Phase 2: Suite de aceitacao para roteamento de modelo
- [ ] Extrair conjunto de perguntas validadas de `qa/feedback`
- [ ] Classificar casos simples, medianos e complexos
- [ ] Definir criterios automáticos de aceitacao/rejeicao do SQL rapido
- **Status:** pending

### Phase 3: Roteamento de modelo com fallback estrito
- [ ] Implementar tentativa com modelo mais rapido apenas quando nao houver candidato deterministico forte
- [ ] Validar por `QueryContract`, relevancia, `SemanticCritic`, guard e custo
- [ ] Cair para o modelo principal em qualquer incerteza
- **Status:** pending

### Phase 4: Benchmark e rollout controlado
- [ ] Medir latencia e qualidade antes/depois por classe de pergunta
- [ ] Definir flags e thresholds de rollout
- [ ] Atualizar relatorio tecnico com a decisao final
- **Status:** pending

## Key Questions
1. Em quais perguntas o gargalo real ainda e o `generate_sql` do modelo principal?
2. Qual modelo rapido pode entrar no sync path sem degradar qualidade?
3. Quais gates sao suficientes para aceitar SQL rapida e quando obrigar fallback?

## Decisions Made
| Decision | Rationale |
|----------|-----------|
| Nao trocar o modelo principal sem suite de aceitacao | Ganho de latencia sem garantia de SQL correta nao serve para producao |
| Priorizar eliminação de chamadas LLM desnecessarias antes de roteamento de modelo | Segue a linha de pruning/verification observada em BIRD/Spider |
| Tratar `deterministic_skip` como caminho preferencial para perguntas simples | Remove o maior custo com risco menor do que trocar modelo |

## Errors Encountered
| Error | Attempt | Resolution |
|-------|---------|------------|
| Cache de engine assíncrona compartilhado entre event loops quebrou benchmark direto | 1 | Corrigido com cache por credencial + `event loop` |
| Corte agressivo de prompt principal reduziu tokens mas não melhorou latencia com consistencia | 1 | Revertido; manter budget anterior |

## Notes
- O proximo ciclo deve focar apenas no caminho que ainda exige LLM.
- O critério de aceite continua sendo `abstention-first`: se houver dúvida, usar o modelo principal.
