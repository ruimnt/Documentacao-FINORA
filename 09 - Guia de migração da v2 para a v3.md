# Guia de migração da v2 para a v3

| | |
|---|---|
| **Público** | Desenvolvedores que mantêm uma integração com a v2 e gerentes que planejam a migração |
| **Pré-requisitos** | Uma integração em funcionamento com a v2 e uma conta no sandbox da v3 |
| **Tempo de leitura** | 15 minutos |
| **Tempo estimado de migração** | De 1 a 3 semanas, conforme o tamanho da integração |
| **Versão da API** | v2 para v3 |

Este guia mostra, passo a passo, como migrar sua integração da v2 para a v3 sem interromper os pagamentos. Para saber **o que mudou**, veja o [changelog](08-changelog.md). Aqui você encontra **o que fazer**.

> [!WARNING]
> A v2 será descontinuada em **16 de junho de 2027**. Recomendamos concluir a migração até **31 de março de 2027**, para ter margem em caso de imprevistos.

## O que muda, em uma tabela

| Área | v2 | v3 | Impacto |
|---|---|---|---|
| Autenticação | `X-Api-Key` | OAuth 2.0 (`/oauth/token`) | Alto |
| Valores | `150.00` (reais, decimal) | `15000` (centavos, inteiro) | Alto |
| Endpoint | `/v2/transactions` | `/v3/payments` | Médio |
| Campos do pagador | `payer.cpf` | `customer.document` | Médio |
| Paginação | `page` e `per_page` | `cursor` e `limit` | Médio |
| Erros | `code` numérico | `code` em texto, `request_id` | Médio |
| Webhooks | Token estático | Assinatura HMAC-SHA256 | Alto |

## Plano de migração

Siga os passos na ordem. Cada um pode ser entregue e testado separadamente.

### Passo 1: faça o inventário

Liste todos os pontos do seu código que falam com a Finora. Procure por estes termos:

```bash
grep -rn "api.finora.example/v2\|X-Api-Key\|X-Finora-Token\|/v2/transactions" ./src
```

Anote cada arquivo encontrado. Esse é o escopo da migração.

### Passo 2: troque a autenticação

