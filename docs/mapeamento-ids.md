# Mapeamento de IDs

PostgreSQL usa chaves numéricas (`pgId`) e o modelo canônico usa UUID ([modelo-canonico.md](modelo-canonico.md)). O integrador mantém uma tabela de correspondência entre os dois. O Neo4j **não usa o ID interno dele**: guarda o UUID em `id` e o ID do PostgreSQL em `pgId`.

## Tabelas do integrador

Ficam no schema `integracao` do PostgreSQL:

```sql
CREATE SCHEMA IF NOT EXISTS integracao;

CREATE TABLE integracao.id_mapping (
  canonical_id UUID        PRIMARY KEY,
  entity_type  VARCHAR(30) NOT NULL,
  pg_id        BIGINT      NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (entity_type, pg_id)
);

CREATE TABLE integracao.processed_event (
  event_id     UUID        PRIMARY KEY,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- `id_mapping` liga cada entidade (`entity_type` + `pg_id`) ao seu UUID canônico.
- `processed_event` guarda os eventos já tratados, para garantir idempotência ([esquema-eventos.md](esquema-eventos.md)).
- `AccountDevice` não tem ID próprio: é identificada por (`accountId`, `deviceId`) e fica fora do `id_mapping`.

## Regras

1. **PostgreSQL → Neo4j.** O integrador consulta `(entity_type, pg_id)`. Se não existir, gera um UUID e insere em `id_mapping`. Depois grava o nó no Neo4j com `id` = UUID e `pgId` = ID do PostgreSQL.
2. **Chaves estrangeiras.** Campos como `customerPgId` ou `originAccountPgId` são convertidos para UUID pelo mesmo mapeamento antes de montar a entidade canônica.
3. **Pai ainda não mapeado.** Se o evento referencia uma entidade que ainda não existe no mapeamento (por exemplo, conta antes do cliente), o evento é reprocessado com *retry* e, esgotadas as tentativas, vai para a fila `spd.dlq`.
4. **Neo4j → PostgreSQL.** Eventos da API trazem o UUID. O integrador resolve o `pg_id` em `id_mapping` e grava a decisão de risco na transação correspondente.
5. **Atomicidade.** A inserção em `id_mapping` e a escrita no destino devem ser tratadas como uma unidade; se a escrita no destino falhar, o evento é refeito sem criar um segundo UUID, graças à restrição `UNIQUE (entity_type, pg_id)`.
6. **Exclusão.** Ao excluir uma entidade, o mapeamento é mantido, para o UUID não ser reaproveitado e eventos atrasados não ressuscitarem o registro.

## Exemplo

| `entity_type` | `pg_id` | `canonical_id` |
|---|---|---|
| `Account` | 42 | `3f2504e0-4f89-41d3-9a0c-0305e82c3301` |

No Neo4j:

```cypher
(:Account {id: "3f2504e0-4f89-41d3-9a0c-0305e82c3301", pgId: 42})
```
