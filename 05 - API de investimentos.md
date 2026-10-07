# API de Investimentos, Cripto e Cartão

| | |
|---|---|
| **Público** | Desenvolvedores que integram os produtos financeiros da Finora ao seu aplicativo ou sistema |
| **Pré-requisitos** | Token com os escopos de cada produto (`investments:*`, `crypto:*`, `cards:*`). Veja [Autenticação](02-autenticacao.md). |
| **Tempo de leitura** | 14 minutos |
| **Versão da API** | v3 |

Esta página reúne as três APIs de produtos financeiros da Finora. Todas seguem as mesmas convenções da [API de Pagamentos](03-api-pagamentos.md): valores em centavos, datas em ISO 8601, paginação por cursor e erros no mesmo formato.

[Área do cliente da Finora com os painéis de cartão de crédito, investimentos e cripto]

<img width="960" height="580" alt="Image" src="https://github.com/user-attachments/assets/e3873c53-1d7a-4008-b04b-7afecf386682" />

> [!NOTE]
> Quantidades de criptoativos e percentuais são enviados como **texto** (`"0.01200000"`), não como número. Números decimais perdem precisão em algumas linguagens e, em finanças, um arredondamento errado vira diferença de dinheiro.

## Investimentos

| Método | Endpoint | O que faz | Escopo |
|---|---|---|---|
| `GET` | `/investments/products` | Lista os produtos disponíveis | `investments:read` |
| `POST` | `/investments/orders` | Cria uma ordem de aplicação | `investments:write` |
| `GET` | `/investments/positions` | Lista as posições do cliente | `investments:read` |

> [!WARNING]
> Os produtos e as taxas desta documentação são fictícios. Em um produto real, informe ao cliente os riscos e use a frase: "Rentabilidade passada não garante rentabilidade futura". Este material não é recomendação de investimento.

### Listar produtos

`GET /investments/products`

| Parâmetro (query) | Tipo | Descrição |
|---|---|---|
| `type` | string | `fixed_income` (renda fixa), `treasury` (tesouro) ou `fund` (fundos) |
| `limit` | integer | Itens por página, de `1` a `100`. Padrão: `20`. |
| `cursor` | string | Valor de `next_cursor` da página anterior |

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/investments/products?type=fixed_income&limit=1" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`):**

```json
{
  "object": "list",
  "data": [
    {
      "id": "prd_cdb_finora_110",
      "object": "investment_product",
      "name": "CDB Finora 110% CDI",
      "type": "fixed_income",
      "index": "CDI",
      "rate_percent": "110.00",
      "min_amount": 10000,
      "liquidity": "D+0",
      "maturity_date": null,
      "risk_level": "low"
    }
  ],
  "has_more": true,
  "next_cursor": "cur_eyJpZCI6InByZF9jZGJfZmlub3JhXzExMCJ9"
}
```

### Criar uma ordem de aplicação

`POST /investments/orders`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `account_id` | string | Sim | Conta de onde sai o dinheiro |
| `product_id` | string | Sim | ID do produto, obtido em `/investments/products` |
| `amount` | integer | Sim | Valor em centavos. Não pode ser menor que `min_amount` do produto. |

```bash
curl --request POST "https://sandbox.api.finora.example/v3/investments/orders" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: aplicacao-marina-001" \
  --data '{
    "account_id": "acc_01JA7Q2M4P6R8T0V2X4Z6B8D0F",
    "product_id": "prd_cdb_finora_110",
    "amount": 500000
  }'
```

**Resposta (`201 Created`):**

```json
{
  "id": "ord_01JA9D2K4M6P8R0T2V4X6Z8B0D",
  "object": "investment_order",
  "status": "processing",
  "account_id": "acc_01JA7Q2M4P6R8T0V2X4Z6B8D0F",
  "product_id": "prd_cdb_finora_110",
  "amount": 500000,
  "created_at": "2026-10-05T14:20:00Z"
}
```

### Consultar posições

`GET /investments/positions`

