# Especificação DynamoDB — Solicitações

## 1. Tabela: Solicitações (`Solicitacoes`)
- Partition Key: `solicitacaoId` (S)
- Sort Key: nenhuma
- TTL: `ttl`

## 2. Atributos

| Atributo | Tipo DynamoDB |
|---|---|
| solicitacaoId | String (S) |
| tipoEmpresa | String (S) |
| empresa | String (S) |
| codigoBacen | String (S) |
| funcionalSolicitante | String (S) |
| statusSolicitacao | Number (N) |
| statusCompetencias | Map (M) |
| tipoSolicitacao | String (S) |
| dataSolicitacao | String (S) |
| dataInicio | String (S) |
| dataFim | String (S) |
| tipoDia | String (S) |
| numeroDia | Number (N) |
| chaveListagem | String (S) |
| chaveProcessamento | String (S) |
| ttl | Number (N) |

### Formatos de armazenamento
- `solicitacaoId`: UUID v4.
- `dataSolicitacao`: ISO 8601 em UTC (`YYYY-MM-DDTHH:mm:ss.SSSZ`).
- `dataInicio` e `dataFim`: `YYYY-MM-DD`.
- `ttl`: Unix timestamp em segundos, somente para eventuais, criação + 2 meses.

### Status por competência
- `statusSolicitacao`: ID numérico do status geral da solicitação, independente dos status individuais.
- `statusCompetencias`: atributo exclusivo das solicitações eventuais, com chaves no formato `YYYY-MM` e valores do tipo Number (N), representando os IDs dos status.
- Na criação da solicitação, o backend gera todas as competências entre `dataInicio` e `dataFim`, inclusive, inicializando cada status com o ID numérico correspondente a pendente. O mapa é persistido junto com a solicitação em um único `PutItem`.
- O frontend envia o intervalo de datas, sem precisar montar o mapa.
- O executor informa o ID da solicitação e os IDs numéricos dos status das competências. Cada competência pode ser atualizada atomicamente via `UpdateItem`, sem leitura prévia, usando `SET statusCompetencias.#competencia = :status`, com alias para a chave do mapa.
- O `GSI_STATUS` continua indexando exclusivamente `statusSolicitacao`.

### Catálogo de status
- O DynamoDB armazena exclusivamente IDs numéricos dos status, sem duplicar descrições.
- O catálogo de IDs e descrições é mantido no backend Python, utilizando `IntEnum`, sem tabela adicional no DynamoDB.
- O backend disponibiliza o catálogo ao frontend. Os seletores exibem as descrições e enviam os IDs numéricos; as listagens retornam os IDs para apresentação pelo frontend.
- O executor utiliza IDs numéricos conforme o contrato de integração.
- Os IDs são imutáveis e nunca reutilizados.

## 3. GSIs

| Índice | Partition Key | Sort Key | Projection |
|---|---|---|---|
| GSI_LISTAGEM | chaveListagem | dataSolicitacao | ALL |
| GSI_STATUS | statusSolicitacao | dataSolicitacao | ALL |
| GSI_PROCESSAMENTO | chaveProcessamento | dataSolicitacao | ALL |

A Partition Key do `GSI_STATUS` é Number (N). As demais chaves dos GSIs são String (S).

## 4. Chaves auxiliares
- `chaveListagem`: `SOLICITACOES`.
- `chaveProcessamento`: `EVENTUAL` para eventuais; `tipoDia#numeroDia` para recorrentes.

## 5. Padrões de acesso

| Operação | Estrutura | Filtros adicionais |
|---|---|---|
| GetItem por ID | Tabela principal | — |
| DeleteItem por ID | Tabela principal | — |
| UpdateItem por ID | Tabela principal | — |
| Query sem status | GSI_LISTAGEM | BACEN e funcional |
| Query com status | GSI_STATUS | BACEN e funcional |
| Query de eventuais | GSI_PROCESSAMENTO | ID do status pendente |
| Query de recorrentes | GSI_PROCESSAMENTO | — |

## 6. Listagem
- Ordenação por `dataSolicitacao`, decrescente por padrão.
- Direção controlada por `ScanIndexForward`.
- Filtros adicionais via `FilterExpression`.
- Paginação via `LastEvaluatedKey` e `ExclusiveStartKey`.
