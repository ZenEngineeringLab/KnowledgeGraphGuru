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

`Collection` · `Note`

Os demais tipos do catálogo (§3.2) permanecem no modelo como visão, sem implementação no MVP.

**Elementos de bloco viajam como sintaxe.** Callouts, listas, tabelas, headings e blocos de código não são objetos no MVP — vivem dentro do `content` da nota, em sintaxe (§3.8). São preservados no round-trip e renderizados em HTML; apenas não são endereçáveis nem consultáveis como objetos próprios.

**Sem objetos binários.** `Attachment` fica fora da Fase 1, e com ele saem o Amazon S3, o upload por presigned URL e todo o estado intermediário de um binário que ainda não chegou. O MVP é **DynamoDB + Lambda + API Gateway + EventBridge**, sem armazenamento de blobs.

### 2.3 Alcance de grafo no MVP

O MVP entrega **vizinhança de 1 salto**:

- relations de saída de um objeto ("o que esta nota referencia");
- relations de entrada — *backlinks* ("quem referencia esta nota");
- pertencimento a collections.

**Não** entrega travessia multi-hop, caminho mais curto ou consultas de grafo. Isso é uma limitação consciente do DynamoDB como armazenamento (§6.1) e é endereçado depois pela **Graph Projection** (§10).

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

**Implementados no MVP:** `Collection` e `Note` (§2.2). Os demais existem como sintaxe dentro do content, não como objetos.

Novos tipos podem ser introduzidos sem alterar o modelo — um tipo é definido pela sua **forma de content**, pelo seu **analyzer** e por como os **renderers** sabem projetá-lo.

### 3.3 Identidade e endereço

Duas coisas que um nome de arquivo acumulava — e que aqui são separadas:

| | `id` | `slug` |
| --- | --- | --- |
| Papel | **Identidade** — qual objeto é este | **Endereço** — como um humano se refere a ele |
| Formato | ULID com prefixo (`ko_01H8Z…`) | Título normalizado (`modelo-canonico`) |
| Muda? | ❌ nunca | ✅ acompanha o título |
| Usado por | Todas as relations | Resolução de `[[…]]`, nome de arquivo na exportação |
| Único? | Por construção | **Por restrição** (§6.1) |

**O `id` é imutável.** Nunca muda, mesmo quando content, structure, properties ou relations mudam. Como as relations referenciam objetos por `id`, mover ou renomear um objeto nunca quebra um vínculo — o diferencial estrutural frente a Markdown + wikilinks, onde o link é texto e quebra ao renomear.

**O `slug` é a chave natural.** É a forma normalizada do título (minúsculas, sem acentos, espaços viram hífen), derivada automaticamente — nunca digitada. Ele existe porque o `id` não resolve tudo que o nome de arquivo resolvia:

- impedir que duas notas representem a mesma coisa sem que ninguém perceba;
- resolver `[[Modelo Canônico]]` para **exatamente um** objeto na importação — sem isso, o mecanismo de âncora (§3.8) é ambíguo e a importação deixa de ser determinística;
- nomear o arquivo na exportação (`modelo-canonico.md`).

**A unicidade é global**, não por coleção. Isso é imposto pelo modelo: uma nota pertence a N collections simultaneamente (§3.5), então não existe "a pasta dela" para servir de escopo. É o preço direto de coleções extrínsecas.

Renomear muda o slug e não quebra nada, porque as relations usam `id`. O slug antigo passa a constar em `properties.aliases`, que compartilha o mesmo namespace de resolução — links escritos com o nome antigo continuam resolvendo.

#### Normalização

O título preserva a grafia original, com acentuação e pontuação. O slug não: ele é restrito a `[a-z0-9-]`.

```
1. Decompõe (NFD) e remove os diacríticos      Canônico → Canonico
2. Minúsculas                                   Canonico → canonico
3. Tudo que não for [a-z0-9] vira separador     "AWS: decisões!" → "aws  decisoes "
4. Sequências de separador viram um "-"         → "aws-decisoes"
5. Remove "-" das pontas
6. Trunca em 100 caracteres, na borda do "-"
```

