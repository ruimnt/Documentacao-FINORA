# Glossário e perguntas frequentes

| | |
|---|---|
| **Público** | Todos os leitores da documentação, em especial analistas financeiros, gerentes e pessoas novas em pagamentos |
| **Pré-requisitos** | Nenhum |
| **Tempo de leitura** | 8 minutos |
| **Versão da API** | v3 |

Esta página explica os termos financeiros e técnicos usados na documentação da Finora e responde às dúvidas mais comuns.

## Glossário

### Termos de pagamentos

| Termo | Significado |
|---|---|
| **Pix** | Meio de pagamento instantâneo brasileiro, que funciona 24 horas por dia. |
| **Boleto** | Documento de cobrança que o cliente paga em um banco ou aplicativo. A compensação leva até 3 dias úteis. |
| **Pagamento** | O registro, na Finora, de uma cobrança. Na API, é o objeto `payment`. |
| **Liquidação** (*settlement*) | O momento em que o dinheiro chega à sua conta. Na API, corresponde ao status `settled`. |
| **Valor bruto** | O valor cobrado do cliente (`amount`). |
| **Valor líquido** | O valor que você recebe, depois de descontar a tarifa (`net_amount`). |
| **Tarifa** (*fee*) | O valor que a Finora cobra por um pagamento. |
| **Estorno** | A devolução, total ou parcial, de um pagamento já liquidado. |
| **Parcelamento** | A divisão de um pagamento de cartão em parcelas. |
| **Chargeback** | A contestação de uma compra feita pelo titular do cartão ao seu banco. |

### Termos de conciliação

| Termo | Significado |
|---|---|
| **Conciliação** | A comparação entre o extrato bancário e os pagamentos registrados, para confirmar que cada real recebido tem origem conhecida. |
| **Extrato bancário** | A lista de entradas e saídas de uma conta no banco. |
| **Lançamento** | Cada linha do extrato bancário. |
| **Correspondência** (*match*) | O vínculo entre um lançamento do extrato e um pagamento. |
| **Divergência** | Quando o lançamento e o pagamento correspondem, mas o valor é diferente do esperado. |
| **Antecipação de recebíveis** | O recebimento antecipado de valores de vendas parceladas no cartão, mediante uma tarifa. |

### Termos de investimentos, cripto e cartão

| Termo | Significado |
|---|---|
| **Renda fixa** | Investimento cuja regra de rendimento é conhecida na hora da aplicação. |
| **CDB** | Certificado de Depósito Bancário, um título de renda fixa emitido por bancos. |
| **CDI** | Taxa de referência do mercado financeiro. "110% do CDI" significa que o investimento rende 110% dessa taxa. |
| **Tesouro Selic** | Título público de renda fixa, ligado à taxa básica de juros. |
| **Liquidez** | A rapidez com que você transforma o investimento em dinheiro. "D+0" é no mesmo dia, "D+1" é no dia útil seguinte. |
| **Posição** | O que um cliente tem investido em um produto, com o valor atual. |
| **Criptoativo** | Ativo digital negociado em redes de blockchain, como o bitcoin (BTC) e o ether (ETH). |
| **Cotação** (*quote*) | O preço oferecido para comprar ou vender um criptoativo, válido por poucos segundos. |
| **Carteira** (*wallet*) | O local onde o saldo de criptoativos de um cliente fica registrado. |
| **Fatura** | O resumo das compras de um cartão de crédito em um período, com a data de vencimento. |
| **Limite** | O valor máximo que o cliente pode gastar no cartão. |
| **Cartão virtual** | Cartão sem plástico, com número próprio, usado em compras online. |

### Termos técnicos

