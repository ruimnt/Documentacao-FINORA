# API de Conciliação

| | |
|---|---|
| **Público** | Analistas financeiros e desenvolvedores que automatizam o fechamento financeiro |
| **Pré-requisitos** | Token com os escopos `reconciliation:read` e `reconciliation:write`. Pagamentos criados pela [API de Pagamentos](03-api-pagamentos.md). |
| **Tempo de leitura** | 10 minutos |
| **Versão da API** | v3 |

A **conciliação** compara o que entrou no seu extrato bancário com os pagamentos registrados na Finora. A API de Conciliação faz essa comparação automaticamente e mostra o que bate, o que está pendente e o que diverge.

## Como funciona

1. Você envia os lançamentos do extrato bancário.
2. A Finora procura, para cada lançamento, o pagamento correspondente.
3. Cada lançamento recebe um status: `matched`, `unmatched` ou `divergent`.
4. Você revisa os casos que não bateram e confirma a conciliação.

### Regras de correspondência

A Finora considera que um lançamento corresponde a um pagamento quando:

| Critério | Regra |
|---|---|
| **Referência** | A descrição do lançamento ou o campo `reference` contém o ID do pagamento ou o `metadata.order_id` |
| **Valor** | O valor do lançamento é igual ao `net_amount` do pagamento |
| **Data** | A data do lançamento está até 1 dia útil de distância do `settled_at` |

| Status | Significado |
|---|---|
| `matched` | Referência, valor e data conferem. A conciliação foi feita automaticamente. |
| `divergent` | A Finora encontrou o pagamento, mas o valor difere do esperado. |
| `unmatched` | Nenhum pagamento corresponde ao lançamento. |

> [!NOTE]
> Por padrão, diferenças de até R$ 5,00 entre o lançamento e o valor esperado são tratadas como `divergent` e não como `unmatched`. Assim você vê o pagamento provável ao lado do lançamento.

## A tela de conciliação

No console, a tela de conciliação reúne o resultado do processamento. Linhas conciliadas aparecem em verde, pendentes em âmbar e divergentes em vermelho, com a diferença em reais destacada.

[Tela de conciliação com 1.248 lançamentos conciliados, 37 pendentes e 6 divergentes, e uma linha de divergência de R$ 1,25 em um crédito de cartão]

(**INSERIR IMAGEM AQUI**../images/06-mockup-conciliacao.svg)

Tudo que o console mostra também está disponível pela API, descrita a seguir.

## Endpoints

| Método | Endpoint | O que faz | Escopo |
|---|---|---|---|
| `POST` | `/reconciliation/statements` | Envia um extrato para conciliação | `reconciliation:write` |
| `GET` | `/reconciliation/matches` | Lista as correspondências | `reconciliation:read` |
| `POST` | `/reconciliation/matches/{match_id}/confirm` | Confirma uma correspondência | `reconciliation:write` |
| `GET` | `/reconciliation/reports/{date}` | Retorna o resumo de um dia | `reconciliation:read` |

## Enviar um extrato

`POST /reconciliation/statements`

Envia os lançamentos de uma conta bancária. O processamento é assíncrono: a API aceita o extrato e você consulta o resultado depois.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `account_id` | string | Sim | ID da conta bancária cadastrada, por exemplo `acc_01JA7Q2M4P6R8T0V2X4Z6B8D0F` |
| `source` | string | Sim | Origem dos dados. Use `bank_statement`. |
| `entries` | array | Sim | Lista de lançamentos (de 1 a 1.000 por requisição) |
| `entries[].external_id` | string | Sim | Identificador único do lançamento no seu banco |
| `entries[].date` | string | Sim | Data do lançamento, no formato `AAAA-MM-DD` |
| `entries[].description` | string | Sim | Texto do lançamento no extrato |
| `entries[].amount` | integer | Sim | Valor em centavos. Positivo para crédito e negativo para débito. |
| `entries[].currency` | string | Sim | Moeda. Use `BRL`. |

```bash
curl --request POST "https://sandbox.api.finora.example/v3/reconciliation/statements" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: extrato-2026-10-04" \
  --data '{
    "account_id": "acc_01JA7Q2M4P6R8T0V2X4Z6B8D0F",
    "source": "bank_statement",
    "entries": [
      {
        "external_id": "ofx-20261003-0042",
        "date": "2026-10-03",
        "description": "PIX RECEBIDO MARINA SOUZA",
        "amount": 15000,
        "currency": "BRL"
      },
      {
        "external_id": "ofx-20261004-0087",
        "date": "2026-10-04",
        "description": "CREDITO CARTAO",
        "amount": 121137,
        "currency": "BRL"
      }
    ]
  }'
```

**Resposta (`202 Accepted`):**

```json
{
  "id": "stm_01JA9C1P3R5T7V9X1Z3B5D7F9H",
  "object": "reconciliation_statement",
  "status": "processing",
  "entries_received": 2,
  "created_at": "2026-10-05T09:10:00Z"
}
```

> [!TIP]
> O status `processing` costuma durar poucos segundos. Em vez de consultar repetidamente, assine o evento `reconciliation.statement.processed` nos [webhooks](06-webhooks.md).

