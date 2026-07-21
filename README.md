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
| Renomear uma nota quebra os links | Renomear uma nota não quebra nada |

## Princípios arquiteturais

1. **Knowledge is the source of truth.** — O conhecimento é a fonte da verdade.
2. **Markdown is a rendering format.** — Markdown é um formato de renderização.
3. Todo conhecimento é representado como um **Knowledge Object**.
4. Todo Knowledge Object tem um **identificador imutável**.
5. **Relations são cidadãs de primeira classe.**
6. Armazenamento é um detalhe de implementação.
7. Renderers nunca modificam o Modelo Canônico de Conhecimento.
8. Analyzers enriquecem o conhecimento.

## Escopo do MVP

O modelo é ambicioso; o MVP é enxuto e deliberadamente focado.

**A decisão central:** no MVP, a **`Note` é a unidade atômica** de conhecimento. O corpo da nota vive inteiro em `content` — não há decomposição em objetos de bloco (`Paragraph`, `Heading`, `Callout`…) ainda.

Isso preserva integralmente o modelo canônico — as quatro dimensões, a identidade imutável, as relations de primeira classe e as collections são as mesmas — e entrega o diferencial real do produto sem pagar, antes da hora, o custo de construir um *block store*. Descer a granularidade depois é **adicionar objetos filhos**, não redesenhar o modelo.

Callouts, listas, tabelas e blocos de código continuam funcionando: eles viajam como **sintaxe dentro do conteúdo**, sobrevivem ao round-trip e são renderizados normalmente. O que ainda não fazem é existir como objetos endereçáveis no grafo.

### Dentro do MVP

- ✅ `Note` e `Collection` como Knowledge Objects
- ✅ **Relations de primeira classe**, tipadas e por `id` imutável
- ✅ Relations **inline** (ancoradas no texto) e **standalone** (puramente semânticas)
- ✅ Uma nota em múltiplas coleções simultaneamente
- ✅ Vizinhança de **1 salto** no grafo — referências e *backlinks*
- ✅ API REST como único caminho de escrita
- ✅ Renderers **Markdown** e **JSON**
- ✅ Eventos de domínio com log durável para replay

### Fora do MVP

- ❌ Decomposição em blocos finos — callouts, listas e tabelas viajam como sintaxe no conteúdo *(Fase 4)*
- ❌ Objetos binários e Amazon S3 — sem `Attachment` no MVP *(pós-MVP)*
- ❌ Travessia multi-hop de grafo *(Fase 5, via Graph Projection)*
- ❌ MCP Server e analyzers automáticos *(Fase 2)*
- ❌ Busca full-text *(Fase 3)*
- ❌ Banco vetorial e banco de grafos *(Fase 5)*
- ❌ Multi-tenancy — o MVP é single-user / self-hosted

## Conceitos centrais

### Knowledge Object

Todo elemento da plataforma é um Knowledge Object. Todos compartilham um modelo comum, composto por quatro dimensões:

- **Structure** — posição na árvore (`parent`, `children`, `order`)
- **Content** — o conteúdo em si (texto, código, mermaid, lista…)
- **Properties** — metadados (`title`, `tags`, `aliases`, `language`, `status`)
- **Relations** — vínculos semânticos (`references`, `depends_on`, `related_to`, `implements`, `alternative_to`)

O catálogo completo de tipos previstos inclui `Collection`, `Note`, `Heading`, `Paragraph`, `Callout`, `List`, `Table`, `Code Block`, `Mermaid`, `Image`, `Attachment` e `Quote`. **O MVP implementa `Collection` e `Note`** — os demais existem como sintaxe dentro do conteúdo, preservados no round-trip e renderizados, mas ainda não endereçáveis como objetos.

### Relations ancoradas no conteúdo

Um link no meio de um texto é duas coisas ao mesmo tempo: uma **relação semântica** e uma **posição no conteúdo**. O modelo trata as duas sem confundi-las:

> **Content guarda a posição. Relations guardam o significado.**

O conteúdo carrega uma **âncora** — a própria notação `[[…]]`, com o `id` da relation no lugar do título. A relation carrega o alvo e o tipo. Como o texto nunca guarda o título do alvo, renomear uma nota faz todos os links que apontam para ela se re-renderizarem corretamente — sem reescrever uma linha de conteúdo.

```
Import    [[Modelo Canônico]]  →  resolve título → id  →  [[rel_01H8Z]] + relation
Export    [[rel_01H8Z]]        →  lê o título ATUAL do alvo  →  [[Modelo Canônico]]
```

Por que só o link recebe esse tratamento, enquanto callouts e tabelas ficam como sintaxe? **Sintaxe fica literal quando é autocontida; vira objeto quando referencia outra coisa.** Um callout não aponta para nada — nada pode mudar e torná-lo errado. Um link guarda o nome de outro objeto, e nomes mudam. Entre todos os construtos do Markdown, o wikilink é o único que cria dependência entre objetos.

