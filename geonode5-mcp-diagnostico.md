# Diagnostico: GeoNode 5 API e viabilidade de um MCP

Data da analise: 2026-03-15

## 1. Resumo executivo

E viavel implementar um servidor MCP para interagir com a API do GeoNode 5.

O principal fundamento tecnico e que o GeoNode 5 continua expondo a familia de endpoints da API v2 para recursos, uploads, documentos, usuarios, grupos, permissoes e, no eixo novo de 5.x, metadados baseados em JSON Schema. A documentacao oficial encontrada mostra exemplos concretos de uso da API e tambem evidencia endpoints de schema e metadata que sao particularmente adequados para exposicao via MCP.

Minha recomendacao e construir um MCP em duas camadas:

1. `resources` MCP para introspeccao e leitura de schemas/metadados.
2. `tools` MCP para busca, consulta detalhada e mutacoes controladas.

O desenho mais seguro para a primeira versao e `read-first`, com escrita habilitada apenas em operacoes explicitamente aprovadas.

## 2. Fontes oficiais usadas

- GeoNode 5.0.x development docs: https://docs.geonode.org/en/5.0.x/devel/
- GeoNode API usage examples: https://docs.geonode.org/en/5.0.x/devel/api/
- GeoNode architecture/security docs: https://docs.geonode.org/en/5.0.0/advanced/components/
- GeoNode metadata intro: https://docs.geonode.org/en/master/devel/metadata/intro.html
- GeoNode metadata design: https://docs.geonode.org/en/master/devel/metadata/design.html
- GeoNode API v2 schema reference: https://docs.geonode.org/en/4.4.0/devel/api/V2/index.html
- MCP tools concept: https://modelcontextprotocol.io/docs/concepts/tools
- MCP Python SDK docs: https://py.sdk.modelcontextprotocol.io/

## 3. O que a documentacao mostra

### 3.1 API v2 ativa e reutilizavel

Na documentacao oficial de 5.0.x, a secao `GeoNode API` descreve fluxos com a API v2 para:

- listagem e consulta de resources
- upload de datasets
- download de resources
- criacao de documentos
- assets e linked resources
- usuarios, grupos e permissoes

Isso indica que o GeoNode 5 continua orientado a uma API REST consistente e suficientemente ampla para ser encapsulada por um MCP.

### 3.2 Metadados em 5.x sao um diferencial importante

As paginas de metadata do GeoNode 5 descrevem um novo engine baseado em JSON Schema, com endpoints como:

- `/api/v2/metadata/schema`
- `/api/v2/metadata/instance/<PK | UUID>`
- familias `/api/v2/metadata/...`

Isso e valioso para MCP porque:

- schemas podem virar `resources` consultivos
- validacoes podem ser feitas antes de chamar operacoes de escrita
- agentes podem descobrir estrutura de metadados dinamicamente

### 3.3 Autenticacao

Os exemplos da API publicados para 5.0.x usam `Authorization: Basic ...`.

Ao mesmo tempo, a documentacao de arquitetura menciona OAuth2 com Django OAuth Toolkit e escopos como `read`, `write` e `groups`.

Diagnostico:

- um MCP pode suportar Basic Auth como caminho inicial pragmatico
- tambem deve prever Bearer/OAuth2 para instalacoes mais restritas
- idealmente o servidor MCP abstrai isso via config

### 3.4 Descoberta de schema

A referencia OpenAPI localizada documenta `GET /api/v2/schema/` com formatos OpenAPI 3 em YAML ou JSON.

Observacao importante:

- encontrei esse indice formal em `4.4.0`
- a documentacao `5.0.x` continua apontando o uso da API v2
- portanto, a OpenAPI encontrada e evidencia forte, mas deve ser validada contra a instancia-alvo do GeoNode 5

## 4. Viabilidade de MCP

## Veredito

Viabilidade: alta

Razoes:

- API REST ampla e documentada
- endpoints de schema e metadata favorecem descoberta
- autenticacao suportavel no lado do servidor MCP
- operacoes de leitura e busca mapeiam muito bem para `tools`
- schemas e catálogos mapeiam bem para `resources`

## Principais riscos

- divergencias entre versoes/documentacao e uma instancia especifica
- variacoes de auth entre deployments
- endpoints de escrita com side effects exigem controle fino
- uploads podem demandar tratamento especial de multipart/form-data

## 5. Como o MCP pode ser implementado

## Stack sugerida

Opcao recomendada: Python

Motivo:

- integra bem com clientes HTTP e auth variados
- o SDK oficial de MCP em Python esta maduro o suficiente para um servidor simples
- facilita operacoes com JSON Schema e payloads dinamicos do GeoNode

Bibliotecas sugeridas:

- `mcp` ou SDK oficial Python do MCP
- `httpx`
- `pydantic`
- `tenacity` para retry

## Arquitetura proposta

