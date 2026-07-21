# KnowledgeGraphGuru — Design

> **Knowledge is the source of truth.**

Este documento descreve o design interno do KnowledgeGraphGuru: o Modelo Canônico de Conhecimento, a estratégia de MVP, como o modelo mapeia para o armazenamento, o pipeline de ingestão, o modelo de eventos e a estratégia de evolução.

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
- **Entregar uma fatia vertical funcional cedo** — o modelo é ambicioso, o MVP é enxuto (§2).

### Não-objetivos (MVP)

- ❌ Sem banco vetorial (embeddings vêm depois, como projeção).
- ❌ Sem banco de grafos (Neo4j/Neptune vêm depois, como projeção).
- ❌ **Sem decomposição em blocos finos** — no MVP a `Note` é a unidade atômica (§2).
- ❌ **Sem travessia profunda de grafo** — o MVP entrega vizinhança de 1 salto (§2.3).
- ❌ Sem escrita direta no armazenamento — tudo passa pela API.
- ❌ Sem tratar Markdown (ou qualquer formato) como formato de armazenamento.
- ❌ Sem adotar o modelo interno do Obsidian — apenas compatibilidade de importação/exportação.
- ❌ Sem multi-tenancy — o MVP é **single-user / self-hosted** (§6.3).

---

## 2. Estratégia de MVP

O Modelo Canônico de Conhecimento (§3) é a visão de longo prazo e não muda. O que o MVP decide é **onde a fronteira do objeto começa** — e essa é a decisão que mais afeta o custo de construção do sistema inteiro.

### 2.1 A decisão central: `Note` como unidade atômica

A visão completa trata cada `Heading`, `Paragraph`, `List` e `Callout` como um Knowledge Object independente, formando uma árvore. Isso é poderoso — e é, na prática, construir um *block store* (o modelo de blocos do Notion). É a parte mais cara do sistema.

**No MVP, a `Note` é a unidade atômica de conhecimento.** O corpo da nota vive inteiro em `content`; não há objetos de bloco abaixo dela.

| | MVP (Fase 1) | Visão completa (Fase 4) |
| --- | --- | --- |
| Menor Knowledge Object | `Note` | `Paragraph`, `Heading`, `Callout`… |
| Itens por nota no armazenamento | 1 (+ arestas) | dezenas |
| Escrita | um item | transação de N itens |
| Renderizar uma nota | ler 1 item | montar a árvore recursivamente |
| Endereçar um parágrafo | ❌ | ✅ |

**Por que isso não trai o modelo:** as quatro dimensões (§3.1), a identidade imutável, as relations de primeira classe e as collections continuam **exatamente as mesmas**. Um `Paragraph` no futuro será um Knowledge Object com a mesma forma de uma `Note`. Descer a granularidade depois é **adicionar objetos filhos**, não redesenhar o modelo.

**O que se ganha:** o diferencial real do produto — grafo de conhecimento com relations tipadas, identidade estável, collections múltiplas, renderers plugáveis e API para agentes — é entregue sem pagar o custo do block store antes de ter usuários.

**O que se perde temporariamente:** endereçar/relacionar um trecho *dentro* de uma nota. Aceitável no MVP; a Fase 4 resolve.

### 2.2 Objetos do MVP

Tipos implementados na Fase 1:

`Collection` · `Note` · `Attachment`

Os demais tipos do catálogo (§3.2) permanecem no modelo como visão, sem implementação no MVP.

### 2.3 Alcance de grafo no MVP

O MVP entrega **vizinhança de 1 salto**:

- relations de saída de um objeto ("o que esta nota referencia");
- relations de entrada — *backlinks* ("quem referencia esta nota");
- pertencimento a collections.

**Não** entrega travessia multi-hop, caminho mais curto ou consultas de grafo. Isso é uma limitação consciente do DynamoDB como armazenamento (§6.1) e é endereçado depois pela **Graph Projection** (§9).

### 2.4 Fases

