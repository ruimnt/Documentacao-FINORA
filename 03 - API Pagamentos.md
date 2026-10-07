# API de Pagamentos

| | |
|---|---|
| **Público** | Desenvolvedores que criam, consultam e estornam pagamentos |
| **Pré-requisitos** | Um token de acesso com os escopos `payments:read` e `payments:write` (veja [Autenticação](02-autenticacao.md)) |
| **Tempo de leitura** | 15 minutos |
| **Versão da API** | v3 |

Use a API de Pagamentos para receber por **Pix**, **boleto** e **cartão de crédito** com uma única integração.

## Visão geral

| Item | Valor |
|---|---|
| **URL base (sandbox)** | `https://sandbox.api.finora.example/v3` |
| **URL base (produção)** | `https://api.finora.example/v3` |
| **Formato** | JSON em UTF-8. Envie `Content-Type: application/json` nas requisições com corpo. |
| **Autenticação** | Cabeçalho `Authorization: Bearer <access_token>` |

### Convenções

- **Valores em centavos.** O campo `amount` é um número inteiro: R$ 150,00 equivale a `15000`. Nunca envie decimais.
- **Datas em ISO 8601 e UTC.** Exemplo: `2026-10-05T13:30:00Z`.
- **IDs com prefixo.** Todo ID indica o tipo do objeto: `pay_` (pagamento), `ref_` (estorno), `whk_` (webhook).
- **Paginação por cursor.** As listas retornam `has_more` e `next_cursor`.
- **Limite de requisições.** 600 por minuto por credencial. Os cabeçalhos `X-RateLimit-Limit`, `X-RateLimit-Remaining` e `X-RateLimit-Reset` mostram seu consumo.

### Endpoints

| Método | Endpoint | O que faz | Escopo |
|---|---|---|---|
| `POST` | `/payments` | Cria um pagamento | `payments:write` |
| `GET` | `/payments/{payment_id}` | Consulta um pagamento | `payments:read` |
| `GET` | `/payments` | Lista pagamentos com filtros | `payments:read` |
| `PATCH` | `/payments/{payment_id}` | Atualiza a descrição e os metadados | `payments:write` |
| `DELETE` | `/payments/{payment_id}` | Cancela um pagamento pendente | `payments:write` |
| `POST` | `/payments/{payment_id}/refunds` | Estorna um pagamento liquidado | `payments:write` |

## Ciclo de vida de um pagamento

[Estados de um pagamento: created, pending, authorized, settled e refunded, com os desvios failed e canceled]

(**INSERIR IMAGEM AQUI**../images/04-ciclo-de-vida-pagamento.svg)

| Status | Significado |
|---|---|
| `created` | A Finora registrou o pagamento |
| `pending` | Aguardando o cliente pagar (Pix, boleto) ou a análise do emissor (cartão) |
| `authorized` | O pagamento foi aprovado, mas o dinheiro ainda não foi liquidado |
| `settled` | O dinheiro foi liquidado na sua conta |
| `failed` | O pagamento foi recusado, expirou ou houve erro de processamento |
| `canceled` | Você cancelou o pagamento enquanto ele estava pendente |
| `refunded` | O pagamento foi estornado, total ou parcialmente |

> [!NOTE]
> Cada mudança de status gera um evento. Para reagir a eles sem consultar a API, use [webhooks](06-webhooks.md).

## Criar um pagamento

`POST /payments`

Cria um pagamento Pix, boleto ou cartão.

### Cabeçalhos