Relations que não têm posição no texto — como uma seta extraída de um diagrama Mermaid — são igualmente cidadãs de primeira classe, apenas sem âncora. Veja [DESIGN.md §3.8](DESIGN.md).

### A plataforma recusa ambiguidade

Um nome de arquivo acumula dois papéis: identidade e endereço. Aqui eles são separados — o **`id`** é imutável e é o que as relations usam; o **`slug`** (título normalizado) é o endereço legível, único, usado para resolver `[[…]]` e nomear o arquivo exportado.

Como uma nota pertence a várias coleções ao mesmo tempo, não existe "a pasta dela" — a unicidade é **global**. E quando dois títulos colidem, o sistema **não escolhe por você**: devolve o conflito com o conteúdo existente e deixa quem escreveu decidir se é a mesma nota ou se os títulos precisam mudar.

Duas notas com o mesmo título não são um problema de nomes — são um sinal de que ou o conhecimento está duplicado, ou o vocabulário está degradando. Resolver isso é trabalho semântico, e é do autor (ou do agente), não da plataforma.

### Organização

A estrutura de diretórios deixa de ser física. Ela é expressa por meio de objetos `Collection`, e uma mesma nota pode pertencer a **múltiplas coleções simultaneamente**.

### Renderers

O conhecimento é projetado sob demanda em múltiplos formatos — **Markdown** e **JSON** no MVP, **HTML** e **MCP** na sequência. Novos renderers podem ser adicionados sem alterar o modelo.

### Analyzers

Cada tipo de objeto pode ter um analisador dedicado (Markdown, Mermaid, Terraform, SQL, Java…) que **enriquece o grafo de conhecimento** — por exemplo, extraindo relacionamentos de um diagrama Mermaid para alimentar o grafo automaticamente. No MVP, as relations são explícitas; a extração automática entra na Fase 2.

## Arquitetura

Totalmente **serverless na AWS**:

- **API Gateway** — ponto de entrada de todas as leituras e escritas
- **AWS Lambda** — validação, parsing, enriquecimento, persistência
- **DynamoDB** — objetos de conhecimento e seus relacionamentos (single-table, *adjacency list*)
- **Amazon S3** — objetos binários (imagens, vídeos, PDFs, anexos) — *pós-MVP*
- **EventBridge** — eventos de domínio, com Archive & Replay
- **CloudWatch** — observabilidade
- **IAM** — controle de acesso

> O MVP **não usa banco vetorial nem banco de grafos**. Ambos podem ser adicionados depois como *projeções* opcionais, sem alterar o modelo canônico.

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

Toda escrita **precisa** passar pela API — nenhuma escrita no modelo de conhecimento ocorre fora dela. A API é responsável por validação, parsing, enriquecimento, criação de relacionamentos, indexação, persistência e publicação de eventos.

**Stack:** TypeScript/Node em Lambda, infraestrutura com AWS CDK, monorepo com um pacote `core` livre de dependências AWS — modelo, parser e renderers testáveis sem nuvem.

## Para humanos e agentes

A plataforma foi concebida para ser consumida por pessoas *e* por agentes de IA através de:

- **REST API** — MVP
- **MCP Server** — Fase 2

## Compatibilidade

O sistema busca compatibilidade com o ecossistema Markdown/Obsidian para **importação e exportação**, mas **não** adota o modelo interno dessas ferramentas. A representação interna é sempre o Modelo Canônico de Conhecimento.

> **Markdown é interchange com perdas, por design.** Importar e exportar preserva o *conhecimento* — conteúdo semântico, properties e relations — mas normaliza a formatação de origem. A compatibilidade é de interoperabilidade, não de fidelidade byte a byte.

## Roadmap

| Fase | Escopo |
| --- | --- |
| **1 — MVP** | `Note` atômica, `Collection`, relations de 1ª classe, API REST, renderers Markdown + JSON, eventos |
| **2 — Agentes** | MCP Server, renderer HTML, Markdown e Mermaid Analyzers |
| **3 — Busca** | Search Projection |
| **4 — Granularidade fina** | Objetos de bloco, se o uso real justificar |
| **5 — Projeções avançadas** | Vector Projection, Graph Projection, Analytics Projection |

A Fase 3 vem antes da Fase 4 de propósito: validar o mecanismo de projeções (barato) antes de investir no *block store* (caro).

## Documentação

- **[DESIGN.md](DESIGN.md)** — a estratégia de MVP, o Modelo Canônico de Conhecimento, o mapeamento de dados, a semântica de escrita, o pipeline, os eventos e a estratégia de evolução em profundidade.

## Status

🚧 Design concluído para a Fase 1. Modelo, arquitetura e decisões técnicas estão registrados; a implementação ainda não começou.

**Modo de operação (MVP):** single-user / self-hosted — uma base de conhecimento por instância.

---

_KnowledgeGraphGuru — modelar conhecimento, não documentos._
