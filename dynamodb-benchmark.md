# Benchmark estimado — DynamoDB

## 1. Premissas

| Parâmetro | Valor |
|---|---:|
| Total de registros | 10.000 |
| Consultas diárias do frontend | 100 a 500 |
| Registros por página | 20 |
| Tamanho médio por registro | 1 KB |
| Cardinalidade de status | 12 |
| Solicitações eventuais armazenadas | 600 |
| Execuções da Lambda | 1/dia |
| Capacidade | On-demand |
| Região de referência | us-east-1 |
| Preço estimado por milhão de RRUs | US$ 0,125 |

## 2. Complexidade de tempo

- n: total de registros.
- m: registros associados à chave consultada.
- k: registros retornados.

| Consulta | Melhor caso | Pior caso |
|---|---|---|
| GetItem por ID | Ω(1) | O(1) |
| Frontend sem filtros | Ω(1) | O(1) |
| Frontend somente com status | Ω(1) | O(1) |
| Frontend com status + filtro adicional | Ω(1) | O(m) |
| Lambda — eventuais | Ω(1) | O(m) |
| Lambda — recorrentes | Ω(1 + k) | O(1 + k) |

O acesso direto pela Partition Key é abstraído como O(1), sem representar uma garantia formal da implementação interna do DynamoDB. A leitura dos registros retornados é inevitável, com complexidade Θ(k). Somente as consultas com FilterExpression podem avaliar registros desnecessários. No pior caso absoluto, m = n.

## 3. Custos de leitura do frontend

Considerando 30 dias e consistência eventual.

| Cenário | Registros avaliados por consulta | 100 consultas/dia | 500 consultas/dia |
|---|---:|---:|---:|
| Sem filtros | 20 | US$ 0,001 | US$ 0,005 |
| Com filtros — status uniforme | 834 | US$ 0,039 | US$ 0,196 |
| Com filtros — pior caso absoluto | 10.000 | US$ 0,469 | US$ 2,344 |

O cenário filtrado considera um parâmetro indexado (statusSolicitacao) e outro aplicado via FilterExpression, percorrendo toda a partição correspondente no pior caso.

## 4. Processamento diário da Lambda

Uma execução diária, com até três consultas lógicas ao GSI_PROCESSAMENTO (cada uma pode exigir paginação).

| Cenário | Registros avaliados/dia | Custo mensal |
|---|---:|---:|
| Distribuição uniforme | 956 | US$ 0,00045 |
| Pior caso absoluto | 10.000 | US$ 0,00469 |

As consultas de recorrentes são assintoticamente ótimas quanto ao volume de registros avaliados, pois acessam diretamente os registros necessários ao processamento.

## 5. Armazenamento

Tabela principal e três GSIs com projeção ALL:
- Volume aproximado: 40 MB.
- Custo bruto mensal: US$ 0,01.

## 6. Custo mensal consolidado

500 consultas diárias + processamento diário + armazenamento.

| Cenário | Custo mensal |
|---|---:|
| Sem filtros | US$ 0,02 |
| Com filtros — distribuição uniforme | US$ 0,21 |
| Pior caso absoluto | US$ 2,36 |

## 7. Conclusão

O modelo apresenta acesso direto por chave e leitura proporcional aos resultados nas consultas sem filtros adicionais.

Mesmo no pior caso absoluto, o custo operacional de leitura e armazenamento permanece baixo. O principal ponto de atenção é a latência das consultas com FilterExpression, não o custo financeiro.

*Estimativas teóricas, sem arredondamentos adicionais por chamada, franquias gratuitas, custos de escrita, execução da Lambda ou S3. Não representam medições reais de latência. A tarifa é uma premissa de cálculo, não uma cotação verificada.*

Referência: [Especificação DynamoDB](dynamodb-spec.md).
