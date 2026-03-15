# Findings & Decisions

## Requirements
- Encontrar a documentacao da API do GeoNode 5 na internet.
- Verificar a possibilidade de implementar um MCP para interagir com essa API.
- Gerar um relatorio de diagnostico explicando como isso pode ser feito e como ficaria.

## Research Findings
- A documentacao oficial relevante para GeoNode 5 esta em `docs.geonode.org`, especialmente a secao `GeoNode API` em `https://docs.geonode.org/en/5.0.x/devel/`.
- A secao `GeoNode API` lista casos de uso da API v2 para recursos, upload, download, permissoes, assets e linked resources.
- A documentacao de seguranca mostra que GeoNode usa OAuth2 internamente e menciona `AUTH_IP_WHITELIST` para restringir chamadas REST de usuarios/grupos.
- No GeoNode 5.x foi introduzido um novo motor de metadados baseado em JSON Schema.
- A documentacao de design do metadata informa endpoints especificos: `/api/v2/metadata/schema`, `/api/v2/metadata/instance/<PK | UUID>` e outros `/api/v2/metadata/<...>`.
- A busca por schema OpenAPI retornou a pagina `API v2 - Schema` em `docs.geonode.org/en/4.4.0/devel/api/V2/index.html`, o que indica que a referencia de schema formal ainda esta publicada em uma trilha 4.4.0, embora a documentacao 5.0.x aponte para a mesma familia de API v2.
- A pagina oficial `API usage examples` em `5.0.x` documenta operacoes concretas como `POST /api/v2/uploads/upload`, `POST /api/v2/documents`, `PATCH /api/v2/users/{pk}` e operacoes de grupos/permissoes.
- Os exemplos oficiais da API v2 usam header `Authorization: Basic ...`, indicando que Basic Auth e um caminho viavel em muitas instalacoes para um MCP servidor-servidor.
- A documentacao de arquitetura descreve OAuth2 com Django OAuth Toolkit e escopos `read`, `write` e `groups`, entao um MCP tambem pode suportar OAuth2/Bearer quando a instancia exigir.
- A referencia OpenAPI declara `GET /api/v2/schema/` com resposta OpenAPI 3 em YAML ou JSON, o que abre caminho para descoberta automatizada de capacidades.
- A API v2 exposta na referencia usa padroes previsiveis de listagem e filtragem, como `page`, `page_size`, `ordering` e `search`.
- Diagnostico preliminar: a API do GeoNode 5 e suficiente para sustentar um servidor MCP util para consulta, busca, inspecao de schema, leitura/escrita de metadados e operacoes selecionadas de upload/gestao.

## Technical Decisions
| Decision | Rationale |
|----------|-----------|
| Produzir relatorio final tambem em arquivo Markdown local | Facilita revisao e reutilizacao posterior |
| Propor `resources` MCP para schemas e `tools` MCP para operacoes REST | Fica alinhado com a natureza consultiva versus transacional da API do GeoNode |

## Issues Encountered
| Issue | Resolution |
|-------|------------|
| Nenhum ate o momento | N/A |

## Resources
- Skill `planning-with-files`: /Users/davi/.codex/skills/planning-with-files/SKILL.md
- Skill `valyu-best-practices`: /Users/davi/.agents/skills/valyu-best-practices/SKILL.md
- GeoNode API 5.0.x: https://docs.geonode.org/en/5.0.x/devel/
- GeoNode OAuth2/security docs: https://docs.geonode.org/en/5.0.0/advanced/components/
- GeoNode 5 metadata intro: https://docs.geonode.org/en/master/devel/metadata/intro.html
- GeoNode metadata design: https://docs.geonode.org/en/master/devel/metadata/design.html
- API v2 schema reference: https://docs.geonode.org/en/4.4.0/devel/api/V2/index.html
- MCP tools concept: https://modelcontextprotocol.io/docs/concepts/tools
- MCP Python SDK docs: https://py.sdk.modelcontextprotocol.io/

## Visual/Browser Findings
- A pagina `GeoNode API` em 5.0.x funciona como indice de capacidades da API, nao como schema formal completo.
- As paginas de metadata em `master` deixam explicito que a novidade de 5.x esta no modelo dinamico de metadados e em endpoints REST/JSON Schema adicionais.
- A referencia OpenAPI documenta explicitamente `GET /api/v2/schema/`, reforcando a possibilidade de descoberta dinamica num servidor MCP.
