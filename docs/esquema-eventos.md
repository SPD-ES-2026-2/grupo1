# Esquema dos eventos

Eventos em JSON, publicados em Redis Pub/Sub e consumidos pelo integrador (Lettuce + Camel). Os campos seguem o [modelo canônico](modelo-canonico.md) e os IDs seguem o [mapeamento de IDs](mapeamento-ids.md).

## Canais

| Canal | Quem publica | Quem consome |
|---|---|---|
| `spd.pg.events` | Desktop (repositório JPA) | Integrador |
| `spd.neo.events` | API (decisões de risco) | Integrador |
| `spd.dlq` | Integrador (falhas) | Operação/análise manual |

## Envelope

Todo evento tem os campos abaixo.

| Campo | Tipo | Descrição |
|---|---|---|
| `eventId` | UUID | Identificador único; chave de idempotência |
| `eventType` | texto | Tipo do evento (tabela abaixo) |
| `entityType` | texto | `Customer`, `Account`, `Device`, `AccountDevice`, `Transaction` ou `RiskDecision` |
| `schemaVersion` | inteiro | Versão do esquema (hoje `1`) |
| `occurredAt` | data/hora | Quando ocorreu; permite descartar `updated` fora de ordem |
| `source` | texto | `desktop` ou `api` |
| `correlationId` | UUID | Liga eventos do mesmo fluxo (ex.: criação e decisão da mesma transação) |
| `payload` | objeto | Conteúdo, descrito abaixo |

### Payload

- **Desktop** (`source = desktop`): `payload.pgId` com o ID do PostgreSQL e `payload.data` com os campos da entidade, em que as chaves estrangeiras vêm como `…PgId`.
- **API** (`source = api`): `payload.canonicalId` com o UUID e `payload.data` com os campos.
- Eventos de exclusão (`*.deleted`, `account_device.unlinked`) não precisam de `data` completo, só do identificador.

## Tipos de evento

| `eventType` | `source` | Efeito no destino |
|---|---|---|
| `customer.created` / `updated` / `deleted` | desktop | `MERGE` ou `DETACH DELETE` em `Customer` |
| `account.created` / `updated` / `deleted` | desktop | `Account` + `OWNS` |
| `device.created` / `updated` | desktop | `Device` |
| `account_device.linked` / `unlinked` | desktop | cria ou remove `USES_DEVICE` |
| `transaction.created` | desktop | `Transaction` + `SENT` + `RECEIVED_BY` |
| `transaction.decided` | api | grava a decisão de risco no PostgreSQL |

## Exemplos

Evento do desktop:

```json
{
  "eventId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "eventType": "account.created",
  "entityType": "Account",
  "schemaVersion": 1,
  "occurredAt": "2026-11-27T14:03:22Z",
  "source": "desktop",
  "correlationId": "b1a2c3d4-0000-4000-8000-000000000001",
  "payload": {
    "pgId": 42,
    "data": {
      "customerPgId": 7,
      "number": "0001-23456-7",
      "status": "ATIVA",
      "openedAt": "2026-11-27T14:03:00Z"
    }
  }
}
```

Evento da API:

```json
{
  "eventId": "0f8fad5b-d9cb-469f-a165-70867728950e",
  "eventType": "transaction.decided",
  "entityType": "RiskDecision",
  "schemaVersion": 1,
  "occurredAt": "2026-11-27T14:05:01Z",
  "source": "api",
  "correlationId": "b1a2c3d4-0000-4000-8000-000000000002",
  "payload": {
    "canonicalId": "9b2f5c1e-6a3d-4e8b-a1c7-2d4f8e0b1a33",
    "data": {
      "score": 87,
      "decision": "BLOQUEAR",
      "evaluatedAt": "2026-11-27T14:05:01Z",
      "rules": ["R03", "R07"]
    }
  }
}
```

Aqui `canonicalId` é o UUID da transação avaliada.

## Garantias e limitações

- **Idempotência.** O integrador grava o `eventId` em `processed_event`; evento repetido é descartado sem alterar o resultado.
- **Ordem.** O Redis Pub/Sub não garante ordem entre publicadores. Para `updated`, o integrador ignora eventos com `occurredAt` anterior ao último aplicado.
- **Perda de mensagens.** Redis Pub/Sub não armazena mensagens: se o integrador estiver fora do ar, os eventos publicados nesse período se perdem. É uma limitação conhecida da abordagem pedida no tutorial. Uma mitigação possível é uma tabela *outbox* no PostgreSQL, com reenvio dos eventos pendentes.
- **Falhas.** Evento inválido ou que esgota as tentativas de *retry* vai para `spd.dlq`, com o erro e o evento original.
- **Evolução.** Mudanças incompatíveis incrementam `schemaVersion`; o integrador rejeita versões que não conhece.
