# Desafio Técnico — Marketplace + Consumer

Projeto composto por duas APIs Spring Boot que se comunicam via webhook para simular um fluxo de marketplace.

- **Marketplace API** (porta 8080): gerencia lojas, pedidos e webhooks.
- **Consumer API** (porta 8081): recebe eventos de webhook, consulta os dados completos do pedido na Marketplace e persiste o registro.

---

## 1. Inicialização do projeto

Na raiz do projeto, execute:
```bash
git clone https://github.com/felipeamignone/marketplace-api.git
git clone https://github.com/felipeamignone/marketplace-consumer.git
```
Esse comando irá clonar os sub-diretórios do projeto.

Após os sub-diretórios serem clonados, execute:
```bash
docker-compose up --build
```

Esse comando irá subir três containers:

| Container              | Descrição                | Porta externa |
|------------------------|--------------------------|---------------|
| marketplace-postgres   | Banco de dados PostgreSQL | 5432          |
| marketplace-api        | Marketplace API           | 8080          |
| marketplace-consumer   | Consumer API              | 8081          |

Aguarde até que os logs indiquem que ambas as aplicações foram inicializadas com sucesso.

---

## 2. Cadastrar uma Store

```bash
curl -X POST http://localhost:8080/stores \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Loja Exemplo",
    "cnpj": "12345678000199"
  }'
```

**Resposta esperada (200):**

```json
{
  "id": "<store-id>",
  "name": "Loja Exemplo",
  "cnpj": "12345678000199"
}
```

> Copie o valor do campo `id` retornado. Ele será utilizado nos próximos passos como `<store-id>`.

---

## 3. Cadastrar um Webhook

Registre um webhook para que a Consumer API receba os eventos da loja cadastrada.

```bash
curl -X POST http://localhost:8080/webhooks \
  -H "Content-Type: application/json" \
  -d '{
    "storeIds": ["<store-id>"],
    "callbackUrl": "http://consumer:8080/events"
  }'
```

> Substitua `<store-id>` pelo UUID retornado no passo anterior.

**Resposta esperada (200):**

```json
{
  "id": "<webhook-id>",
  "callbackUrl": "http://consumer:8080/events",
  "storeIds": ["<store-id>"]
}
```

---

## 4. Criar um Pedido

Crie um pedido associado à loja cadastrada. Os itens são fictícios — preencha livremente.

```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{
    "storeId": "<store-id>",
    "items": [
      { "name": "Camiseta Azul", "quantity": 2 },
      { "name": "Calça Jeans", "quantity": 1 }
    ]
  }'
```

> Substitua `<store-id>` pelo UUID da loja.

**Resposta esperada (200):**

```json
{
  "orderId": "<order-id>",
  "storeId": "<store-id>",
  "status": "CREATED",
  "items": [...],
  "totalPrice": 32.97
}
```

> Copie o valor do campo `orderId`. Ele será utilizado nos próximos passos como `<order-id>`.

Neste momento, o evento `order.created` já foi disparado via webhook para a Consumer API, que consultou os dados do pedido e salvou o registro no banco.

---

## 5. Alterar status para SHIPPED (erro esperado)

Tente alterar o status do pedido diretamente de `CREATED` para `SHIPPED`. Essa transição é inválida pela máquina de estados do sistema.

```bash
curl -X PATCH http://localhost:8080/orders/<order-id>/status \
  -H "Content-Type: application/json" \
  -d '{
    "status": "SHIPPED"
  }'
```

> Substitua `<order-id>` pelo UUID do pedido.

**Resposta esperada (400 Bad Request):**

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "Cannot change status to 'Shipped' from current status"
}
```

A transição permitida a partir de `CREATED` é apenas para `PAID`.

---

## 6. Alterar status para PAID (sucesso)

```bash
curl -X PATCH http://localhost:8080/orders/<order-id>/status \
  -H "Content-Type: application/json" \
  -d '{
    "status": "PAID"
  }'
```

> Substitua `<order-id>` pelo UUID do pedido.

**Resposta esperada (200):**

```json
true
```

Neste momento, o evento `order.paid` foi disparado via webhook para a Consumer API.

---

## Validação dos resultados no banco de dados

Cada API se conecta em um database diferente dentro da mesma instância PostgreSQL:

| API              | Database              |
|------------------|-----------------------|
| Marketplace API  | `marketplace`         |
| Consumer API     | `marketplace_consumer`|

### Credenciais de acesso

- **Host:** `localhost`
- **Porta:** `5432`
- **Usuário:** `marketplace`
- **Senha:** `marketplace`

Você pode acessar via qualquer cliente PostgreSQL (DBeaver, pgAdmin, psql, etc.).

Exemplo com `psql`:

```bash
psql -h localhost -p 5432 -U marketplace -d marketplace
```

---

### Consultas no database `marketplace`

Conecte-se ao database `marketplace` e execute as queries abaixo.

**Listar lojas cadastradas:**

```sql
SELECT * FROM stores;
```

**Listar pedidos:**

```sql
SELECT * FROM orders;
```

**Listar itens dos pedidos:**

```sql
SELECT * FROM order_items;
```

**Listar webhooks cadastrados:**

```sql
SELECT * FROM webhooks;
```

**Listar store IDs associados aos webhooks:**

```sql
SELECT * FROM webhook_store_ids;
```

---

### Consultas no database `marketplace_consumer`

Conecte-se ao database `marketplace_consumer` e execute a query abaixo.

```bash
psql -h localhost -p 5432 -U marketplace -d marketplace_consumer
```

**Listar eventos recebidos pela Consumer API:**

```sql
SELECT id, event, order_id, store_id, event_timestamp, received_at FROM received_events;
```

**Visualizar o snapshot completo do pedido salvo em um evento:**

```sql
SELECT event, order_snapshot FROM received_events;
```

Você deverá ver dois registros:

1. Evento `order.created` — gerado no momento da criação do pedido (passo 4).
2. Evento `order.paid` — gerado na alteração de status para PAID (passo 6).

Cada registro contém o campo `order_snapshot` com os dados completos do pedido no momento em que o evento foi recebido.