| Título | Slug |
| --- | --- |
| `Modelo Canônico` | `modelo-canonico` |
| `Arquitetura AWS: decisões` | `arquitetura-aws-decisoes` |
| `Análise 2026 — Fase 1` | `analise-2026-fase-1` |
| `O que é um "Knowledge Object"?` | `o-que-e-um-knowledge-object` |
| `🚀 Deploy` | `deploy` |

**Títulos que normalizam para vazio são rejeitados com `400`.** Isso alcança dois casos: títulos compostos apenas de símbolos (`###`, `🚀`) e títulos em escrita não-latina (`設計ノート`), que a regra `[a-z0-9]` esvazia por completo.

A consequência assumida é que **títulos precisam usar alfabeto latino**. A alternativa — cair para o `id` como slug — produziria endereços ilegíveis e arquivos exportados sem sentido, esvaziando o propósito do slug. Aceitar letras Unicode resolveria o caso internacional ao custo de nomes de arquivo e URLs menos portáveis; é uma abertura retrocompatível, se o uso vier a pedir.

> **A normalização torna a detecção de colisão deliberadamente tolerante.** `Modelo Canônico`, `modelo canonico` e `Modelo, Canônico!` produzem o mesmo slug e colidem entre si (§8.1). Ela não pega apenas duplicata exata — pega **quase-duplicata**, que é justamente o caso em que alguém não percebeu que o conhecimento já existia.

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
    "value": "Conforme o [[rel_01H8Z]], relations são cidadãs de primeira classe."
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

#### Vocabulário

O conjunto de tipos é **fechado e validado** no MVP:

`references` · `depends_on` · `related_to` · `implements` · `alternative_to` · `embeds` · `in_collection`

**Alternativa rejeitada:** vocabulário aberto, definido pelo usuário. Degrada rapidamente em sinônimos (`refers_to`, `ref`, `references`) que inviabilizam consultas consistentes no grafo. Abrir o conjunto depois é retrocompatível; fechá-lo depois, não.

Note que `in_collection` é uma relation como qualquer outra (§6.1) — pertencer a uma coleção não é um mecanismo à parte.

### 3.8 Mecanismo de âncora

#### Por que o link é diferente de todo o resto

Callouts, listas, tabelas e blocos de código ficam como sintaxe dentro do content (§2.2), mas o link não. A regra que explica os dois:

> **Sintaxe fica literal quando é autocontida. Vira objeto quando referencia outra coisa.**

Um callout — `> [!warning] Cuidado com throttling` — não aponta para lugar nenhum. Nada no acervo pode mudar e tornar aquele texto errado; guardá-lo verbatim é seguro para sempre.

Um link guarda o **título** de outro objeto. E títulos mudam. No instante em que alguém renomeia o alvo, todo `[[Título Antigo]]` espalhado pelo acervo fica errado — restando quebrar o link, ou reescrever todas as notas que citam. Este último é literalmente o que o Obsidian faz ao renomear um arquivo.

Entre todos os construtos do Markdown, **o wikilink é o único que cria dependência entre objetos**. A exceção é de exatamente um caso — e é o caso sobre o qual o produto inteiro se sustenta.

#### Abordagens que falham

- **Guardar `[[Título]]` literal** — o link volta a ser texto. Renomear quebra o vínculo, e Markdown volta a ser o formato de armazenamento. Contraria os princípios 2 e 5.
- **Guardar offsets de caractere na relation** — qualquer edição do texto invalida todas as posições seguintes.

> **Granularidade não resolve isso.** Decompor o conteúdo em `Paragraph` não preserva a posição de um link: um parágrafo com dois links continua sendo um único bloco de texto com dois links no meio dele. Âncoras são necessárias em **qualquer** nível de granularidade — e, com elas, a `Note` atômica do MVP não perde nada de posição. O que a granularidade compra é outra coisa: endereçar um bloco, transcluí-lo e dar-lhe metadados próprios.

#### Solução adotada

