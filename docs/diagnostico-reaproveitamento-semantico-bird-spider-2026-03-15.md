# Diagnóstico de reaproveitamento semântico para NL2SQL

Data: 2026-03-15  
Escopo: benchmark BIRD, benchmark Spider e runtime atual de `api-geo-nlp`

## Objetivo

Investigar o que o ecossistema BIRD e Spider está propondo para otimizar tempo de resposta quando a pergunta nova é semanticamente parecida com uma pergunta já respondida corretamente, e avaliar se a API atual tem espaço para uma camada desse tipo sem voltar ao erro de reaproveitar SQL estática incorretamente.

## Resposta curta

Os melhores trabalhos e benchmarks não apontam para "cachear SQL final de pergunta parecida e devolver direto" como estratégia principal.

O padrão dominante é outro:

1. recuperar exemplos corretos e contexto relevante;
2. selecionar poucos exemplos semanticamente compatíveis;
3. adaptar a estrutura da consulta ao novo pedido;
4. verificar contrato semântico, schema e custo;
5. só então usar a resposta reaproveitada, ou cair para geração mais cara.

Portanto, a oportunidade correta para a nossa API não é voltar ao cache semântico ingênuo. A oportunidade correta é criar uma camada de **reaproveitamento estrutural validado**, baseada em `query family`/`query template`, com placeholders e checagens de contrato.

## O que o BIRD está sinalizando

### 1. Eficiência virou métrica de primeira classe

O site oficial do BIRD destaca `R-VES` e o leaderboard passou a premiar sistemas que combinam correção com eficiência, não apenas `Execution Accuracy`.

Fontes:

