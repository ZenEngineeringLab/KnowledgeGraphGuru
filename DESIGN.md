# KnowledgeGraphGuru — Design

> **Knowledge is the source of truth.**

Este documento descreve o design interno do KnowledgeGraphGuru: o Modelo Canônico de Conhecimento, como ele mapeia para o armazenamento, o pipeline de ingestão, o modelo de eventos e a estratégia para evoluir o sistema sem quebrar o modelo.

Para uma visão geral, veja **[README.md](README.md)**.

---

## 1. Objetivos e não-objetivos

### Objetivos

- Representar conhecimento como um **grafo de objetos conectados**, independente de qualquer formato de arquivo.
- Manter um único **Modelo Canônico de Conhecimento** como a única fonte da verdade.
- Tornar as **relations cidadãs de primeira classe**, e não texto embutido no conteúdo.
- Servir **humanos e agentes de IA** via REST e MCP.
- Permanecer **serverless, extensível e agnóstico a formato**.
- Permitir que novos **renderers, analyzers e projeções** sejam adicionados sem alterar o modelo.

### Não-objetivos (v1 / MVP)

- ❌ Sem banco vetorial (embeddings vêm depois, como projeção).
- ❌ Sem banco de grafos (Neo4j/Neptune vêm depois, como projeção).
- ❌ Sem escrita direta no armazenamento — tudo passa pela API.
- ❌ Sem tratar Markdown (ou qualquer formato) como formato de armazenamento.
- ❌ Sem adotar o modelo interno do Obsidian — apenas compatibilidade de importação/exportação.
- ❌ Sem multi-tenancy — a v1 é **single-user / self-hosted** (ver §5.3).

## 2. O Modelo Canônico de Conhecimento

Tudo na plataforma é um **Knowledge Object**. Não existe o conceito de "arquivo" como entidade principal — existe apenas conhecimento.

### 2.1 As quatro dimensões

Todo Knowledge Object compartilha uma forma comum composta por quatro dimensões independentes:

| Dimensão | Propósito | Exemplos |
| --- | --- | --- |
| **Structure** | Posição na árvore | `parent`, `children`, `order` |
| **Content** | O payload do objeto | texto, código, mermaid, itens de lista |
| **Properties** | Metadados específicos do tipo | `title`, `tags`, `aliases`, `language`, `status` |
| **Relations** | Vínculos semânticos com outros objetos | `references`, `depends_on`, `related_to`, `implements`, `alternative_to` |

Separar essas dimensões é o que permite que o mesmo objeto seja renderizado para qualquer formato, enriquecido por analyzers e projetado em outros armazenamentos — sem que o conteúdo e os relacionamentos se misturem.

### 2.2 Tipos de objeto

Um catálogo não-exaustivo de tipos de Knowledge Object:

`Collection` · `Note` · `Heading` · `Paragraph` · `Callout` · `List` · `Table` · `Code Block` · `Mermaid` · `Image` · `Attachment` · `Quote`

Novos tipos podem ser introduzidos sem alterar o modelo — um tipo é definido pela sua **forma de content**, pelo seu **analyzer** e por como os **renderers** sabem projetá-lo.

### 2.3 Identidade

- Todo Knowledge Object tem um **identificador imutável** (`id`).
- O `id` nunca muda, mesmo quando content, structure, properties ou relations mudam.
- As relations referenciam objetos por `id`, então mover ou renomear um objeto nunca quebra um vínculo.

### 2.4 Exemplo ilustrativo de um objeto

> Apenas ilustrativo — o schema concreto será fixado durante a implementação.

```json
{
  "id": "ko_01H...",              // identificador imutável
  "type": "Paragraph",
  "structure": {
    "parent": "ko_01H...note",
    "order": 3
  },
  "content": {
    "format": "text",
    "value": "Relations são cidadãs de primeira classe."
  },
  "properties": {
    "tags": ["design", "principios"]
  },
  "relations": [
    { "type": "references", "target": "ko_01H...other" }
  ],
  "meta": {
    "createdAt": "2026-07-18T00:00:00Z",
    "updatedAt": "2026-07-18T00:00:00Z",
    "version": 1
  }
}
```

