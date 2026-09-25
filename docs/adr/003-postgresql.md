# ADR-003 — PostgreSQL no servico-transacao

## Contexto

O `servico-transacao` persiste usuários e transações financeiras estruturadas, com relacionamentos e necessidade de consistência transacional.

## Decisão

O `servico-transacao` usa PostgreSQL 16 como seu banco exclusivo, acessado por Spring Data JPA. No ambiente central, o banco se chama `antifraude` e as credenciais são compartilhadas pelo Compose por `DB_USERNAME` e `DB_PASSWORD`.

## Consequências

- O serviço possui persistência relacional e transacional para seu próprio domínio.
- O schema é gerenciado pela configuração atual do serviço.
- `motor-risco` e `servico-auditoria` não acessam o PostgreSQL; recebem os dados necessários nos eventos RabbitMQ.
- O Compose mantém os dados no volume `postgres_data`.