O content carrega uma **âncora** que referencia a relation pelo seu `id`. A relation carrega o alvo e o tipo. Nenhum dos dois duplica o outro.

A âncora usa a própria notação de wikilink, `[[ ]]`, com o `id` da relation no lugar do título:

```json
{
  "content": {
    "format": "ckm/text",
    "value": "Conforme o [[rel_01H8Z]], relations são cidadãs de primeira classe."
  },
  "relations": [
    { "id": "rel_01H8Z", "type": "references", "target": "ko_01H...other",   "inline": true  },
    { "id": "rel_01H8W", "type": "depends_on", "target": "ko_01H...another", "inline": false }
  ]
}
```

A cadeia de resolução na renderização:

```
content: [[rel_01H8Z]]
   ↓ procura em relations
relation: target = ko_01H...other
   ↓ busca o objeto
alvo: properties.title = "Modelo Canônico"
   ↓
saída: [[Modelo Canônico]]
```

Repare no contraste: `rel_01H8Z` é `inline: true` e seu `id` aparece no texto; `rel_01H8W` é `inline: false` e não aparece em lugar nenhum — existe no grafo, mas não se manifesta na prosa. O `content` nunca guarda o `target` nem o título, só o ponteiro para a relation. É isso que faz renomear o alvo não tocar em nenhuma nota.

> O campo `inline` é tecnicamente derivável varrendo o content, mas é mantido explícito para não exigir parsing de texto ao responder uma pergunta estrutural. Ele também serve de invariante na validação: `inline: true` exige âncora presente; `false` exige âncora ausente.

O formato `ckm/text` é, portanto, uma **string legível** — sintaxe de Markdown mais âncoras. A sintaxe cobre tanto o nível inline (`**negrito**`, `` `código` ``) quanto o de bloco: callouts, listas, tabelas, headings, blocos de código. Enquanto a `Note` for atômica, é aqui que esses elementos vivem (§2.2).

#### Ciclo de importação e exportação

```
Import    [[Modelo Canônico]]  →  resolve título → id  →  [[rel_01H8Z]]  +  relation
Export    [[rel_01H8Z]]        →  lê o título ATUAL do alvo  →  [[Modelo Canônico]]
```

Links para notas inexistentes (`[[Nota Que Não Existe]]`) criam uma `Note` com `status: "stub"`, mantendo a regra de que toda relation aponta para um objeto real (§8.2).

Consequências do mecanismo:

- **Relations passam a ter identidade própria** (`id`). Isso não é acessório: um cidadão de primeira classe tem identidade. Antes, relations eram pares anônimos `{type, target}`.
- **O content não é Markdown consumível diretamente.** A notação é a mesma, mas o miolo dos colchetes é um `id` — só o renderer sabe transformá-lo em texto legível. Isso não é efeito colateral indesejado: é o princípio *"Markdown is a rendering format"* sendo cumprido literalmente. Se o content fosse Markdown pronto para uso, Markdown seria o formato de armazenamento.
- **O texto nunca guarda o título do alvo.** Renomear a nota alvo faz todos os links se re-renderizarem corretamente, sem reescrever nenhum content — comportamento que Markdown puro não entrega.
- **Embeds usam o mesmo mecanismo** — `![[rel_id]]` é uma âncora cuja relation tem tipo `embeds`.
- **Sobrevive à Fase 4** — quando o conteúdo for decomposto em blocos, a âncora funciona igual dentro de um `Paragraph`.

#### Alias

O separador `|` significa **alias autoral**, sempre — nunca um cache do título. Ele vive no próprio content, não como campo da relation: é texto de apresentação, e apresentação é responsabilidade do conteúdo.

| Forma | Renderiza |
| --- | --- |
| `[[rel_01H8Z]]` | O **título atual** do alvo, resolvido na hora |
| `[[rel_01H8Z\|meu texto]]` | `meu texto`, literal — o autor escolheu assim |