| Termo | Significado |
|---|---|
| **API** | Interface que permite que um sistema converse com outro. |
| **REST** | Estilo de API que usa URLs para identificar recursos e métodos HTTP para as ações. |
| **Método HTTP** | A ação da requisição: `GET` consulta, `POST` cria, `PATCH` atualiza em parte, `DELETE` remove. |
| **JSON** | Formato de texto para trocar dados, feito de pares de chave e valor. |
| **Endpoint** | O endereço (URL) de uma operação da API. |
| **Sandbox** | Ambiente de testes, sem dinheiro real. |
| **OAuth 2.0** | Padrão de autenticação em que você troca credenciais por um token de curta duração. |
| **Token de acesso** (*access token*) | Código temporário que prova a sua identidade em cada requisição. |
| **Bearer** | Tipo de token enviado no cabeçalho `Authorization: Bearer <token>`. |
| **Escopo** (*scope*) | Permissão que limita o que um token pode fazer. |
| **Idempotência** | Propriedade de uma operação que, repetida, produz o mesmo resultado, sem duplicar efeitos. |
| **Webhook** | Chamada que a Finora faz ao seu sistema quando um evento acontece. |
| **HMAC** | Técnica criptográfica que gera uma assinatura a partir de um segredo, usada para provar que o webhook veio da Finora. |
| **Paginação por cursor** | Forma de percorrer listas longas usando um marcador (`cursor`) que aponta para a página seguinte. |
| **Limite de requisições** (*rate limit*) | O número máximo de chamadas permitidas por minuto. |
| **`request_id`** | Identificador único de cada requisição, usado para investigar problemas. |

## Perguntas frequentes

### Por que os valores são em centavos?

Números decimais podem gerar erros de arredondamento. Com inteiros, R$ 150,00 é sempre `15000`, e a soma de vários pagamentos não perde nem ganha centavos. A [API de Pagamentos](03-api-pagamentos.md#convenções) explica as convenções.

### Qual a diferença entre `authorized` e `settled`?

Em `authorized`, o pagamento foi aprovado, mas o dinheiro ainda não chegou à sua conta. Em `settled`, o dinheiro já foi liquidado. Libere o produto ou o serviço ao cliente quando receber `settled`, a menos que o seu negócio aceite o risco de agir antes da liquidação.

### Por que meu extrato não bate com o valor do pagamento?

O extrato mostra o valor **líquido** (`net_amount`), depois da tarifa. Compare com `net_amount`, não com `amount`. Se ainda houver diferença, veja o troubleshooting de [divergência de centavos](07-troubleshooting.md#conciliação-com-divergência-de-centavos).

### Posso testar em produção com valores pequenos?

Não recomendamos. Use o sandbox, que simula todos os cenários sem custo e sem dinheiro real. Pagamentos de produção geram tarifas e movimentam dinheiro de verdade.

### Por quanto tempo uma `Idempotency-Key` vale?

24 horas. Depois disso, a mesma chave é tratada como nova.

### O que acontece se meu endpoint de webhook ficar fora do ar?

A Finora tenta reenviar o evento por até 24 horas, com intervalos crescentes. Passado esse prazo, o evento fica marcado como `failed` no console, onde você pode reenviá-lo. Veja a [política de reenvio](06-webhooks.md#política-de-reenvio).

### Como sei se estou usando a v2 ou a v3?

Olhe a URL: `/v2/...` é a v2 e `/v3/...` é a v3. Se você autentica com `X-Api-Key`, está na v2. A v2 será descontinuada em 16/06/2027. Veja o [guia de migração](09-guia-de-migracao-v2-para-v3.md).

### Onde encontro o `request_id` de uma requisição?

No corpo de qualquer erro (`error.request_id`) e no cabeçalho `X-Request-Id` de todas as respostas, inclusive as de sucesso.

### Os investimentos e as cotações de cripto desta documentação são reais?

Não. A Finora é uma empresa fictícia, e todos os produtos, taxas, preços e dados desta documentação são inventados para fins de demonstração.

## Próximos passos

- Comece a integração em [Primeiros passos](01-getting-started.md).
- Se encontrou um erro, consulte o [Troubleshooting](07-troubleshooting.md).

<details>
<summary>Nota do autor: público e decisões de estrutura</summary>

- **Público:** leitores de perfis muito diferentes. A analista financeira conhece "conciliação" e "liquidação", mas não "idempotência". O desenvolvedor conhece "idempotência", mas não "CDI". O glossário serve aos dois.
- **Glossário dividido em quatro grupos** (pagamentos, conciliação, produtos e termos técnicos). Uma lista única e longa dificultaria a busca, e os grupos permitem ao leitor pular para o seu domínio.
- **Definições em uma frase,** sem jargão novo para explicar o jargão, e com o nome do campo da API quando existe (por exemplo, `settled`).
- **Perguntas frequentes escritas como o leitor perguntaria** ("Por que meu extrato não bate?"), com respostas curtas e links para a explicação completa. A FAQ é um atalho, e não duplica o conteúdo das outras páginas.
</details>
