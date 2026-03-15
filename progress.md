# Progress Log

## Session: 2026-03-15

### Status
- Nenhuma pendencia local sem commit no repo raiz.
- Nenhuma pendencia local sem commit em `api-geo-nlp`.
- Ultima rodada entregue e publicada:
  - `api-geo-nlp`: `5b6136c`
  - repo raiz: `bf2b626`

### Entregas Concluidas
- Cache do evidence pack por versao ativa.
- Cache de custo SQL por SQL normalizada.
- Reuso da estimativa do candidato vencedor em `R4`.
- Cache TTL de credenciais por `secret_ref`.
- Cache TTL de metadata de grounding por projeto.
- Skip do `generate_sql` para candidato deterministico forte em `tier_1/tier_2`.

### Benchmark Consolidado
- Caso: `qual o estado com maior producao de uva`
- Resultado atual:
  - `avg backend ms`: `1345.41`
  - `avg llm ms`: `0.00`
  - `p50 end-to-end ms`: `1074.49`
  - `warm backend ms`: `860.18-908.43`

### Proximo Trabalho
- Construir suite de aceitacao para `llm_required`.
- Testar roteamento de modelo rapido com fallback estrito.
- Medir latencia e qualidade antes/depois por classe de pergunta.

## Test Results
| Test | Status | Notes |
|------|--------|-------|
| `pytest ... test_runtime_contextual_candidate.py ...` | pass | `59 passed` |
| Benchmark direto `qual o estado com maior producao de uva` | pass | `llm_ms = 0.00` com skip deterministico |

## Error Log
| Timestamp | Error | Resolution |
|-----------|-------|------------|
| 2026-03-15 | Cache de engine assíncrona preso ao event loop do benchmark | Cache corrigido por `event loop` |
| 2026-03-15 | Experimento de corte agressivo de prompt sem ganho consistente | Revertido |