Isso preserva a semântica do Obsidian: quem escreveu `[[Modelo Canônico|o modelo]]` continua vendo *o modelo*. E elimina qualquer risco de defasagem — não existe título armazenado para envelhecer, porque quando há texto ali ele é fixo por intenção.

*(Não confundir com `properties.aliases` do objeto alvo, que são nomes alternativos para **resolução** — mecanismo distinto, §3.3.)*

#### Escaping

Como `[[ ]]` é notação comum, escrever colchetes duplos literais é um caso concreto — documentar o próprio sistema já exige isso. Valem as convenções que qualquer parser Markdown implementa:

- dentro de crase — `` `[[texto]]` `` — nada é interpretado;
- barra invertida — `\[[texto]]` — para o restante.

**Alternativa rejeitada:** delimitadores exóticos como `⟦ ⟧`, que praticamente não ocorrem em texto natural e reduziriam o escaping a um caso de borda. Perde-se mais do que se ganha: a notação familiar mantém o content parecido com Markdown e torna o round-trip quase idêntico — só o miolo dos colchetes muda.

**Alternativa rejeitada:** representar o content como um AST de nós inline (`[{"text":…},{"ref":…}]`). Elimina o escaping e é conceitualmente mais puro, mas sacrifica a legibilidade do conteúdo e a utilidade dos diffs no git — que são requisitos deste projeto. O AST inline fica para a Fase 4, junto com a decomposição em blocos: granularidade vertical e horizontal são a mesma máquina, e serão construídas de uma vez.

#### Validação por rota

Como a API canônica aceita content pronto, um cliente pode enviar `[[…]]` com qualquer coisa dentro:

| Rota | Regra |
| --- | --- |
| `POST /notes` | Todo `[[…]]` deve referenciar uma relation presente no payload. Título solto é erro |
| `POST /import/markdown` | Títulos são aceitos e resolvidos para âncoras — é o papel do adaptador |

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

**Padrão adotado: single-table com adjacency list**. Objetos e arestas coexistem na mesma tabela; um GSI invertido resolve os lookups reversos (backlinks).

| Item | PK | SK | Papel |
| --- | --- | --- | --- |
| Knowledge Object | `KO#<id>` | `META` | O objeto |
| Aresta (relation) | `KO#<origem>` | `REL#<tipo>#<relId>` | Vínculo |
| Lock de slug | `SLUG#<slug>` | `SLUG` | Unicidade + resolução de nome |

O **lock de slug** implementa a restrição de unicidade (§3.3), que o DynamoDB não oferece nativamente para atributos fora da chave: o item é escrito na mesma `TransactWriteItems` do objeto, sob condição `attribute_not_exists(PK)`. Se o slug já existe, a transação inteira falha — não há janela para duplicata.

Ele também entrega, de graça, o `GetItem` que a importação usa para resolver `[[Título]]` → `id`. Aliases (§3.3) ocupam o mesmo namespace: cada alias é um lock apontando para o mesmo objeto.

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

**Consistência:** as relations de saída (padrões 2 e 5) são fortemente consistentes, pois vivem na partição do próprio objeto. Já os **backlinks (padrões 3 e 4) são eventualmente consistentes por contrato** — GSIs do DynamoDB são atualizados de forma assíncrona, e a API não promete que uma relation recém-criada apareça imediatamente no alvo.

**Limitação assumida:** o DynamoDB resolve bem **um salto**. Travessia multi-hop exige N queries sequenciais e não é objetivo do MVP (§2.3) — a Graph Projection (§10) endereça isso quando houver necessidade real.

**Limite operacional:** `TransactWriteItems` suporta no máximo 100 itens. Com a `Note` atômica do MVP, uma escrita típica é de poucos itens (a nota + suas arestas), bem dentro do limite. Este limite é uma das razões concretas para adiar a decomposição em blocos (§2.1).

### 6.2 Amazon S3 — objetos binários (pós-MVP)

Quando `Attachment` entrar no modelo, o S3 armazenará **exclusivamente payloads binários**: imagens, vídeos, PDFs, anexos. O Knowledge Object correspondente vive no DynamoDB e referencia o objeto no S3; os bytes em si nunca entram no DynamoDB.