| Fase | Escopo | Entrega |
| --- | --- | --- |
| **1 — MVP** | `Note` atômica, `Collection`, relations de 1ª classe, API REST, DynamoDB single-table, renderers Markdown + JSON, eventos com log durável | Fatia vertical funcional |
| **2 — Agentes** | MCP Server, renderer HTML, Markdown Analyzer, Mermaid Analyzer | Consumo por agentes de IA |
| **3 — Busca** | Search Projection | Valida o padrão orientado a eventos ponta a ponta |
| **4 — Granularidade fina** | Objetos de bloco (`Heading`, `Paragraph`, `Callout`…) — **somente se o uso real justificar** | Endereçamento sub-nota |
| **5 — Projeções avançadas** | Vector Projection, Graph Projection, Analytics Projection | Busca semântica e travessia nativa |

A Fase 3 antes da Fase 4 é deliberada: validar o mecanismo de projeções (barato) antes de investir no block store (caro).

---

## 3. O Modelo Canônico de Conhecimento

Tudo na plataforma é um **Knowledge Object**. Não existe o conceito de "arquivo" como entidade principal — existe apenas conhecimento.

### 3.1 As quatro dimensões

Todo Knowledge Object compartilha uma forma comum composta por quatro dimensões independentes:

| Dimensão | Propósito | Exemplos |
| --- | --- | --- |
| **Structure** | Posição na árvore | `parent`, `children`, `order` |
| **Content** | O payload do objeto | texto, código, mermaid, itens de lista |
| **Properties** | Metadados específicos do tipo | `title`, `tags`, `aliases`, `language`, `status` |
| **Relations** | Vínculos semânticos com outros objetos | `references`, `depends_on`, `related_to`, `implements`, `alternative_to` |

Separar essas dimensões é o que permite que o mesmo objeto seja renderizado para qualquer formato, enriquecido por analyzers e projetado em outros armazenamentos — sem que o conteúdo e os relacionamentos se misturem.

**Content e Relations não são independentes: eles conversam.** Um link no meio de um texto é, ao mesmo tempo, uma relação semântica *e* uma posição no conteúdo. A regra que governa essa conversa:

> **Content guarda a posição. Relations guardam o significado.**

Nenhum dos dois duplica o outro — eles se ligam por referência, através do mecanismo de âncora (§3.8).

> **No MVP**, `Structure` de uma `Note` é rasa (sem filhos), já que a nota é atômica (§2.1). A dimensão existe e é preenchida — apenas não há sub-objetos ainda.

### 3.2 Tipos de objeto

Catálogo da visão completa:

`Collection` · `Note` · `Heading` · `Paragraph` · `Callout` · `List` · `Table` · `Code Block` · `Mermaid` · `Image` · `Attachment` · `Quote`

**Implementados no MVP:** `Collection`, `Note`, `Attachment` (§2.2).

Novos tipos podem ser introduzidos sem alterar o modelo — um tipo é definido pela sua **forma de content**, pelo seu **analyzer** e por como os **renderers** sabem projetá-lo.

### 3.3 Identidade

- Todo Knowledge Object tem um **identificador imutável** (`id`).
- O `id` nunca muda, mesmo quando content, structure, properties ou relations mudam.
- As relations referenciam objetos por `id`, então mover ou renomear um objeto nunca quebra um vínculo.

Este é o diferencial estrutural frente a Markdown + wikilinks, onde o link é texto e quebra ao renomear.

### 3.4 Exemplo ilustrativo de uma `Note` no MVP

> Apenas ilustrativo — o schema concreto será fixado durante a implementação.

```json
{
  "id": "ko_01H...",              // identificador imutável
  "type": "Note",
  "structure": {
    "parent": null,
    "collections": ["ko_01H...arquitetura", "ko_01H...aws"]
  },
  "content": {
    "format": "ckm/text",
    "value": "Conforme o ⟦rel_01H8Z⟧, relations são cidadãs de primeira classe."
  },
  "properties": {
    "title": "Modelo canônico",
    "tags": ["design", "principios"],
    "status": "draft"
  },
  "relations": [
    { "id": "rel_01H8Z", "type": "references", "target": "ko_01H...other",   "inline": true  },
    { "id": "rel_01H8W", "type": "depends_on", "target": "ko_01H...another", "inline": false }
  ],
  "meta": {
    "createdAt": "2026-07-21T00:00:00Z",
    "updatedAt": "2026-07-21T00:00:00Z",
    "version": 1
  }
}
```

