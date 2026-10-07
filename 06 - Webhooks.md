# Webhooks

| | |
|---|---|
| **Público** | Desenvolvedores que recebem eventos da Finora e engenheiros de suporte |
| **Pré-requisitos** | Um endpoint HTTPS público no seu sistema. Token com o escopo `webhooks:manage`. |
| **Tempo de leitura** | 10 minutos |
| **Versão da API** | v3 |

Um **webhook** é uma chamada que a Finora faz ao seu sistema quando algo acontece, por exemplo quando um pagamento é liquidado. Com webhooks, você não precisa consultar a API a cada poucos segundos para saber se o status mudou.

## Como funciona

1. Você cadastra a URL do seu endpoint e escolhe os eventos que quer receber.
2. Quando um evento ocorre, a Finora envia um `POST` com os dados em JSON para essa URL.
3. Seu endpoint **valida a assinatura** da requisição e responde com um código `2xx`.
4. Se a resposta não for `2xx`, a Finora tenta de novo, com intervalos crescentes.

Na [arquitetura da plataforma](../README.md#arquitetura-em-uma-imagem), o despachante de webhooks fica a jusante do *event bus* e entrega cada evento ao endpoint do cliente.

## Eventos disponíveis

| Evento | Quando é enviado |
|---|---|
| `payment.created` | Um pagamento foi criado |
| `payment.pending` | Um pagamento aguarda o pagador |
| `payment.authorized` | Um pagamento foi aprovado |
| `payment.settled` | O dinheiro de um pagamento foi liquidado |
| `payment.failed` | Um pagamento foi recusado, expirou ou falhou |
| `payment.canceled` | Um pagamento pendente foi cancelado |
| `refund.created` | Um estorno foi criado |
| `reconciliation.statement.processed` | Um extrato terminou de ser processado |
| `investment.order.completed` | Uma ordem de investimento foi concluída |
| `crypto.order.filled` | Uma ordem de cripto foi executada |
| `card.invoice.closed` | Uma fatura de cartão foi fechada |

## Formato do evento

Todo evento tem a mesma estrutura. O objeto que mudou fica em `data.object`:

```json
{
  "id": "evt_01JA8ZK6V8X0Z2B4D6F8H0J2L4",
  "object": "event",
  "type": "payment.settled",
  "api_version": "v3",
  "created_at": "2026-10-05T13:32:41Z",
  "data": {
    "object": {
      "id": "pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F",
      "object": "payment",
      "status": "settled",
      "amount": 15000,
      "currency": "BRL",
      "method": "pix",
      "net_amount": 15000,
      "metadata": {
        "order_id": "4821"
      },
      "settled_at": "2026-10-05T13:32:41Z"
    }
  }
}
```

## Cadastrar um endpoint

`POST /webhooks`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `url` | string | Sim | URL HTTPS do seu endpoint |
| `events` | array | Sim | Lista de eventos desejados. Use `["*"]` para receber todos. |
| `description` | string | Não | Texto livre para identificar o endpoint |

```bash
curl --request POST "https://sandbox.api.finora.example/v3/webhooks" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "url": "https://loja.boapagina.example/webhooks/finora",
    "events": ["payment.settled", "payment.failed", "refund.created"],
    "description": "Loja Boa Página - pedidos"
  }'
```

**Resposta (`201 Created`):**

```json
{
  "id": "whk_01JA8ZM8X0Z2B4D6F8H0J2L4N6",
  "object": "webhook_endpoint",
  "url": "https://loja.boapagina.example/webhooks/finora",
  "events": ["payment.settled", "payment.failed", "refund.created"],
  "description": "Loja Boa Página - pedidos",
  "status": "enabled",
  "secret": "whsec_7c1e9a4b2d6f8031a5c7e9b1d3f50826",
  "created_at": "2026-10-05T13:40:00Z"
}
```

> [!WARNING]
> O campo `secret` aparece **apenas nesta resposta**. Guarde-o agora em um gerenciador de segredos. Você precisa dele para validar a assinatura dos eventos. Se o perder, crie um novo endpoint.

### Listar e remover endpoints

| Método | Endpoint | O que faz |
|---|---|---|
| `GET` | `/webhooks` | Lista os endpoints cadastrados |
| `POST` | `/webhooks/{webhook_id}/test` | Envia um evento de teste `webhook.test` ao endpoint |
| `DELETE` | `/webhooks/{webhook_id}` | Remove o endpoint. A resposta é `204 No Content`. |

```bash
curl --request DELETE \
  "https://sandbox.api.finora.example/v3/webhooks/whk_01JA8ZM8X0Z2B4D6F8H0J2L4N6" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

## Valide a assinatura

Qualquer pessoa que conheça a URL do seu endpoint pode enviar uma requisição falsa. Por isso, a Finora assina cada evento, e **você deve conferir a assinatura antes de confiar no conteúdo**.

Cada requisição traz o cabeçalho `Finora-Signature`:

```text
Finora-Signature: t=1791207161,v1=a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90
```

- `t` é o momento do envio, em segundos desde 1970 (timestamp Unix).
- `v1` é a assinatura, calculada com HMAC-SHA256.

Para validar:

1. Separe `t` e `v1` do cabeçalho.
2. Monte o texto assinado: o valor de `t`, um ponto e o **corpo bruto** da requisição (`t + "." + corpo`).
3. Calcule o HMAC-SHA256 desse texto usando o `secret` do endpoint.
4. Compare o resultado com `v1` usando uma comparação em tempo constante.
5. Recuse a requisição se `t` for mais antigo que 5 minutos. Isso impede o reenvio de uma requisição interceptada.

> [!WARNING]
> Use o **corpo bruto** (os bytes exatos que chegaram). Se o seu framework converter o JSON e o serializar de novo, o espaçamento muda e a assinatura deixa de bater.

### Exemplo em Python

```python
import hashlib
import hmac
import time


def verificar_assinatura(corpo_bruto: bytes, cabecalho: str, segredo: str, tolerancia: int = 300) -> bool:
    partes = dict(item.split("=", 1) for item in cabecalho.split(","))
    timestamp = partes.get("t")
    recebida = partes.get("v1")
    if not timestamp or not recebida:
        return False

    # Recusa requisições antigas para evitar reenvio (replay).
    if abs(time.time() - int(timestamp)) > tolerancia:
        return False

    texto_assinado = f"{timestamp}.".encode() + corpo_bruto
    esperada = hmac.new(segredo.encode(), texto_assinado, hashlib.sha256).hexdigest()
    return hmac.compare_digest(esperada, recebida)
```

Exemplo de uso com Flask:

```python
import os

from flask import Flask, request

app = Flask(__name__)


@app.post("/webhooks/finora")
def receber_evento():
    corpo_bruto = request.get_data()
    cabecalho = request.headers.get("Finora-Signature", "")

    if not verificar_assinatura(corpo_bruto, cabecalho, os.environ["FINORA_WEBHOOK_SECRET"]):
        return "assinatura inválida", 400

    evento = request.get_json()
    if evento["type"] == "payment.settled":
        pedido = evento["data"]["object"]["metadata"]["order_id"]
        print(f"Pedido {pedido} pago.")

    return "", 200
```

### Exemplo em JavaScript (Node.js)

```javascript
const crypto = require("crypto");
const express = require("express");

function verificarAssinatura(corpoBruto, cabecalho, segredo, tolerancia = 300) {
  const partes = Object.fromEntries(cabecalho.split(",").map((item) => item.split("=", 2)));
  const timestamp = partes.t;
  const recebida = partes.v1;
  if (!timestamp || !recebida) return false;

  // Recusa requisições antigas para evitar reenvio (replay).
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > tolerancia) return false;

  const esperada = crypto
    .createHmac("sha256", segredo)
    .update(`${timestamp}.${corpoBruto}`)
    .digest("hex");

  const a = Buffer.from(esperada);
  const b = Buffer.from(recebida);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

const app = express();

// express.raw preserva o corpo bruto, necessário para validar a assinatura.
app.post("/webhooks/finora", express.raw({ type: "application/json" }), (req, res) => {
  const corpoBruto = req.body.toString("utf8");
  const cabecalho = req.get("Finora-Signature") || "";

  if (!verificarAssinatura(corpoBruto, cabecalho, process.env.FINORA_WEBHOOK_SECRET)) {
    return res.status(400).send("assinatura inválida");
  }

  const evento = JSON.parse(corpoBruto);
  if (evento.type === "payment.settled") {
    console.log(`Pedido ${evento.data.object.metadata.order_id} pago.`);
  }
  res.sendStatus(200);
});

app.listen(3000);
```

## Responda e processe com segurança

- **Responda rápido.** Devolva `2xx` em até 10 segundos. Se o processamento for demorado, guarde o evento em uma fila e responda logo.
- **Trate eventos repetidos.** A Finora garante a entrega *pelo menos uma vez*, então o mesmo evento pode chegar duas vezes. Guarde o `id` de cada evento processado (`evt_...`) e ignore os repetidos.
- **Não dependa da ordem.** Um `payment.settled` pode chegar antes de um `payment.pending` atrasado. Use `created_at` ou consulte o pagamento se a ordem importar.
- **Use HTTPS.** A Finora só envia eventos para URLs com HTTPS válido.

> [!TIP]
> Em caso de dúvida sobre o estado de um pagamento, consulte `GET /payments/{payment_id}`. O webhook avisa que algo mudou, e a API é a fonte da verdade.

## Política de reenvio

Se o seu endpoint não responder com `2xx` em 10 segundos, a Finora tenta novamente, com intervalos crescentes, por até 24 horas:

| Tentativa | Intervalo após a tentativa anterior |
|---|---|
| 1 | Imediata |
| 2 | 1 minuto |
| 3 | 5 minutos |
| 4 | 30 minutos |
| 5 | 1 hora |
| 6 | 2 horas |
| 7 | 4 horas |
| 8 | 8 horas |

Depois da oitava tentativa sem sucesso, a Finora marca o evento como `failed` no console, onde você pode reenviá-lo manualmente. Um endpoint que falha por 3 dias seguidos é desativado (`status: "disabled"`).

## Próximos passos

- Se os eventos não chegam, veja [Webhooks que não chegam](07-troubleshooting.md#webhook-não-chega-ou-a-assinatura-não-confere) no troubleshooting.
- Entenda como o evento se encaixa no fluxo de [pagamentos](03-api-pagamentos.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** desenvolvedores que implementam o receptor e engenheiros de suporte que investigam eventos perdidos.
- **Estrutura de ponta a ponta:** o que é, como cadastrar, como validar, como processar com segurança e o que acontece quando falha. A ordem segue a vida real da integração.
- **Segurança como passo obrigatório,** não como apêndice. A validação da assinatura tem uma lista numerada, um aviso sobre o corpo bruto e exemplos completos em duas linguagens.
- **Garantias explícitas** (pelo menos uma vez, sem ordem garantida). Dizer o que a plataforma *não* promete evita bugs em produção.
- **Tabela de reenvio** com os intervalos reais, para o engenheiro de suporte saber quanto tempo esperar antes de intervir.
</details>
