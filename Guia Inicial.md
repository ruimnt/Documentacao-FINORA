# Primeiros passos com a API da Finora

| | |
|---|---|
| **Público** | Desenvolvedores que vão integrar a API pela primeira vez |
| **Pré-requisitos** | Conhecimento básico de HTTP e JSON. cURL, Python 3.9 ou superior, ou Node.js 18 ou superior |
| **Tempo de leitura** | 12 minutos |
| **Versão da API** | v3 |

Neste guia você vai criar um pagamento Pix no ambiente de testes (sandbox) da Finora, do cadastro até a confirmação do pagamento. No fim, você terá feito quatro coisas:

1. Criado uma conta de desenvolvedor.
2. Gerado suas credenciais.
3. Obtido um token de acesso.
4. Criado e consultado um pagamento.

![Quatro passos: criar a conta sandbox, gerar as credenciais, obter o token e criar um pagamento](../images/01-passos-getting-started.svg)

> [!NOTE]
> Os exemplos seguem a **Livraria Boa Página**, um e-commerce fictício de livros. Você vai criar o pagamento do pedido #4821, no valor de R$ 150,00.

## Antes de começar

A Finora oferece dois ambientes. Use sempre o sandbox enquanto desenvolve.

| Ambiente | URL base da API | URL de autenticação | Dinheiro real? |
|---|---|---|---|
| **Sandbox** | `https://sandbox.api.finora.example/v3` | `https://sandbox.api.finora.example/oauth/token` | Não |
| **Produção** | `https://api.finora.example/v3` | `https://api.finora.example/oauth/token` | Sim |

> [!WARNING]
> Credenciais de sandbox e de produção são diferentes e não funcionam entre ambientes. Um token de sandbox enviado à produção retorna erro `401`.

## Passo 1: crie a conta sandbox

1. Acesse o portal do desenvolvedor em `https://developers.finora.example`.
2. Selecione **Criar conta** e preencha seus dados.
3. Confirme o e-mail pelo link que você receber.
4. No primeiro acesso, ative o **Ambiente sandbox**.

Quando o ambiente estiver ativo, o console exibe a etiqueta **Sandbox** no canto superior direito.

![Painel da Finora com a etiqueta Sandbox, indicadores de volume e a lista de últimos pagamentos](../images/05-mockup-painel-pagamentos.svg)

## Passo 2: gere as credenciais

1. No menu lateral, acesse **Desenvolvedores > Credenciais**.
2. Selecione **Criar credencial** e dê um nome, por exemplo `integracao-loja-sandbox`.
3. Copie o `client_id` e o `client_secret`.

> [!WARNING]
> O `client_secret` aparece **uma única vez**. Guarde-o agora em um gerenciador de segredos ou em uma variável de ambiente. Nunca o inclua no código-fonte nem o envie para o GitHub.

Salve as credenciais como variáveis de ambiente. Assim, os comandos deste guia funcionam sem edição:

```bash
export FINORA_CLIENT_ID="fin_cli_7Hq2xLmN"
export FINORA_CLIENT_SECRET="cole_seu_client_secret_aqui"
```

## Passo 3: obtenha um token de acesso

A API usa o protocolo OAuth 2.0 com o fluxo *client credentials*. Você troca o `client_id` e o `client_secret` por um token que vale por uma hora.

```bash
curl --request POST "https://sandbox.api.finora.example/oauth/token" \
  --header "Content-Type: application/json" \
  --data "{
    \"grant_type\": \"client_credentials\",
    \"client_id\": \"$FINORA_CLIENT_ID\",
    \"client_secret\": \"$FINORA_CLIENT_SECRET\",
    \"scope\": \"payments:write payments:read\"
  }"
```

Se tudo estiver certo, a resposta tem o status `200 OK`:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.exemplo.assinatura",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "payments:write payments:read"
}
```

Guarde o `access_token` em uma variável de ambiente:

```bash
export FINORA_TOKEN="eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.exemplo.assinatura"
```

> [!TIP]
> Em produção, não peça um token novo a cada requisição. Guarde o token e renove-o um pouco antes de expirar. O guia de [Autenticação](02-autenticacao.md) mostra como.

## Passo 4: crie seu primeiro pagamento

Crie um pagamento Pix de R$ 150,00. O campo `amount` usa **centavos**, então R$ 150,00 vira `15000`.

> [!NOTE]
> O cabeçalho `Idempotency-Key` evita pagamentos duplicados se você repetir a requisição por causa de uma falha de rede. Use um valor único para cada pedido.

#### cURL

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
    "customer": {
      "name": "Marina Souza",
      "document": "123.456.789-09",
      "email": "marina.souza@example.com"
    },
    "metadata": { "order_id": "4821" }
  }'
```

