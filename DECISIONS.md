# KnowledgeGraphGuru — Decisões de Arquitetura

Registro das decisões tomadas antes do início da implementação da Fase 1 (MVP). Cada entrada entrega a decisão, o motivo e as consequências assumidas.

O escopo e o modelo estão em **[DESIGN.md](DESIGN.md)**. Questões ainda em aberto ficam em [DESIGN.md §11](DESIGN.md).

| # | Decisão | Status |
| --- | --- | --- |
| [ADR-001](#adr-001--runtime-typescript--nodejs) | Runtime: TypeScript / Node.js | ✅ Aceita |
| [ADR-002](#adr-002--iac-aws-cdk) | IaC: AWS CDK | ✅ Aceita |
| [ADR-003](#adr-003--monorepo-com-core-isolado-da-aws) | Monorepo com `core` isolado da AWS | ✅ Aceita |
| [ADR-004](#adr-004--testes-em-camadas-com-dynamodb-local) | Testes em camadas com DynamoDB Local | ✅ Aceita |
| [ADR-005](#adr-005--content-como-string-legível-com-âncoras) | Content como string legível com âncoras | ✅ Aceita |
| [ADR-006](#adr-006--single-table-com-adjacency-list) | Single-table com adjacency list | ✅ Aceita |
| [ADR-007](#adr-007--api-canônica-com-importação-como-adaptador) | API canônica, importação como adaptador | ✅ Aceita |
| [ADR-008](#adr-008--identificadores-ulid-com-prefixo) | Identificadores ULID com prefixo | ✅ Aceita |
| [ADR-009](#adr-009--vocabulário-fechado-de-relations) | Vocabulário fechado de relations | ✅ Aceita |
| [ADR-010](#adr-010--delete-cascateia) | Delete cascateia | ✅ Aceita |
| [ADR-011](#adr-011--âncoras-órfãs-são-removidas-na-mesma-transação) | Âncoras órfãs removidas na mesma transação | ✅ Aceita |
| [ADR-012](#adr-012--links-não-resolvidos-viram-notas-stub) | Links não resolvidos viram notas stub | ✅ Aceita |
| [ADR-013](#adr-013--uma-relation-por-ocorrência) | Uma relation por ocorrência | ✅ Aceita |
| [ADR-014](#adr-014--um-único-contentformat-no-mvp) | Um único `content.format` no MVP | ✅ Aceita |
| [ADR-015](#adr-015--autenticação-por-api-key-única) | Autenticação por API key única | ✅ Aceita |
| [ADR-016](#adr-016--bytes-de-anexo-não-passam-pela-api) | Bytes de anexo não passam pela API | ✅ Aceita |
| [ADR-017](#adr-017--backlinks-são-eventualmente-consistentes) | Backlinks são eventualmente consistentes | ✅ Aceita |

---

## ADR-001 — Runtime: TypeScript / Node.js

**Decisão.** As Lambdas rodam **TypeScript sobre Node.js 22**, em arquitetura ARM64.

**Por quê.** O coração do MVP é parsing e renderização de Markdown, e o melhor ecossistema para isso é JavaScript (`unified`/`remark`/`mdast`). O SDK oficial de MCP também é TypeScript, o que elimina uma troca de linguagem na Fase 2. O sistema de tipos expressa bem as quatro dimensões do modelo canônico.

**Consequências.** Uma única linguagem do `core` à infraestrutura (ADR-002). Analyzers de outras linguagens (Java, SQL, Terraform) precisarão de parsers em JS ou de um mecanismo de plugin fora do processo — problema pós-MVP.

---

## ADR-002 — IaC: AWS CDK

**Decisão.** A infraestrutura é definida com **AWS CDK em TypeScript**.

**Por quê.** Mesma linguagem da aplicação, com tipagem e testes sobre a própria infraestrutura. Boa cobertura dos serviços serverless em uso.

**Consequências.** Deploy depende de bootstrap do CDK na conta. Quem for operar a instância self-hosted precisa de Node e credenciais AWS.

---

## ADR-003 — Monorepo com `core` isolado da AWS

**Decisão.** Monorepo com três pacotes:

| Pacote | Responsabilidade | Depende de AWS? |
| --- | --- | --- |
| `core` | Modelo canônico, parser, renderers, resolução de âncoras, regras de negócio | ❌ **não** |
| `api` | Handlers Lambda, repositórios, publicação de eventos | ✅ |
| `infra` | Stacks CDK | ✅ |

**Por quê.** A complexidade real do sistema está no `core` — parsing, âncoras e integridade do grafo. Mantê-lo livre de AWS torna essa parte testável em milissegundos, sem container nem nuvem, e protege o modelo canônico de vazamentos do detalhe de armazenamento (princípio: *storage is an implementation detail*).

**Consequências.** Persistência e publicação de eventos entram no `core` como interfaces, implementadas em `api`. Exige disciplina para não importar SDK da AWS no `core`.

---

## ADR-004 — Testes em camadas com DynamoDB Local

**Decisão.** Três camadas:

1. **Unit (`core`)** — puro, em memória, sem AWS. A maior parte da suíte.
2. **Integração** — repositórios contra **DynamoDB Local** (`amazon/dynamodb-local`, `-inMemory -sharedDb`) via Docker.
3. **E2E** — deploy real em conta sandbox, valida API e eventos.

EventBridge não é emulado: o publisher é abstraído por uma interface e substituído por um fake nos testes de integração.

**Por quê.** Feedback rápido onde está a complexidade, e validação real onde está o risco de infraestrutura.

**Consequências e limitações assumidas.** DynamoDB Local **atualiza GSIs de forma síncrona**, enquanto o DynamoDB real é eventualmente consistente — ver [ADR-017](#adr-017--backlinks-são-eventualmente-consistentes). Também não há IAM, throttling real nem expiração de TTL.

---

## ADR-005 — Content como string legível com âncoras

**Decisão.** `content.value` é uma **string** no formato `ckm/text`: sintaxe inline de Markdown (`**negrito**`, `` `código` ``) somada a **âncoras** delimitadas por `⟦ ⟧` (U+27E6 / U+27E7), que referenciam relations por `id`.

```
"Conforme o ⟦rel_01H8Z⟧, o ⟦rel_01H8W⟧ resolve cada relação."
```

**Alternativa rejeitada.** AST de nós inline (`[{"text":...},{"ref":...}]`). Elimina escaping e é conceitualmente mais puro, mas sacrifica legibilidade e diffabilidade — que são requisitos explícitos deste projeto.

**Por quê.** O conteúdo permanece inspecionável a olho nu e legível em diffs do git. Os delimitadores `⟦ ⟧` praticamente não ocorrem em texto natural, o que reduz o escaping a um caso de borda em vez de um problema de design.

**Consequências.** O AST inline fica para a Fase 4, junto com a decomposição em blocos — granularidade vertical (blocos) e horizontal (nós inline) são a mesma máquina e serão construídas de uma vez.

> **Nota conceitual.** Granularizar em blocos **não** preserva a posição de um link dentro de um parágrafo — um parágrafo com dois links continua sendo um único bloco de texto. Âncoras são necessárias em qualquer nível de granularidade, e são elas, não a granularidade, que resolvem a remontagem do Markdown.

---

## ADR-006 — Single-table com adjacency list

**Decisão.** Uma tabela DynamoDB, com objetos e arestas coexistindo.

| Item | PK | SK |
| --- | --- | --- |
| Knowledge Object | `KO#<id>` | `META` |
| Aresta (relation) | `KO#<origem>` | `REL#<tipo>#<relId>` |

| Índice | PK | Serve |
| --- | --- | --- |
| **GSI1** (invertido) | `KO#<alvo>` | Backlinks e "de quais collections este objeto faz parte" |
| **GSI2** | `TYPE#<tipo>` | Listagens por tipo |

**Pertencer a uma collection é uma relation** (`in_collection`), não um mecanismo separado.

**Por quê.** Um único mecanismo — arestas — resolve relations, backlinks e pertencimento a collections. Menos código, menos casos especiais, e collections ganham de graça tudo que vale para relations.

**Consequências.** Resolve bem **um salto**. Travessia multi-hop exige N queries sequenciais e fica para a Graph Projection (Fase 5). O limite de 100 itens do `TransactWriteItems` é confortável com a `Note` atômica.

---

## ADR-007 — API canônica, importação como adaptador

**Decisão.**

| Rota | Recebe |
| --- | --- |
| `POST /notes` | Objeto **canônico** |
| `POST /import/markdown` | Markdown bruto |
| `GET /notes/{id}?render=markdown\|json` | — |

**Por quê.** Mantém o núcleo da API canônico e trata Markdown como o que ele é no modelo: um formato de entrada, não a primitiva. Novos formatos de importação viram novas rotas de adaptador, sem tocar no núcleo.

**Consequências.** Clientes que só falam Markdown usam a rota de importação; agentes e integrações usam a canônica.

---

## ADR-008 — Identificadores ULID com prefixo

**Decisão.** **ULID** com prefixo por tipo: `ko_01H8Z...` para Knowledge Objects, `rel_01H8Z...` para relations.

**Por quê.** Ordenável por tempo de criação (útil como SK e em listagens), sem coordenação, e o prefixo torna o identificador autoexplicativo em logs e depuração.

**Consequências.** Relations passam a ter identidade própria — requisito do mecanismo de âncoras ([ADR-005](#adr-005--content-como-string-legível-com-âncoras)) e coerente com o princípio de que relations são cidadãs de primeira classe.

---

## ADR-009 — Vocabulário fechado de relations

**Decisão.** Conjunto fechado e validado no MVP:

`references` · `depends_on` · `related_to` · `implements` · `alternative_to` · `embeds` · `in_collection`

**Por quê.** Qualidade de dado acima de flexibilidade prematura. Um vocabulário aberto degrada rápido em sinônimos (`refers_to`, `ref`, `references`) e inviabiliza consultas consistentes no grafo.

**Consequências.** Ampliar o vocabulário exige mudança de código no MVP. Abrir o conjunto depois é retrocompatível; fechá-lo depois, não.

---

## ADR-010 — Delete cascateia

**Decisão.** Deletar um Knowledge Object remove também as relations que **apontam para ele**, emitindo `RelationRemoved` para cada uma.

**Por quê.** Evita arestas pendentes apontando para o vazio. Um grafo com alvos inexistentes contamina backlinks, renderização e toda projeção futura.

**Consequências.** O delete é uma operação de custo variável, proporcional ao número de backlinks. As âncoras correspondentes são tratadas por [ADR-011](#adr-011--âncoras-órfãs-são-removidas-na-mesma-transação).

---

## ADR-011 — Âncoras órfãs são removidas na mesma transação

**Decisão.** Ao remover uma relation inline, a âncora correspondente é removida do `content` do objeto de origem na **mesma transação**.

**Por quê.** Content e relations precisam permanecer coerentes. Uma âncora apontando para uma relation inexistente é um estado inválido que o renderer teria que adivinhar como tratar.

**Consequências.** Remover uma relation altera o conteúdo do objeto de origem — e portanto incrementa sua versão e emite `KnowledgeUpdated`. O texto que envolvia o link é preservado; some apenas o vínculo.

---

## ADR-012 — Links não resolvidos viram notas stub

**Decisão.** Ao importar `[[Nota Que Não Existe]]`, o sistema cria uma `Note` com `properties.status = "stub"` e estabelece a relation normalmente.

**Por quê.** Mantém o grafo íntegro e sem casos especiais: toda relation aponta para um objeto real. Preserva o fluxo de escrita do ecossistema Markdown, em que se cita algo antes de escrevê-lo — e a nota stub passa a ser um item de trabalho visível no grafo.

**Consequências.** Importar um vault gera notas stub. Escrever conteúdo em uma stub é apenas atualizá-la; o `id` já existe e nenhum vínculo precisa ser refeito.

---

## ADR-013 — Uma relation por ocorrência

**Decisão.** Citar a mesma nota duas vezes no mesmo texto gera **duas relations**, com `id` distinto e âncoras distintas.

**Por quê.** Cada ocorrência é uma asserção própria, em uma posição própria. Modelar como uma relation com múltiplas âncoras complica escrita, remoção e o mecanismo de ancoragem sem benefício correspondente.

**Consequências.** Consultas e projeções deduplicam quando a pergunta for "quais notas esta referencia".

---

## ADR-014 — Um único `content.format` no MVP

**Decisão.** O MVP reconhece apenas `ckm/text`.

**Por quê.** A taxonomia de formatos só ganha sentido quando houver múltiplos tipos de content com analyzers distintos — o que começa na Fase 2. Definir o conjunto completo agora seria especular.

**Consequências.** O campo existe desde o início e é validado, então introduzir novos formatos não quebra dados já gravados.

---

## ADR-015 — Autenticação por API key única

**Decisão.** Uma API key única, guardada no Secrets Manager e validada no API Gateway.

**Por quê.** O MVP é single-user / self-hosted: não há recortes de conhecimento por usuário a proteger, apenas a instância inteira. Qualquer coisa além disso seria construir para um requisito que não existe.

**Consequências.** Não há identidade de usuário nos eventos nem trilha de auditoria por pessoa. Se o projeto evoluir para multi-tenant, autenticação e autorização serão redesenhadas — o que já está registrado como questão em aberto.

---

## ADR-016 — Bytes de anexo não passam pela API

**Decisão.** O upload de binários usa **presigned URL do S3**. A API autoriza a operação e cria o Knowledge Object `Attachment`; apenas os bytes vão direto ao S3.

A regra passa a ser enunciada como: *"nenhuma escrita no **modelo de conhecimento** ocorre fora da API"*.

**Por quê.** O payload máximo do Lambda (6 MB) inviabiliza proxy de arquivos grandes. Os bytes de um anexo são payload, não conhecimento — o conhecimento é o objeto `Attachment`, e esse continua sendo criado exclusivamente pela API.

**Consequências.** Existe uma janela entre criar o objeto e concluir o upload. Um `Attachment` pode existir apontando para um binário ainda não enviado, e o estado precisa refletir isso.

---

## ADR-017 — Backlinks são eventualmente consistentes

**Decisão.** A leitura de backlinks (via GSI1) é **eventualmente consistente por contrato**. A API não promete que uma relation recém-criada apareça imediatamente nos backlinks do alvo.

**Por quê.** GSIs do DynamoDB são atualizados de forma assíncrona. Prometer consistência forte seria prometer algo que o armazenamento não entrega.

**Consequências.** Nenhum teste deve assumir leitura imediata de GSI após escrita — inclusive porque o DynamoDB Local atualiza GSIs de forma **síncrona** ([ADR-004](#adr-004--testes-em-camadas-com-dynamodb-local)), o que faria esse teste passar localmente e falhar em produção. Clientes que precisem de confirmação imediata devem ler as relations de saída do objeto de origem, que são fortemente consistentes.

---

_KnowledgeGraphGuru — modelar conhecimento, não documentos._
