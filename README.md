# KnowledgeGraphGuru

> **A knowledge platform designed for humans and AI agents, built on a canonical knowledge graph. Knowledge is the source of truth.**

Uma plataforma serverless de conhecimento que modela conhecimento como **objetos conectados**, e não como documentos — oferecendo um grafo de conhecimento canônico para humanos e agentes de IA.

---

## Visão

O **KnowledgeGraphGuru** é uma plataforma de gerenciamento de conhecimento orientada a grafos, construída para humanos e agentes de IA.

O objetivo do projeto **não é armazenar documentos** — é representar conhecimento como objetos conectados. Markdown deixa de ser o formato nativo e passa a ser apenas uma das formas de *renderizar* o conhecimento.

- **Missão** — Modelar conhecimento, não documentos.
- **Slogan** — Knowledge is the source of truth.

## O que torna o projeto diferente

Em vez de armazenar arquivos Markdown como entidade principal, o sistema é construído sobre um **Modelo Canônico de Conhecimento** (*Canonical Knowledge Model*), onde cada elemento é um **Knowledge Object** conectado dentro de um grafo.

Markdown, HTML, JSON, MCP e outros formatos são apenas **representações diferentes do mesmo modelo**.

| Ferramentas tradicionais | KnowledgeGraphGuru |
| --- | --- |
| O arquivo é a fonte da verdade | O grafo de conhecimento é a fonte da verdade |
| Markdown é o formato de armazenamento | Markdown é formato de importação/exportação/visualização |
| Pastas são diretórios físicos | Collections são Knowledge Objects |
| Frontmatter vive no arquivo | Properties vivem no objeto |
| Links são texto | Relations são cidadãs de primeira classe |

## Princípios arquiteturais

1. **Knowledge is the source of truth.** — O conhecimento é a fonte da verdade.
2. **Markdown is a rendering format.** — Markdown é um formato de renderização.
3. Todo conhecimento é representado como um **Knowledge Object**.
4. Todo Knowledge Object tem um **identificador imutável**.
5. **Relations são cidadãs de primeira classe.**
6. Armazenamento é um detalhe de implementação.
7. Renderers nunca modificam o Modelo Canônico de Conhecimento.
8. Analyzers enriquecem o conhecimento.

## Conceitos centrais

### Knowledge Object

Todo elemento da plataforma é um Knowledge Object — `Collection`, `Note`, `Heading`, `Paragraph`, `Callout`, `List`, `Table`, `Code Block`, `Mermaid`, `Image`, `Attachment`, `Quote`, entre outros. Todos compartilham um modelo comum, composto por quatro dimensões:

- **Structure** — posição na árvore (`parent`, `children`, `order`)
- **Content** — o conteúdo em si (texto, código, mermaid, lista…)
- **Properties** — metadados (`title`, `tags`, `aliases`, `language`, `status`)
- **Relations** — vínculos semânticos (`references`, `depends_on`, `related_to`, `implements`, `alternative_to`)

### Organização

A estrutura de diretórios deixa de ser física. Ela é expressa por meio de objetos `Collection`, e uma mesma nota pode pertencer a **múltiplas coleções simultaneamente**.

### Renderers

O conhecimento é projetado sob demanda em múltiplos formatos — inicialmente **Markdown**, **HTML**, **JSON** e **MCP**. Novos renderers podem ser adicionados sem alterar o modelo.

### Analyzers

Cada tipo de objeto pode ter um analisador dedicado (Markdown, Mermaid, Terraform, SQL, Java…) que **enriquece o grafo de conhecimento** — por exemplo, extraindo relacionamentos de um diagrama Mermaid para alimentar o grafo automaticamente.

## Arquitetura

Totalmente **serverless na AWS**:

- **API Gateway** — ponto de entrada de todas as leituras e escritas
- **AWS Lambda** — validação, parsing, enriquecimento, persistência
- **DynamoDB** — objetos de conhecimento e seus relacionamentos
- **Amazon S3** — apenas objetos binários (imagens, vídeos, PDFs, anexos)
- **EventBridge** — eventos de domínio
- **CloudWatch** — observabilidade
- **IAM** — controle de acesso

> A primeira versão **não usa banco vetorial nem banco de grafos**. Ambos podem ser adicionados depois como *projeções* opcionais, sem alterar o modelo canônico.

### Pipeline de ingestão

```
Cliente
  ↓
Knowledge API
  ↓
Parser
  ↓
Knowledge Objects
  ↓
Analyzers
  ↓
Knowledge Graph
  ↓
Persistência
  ↓
Eventos
```

Toda escrita **precisa** passar pela API — não há escrita direta no armazenamento. A API é responsável por validação, parsing, enriquecimento, criação de relacionamentos, indexação, persistência e publicação de eventos.

## Para humanos e agentes

A plataforma foi concebida para ser consumida por pessoas *e* por agentes de IA através de:

- **REST API**
- **MCP Server**

## Compatibilidade

O sistema busca compatibilidade com o ecossistema Markdown/Obsidian para **importação e exportação**, mas **não** adota o modelo interno dessas ferramentas. A representação interna é sempre o Modelo Canônico de Conhecimento.

## Roadmap

Projeções futuras que poderão ser adicionadas sem alterar o modelo canônico:

- **Search Projection**
- **Vector Projection** — embeddings como projeção opcional
- **Graph Projection** — Neo4j ou Amazon Neptune
- **Analytics Projection**

## Documentação

- **[DESIGN.md](DESIGN.md)** — o Modelo Canônico de Conhecimento, o mapeamento de dados, o pipeline, os eventos e a estratégia de evolução em profundidade.

## Status

🚧 Fase inicial de design. O modelo e a arquitetura estão sendo definidos antes do início da implementação.

**Modo de operação (v1):** single-user / self-hosted — uma base de conhecimento por instância.

---

_KnowledgeGraphGuru — modelar conhecimento, não documentos._