**Fora do escopo da Fase 1** (§2.2) — o MVP não tem armazenamento de blobs.

### 6.3 Modo de operação: single-user / self-hosted (MVP)

O MVP opera em modo **single-user / self-hosted**: uma base de conhecimento por instância, sem isolamento de tenants.

Consequências de design:

- As chaves do DynamoDB **não** precisam de uma dimensão de tenant/workspace.
- A autorização (IAM + nível de API) protege a *instância* inteira, não recortes de conhecimento por usuário.
- Escritas concorrentes são raras por definição, o que reduz muito a pressão sobre o controle de concorrência (§8.4).
- Multi-tenancy permanece um não-objetivo (§1). Se necessário no futuro, introduzir escopo de tenant nas chaves e nos eventos é uma evolução conhecida, mas não prevista.

---

## 7. Pipeline de ingestão

Toda escrita flui pela API: **nenhuma escrita no modelo de conhecimento ocorre fora dela.** No MVP a regra vale sem exceção — não há armazenamento binário (§2.2).

> Quando `Attachment` entrar, o *payload binário* subirá direto para o S3 via presigned URL. Isso não abre exceção: os bytes não são conhecimento, e o objeto que os representa continuará sendo criado exclusivamente pela API.

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
Persistência         ← DynamoDB
  ↓