`meta` carrega metadados **técnicos** (auditoria e controle de concorrência), distintos de `properties`, que são metadados **de conhecimento**.

### 3.5 Structure vs. Collections

Duas noções diferentes de "conter" coexistem:

- **Structure** (`parent`/`children`/`order`) — a árvore *intrínseca* de um objeto. No MVP, rasa; a partir da Fase 4, uma `Note` contém `Heading`s e `Paragraph`s.
- **Collections** — agrupamento *extrínseco*. Uma `Collection` é ela mesma um Knowledge Object, e uma nota pode pertencer a **muitas coleções simultaneamente**. Isso substitui a árvore de diretórios físicos, que deixa de existir.

### 3.6 Properties substituem o Frontmatter

O Frontmatter deixa de existir internamente. Tudo que viveria no frontmatter agora vive em **Properties**, no objeto. O renderer Markdown ainda vai *emitir* frontmatter na exportação, puramente para manter compatibilidade com ferramentas externas.

### 3.7 Relations inline vs. standalone

Nem toda relation tem posição no texto. O modelo distingue dois casos:

| | **Inline** | **Standalone** |
| --- | --- | --- |
| Origem | Um link escrito no meio do conteúdo | Declarada via API ou extraída por um analyzer |
| Exemplo | `[[Modelo Canônico]]` dentro de um parágrafo | `depends_on`, ou uma seta de um diagrama Mermaid |
| Posição no content | ✅ ancorada (§3.8) | ❌ nenhuma |
| Campo | `inline: true` | `inline: false` |

Ambas são relations de primeira classe, com o mesmo peso no grafo. A diferença é apenas se existe — ou não — um ponto no conteúdo onde ela se manifesta.

### 3.8 Mecanismo de âncora

Uma relation inline precisa ser renderizada **no lugar certo** do texto. Duas abordagens ingênuas falham:

- **Guardar `[[Título]]` literal no content** — o link volta a ser texto. Renomear o alvo quebra o vínculo, e Markdown volta a ser o formato de armazenamento. Contraria os princípios 2 e 5.
- **Guardar offsets de caractere na relation** — qualquer edição do texto invalida todas as posições seguintes.

> **Granularidade não resolve isso.** Decompor o conteúdo em `Paragraph` não preserva a posição de um link: um parágrafo com dois links continua sendo um único bloco de texto com dois links no meio dele. Âncoras são necessárias em **qualquer** nível de granularidade — e, com elas, a `Note` atômica do MVP não perde nada de posição. O que a granularidade compra é outra coisa: endereçar um bloco (`[[Nota#^bloco]]`), transcluí-lo e dar-lhe metadados próprios.

**Solução adotada:** o content carrega um **token de âncora** que referencia a relation pelo seu `id`. A relation carrega o alvo e o tipo. Nenhum dos dois duplica o outro.

O formato `ckm/text` é uma **string legível**: sintaxe inline de Markdown (`**negrito**`, `` `código` ``) mais âncoras delimitadas por `⟦ ⟧` (U+27E6 / U+27E7) — caracteres que praticamente não ocorrem em texto natural, o que reduz o escaping a um caso de borda.

```json
{
  "content": {
    "format": "ckm/text",
    "value": "Conforme o ⟦rel_01H8Z⟧, relations são cidadãs de primeira classe."
  },
  "relations": [
    { "id": "rel_01H8Z", "type": "references", "target": "ko_01H...", "inline": true }
  ]
}
```

Manter o content como string legível — em vez de um AST de nós inline — é uma escolha deliberada por inspecionabilidade e diffs úteis no git ([ADR-005](DECISIONS.md)). O AST inline fica para a Fase 4, junto com a decomposição em blocos: granularidade vertical e horizontal são a mesma máquina.