| Cabeçalho | Obrigatório | Descrição |
|---|---|---|
| `Authorization` | Sim | `Bearer <access_token>` |
| `Content-Type` | Sim | `application/json` |
| `Idempotency-Key` | Recomendado | Texto único (até 64 caracteres) que identifica a tentativa. Veja [Idempotência](#idempotência). |

### Parâmetros do corpo

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `amount` | integer | Sim | Valor em centavos. Mínimo de `100` (R$ 1,00). |
| `currency` | string | Sim | Moeda. Use `BRL`. |
| `method` | string | Sim | Forma de pagamento: `pix`, `boleto` ou `card`. |
| `description` | string | Não | Descrição exibida ao cliente. Até 140 caracteres. |
| `customer` | object | Sim | Dados do pagador. |
| `customer.name` | string | Sim | Nome completo. |
| `customer.document` | string | Sim | CPF ou CNPJ, com ou sem pontuação. |
| `customer.email` | string | Sim | E-mail do pagador. |
| `expires_in` | integer | Não | Validade em segundos para Pix e boleto. Padrão: `3600` para Pix e `259200` (3 dias) para boleto. |
| `card_token` | string | Condicional | Obrigatório quando `method` é `card`. Token gerado pelo SDK de cartão da Finora. |
| `installments` | integer | Não | Número de parcelas, de `1` a `12`. Só vale para `card`. Padrão: `1`. |
| `metadata` | object | Não | Até 20 pares de chave e valor, para guardar dados seus (por exemplo, o número do pedido). |

### Exemplo de requisição

```bash
curl --request POST "https://sandbox.api.finora.example/v3/payments" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: pedido-4821-tentativa-1" \
  --data '{
    "amount": 15000,
    "currency": "BRL",
    "method": "pix",
    "description": "Pedido #4821 - Livraria Boa Página",
    "expires_in": 3600,
    "customer": {
      "name": "Marina Souza",
      "document": "123.456.789-09",
      "email": "marina.souza@example.com"
    },
    "metadata": { "order_id": "4821" }
  }'
```

### Exemplo de resposta (`201 Created`)

```json
{
  "id": "pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F",
  "object": "payment",
  "status": "pending",
  "amount": 15000,
  "currency": "BRL",
  "method": "pix",
  "description": "Pedido #4821 - Livraria Boa Página",
  "fee": 0,
  "net_amount": 15000,
  "customer": {
    "name": "Marina Souza",
    "document": "123.456.789-09",
    "email": "marina.souza@example.com"
  },
  "pix": {
    "copy_and_paste": "00020126580014br.gov.bcb.pix0136finora-demo-000000000000520400005303986540515.005802BR5913Livraria Boa6009SAO PAULO62070503***63041D3F",
    "expires_at": "2026-10-05T14:30:00Z"
  },
  "metadata": {
    "order_id": "4821"
  },
  "created_at": "2026-10-05T13:30:00Z",
  "updated_at": "2026-10-05T13:30:00Z"
}
```

> [!NOTE]
> `fee` é a tarifa cobrada pelo pagamento e `net_amount` é o valor líquido que você recebe (`amount` menos `fee`). A [conciliação](04-api-conciliacao.md) compara o extrato bancário com o `net_amount`.

### Exemplo com cartão de crédito

Para cartão, envie `card_token` e, se quiser, `installments`. No sandbox, use os tokens de teste da [tabela abaixo](#dados-de-teste-no-sandbox).

```json
{
  "amount": 125000,
  "currency": "BRL",
  "method": "card",
  "card_token": "tok_test_approved",
  "installments": 3,
  "description": "Pedido #4822 - Livraria Boa Página",
  "customer": {
    "name": "Marina Souza",
    "document": "123.456.789-09",
    "email": "marina.souza@example.com"
  }
}
```

### Exemplo em Python

```python
import os
import uuid

import requests

BASE_URL = "https://sandbox.api.finora.example/v3"


def criar_pagamento_pix(valor_em_centavos: int, pedido: str) -> dict:
    response = requests.post(
        f"{BASE_URL}/payments",
        headers={
            "Authorization": f"Bearer {os.environ['FINORA_TOKEN']}",
            # Reutilize a mesma chave ao repetir a MESMA tentativa.
            "Idempotency-Key": f"pedido-{pedido}-{uuid.uuid4()}",
        },
        json={
            "amount": valor_em_centavos,
            "currency": "BRL",
            "method": "pix",
            "description": f"Pedido #{pedido} - Livraria Boa Página",
            "customer": {
                "name": "Marina Souza",
                "document": "123.456.789-09",
                "email": "marina.souza@example.com",
            },
            "metadata": {"order_id": pedido},
        },
        timeout=10,
    )
    response.raise_for_status()
    return response.json()


pagamento = criar_pagamento_pix(15000, "4821")
print(pagamento["id"], pagamento["status"])
```

### Exemplo em JavaScript

```javascript
const BASE_URL = "https://sandbox.api.finora.example/v3";

async function criarPagamentoPix(valorEmCentavos, pedido) {
  const response = await fetch(`${BASE_URL}/payments`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.FINORA_TOKEN}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `pedido-${pedido}-${crypto.randomUUID()}`,
    },
    body: JSON.stringify({
      amount: valorEmCentavos,
      currency: "BRL",
      method: "pix",
      description: `Pedido #${pedido} - Livraria Boa Página`,
      customer: {
        name: "Marina Souza",
        document: "123.456.789-09",
        email: "marina.souza@example.com",
      },
      metadata: { order_id: pedido },
    }),
  });

  if (!response.ok) {
    const { error } = await response.json();
    throw new Error(`${error.code}: ${error.message} (${error.request_id})`);
  }
  return response.json();
}