Eventos              ← publica no EventBridge
```

A API é responsável por: **validação → parsing → enriquecimento → criação de relations → indexação → persistência → publicação de eventos.**

No MVP, o estágio *Analyzers* é um passo vazio (pass-through) — o pipeline já existe e o slot está reservado, mas o enriquecimento automático entra na Fase 2.

---

## 8. Semântica de escrita e ciclo de vida

As regras que governam o que acontece ao criar, importar, atualizar e remover conhecimento. Todas existem para sustentar uma única invariante:

> **Toda relation aponta para um objeto que existe, e toda âncora aponta para uma relation que existe.**

### 8.1 Colisão de slug: a plataforma recusa, não resolve

Duas notas com o mesmo título não são um problema técnico de nomes — são um **sinal semântico**. Ou representam o mesmo conhecimento e deveriam ser uma só, ou os títulos estão ruins e o vocabulário do acervo está degradando. Nos dois casos, quem sabe a resposta é o autor.

> **A plataforma recusa ambiguidade. Ela não a resolve.**

Sufixar `-2` silenciosamente é o que um sistema de *arquivos* faz. Um sistema que modela *conhecimento* recusa a escrita e devolve o contexto necessário para decidir.

**Por que a plataforma não faz merge.** Do outro lado da API há um agente ou uma pessoa. Reconciliar dois textos é trabalho semântico — exatamente o que o cliente sabe fazer e a plataforma não. Não existe algoritmo de merge aqui, nem política de "substitui ou anexa".

O papel da plataforma se reduz a três: **detectar**, **recusar**, **informar**.

#### Contrato

Uma escrita cujo slug já existe recebe `409` com tudo que o cliente precisa para decidir em uma única ida:

```json
{
  "error": "slug_conflict",
  "slug": "modelo-canonico",
  "existing": {
    "id": "ko_01H8Z...",
    "title": "Modelo Canônico",
    "version": 7,
    "content": { "format": "ckm/text", "value": "..." }
  }
}
```

O `content` vem inteiro porque o cliente faria esse `GET` em todos os casos. O `version` é o que ele devolve na atualização, para o controle otimista (§8.4) — se outra escrita entrar no meio, a operação falha limpa em vez de sobrescrever.

#### Resolução

Não há endpoint de resolução. A decisão vira uma escrita comum:

| Decisão do cliente | O que ele faz |
| --- | --- |
| "É a mesma nota" | `PUT` no objeto existente, com o texto que ele mesmo reconciliou e a `version` recebida |
| "São notas diferentes" | Nova escrita, com outro título — o dele ou o da nota já existente |

Todo o mecanismo custa **um código de status e um payload bem desenhado**. Nenhuma rota nova.

> **Política.** No MVP existe uma só: recusar e exigir resolução explícita. O comportamento fica atrás de uma *policy* para que variantes futuras — auto-merge na reimportação de um acervo conhecido, por exemplo — tenham onde ser plugadas sem redesenhar o fluxo.

**Limitação conhecida:** resolver conflito a conflito não escala para importar um acervo inteiro. Um vault com centenas de arquivos exigirá uma *sessão de importação* que acumula as pendências e as apresenta em lote — fora do escopo da Fase 1, cujo alvo é a importação de documento único.

### 8.2 Criação e importação

**Links não resolvidos criam notas stub.** Importar `[[Nota Que Não Existe]]` cria uma `Note` com `properties.status = "stub"` e estabelece a relation normalmente.

Isso preserva o fluxo de escrita do ecossistema Markdown — citar algo antes de escrevê-lo — sem abrir exceção na invariante. A nota stub passa a ser um item de trabalho visível no próprio grafo, e escrever nela depois é apenas atualizá-la: o `id` já existe, nenhum vínculo precisa ser refeito.

**Escrever numa stub não é colisão.** Tecnicamente o slug já está ocupado, mas uma stub não tem conteúdo — não há nada a reconciliar. Preencher a lacuna é o comportamento pretendido, então essa escrita não devolve `409` (§8.1).

**Cada ocorrência é uma relation.** Citar a mesma nota duas vezes no mesmo texto gera duas relations, com `id` e âncoras distintos. Cada ocorrência é uma asserção própria, em uma posição própria; modelar como uma relation com múltiplas âncoras complicaria escrita e remoção sem benefício. A deduplicação fica a cargo das consultas — "quais notas esta referencia" agrupa por alvo.

### 8.3 Remoção

**Deletar um objeto cascateia.** As relations que **apontam para ele** também são removidas, com um `RelationRemoved` emitido para cada uma. Um grafo com alvos inexistentes contaminaria backlinks, renderização e toda projeção futura.

Consequência operacional: o custo do delete é proporcional ao número de backlinks do objeto.

**Remover uma relation inline remove sua âncora.** As duas operações ocorrem na **mesma transação** — uma âncora apontando para uma relation inexistente é um estado inválido que o renderer teria de adivinhar como tratar.

Como isso altera o `content` do objeto de origem, a remoção incrementa a versão dele e emite `KnowledgeUpdated`. O texto ao redor do link é preservado; some apenas o vínculo.

### 8.4 Concorrência

Controle **otimista por objeto**: `meta.version` é incrementado a cada escrita e validado por condição no `PutItem`/`UpdateItem`. Uma escrita sobre versão desatualizada falha, e o cliente reconcilia.

Com a `Note` atômica, o objeto é a unidade de conflito — não há consistência em nível de árvore a manter. Esse problema só aparece na Fase 4.

---

## 9. Eventos

Toda alteração publica eventos de domínio via EventBridge. Os eventos são o ponto de extensão que permite às projeções ficarem em sincronia sem que o núcleo precise conhecê-las.

Eventos principais:

- `KnowledgeCreated`
- `KnowledgeUpdated`
- `KnowledgeDeleted`
- `RelationCreated`
- `RelationRemoved`

### 9.1 Log durável e replay

EventBridge é um **barramento**, não um log durável — por padrão, eventos publicados não ficam disponíveis para reprocessamento. Como a promessa de reconstruir projeções do zero (§10) depende disso, o MVP habilita **EventBridge Archive & Replay** desde o início.

Consequência: qualquer projeção pode ser criada ou reconstruída depois, reprocessando o arquivo — sem que o núcleo precise ser alterado.

> A fonte da verdade continua sendo o grafo no DynamoDB. O arquivo de eventos é o mecanismo de *reconstrução de projeções*, não um event store canônico.

### 9.2 Envelope

Todo evento carrega, no mínimo: `eventId`, `eventType`, `eventVersion`, `occurredAt`, `objectId` e o payload relevante. `eventId` serve como chave de idempotência para consumidores.

**Ordenação:** o EventBridge não garante ordem. As projeções devem ser tolerantes a reordenação — na prática, aplicando eventos de forma idempotente e usando `meta.version` do objeto para descartar atualizações obsoletas.

---

## 10. Evolução: projeções

A arquitetura é desenhada para que novas **projeções** — visões otimizadas para leitura, construídas consumindo eventos — possam ser adicionadas **sem alterar o modelo canônico**. Uma projeção é estado derivado; o grafo no DynamoDB continua sendo a fonte da verdade.

| Projeção | Fase | Propósito | Armazenamento provável |
| --- | --- | --- | --- |
| **Search Projection** | 3 | Busca full-text / estruturada | ex.: OpenSearch |
| **Vector Projection** | 5 | Busca semântica via embeddings | vector store |
| **Graph Projection** | 5 | Travessia multi-hop e consultas nativas de grafo | Neo4j / Amazon Neptune |
| **Analytics Projection** | 5 | Métricas e relatórios | armazenamento analítico |

Cada projeção assina o stream de eventos (§9), constrói sua própria visão e pode ser reconstruída do zero via replay do arquivo (§9.1).

A **Search Projection** é deliberadamente a primeira: ela valida o mecanismo de projeções ponta a ponta com custo baixo, antes de qualquer investimento maior.

---

## 11. Stack e organização do código

| Camada | Decisão |
| --- | --- |
| Runtime | TypeScript / Node.js 22 em Lambda ARM64 |
| Infraestrutura como código | AWS CDK em TypeScript |
| Testes | Unit no `core` · integração com DynamoDB Local · E2E em conta sandbox |

Monorepo com três pacotes:

| Pacote | Responsabilidade | Depende de AWS? |
| --- | --- | --- |
| `core` | Modelo canônico, parser, renderers, resolução de âncoras, regras de negócio | ❌ **não** |
| `api` | Handlers Lambda, repositórios, publicação de eventos | ✅ |
| `infra` | Stacks CDK | ✅ |

O `core` livre de AWS é a materialização do princípio *storage is an implementation detail*: persistência e publicação de eventos entram como interfaces, implementadas em `api`. A complexidade real do sistema — parsing, âncoras, integridade do grafo — fica testável em milissegundos, sem container nem nuvem.

---

## 12. Glossário

- **Modelo Canônico de Conhecimento** — a única representação interna de todo o conhecimento; a fonte da verdade.
- **Knowledge Object** — qualquer elemento da plataforma, compartilhando a forma de quatro dimensões.
- **Knowledge Graph** — a rede de Knowledge Objects conectados por relations.
- **Relation** — um vínculo semântico tipado, direcionado e **identificado**, de primeira classe, entre dois objetos.
- **Relation inline** — relation que se manifesta em um ponto específico do conteúdo, marcada por uma âncora (§3.7).
- **Relation standalone** — relation puramente semântica, sem posição no conteúdo (§3.7).
- **Âncora** — `[[rel_id]]` no `content`, marcando a posição de uma relation inline e referenciando-a por `id`. Nunca guarda o título do alvo (§3.8).
- **Slug** — título normalizado (`Modelo Canônico` → `modelo-canonico`) que serve de endereço único e legível do objeto; derivado automaticamente, nunca digitado (§3.3).
- **Stub** — nota criada automaticamente por um link para conhecimento que ainda não foi escrito (§8.2).
- **Renderer** — uma projeção pura e unidirecional dos objetos em uma representação (Markdown, HTML, JSON, MCP).
- **Analyzer** — um componente que enriquece o grafo a partir do content de um objeto.
- **Projeção** — uma visão otimizada para leitura, derivada do consumo de eventos (search, vector, graph, analytics).
- **Collection** — um Knowledge Object usado para agrupar outros objetos; substitui os diretórios físicos.
- **Unidade atômica** — o menor Knowledge Object que o sistema endereça. No MVP, a `Note` (§2.1).

---

_KnowledgeGraphGuru — modelar conhecimento, não documentos._