Consequências:

- **Relations passam a ter identidade própria** (`id`). Isso não é acessório: um cidadão de primeira classe tem identidade. Antes, relations eram pares anônimos `{type, target}`.
- **O content deixa de ser Markdown válido** e passa a ser conteúdo canônico com tokens — daí `content.format: "ckm/text"`. Isso não é um efeito colateral indesejado: é o princípio *"Markdown is a rendering format"* sendo cumprido literalmente. Se o content fosse Markdown válido, Markdown seria o formato de armazenamento.
- **O texto nunca guarda o título do alvo.** Renomear a nota alvo faz todos os links se re-renderizarem corretamente, sem reescrever nenhum content. É o comportamento que Markdown puro não entrega.
- **Alias é opcional** (`[[Título|texto exibido]]`): quando informado, vira um campo da relation; quando ausente, o renderer resolve o título atual do alvo.
- **Embeds usam o mesmo mecanismo** — `![[imagem]]` é uma âncora cuja relation tem tipo `embeds`.
- **Sobrevive à Fase 4** — quando o conteúdo for decomposto em blocos, a âncora funciona igual dentro de um `Paragraph`.

#### Ciclo de importação e exportação

```
Import    [[Modelo Canônico]]   →  resolve título → id  →  ⟦rel_01H8Z⟧  +  relation
Export    ⟦rel_01H8Z⟧           →  lê o título ATUAL do alvo           →  [[Modelo Canônico]]
```

Links para notas inexistentes (`[[Nota Que Não Existe]]`) criam uma `Note` com `status: "stub"`, mantendo a regra de que toda relation aponta para um objeto real ([ADR-012](DECISIONS.md)).

---

## 4. Renderers

Um **renderer** é uma projeção pura: `Knowledge Object(s) → representação`. Renderers **nunca modificam** o modelo canônico.

| Renderer | Fase | Propósito |
| --- | --- | --- |
| **Markdown** | 1 | Importação/exportação/visualização; re-emite frontmatter |
| **JSON** | 1 | Serialização canônica para máquinas |
| **HTML** | 2 | Visualização para humanos |
| **MCP** | 2 | Representação para agentes |

Como a renderização é unidirecional e sem efeitos colaterais, novos renderers podem ser adicionados a qualquer momento sem risco para o modelo.

### 4.1 Markdown é interchange com perdas — por design

**Importar Markdown e exportar de volta não produz um arquivo byte-idêntico ao original.** O texto é normalizado.

Isso é uma consequência inevitável de ter um modelo canônico: Markdown tem múltiplos dialetos, espaçamento significativo, HTML embutido e variações sintáticas que representam o mesmo conhecimento. O sistema preserva o **conhecimento**, não a formatação de origem.

O que é garantido no round-trip:
- ✅ o conteúdo semântico da nota;
- ✅ properties (via frontmatter na exportação);
- ✅ relations, **inclusive sua posição no texto**, via âncoras (§3.8);
- ✅ links sempre apontando para o alvo correto — mesmo que ele tenha sido renomeado desde a importação.

O que **não** é garantido:
- ❌ espaçamento, quebras de linha e estilo sintático originais;
- ❌ extensões de dialeto não suportadas pelo parser;
- ❌ HTML arbitrário embutido.

A compatibilidade com Obsidian é de **interoperabilidade**, não de fidelidade byte a byte.

---

## 5. Analyzers

Um **analyzer** inspeciona o content de um objeto e **enriquece o grafo de conhecimento** — tipicamente descobrindo relations, extraindo metadados ou derivando novos objetos.

| Analyzer | Fase | O que extrai |
| --- | --- | --- |
| **Markdown Analyzer** | 2 | Tags, títulos, estrutura de headings |
| **Mermaid Analyzer** | 2 | Relacionamentos expressos no diagrama → relations standalone |
| **Terraform / SQL / Java** | pós-MVP | Relacionamentos estruturais a partir de código |