#### Python

```python
import os
import requests

BASE_URL = "https://sandbox.api.finora.example/v3"

response = requests.post(
    f"{BASE_URL}/payments",
    headers={
        "Authorization": f"Bearer {os.environ['FINORA_TOKEN']}",
        "Idempotency-Key": "pedido-4821-tentativa-1",
    },
    json={
        "amount": 15000,
        "currency": "BRL",
        "method": "pix",
        "description": "Pedido #4821 - Livraria Boa Página",
        "customer": {
            "name": "Marina Souza",
            "document": "123.456.789-09",
            "email": "marina.souza@example.com",
        },
        "metadata": {"order_id": "4821"},
    },
    timeout=10,
)
response.raise_for_status()
print(response.json()["id"])
```

#### JavaScript (Node.js 18 ou superior)

```javascript
const BASE_URL = "https://sandbox.api.finora.example/v3";

const response = await fetch(`${BASE_URL}/payments`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.FINORA_TOKEN}`,
    "Content-Type": "application/json",
    "Idempotency-Key": "pedido-4821-tentativa-1",
  },
  body: JSON.stringify({
    amount: 15000,
    currency: "BRL",
    method: "pix",
    description: "Pedido #4821 - Livraria Boa Página",
    customer: {
      name: "Marina Souza",
      document: "123.456.789-09",
      email: "marina.souza@example.com",
    },
    metadata: { order_id: "4821" },
  }),
});

if (!response.ok) {
  throw new Error(`Erro ${response.status}: ${await response.text()}`);
}

const payment = await response.json();
console.log(payment.id);
```

A API responde com `201 Created`. O pagamento nasce com o status `pending`, à espera do pagamento do Pix:

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

Guarde o valor de `id`. Você vai usá-lo no próximo passo.

## Passo 5: simule o pagamento e consulte o status

No sandbox ninguém paga o Pix de verdade. Em vez disso, você simula o pagamento com um endpoint exclusivo de testes:

```bash
curl --request POST \
  "https://sandbox.api.finora.example/v3/sandbox/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F/simulate" \
  --header "Authorization: Bearer $FINORA_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{ "event": "settled" }'
```

Em seguida, consulte o pagamento:

```bash
curl --request GET \
  "https://sandbox.api.finora.example/v3/payments/pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

O status agora é `settled` (liquidado):

```json
{
  "id": "pay_01JA8Z3K5M2Q7X9T4B6C8D0E1F",
  "object": "payment",
  "status": "settled",
  "amount": 15000,
  "currency": "BRL",
  "method": "pix",
  "fee": 0,
  "net_amount": 15000,
  "settled_at": "2026-10-05T13:32:41Z",
  "created_at": "2026-10-05T13:30:00Z",
  "updated_at": "2026-10-05T13:32:41Z"
}
```

> [!TIP]
> Em produção, você não precisa consultar o pagamento repetidamente. Configure um [webhook](06-webhooks.md) e a Finora avisa seu sistema quando o status mudar.

## Verifique seu resultado

Marque cada item para confirmar que a integração funciona:

- [ ] Recebi o `access_token` com `expires_in` igual a `3600`.
- [ ] Criei um pagamento e recebi `201 Created` com status `pending`.
- [ ] Simulei o pagamento e consultei o status `settled`.
- [ ] Guardei o `client_secret` fora do código-fonte.

Algo deu errado? Consulte o [guia de troubleshooting](07-troubleshooting.md).

## Próximos passos

- Entenda o fluxo de segurança em [Autenticação](02-autenticacao.md).
- Veja todos os parâmetros e erros em [API de Pagamentos](03-api-pagamentos.md).
- Receba eventos automaticamente com [Webhooks](06-webhooks.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** desenvolvedores novos na Finora, que querem um resultado funcionando antes de ler a referência completa.
- **Um único caminho.** O guia não apresenta alternativas nem opções avançadas. Cada passo termina com algo que o leitor pode ver (uma resposta JSON), o que dá confiança para continuar.
- **Imagem de abertura** com os quatro passos, para o leitor enxergar o caminho inteiro antes de começar.
- **Exemplos em três formas** (cURL, Python e JavaScript), porque a mesma pessoa pode testar no terminal e depois escrever o código em uma linguagem.
- **Avisos no ponto exato do risco.** O alerta sobre o `client_secret` aparece no passo em que ele é gerado, não em uma seção de segurança no fim.
- **Lista de verificação final** para o leitor confirmar sozinho que terminou.
</details>.
