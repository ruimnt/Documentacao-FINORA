# Changelog da API

| | |
|---|---|
| **Público** | Gerentes de produto, líderes técnicos e desenvolvedores que acompanham mudanças na API |
| **Pré-requisitos** | Nenhum |
| **Tempo de leitura** | 7 minutos |
| **Versão da API** | v3 (atual) |

Esta página registra as mudanças da API da Finora, da mais recente para a mais antiga. Use-a para saber **o que mudou** e **se você precisa agir**.

> [!WARNING]
> A **v2 será descontinuada em 16 de junho de 2027**. Depois dessa data, as requisições à v2 retornam erro. Veja o [guia de migração da v2 para a v3](09-guia-de-migracao-v2-para-v3.md).

## Como ler este changelog

### Tipos de mudança

| Tipo | Significado |
|---|---|
| **Added** (novo) | Um recurso ou campo novo, compatível com o que você já usa |
| **Changed** (alterado) | Uma mudança no comportamento existente |
| **Deprecated** (depreciado) | Um recurso que ainda funciona, mas será removido |
| **Removed** (removido) | Um recurso que deixou de existir |
| **Fixed** (corrigido) | Uma correção de erro |
| **Security** (segurança) | Uma mudança relacionada à segurança |

### Ação necessária

Cada versão traz uma linha **Ação necessária**:

| Valor | O que significa |
|---|---|
| **Nenhuma** | Sua integração continua funcionando sem alterações |
| **Recomendada** | Nada quebra agora, mas você deve se preparar |
| **Obrigatória** | Sua integração para de funcionar se você não agir |

## Política de versionamento

A Finora segue o [versionamento semântico](https://semver.org/lang/pt-BR/):

| Tipo | Exemplo | O que muda | Aviso prévio |
|---|---|---|---|
| **Maior** | 2.x para 3.0 | Mudanças que quebram a compatibilidade | 12 meses |
| **Menor** | 3.1 para 3.2 | Recursos novos, compatíveis | Nenhum |
| **Correção** | 3.2.0 para 3.2.1 | Correções de erros | Nenhum |

Só a versão **maior** aparece na URL (`/v3`). Mudanças menores e correções chegam automaticamente, sem você alterar nada.

> [!NOTE]
> Quando uma versão é descontinuada, a Finora envia um e-mail aos contatos técnicos da conta e inclui, nas respostas, o cabeçalho `Sunset` com a data de encerramento. Exemplo: `Sunset: Wed, 16 Jun 2027 00:00:00 GMT`.

## Linha do tempo das versões

| Versão | Status | Fim do suporte |
|---|---|---|
| **v3** | Atual | Sem data prevista |
| **v2** | Descontinuada (*deprecated*) | 16/06/2027 |
| **v1** | Desativada | 31/12/2025 |

---

## [3.2.0] - 22/09/2026

**Ação necessária:** Recomendada (apenas a mudança de segurança, para endpoints de webhook antigos).

### Added

- `GET /cards/{card_id}/invoices` aceita o filtro `status` (`open`, `closed`, `paid` e `overdue`).
- `GET /crypto/quotes` aceita os pares `ETH-BRL` e `SOL-BRL`.
- Novo evento de webhook `card.invoice.closed`.

### Changed

- O limite de requisições passou de **300 para 600 por minuto**, por credencial.

### Fixed

- `GET /payments` repetia itens entre páginas quando vários pagamentos tinham o mesmo `created_at`.

### Security

- Endpoints de webhook cadastrados a partir de 22/09/2026 exigem HTTPS com certificado válido. Endpoints mais antigos com HTTP continuam funcionando até **31/12/2026**. Atualize-os antes dessa data.

---

## [3.1.0] - 11/08/2026

**Ação necessária:** Recomendada (apenas se você usa o parâmetro `expand=customer`).

### Added

- `PATCH /payments/{payment_id}` para atualizar `description` e `metadata`.
- `POST /reconciliation/matches/{match_id}/confirm` para confirmar uma correspondência.
- `GET /reconciliation/reports/{date}` para consultar o resumo diário da conciliação.
- O campo `reason` em estornos, com os valores `customer_request`, `duplicate` e `fraud`.

### Deprecated

- O parâmetro `expand=customer` em `GET /payments`. O objeto `customer` já vem sempre incluído. O parâmetro será removido na v4.

### Fixed

- O `net_amount` de pagamentos de cartão parcelado agora é arredondado sobre o total, e não parcela a parcela. Isso reduz divergências de centavos na conciliação.

---

## [3.0.0] - 16/06/2026

**Ação necessária:** Obrigatória para quem usa a v2. Quem começou a integração depois desta data já usa a v3.

Esta é uma versão **maior** e traz mudanças que quebram a compatibilidade.

### Changed (mudanças que quebram a compatibilidade)

| Área | v2 | v3 |
|---|---|---|
| **Autenticação** | Cabeçalho `X-Api-Key` com chave fixa | OAuth 2.0 com `POST /oauth/token` e token de 1 hora |
| **Valores** | `amount` decimal em reais (`150.00`) | `amount` inteiro em centavos (`15000`) |
| **Recurso principal** | `/v2/transactions` | `/v3/payments` |
| **Paginação** | `page` e `per_page` | `cursor` e `limit` |
| **Formato de erro** | `{ "status": "error", "message": "...", "code": 4022 }` | Objeto `error` com `code` em texto, `param` e `request_id` |
| **Webhooks** | Token estático no cabeçalho `X-Finora-Token` | Assinatura HMAC-SHA256 no cabeçalho `Finora-Signature` |
| **Status do pagamento** | `waiting`, `approved`, `refused` | `pending`, `settled`, `failed` |

### Added

- APIs de **Investimentos**, **Cripto**, **Cartão** e **Conciliação**.
- Cabeçalho `Idempotency-Key` para operações que criam ou alteram dinheiro.
- Endpoint `POST /sandbox/payments/{payment_id}/simulate` para simular eventos no sandbox.
- Escopos de acesso por produto (`payments:read`, `cards:write` e outros).

### Deprecated

- Toda a **v2**, com encerramento em 16/06/2027.

---

## [2.9.0] - 10/02/2026

**Ação necessária:** Nenhuma.

### Added

- O campo `expires_in` na criação de pagamentos Pix e boleto.

### Fixed

- O evento `transaction.refused` não era enviado para boletos expirados.

---

**Próximos passos:** se você usa a v2, comece pelo [guia de migração](09-guia-de-migracao-v2-para-v3.md). Se você já usa a v3, revise as tabelas de ação necessária acima.

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** quem precisa decidir e planejar, como gerentes de produto e líderes técnicos, além de desenvolvedores que acompanham as versões. Esse leitor quer saber "isso afeta a minha operação?", não os detalhes da implementação.
- **Linha "Ação necessária" em cada versão.** É a informação mais importante, e por isso aparece no topo de cada entrada, em um vocabulário fixo de três valores.
- **Formato inspirado no padrão *Keep a Changelog*:** versões da mais recente para a mais antiga e categorias fixas. Quem já leu um changelog reconhece a estrutura.
- **Mudanças que quebram em tabela comparativa** (v2 e v3), para o leitor enxergar o antes e o depois lado a lado.
- **Política de versionamento e de descontinuação no mesmo documento,** com datas explícitas. Prazos vagos geram incidentes. O "como migrar" fica no guia separado.
</details>