### Camada 1: cliente GeoNode

Responsabilidades:

- guardar `base_url`
- aplicar auth Basic ou Bearer
- centralizar retries, timeout e tratamento de erro
- expor metodos internos como:
  - `list_resources`
  - `get_resource`
  - `search_resources`
  - `get_metadata_schema`
  - `get_metadata_instance`
  - `upload_dataset`
  - `create_document`
  - `update_user`

### Camada 2: adaptador MCP

Responsabilidades:

- registrar `tools`
- registrar `resources`
- converter entradas do MCP em chamadas REST
- padronizar erros para o cliente MCP

### Camada 3: politicas de seguranca

Responsabilidades:

- habilitar escrita por feature flag
- permitir lista explicita de endpoints mutaveis
- sanitizar logs para nao expor credenciais
- opcionalmente exigir confirmacao para mutacoes

## 6. Como ficaria o MCP

## Resources MCP recomendados

1. `geonode://schema/openapi`
   Retorna o schema OpenAPI obtido de `/api/v2/schema/`.

2. `geonode://metadata/schema`
   Retorna o JSON Schema de metadados.

3. `geonode://resource/{id}`
   Retorna snapshot normalizado de um resource especifico.

4. `geonode://metadata/instance/{id}`
   Retorna a instancia de metadados de um objeto.

## Tools MCP recomendados

1. `geonode_search_resources`
   Params: `query`, `page`, `page_size`, `ordering`, `resource_type`

2. `geonode_get_resource`
   Params: `id`

3. `geonode_list_datasets`
   Params: `page`, `page_size`, `ordering`, `filters`

4. `geonode_get_metadata_schema`
   Params: `schema_name?`

5. `geonode_get_metadata_instance`
   Params: `id_or_uuid`

6. `geonode_create_document`
   Params: payload do documento
   Estado: opcional, somente quando escrita estiver habilitada

7. `geonode_upload_dataset`
   Params: arquivo + campos de upload
   Estado: opcional, somente quando escrita estiver habilitada

8. `geonode_update_permissions`
   Params: `resource_id`, payload
   Estado: opcional, sensivel, deve ficar desabilitado por padrao

## Exemplo de configuracao

```json
{
  "geonode": {
    "baseUrl": "https://geonode.exemplo.com",
    "authMode": "basic",
    "username": "svc_mcp",
    "passwordEnv": "GEONODE_PASSWORD",
    "timeoutSeconds": 20,
    "enableWriteTools": false
  }
}
```

## Exemplo de esqueleto em Python

```python
import os
import httpx
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("geonode")


class GeoNodeClient:
    def __init__(self) -> None:
        self.base_url = os.environ["GEONODE_BASE_URL"].rstrip("/")
        self.username = os.environ.get("GEONODE_USERNAME")
        self.password = os.environ.get("GEONODE_PASSWORD")
        self.token = os.environ.get("GEONODE_TOKEN")

    def _client(self) -> httpx.Client:
        headers = {}
        auth = None
        if self.token:
            headers["Authorization"] = f"Bearer {self.token}"
        elif self.username and self.password:
            auth = (self.username, self.password)
        return httpx.Client(base_url=self.base_url, headers=headers, auth=auth, timeout=20.0)

    def search_resources(self, query: str, page: int = 1, page_size: int = 10) -> dict:
        with self._client() as client:
            resp = client.get("/api/v2/resources", params={
                "search": query,
                "page": page,
                "page_size": page_size,
            })
            resp.raise_for_status()
            return resp.json()


client = GeoNodeClient()


@mcp.tool()
def geonode_search_resources(query: str, page: int = 1, page_size: int = 10) -> dict:
    return client.search_resources(query=query, page=page, page_size=page_size)
```

## 7. Estrategia de implementacao recomendada

### Fase 1

- implementar auth Basic e Bearer
- expor apenas leitura
- adicionar `search`, `get_resource`, `get_metadata_schema`, `get_metadata_instance`
- validar contra uma instancia GeoNode 5 real

### Fase 2

- adicionar `create_document`
- adicionar `upload_dataset`
- adicionar dry-run ou validacao previa por schema

### Fase 3

- adicionar permissoes/grupos apenas se houver caso de uso forte
- instrumentar auditoria
- tratar rate limit, retries e observabilidade

## 8. Diagnostico final

Sim, faz sentido implementar um MCP para GeoNode 5.

O encaixe tecnico e bom porque a API v2 ja entrega uma base REST ampla e o novo modelo de metadata do 5.x adiciona descoberta estruturada, algo especialmente util para agentes. O ponto de cuidado nao e a existencia da API, e sim a variacao de deployment: auth, endpoints habilitados e aderencia exata ao schema devem ser validados na instancia-alvo.

Se eu fosse executar o projeto, comecaria por um MCP read-only pequeno e testavel, deixando uploads e alteracoes administrativas para uma segunda etapa.
