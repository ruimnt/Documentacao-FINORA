# Autenticação

| | |
|---|---|
| **Público** | Desenvolvedores e engenheiros de suporte |
| **Pré-requisitos** | Credenciais criadas no portal (veja [Primeiros passos](01-getting-started.md)) |
| **Tempo de leitura** | 8 minutos |
| **Versão da API** | v3 |

A API da Finora usa **OAuth 2.0** com o fluxo *client credentials*. Esse fluxo serve para comunicação entre servidores: sua aplicação se identifica com um `client_id` e um `client_secret` e recebe um token de acesso de curta duração.

## Como o fluxo funciona

**Diagrama de sequência: a aplicação pede o token, o servidor de autorização responde com o access_token e a API valida o token a cada requisição**

<img width="960" height="520" alt="Image" src="https://github.com/user-attachments/assets/160f5201-a42e-4d5a-b455-569a1d1f9603" />

1. Sua aplicação envia o `client_id`, o `client_secret` e os escopos desejados para `POST /oauth/token`.
2. O servidor de autorização responde com um `access_token` que expira em 3600 segundos (1 hora).
3. Sua aplicação envia o token no cabeçalho `Authorization` de cada requisição.
4. A API valida a assinatura, a validade e o escopo do token antes de responder.

> [!NOTE]
> A Finora não usa chaves de API fixas na v3. Se você ainda usa o cabeçalho `X-Api-Key` da v2, siga o [guia de migração](09-guia-de-migracao-v2-para-v3.md).

## Obtenha um token

**Endpoint:** `POST /oauth/token`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `grant_type` | string | Sim | Use sempre `client_credentials`. |
| `client_id` | string | Sim | Identificador público da credencial, por exemplo `fin_cli_7Hq2xLmN`. |
| `client_secret` | string | Sim | Segredo da credencial. Trate-o como uma senha. |
| `scope` | string | Não | Escopos separados por espaço. Se omitido, o token recebe todos os escopos da credencial. |

**Requisição:**

```bash
curl --request POST "https://sandbox.api.finora.example/oauth/token" \
  --header "Content-Type: application/json" \
  --data "{
    \"grant_type\": \"client_credentials\",
    \"client_id\": \"$FINORA_CLIENT_ID\",
    \"client_secret\": \"$FINORA_CLIENT_SECRET\",
    \"scope\": \"payments:read reconciliation:read\"
  }"
```

**Resposta (`200 OK`):**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.exemplo.assinatura",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "payments:read reconciliation:read"
}
```

## Use o token nas requisições

Envie o token no cabeçalho `Authorization`, com o prefixo `Bearer`:

```bash
curl --request GET "https://sandbox.api.finora.example/v3/payments?limit=5" \
  --header "Authorization: Bearer $FINORA_TOKEN"
```

Toda resposta da API inclui o cabeçalho `X-Request-Id`. Guarde esse valor: o suporte da Finora usa o identificador para localizar sua requisição.

## Escopos

Os escopos limitam o que um token pode fazer. Peça apenas os escopos de que sua integração precisa.

| Escopo | Permite |
|---|---|
| `payments:read` | Consultar e listar pagamentos |
| `payments:write` | Criar, atualizar, cancelar e estornar pagamentos |
| `reconciliation:read` | Consultar correspondências e relatórios de conciliação |
| `reconciliation:write` | Enviar extratos e confirmar correspondências |
| `investments:read` | Consultar produtos e posições de investimento |
| `investments:write` | Criar ordens de investimento |
| `crypto:read` | Consultar cotações e carteiras de cripto |
| `crypto:write` | Criar ordens de compra e venda de cripto |
| `cards:read` | Consultar cartões e faturas |
| `cards:write` | Criar cartões, bloquear e alterar limites |
| `webhooks:manage` | Criar, listar e remover endpoints de webhook |

> [!TIP]
> Siga o princípio do menor privilégio. Um sistema que só consulta pagamentos deve usar apenas `payments:read`. Se o token vazar, o estrago fica limitado.

## Renove o token antes de expirar

O token vale por uma hora. Peça um novo token **cerca de 60 segundos antes de expirar**, para evitar erros `401` em requisições em andamento. Não peça um token a cada requisição: isso é lento e consome seu limite de chamadas.

O exemplo em Python abaixo guarda o token em memória e o renova quando necessário:

```python
import os
import time