Analyzers rodam dentro do pipeline de ingestão (§7). Cada tipo de objeto pode ter um analyzer dedicado; adicionar um enriquece o conhecimento sem alterar o modelo.

> **Onde entram as relations no MVP.** A resolução de wikilinks (`[[...]]` → âncora + relation, §3.8) **não** é trabalho de analyzer: é responsabilidade do **parser Markdown**, e faz parte da Fase 1, já que importação/exportação Markdown é escopo do MVP. Relations standalone também podem ser informadas explicitamente pelo cliente via API na Fase 1. O que a Fase 2 acrescenta é o *enriquecimento automático* — descobrir relations que ninguém escreveu, como as arestas de um diagrama Mermaid.

---

## 6. Mapeamento de armazenamento

> **Armazenamento é um detalhe de implementação.** O modelo não depende dele, e o mapeamento abaixo pode mudar sem afetar o Modelo Canônico de Conhecimento.

### 6.1 DynamoDB — objetos de conhecimento e relations

O DynamoDB é o armazenamento principal. Ele guarda **apenas** Knowledge Objects e suas relations — nenhum dado binário.

**Padrão adotado: single-table com adjacency list** ([ADR-006](DECISIONS.md)). Objetos e arestas coexistem na mesma tabela; um GSI invertido resolve os lookups reversos (backlinks).

| Item | PK | SK |
| --- | --- | --- |
| Knowledge Object | `KO#<id>` | `META` |
| Aresta (relation) | `KO#<origem>` | `REL#<tipo>#<relId>` |

| Índice | PK | Serve |
| --- | --- | --- |
| **GSI1** (invertido) | `KO#<alvo>` | Backlinks; collections de um objeto |
| **GSI2** | `TYPE#<tipo>` | Listagens por tipo |

**Pertencer a uma collection é uma relation** (`in_collection`), não um mecanismo separado — um único mecanismo de arestas resolve relations, backlinks e coleções.

Access patterns que o MVP precisa suportar:

| # | Padrão | Como |
| --- | --- | --- |
| 1 | Buscar um objeto por `id` | `GetItem` em `KO#<id>` / `META` |
| 2 | Listar relations de saída de um objeto | `Query` em `KO#<id>`, prefixo `REL#` |
| 3 | Listar relations de entrada (backlinks) | `Query` no GSI1 |
| 4 | Listar os membros de uma collection | `Query` no GSI1, tipo `in_collection` |
| 5 | Listar as collections de um objeto | `Query` em `KO#<id>`, prefixo `REL#in_collection#` |
| 6 | Listar objetos por tipo | `Query` no GSI2 |

**Consistência:** as relations de saída (padrões 2 e 5) são fortemente consistentes, pois vivem na partição do próprio objeto. Já os **backlinks (padrões 3 e 4) são eventualmente consistentes por contrato** — GSIs do DynamoDB são atualizados de forma assíncrona, e a API não promete que uma relation recém-criada apareça imediatamente no alvo ([ADR-017](DECISIONS.md)).

**Limitação assumida:** o DynamoDB resolve bem **um salto**. Travessia multi-hop exige N queries sequenciais e não é objetivo do MVP (§2.3) — a Graph Projection (§9) endereça isso quando houver necessidade real.

**Limite operacional:** `TransactWriteItems` suporta no máximo 100 itens. Com a `Note` atômica do MVP, uma escrita típica é de poucos itens (a nota + suas arestas), bem dentro do limite. Este limite é uma das razões concretas para adiar a decomposição em blocos (§2.1).

### 6.2 Amazon S3 — apenas objetos binários

O S3 armazena **exclusivamente payloads binários**: imagens, vídeos, PDFs, anexos, documentos. O Knowledge Object correspondente (ex.: um `Attachment`) vive no DynamoDB e referencia o objeto no S3; os bytes em si nunca entram no DynamoDB.

### 6.3 Modo de operação: single-user / self-hosted (MVP)

O MVP opera em modo **single-user / self-hosted**: uma base de conhecimento por instância, sem isolamento de tenants.

Consequências de design:

- As chaves do DynamoDB **não** precisam de uma dimensão de tenant/workspace.
- A autorização (IAM + nível de API) protege a *instância* inteira, não recortes de conhecimento por usuário.
- Escritas concorrentes são raras por definição, o que reduz muito a pressão sobre o controle de concorrência (§6.4).
- Multi-tenancy permanece um não-objetivo (§1). Se necessário no futuro, introduzir escopo de tenant nas chaves e nos eventos é uma evolução conhecida — registrada em §11.

### 6.4 Concorrência

Controle **otimista por objeto**: `meta.version` é incrementado a cada escrita e validado via condição no `PutItem`/`UpdateItem`. Uma escrita sobre versão desatualizada falha e o cliente reconcilia.

Com a `Note` atômica, o objeto é a unidade de conflito — não há necessidade de consistência em nível de árvore. Esse problema só aparece na Fase 4.

---

## 7. Pipeline de ingestão

Toda escrita flui pela API: **nenhuma escrita no modelo de conhecimento ocorre fora dela.**

> A única exceção é o *payload binário* de anexos, que sobe direto para o S3 via presigned URL — os bytes não são conhecimento; o conhecimento é o objeto `Attachment`, e esse continua sendo criado exclusivamente pela API ([ADR-016](DECISIONS.md)).

```
Cliente
  ↓
Knowledge API        ← ponto de entrada único (API Gateway + Lambda)
  ↓
Parser               ← transforma a entrada (ex.: Markdown) em objetos candidatos
  ↓
Knowledge Objects    ← objetos canônicos (4 dimensões)
  ↓
Analyzers            ← enriquecem: extraem relations e metadados   [Fase 2]
  ↓
Knowledge Graph      ← reconcilia objetos + relations
  ↓
Persistência         ← DynamoDB (+ S3 para binários)
  ↓
Eventos              ← publica no EventBridge
```

A API é responsável por: **validação → parsing → enriquecimento → criação de relations → indexação → persistência → publicação de eventos.**

No MVP, o estágio *Analyzers* é um passo vazio (pass-through) — o pipeline já existe e o slot está reservado, mas o enriquecimento automático entra na Fase 2.

---

## 8. Eventos

Toda alteração publica eventos de domínio via EventBridge. Os eventos são o ponto de extensão que permite às projeções ficarem em sincronia sem que o núcleo precise conhecê-las.

Eventos principais:

- `KnowledgeCreated`
- `KnowledgeUpdated`
- `KnowledgeDeleted`
- `RelationCreated`
- `RelationRemoved`

### 8.1 Log durável e replay

EventBridge é um **barramento**, não um log durável — por padrão, eventos publicados não ficam disponíveis para reprocessamento. Como a promessa de reconstruir projeções do zero (§9) depende disso, o MVP habilita **EventBridge Archive & Replay** desde o início.

Consequência: qualquer projeção pode ser criada ou reconstruída depois, reprocessando o arquivo — sem que o núcleo precise ser alterado.

> A fonte da verdade continua sendo o grafo no DynamoDB. O arquivo de eventos é o mecanismo de *reconstrução de projeções*, não um event store canônico.

### 8.2 Envelope

Todo evento carrega, no mínimo: `eventId`, `eventType`, `eventVersion`, `occurredAt`, `objectId` e o payload relevante. `eventId` serve como chave de idempotência para consumidores.

**Ordenação:** o EventBridge não garante ordem. As projeções devem ser tolerantes a reordenação — na prática, aplicando eventos de forma idempotente e usando `meta.version` do objeto para descartar atualizações obsoletas.

---

## 9. Evolução: projeções

A arquitetura é desenhada para que novas **projeções** — visões otimizadas para leitura, construídas consumindo eventos — possam ser adicionadas **sem alterar o modelo canônico**. Uma projeção é estado derivado; o grafo no DynamoDB continua sendo a fonte da verdade.