1. No portal, crie uma credencial OAuth (`client_id` e `client_secret`) para a v3.
2. Implemente a obtenção e a renovação do token, como mostra o guia de [Autenticação](02-autenticacao.md#renove-o-token-antes-de-expirar).
3. Substitua o cabeçalho `X-Api-Key` por `Authorization: Bearer <access_token>`.

Antes (v2):

```bash
curl --request GET "https://api.finora.example/v2/transactions/txn_8f3a2c" \
  --header "X-Api-Key: fin_live_xxxxxxxxxxxxxxxx"
```

Depois (v3):

```bash
curl --request GET "https://api.finora.example/v3/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

> [!NOTE]
> As chaves `X-Api-Key` continuam funcionando na v2 até o encerramento. Você pode manter as duas autenticações durante a migração.

### Passo 3: troque os endpoints e os campos

Antes (v2), criação de um pagamento Pix:

```json
{
  "amount": 150.00,
  "type": "pix",
  "reference": "4821",
  "payer": {
    "name": "Marina Souza",
    "cpf": "123.456.789-09",
    "email": "marina.souza@example.com"
  }
}
```

Depois (v3):

```json
{
  "amount": 15000,
  "currency": "BRL",
  "method": "pix",
  "description": "Pedido #4821 - Livraria Boa Página",
  "customer": {
    "name": "Marina Souza",
    "document": "123.456.789-09",
    "email": "marina.souza@example.com"
  },
  "metadata": {
    "order_id": "4821"
  }
}
```

Mapa de campos:

| v2 | v3 | Observação |
|---|---|---|
| `POST /v2/transactions` | `POST /v3/payments` | Envie também o cabeçalho `Idempotency-Key` |
| `amount` | `amount` | Mudou de reais para centavos (passo 4) |
| `type` | `method` | Valores: `pix`, `boleto`, `card` |
| `payer` | `customer` | |
| `payer.cpf` | `customer.document` | Aceita CPF e CNPJ |
| `reference` | `metadata.order_id` | Use `metadata` para qualquer dado seu |
| (não existia) | `currency` | Obrigatório. Use `BRL`. |

Mapa de status:

| Status na v2 | Status na v3 |
|---|---|
| `waiting` | `pending` |
| `approved` | `settled` |
| `refused` | `failed` |
| `canceled` | `canceled` |
| `refunded` | `refunded` |

### Passo 4: converta os valores para centavos

Esta é a mudança mais perigosa. Se você enviar `150.00` à v3, a API rejeita a requisição. Se enviar `150`, ela aceita o valor como R$ 1,50.

> [!WARNING]
> Não converta com números de ponto flutuante: `1.15 * 100` resulta em `114.99999999999999` em Python. Use aritmética decimal, como no exemplo abaixo.

```python
from decimal import ROUND_HALF_UP, Decimal


def reais_para_centavos(valor: str) -> int:
    """Converte '150.00' em 15000, sem erros de arredondamento."""
    return int((Decimal(valor) * 100).quantize(Decimal("1"), rounding=ROUND_HALF_UP))


def centavos_para_reais(centavos: int) -> str:
    """Converte 15000 em '150.00', para exibir ao usuário."""
    return f"{Decimal(centavos) / 100:.2f}"


STATUS_V2_PARA_V3 = {
    "waiting": "pending",
    "approved": "settled",
    "refused": "failed",
    "canceled": "canceled",
    "refunded": "refunded",
}

assert reais_para_centavos("150.00") == 15000
assert reais_para_centavos("1.15") == 115
assert centavos_para_reais(15000) == "150.00"
```

Revise também os dados que **você guarda**: se o seu banco de dados armazena os valores da v2 em reais, decida se vai convertê-los ou manter os dois formatos com um nome de coluna claro (por exemplo, `valor_centavos`).

### Passo 5: atualize a paginação

Antes (v2):

```bash
curl --request GET "https://api.finora.example/v2/transactions?page=2&per_page=50" \
  --header "X-Api-Key: fin_live_xxxxxxxxxxxxxxxx"
```

```json
{
  "total": 312,
  "page": 2,
  "per_page": 50,
  "items": []
}
```

Depois (v3):

```bash
curl --request GET "https://api.finora.example/v3/payments?limit=50&cursor=cur_eyJpZCI6InBheV8wMUpBOFY1SDdKOUwxTjNQNVI3VDlWMVgzWiJ9" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

```json
{
  "object": "list",
  "data": [],
  "has_more": true,
  "next_cursor": "cur_eyJpZCI6InBheV8wMUpBOFY0RzZIOEs5TTFRM1M1VTdXOVkifQ"
}
```

Três diferenças importantes:

- A lista fica em `data` (antes, em `items`).
- A v3 **não informa o total**. Continue buscando enquanto `has_more` for `true`.
- Para ir à página seguinte, envie o `next_cursor` no parâmetro `cursor`.

### Passo 6: atualize o tratamento de erros

Antes (v2):

```json
{
  "status": "error",
  "message": "valor invalido",
  "code": 4022
}
```

Depois (v3):

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

Ajuste o seu código para:

1. Ler `error.code` (texto) em vez de `code` (número).
2. Registrar `error.request_id` nos logs.
3. Decidir se repete a requisição com base no [código HTTP](03-api-pagamentos.md#códigos-http).

### Passo 7: migre os webhooks

A v2 identificava a Finora com um token fixo. A v3 assina cada evento, o que é mais seguro.

| | v2 | v3 |
|---|---|---|
| Cabeçalho | `X-Finora-Token: <token estático>` | `Finora-Signature: t=...,v1=...` |
| Validação | Comparar o token com um valor guardado | Calcular o HMAC-SHA256 do corpo bruto |
| Evento de pagamento aprovado | `transaction.approved` | `payment.settled` |
| Evento de pagamento recusado | `transaction.refused` | `payment.failed` |
| Evento de estorno | `transaction.refunded` | `refund.created` |

Para implementar a validação, siga os exemplos em [Webhooks](https://github.com/ruimnt/Documentacao-FINORA/blob/main/06%20-%20Webhooks.md). Você pode cadastrar o novo endpoint da v3 ao lado do antigo e receber os dois durante a transição.

### Passo 8: teste no sandbox

Antes de ir para produção, confirme cada item no sandbox da v3:

- [ ] Obtive e renovei o token sem erros `401`
- [ ] Criei pagamentos Pix, boleto e cartão com `amount` em centavos
- [ ] Os valores exibidos ao usuário estão corretos (R$ 150,00 e não R$ 1,50)
- [ ] Percorri todas as páginas de uma listagem usando `cursor`
- [ ] Meu sistema trata `error.code` em texto e registra o `request_id`
- [ ] Meu receptor valida a assinatura dos webhooks e recusa assinaturas inválidas
- [ ] Simulei `settled`, `failed` e estorno e o sistema atualizou os pedidos
- [ ] Repeti uma criação com a mesma `Idempotency-Key` e não houve pagamento duplicado

### Passo 9: faça a virada gradual

Não troque tudo de uma vez. Faça a virada em etapas, com a possibilidade de voltar atrás:

| Etapa | Tráfego na v3 | O que observar |
|---|---|---|
| 1 | 5% dos pagamentos | Erros `4xx` e `5xx`, valores criados |
| 2 | 25% | Divergências na [conciliação](04-api-conciliacao.md) |
| 3 | 100% | Todos os indicadores por pelo menos uma semana |

> [!TIP]
> Use uma chave de funcionalidade (*feature flag*) para decidir, a cada pedido, se a chamada vai para a v2 ou para a v3. Se algo der errado, você desliga a chave e volta à v2 em segundos, sem uma nova publicação.

Depois da etapa 3, remova o código da v2 e revogue as chaves `X-Api-Key` no portal.

## Perguntas frequentes sobre a migração

**Posso usar a v2 e a v3 ao mesmo tempo?**
Sim. As duas funcionam em paralelo até 16/06/2027. Os pagamentos criados em uma versão podem ser consultados na outra, mas o formato da resposta segue a versão que você chamou.

**Preciso migrar os pagamentos antigos?**
Não. Os pagamentos já criados continuam consultáveis pela v3, com os novos campos e status.

**O que acontece com os webhooks cadastrados na v2?**
Eles continuam sendo enviados até o encerramento da v2. Cadastre os novos endpoints antes dessa data.

## Próximos passos

- Se algo falhar durante a migração, consulte o [troubleshooting](07-troubleshooting.md).
- Veja todos os termos usados neste guia no [Glossário](10-glossario-e-faq.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** desenvolvedores que já conhecem a v2 e querem saber o que fazer, e gerentes que precisam estimar prazo e risco. Por isso o guia abre com o tempo estimado, a data limite e uma tabela de impacto.
- **Plano em passos numerados e ordenados do mais estrutural ao mais detalhado:** autenticação, endpoints, valores, paginação, erros, webhooks. Cada passo pode ser entregue e testado sozinho.
- **Antes e depois lado a lado.** Quem migra compara o código antigo com o novo. Mostrar os dois no mesmo trecho evita ter de abrir o changelog.
- **Destaque para o risco financeiro** (centavos em vez de reais), com um aviso e um código seguro, porque é o erro que gera cobrança errada.
- **Estratégia de virada gradual e plano de retorno.** Um guia de migração que ensina só a trocar o código ignora o que mais preocupa o gerente: como fazer isso sem derrubar a operação.
</details>