### 2.5 Structure vs. Collections

Duas noções diferentes de "conter" coexistem:

- **Structure** (`parent`/`children`/`order`) — a árvore *intrínseca* de um objeto (uma `Note` contém `Heading`s e `Paragraph`s).
- **Collections** — agrupamento *extrínseco*. Uma `Collection` é ela mesma um Knowledge Object, e uma nota pode pertencer a **muitas coleções simultaneamente**. Isso substitui a árvore de diretórios físicos, que deixa de existir.

### 2.6 Properties substituem o Frontmatter

O Frontmatter deixa de existir internamente. Tudo que viveria no frontmatter agora vive em **Properties**, no objeto. O renderer Markdown ainda vai *emitir* frontmatter na exportação, puramente para manter compatibilidade com ferramentas externas.

## 3. Renderers

Um **renderer** é uma projeção pura: `Knowledge Object(s) → representação`. Renderers **nunca modificam** o modelo canônico.

Renderers iniciais:

- **Markdown** — importação/exportação/visualização; re-emite frontmatter para compatibilidade.
- **HTML** — visualização para humanos.
- **JSON** — serialização canônica para máquinas.
- **MCP** — representação para agentes.

Como a renderização é unidirecional e sem efeitos colaterais, novos renderers podem ser adicionados a qualquer momento sem risco para o modelo.

## 4. Analyzers

Um **analyzer** inspeciona o content de um objeto e **enriquece o grafo de conhecimento** — tipicamente descobrindo relations, extraindo metadados ou derivando novos objetos.

Exemplos:

- **Markdown Analyzer** — headings, links, tags.
- **Mermaid Analyzer** — trata Mermaid como um Knowledge Object e extrai os relacionamentos expressos no diagrama para alimentar o grafo automaticamente.
- **Terraform / SQL / Java Analyzers** — extraem relacionamentos estruturais a partir de código.

Analyzers rodam dentro do pipeline de ingestão (ver §6). Cada tipo de objeto pode ter um analyzer dedicado; adicionar um enriquece o conhecimento sem alterar o modelo.

## 5. Mapeamento de armazenamento

> **Armazenamento é um detalhe de implementação.** O modelo não depende dele, e o mapeamento abaixo pode mudar sem afetar o Modelo Canônico de Conhecimento.

### 5.1 DynamoDB — objetos de conhecimento e relations

O DynamoDB é o armazenamento principal. Ele guarda **apenas** Knowledge Objects e suas relations — nenhum dado binário.

O schema de chaves concreto e os padrões de acesso serão desenhados durante a implementação. O design precisa suportar pelo menos:

- Buscar um objeto por `id`.
- Buscar os filhos de um objeto em `order`.
- Buscar as coleções às quais um objeto pertence.
- Percorrer relations de saída / de entrada de um objeto.
- Listar objetos por tipo / status / tag.

> **Questão em aberto:** single-table vs. multi-table, e como modelar as arestas de relação (itens de adjacência, GSIs para lookups reversos). Registrado em §9.

### 5.2 Amazon S3 — apenas objetos binários

O S3 armazena **exclusivamente payloads binários**: imagens, vídeos, PDFs, anexos, documentos. O Knowledge Object correspondente (ex.: um `Image` ou `Attachment`) vive no DynamoDB e referencia o objeto no S3; os bytes em si nunca entram no DynamoDB.

### 5.3 Modo de operação: single-user / self-hosted (v1)

A v1 opera em modo **single-user / self-hosted**: uma base de conhecimento por instância, sem isolamento de tenants.

Consequências de design:

- As chaves do DynamoDB **não** precisam de uma dimensão de tenant/workspace na v1.
- A autorização (IAM + nível de API) protege a *instância* inteira, não recortes de conhecimento por usuário.
- Multi-tenancy permanece um não-objetivo (§1). Se for necessário no futuro, a introdução de um escopo de tenant nas chaves e nos eventos é uma evolução conhecida — registrada em §9.

## 6. Pipeline de ingestão

Toda escrita flui pela API. **Não há escrita direta no armazenamento.**

