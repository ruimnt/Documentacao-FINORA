# Troubleshooting

| | |
|---|---|
| **Público** | Engenheiros de suporte, de plantão e desenvolvedores que investigam um erro |
| **Pré-requisitos** | Acesso ao log da sua integração e ao console da Finora |
| **Tempo de leitura** | 12 minutos |
| **Versão da API** | v3 |

Este guia ajuda você a descobrir o que deu errado e como corrigir. Ele está organizado pelo **sintoma que você observa**, não pela parte interna do sistema. Cada problema segue o mesmo modelo: sintoma, causa provável, como resolver e como prevenir.

## Diagnóstico rápido em 4 passos

Antes de procurar o seu caso, siga estes passos. Eles resolvem a maioria dos problemas.

1. **Leia o código HTTP.** `4xx` indica um problema na requisição. `5xx` indica um problema do lado da Finora.
2. **Leia o campo `error.code` da resposta.** Ele é a forma mais precisa de identificar o erro. Procure-o na [tabela de erros](#tabela-de-erros).
3. **Guarde o `request_id`.** Ele está no corpo do erro e no cabeçalho `X-Request-Id`. O suporte da Finora localiza sua requisição por ele.
4. **Reproduza no sandbox.** Se o erro acontece em produção, tente repetir a requisição no sandbox com dados de teste.

> [!TIP]
> Registre o `request_id` em todos os logs da sua integração, inclusive nas respostas de sucesso. Quando um problema surgir, você já terá o identificador.

## Tabela de erros

| Código | HTTP | O que significa | Onde resolver |
|---|---|---|---|
| `invalid_request` | 400 | O corpo da requisição não é um JSON válido | Valide o JSON antes de enviar |
| `invalid_client` | 401 | Credenciais incorretas em `/oauth/token` | [Autenticação](02-autenticacao.md#erros-de-autenticação) |
| `invalid_token` | 401 | Token ausente, malformado ou adulterado | [Token expirado ou inválido](#token-expirado-ou-inválido) |
| `token_expired` | 401 | O token passou de 1 hora | [Token expirado ou inválido](#token-expirado-ou-inválido) |
| `insufficient_scope` | 403 | O token não tem o escopo exigido | [Escopo insuficiente](#escopo-insuficiente) |
| `not_found` | 404 | O recurso não existe | Confira o ID e o ambiente |
| `idempotency_conflict` | 409 | A chave foi usada com outro corpo | [Conflito de idempotência](#conflito-de-idempotência) |
| `payment_not_cancelable` | 409 | O pagamento já foi liquidado | Faça um [estorno](03-api-pagamentos.md#estornar-um-pagamento) |
| `card_blocked` | 409 | O cartão está bloqueado | Desbloqueie com `PATCH /cards/{card_id}` |
| `duplicate_entry` | 409 | O lançamento já foi enviado | Envie só lançamentos novos |
| `invalid_amount` | 422 | O valor é inválido | [Valor inválido](#valor-inválido-reais-em-vez-de-centavos) |
| `invalid_document` | 422 | CPF ou CNPJ inválido | Confira o documento do pagador |
| `invalid_entry` | 422 | Lançamento de extrato inválido | Corrija o item indicado em `param` |
| `statement_too_large` | 422 | O extrato tem mais de 1.000 lançamentos | Divida em lotes |
| `card_declined` | 422 | O emissor recusou o cartão | [Cartão recusado no sandbox](#cartão-recusado-no-sandbox) |
| `insufficient_funds` | 422 | Saldo insuficiente | Confira o saldo da conta |
| `quote_expired` | 422 | A cotação de cripto expirou | Peça uma nova cotação |
| `below_minimum_amount` | 422 | Valor abaixo do mínimo do produto | Use um valor maior |
| `limit_exceeded` | 422 | O limite pedido passa do limite do titular | Peça um limite menor |
| `invalid_pair` | 422 | Par de cripto não aceito | Use `BTC-BRL`, `ETH-BRL` ou `SOL-BRL` |
| `rate_limited` | 429 | Muitas requisições por minuto | [Limite de requisições](#limite-de-requisições-excedido) |
| `internal_error` | 500 | Erro interno da Finora | Repita com a mesma `Idempotency-Key` |
| `service_unavailable` | 503 | Serviço temporariamente indisponível | Repita com espera crescente |

## Problemas frequentes

### Token expirado ou inválido

**Sintoma:** a API responde `401 Unauthorized` com `token_expired` ou `invalid_token`. A integração funcionava e parou depois de algum tempo.

**Causa provável:** o `access_token` vale por uma hora. Depois disso, a API recusa o token. O erro `invalid_token` também aparece se o cabeçalho `Authorization` estiver incompleto ou se o token for de outro ambiente.

**Como resolver:**

1. Confira o formato do cabeçalho: `Authorization: Bearer <access_token>`, com a palavra `Bearer` e um espaço.
2. Confira se o token e a URL são do mesmo ambiente (sandbox ou produção).
3. Peça um novo token em `POST /oauth/token`:

```bash
curl --request POST "https://sandbox.api.finora.example/oauth/token" \
  --header "Content-Type: application/json" \
  --data "{
    \"grant_type\": \"client_credentials\",
    \"client_id\": \"$FINORA_CLIENT_ID\",
    \"client_secret\": \"$FINORA_CLIENT_SECRET\"
  }"
```

**Como prevenir:** renove o token cerca de 60 segundos antes de expirar. O guia de [Autenticação](02-autenticacao.md#renove-o-token-antes-de-expirar) traz um exemplo pronto.

### Escopo insuficiente

**Sintoma:** a API responde `403 Forbidden` com `insufficient_scope`, embora o token seja válido.

**Causa provável:** o token foi emitido sem o escopo que o endpoint exige. Por exemplo, `POST /payments` exige `payments:write`, e o token só tinha `payments:read`.

**Como resolver:**

1. Veja na [tabela de escopos](02-autenticacao.md#escopos) qual escopo o endpoint exige.
2. Confirme se a credencial no portal tem esse escopo habilitado.
3. Peça um novo token incluindo o escopo no campo `scope`.

**Como prevenir:** liste os escopos de cada integração na sua documentação interna e peça apenas os necessários.

### Conflito de idempotência

**Sintoma:** a API responde `409 Conflict` com `idempotency_conflict`:

```json
{
  "error": {
    "code": "idempotency_conflict",
    "message": "A Idempotency-Key já foi usada com um corpo diferente.",
    "param": "Idempotency-Key",
    "request_id": "req_01JA8ZQ2Z4B6D8F0H2J4L6N8P0",
    "doc_url": "https://docs.finora.example/errors/idempotency_conflict"
  }
}
```

**Causa provável:** você reutilizou uma `Idempotency-Key` em uma operação **diferente** (outro valor, outro cliente). A chave só deve se repetir quando você repete exatamente a mesma operação.

**Como resolver:**

1. Se você quer repetir a mesma operação (por exemplo, depois de um erro de rede), envie o corpo idêntico ao da primeira tentativa.
2. Se é uma operação nova, gere uma chave nova.

**Como prevenir:** monte a chave a partir de dados que identificam a operação, como `pedido-4821-tentativa-1`. Não use uma chave fixa ou o horário atual.

### Valor inválido: reais em vez de centavos

**Sintoma:** a API responde `422 Unprocessable Entity` com `invalid_amount`, ou o pagamento é criado com um valor 100 vezes menor que o esperado.

**Causa provável:** o campo `amount` deve ser um **inteiro em centavos**. Enviar `150.00` ou `150` (pensando em reais) está errado.

Requisição incorreta:

```json
{
  "amount": 150.00,
  "currency": "BRL",
  "method": "pix"
}
```

Requisição correta para R$ 150,00:

```json
{
  "amount": 15000,
  "currency": "BRL",
  "method": "pix"
}
```

> [!WARNING]
> Se você enviar `150` por engano, a API aceita o valor como R$ 1,50. Revise a conversão em todo código que monta `amount`.

**Como resolver:** converta o valor para centavos antes de enviar. Use aritmética decimal, não número de ponto flutuante (`1.15 * 100` resulta em `114.99999999999999` em Python).

**Como prevenir:** centralize a conversão em uma função única e cubra-a com testes.

### Limite de requisições excedido

**Sintoma:** a API responde `429 Too Many Requests` com `rate_limited`.

**Causa provável:** sua credencial passou de 600 requisições por minuto.

**Como resolver:**

1. Leia o cabeçalho `Retry-After`, que indica quantos segundos esperar.
2. Repita a requisição depois desse tempo.

```python
import time

import requests


def get_com_repeticao(url: str, headers: dict, max_tentativas: int = 5) -> requests.Response:
    for tentativa in range(max_tentativas):
        resposta = requests.get(url, headers=headers, timeout=10)
        if resposta.status_code != 429:
            return resposta
        # Usa o Retry-After; se faltar, espera 1 s, 2 s, 4 s...
        espera = int(resposta.headers.get("Retry-After", 2 ** tentativa))
        time.sleep(espera)
    return resposta
```

**Como prevenir:**

- Use [webhooks](06-webhooks.md) em vez de consultar o status repetidamente.
- Reutilize o token em vez de pedir um novo a cada chamada.
- Acompanhe `X-RateLimit-Remaining` e reduza o ritmo quando o número estiver baixo.

### Pix fica pendente por muito tempo

**Sintoma:** um pagamento Pix continua `pending` e nunca vira `settled`.

**Causa provável:** depende do ambiente.

| Ambiente | Causa |
|---|---|
| **Sandbox** | Ninguém paga o Pix de teste. É preciso simular o pagamento. |
| **Produção** | O cliente ainda não pagou, ou o Pix expirou (`pix.expires_at`). |

**Como resolver:**

- No **sandbox**, simule o pagamento:

```bash
curl --request POST \
  "https://sandbox.api.finora.example/v3/sandbox/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F/simulate" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{ "event": "settled" }'
```

- Em **produção**, confira `pix.expires_at`. Depois desse horário, o pagamento passa para `failed`. Crie um novo pagamento e envie o código ao cliente.

**Como prevenir:** assine o evento `payment.settled` e `payment.failed` e atualize o pedido a partir dos webhooks.

### Webhook não chega ou a assinatura não confere

**Sintoma:** seu sistema não recebe eventos, ou o receptor recusa os eventos com "assinatura inválida".

**Causa provável e como resolver**, na ordem em que costuma acontecer:

| # | Verifique | Como conferir ou corrigir |
|---|---|---|
| 1 | O endpoint está ativo | `GET /webhooks`: o `status` deve ser `enabled`. Se estiver `disabled`, corrija o endpoint e cadastre-o de novo. |
| 2 | A URL é pública e usa HTTPS válido | Teste a URL de fora da sua rede. Certificados vencidos ou autoassinados são recusados. |
| 3 | O endpoint responde `2xx` em até 10 segundos | Veja as tentativas e os códigos de resposta no console, em **Desenvolvedores > Webhooks**. |
| 4 | O evento foi assinado em `events` | Confirme que o tipo (por exemplo, `payment.settled`) está na lista do endpoint. |
| 5 | O `secret` pertence a este endpoint e a este ambiente | Cada endpoint tem um segredo próprio. Segredo de sandbox não vale em produção. |
| 6 | Você valida o **corpo bruto** | Se o framework converte o JSON antes da validação, a assinatura não bate. Veja os [exemplos de validação](06-webhooks.md#valide-a-assinatura). |
| 7 | O relógio do servidor está correto | A Finora recusa a assinatura se `t` difere em mais de 5 minutos do seu horário. Sincronize o servidor (NTP). |

Para testar, envie um evento de teste:

```bash
curl --request POST \
  "https://sandbox.api.finora.example/v3/webhooks/whk_01JA8ZM8X0Z2B4D6F8H0J2L4N6/test" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

**Como prevenir:** monitore a taxa de respostas `2xx` do seu endpoint e configure um alerta para falhas seguidas.

### Conciliação com divergência de centavos

**Sintoma:** a API de Conciliação marca lançamentos como `divergent`, com diferenças pequenas, como R$ 1,25.

**Causa provável:** o valor creditado pelo banco difere do `net_amount` esperado. As causas mais comuns são:

| Causa | Como reconhecer | O que fazer |
|---|---|---|
| Arredondamento nas parcelas | A diferença é de poucos centavos e o pagamento é de cartão parcelado | Confirme a correspondência com uma nota explicando a causa |
| Tarifa de antecipação | A diferença é maior e há antecipação de recebíveis no contrato | Compare com a tarifa do contrato antes de confirmar |
| Estorno parcial | O pagamento tem estornos (`refunded`) | Some os estornos ao valor esperado |
| Data fora da janela | O lançamento está a mais de 1 dia útil do `settled_at` | Revise o lançamento manualmente |

**Como resolver:**

1. Liste as divergências do período:

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/reconciliation/matches?status=divergent&date_from=2026-10-01&date_to=2026-10-05" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

2. Compare `amount` (o que o banco creditou) com `expected_amount` e veja `difference`, em centavos.
3. Identifique a causa na tabela acima.
4. Confirme a correspondência com `POST /reconciliation/matches/{match_id}/confirm` e registre a causa em `note`.

**Como prevenir:** concilie todos os dias, não só no fechamento do mês. Divergências pequenas e recentes são mais fáceis de explicar.

### Cartão recusado no sandbox

**Sintoma:** um pagamento de cartão falha com `card_declined` durante os testes.

**Causa provável:** você usou o token `tok_test_declined` ou `tok_test_insufficient`, que simulam recusas de propósito. Também pode ter usado um token que não é de teste.

**Como resolver:** use `tok_test_approved` para simular um pagamento aprovado. A tabela completa está em [Dados de teste no sandbox](03-api-pagamentos.md#dados-de-teste-no-sandbox).

## Quando falar com o suporte

Abra um chamado quando você já seguiu o diagnóstico rápido e o problema continua, ou quando receber `500` ou `503` por mais de 10 minutos.

Inclua no chamado:

- [ ] O `request_id` (ou vários, se o problema se repete)
- [ ] O horário aproximado, com fuso horário
- [ ] O endpoint e o método HTTP (por exemplo, `POST /v3/payments`)
- [ ] O ambiente (sandbox ou produção)
- [ ] O corpo da requisição, **sem** o `client_secret`, o token e dados pessoais completos
- [ ] O que você esperava que acontecesse e o que aconteceu

> [!WARNING]
> Nunca envie ao suporte o `client_secret`, o `access_token` ou o `secret` de um webhook. A equipe da Finora nunca pede esses dados. Se você os expôs, rotacione-os no portal imediatamente.

## Próximos passos

- Conheça o formato completo dos erros em [API de Pagamentos](https://github.com/ruimnt/Documentacao-FINORA/blob/main/03%20-%20API%20Pagamentos.md).
- Consulte termos desconhecidos no [Glossário](10-glossario-e-faq.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** quem está sob pressão. O engenheiro de plantão não lê, ele procura. Por isso a página prioriza a busca rápida.
- **Organizado pelo sintoma,** não pelo código interno. A tabela de erros faz a ponte: o leitor que parte do código chega à seção certa, e quem parte do sintoma encontra o título.
- **Modelo fixo para cada problema** (sintoma, causa, solução, prevenção). A repetição permite que o leitor localize a solução com os olhos, sem ler tudo.
- **Diagnóstico em 4 passos antes dos casos,** porque resolve a maioria dos problemas e reduz a quantidade de leitura.
- **Seção de suporte com lista de itens** para o chamado chegar completo na primeira vez, e um aviso de segurança sobre o que nunca enviar.
</details>