```json
{
  "object": "list",
  "data": [
    {
      "product_id": "prd_cdb_finora_110",
      "name": "CDB Finora 110% CDI",
      "current_value": 1000000,
      "return_month_percent": "0.91"
    },
    {
      "product_id": "prd_tesouro_selic_2029",
      "name": "Tesouro Selic 2029",
      "current_value": 620000,
      "return_month_percent": "0.84"
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

### Erros de investimentos

| HTTP | `code` | Quando acontece | Como resolver |
|---|---|---|---|
| 422 | `below_minimum_amount` | O valor é menor que o mínimo do produto | Envie um valor igual ou maior que `min_amount` |
| 422 | `insufficient_funds` | A conta não tem saldo para a aplicação | Confira o saldo antes de enviar |
| 404 | `not_found` | O produto ou a conta não existe | Confira os IDs |

## Cripto

| Método | Endpoint | O que faz | Escopo |
|---|---|---|---|
| `GET` | `/crypto/quotes` | Obtém uma cotação com validade curta | `crypto:read` |
| `POST` | `/crypto/orders` | Compra ou vende usando uma cotação | `crypto:write` |
| `GET` | `/crypto/wallets` | Lista os saldos por criptoativo | `crypto:read` |

### Como funciona uma ordem

1. Peça uma **cotação**. Ela vale por 15 segundos.
2. Crie a **ordem** enviando o `quote_id` da cotação.
3. A Finora executa a ordem pelo preço da cotação, mesmo que o mercado tenha mudado nesse intervalo.

> [!WARNING]
> Criptoativos são voláteis e podem gerar perdas. Mostre o preço e o prazo de validade da cotação ao cliente antes de confirmar a ordem.

### Obter uma cotação

`GET /crypto/quotes?pair=BTC-BRL`

| Parâmetro (query) | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `pair` | string | Sim | Par de negociação: `BTC-BRL`, `ETH-BRL` ou `SOL-BRL` |

```bash
curl --request GET "https://sandbox.api.finora.example/v3/crypto/quotes?pair=BTC-BRL" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`):**

```json
{
  "id": "qte_01JA9D5M7P9R1T3V5X7Z9B1D3F",
  "object": "crypto_quote",
  "pair": "BTC-BRL",
  "bid": 26946000,
  "ask": 27000000,
  "expires_at": "2026-10-05T14:25:15Z"
}
```

`ask` é o preço de compra e `bid` é o preço de venda, ambos em centavos de real por unidade. Aqui, `27000000` equivale a R$ 270.000,00 por bitcoin.

### Criar uma ordem

`POST /crypto/orders`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `quote_id` | string | Sim | ID de uma cotação ainda válida |
| `side` | string | Sim | `buy` (comprar) ou `sell` (vender) |
| `amount` | integer | Sim | Valor em reais, em centavos |

```bash
curl --request POST "https://sandbox.api.finora.example/v3/crypto/orders" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: cripto-marina-001" \
  --data '{
    "quote_id": "qte_01JA9D5M7P9R1T3V5X7Z9B1D3F",
    "side": "buy",
    "amount": 100000
  }'
