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
| statusSolicitacao | String (S) |
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

## 3. GSIs

| Índice | Partition Key | Sort Key | Projection |
|---|---|---|---|
| GSI_LISTAGEM | chaveListagem | dataSolicitacao | ALL |
| GSI_STATUS | statusSolicitacao | dataSolicitacao | ALL |
| GSI_PROCESSAMENTO | chaveProcessamento | dataSolicitacao | ALL |

Todas as chaves dos GSIs são String (S).

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
| Query de eventuais | GSI_PROCESSAMENTO | Status pendente |
| Query de recorrentes | GSI_PROCESSAMENTO | — |

## 6. Listagem
- Ordenação por `dataSolicitacao`, decrescente por padrão.
- Direção controlada por `ScanIndexForward`.
- Filtros adicionais via `FilterExpression`.
- Paginação via `LastEvaluatedKey` e `ExclusiveStartKey`.