| Projeção | Fase | Propósito | Armazenamento provável |
| --- | --- | --- | --- |
| **Search Projection** | 3 | Busca full-text / estruturada | ex.: OpenSearch |
| **Vector Projection** | 5 | Busca semântica via embeddings | vector store |
| **Graph Projection** | 5 | Travessia multi-hop e consultas nativas de grafo | Neo4j / Amazon Neptune |
| **Analytics Projection** | 5 | Métricas e relatórios | armazenamento analítico |

Cada projeção assina o stream de eventos (§8), constrói sua própria visão e pode ser reconstruída do zero via replay do arquivo (§8.1).

A **Search Projection** é deliberadamente a primeira: ela valida o mecanismo de projeções ponta a ponta com custo baixo, antes de qualquer investimento maior.

---

## 10. Stack e organização do código

| Camada | Decisão |
| --- | --- |
| Runtime | TypeScript / Node.js 22 em Lambda ARM64 ([ADR-001](DECISIONS.md)) |
| Infraestrutura como código | AWS CDK em TypeScript ([ADR-002](DECISIONS.md)) |
| Testes | Unit no `core` · integração com DynamoDB Local · E2E em conta sandbox ([ADR-004](DECISIONS.md)) |

Monorepo com três pacotes ([ADR-003](DECISIONS.md)):

| Pacote | Responsabilidade | Depende de AWS? |
| --- | --- | --- |
| `core` | Modelo canônico, parser, renderers, resolução de âncoras, regras de negócio | ❌ **não** |
| `api` | Handlers Lambda, repositórios, publicação de eventos | ✅ |
| `infra` | Stacks CDK | ✅ |

O `core` livre de AWS é a materialização do princípio *storage is an implementation detail*: persistência e publicação de eventos entram como interfaces, implementadas em `api`. A complexidade real do sistema — parsing, âncoras, integridade do grafo — fica testável em milissegundos, sem container nem nuvem.

---

## 11. Questões de design em aberto

As decisões já tomadas estão registradas em **[DECISIONS.md](DECISIONS.md)**. Permanecem em aberto:

1. **Taxonomia de formatos de content** — o conjunto canônico de valores de `content.format` além de `ckm/text`, e como renderers/analyzers se registram para eles (relevante a partir da Fase 2).
2. **Critério para a Fase 4** — que evidência de uso justifica descer para blocos finos e para o AST inline (§2.1).
3. **Multi-tenancy futuro** — como introduzir escopo de tenant nas chaves e nos eventos, caso o projeto evolua para SaaS (§6.3).
4. **Reconciliação de anexos** — o que fazer com um `Attachment` cujo upload via presigned URL nunca se completou ([ADR-016](DECISIONS.md)).
5. **Escape de âncoras** — regra para o caso de borda em que `⟦ ⟧` aparece literalmente no conteúdo do usuário ([ADR-005](DECISIONS.md)).

---

## 12. Glossário

- **Modelo Canônico de Conhecimento** — a única representação interna de todo o conhecimento; a fonte da verdade.
- **Knowledge Object** — qualquer elemento da plataforma, compartilhando a forma de quatro dimensões.
- **Knowledge Graph** — a rede de Knowledge Objects conectados por relations.
- **Relation** — um vínculo semântico tipado, direcionado e **identificado**, de primeira classe, entre dois objetos.
- **Relation inline** — relation que se manifesta em um ponto específico do conteúdo, marcada por uma âncora (§3.7).
- **Relation standalone** — relation puramente semântica, sem posição no conteúdo (§3.7).
- **Âncora** — token no `content` que marca a posição de uma relation inline, referenciando-a por `id` (§3.8).
- **Renderer** — uma projeção pura e unidirecional dos objetos em uma representação (Markdown, HTML, JSON, MCP).
- **Analyzer** — um componente que enriquece o grafo a partir do content de um objeto.
- **Projeção** — uma visão otimizada para leitura, derivada do consumo de eventos (search, vector, graph, analytics).
- **Collection** — um Knowledge Object usado para agrupar outros objetos; substitui os diretórios físicos.
- **Unidade atômica** — o menor Knowledge Object que o sistema endereça. No MVP, a `Note` (§2.1).

---

_KnowledgeGraphGuru — modelar conhecimento, não documentos._