```

**Resposta (`201 Created`):**

```json
{
  "id": "cro_01JA9D7P9R1T3V5X7Z9B1D3F5H",
  "object": "crypto_order",
  "status": "filled",
  "pair": "BTC-BRL",
  "side": "buy",
  "amount": 100000,
  "price": 27000000,
  "quantity": "0.00370370",
  "created_at": "2026-10-05T14:25:03Z"
}
```

### Consultar carteiras

`GET /crypto/wallets`

```json
{
  "object": "list",
  "data": [
    {
      "asset": "BTC",
      "balance": "0.01200000",
      "value": 324000
    },
    {
      "asset": "ETH",
      "balance": "0.15000000",
      "value": 67240
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

`balance` é a quantidade do criptoativo (texto) e `value` é o valor equivalente em reais, em centavos.

### Erros de cripto

| HTTP | `code` | Quando acontece | Como resolver |
|---|---|---|---|
| 422 | `quote_expired` | A cotação passou dos 15 segundos | Peça uma nova cotação e crie a ordem em seguida |
| 422 | `insufficient_funds` | Saldo insuficiente para a ordem | Confira o saldo do cliente |
| 422 | `invalid_pair` | O par informado não é aceito | Use `BTC-BRL`, `ETH-BRL` ou `SOL-BRL` |

## Cartão de crédito

| Método | Endpoint | O que faz | Escopo |
|---|---|---|---|
| `GET` | `/cards` | Lista os cartões do cliente | `cards:read` |
| `POST` | `/cards` | Cria um cartão virtual | `cards:write` |
| `PATCH` | `/cards/{card_id}` | Bloqueia, desbloqueia ou altera o limite | `cards:write` |
| `GET` | `/cards/{card_id}/invoices` | Lista as faturas de um cartão | `cards:read` |

### Criar um cartão virtual

`POST /cards`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `type` | string | Sim | Por enquanto, apenas `virtual` |
| `holder_id` | string | Sim | ID do titular cadastrado, por exemplo `cus_01JA7R4N6Q8S0U2W4Y6A8C0E2G` |
| `limit` | integer | Sim | Limite em centavos. Não pode passar do limite total do titular. |
| `label` | string | Não | Apelido do cartão. Até 40 caracteres. |

```bash
curl --request POST "https://sandbox.api.finora.example/v3/cards" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: cartao-virtual-marina-001" \
  --data '{
    "type": "virtual",
    "holder_id": "cus_01JA7R4N6Q8S0U2W4Y6A8C0E2G",
    "limit": 500000,
    "label": "Assinaturas"
  }'
```

**Resposta (`201 Created`):**

```json
{
  "id": "crd_01JA9E4M6P8R0T2V4X6Z8B0D2F",
  "object": "card",
  "type": "virtual",
  "status": "active",
  "label": "Assinaturas",
  "last4": "4821",
  "expires": "2030-08",
  "limit": 500000,
  "limit_used": 0,
  "created_at": "2026-10-05T14:30:00Z"
}
```

> [!NOTE]
> A API nunca devolve o número completo nem o código de segurança do cartão. Para exibir esses dados ao cliente, use o componente seguro do SDK da Finora, que mostra as informações sem que elas passem pelo seu servidor.

### Bloquear um cartão

`PATCH /cards/{card_id}`

Envie apenas os campos que deseja alterar: `status` (`active` ou `blocked`) e `limit`.

```bash
curl --request PATCH \
  "https://sandbox.api.finora.example/v3/cards/crd_01JA9E4M6P8R0T2V4X6Z8B0D2F" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{ "status": "blocked" }'
```

**Resposta (`200 OK`):**

```json
{
  "id": "crd_01JA9E4M6P8R0T2V4X6Z8B0D2F",
  "object": "card",
  "status": "blocked",
  "last4": "4821",
  "limit": 500000,
  "limit_used": 0,
  "updated_at": "2026-10-05T14:41:12Z"
}
```

> [!TIP]
> Ofereça um botão de bloqueio imediato no seu aplicativo. Se o cliente perder o cartão, poucos segundos fazem diferença.

### Consultar faturas

`GET /cards/{card_id}/invoices`

| Parâmetro (query) | Tipo | Descrição |
|---|---|---|
| `status` | string | `open` (em aberto), `closed` (fechada, aguardando pagamento), `paid` ou `overdue` (vencida) |
| `limit` | integer | Itens por página, de `1` a `100`. Padrão: `20`. |
| `cursor` | string | Valor de `next_cursor` da página anterior |

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/cards/crd_01JA9E4M6P8R0T2V4X6Z8B0D2F/invoices?status=closed" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Resposta (`200 OK`):**

```json
{
  "object": "list",
  "data": [
    {
      "id": "inv_01JA9E6R8T0V2X4Z6B8D0F2H4J",
      "object": "invoice",
      "status": "closed",
      "total": 213460,
      "minimum_payment": 32019,
      "closing_date": "2026-10-04",
      "due_date": "2026-10-12"
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

### Erros de cartão

| HTTP | `code` | Quando acontece | Como resolver |
|---|---|---|---|
| 404 | `not_found` | O cartão não existe | Confira o `card_id` |
| 409 | `card_blocked` | A operação exige um cartão ativo | Desbloqueie o cartão com `PATCH` |
| 422 | `limit_exceeded` | O limite pedido ultrapassa o limite total do titular | Envie um valor menor |

## Próximos passos

- Receba avisos de fatura fechada e de ordem executada em [Webhooks](06-webhooks.md).
- Consulte os códigos de erro mais comuns em [Troubleshooting](07-troubleshooting.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** desenvolvedores que já conhecem as convenções da API e querem integrar um produto específico.
- **Três produtos em uma página.** Eles compartilham convenções, e o leitor costuma integrar mais de um. Cada produto tem sua própria seção, com tabela de endpoints no início e erros no fim, para o leitor pular direto ao que precisa.
- **Mockup único** com os três painéis, que ajuda quem não conhece o domínio a entender o que cada API alimenta na tela do cliente.
- **Avisos regulatórios no lugar certo.** O aviso de risco de investimento e o de volatilidade de cripto aparecem logo antes de o leitor criar ordens, porque são decisões que ele precisa comunicar ao cliente final.
- **Nota sobre texto em vez de número** para quantidades e percentuais, explicando a decisão de design da API em vez de deixar o leitor estranhar.
</details>