- [BIRD benchmark](https://bird-bench.github.io/)
- [LiveSQLBench](https://livesqlbench.ai/)

### 2. Escalar interação e candidatos, não copiar SQL pronta

O ecossistema BIRD está empurrando para:

- self-consistency com poucos/muitos candidatos;
- `Interaction-Time Scaling` no BIRD-Interact;
- loops de critic/repair no BIRD-CRITIC;
- recuperação de evidências, valores e schema antes da geração.

Isso otimiza o tempo médio porque:

- perguntas fáceis passam por caminhos curtos;
- perguntas difíceis recebem mais esforço só quando necessário.

Fontes:

- [BIRD-Interact](https://bird-interact.github.io/)
- [BIRD-CRITIC](https://bird-critic.github.io/)
- [Agentar-Scale-SQL](https://arxiv.org/abs/2509.24403)
- [CHASE-SQL](https://arxiv.org/abs/2410.01943)
- [CHESS](https://arxiv.org/abs/2405.16755)

### 3. Reaproveitamento aparece como retrieval de exemplos e workload memory

Nos materiais ligados ao BIRD, o reaproveitamento mais consistente aparece como:

- retrieval de valores e contexto;
- dynamic few-shot;
- seleção de candidatos usando experiência anterior;
- aprendizado sobre workload/banco.

O paper de extração automática de metadados também reforça que metadados, perfis e sinais derivados do banco reduzem o custo inferencial porque evitam que o modelo "redescubra" o domínio a cada pergunta.

Fontes:

- [Automatic Metadata Extraction for Text-to-SQL](https://arxiv.org/abs/2505.19988)
- [TailorSQL](https://www.vldb.org/cidrdb/papers/2025/p49-zhong.pdf)

## O que o Spider e seus métodos mais influentes estão propondo

### 1. Dynamic few-shot com seleção por similaridade

O ecossistema Spider consolidou o padrão de selecionar exemplos resolvidos por similaridade semântica e estrutural, em vez de reaproveitar a SQL final diretamente.

O caso mais explícito é `DAIL-SQL`, que enfatiza seleção de exemplos por:

- similaridade da pergunta;
- similaridade da query;
- organização eficiente de poucos exemplos no prompt.

Fontes:

- [Spider benchmark](https://yale-lily.github.io/spider)
- [DAIL-SQL repository](https://github.com/BeachWang/DAIL-SQL)
- [DAIL-SQL paper](https://bolinding.github.io/papers/vldb24dailsql.pdf)

### 2. Recuperação seletiva de contexto para reduzir tokens e latência

Trabalhos na linha `CHESS` e similares, muito usados em Spider/BIRD, fazem:

- schema pruning;
- value retrieval;
- seleção de sub-schema;
- refinement iterativo só quando necessário.

O reaproveitamento, nesse desenho, é de **contexto útil** e **estruturas prováveis**, não de SQL final sem adaptação.

Fonte:

- [CHESS](https://arxiv.org/abs/2405.16755)

### 3. Workload tailoring

`TailorSQL` é particularmente relevante para o nosso problema: ele propõe adaptar o sistema ao workload real, reaproveitando padrões que já funcionaram naquele banco específico.

O ponto importante é que esse reaproveitamento é condicionado ao workload e à semântica do banco, não tratado como cache textual genérico.

Fonte:

- [TailorSQL](https://www.vldb.org/cidrdb/papers/2025/p49-zhong.pdf)

## Diagnóstico da API atual

### 1. Há memória validada, mas ela não virou fast path de adaptação

Hoje a API já tem partes importantes do mecanismo:

- só salva memória após feedback validado do usuário, em vez de `runtime_auto`;
- a Tool Memory é versionada por `project_id + version_tag`;
- a busca vetorial por usos similares já existe;
- o evidence pack já injeta exemplos validados no prompt contextual.

Pontos observados no código:

- `save_runtime_feedback()` grava na memória apenas SQL aprovada/corrigida pelo usuário:
  - `api-geo-nlp/app/api/routes/control_plane.py:1741`
- `search_similar_usage_sync()` faz busca vetorial na Tool Memory apenas em memórias `success = TRUE`:
  - `api-geo-nlp/app/integrations/vanna/tool_memory.py:383`
- `_load_memory_examples_from_tool_memory()` já recupera exemplos `question -> sql`:
  - `api-geo-nlp/app/integrations/vanna/agent.py:1425`
- `EvidencePackBuilder` já puxa exemplos `qa/feedback` parecidos para o evidence pack:
  - `api-geo-nlp/app/modules/runtime/semantic_runtime.py:201`

### 2. O gap principal está em `generate_sql_candidates()`

Apesar da infraestrutura acima, o caminho principal ainda faz:

1. `build_rule_based_candidates()`;
2. opcionalmente `generate_contextual_candidate()` para tiers mais altos;
3. `generate_sql_candidates()`, que chama `vn.generate_sql(...)` diretamente.

O problema é que `generate_sql_candidates()` hoje:

- não consulta a Tool Memory para tentar adaptação rápida;
- não usa `_candidate_cache_key()` para nada prático;
- não possui camada de `semantic_template_reuse`;
- não tenta preencher placeholders de um padrão validado antes do LLM.

Pontos observados no código:

- o runtime sempre acaba chamando `generate_sql_candidates()` no fluxo padrão:
  - `api-geo-nlp/app/modules/runtime/orchestrator.py:487`
- `generate_sql_candidates()` vai direto para `vn.generate_sql(question=question, model=model_label)`:
  - `api-geo-nlp/app/integrations/vanna/agent.py:2845`
- existe uma infraestrutura de chave de cache `_candidate_cache_key()`, mas ela está ociosa:
  - `api-geo-nlp/app/integrations/vanna/agent.py:1486`

### 3. O runtime atual já faz um reaproveitamento parcial, mas ainda caro

Há um reaproveitamento parcial via `EvidencePackBuilder` + `generate_contextual_candidate()`.

Mas ele ainda:

- depende de chamada LLM;
- entra só em `tier_2` e `tier_3`;
- não transforma um exemplo validado em template reutilizável com placeholders;
- não tem uma política explícita de abstenção por contrato.

Resultado: a API reaproveita contexto, mas ainda não reaproveita estrutura de forma barata e segura.

## Por que o cache semântico antigo falhava

O desenho antigo falhava porque misturava duas coisas diferentes:

1. recuperar um caso parecido;
2. assumir que a mesma SQL serve para a nova pergunta.

Esse salto é justamente o que os trabalhos fortes evitam.

Exemplo do seu caso:

- "qual o bioma que produz mais feijão"
- "qual o bioma que produz mais arroz"

A família da consulta é a mesma, mas a SQL final não é a mesma.

O que deve ser reaproveitado aqui é:

- o esqueleto da consulta;
- a dimensão de agrupamento;
- a métrica agregada;
- o `ORDER BY`;
- o `LIMIT 1`;
- a regra de join;
- e o fato de que `nome_produto` é um placeholder a preencher.

## Estratégia recomendada para a API

### 1. Introduzir `semantic template reuse`, não `semantic SQL cache`

Criar uma nova camada antes de `vn.generate_sql(...)`:

`validated memory -> template retrieval -> slot filling -> semantic verification -> candidate`

Essa camada deve operar somente sobre memórias validadas.

### 2. Materializar "famílias de consulta"

Para cada SQL validada, extrair e persistir:

- `query_contract` canônico:
  - intenção;
  - agregação;
  - dimensão geográfica;
  - cardinalidade pedida;
  - presença de filtro temporal;
  - presença de filtros explícitos;
- `sql_skeleton` com placeholders:
  - `produto`
  - `estado`
  - `ano_inicial`
  - `ano_final`
  - etc.;
- `required_tables`;
- `required_columns`;
- `group_by_signature`;
- `order_by_signature`;
- `safety_constraints`.

### 3. Fast path por assinatura semântica

No runtime:

1. extrair `QueryContract` da nova pergunta;
2. resolver entidades/literais com grounding de domínio;
3. buscar templates com mesmo contrato estrutural e alta similaridade;
4. preencher placeholders;
5. rejeitar se houver qualquer divergência semântica material.

### 4. Regras obrigatórias de aceitação

O template só pode virar candidato pronto quando passar em todos estes gates:

- mesmo `GROUP BY` esperado;
- mesmo `LIMIT`/cardinalidade;
- nenhuma entidade extra não pedida;
- nenhuma restrição geográfica extra;
- mesmos agregadores esperados;
- placeholders totalmente resolvidos;
- somente tabelas/joins permitidos;
- `sqlglot` confirma que as diferenças ficaram restritas aos nós parametrizáveis.

### 5. Política de abstenção

Se qualquer checagem falhar, a camada deve se abster e cair para:

1. `generate_contextual_candidate()` com evidence pack;
2. `vn.generate_sql(...)`;
3. critic/repair/ranking já existentes.

Isso é coerente com BIRD e Spider: reaproveitar agressivamente quando o caso é claro; aumentar o esforço inferencial quando não é.

## Oportunidade prática no código atual

Oportunidade alta, porque a maior parte dos blocos já existe:

- Tool Memory validada;
- vector search por versão;
- `QueryContract`;
- `domain grounding`;
- `semantic_critic`;
- `sqlglot` no runtime;
- `build_rule_based_candidates()`;
- `generate_contextual_candidate()`.

O que falta é a peça do meio:

- **compilar memória validada em template seguro**;
- **usar isso como candidato barato antes do Vanna**.

## Recomendações priorizadas

### Prioridade 1

Implementar um novo candidato `semantic_template_reuse` antes de `generate_sql_candidates()`.

### Prioridade 2

Persistir templates/famílias derivados de:

- feedback explícito correto;
- `questions` curadas e aprovadas no pipeline;
- nunca de interações automáticas não validadas.

### Prioridade 3

Separar caches em três níveis:

1. cache exato por pergunta normalizada;
2. cache de contexto/evidence pack;
3. cache estrutural por template semântico.

Somente o nível 3 deve ser usado para perguntas semanticamente parecidas.

### Prioridade 4

Adicionar benchmark interno com pelo menos estes cenários:

- mesma pergunta repetida;
- mesma família com literal trocado;
- mesma família com cardinalidade trocada;
- mesma família com dimensão trocada;
- caso parecido, mas semanticamente incompatível.

## Conclusão

O pipeline atual está seguro do ponto de vista de memória validada, mas ainda deixa performance na mesa porque não tem uma camada explícita de reaproveitamento estrutural.

O que BIRD e Spider sugerem não é reviver o cache semântico antigo. O que eles sugerem, na prática, é:

- retrieval seletivo;
- dynamic few-shot;
- workload awareness;
- adaptação condicionada;
- verificação forte;
- abstenção quando a similaridade estrutural não basta.

Em termos práticos para esta API, a melhor próxima etapa é implementar **reaproveitamento semântico por template validado e parametrizado**, não reaproveitamento por SQL final cacheada.