```
Cliente
  ↓
Knowledge API        ← ponto de entrada único (API Gateway + Lambda)
  ↓
Parser               ← transforma a entrada (ex.: Markdown) em objetos candidatos
  ↓
Knowledge Objects    ← objetos canônicos (4 dimensões)
  ↓
Analyzers            ← enriquecem: extraem relations e metadados
  ↓
Knowledge Graph      ← reconcilia objetos + relations
  ↓
Persistência         ← DynamoDB (+ S3 para binários)
  ↓
Eventos              ← publica no EventBridge
```

A API é responsável por: **validação → parsing → enriquecimento → criação de relations → indexação → persistência → publicação de eventos.**

## 7. Eventos

Toda alteração publica eventos de domínio (via EventBridge). Os eventos são o ponto de extensão que permite às projeções ficarem em sincronia sem que o núcleo precise conhecê-las.

Eventos principais:

- `KnowledgeCreated`
- `KnowledgeUpdated`
- `KnowledgeDeleted`
- `RelationCreated`
- `RelationRemoved`

> **Questão em aberto:** o envelope exato do evento (schema, versionamento, chaves de idempotência, garantias de ordenação). Registrado em §9.

## 8. Evolução: projeções

A arquitetura é desenhada para que novas **projeções** — visões otimizadas para leitura, construídas consumindo eventos — possam ser adicionadas **sem alterar o modelo canônico**. Uma projeção é estado derivado; o grafo no DynamoDB continua sendo a fonte da verdade.

Projeções planejadas:

| Projeção | Propósito | Armazenamento provável |
| --- | --- | --- |
| **Search Projection** | Busca full-text / estruturada | ex.: OpenSearch |
| **Vector Projection** | Busca semântica via embeddings | vector store (pós-MVP) |
| **Graph Projection** | Travessia e consultas nativas de grafo | Neo4j / Amazon Neptune (pós-MVP) |
| **Analytics Projection** | Métricas e relatórios | armazenamento analítico |

Cada projeção assina o stream de eventos (§7), constrói sua própria visão e pode ser reconstruída do zero reprocessando os eventos.

## 9. Questões de design em aberto

Registradas aqui para serem resolvidas durante a implementação:

1. **Modelo de dados do DynamoDB** — single- vs. multi-table; como as arestas de relação e os lookups reversos são indexados (§5.1).
2. **Envelope de eventos** — schema, versionamento, idempotência, ordenação (§7).
3. **Esquema de identificador** — ULID vs. UUIDv7 vs. prefixo `ko_` customizado; unicidade global (§2.3).
4. **Concorrência e versionamento** — optimistic locking / semântica de versão do objeto na atualização.
5. **Autorização** — como IAM e a autorização em nível de API protegem a instância single-user (§5.3).
6. **Taxonomia de formatos de content** — o conjunto canônico de valores de `content.format` e como renderers/analyzers se registram para eles.
7. **Integridade referencial** — o que acontece com as relations de entrada quando um objeto é deletado (cascata de `RelationRemoved` vs. dangling proposital).
8. **Multi-tenancy futuro** — como introduzir escopo de tenant nas chaves e nos eventos, caso o projeto evolua para SaaS (§5.3).

## 10. Glossário

- **Modelo Canônico de Conhecimento** — a única representação interna de todo o conhecimento; a fonte da verdade.
- **Knowledge Object** — qualquer elemento da plataforma, compartilhando a forma de quatro dimensões.
- **Knowledge Graph** — a rede de Knowledge Objects conectados por relations.
- **Relation** — um vínculo semântico tipado e direcionado, de primeira classe, entre dois objetos.
- **Renderer** — uma projeção pura e unidirecional dos objetos em uma representação (Markdown, HTML, JSON, MCP).
- **Analyzer** — um componente que enriquece o grafo a partir do content de um objeto.
- **Projeção** — uma visão otimizada para leitura, derivada do consumo de eventos (search, vector, graph, analytics).
- **Collection** — um Knowledge Object usado para agrupar outros objetos; substitui os diretórios físicos.

---

_KnowledgeGraphGuru — modelar conhecimento, não documentos._