import requests

TOKEN_URL = "https://sandbox.api.finora.example/oauth/token"


class FinoraAuth:
    """Guarda o access_token e o renova pouco antes de expirar."""

    def __init__(self, client_id: str, client_secret: str, scope: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.scope = scope
        self._token = None
        self._expires_at = 0.0

    def get_token(self) -> str:
        # Renova 60 segundos antes do vencimento.
        if self._token is None or time.time() >= self._expires_at - 60:
            response = requests.post(
                TOKEN_URL,
                json={
                    "grant_type": "client_credentials",
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                    "scope": self.scope,
                },
                timeout=10,
            )
            response.raise_for_status()
            data = response.json()
            self._token = data["access_token"]
            self._expires_at = time.time() + data["expires_in"]
        return self._token


auth = FinoraAuth(
    os.environ["FINORA_CLIENT_ID"],
    os.environ["FINORA_CLIENT_SECRET"],
    "payments:read payments:write",
)
headers = {"Authorization": f"Bearer {auth.get_token()}"}
```

## Erros de autenticação

O endpoint `/oauth/token` segue o formato de erro do padrão OAuth 2.0 (RFC 6749). Já os endpoints da API `/v3` usam o [formato de erro da Finora](03-api-pagamentos.md#formato-dos-erros).

**Erro do endpoint `/oauth/token` (`401 Unauthorized`):**

```json
{
  "error": "invalid_client",
  "error_description": "O client_id ou o client_secret está incorreto."
}
```

| HTTP | `error` ou `code` | Causa | Como resolver |
|---|---|---|---|
| 400 | `invalid_scope` | Você pediu um escopo que a credencial não possui | Peça apenas os escopos habilitados na credencial |
| 401 | `invalid_client` | `client_id` ou `client_secret` incorretos, ou credencial de outro ambiente | Confira os valores e o ambiente (sandbox ou produção) |
| 401 | `token_expired` | O token passou de 3600 segundos | Peça um novo token |
| 401 | `invalid_token` | Token malformado, adulterado ou ausente | Envie o cabeçalho `Authorization: Bearer <token>` completo |
| 403 | `insufficient_scope` | O token não tem o escopo exigido pelo endpoint | Peça um token com o escopo correto |

Para outros cenários, consulte o [troubleshooting](07-troubleshooting.md).

## Boas práticas de segurança

> [!WARNING]
> Nunca coloque o `client_secret` em aplicativos móveis, em código JavaScript de navegador ou em repositórios públicos. Quem tem o segredo pode agir em nome da sua empresa. Faça as chamadas à Finora **somente a partir do seu servidor**.

- **Guarde segredos em variáveis de ambiente** ou em um gerenciador de segredos.
- **Use uma credencial por sistema.** Se uma credencial vazar, você a revoga sem afetar os outros sistemas.
- **Rotacione os segredos periodicamente.** No portal, em **Desenvolvedores > Credenciais > Rotacionar segredo**, o segredo antigo continua válido por 24 horas para você atualizar seus sistemas sem interrupção.
- **Restrinja endereços IP em produção.** Cadastre os IPs do seu servidor na lista de permissões da credencial.
- **Nunca registre tokens em logs.** Registre apenas o `X-Request-Id`.

## Próximos passos

- Crie e consulte pagamentos em [API de Pagamentos](03-api-pagamentos.md).
- Entenda como autenticar os eventos recebidos em [Webhooks](06-webhooks.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** desenvolvedores que implementam a autenticação e engenheiros de suporte que diagnosticam erros `401` e `403`.
- **Explicação antes da prática.** A página abre com o diagrama e quatro frases, e só depois mostra os campos e os exemplos. Quem entende o fluxo comete menos erros ao implementar.
- **Tabela de escopos** reunida em um único lugar, para o leitor decidir o que pedir sem abrir outras páginas.
- **Código de renovação do token** incluído porque é a dúvida mais comum depois da primeira integração.
- **Erros em tabela com a causa e a solução**, ligada ao guia de troubleshooting para os casos que exigem diagnóstico.
- **Segurança ao final**, com o aviso mais importante em destaque, em vez de espalhar recomendações genéricas.
</details>