const pagamento = await criarPagamentoPix(15000, "4821");
console.log(pagamento.id, pagamento.status);
```

### Erros desta operação

| HTTP | `code` | Quando acontece |
|---|---|---|
| 400 | `invalid_request` | O corpo não é um JSON válido |
| 401 | `invalid_token` ou `token_expired` | Token ausente, inválido ou expirado |
| 403 | `insufficient_scope` | O token não tem o escopo `payments:write` |
| 409 | `idempotency_conflict` | A `Idempotency-Key` já foi usada com outro corpo |
| 422 | `invalid_amount` | O valor é menor que `100` ou não é inteiro |
| 422 | `invalid_document` | CPF ou CNPJ inválido |
| 422 | `card_declined` | O emissor recusou o cartão |
| 429 | `rate_limited` | Você passou do limite de requisições |

## Consultar um pagamento

`GET /payments/{payment_id}`

Retorna um pagamento pelo ID.

| Parâmetro (caminho) | Tipo | Descrição |
|---|---|---|
| `payment_id` | string | ID do pagamento, por exemplo `pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F` |

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`):**

```json
{
  "id": "pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F",
  "object": "payment",
  "status": "settled",
  "amount": 15000,
  "currency": "BRL",
  "method": "pix",
  "description": "Pedido #4821 - Livraria Boa Página",
  "fee": 0,
  "net_amount": 15000,
  "metadata": {
    "order_id": "4821"
  },
  "settled_at": "2026-10-05T13:32:41Z",
  "created_at": "2026-10-05T13:30:00Z",
  "updated_at": "2026-10-05T13:32:41Z"
}
```

**Erro quando o pagamento não existe (`404 Not Found`):**

```json
{
  "error": {
    "code": "not_found",
    "message": "Não encontramos um pagamento com o ID informado.",
    "param": "payment_id",
    "request_id": "req_01JA8ZB7N4P6R8T0V2X4Z6B8D0",
    "doc_url": "https://docs.finora.example/errors/not_found"
  }
}
```

## Listar pagamentos

`GET /payments`

Retorna os pagamentos do mais recente para o mais antigo, com paginação por cursor.

| Parâmetro (query) | Tipo | Descrição |
|---|---|---|
| `status` | string | Filtra por status, por exemplo `settled` |
| `method` | string | Filtra por método: `pix`, `boleto` ou `card` |
| `created_from` | string | Início do período, em ISO 8601 |
| `created_to` | string | Fim do período, em ISO 8601 |
| `limit` | integer | Itens por página, de `1` a `100`. Padrão: `20`. |
| `cursor` | string | Valor de `next_cursor` da página anterior |

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/payments?status=settled&method=pix&limit=2" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`), com os objetos abreviados:**

```json
{
  "object": "list",
  "data": [
    {
      "id": "pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F",
      "status": "settled",
      "amount": 15000,
      "method": "pix",
      "created_at": "2026-10-05T13:30:00Z"
    },
    {
      "id": "pay_01JA8V5H7J9L1N3P5R7T9V1X3Z",
      "status": "settled",
      "amount": 4990,
      "method": "pix",
      "created_at": "2026-10-05T11:14:22Z"
    }
  ],
  "has_more": true,
  "next_cursor": "cur_eyJpZCI6InBheV8wMUpBOFY1SDdKOUwxTjNQNVI3VDlWMVgzWiJ9"
}
```

> [!TIP]
> Para buscar a página seguinte, repita a requisição com `cursor=<next_cursor>`. Pare quando `has_more` for `false`.

## Atualizar um pagamento

`PATCH /payments/{payment_id}`

Altera apenas `description` e `metadata`. Os demais campos não mudam depois que o pagamento é criado.

```bash
curl --request PATCH \
  "https://sandbox.api.finora.example/v3/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{ "metadata": { "order_id": "4821", "canal": "site" } }'
```

A resposta (`200 OK`) devolve o pagamento atualizado.

## Cancelar um pagamento

`DELETE /payments/{payment_id}`

Cancela um pagamento que ainda está com o status `pending`. A resposta (`200 OK`) devolve o pagamento com o status `canceled`.

> [!WARNING]
> Você não pode cancelar um pagamento `settled`. Nesse caso, a API retorna `409` com o código `payment_not_cancelable`. Para devolver o dinheiro ao cliente, faça um estorno.

```json
{
  "error": {
    "code": "payment_not_cancelable",
    "message": "O pagamento já foi liquidado e não pode ser cancelado. Crie um estorno.",
    "param": "payment_id",
    "request_id": "req_01JA8ZD9Q1S3U5W7Y9A1C3E5G7",
    "doc_url": "https://docs.finora.example/errors/payment_not_cancelable"
  }
}
```

## Estornar um pagamento

`POST /payments/{payment_id}/refunds`

Devolve o valor, total ou parcial, de um pagamento `settled`. Você pode fazer vários estornos parciais, desde que a soma não passe do valor do pagamento. Para Pix, o prazo é de até 90 dias após a liquidação.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `amount` | integer | Não | Valor do estorno em centavos. Se omitido, estorna o valor restante. |
| `reason` | string | Não | Motivo: `customer_request`, `duplicate` ou `fraud`. |