## Listar correspondências

`GET /reconciliation/matches`

| Parâmetro (query) | Tipo | Descrição |
|---|---|---|
| `status` | string | `matched`, `unmatched` ou `divergent` |
| `date_from` | string | Início do período (`AAAA-MM-DD`) |
| `date_to` | string | Fim do período (`AAAA-MM-DD`) |
| `limit` | integer | Itens por página, de `1` a `100`. Padrão: `20`. |
| `cursor` | string | Valor de `next_cursor` da página anterior |

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/reconciliation/matches?status=divergent&date_from=2026-10-01&date_to=2026-10-05" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`):**

```json
{
  "object": "list",
  "data": [
    {
      "id": "mat_01JA9C4R8S2T6V0W3X5Y7Z9A1B",
      "object": "reconciliation_match",
      "status": "divergent",
      "entry": {
        "external_id": "ofx-20261004-0087",
        "date": "2026-10-04",
        "description": "CREDITO CARTAO",
        "amount": 121137,
        "currency": "BRL"
      },
      "payment_id": "pay_01JA8X2D4F6H8K0M2N4P6Q8A90",
      "expected_amount": 121262,
      "difference": -125,
      "reason": "amount_mismatch",
      "created_at": "2026-10-05T09:12:00Z"
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

Como ler esse resultado:

- `amount` é o valor que entrou no banco: R$ 1.211,37.
- `expected_amount` é o `net_amount` do pagamento: R$ 1.212,62.
- `difference` é `amount` menos `expected_amount`, em centavos: `-125` equivale a R$ 1,25 a menos.

## Confirmar uma correspondência

`POST /reconciliation/matches/{match_id}/confirm`

Confirma que você revisou o lançamento e aceita a correspondência, mesmo com divergência.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `note` | string | Não | Justificativa registrada na auditoria. Até 280 caracteres. |

```bash
curl --request POST \
  "https://sandbox.api.finora.example/v3/reconciliation/matches/mat_01JA9C4R8S2T6V0W3X5Y7Z9A1B/confirm" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{ "note": "Diferença de R$ 1,25 por arredondamento nas parcelas." }'
```

**Resposta (`200 OK`):**

```json
{
  "id": "mat_01JA9C4R8S2T6V0W3X5Y7Z9A1B",
  "object": "reconciliation_match",
  "status": "matched",
  "confirmed": true,
  "note": "Diferença de R$ 1,25 por arredondamento nas parcelas.",
  "confirmed_at": "2026-10-05T10:01:33Z"
}
```

> [!WARNING]
> A confirmação não pode ser desfeita pela API. Confirme só depois de entender a causa da diferença. Em caso de dúvida, veja o [troubleshooting de conciliação](07-troubleshooting.md#conciliação-com-divergência-de-centavos).

## Consultar o relatório do dia

`GET /reconciliation/reports/{date}`

Retorna o resumo da conciliação de um dia (`AAAA-MM-DD`).

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/reconciliation/reports/2026-10-04" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`):**

```json
{
  "object": "reconciliation_report",
  "date": "2026-10-04",
  "totals": {
    "entries": 214,
    "matched": 205,
    "unmatched": 7,
    "divergent": 2
  },
  "amounts": {
    "expected": 4851230,
    "received": 4851105,
    "difference": -125
  },
  "currency": "BRL"
}
```

## Erros desta API

| HTTP | `code` | Quando acontece | Como resolver |
|---|---|---|---|
| 400 | `invalid_request` | O JSON está malformado | Valide o corpo antes de enviar |
| 403 | `insufficient_scope` | Falta o escopo `reconciliation:read` ou `reconciliation:write` | Peça um token com o escopo correto |
| 404 | `not_found` | O `match_id` ou o `account_id` não existe | Confira o ID |
| 409 | `duplicate_entry` | Um `external_id` já foi enviado antes | Envie apenas lançamentos novos |
| 422 | `invalid_entry` | Data inválida ou valor não inteiro em um lançamento | Corrija o lançamento indicado em `param` |
| 422 | `statement_too_large` | O extrato tem mais de 1.000 lançamentos | Divida o envio em lotes menores |

## Próximos passos

- Receba o aviso de extrato processado em [Webhooks](06-webhooks.md).
- Entenda termos como *líquido* e *liquidação* no [Glossário](10-glossario-e-faq.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** duas pessoas ao mesmo tempo. A analista financeira precisa entender *o que a conciliação faz*, e o desenvolvedor precisa saber *como automatizá-la*.
- **Conceito antes da API.** A página explica as regras de correspondência em uma tabela, antes dos endpoints, porque sem esse modelo mental os resultados parecem arbitrários.
- **Mockup da tela** ao lado da API, para ligar o que a analista vê no console ao que o desenvolvedor recebe no JSON.
- **Exemplo comentado de divergência.** O leitor vê o resultado e a explicação campo a campo (`amount`, `expected_amount`, `difference`), com os mesmos números do mockup.
- **Aviso sobre a confirmação irreversível** colocado junto da operação, no momento em que o leitor pode cometer o erro.
</details>