```bash
curl --request POST \
  "https://sandbox.api.finora.example/v3/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F/refunds" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: estorno-4821-1" \
  --data '{ "amount": 5000, "reason": "customer_request" }'
```

**Resposta (`201 Created`):**

```json
{
  "id": "ref_01JA8ZF2R4T6V8X0Z2B4D6F8H0",
  "object": "refund",
  "payment_id": "pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F",
  "amount": 5000,
  "reason": "customer_request",
  "status": "pending",
  "created_at": "2026-10-05T14:02:10Z"
}
```

## Idempotência

Redes falham. Se você não recebe a resposta de um `POST`, repetir a requisição pode criar um pagamento duplicado. A **idempotência** resolve isso: com a mesma `Idempotency-Key`, a Finora devolve o resultado da primeira tentativa em vez de criar outro pagamento.

- Envie `Idempotency-Key` em todo `POST` que cria ou altera dinheiro.
- Use uma chave nova para cada operação diferente e a **mesma chave** para repetir a mesma operação.
- A chave vale por 24 horas.
- Se você reutilizar a chave com um **corpo diferente**, a API retorna `409` com o código `idempotency_conflict`.

## Formato dos erros

Todo erro da API `/v3` tem o mesmo formato:

```json
{
  "error": {
    "code": "invalid_amount",
    "message": "O valor mínimo de um pagamento é 100 centavos (R$ 1,00).",
    "param": "amount",
    "request_id": "req_01JA8ZH4T6V8X0Z2B4D6F8H0J2",
    "doc_url": "https://docs.finora.example/errors/invalid_amount"
  }
}
```

| Campo | Descrição |
|---|---|
| `code` | Identificador estável do erro. Use-o no código para decidir o que fazer. |
| `message` | Explicação em linguagem natural. Pode mudar, então não dependa do texto. |
| `param` | O campo que causou o erro, quando houver |
| `request_id` | Identificador da requisição. Informe-o ao suporte. |
| `doc_url` | Link para a explicação do erro |

### Códigos HTTP

| HTTP | Significado | Repetir a requisição? |
|---|---|---|
| 200 | Sucesso | Não é necessário |
| 201 | Recurso criado | Não é necessário |
| 400 | Requisição malformada | Não. Corrija o corpo. |
| 401 | Não autenticado | Sim, depois de renovar o token |
| 403 | Sem permissão (escopo) | Não. Peça o escopo correto. |
| 404 | Recurso não encontrado | Não. Confira o ID. |
| 409 | Conflito de estado ou de idempotência | Não. Corrija a causa. |
| 422 | Dados inválidos | Não. Corrija os dados. |
| 429 | Limite de requisições excedido | Sim, após o tempo do cabeçalho `Retry-After` |
| 500 | Erro interno da Finora | Sim, com a mesma `Idempotency-Key` |
| 503 | Serviço indisponível | Sim, com espera crescente entre as tentativas |

> [!TIP]
> Ao receber `500` ou `503`, espere um pouco e tente de novo com a **mesma** `Idempotency-Key`. Aumente a espera a cada tentativa (1 s, 2 s, 4 s). Isso evita cobranças duplicadas e não sobrecarrega o serviço.

## Dados de teste no sandbox

Use estes valores para simular cenários sem dinheiro real.

| `card_token` | Resultado |
|---|---|
| `tok_test_approved` | Pagamento aprovado |
| `tok_test_declined` | Recusado com `card_declined` |
| `tok_test_insufficient` | Recusado por limite insuficiente |

Para Pix e boleto, simule o evento com `POST /sandbox/payments/{payment_id}/simulate` e o corpo `{ "event": "settled" }` ou `{ "event": "failed" }`.

## Próximos passos

- Receba os eventos de status em [Webhooks](06-webhooks.md).
- Confira o extrato com os pagamentos em [API de Conciliação](04-api-conciliacao.md).
- Resolva problemas comuns em [Troubleshooting](07-troubleshooting.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** desenvolvedores que já fizeram a primeira integração e agora consultam a referência no dia a dia.
- **Estrutura repetida em cada endpoint:** método e caminho, parâmetros, requisição, resposta, erros. O leitor aprende o padrão uma vez e encontra qualquer informação rapidamente.
- **Tabela de endpoints no início** para dar a visão geral antes dos detalhes.
- **Estados do pagamento ilustrados** com um diagrama, porque o texto sozinho obrigaria o leitor a montar a máquina de estados mentalmente.
- **Idempotência e formato dos erros** em seções próprias, pois valem para toda a API. Se ficassem em cada endpoint, haveria repetição.
- **Coluna "Repetir a requisição?"** na tabela de códigos HTTP, que responde à dúvida prática mais comum diante de um erro.
</details>
